---
title: dex-load-process-native
tags:
cover:
---







# 背景



前期我们分析了文章

{% post_link android/os/framework/dex-load-process 'Android Dex加载流程-Java层分析' %}



最近突然对底层Native的加载逻辑起了兴趣想尝试分析下



# 环境



Android系统：android13_r78

{% post_link android/os/framework/aosp-env 'Android 源码阅读环境搭建' %}



# 前置概念



- apk文件
- dex文件
- odex文件
- vdex文件
- art文件
- prof文件
- 



# 分析过程



入口：续之前的文章分析，native加载Dex文件的入口是DexFile.openDexNative



## openDexFileNative



> 这是代码执行的入口～

``` c++
static jobject DexFile_openDexFileNative(JNIEnv* env,
                                         jclass,
                                         jstring javaSourceName,
                                         jstring javaOutputName ATTRIBUTE_UNUSED,
                                         jint flags ATTRIBUTE_UNUSED,
                                         jobject class_loader,
                                         jobjectArray dex_elements) {
    // 边界case检查
  ScopedUtfChars sourceName(env, javaSourceName);
  if (sourceName.c_str() == nullptr) {
    return nullptr;
  }
  // open dexFiles
  std::vector<std::string> error_msgs;
  const OatFile* oat_file = nullptr;
  std::vector<std::unique_ptr<const DexFile>> dex_files =
      Runtime::Current()->GetOatFileManager().OpenDexFilesFromOat(sourceName.c_str(),
                                                                  class_loader,
                                                                  dex_elements,
                                                                  /*out*/ &oat_file,
                                                                  /*out*/ &error_msgs);
    // 将dex_files转为java数组返回
  return CreateCookieFromOatFileManagerResult(env, dex_files, oat_file, error_msgs);
}
```



## openDexFilesFromOat



> TODO

