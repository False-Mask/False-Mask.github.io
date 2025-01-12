---
title: AOSP源码阅读环境配置
tags:
  - android
  - aosp
date: 2025-01-12 19:24:30
cover: https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/android_stack.png
---




# AOSP阅读环境配置

![android_stack](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/android_stack.png)



# 硬件环境



DIY主机

CPU：AMD R9 7950X

内存：光威 6400Hz 32G（16G * 2）

主板：GIGABYTE 冰雕小板

硬盘：ZhiTai TiPlus 7100 Gen4 2TB



# 软件配置



系统：Ubuntu 24.04





# 源代码安装



具体流程可分为

1.repo工具下载

2.源代码下载



略



安装流程可参考

[清华镜像源帮助问题](https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/)



# 工程基础配置



> 注意这里不是说repo工具需要做一些配置。
>
> 而是说需要使用repo做一些工程上的配置。



 1.repo start branchName --all

将所有工程开启新分支（方便后续拉取多分支）



2.配置脚本

>  可将常用的脚本书写复用
>
> 如下是笔者封装的lunch脚本。（用于source aosp脚本）

```shell
source build/envsetup.sh
# 自定义output路径，防止在不同分支check的时候输出相互覆盖。打包异常。
export OUT_DIR=out/$(repo branch | grep "\*" | awk -F ' ' '{print $2}')  && lunch aosp_redfin-userdebug
```



3.zshrc配置

> aosp soong build system配置，用于方便调试。

```shell
# aosp soong 
# 生成compile_commands.json
# ref https://android.googlesource.com/platform/build/soong/+show/master/docs/compdb.md
export SOONG_GEN_COMPDB=1
export SOONG_GEN_COMPDB_DEBUG=1
```







# 源码阅读环境配置



> 现在是2025/1/12日
>
> AOSP的源代码主要是C++/Java, 目前没找到特别满意的能同时阅读 C++ & Java的工具。所以C++ Java环境目前是单独配置的。
>
> 其中Java使用的是ASfp, C++使用的VsCode + Clangd + LLDB



## java环境



**1.Asfp配置**

