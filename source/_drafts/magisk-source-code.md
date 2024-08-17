---

\title: Magisk源码解析
tags:
- magisk
cover:
---



# Magisk源码解析



> 由前面的基础部分我们知道了，Magisk核心就是通过Magisk App Patch boot,img的过程。
>
> 接下来我们对Magisk Patch的部分逻辑





# Patch



## 线索分析



> 我们可以从Magisk的Install界面中找到Patch选择栏

<img src="https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20240602200203673.png" alt="image-20240602200203673" style="zoom:25%;" />



> 通过全局搜索一下，寻找思路
>
> 很轻松就找到了string为select_patch_file

![image-20240602200313249](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20240602200313249.png)



> 再接着search就找到了layout文件
>
> 其实我们可以这是个RadioGroup
>
> 其中一共有3个按钮。
>
> 第一个是Patch下载
>
> 第二个是直接下载（需要root）
>
> 第三个为OTA下载（需要有AB分区）
>
> 本机用的手机未满足后面两个情况，因此界面上看来就只有一个RadioButton

![image-20240602200519841](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20240602200519841.png)



> 回到正题我们通过search以后能很容易发现有两个地方用到了这个id

![image-20240602201003567](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20240602201003567.png)



> 经过排除后确认了安装的具体位置

```kotlin
// InstallViewModel.kt
fun install() {
        when (method) {
            R.id.method_patch -> FlashFragment.patch(data.value!!).navigate(true)
            R.id.method_direct -> FlashFragment.flash(false).navigate(true)
            R.id.method_inactive_slot -> FlashFragment.flash(true).navigate(true)
            else -> error("Unknown value")
        }
    }
```



## 具体分析



> 前一部分发现了Patch的位置，我们后面对具体的patch流程进行分析
>
> 通过最后的install代码我们可以得知代码的逻辑是可能是和FlashFragment高度相关的。
>
> 不过这不是重点，现接着FlashFragment.patch

```kotlin
// 简单查看了下这是个到FlashFragment的navigation  
fun patch(uri: Uri) = MainDirections.actionFlashFragment(
            action = Const.Value.PATCH_FILE,
            additionalData = uri
        )
```



> 然后发现具体的逻辑在FlashViewModel的一个分支里面
>
> 其中我们需要的逻辑在`MagiskInstaller.Patch(uri, outItems, logItems).exec()`这一行

```kotlin
fun startFlashing() {
        val (action, uri) = args

        viewModelScope.launch {
            val result = when (action) {
                Const.Value.FLASH_ZIP -> {
                    uri ?: return@launch
                    FlashZip(uri, outItems, logItems).exec()
                }
                Const.Value.UNINSTALL -> {
                    showReboot = false
                    MagiskInstaller.Uninstall(outItems, logItems).exec()
                }
                Const.Value.FLASH_MAGISK -> {
                    if (Info.isEmulator)
                        MagiskInstaller.Emulator(outItems, logItems).exec()
                    else
                        MagiskInstaller.Direct(outItems, logItems).exec()
                }
                Const.Value.FLASH_INACTIVE_SLOT -> {
                    MagiskInstaller.SecondSlot(outItems, logItems).exec()
                }
                Const.Value.PATCH_FILE -> {
                    uri ?: return@launch
                    showReboot = false
                    MagiskInstaller.Patch(uri, outItems, logItems).exec()
                }
                else -> {
                    back()
                    return@launch
                }
            }
            onResult(result)
        }
    }
```



> MagiskInstall是一个抽象。
>
> Patch是它的实现之一
>
> 调用Patch的exec最后估计调用到patchFile

```kotlin
class Patch(
        private val uri: Uri,
        console: MutableList<String>,
        logs: MutableList<String>
    ) : MagiskInstaller(console, logs) {
        override suspend fun operations() = patchFile(uri)
    }
```



> 然后patchFile会调用两个方法
>
> extractFiles
>
> 估计是用来把一些Magisk的资源文件解压
>
> handleFile
>
> 估计是对文件进行Patch等操作

```kotlin
protected fun patchFile(file: Uri) = extractFiles() && handleFile(file)
```





### extractFiles



> 主要用于将apk内的文件复制到app的data路径



> 部分核心代逻辑如下
>
> 1. 将所有的lib.so通过symbol link到`../${context.filesDir}/install`
> 2. 导出apk assets路径中的scripts以及chromeos文件

