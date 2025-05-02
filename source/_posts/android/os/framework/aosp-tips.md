---
title: Aosp小tips
tags:
  - aosp
date: 2025-03-23 23:41:17
cover: https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/aosp-andvanced-development-tricks.png
---


# 关于本文



> 主要记录在AOSP调试学习过程中遇到的坑～

![aosp-andvanced-development-tricks](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/aosp-andvanced-development-tricks.png)



# Android 14_r73



## userdebug 编译out of space? out of inodes? the tree size XXX 



- 问题：

> out of space? out of inodes?XX
>
> 这个报错包含有3个原因。
>
> 1. 内存耗尽
> 2. inodes耗尽
> 3. 生成的文件超过文件系统的大小限制
>
> 
>
> ***这里的报错主要是因为#3导致，原因如下：***
>
> img文件生成目前主要有两种方式ex4和f2fs, 图中的报错主要是由于userdata默认采用了f2fs导致大小超过了f2fs的最大限制，需要***调整文件类型为ext4***并且将***文件的大小限制设置为合适的大小***

![Image_410772202840384](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/Image_410772202840384.jpg)

- 解决方式

https://juejin.cn/post/7484089347156295714



> note：
>
> 1. 报错原因有三个！！需要明确原因。不能盲目抄
> 2. 需要明确报错的文件，每个img类型都有不同的参数控制
>
> 3. 需要明确lunch type,不同的lunch type可能配置的参数不一样！！



## userdebug无法调试的问题





- 问题：

1.现象一

``` shell
# 当我们调用adb jdwp查看可调式的应用时会一直卡住。（）
➜  ~  adb jdwp
^C
```

2.现象二

> 设备能被识别，当时attach process找不到设备（右上角可见设备名称）

<img src="https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/Screenshot from 2025-03-23 23-23-12.png" alt="Screenshot from 2025-03-23 23-23-12" />



- 解决方案：

https://www.bilibili.com/opus/911162975264440345





## Winscope编译问题



问题描述：

>  依据README给出的构建流程最终会有一个报错