``` c++
std::vector<std::unique_ptr<const DexFile>> OatFileManager::OpenDexFilesFromOat(
    const char* dex_location,
    jobject class_loader,
    jobjectArray dex_elements,
    const OatFile** out_oat_file,
    std::vector<std::string>* error_msgs) {
  ScopedTrace trace(StringPrintf("%s(%s)", __FUNCTION__, dex_location));
  CHECK(dex_location != nullptr);
  CHECK(error_msgs != nullptr);

  // Verify we aren't holding the mutator lock, which could starve GC when
  // hitting the disk.
  Thread* const self = Thread::Current();
  Locks::mutator_lock_->AssertNotHeld(self);
  Runtime* const runtime = Runtime::Current();

  std::vector<std::unique_ptr<const DexFile>> dex_files;
    // 根据ClassLoader & dex_elements创建 ContextClassLoader
  std::unique_ptr<ClassLoaderContext> context(
      ClassLoaderContext::CreateContextForClassLoader(class_loader, dex_elements));

  // If the class_loader is null there's not much we can do. This happens if a dex files is loaded
  // directly with DexFile APIs instead of using class loaders.
  if (class_loader == nullptr) {
    LOG(WARNING) << "Opening an oat file without a class loader. "
                 << "Are you using the deprecated DexFile APIs?";
  } else if (context != nullptr) {
    OatFileAssistant oat_file_assistant(dex_location,
                                        kRuntimeISA,
                                        context.get(),
                                        runtime->GetOatFilesExecutable(),
                                        only_use_system_oat_files_);

    // Get the current optimization status for trace debugging.
    // Implementation detail note: GetOptimizationStatus will select the same
    // oat file as GetBestOatFile used below, and in doing so it already pre-populates
    // some OatFileAssistant internal fields.
    std::string odex_location;
    std::string compilation_filter;
    std::string compilation_reason;
    std::string odex_status;
      // 获取oat status状态
    oat_file_assistant.GetOptimizationStatus(
        &odex_location,
        &compilation_filter,
        &compilation_reason,
        &odex_status);

    Runtime::Current()->GetAppInfo()->RegisterOdexStatus(
        dex_location,
        compilation_filter,
        compilation_reason,
        odex_status);

    ScopedTrace odex_loading(StringPrintf(
        "location=%s status=%s filter=%s reason=%s",
        odex_location.c_str(),
        odex_status.c_str(),
        compilation_filter.c_str(),
        compilation_reason.c_str()));

    // Proceed with oat file loading.
    std::unique_ptr<const OatFile> oat_file(oat_file_assistant.GetBestOatFile().release());
    VLOG(oat) << "OatFileAssistant(" << dex_location << ").GetBestOatFile()="
              << (oat_file != nullptr ? oat_file->GetLocation() : "")
              << " (executable=" << (oat_file != nullptr ? oat_file->IsExecutable() : false) << ")";

    CHECK(oat_file == nullptr || odex_location == oat_file->GetLocation())
        << "OatFileAssistant non-determinism in choosing best oat files. "
        << "optimization-status-location=" << odex_location
        << " best_oat_file-location=" << oat_file->GetLocation();

    if (oat_file != nullptr) {
      // Load the dex files from the oat file.
      bool added_image_space = false;
      if (oat_file->IsExecutable()) {
        ScopedTrace app_image_timing("AppImage:Loading");

        // We need to throw away the image space if we are debuggable but the oat-file source of the
        // image is not otherwise we might get classes with inlined methods or other such things.
        std::unique_ptr<gc::space::ImageSpace> image_space;
        if (ShouldLoadAppImage(oat_file.get())) {
          image_space = oat_file_assistant.OpenImageSpace(oat_file.get());
        }
        if (image_space != nullptr) {
          ScopedObjectAccess soa(self);
          StackHandleScope<1> hs(self);
          Handle<mirror::ClassLoader> h_loader(
              hs.NewHandle(soa.Decode<mirror::ClassLoader>(class_loader)));
          // Can not load app image without class loader.
          if (h_loader != nullptr) {
            std::string temp_error_msg;
            // Add image space has a race condition since other threads could be reading from the
            // spaces array.
            {
              ScopedThreadSuspension sts(self, ThreadState::kSuspended);
              gc::ScopedGCCriticalSection gcs(self,
                                              gc::kGcCauseAddRemoveAppImageSpace,
                                              gc::kCollectorTypeAddRemoveAppImageSpace);
              ScopedSuspendAll ssa("Add image space");
              runtime->GetHeap()->AddSpace(image_space.get());
            }
            {
              ScopedTrace image_space_timing("Adding image space");
              added_image_space = runtime->GetClassLinker()->AddImageSpace(image_space.get(),
                                                                           h_loader,
                                                                           /*out*/&dex_files,
                                                                           /*out*/&temp_error_msg);
            }
            if (added_image_space) {
              // Successfully added image space to heap, release the map so that it does not get
              // freed.
              image_space.release();  // NOLINT b/117926937

              // Register for tracking.
              for (const auto& dex_file : dex_files) {
                dex::tracking::RegisterDexFile(dex_file.get());
              }
            } else {
              LOG(INFO) << "Failed to add image file " << temp_error_msg;
              dex_files.clear();
              {
                ScopedThreadSuspension sts(self, ThreadState::kSuspended);
                gc::ScopedGCCriticalSection gcs(self,
                                                gc::kGcCauseAddRemoveAppImageSpace,
                                                gc::kCollectorTypeAddRemoveAppImageSpace);
                ScopedSuspendAll ssa("Remove image space");
                runtime->GetHeap()->RemoveSpace(image_space.get());
              }
              // Non-fatal, don't update error_msg.
            }
          }
        }
      }
      if (!added_image_space) {
        DCHECK(dex_files.empty());

        if (oat_file->RequiresImage()) {
          LOG(WARNING) << "Loading "
                       << oat_file->GetLocation()
                       << " non-executable as it requires an image which we failed to load";
          // file as non-executable.
          OatFileAssistant nonexecutable_oat_file_assistant(dex_location,
                                                            kRuntimeISA,
                                                            context.get(),
                                                            /*load_executable=*/false,
                                                            only_use_system_oat_files_);
          oat_file.reset(nonexecutable_oat_file_assistant.GetBestOatFile().release());

          // The file could be deleted concurrently (for example background
          // dexopt, or secondary oat file being deleted by the app).
          if (oat_file == nullptr) {
            LOG(WARNING) << "Failed to reload oat file non-executable " << dex_location;
          }
        }

        if (oat_file != nullptr) {
          dex_files = oat_file_assistant.LoadDexFiles(*oat_file.get(), dex_location);

          // Register for tracking.
          for (const auto& dex_file : dex_files) {
            dex::tracking::RegisterDexFile(dex_file.get());
          }
        }
      }
      if (dex_files.empty()) {
        ScopedTrace failed_to_open_dex_files("FailedToOpenDexFilesFromOat");
        error_msgs->push_back("Failed to open dex files from " + odex_location);
      } else {
        // Opened dex files from an oat file, madvise them to their loaded state.
         for (const std::unique_ptr<const DexFile>& dex_file : dex_files) {
           OatDexFile::MadviseDexFileAtLoad(*dex_file);
         }
      }

      if (oat_file != nullptr) {
        VdexFile* vdex_file = oat_file->GetVdexFile();
        if (vdex_file != nullptr) {
          // Opened vdex file from an oat file, madvise it to its loaded state.
          // TODO(b/196052575): Unify dex and vdex madvise knobs and behavior.
          const size_t madvise_size_limit = Runtime::Current()->GetMadviseWillNeedSizeVdex();
          Runtime::MadviseFileForRange(madvise_size_limit,
                                       vdex_file->Size(),
                                       vdex_file->Begin(),
                                       vdex_file->End(),
                                       vdex_file->GetName());
        }

        VLOG(class_linker) << "Registering " << oat_file->GetLocation();
        *out_oat_file = RegisterOatFile(std::move(oat_file));
      }
    } else {
      // oat_file == nullptr
      // Verify if any of the dex files being loaded is already in the class path.
      // If so, report an error with the current stack trace.
      // Most likely the developer didn't intend to do this because it will waste
      // performance and memory.
      if (oat_file_assistant.GetBestStatus() == OatFileAssistant::kOatContextOutOfDate) {
        std::set<const DexFile*> already_exists_in_classpath =
            context->CheckForDuplicateDexFiles(MakeNonOwningPointerVector(dex_files));
        if (!already_exists_in_classpath.empty()) {
          ScopedTrace duplicate_dex_files("DuplicateDexFilesInContext");
          auto duplicate_it = already_exists_in_classpath.begin();
          std::string duplicates = (*duplicate_it)->GetLocation();
          for (duplicate_it++ ; duplicate_it != already_exists_in_classpath.end(); duplicate_it++) {
            duplicates += "," + (*duplicate_it)->GetLocation();
          }

          std::ostringstream out;
          out << "Trying to load dex files which is already loaded in the same ClassLoader "
              << "hierarchy.\n"
              << "This is a strong indication of bad ClassLoader construct which leads to poor "
              << "performance and wastes memory.\n"
              << "The list of duplicate dex files is: " << duplicates << "\n"
              << "The current class loader context is: "
              << context->EncodeContextForOatFile("") << "\n"
              << "Java stack trace:\n";

          {
            ScopedObjectAccess soa(self);
            self->DumpJavaStack(out);
          }

          // We log this as an ERROR to stress the fact that this is most likely unintended.
          // Note that ART cannot do anything about it. It is up to the app to fix their logic.
          // Here we are trying to give a heads up on why the app might have performance issues.
          LOG(ERROR) << out.str();
        }
      }
    }
  }

  // If we arrive here with an empty dex files list, it means we fail to load
  // it/them through an .oat file.
  if (dex_files.empty()) {
    std::string error_msg;
    static constexpr bool kVerifyChecksum = true;
    const ArtDexFileLoader dex_file_loader;
    if (!dex_file_loader.Open(dex_location,
                              dex_location,
                              Runtime::Current()->IsVerificationEnabled(),
                              kVerifyChecksum,
                              /*out*/ &error_msg,
                              &dex_files)) {
      ScopedTrace fail_to_open_dex_from_apk("FailedToOpenDexFilesFromApk");
      LOG(WARNING) << error_msg;
      error_msgs->push_back("Failed to open dex files from " + std::string(dex_location)
                            + " because: " + error_msg);
    }
  }

  if (Runtime::Current()->GetJit() != nullptr) {
    Runtime::Current()->GetJit()->RegisterDexFiles(dex_files, class_loader);
  }

  // Now that we loaded the dex/odex files, notify the runtime.
  // Note that we do this everytime we load dex files.
  Runtime::Current()->NotifyDexFileLoaded();

  return dex_files;
}
```