```kotlin
// 获取application、app的native lib64路径
val info = context.applicationInfo
var libs = File(info.nativeLibraryDir).listFiles { _, name ->
    name.startsWith("lib") && name.endsWith(".so")
} ?: emptyArray()

// 同时添加部分lib32的库路径
val lib32 = info.javaClass.getDeclaredField("secondaryNativeLibraryDir")
    .get(info) as String?
if (lib32 != null) {
    libs += File(lib32, "libmagisk32.so")
}

// 将所有的lib用符号链接link
for (lib in libs) {
    val name = lib.name.substring(3, lib.name.length - 3)
    Os.symlink(lib.path, "$installDir/$name")
}

// ......
// 导出assets中的scripts文件
for (script in listOf("util_functions.sh", "boot_patch.sh", "addon.d.sh", "stub.apk")) {
    val dest = File(installDir, script)
    context.assets.open(script).writeTo(dest)
}

// Extract chromeos tools
// 导出assets中的chromeos文件
File(installDir, "chromeos").mkdir()
for (file in listOf("futility", "kernel_data_key.vbprivk", "kernel.keyblock")) {
    val name = "chromeos/$file"
    val dest = File(installDir, name)
    context.assets.open(name).writeTo(dest)
}
```





> 在运行以后我们可以通过查看magisk app的路径查看移动的文件内容

```shell
vermeer:/data/user_de/0/com.topjohnwu.magisk/install $ ls

addon.d.sh     busybox   magisk32  magiskboot  magiskpolicy  util_functions.sh
boot_patch.sh  chromeos  magisk64  magiskinit  stub.apk
```





### handleFile



> 1. 解析input文件（提取tar、zip、payload.bin文件进行解析）
> 2. 给boot img打补丁
> 3. 输出Patched boot img文件

```kotlin
try {
  uri.inputStream().buffered().use { src ->
            src.mark(500)
            val magic = ByteArray(4)
            val tarMagic = ByteArray(5)
            if (src.read(magic) != magic.size || src.skip(253) != 253L ||
                src.read(tarMagic) != tarMagic.size
            ) {
                console.add("! Invalid input file")
                return false
            }
            src.reset()
            val alpha = "abcdefghijklmnopqrstuvwxyz"
            val alphaNum = "$alpha${alpha.uppercase(Locale.ROOT)}0123456789"
            val random = SecureRandom()
            val filename = StringBuilder("magisk_patched-${BuildConfig.VERSION_CODE}_").run {
                for (i in 1..5) {
                    append(alphaNum[random.nextInt(alphaNum.length)])
                }
                toString()
            }

            srcBoot = if (tarMagic.contentEquals("ustar".toByteArray())) {
                // tar file
                outFile = MediaStoreUtils.getFile("$filename.tar", true)
                outStream = TarOutputStream(outFile.uri.outputStream())

                try {
                    processTar(TarInputStream(src), outStream)
                } catch (e: IOException) {
                    outStream.close()
                    outFile.delete()
                    throw e
                }
            } else {
                // raw image
                outFile = MediaStoreUtils.getFile("$filename.img", true)
                outStream = outFile.uri.outputStream()

                try {
                    if (magic.contentEquals("CrAU".toByteArray())) { // payload魔数
                        processPayload(src)
                    } else if (magic.contentEquals("PK\u0003\u0004".toByteArray())) { // zip魔数
                        processZip(ZipInputStream(src))
                    } else {
                        console.add("- Copying image to cache")
                        installDir.getChildFile("boot.img").also {
                            src.copyAndCloseOut(it.newOutputStream())
                        }
                    }
                } catch (e: IOException) {
                    outStream.close()
                    outFile.delete()
                    throw e
                }
            }
        }
    } catch (e: IOException) {
        if (e is NoBootException)
            console.add("! No boot image found")
        console.add("! Process error")
        Timber.e(e)
        return false
    }
```



#### input



> 对tar，payload.bin，zip格式进行解压，提取其中的boot.img镜像。
>
> 并复制到$installDir/boot.img