``` plaintText
➜  winscope git:(android-14.0.0_r73) ✗ npm run build:prod

> winscope@0.0.0 build:prod
> npm run build:trace_processor && npm run build:protos && rm -rf dist/prod/ && webpack --config webpack.config.prod.js --progress && cp deps_build/trace_processor/to_be_served/* src/adb/winscope_proxy.py dist/prod/


> winscope@0.0.0 build:trace_processor
> PERFETTO_TOP=../../../external/perfetto; (cd $PERFETTO_TOP && tools/install-build-deps --ui && ui/node ui/build.js --out trace_processor_build) && rm -rf deps_build/trace_processor && mkdir -p deps_build/trace_processor && rsync -ar $PERFETTO_TOP/trace_processor_build/ deps_build/trace_processor && mkdir deps_build/trace_processor/to_be_served && cp deps_build/trace_processor/ui/dist_version/engine_bundle.js deps_build/trace_processor/to_be_served/ && cp deps_build/trace_processor/wasm/trace_processor.wasm deps_build/trace_processor/to_be_served/

INFO:root:Running `pnpm install --shamefully-hoist --frozen-lockfile` in /Users/rose/aosp/android-14.0.0_r73/external/perfetto/ui
Lockfile is up to date, resolution step is skipped
Packages: +1
+
Progress: resolved 1, reused 0, downloaded 1, added 1, done
Done in 433ms
rm /Users/rose/aosp/android-14.0.0_r73/external/perfetto/trace_processor_build/ui
Entering /Users/rose/aosp/android-14.0.0_r73/external/perfetto/trace_processor_build
[00.487] 1/63   Waiting for first build to complete... 1 s
[00.487] 2/63   ../tools/gn gen --args=is_debug=false 
Done. Made 1083 targets from 226 files in 33ms
[00.596] 3/63   ../tools/ninja -C  trace_processor_wasm traceconv_wasm
ninja: Entering directory `/Users/rose/aosp/android-14.0.0_r73/external/perfetto/trace_processor_build'
ninja: no work to do.
[00.709] 4/63   cp wasm/trace_processor.wasm ui/dist/v42.0-7c79c3ab0/trace_processor.wasm
[00.714] 5/63   cp wasm/trace_processor.js ui/tsc/gen/trace_processor.js
[00.714] 6/63   cp wasm/trace_processor.d.ts ui/tsc/gen/trace_processor.d.ts
[00.714] 7/63   cp wasm/traceconv.wasm ui/dist/v42.0-7c79c3ab0/traceconv.wasm
[00.719] 8/63   cp wasm/traceconv.js ui/tsc/gen/traceconv.js
[00.720] 9/63   cp wasm/traceconv.d.ts ui/tsc/gen/traceconv.d.ts
[00.720] 10/63  cp ../ui/src/assets/brand.png ui/dist/v42.0-7c79c3ab0/assets/brand.png
[00.720] 11/63  sass --quiet ../ui/src/assets/perfetto.scss ui/dist/v42.0-7c79c3ab0/perfetto.css
[01.072] 12/63  cp ../ui/src/assets/favicon.png ui/dist/v42.0-7c79c3ab0/assets/favicon.png
[01.073] 13/63  cpHtml ../ui/src/assets/index.html index.html
[01.073] 14/63  cp ../ui/src/assets/logo-128.png ui/dist/v42.0-7c79c3ab0/assets/logo-128.png
[01.073] 15/63  cp ../ui/src/assets/logo-3d.png ui/dist/v42.0-7c79c3ab0/assets/logo-3d.png
[01.073] 16/63  cp ../ui/src/assets/rec_atrace.png ui/dist/v42.0-7c79c3ab0/assets/rec_atrace.png
[01.073] 17/63  cp ../ui/src/assets/rec_battery_counters.png ui/dist/v42.0-7c79c3ab0/assets/rec_
[01.074] 18/63  cp ../ui/src/assets/rec_board_voltage.png ui/dist/v42.0-7c79c3ab0/assets/rec_boa
[01.074] 19/63  cp ../ui/src/assets/rec_cpu_coarse.png ui/dist/v42.0-7c79c3ab0/assets/rec_cpu_co
[01.074] 20/63  cp ../ui/src/assets/rec_cpu_fine.png ui/dist/v42.0-7c79c3ab0/assets/rec_cpu_fine
[01.074] 21/63  cp ../ui/src/assets/rec_cpu_freq.png ui/dist/v42.0-7c79c3ab0/assets/rec_cpu_freq
[01.074] 22/63  cp ../ui/src/assets/rec_cpu_voltage.png ui/dist/v42.0-7c79c3ab0/assets/rec_cpu_v
[01.074] 23/63  cp ../ui/src/assets/rec_frame_timeline.png ui/dist/v42.0-7c79c3ab0/assets/rec_fr
[01.074] 24/63  cp ../ui/src/assets/rec_ftrace.png ui/dist/v42.0-7c79c3ab0/assets/rec_ftrace.png
[01.074] 25/63  cp ../ui/src/assets/rec_gpu_mem_total.png ui/dist/v42.0-7c79c3ab0/assets/rec_gpu
[01.074] 26/63  cp ../ui/src/assets/rec_java_heap_dump.png ui/dist/v42.0-7c79c3ab0/assets/rec_ja
[01.075] 27/63  cp ../ui/src/assets/rec_lmk.png ui/dist/v42.0-7c79c3ab0/assets/rec_lmk.png
[01.075] 28/63  cp ../ui/src/assets/rec_logcat.png ui/dist/v42.0-7c79c3ab0/assets/rec_logcat.png
[01.075] 29/63  cp ../ui/src/assets/rec_long_trace.png ui/dist/v42.0-7c79c3ab0/assets/rec_long_t
[01.075] 30/63  cp ../ui/src/assets/rec_mem_hifreq.png ui/dist/v42.0-7c79c3ab0/assets/rec_mem_hi
[01.075] 31/63  cp ../ui/src/assets/rec_meminfo.png ui/dist/v42.0-7c79c3ab0/assets/rec_meminfo.p
[01.075] 32/63  cp ../ui/src/assets/rec_native_heap_profiler.png ui/dist/v42.0-7c79c3ab0/assets/
[01.076] 33/63  cp ../ui/src/assets/rec_one_shot.png ui/dist/v42.0-7c79c3ab0/assets/rec_one_shot
[01.076] 34/63  cp ../ui/src/assets/rec_profiling.png ui/dist/v42.0-7c79c3ab0/assets/rec_profili
[01.076] 35/63  cp ../ui/src/assets/rec_ps_stats.png ui/dist/v42.0-7c79c3ab0/assets/rec_ps_stats
[01.076] 36/63  cp ../ui/src/assets/rec_ring_buf.png ui/dist/v42.0-7c79c3ab0/assets/rec_ring_buf
[01.076] 37/63  cp ../ui/src/assets/rec_syscalls.png ui/dist/v42.0-7c79c3ab0/assets/rec_syscalls
[01.076] 38/63  cp ../ui/src/assets/rec_vmstat.png ui/dist/v42.0-7c79c3ab0/assets/rec_vmstat.png
[01.076] 39/63  cp ../ui/src/assets/scheduling_latency.png ui/dist/v42.0-7c79c3ab0/assets/schedu
[01.077] 40/63  cp ../ui/src/assets/logo-128.png ui/chrome_extension/logo-128.png
[01.077] 41/63  cp ../ui/src/chrome_extension/manifest.json ui/chrome_extension/manifest.json
[01.077] 42/63  cp ../ui/src/test/diff_viewer/index.html ui-test-artifacts/index.html
[01.077] 43/63  cp ../ui/src/test/diff_viewer/script.js ui-test-artifacts/script.js
[01.077] 44/63  cp ../buildtools/typefaces/._MaterialSymbolsOutlined.woff2 ui/dist/v42.0-7c79c3a
[01.077] 45/63  cp ../buildtools/typefaces/MaterialSymbolsOutlined.woff2 ui/dist/v42.0-7c79c3ab0
[01.079] 46/63  cp ../buildtools/typefaces/Roboto-100.woff2 ui/dist/v42.0-7c79c3ab0/assets/Robot
[01.079] 47/63  cp ../buildtools/typefaces/Roboto-300.woff2 ui/dist/v42.0-7c79c3ab0/assets/Robot
[01.079] 48/63  cp ../buildtools/typefaces/Roboto-400.woff2 ui/dist/v42.0-7c79c3ab0/assets/Robot
[01.079] 49/63  cp ../buildtools/typefaces/Roboto-500.woff2 ui/dist/v42.0-7c79c3ab0/assets/Robot
[01.079] 50/63  cp ../buildtools/typefaces/RobotoCondensed-Light.woff2 ui/dist/v42.0-7c79c3ab0/a
[01.079] 51/63  cp ../buildtools/typefaces/RobotoCondensed-Regular.woff2 ui/dist/v42.0-7c79c3ab0
[01.079] 52/63  cp ../buildtools/typefaces/RobotoMono-Regular.woff2 ui/dist/v42.0-7c79c3ab0/asse
[01.079] 53/63  cp ../buildtools/catapult_trace_viewer/catapult_trace_viewer.html ui/dist/v42.0-
[01.080] 54/63  cp ../buildtools/catapult_trace_viewer/catapult_trace_viewer.js ui/dist/v42.0-7c
[01.081] 55/63  python3 ../tools/gen_ui_imports ../ui/src/tracks --out ../ui/src/gen/all_tracks.
[01.157] 56/63  python3 ../tools/gen_ui_imports ../ui/src/plugins --out ../ui/src/gen/all_plugin
[01.231] 57/63  pbjs --no-beautify --force-number --no-delimited --no-verify -t static-module -w
[02.366] 58/63  pbts --no-comments -p .. -o ui/tsc/gen/protos.d.ts ui/tsc/gen/protos.js
[16.294] 59/63  python3 ../tools/write_version_header.py --ts_out ui/tsc/gen/perfetto_version.ts
[16.388] 60/63  tsc --project ../ui
[23.376] 61/63  tsc --project ../ui/src/service_worker
[23.896] 62/63  rollup -c ../ui/config/rollup.config.js --no-indent --silent
[50.313] 63/63  makeManifest