##  getOptimizationStatus



> 只是针对于GetBestInfo返回值做结果做了分叉和处理

``` c++
void OatFileAssistant::GetOptimizationStatus(
    std::string* out_odex_location,
    std::string* out_compilation_filter,
    std::string* out_compilation_reason,
    std::string* out_odex_status) {
  OatFileInfo& oat_file_info = GetBestInfo();
  const OatFile* oat_file = GetBestInfo().GetFile();

  if (oat_file == nullptr) {
    *out_odex_location = "error";
    *out_compilation_filter = "run-from-apk";
    *out_compilation_reason = "unknown";
    // This mostly happens when we cannot open the oat file.
    // Note that it's different than kOatCannotOpen.
    // TODO: The design of getting the BestInfo is not ideal,
    // as it's not very clear what's the difference between
    // a nullptr and kOatcannotOpen. The logic should be revised
    // and improved.
    *out_odex_status = "io-error-no-oat";
    return;
  }

  *out_odex_location = oat_file->GetLocation();
  OatStatus status = oat_file_info.Status();
  const char* reason = oat_file->GetCompilationReason();
  *out_compilation_reason = reason == nullptr ? "unknown" : reason;
  switch (status) {
    case kOatUpToDate:
      *out_compilation_filter = CompilerFilter::NameOfFilter(oat_file->GetCompilerFilter());
      *out_odex_status = "up-to-date";
      return;

    case kOatCannotOpen:  // This should never happen, but be robust.
      *out_compilation_filter = "error";
      *out_compilation_reason = "error";
      // This mostly happens when we cannot open the vdex file,
      // or the file is corrupt.
      *out_odex_status = "io-error-or-corruption";
      return;

    case kOatBootImageOutOfDate:
      *out_compilation_filter = "run-from-apk-fallback";
      *out_odex_status = "boot-image-more-recent";
      return;

    case kOatContextOutOfDate:
      *out_compilation_filter = "run-from-apk-fallback";
      *out_odex_status = "context-mismatch";
      return;

    case kOatDexOutOfDate:
      *out_compilation_filter = "run-from-apk-fallback";
      *out_odex_status = "apk-more-recent";
      return;
  }
  LOG(FATAL) << "Unreachable";
  UNREACHABLE();
}
```



