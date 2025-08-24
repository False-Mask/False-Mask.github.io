---
title: android-ebpf-develop-env
tags:
cover:
---

# Android ebpf开发环境搭建

前置条件：Andorid ebpf运行环境配置完成



# BCC

在Android上运行vscode-server

通过远程ssh连接

## vscode-remote环境搭建



see vscode开发环境搭建



## vscode插件下载



![image-20250726142235947](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250726142235947.png)



## 配置Python解释器



![image-20250726142320439](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250726142320439.png)



## 配置settings文件



``` json
{
    // python language server选用pylance
    "python.languageServer": "Pylance",
    "python-envs.pythonProjects": [
    ],
    // 添加bcc的源码路径。
    // 此处笔者已经将egg文件提前解压好了
    "python.analysis.extraPaths": [
        "/usr/lib/python3/dist-packages/bcc-0.35.0b2+14bbc93d-py3.13"
    ]
}
```



## End

![image-20250726142619705](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250726142619705.png)

现在就可以开始愉快的代码开发了～
