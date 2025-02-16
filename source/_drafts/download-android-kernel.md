---
title: 下载 Android 内核
date: 
tags:
cover:
---



# 下载Android 内核

refs:

 [构建android kernel](https://source.android.com/docs/setup/build/building-kernels) 

[构建Pixel Kernel](https://source.android.com/docs/setup/build/building-pixel-kernels)





# 配置代理



这一点很容易想到，很多镜像站没有对于的教程。

不过无意间发现中科大镜像站虽然没有教程。但是点击他们的aosp地址，能search到（其他可能也行，但是我没试过）

![image-20250119002412714](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250119002412714.png)

结合之前的AOSP下载地址我们不难猜出地址为`https://mirrors.ustc.edu.cn/aosp/kernel/manifest`

```shell
repo init -u https://mirrors.ustc.edu.cn/aosp/kernel/manifest -b android-gs-raviole-5.10-android13-qpr3（branch按需指定）
```



低版本的kernel manifest有一些缺陷，这个缺陷会导致repo init设置的代理失效。具体可通过repo manifest查看aosp是否是..如果不是。

请使用vim .repo/manifests/default.xml将其修改为..

![image-20250119111634276](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250119111634276.png)



接着就是很经典的repo sync操作。几个小时后内核就下载好了





# 构建内核



refs：

[Compile the kernel kleaf](https://source.android.com/docs/setup/build/building-pixel-kernels#compile_the_kernel_kleaf)

```shell
./build_slider.sh
```



> Note:
>
> 如果你跟我一样使用的Ubuntu 24.04那么编译过程中会有一个报错
>
> ![image-20250119142631268](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250119142631268.png)
>
> 经过排查分析是Ubuntu 24.04更新了glibc 2.38 导致符号表少了。
>
> https://android.googlesource.com/kernel/common/+/75f82c6a15c4188cbb32825892fc6ae3e95479f0
>
> 虽然代码中合入了修复。but还是无法构建，推测可能需要额外做一些配置。（感兴趣的可以深入挖掘下怎么利用上述修复）
>
> 但是我没太理解需要怎么做去开启他的修复，所以我考虑使用docker换个系统镜像进行构建。
>
> ![image-20250119170113055](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250119170113055.png)
>
> 
>
> 查询一番资料后发现Ubunutu 22.04是可以完成的
>
> https://www.linkedin.com/pulse/building-android-linux-kernel-docker-container-abylay-ospan-6ct6e/
>
> ![image-20250119165158427](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250119165158427.png)
>
> 理论上Ubuntu 24.04也是可以构建的
>
> https://forum.khadas.com/t/build-kernel6-1-fail-undefined-symbol-isoc23-strtoull/23432
>
> 就是不知道这个补丁是啥？已经是否做了其他的配置。
>
> ![image-20250119165241596](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250119165241596.png)

