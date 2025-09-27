---
title: Android AGP源码下载编译
tags:
  - android
cover: https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250928001554139.png
date: 2025-09-28 00:23:27
---




# Andriod AGP下载 & 编译



# 源码安装



简单问了下AI,虽然我不知道源码在哪，但是他知道。

https://android.googlesource.com/platform/tools/base/

![image-20250927121150822](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250927121150822.png)



但是再经过一些文档的查阅，让我发现不完全是这样的。

https://android.googlesource.com/platform/tools/base/+/studio-master-dev/build-system/README.md



最终我还是决定参考[官方教程](https://android.googlesource.com/platform/tools/base/+/studio-master-dev/source.md)来。不过我这里没有拉官方的master分支，主要考虑是release分支会比较稳定。

```
repo init -u https://mirrors.ustc.edu.cn/aosp/platform/manifest.git -b studio-2025.1.3
repo sync
```





# SDK软连接



下载源码后我们的下一步就是用ide打开源码

> The code of the plugin and its dependencies is located in `tools/base`. You can open this project with IntelliJ as there is already a `tools/base/.idea` setup.
>
> ——[DOCS](https://android.googlesource.com/platform/tools/base/+/studio-master-dev/build-system/README.md#editing-the-plugin)



但是我们的项目工程估计会有一堆的报错

![image-20250927233318564](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250927233318564.png)



经过**简单的排查**我们不难发现崩溃的主要原因是由于ANDROID_HOME为None导致的问题。

ANDROID_HOME为什么会为None呢？原因有两点：

a. 问题一：第27行没兜住（笔者使用的Linux主机， 也就是说prebuilts/studio/sdk/linux路径是空的）

b. 问题二：第28行还是没兜住，环境变量没设置

当然这是两个问题。

![image-20250927233518551](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250927233518551.png)



## 问题一



我简单查阅了下资料，官方的意思是说repo sync以后文件路径是一定存在的。但是作者看过没有。（文档年久失修了）[Docs](https://android.googlesource.com/platform/tools/base/+/studio-master-dev/bazel/sdk/#updating-the-development-sdk)

![image-20250927234231709](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250927234231709.png)





然后作者非常坚持地又查阅了一番资料。简单来说就是官方源码中有部分不是完全开源的（platform/sdk部分），而这部分就是我们需要的。

但是看对话结论应该是有下载的方法的，但是这对话太长了，我懒得看了(其实不是， 就是英语比较差。摊牌了，不装了)

https://forum.f-droid.org/t/open-source-android-development-tools/30599/6

![image-20250927234559319](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250927234559319.png)



所以这条线就这样了，就是官方故意为之，不让开发者下载 (坏！！！)





## 问题二



这个问题很奇怪，因为最开始以为会很简单，我直接在\~/.zshrc，\~/.bashrc上设置一下ANDROID_HOME路径就行了。

但是我很快发现貌似没用。我们回顾性地看下代码，看看有啥发现。

他是通过python的语法读取的os.environ.get('ANDROID_HOME')，不知道有啥特殊之处，作者python也不熟练。

```python
 workspace = find_workspace(os.path.dirname(os.path.realpath(__file__)))
  if not workspace:
    sys.exit('Must run %s within a workspace.' % os.path.basename(sys.argv[0]))

  if sys.platform.startswith('linux'):
    platform = 'linux-x86_64'
    android_home = f'{workspace}/prebuilts/studio/sdk/linux'
  elif sys.platform == 'darwin':
    platform = 'darwin-arm64' if plat.machine() == 'arm64' else 'darwin-x86_64'
    android_home = f'{workspace}/prebuilts/studio/sdk/darwin'
  elif sys.platform == 'win32':
    platform = 'windows-x86_64'
    android_home = f'{workspace}/prebuilts/studio/sdk/windows'
  else:
    sys.exit('Platform %s is not yet supported.' % sys.platform)
  if not os.path.exists(android_home):
    android_home = os.environ.get('ANDROID_HOME')
```



但是这个也不是没办法解决，直接把ANDROID_HOME代码中写死就好了。

```python
env['ANDROID_HOME'] = "/home/rose/Android/Sdk"
```



但是，上天会惩罚每一个想走捷径的人，还有其他报错。看样子bazel中有部分代码是死了心要读prebuilts/studio/sdk/linux这个路径。

![image-20250927235728809](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250927235728809.png)





## 最终解决方法



但但是，这还是有解决方案，我直接把prebuilts/studio/sdk/linux这个路径link到sdk路径不就行了

Note: 使用这个方案，前面的ANDROID_HOME可以不配置，也就是说前面的排查过程可以全忘记～

```shell
cd $work_dir
ln -s /home/rose/Android/Sdk prebuilts/studio/sdk/linux
```



然后然后再用idea打开 tools/base 路径就好了





# 编译AGP代码



今日宜梭哈～

> To build the Android Gradle Plugin, run
>
> ```
> $ ./gradlew :publishAndroidGradleLocal
> ```

[DOCS](https://android.googlesource.com/platform/tools/base/+/studio-master-dev/build-system/README.md#building-the-plugin)



问题又来了，我从哪里运行这行指令? 经过实验发现是tools路径。

```shell
cd $work_dir/tools
./gradlew :publishAndroidGradleLocal 
```



这里再提醒以下，使用./gradlew 执行需要注意JAVA_HOME, 因为gradlew会获取JAVA_HOME or 环境变量中的java地址。

启动gradle server进程。如果jdk版本过高or过低会导致一些奇怪的崩溃。（配置Idea/Android Studio中的jdk路径是没用的）





最后在一小会之后，agp的源码就打包出来了～

```shell
➜  cd $work_dir
➜  studio tree out/repo/com/android/tools/build/gradle
out/repo/com/android/tools/build/gradle
├── 8.13.0-dev      backup/             common/             ddms/               emulator/           fakeadbserver/      layoutlib/          perf-logger/        repository/         sdk-common/         testutils/                                                                                                                                                                                                                
│   ├── gradle-8.13.0-dev.jar
│   ├── gradle-8.13.0-dev.jar.md5
│   ├── gradle-8.13.0-dev.jar.sha1
│   ├── gradle-8.13.0-dev.jar.sha256
│   ├── gradle-8.13.0-dev.jar.sha512
│   ├── gradle-8.13.0-dev.module
│   ├── gradle-8.13.0-dev.module.md5
│   ├── gradle-8.13.0-dev.module.sha1
│   ├── gradle-8.13.0-dev.module.sha256
│   ├── gradle-8.13.0-dev.module.sha512
│   ├── gradle-8.13.0-dev.pom
│   ├── gradle-8.13.0-dev.pom.md5
│   ├── gradle-8.13.0-dev.pom.sha1
│   ├── gradle-8.13.0-dev.pom.sha256
│   └── gradle-8.13.0-dev.pom.sha512
├── maven-metadata.xml
├── maven-metadata.xml.md5
├── maven-metadata.xml.sha1
├── maven-metadata.xml.sha256
└── maven-metadata.xml.sha512

2 directories, 20 files

3 directories, 10 files
```



# 寄语



![img](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/72421304)

***即使前路未明， 也请相信—— 总有一道光，正在等你走近***