> 安装过程略。
>
> 具体可见[Andriod Developer官网](https://developer.android.com/studio/platform?hl=zh-tw)
>
> 简单介绍下Asfp.（Android Studio for Platform）其实就是Android Studio. （只有Linux可以安装）
>
> 差别就是这个Android Studio兼容了soong构建系统，可以用于直接阅读 aosp source code



ASfp其实就比普通Android Studio多了一个tab.

![image-20250112180210389](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250112180210389.png)

图一 Asfp tab



使用方式就简单说下可以通过图一的 Asfp new Project开启一个弹窗，填写部分信息等待Sync完成就好了～

a. 输入aosp路径

![image-20250112180659865](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250112180659865.png)

图二Asfp WorkSpace设置



b.填写lunch target，阅读源码的路径（一般都是frameworks/base, 按需填写）

![image-20250112180845308](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250112180845308.png)

图三 Asfp 打开模块配置



**2.配置asfp json文件（如果你跟我一样自定义了OUT_DIR或者其他路径）**

![image-20250112182709597](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250112182709597.png)

图四 config.json配置





> 到这就基本上配置好了～





## native环境



0.安装vscode

1.插件安装

[CodeLLDB](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb)

[Clangd](https://marketplace.visualstudio.com/items?itemName=llvm-vs-code-extensions.vscode-clangd)



2.配置Clangd

配置如下XXX.code-workspace文件

```json
{
    "folders": [
        {
            "name": "android13-r78",
            "path": "/Users/rose/aosp/source/"
        }
    ],
    "settings": {
           // 开启粘贴保存自动格式化
        "editor.formatOnPaste": true,
        "editor.formatOnType": true,
        "C_Cpp.errorSquiggles": "Disabled",
        "C_Cpp.intelliSenseEngineFallback": "Disabled",
        "C_Cpp.intelliSenseEngine": "Disabled",
        // 由于需系统版本原因，可能路径有一定差异，可以在prebuilts/clang/host/linux-x86/找找
        "clangd.path": "/Users/rose/aosp/source/prebuilts/clang/host/linux-x86/clang-r450784d/bin/clangd",
        // Clangd 运行参数(在终端/命令行输入 clangd --help-list-hidden 可查看更多)
        "clangd.arguments": [
            // compile_commands.json 生成文件夹
            "--compile-commands-dir=/Users/rose/aosp/source/out/android13-r78/soong/development/ide/compdb/",
            // 让 Clangd 生成更详细的日志
            "--log=verbose",
            // 输出的 JSON 文件更美观
            "--pretty",
            // 全局补全(输入时弹出的建议将会提供 CMakeLists.txt 里配置的所有文件中可能的符号，会自动补充头文件)
            "--all-scopes-completion",
            // 建议风格：打包(重载函数只会给出一个建议）
            // 相反可以设置为detailed
            "--completion-style=bundled",
            // 跨文件重命名变量
            "--cross-file-rename",
            // 允许补充头文件
            "--header-insertion=iwyu",
            // 输入建议中，已包含头文件的项与还未包含头文件的项会以圆点加以区分
            "--header-insertion-decorators",
            // 在后台自动分析文件(基于 complie_commands，我们用CMake生成)
            "--background-index",
            // 启用 Clang-Tidy 以提供「静态检查」
            "--clang-tidy",
            // Clang-Tidy 静态检查的参数，指出按照哪些规则进行静态检查，详情见「与按照官方文档配置好的 VSCode 相比拥有的优势」
            // 参数后部分的*表示通配符
            // 在参数前加入-，如-modernize-use-trailing-return-type，将会禁用某一规则
            "--clang-tidy-checks=cppcoreguidelines-*,performance-*,bugprone-*,portability-*,modernize-*,google-*",
            // 默认格式化风格: 谷歌开源项目代码指南
            // "--fallback-style=file",
            // 同时开启的任务数量， 可以依据核心数调节
            "-j=32",
            // pch优化的位置(memory 或 disk，选择memory会增加内存开销，但会提升性能) 推荐在板子上使用disk
            "--pch-storage=disk",
            // 启用这项时，补全函数时，将会给参数提供占位符，键入后按 Tab 可以切换到下一占位符，乃至函数末
            // 我选择禁用
            "--function-arg-placeholders=false"
        ],
        "lldb.displayFormat": "auto",
        "lldb.showDisassembly": "auto",
        "lldb.dereferencePointers": true,
        "lldb.consoleMode": "commands",
     },
    "launch": {
        "configurations": [
            
        ]
    }
}

```





3.配置LLDB



在aosp root路径下下配置lldb init file

```shell
# 排除SIGSTOP，SIGSEGV断点
process handle SIGSTOP -n true -p true -s false
process handle SIGSEGV -n true -p true -s false

```

具体调试流程不做详细介绍，具体可见[Andriod Developer 官网](https://source.android.com/docs/core/tests/debug/gdb)



# 其他





## repo技巧



> repo其实本质上就是通过manifest组织多个git repository, 通过统一对仓库执行git指令实现管理



1.repo forall

repo forall是一个非常有用的指令他可以实现特别多想法。

```shell
# 清除untrack commit(清除以前记得做check, 不可回滚！！！)
repo forall -c "git clean -dfn"

# stash所有变更
repo forall -c "git stash push -m XXX"

# 查看所有project路径
repo forall -c "pwd"

```



2.repo init

init不仅仅能用于初始化下载源码，还能用于切换分支。

假如有一个场景：我现在下载了android13的源码，如果我想且回去。就可以使用如下指令实现

```shell
# 下载android13源码
repo init -u https://XXX -b android-13.0.0_r78

# 下载
repo sync 

# 切换分支
repo init -b android-11.0.0_r48

# checkout到制定的指定的分支
# 这里-l是指 `不使用network,本地同步修改工作区`
repo sync -l 
```



3.repo checkout

用于将所有的project切换到特定分支

等价于`repo forall -c "git checkout XXX"`



# 总结



1.主要针对于Java & C++源码阅读环境 & 调试环境进行了配置。

2.其中Java可以使用Asfp进行源码查阅以及调试

> Asfp其实也可以阅读C++代码，只是阅读体验不佳。
>
> 代码中会有很多报错。语法提示，代码跳转都存在一些问题。
>
> 不过好在Java源码阅读、debug特别给力。（很吃性能，内存20G以下电脑就别尝试了，32G勉强能用）

3.C++需要使用Vscode + LLDB + Clangd

> LLDB用于调试，Clangd用于解析源码查阅，阅读体验不逊于Clion, 而且不吃性能
>
> 缺点很多，主要是配置比较繁琐，不过反过来看定制性很强～







