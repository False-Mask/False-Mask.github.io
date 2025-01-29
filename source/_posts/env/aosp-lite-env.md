---
title: AOSP 轻量级环境配置
tags:
  - aosp
  - env
cover: 'https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/android_stack.png'
date: 2025-01-29 23:00:52
---


# AOSP轻量级环境配置



<img src="https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250129113119104.png" alt="Android Emulator" style="zoom:50%;" />





## 背景

> 近期在学习 ART 执行执行过程，但是没带我自己的 Pixel。
>
> 所以尝试搭建了一个轻量级的 AOSP 源码阅读环境



>  事先说明，你需要两台主机。
>
> 其中一台用于运行模拟器 & 调试，可以是 Linux/Win/Mac（笔者这里是使用的 Mac M3）
>
> 还有一台 Linux 主机用于打包 AOSP，这里笔者是用 Ubuntu20.04。（可以没有 GUI，通过 SSH 直连，但是蛮吃配置的）



从上述描述来看其实也不是特别轻量级～主要是 Linux 主机是必须的，但是好在可以使用自己的模拟器进行调试。这样便携性会比较高。

但是这也是万不得已的情况，如果有实体机，最好还是用实体机。用着比较舒服～（模拟器不知道会不会有些内核 bug？！）

# 流程



Refs：https://weishu.me/2016/05/30/how-to-debug-android-framework/

Refs：https://weishu.me/2017/01/14/how-to-debug-android-native-framework-source/



## 打包 & 下载

> 需要一个 Linux主机，打包模拟器镜像。



> 简单讲一下流程。
>
> refs：
>
> [打包 SDK 模拟器](https://source.android.com/docs/setup/create/avd?hl=zh-cn)
>
> 
>
> 1.安装 repo & repo sync 安装源代码
>
> 2.lunch sdk_phone64_arm64-userdebug（依据PC 电脑的机构，笔者最终需要在 Mac arm64 上运行）
>
> Note: 
>
> lunch type 一定得选对，不然会崩溃。笔者最终运行模拟器的是 Mac arm64，但是不小心选错成了sdk_phone_arm64-userdebug 运行模拟器时发现了Kernel Panic 
>
> “Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)”
>
> Refs：https://stackoverflow.com/questions/77404675/aosp-emulator-images-not-running
>
> "I was having exact same issue but with Android 13 emulator. We were able fix the problem by setting `lunch` target as `sdk_phone64_arm64-userdebug`".
>
> 
>
> 3.m 打包镜像
>
> 4.make emu_img_zip打包模拟器压缩包



打包完成后能看到☝️提示

![image-20250128173427602](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250128173427602.png)

> 可以通过 rsync, scp 一些必要的内容 下载下来

```shell
# 路径做过一些修改，可以对照着查看下自己的文件名称是什么。
# 下载系统镜像
scp vpc-aosp:工程路径/out/target/product/emulator64_arm64/sdk-repo-linux-system-images-eng.tuzhiqiang03.zip .

# 下载symbols
rsync -avP vpc-aosp:/data/tuzhiqiang03/aosp/aosp/android13-r78/target/product/emulator64_arm64/symbols symbols
```



> 可以发现下载的东西蛮多的。

![image-20250128174734987](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250128174734987.png)

> 如果是硬盘较小的小伙伴可以按需下载～

![image-20250128174920118](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250128174920118.png)



## 模拟器配置

> Note：在此之前记得先创建一个指定系统版本的模拟器



> 这里也简单总结下，这里所谓的配置其实就是要将模拟器原来的镜像文件进行覆盖。
>
> 让模拟器运行我们自编的镜像 & 内核。



1.我们可以先将我们的zip文件解压

![image-20250128182215631](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250128182215631.png)

2.改名称，移动到emulator路径

```shell
mv ~/Downloads/arm64-v8a.android13_r78-userdebug  ~/Library/Android/sdk/system-images/android-33/google_apis/ 
```

3.将arm64-v8a文件修改名称，在通过 ln 链接到arm64-v8a.android13_r78-userdebug

```shell
mv arm64-v8a arm64-v8a.origin
ln -s arm64-v8a.android13_r78-userdebug arm64-v8a
```

4.启动模拟器

refs：https://developer.android.com/studio/run/emulator-commandline?hl=zh-cn

(首次启动务必加上-wipe-data格式化数据，后续就不用加了)

```shell
emulator @Pixel_6_API_33-userdebug -sysdir ~/Library/Android/sdk/system-images/android-33/google_apis/arm64-v8a.new.userdebug -wipe-data 
```



>  启动完成的模拟器就长这样

<img src="https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250129113119104.png" alt="image-20250129113119104" style="zoom:50%;" />



> Note 如果不喜欢模拟器独立窗口的可以参考下[官网文档](https://developer.android.com/studio/run/emulator-commandline?hl=zh-cn#starting)
>
> -qt-hide-window -grpc-use-token -idle-grpc-timeout
>
> 我试过好像不太行，懒得弄了 ٩(•̤̀ᵕ•̤́๑)ᵒᵏᵎᵎᵎᵎ



## AS配置



> 这里的配置其实主要就是 lldbinit 文件配置，此处的lldbinit 脚本主要是用于申明 symbol 文件地址。
>
> 最后我们还会将申明好的 lldb 文件通过 Android Studio Configure 导入，最后就实现了上述的调试流程。



1. 首先创建一个demo工程（就简单的 template 就行）
2. 将symbol存放到指定路径
3. 书写 lldbinit 文件

```shell
# 首先我check下哪些需要添加到符号表的search路径
~ cd symbols # 进入符号表路径


# 寻找所有需要的符号文件，输出一条symbol路径到 lldbinit 文件中
# 笔者这里针对于之前 sync 的 *.so, *.oat, linker, app_process 文件路径做了解析
# 目前来看覆盖了进程用到的所有系统symbol，当然如果有新增可以按需自行添加～
~ find . \( -name "*.so" -o -name "*.oat" -o -name "linker*" -o -name "app_process*" \) -exec sh -c 'dirname $(realpath "{}")' \; | sort | uniq | awk 'BEGIN {printf "settings append target.exec-search-paths "} {printf "%s ",$0}' >> .lldb-preinit 


```

4. 配置Gradle

> 你没看错是需要进行一些配置。

​	a.首先第一个解决的就是attach失败的问题

> 这个本质其实就是 gradle 的 packageId和实际attach process 的 pkgId 不一致导致的

![image-20250129222739019](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250129222739019.png)

要解决这个问题也很简单，就packageId配置成你想调试的进程就好了

```groovy
android {
  // ......
  defaultConfig {
    ndk {
      abiFilters += listOf("armeabi-v7a", "arm64-v8a")
    }
	  applicationId = "com.example.hookee" /*依据你需要 attach 进程而定*/
  }
}
```





5. 将lldbinit文件配置到demo project Configureation 中

> 如果有需要可以自行使用LLDB Post Attach Commands

![image-20250129223240584](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250129223240584.png)

> lldb-preinit内容如下，其实主要就是申明symbol file的路径而已

![image-20250129223433435](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250129223433435.png)



6. 打断点 & attach 进程

> 我们可以发现他停下了，停在了我们断点的位置～

![image-20250129222355382](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250129222355382.png)





> 就这样大功告成了～