> winscope@0.0.0 build:protos
> node protos/build.js

node:internal/process/promises:279
            triggerUncaughtException(err, true /* fromPromise */);
            ^

[UnhandledPromiseRejection: This error originated either by throwing inside of an async function without a catch block, or by rejecting a promise which was not handled with .catch(). The promise rejected with the reason "Failed to execute command

command: npx pbjs --force-long --target json-module --wrap es6 --out /Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/protos/../deps_build/protos/windowmanager/latest/json.js --root windowmanager_latest --path /Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/protos/../../../../../external/perfetto --path /Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/protos/.. --path /Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/protos/../../../../.. /Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/protos/../../../../frameworks/base/core/proto/android/server/windowmanagertrace.proto

stdout: 

stderr: /Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/node_modules/protobufjs-cli/pbjs.js:254
            throw err;
            ^

Error: no such Type or Enum 'WindowManagerServiceDumpProto' in Type .com.android.server.wm.WindowManagerTraceProto
    at Type.lookupTypeOrEnum (/Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/node_modules/protobufjs/src/namespace.js:410:15)
    at Field.resolve (/Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/node_modules/protobufjs/src/field.js:268:94)
    at Type.set (/Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/node_modules/protobufjs/src/type.js:177:38)
    at Type.get (/Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/node_modules/protobufjs/src/type.js:155:45)
    at Field.resolve (/Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/node_modules/protobufjs/src/field.js:320:21)
    at Type.resolveAll (/Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/node_modules/protobufjs/src/type.js:304:21)
    at Namespace.resolveAll (/Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/node_modules/protobufjs/src/namespace.js:307:25)
    at Namespace.resolveAll (/Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/node_modules/protobufjs/src/namespace.js:307:25)
    at Namespace.resolveAll (/Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/node_modules/protobufjs/src/namespace.js:307:25)
    at Namespace.resolveAll (/Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/node_modules/protobufjs/src/namespace.js:307:25)
".] {
  code: 'ERR_UNHANDLED_REJECTION'
```



> 解决方法：
>
> https://blog.csdn.net/fromair/article/details/145961344





# 通用问题



## 修改frameworks无法开机的问题



> Updated:
>
> 2025.5.2



> 系统：Andriod 11_r46
>
> 设备：Pixel 3XL



总结：

> 问题是由于dex文件和oat文件不对应导致。
>
> 如果有更新jar,dex的需要，一定一定要刷入对应的oat文件(.art, .oat, .vdex文件)



背景：

> 修改了framework.jar中部分类的源代码，将framework.jar刷入到system下
>
> 刷入后冷启动发现有如下报错
>
> 简单来说就是Pending exception java.lang.ClassNotFoundException: com.android.internal.os.RuntimeInit

``` plaintText
 Fatal signal 6 (SIGABRT), code -1 (SI_QUEUE) in tid 3489 (main), pid 3489 (main)
2025-05-02 12:51:36.966  3490-3490  zygote                  app_process32                        A  runtime.cc:655] Runtime aborting...
                                                                                                    runtime.cc:655] Dumping all threads without mutator lock held
                                                                                                    runtime.cc:655] All threads:
                                                                                                    runtime.cc:655] DALVIK THREADS (6):
                                                                                                    runtime.cc:655] "Jit thread pool worker thread 0" prio=5 tid=2 Runnable
                                                                                                    runtime.cc:655]   | group="" sCount=0 dsCount=0 flags=0 obj=0x12c40030 self=0xebd05410
                                                                                                    runtime.cc:655]   | sysTid=3509 nice=9 cgrp=default sched=0/0 handle=0xdfd1cd60
                                                                                                    runtime.cc:655]   | state=R schedstat=( 17630782 74584 11 ) utm=1 stm=0 core=2 HZ=100
                                                                                                    runtime.cc:655]   | stack=0xdfc1e000-0xdfc20000 stackSize=1023KB
                                                                                                    runtime.cc:655]   | held mutexes= "mutator lock"(shared held)
                                                                                                    runtime.cc:655]   native: #00 pc 0037519d  /apex/com.android.art/lib/libart.so (art::DumpNativeStack(std::__1::basic_ostream<char, std::__1::char_traits<char> >&, int, BacktraceMap*, char const*, art::ArtMethod*, void*, bool)+76)
                                                                                                    runtime.cc:655]   native: #01 pc 00444d73  /apex/com.android.art/lib/libart.so (art::Thread::DumpStack(std::__1::basic_ostream<char, std::__1::char_traits<char> >&, bool, BacktraceMap*, bool) const+386)
                                                                                                    runtime.cc:655]   native: #02 pc 0044066b  /apex/com.android.art/lib/libart.so (art::Thread::Dump(std::__1::basic_ostream<char, std::__1::char_traits<char> >&, bool, BacktraceMap*, bool) const+34)
                                                                                                    runtime.cc:655]   native: #03 pc 0045d5ff  /apex/com.android.art/lib/libart.so (art::DumpCheckpoint::Run(art::Thread*)+674)
                                                                                                    runtime.cc:655]   native: #04 pc 00445981  /apex/com.android.art/lib/libart.so (art::Thread::RunCheckpointFunction()+124)
                                                                                                    runtime.cc:655]   native: #05 pc 0015c8d5  /apex/com.android.art/lib/libart.so (art::ClassLinker::LinkFields(art::Thread*, art::Handle<art::mirror::Class>, bool, unsigned int*)+5520)
                                                                                                    runtime.cc:655]   native: #06 pc 0015579f  /apex/com.android.art/lib/libart.so (art::ClassLinker::LinkInstanceFields(art::Thread*, art::Handle<art::mirror::Class>)+38)
                                                                                                    runtime.cc:655]   native: #07 pc 0014dced  /apex/com.android.art/lib/libart.so (art::ClassLinker::LinkClass(art::Thread*, char const*, art::Handle<art::mirror::Class>, art::Handle<art::mirror::ObjectArray<art::mirror::Class> >, art::MutableHandle<art::mirror::Class>*)+252)
                                                                                                    runtime.cc:655]   native: #08 pc 0014ad3d  /apex/com.android.art/lib/libart.so (art::ClassLinker::DefineClass(art::Thread*, char const*, unsigned int, art::Handle<art::mirror::ClassLoader>, art::DexFile const&, art::dex::ClassDef const&)+876)
                                                                                                    runtime.cc:655]   native: #09 pc 0014b90f  /apex/com.android.art/lib/libart.so (art::ClassLinker::FindClass(art::Thread*, char const*, art::Handle<art::mirror::ClassLoader>)+1814)
                                                                                                    runtime.cc:655]   native: #10 pc 0013cb57  /apex/com.android.art/lib/libart.so (art::ClassLinker::DoResolveType(art::dex::TypeIndex, art::Handle<art::mirror::DexCache>, art::Handle<art::mirror::ClassLoader>)+122)
                                                                                                    runtime.cc:655]   native: #11 pc 00470277  /apex/com.android.art/lib/libart.so (art::verifier::RegType const& art::verifier::impl::(anonymous namespace)::MethodVerifier<false>::ResolveClass<(art::verifier::impl::(anonymous namespace)::CheckAccess)2>(art::dex::TypeIndex)+106)
                                                                                                    runtime.cc:655]   native: #12 pc 0047be71  /apex/com.android.art/lib/libart.so (art::verifier::impl::(anonymous namespace)::MethodVerifier<false>::VerifyInvocationArgs(art::Instruction const*, art::verifier::MethodType, bool)+60)
                                                                                                    runtime.cc:655]   native: #13 pc 004812d7  /apex/com.android.art/lib/libart.so (art::verifier::impl::(anonymous namespace)::MethodVerifier<false>::CodeFlowVerifyInstruction(unsigned int*)+3290)
                                                                                                    runtime.cc:655]   native: #14 pc 0046ca23  /apex/com.android.art/lib/libart.so (_ZN3art8verifier4impl12_GLOBAL__N_114MethodVerifierILb0EE6VerifyEv$56e57d29407519c2e5b6f0fe4b92f70f+5294)
