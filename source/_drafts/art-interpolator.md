---
title: art-interpretor
tags:
cover:
---



# Android ART 解释器



# 基础说明



源码路径: art/runtime/interpreter/



# genrule分析



如果你打开art/runtime/interpreter/mterp这个路径那么你一定会看到一个readme文件

art/runtime/interpreter/mterp/README.txt

他会告诉你，art的assembly文件不是标准的语法格式，而是通过一些脚本进行过扩展的。

而这里提到的脚本就是通过genrule配置的。