## GetBestInfo



> 内部选择最优的文件。
>
> 其中
>
> oat > oat > vdex > dm

``` c++
OatFileAssistant::OatFileInfo& OatFileAssistant::GetBestInfo() {
  ScopedTrace trace("GetBestInfo");
  // TODO(calin): Document the side effects of class loading when
  // running dalvikvm command line.
  if (dex_parent_writable_ || UseFdToReadFiles()) {
    // If the parent of the dex file is writable it means that we can
    // create the odex file. In this case we unconditionally pick the odex
    // as the best oat file. This corresponds to the regular use case when
    // apps gets installed or when they load private, secondary dex file.
    // For apps on the system partition the odex location will not be
    // writable and thus the oat location might be more up to date.

    // If the odex is not useable, and we have a useable vdex, return the vdex
    // instead.
    if (!odex_.IsUseable()) {
      if (vdex_for_odex_.IsUseable()) {
        return vdex_for_odex_;
      } else if (dm_for_odex_.IsUseable()) {
        return dm_for_odex_;
      }
    }
    return odex_;
  }

  // We cannot write to the odex location. This must be a system app.

  // If the oat location is useable take it.
  if (oat_.IsUseable()) {
    return oat_;
  }

  // The oat file is not useable but the odex file might be up to date.
  // This is an indication that we are dealing with an up to date prebuilt
  // (that doesn't need relocation).
  if (odex_.IsUseable()) {
    return odex_;
  }

  // Look for a useable vdex file.
  if (vdex_for_oat_.IsUseable()) {
    return vdex_for_oat_;
  }
  if (vdex_for_odex_.IsUseable()) {
    return vdex_for_odex_;
  }
  if (dm_for_oat_.IsUseable()) {
    return dm_for_oat_;
  }
  if (dm_for_odex_.IsUseable()) {
    return dm_for_odex_;
  }

  // We got into the worst situation here:
  // - the oat location is not useable
  // - the prebuild odex location is not up to date
  // - the vdex-only file is not useable
  // - and we don't have the original dex file anymore (stripped).
  // Pick the odex if it exists, or the oat if not.
  return (odex_.Status() == kOatCannotOpen) ? oat_ : odex_;
}
```



> 其中oat, odex, vdex_for_oat, vdex_for_odex, dm_for_oat, dm_for_odex均是OatFileInfo
>
> 均是在OatFileAssistant实例创建的时候初始化