2025-05-02 12:51:36.966  3490-3490  zygote                  app_process32                        A  runtime.cc:655]   native: #15 pc 0046ab8b  /apex/com.android.art/lib/libart.so (art::verifier::MethodVerifier::FailureData art::verifier::MethodVerifier::VerifyMethod<false>(art::Thread*, art::ClassLinker*, art::ArenaPool*, unsigned int, art::DexFile const*, art::Handle<art::mirror::DexCache>, art::Handle<art::mirror::ClassLoader>, art::dex::ClassDef const&, art::dex::CodeItem const*, art::ArtMethod*, unsigned int, art::CompilerCallbacks*, art::verifier::VerifierCallback*, bool, art::verifier::HardFailLogMode, bool, unsigned int, bool, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >*)+170)
                                                                                                    runtime.cc:655]   native: #16 pc 0046a049  /apex/com.android.art/lib/libart.so (art::verifier::MethodVerifier::VerifyMethod(art::Thread*, art::ClassLinker*, art::ArenaPool*, unsigned int, art::DexFile const*, art::Handle<art::mirror::DexCache>, art::Handle<art::mirror::ClassLoader>, art::dex::ClassDef const&, art::dex::CodeItem const*, art::ArtMethod*, unsigned int, art::CompilerCallbacks*, art::verifier::VerifierCallback*, bool, art::verifier::HardFailLogMode, bool, unsigned int, bool, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >*)+80)
                                                                                                    runtime.cc:655]   native: #17 pc 0046956f  /apex/com.android.art/lib/libart.so (art::verifier::ClassVerifier::VerifyClass(art::Thread*, art::DexFile const*, art::Handle<art::mirror::DexCache>, art::Handle<art::mirror::ClassLoader>, art::dex::ClassDef const&, art::CompilerCallbacks*, art::verifier::VerifierCallback*, bool, art::verifier::HardFailLogMode, unsigned int, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >*)+1022)
                                                                                                    runtime.cc:655]   native: #18 pc 00468f31  /apex/com.android.art/lib/libart.so (art::verifier::ClassVerifier::CommonVerifyClass(art::Thread*, art::ObjPtr<art::mirror::Class>, art::CompilerCallbacks*, art::verifier::VerifierCallback*, bool, art::verifier::HardFailLogMode, unsigned int, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >*)+344)
                                                                                                    runtime.cc:655]   native: #19 pc 00469155  /apex/com.android.art/lib/libart.so (art::verifier::ClassVerifier::VerifyClass(art::Thread*, art::ObjPtr<art::mirror::Class>, art::CompilerCallbacks*, bool, art::verifier::HardFailLogMode, unsigned int, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >*)+56)
                                                                                                    runtime.cc:655]   native: #20 pc 00151487  /apex/com.android.art/lib/libart.so (art::ClassLinker::PerformClassVerification(art::Thread*, art::Handle<art::mirror::Class>, art::verifier::HardFailLogMode, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >*)+66)
                                                                                                    runtime.cc:655]   native: #21 pc 00150887  /apex/com.android.art/lib/libart.so (art::ClassLinker::VerifyClass(art::Thread*, art::Handle<art::mirror::Class>, art::verifier::HardFailLogMode)+1018)
                                                                                                    runtime.cc:655]   native: #22 pc 00262e2f  /apex/com.android.art/lib/libart.so (art::jit::ZygoteVerificationTask::Run(art::Thread*)+762)
                                                                                                    runtime.cc:655]   native: #23 pc 0045e213  /apex/com.android.art/lib/libart.so (art::ThreadPoolWorker::Run()+54)
                                                                                                    runtime.cc:655]   native: #24 pc 0045de79  /apex/com.android.art/lib/libart.so (art::ThreadPoolWorker::Callback(void*)+116)
                                                                                                    runtime.cc:655]   native: #25 pc 00080a9f  /apex/com.android.runtime/lib/bionic/libc.so (__pthread_start(void*)+40)
                                                                                                    runtime.cc:655]   native: #26 pc 00039dc5  /apex/com.android.runtime/lib/bionic/libc.so (__start_thread+30)
                                                                                                    runtime.cc:655]   (no managed stack frames)
                                                                                                    runtime.cc:655] 
                                                                                                    runtime.cc:655] "main" prio=10 tid=1 Runnable
                                                                                                    runtime.cc:655]   | group="" sCount=0 dsCount=0 flags=0 obj=0x12c02e18 self=0xebd04610
                                                                                                    runtime.cc:655]   | sysTid=3490 nice=-20 cgrp=default sched=0/0 handle=0xf77b3470
                                                                                                    runtime.cc:655]   | state=R schedstat=( 216038540 8189061 35 ) utm=14 stm=6 core=1 HZ=100
                                                                                                    runtime.cc:655]   | stack=0xff5ef000-0xff5f1000 stackSize=8192KB
                                                                                                    runtime.cc:655]   | held mutexes= "abort lock" "mutator lock"(shared held)