```kotlin
 uri.inputStream().buffered().use { src ->
                src.mark(500)
                val magic = ByteArray(4)
                val tarMagic = ByteArray(5)
                if (src.read(magic) != magic.size || src.skip(253) != 253L ||
                    src.read(tarMagic) != tarMagic.size
                ) {
                    console.add("! Invalid input file")
                    return false
                }
                src.reset()

                val alpha = "abcdefghijklmnopqrstuvwxyz"
                val alphaNum = "$alpha${alpha.uppercase(Locale.ROOT)}0123456789"
                val random = SecureRandom()
                val filename = StringBuilder("magisk_patched-${BuildConfig.VERSION_CODE}_").run {
                    for (i in 1..5) {
                        append(alphaNum[random.nextInt(alphaNum.length)])
                    }
                    toString()
                }

                srcBoot = if (tarMagic.contentEquals("ustar".toByteArray())) {
                    // tar file
                    outFile = MediaStoreUtils.getFile("$filename.tar", true)
                    outStream = TarOutputStream(outFile.uri.outputStream())

                    try {
                        processTar(TarInputStream(src), outStream)
                    } catch (e: IOException) {
                        outStream.close()
                        outFile.delete()
                        throw e
                    }
                } else {
                    // raw image
                    outFile = MediaStoreUtils.getFile("$filename.img", true)
                    outStream = outFile.uri.outputStream()

                    try {
                        if (magic.contentEquals("CrAU".toByteArray())) { // payload
                            processPayload(src)
                        } else if (magic.contentEquals("PK\u0003\u0004".toByteArray())) { // zip
                            processZip(ZipInputStream(src))
                        } else {
                            console.add("- Copying image to cache")
                            // 将img拷贝到$installDir/boot.img
                            installDir.getChildFile("boot.img").also {
                                src.copyAndCloseOut(it.newOutputStream())
                            }
                        }
                    } catch (e: IOException) {
                        outStream.close()
                        outFile.delete()
                        throw e
                    }
                }
            }
        } catch (e: IOException) {
            if (e is NoBootException)
                console.add("! No boot image found")
            console.add("! Process error")
            Timber.e(e)
            return false
        }
```





#### patch



> 打补丁并执行如下三条指令。
>
> 1.  cd $installDir
> 2.  sh boot_patch.sh $srcBoot
> 3.  ./magiskboot cleanup

```kotlin
private fun patchBoot(): Boolean {
        val newBoot = installDir.getChildFile("new-boot.img")
        if (!useRootDir) {
            // Create output files before hand
            newBoot.createNewFile()
            File(installDir, "stock_boot.img").createNewFile()
        }

    	// 执行通过命令行
        val cmds = arrayOf(
            "cd $installDir",
            "KEEPFORCEENCRYPT=${Config.keepEnc} " +
            "KEEPVERITY=${Config.keepVerity} " +
            "PATCHVBMETAFLAG=${Info.patchBootVbmeta} " +
            "RECOVERYMODE=${Config.recovery} " +
            "LEGACYSAR=${Info.legacySAR} " +
            ""
        )
        val isSuccess = cmds.sh().isSuccess

        shell.newJob().add("./magiskboot cleanup", "cd /").exec()

        return isSuccess
 }
```





##### boot_patch.sh



> initialize

```shell
# SOURCEDMODE如果没有设置
if [ -z $SOURCEDMODE ]; then
  # 切换到bash路径或者boot脚本路径（自定函数）
  cd "$(getdir "${BASH_SOURCE:-$0}")"
  # Load utility functions
  . ./util_functions.sh
  # Check if 64-bit 执行函数 记录ABI
  api_level_arch_detect
fi

api_level_arch_detect() {
  API=$(grep_get_prop ro.build.version.sdk)
  ABI=$(grep_get_prop ro.product.cpu.abi)
  if [ "$ABI" = "x86" ]; then
    ARCH=x86
    ABI32=x86
    IS64BIT=false
  elif [ "$ABI" = "arm64-v8a" ]; then
    ARCH=arm64
    ABI32=armeabi-v7a
    IS64BIT=true
  elif [ "$ABI" = "x86_64" ]; then
    ARCH=x64
    ABI32=x86
    IS64BIT=true
  else
    ARCH=arm
    ABI=armeabi-v7a
    ABI32=armeabi-v7a
    IS64BIT=false
  fi
}

```



> 为一些特殊的设备单独dump 镜像

```shell
BOOTIMAGE="$1"
[ -e "$BOOTIMAGE" ] || abort "$BOOTIMAGE does not exist!"

# Dump image for MTD/NAND character device boot partitions
if [ -c "$BOOTIMAGE" ]; then
  nanddump -f boot.img "$BOOTIMAGE"
  BOOTNAND="$BOOTIMAGE"
  BOOTIMAGE=boot.img
fi
```



> 设置Flags

