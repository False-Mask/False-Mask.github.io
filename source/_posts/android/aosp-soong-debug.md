---
title: AOSP Soong 构建系统调试环境配置
tags:
  - aosp
  - build
  - soong
  - env
cover: 'https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/android_stack.png'
date: 2025-09-20 11:46:12
---




# AOSP Soong debug 环境配置



# 前言



Soong是一个用Go语言编写的专用于AOSP的构建系统。

近期"突发恶疾"，想着分析下构建流程。

由于网上关于Soong的调试教程较少，因此就有了本文～



# 流程



1.go/dlv安装配置

2.waiting for debugger

3.attach to debugger



# dlv安装



1.go语言安装

2.安装dlv调试工具工具



# wait for debugger



soong_ui的启动代码在如下的脚本中

build/soong/soong_ui.bash

```shell
#!/bin/bash -eu
#
# Copyright 2017 Google Inc. All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# To track how long we took to startup.
case $(uname -s) in
  Darwin)
    export TRACE_BEGIN_SOONG=`$T/prebuilts/build-tools/path/darwin-x86/date +%s%3N`
    ;;
  *)
    export TRACE_BEGIN_SOONG=$(date +%s%N)
    ;;
esac

source $(cd $(dirname $BASH_SOURCE) &> /dev/null && pwd)/../make/shell_utils.sh
require_top

# Save the current PWD for use in soong_ui
export ORIGINAL_PWD=${PWD}
export TOP=$(gettop)
source ${TOP}/build/soong/scripts/microfactory.bash

soong_build_go soong_ui android/soong/cmd/soong_ui
soong_build_go mk2rbc android/soong/mk2rbc/mk2rbc
soong_build_go rbcrun rbcrun/rbcrun

exec "$(getoutdir)/soong_ui" "$@"

```



为了能使得soong程序等待调试器，我们需要做一些简单的操作～

将最后一行代码exec "$(getoutdir)/soong_ui" "$@"做如下修改

```shell
if echo "$@" | grep -q -- "--build-mode"; then
    echo "参数 --build-mode 存在"
    if [[ -v SOONG_DEBUG ]]; then 
      echo "进入调试模式"
      exec dlv --listen=:2345 --headless=true --api-version=2 exec -- "$(getoutdir)/soong_ui" "$@"
    else 
      exec "$(getoutdir)/soong_ui" "$@"
    fi
else
    #echo "参数 --build-mode 不存在"
    exec "$(getoutdir)/soong_ui" "$@"
fi
```



# 启动soong



SOONG_DEBUG参数是我们为调试手动添加的！！

```shell
source build/envsetup.sh
lunch XXX
SOONG_DEBUG=1 m 
```



当我们执行m脚本以后，soong将进入等待状态

```shell
➜  android-14.0.0_r73 SOONG_DEBUG=1 m 
参数 --build-mode 存在
进入调试模式
API server listening at: [::]:2345
2025-09-16T09:17:02+08:00 warn layer=rpc Listening for remote connections (connections are not authenticated nor encrypted)
```



# attch debugger 



1.使用Goland打开build/soong路径

2.配置一个remote debug configuration

![image-20250916092140901](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250916092140901.png)

3.设置断点 & 点击debug 按钮, 成功断点

![image-20250916225206044](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250916225206044.png)