2025-05-02 12:51:36.966  3490-3490  zygote                  app_process32                        A  runtime.cc:655]   native: #00 pc 0037519d  /apex/com.android.art/lib/libart.so (art::DumpNativeStack(std::__1::basic_ostream<char, std::__1::char_traits<char> >&, int, BacktraceMap*, char const*, art::ArtMethod*, void*, bool)+76)
                                                                                                    runtime.cc:655]   native: #01 pc 00444d73  /apex/com.android.art/lib/libart.so (art::Thread::DumpStack(std::__1::basic_ostream<char, std::__1::char_traits<char> >&, bool, BacktraceMap*, bool) const+386)
                                                                                                    runtime.cc:655]   native: #02 pc 0044066b  /apex/com.android.art/lib/libart.so (art::Thread::Dump(std::__1::basic_ostream<char, std::__1::char_traits<char> >&, bool, BacktraceMap*, bool) const+34)
                                                                                                    runtime.cc:655]   native: #03 pc 0045d5ff  /apex/com.android.art/lib/libart.so (art::DumpCheckpoint::Run(art::Thread*)+674)
                                                                                                    runtime.cc:655]   native: #04 pc 00458b7b  /apex/com.android.art/lib/libart.so (art::ThreadList::RunCheckpoint(art::Closure*, art::Closure*)+354)
                                                                                                    runtime.cc:655]   native: #05 pc 004580a5  /apex/com.android.art/lib/libart.so (art::ThreadList::Dump(std::__1::basic_ostream<char, std::__1::char_traits<char> >&, bool)+1496)
                                                                                                    runtime.cc:655]   native: #06 pc 00403389  /apex/com.android.art/lib/libart.so (art::Runtime::Abort(char const*)+1444)
                                                                                                    runtime.cc:655]   native: #07 pc 0000d993  /system/lib/libbase.so (android::base::SetAborter(std::__1::function<void (char const*)>&&)::$_3::__invoke(char const*)+46)
                                                                                                    runtime.cc:655]   native: #08 pc 000052ef  /system/lib/liblog.so (__android_log_assert+174)
                                                                                                    runtime.cc:655]   native: #09 pc 00002dad  /apex/com.android.art/lib/libnativehelper.so (jniRegisterNativeMethods+80)
                                                                                                    runtime.cc:655]   native: #10 pc 00068da9  /system/lib/libandroid_runtime.so (android::register_com_android_internal_os_RuntimeInit(_JNIEnv*)+44)
                                                                                                    runtime.cc:655]   native: #11 pc 0006c609  /system/lib/libandroid_runtime.so (android::AndroidRuntime::startReg(_JNIEnv*)+44)
                                                                                                    runtime.cc:655]   native: #12 pc 0006c3bb  /system/lib/libandroid_runtime.so (android::AndroidRuntime::start(char const*, android::Vector<android::String8> const&, bool)+402)
                                                                                                    runtime.cc:655]   native: #13 pc 00002e19  /system/bin/app_process32 (main+992)
                                                                                                    runtime.cc:655]   native: #14 pc 00033157  /apex/com.android.runtime/lib/bionic/libc.so (__libc_init+66)
                                                                                                    runtime.cc:655]   (no managed stack frames)
                                                                                                    runtime.cc:655] 
                                                                                                    runtime.cc:655] "FinalizerDaemon" prio=10 tid=3 Waiting
                                                                                                    runtime.cc:655]   | group="" sCount=1 dsCount=0 flags=1 obj=0x12c03c60 self=0xebd03810
                                                                                                    runtime.cc:655]   | sysTid=3512 nice=-8 cgrp=default sched=0/0 handle=0xdfa061c0
                                                                                                    runtime.cc:655]   | state=S schedstat=( 682449 16250 7 ) utm=0 stm=0 core=0 HZ=100
                                                                                                    runtime.cc:655]   | stack=0xdf903000-0xdf905000 stackSize=1040KB
                                                                                                    runtime.cc:655]   | held mutexes=
                                                                                                    runtime.cc:655]   native: #00 pc 00034044  /apex/com.android.runtime/lib/bionic/libc.so (syscall+28)
                                                                                                    runtime.cc:655]   native: #01 pc 00131b59  /apex/com.android.art/lib/libart.so (art::ConditionVariable::WaitHoldingLocks(art::Thread*)+80)
                                                                                                    runtime.cc:655]   native: #02 pc 00371775  /apex/com.android.art/lib/libart.so (art::Monitor::Wait(art::Thread*, long long, int, bool, art::ThreadState)+480)
                                                                                                    runtime.cc:655]   native: #03 pc 00372a27  /apex/com.android.art/lib/libart.so (art::Monitor::Wait(art::Thread*, art::ObjPtr<art::mirror::Object>, long long, int, bool, art::ThreadState)+174)
                                                                                                    runtime.cc:655]   native: #04 pc 0038cd29  /apex/com.android.art/lib/libart.so (art::Object_waitJI(_JNIEnv*, _jobject*, long long, int)+36)
                                                                                                    runtime.cc:655]   at java.lang.Object.wait(Native method)
                                                                                                    runtime.cc:655]   - waiting on <0x02f2b809> (a java.lang.Object)
                                                                                                    runtime.cc:655]   at java.lang.Object.wait(Object.java:442)
                                                                                                    runtime.cc:655]   at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:190)
                                                                                                    runtime.cc:655]   - locked <0x02f2b809> (a java.lang.Object)
                                                                                                    runtime.cc:655]   at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:211)
                                                                                                    runtime.cc:655]   at java.lang.Daemons$FinalizerDaemon.runInternal(Daemons.java:273)
                                                                                                    runtime.cc:655]   at java.lang.Daemons$Daemon.run(Daemons.java:139)
                                                                                                    runtime.cc:655]   at java.lang.Thread.run(Thread.java:923)