```c++
OatFileAssistant::OatFileAssistant(const char* dex_location,
                                   const InstructionSet isa,
                                   ClassLoaderContext* context,
                                   bool load_executable,
                                   bool only_load_trusted_executable,
                                   int vdex_fd,
                                   int oat_fd,
                                   int zip_fd)
    : context_(context),
      isa_(isa),
      load_executable_(load_executable),
      only_load_trusted_executable_(only_load_trusted_executable),
      odex_(this, /*is_oat_location=*/ false),
      oat_(this, /*is_oat_location=*/ true),
      vdex_for_odex_(this, /*is_oat_location=*/ false),
      vdex_for_oat_(this, /*is_oat_location=*/ true),
      dm_for_odex_(this, /*is_oat_location=*/ false),
      dm_for_oat_(this, /*is_oat_location=*/ true),
	  zip_fd_(zip_fd) 
```



> 其中OatFileInfo的初始化也特别简单

```c++
OatFileAssistant::OatFileInfo::OatFileInfo(OatFileAssistant* oat_file_assistant, // oat_file_assitant实例。
                                           bool is_oat_location // 是否是for oat（为oat服务）
                                          )
  : oat_file_assistant_(oat_file_assistant), is_oat_location_(is_oat_location)
{}
```



## OatFileAssistant::OatFileInfo::IsUseable

> 这个方法看样子也只是个中间方法，根据Status的状态指判断是否可用

```c++
bool OatFileAssistant::OatFileInfo::IsUseable() {
  ScopedTrace trace("IsUseable");
  switch (Status()) {
    case kOatCannotOpen:
    case kOatDexOutOfDate:
    case kOatContextOutOfDate:
    case kOatBootImageOutOfDate: return false;

    case kOatUpToDate: return true;
  }
  UNREACHABLE();
}
```



> 具体的状态指如下

```c++
  enum OatStatus {
    // kOatCannotOpen - The oat file cannot be opened, because it does not
    // exist, is unreadable, or otherwise corrupted.
    // 由于损坏、路径不可达、文件不存在等原因无法打开
    kOatCannotOpen,

    // kOatDexOutOfDate - The oat file is out of date with respect to the dex file.
    // oat文件过期（dex被修改，oat没有重新生成） 
    kOatDexOutOfDate,

    // kOatBootImageOutOfDate - The oat file is up to date with respect to the
    // dex file, but is out of date with respect to the boot image.
    // 设备升级系统后，启动映像更新（如 Android 版本升级、安全补丁更新）。
    // 但应用的 OAT 文件 未针对新启动映像重新编译
    kOatBootImageOutOfDate,

    // kOatContextOutOfDate - The context in the oat file is out of date with
    // respect to the class loader context.
    // ClassLoaderContext mismatch
    kOatContextOutOfDate,

    // kOatUpToDate - The oat file is completely up to date with respect to
    // the dex file and boot image.
    // oat文件可用
    kOatUpToDate,
  };
```



## OatFileAssistant::OatFileInfo::Status()



> open oatFile并获取当前文件的状态

```c++
OatFileAssistant::OatStatus OatFileAssistant::OatFileInfo::Status() {
  ScopedTrace trace("Status");
  if (!status_attempted_) {
    status_attempted_ = true;
      // 加载oat/vdex/...文件
    const OatFile* file = GetFile();
    if (file == nullptr) {
      status_ = kOatCannotOpen;
    } else {
        // 获取文件的状态
      status_ = oat_file_assistant_->GivenOatFileStatus(*file);
      VLOG(oat) << file->GetLocation() << " is " << status_
          << " with filter " << file->GetCompilerFilter();
    }
  }
  return status_;
}
```





## OatFileAssistant::GivenOatFileStatus()



> 用于获取oat的文件状态，其中有
>
> 1. checksum检查
>
>    a. 校验dex & vdex checksum是否一致
>
>    b. 校验bootimage checksum
>
> 2. ClassLoaderContext 检查