```shell
[ -z $KEEPVERITY ] && KEEPVERITY=false
[ -z $KEEPFORCEENCRYPT ] && KEEPFORCEENCRYPT=false
[ -z $PATCHVBMETAFLAG ] && PATCHVBMETAFLAG=false
[ -z $RECOVERYMODE ] && RECOVERYMODE=false
[ -z $LEGACYSAR ] && LEGACYSAR=false
export KEEPVERITY
export KEEPFORCEENCRYPT
export PATCHVBMETAFLAG

chmod -R 755 .
```



> unpack image
>
> 就调用magiskboot unpack 指定的镜像

```shell
CHROMEOS=false

ui_print "- Unpacking boot image"
./magiskboot unpack "$BOOTIMAGE"

# 对返回指进行判断
case $? in
  0 ) ;;
  1 )
    abort "! Unsupported/Unknown image format"
    ;;
  2 )
    ui_print "- ChromeOS boot image detected"
    CHROMEOS=true
    ;;
  * )
    abort "! Unable to unpack boot image"
    ;;
esac
```



> ramdisk备份
>
> 1. 原厂ramdisk
>
>    计算sha1值、复制镜像文件、备份ramdisk文件
>
> 2. Patch后的ramdisk
>
>    解压cpio文件
>
>    将magisk文件导出到config.orig
>
>    将ramdisk.cpio文件恢复到原厂模式
>
>    备份并删除原厂img文件
>
> 3. 不支持的ramdisk
>
>    报错退出

```shell
ui_print "- Checking ramdisk status"
# 判断ramdisk是否存在
if [ -e ramdisk.cpio ]; then
  # 通过magiskboot获取cpio文件的状态
  ./magiskboot cpio ramdisk.cpio test
  STATUS=$?
  SKIP_BACKUP=""
else
  # Stock A only legacy SAR, or some Android 13 GKIs
  STATUS=0
  SKIP_BACKUP="#"
fi
case $((STATUS & 3)) in
  0 )  # Stock boot 原厂镜像
    ui_print "- Stock boot image detected"
    # 获取原厂镜像的sha1值
    SHA1=$(./magiskboot sha1 "$BOOTIMAGE" 2>/dev/null)
    # 复制原厂镜像
    cat $BOOTIMAGE > stock_boot.img
    # 备份ramdisk.cpio文件
    cp -af ramdisk.cpio ramdisk.cpio.orig 2>/dev/null
    ;;
  1 )  # Magisk patched 经过补丁的镜像
    ui_print "- Magisk patched boot image detected"
    # 解压cpio文件
    ./magiskboot cpio ramdisk.cpio \
    # 将magisk文件导出到config.orig
    "extract .backup/.magisk config.orig" \
    # 恢复ramdisk.cpio文件
    "restore"
    # 备份袁术ramdisk文件
    cp -af ramdisk.cpio ramdisk.cpio.orig
    # 删除原厂镜像
    rm -f stock_boot.img
    ;;
  2 )  # Unsupported
    ui_print "! Boot image patched by unsupported programs"
    abort "! Please restore back to stock boot image"
    ;;
esac
```



> 兼容sony设备

```shell
INIT=init
if [ $((STATUS & 4)) -ne 0 ]; then
  INIT=init.real
fi

# 读取PREINITDEVICE配置信息
if [ -f config.orig ]; then
  # Read existing configs
  chmod 0644 config.orig
  SHA1=$(grep_prop SHA1 config.orig)
  if ! $BOOTMODE; then
    # Do not inherit config if not in recovery
    PREINITDEVICE=$(grep_prop PREINITDEVICE config.orig)
  fi
  rm config.orig
fi
```



> ramdisk patch
>
> 1. 压缩magisk可执行文件、stub.apk文件
> 2. 将环境变量写入config文件
> 3. 想ramdisk中加入magisk文件、init文件、config文件
> 4. 删除中间文件