2025-05-02 12:51:36.966  3490-3490  zygote                  app_process32                        A  runtime.cc:655] 
                                                                                                    runtime.cc:655] "HeapTaskDaemon" prio=10 tid=4 WaitingForTaskProcessor
                                                                                                    runtime.cc:655]   | group="" sCount=1 dsCount=0 flags=1 obj=0x12c03b40 self=0xebd00010
                                                                                                    runtime.cc:655]   | sysTid=3510 nice=-8 cgrp=default sched=0/0 handle=0xdfc181c0
                                                                                                    runtime.cc:655]   | state=S schedstat=( 343333 156616 5 ) utm=0 stm=0 core=2 HZ=100
                                                                                                    runtime.cc:655]   | stack=0xdfb15000-0xdfb17000 stackSize=1040KB
                                                                                                    runtime.cc:655]   | held mutexes=
                                                                                                    runtime.cc:655]   native: #00 pc 00034044  /apex/com.android.runtime/lib/bionic/libc.so (syscall+28)
                                                                                                    runtime.cc:655]   native: #01 pc 00131b59  /apex/com.android.art/lib/libart.so (art::ConditionVariable::WaitHoldingLocks(art::Thread*)+80)
                                                                                                    runtime.cc:655]   native: #02 pc 0021a3ad  /apex/com.android.art/lib/libart.so (art::gc::TaskProcessor::GetTask(art::Thread*)+316)
                                                                                                    runtime.cc:655]   native: #03 pc 0021aaa5  /apex/com.android.art/lib/libart.so (art::gc::TaskProcessor::RunAllTasks(art::Thread*)+48)
                                                                                                    runtime.cc:655]   at dalvik.system.VMRuntime.runHeapTasks(Native method)
                                                                                                    runtime.cc:655]   at java.lang.Daemons$HeapTaskDaemon.runInternal(Daemons.java:531)
                                                                                                    runtime.cc:655]   at java.lang.Daemons$Daemon.run(Daemons.java:139)
                                                                                                    runtime.cc:655]   at java.lang.Thread.run(Thread.java:923)
                                                                                                    runtime.cc:655] 
                                                                                                    runtime.cc:655] "ReferenceQueueDaemon" prio=10 tid=5 Waiting
                                                                                                    runtime.cc:655]   | group="" sCount=1 dsCount=0 flags=1 obj=0x12c03bd0 self=0xebd00e10
                                                                                                    runtime.cc:655]   | sysTid=3511 nice=-8 cgrp=default sched=0/0 handle=0xdfb0f1c0
                                                                                                    runtime.cc:655]   | state=S schedstat=( 515053 2292 9 ) utm=0 stm=0 core=2 HZ=100
                                                                                                    runtime.cc:655]   | stack=0xdfa0c000-0xdfa0e000 stackSize=1040KB
                                                                                                    runtime.cc:655]   | held mutexes=
                                                                                                    runtime.cc:655]   native: #00 pc 00034044  /apex/com.android.runtime/lib/bionic/libc.so (syscall+28)
                                                                                                    runtime.cc:655]   native: #01 pc 00131b59  /apex/com.android.art/lib/libart.so (art::ConditionVariable::WaitHoldingLocks(art::Thread*)+80)
                                                                                                    runtime.cc:655]   native: #02 pc 00371775  /apex/com.android.art/lib/libart.so (art::Monitor::Wait(art::Thread*, long long, int, bool, art::ThreadState)+480)
                                                                                                    runtime.cc:655]   native: #03 pc 00372a27  /apex/com.android.art/lib/libart.so (art::Monitor::Wait(art::Thread*, art::ObjPtr<art::mirror::Object>, long long, int, bool, art::ThreadState)+174)
                                                                                                    runtime.cc:655]   native: #04 pc 0038cd29  /apex/com.android.art/lib/libart.so (art::Object_waitJI(_JNIEnv*, _jobject*, long long, int)+36)
                                                                                                    runtime.cc:655]   at java.lang.Object.wait(Native method)
                                                                                                    runtime.cc:655]   - waiting on <0x05614a0e> (a java.lang.Class<java.lang.ref.ReferenceQueue>)
                                                                                                    runtime.cc:655]   at java.lang.Object.wait(Object.java:442)
                                                                                                    runtime.cc:655]   at java.lang.Object.wait(Object.java:568)
                                                                                                    runtime.cc:655]   at java.lang.Daemons$ReferenceQueueDaemon.runInternal(Daemons.java:217)
                                                                                                    runtime.cc:655]   - locked <0x05614a0e> (a java.lang.Class<java.lang.ref.ReferenceQueue>)
                                                                                                    runtime.cc:655]   at java.lang.Daemons$Daemon.run(Daemons.java:139)
                                                                                                    runtime.cc:655]   at java.lang.Thread.run(Thread.java:923)
                                                                                                    runtime.cc:655] 
                                                                                                    runtime.cc:655] "FinalizerWatchdogDaemon" prio=10 tid=6 Waiting
                                                                                                    runtime.cc:655]   | group="" sCount=1 dsCount=0 flags=1 obj=0x12c03cf0 self=0xebd06210
                                                                                                    runtime.cc:655]   | sysTid=3513 nice=-8 cgrp=default sched=0/0 handle=0xdf8fd1c0
                                                                                                    runtime.cc:655]   | state=S schedstat=( 321353 63698 4 ) utm=0 stm=0 core=0 HZ=100
                                                                                                    runtime.cc:655]   | stack=0xdf7fa000-0xdf7fc000 stackSize=1040KB
                                                                                                    runtime.cc:655]   | held mutexes=
                                                                                                    runtime.cc:655]   native: #00 pc 00034044  /apex/com.android.runtime/lib/bionic/libc.so (syscall+28)
                                                                                                    runtime.cc:655]   native: #01 pc 00131b59  /apex/com.android.art/lib/libart.so (art::ConditionVariable::WaitHoldingLocks(art::Thread*)+80)
                                                                                                    runtime.cc:655]   native: #02 pc 00371775  /apex/com.android.art/lib/libart.so (art::Monitor::Wait(art::Thread*, long long, int, bool, art::ThreadState)+480)
                                                                                                    runtime.cc:655]   native: #03 pc 00372a27  /apex/com.android.art/lib/libart.so (art::Monitor::Wait(art::Thread*, art::ObjPtr<art::mirror::Object>, long long, int, bool, art::ThreadState)+174)