``` c++
OatFileAssistant::OatStatus OatFileAssistant::GivenOatFileStatus(const OatFile& file) {
  // Verify the ART_USE_READ_BARRIER state.
  // TODO: Don't fully reject files due to read barrier state. If they contain
  // compiled code and are otherwise okay, we should return something like
  // kOatRelocationOutOfDate. If they don't contain compiled code, the read
  // barrier state doesn't matter.
  // 1. 判断状态是否为ConcurrentCopying
  // 如果不满足则直接返回KOatCannotOpen
  const bool is_cc = file.GetOatHeader().IsConcurrentCopying();
  constexpr bool kRuntimeIsCC = kUseReadBarrier;
  if (is_cc != kRuntimeIsCC) {
    return kOatCannotOpen;
  }

  // Verify the dex checksum.
  // 2. 计算并对比dex & vdex checksum是否相等
  // 如果不满足直接返回KOatDexOutOfDate
  std::string error_msg;
  VdexFile* vdex = file.GetVdexFile();
  if (!DexChecksumUpToDate(*vdex, &error_msg)) {
    LOG(ERROR) << error_msg;
    return kOatDexOutOfDate;
  }

  CompilerFilter::Filter current_compiler_filter = file.GetCompilerFilter();

  // Verify the image checksum
  // 3. 计算bootimage checksum  
  if (file.IsBackedByVdexOnly()) { // 只有vdex文件，无oat文件
    VLOG(oat) << "Image checksum test skipped for vdex file " << file.GetLocation();
  } else if (CompilerFilter::DependsOnImageChecksum(current_compiler_filter)) { // 以来bootimage,进行bootimage检查
    if (!ValidateBootClassPathChecksums(file)) {
      VLOG(oat) << "Oat image checksum does not match image checksum.";
      return kOatBootImageOutOfDate;
    }
    if (!gc::space::ImageSpace::ValidateApexVersions(file, &error_msg)) {
      VLOG(oat) << error_msg;
      return kOatBootImageOutOfDate;
    }
  } else { // 不依赖bootimage，skip bootimage checksum检查
    VLOG(oat) << "Image checksum test skipped for compiler filter " << current_compiler_filter;
  }

  // zip_file_only_contains_uncompressed_dex_ is only set during fetching the dex checksums.
  // 4. 检测可执行文件是否可信任 
  DCHECK(required_dex_checksums_attempted_);
  if (only_load_trusted_executable_ &&
      !LocationIsTrusted(file.GetLocation(), !Runtime::Current()->DenyArtApexDataFiles()) &&
      file.ContainsDexCode() &&
      zip_file_only_contains_uncompressed_dex_) {
    LOG(ERROR) << "Not loading "
               << dex_location_
               << ": oat file has dex code, but APK has uncompressed dex code";
    return kOatDexOutOfDate;
  }

  // 5. check classloader Context是否一致。
  if (!ClassLoaderContextIsOkay(file)) {
    return kOatContextOutOfDate;
  }

  return kOatUpToDate;
}
```





### checksum





### ClassLoaderContext





# 其他





## oat文件加载







## art文件加载



> art的文件加载逻辑在AppImage:Loading的过程中完成。

![image-20250310001047954](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250310001047954.png)



### 基础概念



1. art文件是什么？