```shell
ui_print "- Patching ramdisk"

# 压缩以节省ramdisk宝贵的内存空间
# Compress to save precious ramdisk space
SKIP32="#"
SKIP64="#"

# 压缩magisk32/64文件
if [ -f magisk64 ]; then
  $BOOTMODE && [ -z "$PREINITDEVICE" ] && PREINITDEVICE=$(./magisk64 --preinit-device)
  ./magiskboot compress=xz magisk64 magisk64.xz
  unset SKIP64
fi
if [ -f magisk32 ]; then
  $BOOTMODE && [ -z "$PREINITDEVICE" ] && PREINITDEVICE=$(./magisk32 --preinit-device)
  ./magiskboot compress=xz magisk32 magisk32.xz
  unset SKIP32
fi
./magiskboot compress=xz stub.apk stub.xz

# 环境变量写入config文件
echo "KEEPVERITY=$KEEPVERITY" > config
echo "KEEPFORCEENCRYPT=$KEEPFORCEENCRYPT" >> config
echo "RECOVERYMODE=$RECOVERYMODE" >> config
if [ -n "$PREINITDEVICE" ]; then
  ui_print "- Pre-init storage partition: $PREINITDEVICE"
  echo "PREINITDEVICE=$PREINITDEVICE" >> config
fi
[ -n "$SHA1" ] && echo "SHA1=$SHA1" >> config

# 向ramdisk中加入init文件以及magisk zip文件
./magiskboot cpio ramdisk.cpio \
"add 0750 $INIT magiskinit" \
"mkdir 0750 overlay.d" \
"mkdir 0750 overlay.d/sbin" \
"$SKIP32 add 0644 overlay.d/sbin/magisk32.xz magisk32.xz" \
"$SKIP64 add 0644 overlay.d/sbin/magisk64.xz magisk64.xz" \
"add 0644 overlay.d/sbin/stub.xz stub.xz" \
"patch" \
"$SKIP_BACKUP backup ramdisk.cpio.orig" \
"mkdir 000 .backup" \
"add 000 .backup/.magisk config" \
|| abort "! Unable to patch ramdisk"

# 删除生成的中间文件
rm -f ramdisk.cpio.orig config magisk*.xz stub.xz
```



> patch 二进制文件
>
> 1. 去除samsung RKP公钥文件
> 2. 对Samsung部分代码做修改
> 3. 对SAR设备做兼容，强制加载rootfs

```shell
if [ -f kernel ]; then
  PATCHEDKERNEL=false
  # Remove Samsung RKP
  # hexpatch即把kernel中的匹配的字符串内容转化为了后面的hex数据
  ./magiskboot hexpatch kernel \
  49010054011440B93FA00F71E9000054010840B93FA00F7189000054001840B91FA00F7188010054 \
  A1020054011440B93FA00F7140020054010840B93FA00F71E0010054001840B91FA00F7181010054 \
  && PATCHEDKERNEL=true

  # Remove Samsung defex
  # Before: [mov w2, #-221]   (-__NR_execve)
  # After:  [mov w2, #-32768]
  ./magiskboot hexpatch kernel 821B8012 E2FF8F12 && PATCHEDKERNEL=true

  # Force kernel to load rootfs for legacy SAR devices
  # skip_initramfs -> want_initramfs
  # 强制修改kernel rootfs
  $LEGACYSAR && ./magiskboot hexpatch kernel \
  736B69705F696E697472616D667300 \
  77616E745F696E697472616D667300 \
  && PATCHEDKERNEL=true

  # If the kernel doesn't need to be patched at all,
  # keep raw kernel to avoid bootloops on some weird devices
  $PATCHEDKERNEL || rm -f kernel
fi
```



> 将当前路径的文件进行打包

```shell
ui_print "- Repacking boot image"
./magiskboot repack "$BOOTIMAGE" || abort "! Unable to repack boot image"

# Sign chromeos boot
$CHROMEOS && sign_chromeos

# Restore the original boot partition path
[ -e "$BOOTNAND" ] && BOOTIMAGE="$BOOTNAND"
```



> 返回

```shell
# Reset any error code
true
```



##### magisk



对于magiskboot的配置是在Android.mk中的

```makefile
ifdef B_BOOT
# 清空变量
include $(CLEAR_VARS)
# 定义模块名称以及依赖
LOCAL_MODULE := magiskboot
LOCAL_STATIC_LIBRARIES := \
    libbase \
    libcompat \
    liblzma \
    liblz4 \
    libbz2 \
    libz \
    libzopfli \
    libboot-rs

# 定义src文件
LOCAL_SRC_FILES := \
    boot/main.cpp \
    boot/bootimg.cpp \
    boot/compress.cpp \
    boot/format.cpp \
    boot/boot-rs.cpp

include $(BUILD_EXECUTABLE)

endif
```



> 其中有一个比较特殊的依赖(boot-rs)是使用rust书写的

```makefile
ifneq (,$(wildcard $(LOCAL_PATH)/$(LIBRARY_PATH)))
include $(CLEAR_VARS)
LOCAL_MODULE := boot-rs
LOCAL_SRC_FILES := $(LIBRARY_PATH)
include $(PREBUILT_STATIC_LIBRARY)
endif
```