2025-05-02 12:51:36.966  3490-3490  zygote                  app_process32                        A  runtime.cc:655]   native: #04 pc 0038cd29  /apex/com.android.art/lib/libart.so (art::Object_waitJI(_JNIEnv*, _jobject*, long long, int)+36)
                                                                                                    runtime.cc:655]   at java.lang.Object.wait(Native method)
                                                                                                    runtime.cc:655]   - waiting on <0x03d0fc2f> (a java.lang.Daemons$FinalizerWatchdogDaemon)
                                                                                                    runtime.cc:655]   at java.lang.Object.wait(Object.java:442)
                                                                                                    runtime.cc:655]   at java.lang.Object.wait(Object.java:568)
                                                                                                    runtime.cc:655]   at java.lang.Daemons$FinalizerWatchdogDaemon.sleepUntilNeeded(Daemons.java:341)
                                                                                                    runtime.cc:655]   - locked <0x03d0fc2f> (a java.lang.Daemons$FinalizerWatchdogDaemon)
                                                                                                    runtime.cc:655]   at java.lang.Daemons$FinalizerWatchdogDaemon.runInternal(Daemons.java:321)
                                                                                                    runtime.cc:655]   at java.lang.Daemons$Daemon.run(Daemons.java:139)
                                                                                                    runtime.cc:655]   at java.lang.Thread.run(Thread.java:923)
                                                                                                    runtime.cc:655] 
                                                                                                    runtime.cc:655] Aborting thread:
                                                                                                    runtime.cc:655] "main" prio=10 tid=1 Native
                                                                                                    runtime.cc:655]   | group="" sCount=0 dsCount=0 flags=0 obj=0x12c02e18 self=0xebd04610
                                                                                                    runtime.cc:655]   | sysTid=3490 nice=-20 cgrp=default sched=0/0 handle=0xf77b3470
                                                                                                    runtime.cc:655]   | state=R schedstat=( 255806824 8317551 78 ) utm=16 stm=8 core=7 HZ=100
                                                                                                    runtime.cc:655]   | stack=0xff5ef000-0xff5f1000 stackSize=8192KB
                                                                                                    runtime.cc:655]   | held mutexes= "abort lock"
                                                                                                    runtime.cc:655]   native: #00 pc 0037519d  /apex/com.android.art/lib/libart.so (art::DumpNativeStack(std::__1::basic_ostream<char, std::__1::char_traits<char> >&, int, BacktraceMap*, char const*, art::ArtMethod*, void*, bool)+76)
                                                                                                    runtime.cc:655]   native: #01 pc 00444d73  /apex/com.android.art/lib/libart.so (art::Thread::DumpStack(std::__1::basic_ostream<char, std::__1::char_traits<char> >&, bool, BacktraceMap*, bool) const+386)
                                                                                                    runtime.cc:655]   native: #02 pc 0044066b  /apex/com.android.art/lib/libart.so (art::Thread::Dump(std::__1::basic_ostream<char, std::__1::char_traits<char> >&, bool, BacktraceMap*, bool) const+34)
                                                                                                    runtime.cc:655]   native: #03 pc 004129bf  /apex/com.android.art/lib/libart.so (art::AbortState::DumpThread(std::__1::basic_ostream<char, std::__1::char_traits<char> >&, art::Thread*) const+30)
                                                                                                    runtime.cc:655]   native: #04 pc 00403539  /apex/com.android.art/lib/libart.so (art::Runtime::Abort(char const*)+1876)
                                                                                                    runtime.cc:655]   native: #05 pc 0000d993  /system/lib/libbase.so (android::base::SetAborter(std::__1::function<void (char const*)>&&)::$_3::__invoke(char const*)+46)
                                                                                                    runtime.cc:655]   native: #06 pc 000052ef  /system/lib/liblog.so (__android_log_assert+174)
                                                                                                    runtime.cc:655]   native: #07 pc 00002dad  /apex/com.android.art/lib/libnativehelper.so (jniRegisterNativeMethods+80)
                                                                                                    runtime.cc:655]   native: #08 pc 00068da9  /system/lib/libandroid_runtime.so (android::register_com_android_internal_os_RuntimeInit(_JNIEnv*)+44)
                                                                                                    runtime.cc:655]   native: #09 pc 0006c609  /system/lib/libandroid_runtime.so (android::AndroidRuntime::startReg(_JNIEnv*)+44)
                                                                                                    runtime.cc:655]   native: #10 pc 0006c3bb  /system/lib/libandroid_runtime.so (android::AndroidRuntime::start(char const*, android::Vector<android::String8> const&, bool)+402)
                                                                                                    runtime.cc:655]   native: #11 pc 00002e19  /system/bin/app_process32 (main+992)
                                                                                                    runtime.cc:655]   native: #12 pc 00033157  /apex/com.android.runtime/lib/bionic/libc.so (__libc_init+66)
                                                                                                    runtime.cc:655]   (no managed stack frames)
                                                                                                    runtime.cc:655] Pending exception java.lang.ClassNotFoundException: com.android.internal.os.RuntimeInit
                                                                                                    runtime.cc:655] (Throwable with empty stack trace)
                                                                                                    runtime.cc:655] 