[官方文档](https://source.android.com/docs/core/runtime/configure#:~:text=apps%20work%20properly.-,How%20ART%20works,-ART%20uses%20ahead)

> 官网文档中对于art文件的描述 ：
>
> ART由两个部分组成，dex2oat工具（虚拟机编译器）、libart（虚拟机运行时）其中dex2oat在编译的过程中需要输入一个apk并生成一系列产物，其中vdex & odex必须生产的，art是可选的。odex包含了apk中方法的aot代码，vdex包含了一些用于加速验证的元信息以及可能会包含未被压缩的dex内容。art文件包含了一些art内部表示的**string & class信息**，**用于加速app启动**。
>
> ART comprises a compiler (the `dex2oat` tool) and a runtime (`libart.so`) that is loaded during boot. The `dex2oat` tool takes an APK file and generates one or more compilation artifact files that the runtime loads. The number of files, their extensions, and names are subject to change across releases, but as of the Android 8 release, these files are generated:
>
> `.vdex`: contains some additional metadata to speed up verification, sometimes along with the uncompressed DEX code of the APK.
>
> `.odex`: contains AOT-compiled code for methods in the APK.
>
> `.art (optional)` contains ART internal representations of some strings and classes listed in the APK, used to speed up app startup.
>
> 



[AppImage介绍](https://juejin.cn/post/7253611796111769637#heading-10:~:text=AppImage%E9%97%AE%E9%A2%98-,AppImage%E4%BB%8B%E7%BB%8D,-Android%207.0%E4%B9%8B%E5%89%8D)（AppImage其实就是art文件）



> 从上述的文档中我们能知道几个信息

a. art文件 == AppImage 只是叫法不同

b. art文件中包含了虚拟机运行时的string & class内存布局。

c. art主要是用于加速app启动过程。



2. art文件如何生成

> 其实这一点官方文档中已经给出了回答——通过dex2oat生成。
>
> 再结合大佬对于AppImage的介绍，不难得出 **art文件是由虚拟机在运行时收集热点代码通过dex2oat生成**。
>
> 可以用如下shell触发dex2oat

```shell
adb shell pm compile -r install -f 包名
```



### 代码分析



> OatFileManager::OpenDexFilesFromOat

``` c++

// 1. 尝试加载oat文件
// 内部判断是否使用 oat or jit，并打开具体文件（这里的文件可能是odex,vdex,dm）
// 将结果写入到下方的变量中。
std::string odex_location;
std::string compilation_filter;
std::string compilation_reason;
std::string odex_status;
oat_file_assistant.GetOptimizationStatus(
        &odex_location,
        &compilation_filter,
        &compilation_reason,
        &odex_status);

// 获取#1 中打开的文件对象
std::unique_ptr<const OatFile> oat_file(oat_file_assistant.GetBestOatFile().release());

// 如果#1 中选用类oat or jit文件（如果fallback到类dex file就不会命中这个分支）
if (oat_file != nullptr) {
	// 2. 从oat文件中获取dex file
    bool added_image_space = false;
    // 2.1. 尝试加载app image file(其实就是.art文件)
    if (oat_file->IsExecutable()) { // 判断oat文件是合法的。（可执行）
     	ScopedTrace app_image_timing("AppImage:Loading");
        
        // 2.1.1 open .art文件，并返回文件指针对象
        std::unique_ptr<gc::space::ImageSpace> image_space;
        if (ShouldLoadAppImage(oat_file.get())) {
          image_space = oat_file_assistant.OpenImageSpace(oat_file.get());
        }
        
        if (image_space != nullptr) {
         	ScopedObjectAccess soa(self);
            StackHandleScope<1> hs(self);
            // 通过java层 classloader对象获取native层的classloader对象
            // Note: 这两个对象不是同一个东西！！ java层创建对象的时候native会同步创建另外一个对象。这两对象是一一对应的。
            Handle<mirror::ClassLoader> h_loader(
              hs.NewHandle(soa.Decode<mirror::ClassLoader>(class_loader)));   
            if (h_loader != nullptr) { // 正常判空
  				// 2.1.2 将art文件加载进入堆内存
                {
                  ScopedThreadSuspension sts(self, ThreadState::kSuspended);
                  gc::ScopedGCCriticalSection gcs(self,
                                                  gc::kGcCauseAddRemoveAppImageSpace,
                                                  gc::kCollectorTypeAddRemoveAppImageSpace);
                  ScopedSuspendAll ssa("Add image space");
                  runtime->GetHeap()->AddSpace(image_space.get());
                }
                // 2.1.3 将art文件存入classloader table中 
                // Note：这一过程会加速类加载的速度
                // 类加载流程 ClassLoader(java) -> ClassLoader(c++) -> ClassTable(c++)
                
                {
                  ScopedTrace image_space_timing("Adding image space");
                  added_image_space = runtime->GetClassLinker()->AddImageSpace(image_space.get(),
                                                                               h_loader,
                                                                               /*out*/&dex_files,
                                                                               /*out*/&temp_error_msg);
                }
                
                // art file加载完成...
            }
        }
    }
    // 3. 如果＃2加载失败，则为oat文件加载完成剩余的部分 dexFiles
    if (!added_image_space) {
        // ......
        // 使用oat文件加载dex files
        if (oat_file != nullptr) {
          dex_files = oat_file_assistant.LoadDexFiles(*oat_file.get(), dex_location);

          // Register for tracking.
          for (const auto& dex_file : dex_files) {
            dex::tracking::RegisterDexFile(dex_file.get());
          }
        }
        
        // ......
    }
}
```



简单总结一下art文件的加载流程

1. 调用`oat_file_assistant.GetOptimizationStatus`尝试打开oat文件（文件对打开后会保存在oat_file_assistant内部变量中）
2. 在#1成功的基础上调用`oat_file_assistant.OpenImageSpace`尝试打开art文件, 并得到art文件句柄`image_space`
3. 在#2成功的基础上调用`runtime->GetHeap()->AddSpace(image_space)`将.art文件添加到虚拟机堆空间中
4. 在#3执行完成后调用`runtime->GetClassLinker()->AddImageSpace(image_space)`将.art文件添加到ClassLoader ClassTable中



## dex文件加载





