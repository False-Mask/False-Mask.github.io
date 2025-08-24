---
title: android-stack-unwind
tags:
cover:
---

# Android Stack Unwind



# frame pointer



依赖于编译器打开-fno-omit-frame-pointer

优点：快

缺点：在32位下由于有两个指令集A32/T32可能不准确



# DWARF CFI



CFI —— Call Frame Information

CFI信息被存在.eh_frame段。