```



> 经过排查发现可能是oat文件没有更新导致，oat文件与dex文件不对应导致ClassNotFound.



> 解决方案。编译后刷入framework.jar以及其对应的oat文件

``` shell
➜  framework adb push arm/boot-framework.* /system/framework/arm/
arm/boot-framework.art: 1 file pushed, 0 skipped. 312.8 MB/s (3571712 bytes in 0.011s)
arm/boot-framework.oat: 1 file pushed, 0 skipped. 67.4 MB/s (9928288 bytes in 0.141s)
arm/boot-framework.vdex: 1 file pushed, 0 skipped. 1402.3 MB/s (37728 bytes in 0.000s)
3 files pushed, 0 skipped. 46.2 MB/s (13537728 bytes in 0.279s)

➜  framework adb push arm64/boot-framework.* /system/framework/arm64/
arm64/boot-framework.art: 1 file pushed, 0 skipped. 371.7 MB/s (3682304 bytes in 0.009s)
arm64/boot-framework.oat: 1 file pushed, 0 skipped. 94.9 MB/s (11949600 bytes in 0.120s)
arm64/boot-framework.vdex: 1 file pushed, 0 skipped. 1649.7 MB/s (37728 bytes in 0.000s)
3 files pushed, 0 skipped. 53.7 MB/s (15669632 bytes in 0.278s)

➜  framework adb sync
/data/: 201 files pushed, 0 skipped. 174.1 MB/s (261029915 bytes in 1.430s)
/product/: 73 files pushed, 0 skipped. 16.9 MB/s (2265863 bytes in 0.128s)
adb: error: failed to copy '/Users/rose/aosp/android-11.0.0_r46/out/target/product/bonito/system/bin/incident-helper-cmd' to '/system/bin/incident-helper-cmd': remote couldn't create file: Read-only file system
```





















