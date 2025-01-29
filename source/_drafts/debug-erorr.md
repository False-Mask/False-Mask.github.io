---
title: debug-erorr
tags:
cover:
---



# Debug Error



 failed to connect to socket 'localabstract:/com.example.myapplication-0/platform-1737977988951.sock': could not connect to localabstract address 'localabstract:/com.example.myapplication-0/platform-1737977988951.sock'



# Debug实现原理



1. 通过opensnoop 查看指令的执行

```shell
root@localhost:/# cat ~/exec.log 
sh               27131   1253      0 /system/bin/sh -c wm dismiss-keyguard
PCOMM            PID     PPID    RET ARGS
sh               27133   1253      0 /system/bin/sh -c wm dismiss-keyguard
wm               27131   1253      0 /system/bin/wm dismiss-keyguard
wm               27133   1253      0 /system/bin/wm dismiss-keyguard
cmd              27135   27131     0 /system/bin/cmd window dismiss-keyguard
cmd              27136   27133     0 /system/bin/cmd window dismiss-keyguard
sh               27141   1253      0 /system/bin/sh -c am start -n com.example.myapplication/com.example.myapplication.MainActivity -a android.intent.action.MAIN -c android.intent.cat
am               27141   1253      0 /system/bin/am start -n com.example.myapplication/com.example.myapplication.MainActivity -a android.intent.action.MAIN -c android.intent.category.LAUNCHER -D --splashscreen-show-icon
cmd              27143   27141     0 /system/bin/cmd activity start -n com.example.myapplication/com.example.myapplication.MainActivity -a android.intent.action.MAIN -c android.intent.category.LAUNCHER -D --splashscreen-show-icon
sh               27155   1253      0 /system/bin/sh -c getprop
getprop          27155   1253      0 /system/bin/getprop
sh               27165   1253      0 /system/bin/sh -c getprop
getprop          27165   1253      0 /system/bin/getprop
sh               27169   1253      0 /system/bin/sh -c echo $USER_ID
sh               27171   1253      0 /system/bin/sh -c run-as com.example.myapplication getprop ro.product.model
run-as           27171   1253      0 /system/bin/run-as com.example.myapplication getprop ro.product.model
getprop          27171   1253      0 /system/bin/getprop ro.product.model
sh               27175   1253      0 /system/bin/sh -c run-as com.example.myapplication sh -c 'mkdir -p /data/data/com.example.myapplication/lldb; mkdir -p /data/data/com.example.myap
run-as           27177   27175     0 /system/bin/run-as com.example.myapplication sh -c mkdir -p /data/data/com.example.myapplication/lldb; mkdir -p /data/data/com.example.myapplication/lldb/bin
sh               27177   27175     0 /system/bin/sh -c mkdir -p /data/data/com.example.myapplication/lldb; mkdir -p /data/data/com.example.myapplication/lldb/bin
mkdir            27178   27177     0 /system/bin/mkdir -p /data/data/com.example.myapplication/lldb
mkdir            27179   27177     0 /system/bin/mkdir -p /data/data/com.example.myapplication/lldb/bin
sh               27180   1253      0 /system/bin/sh -c cat /data/local/tmp/lldb-server | run-as com.example.myapplication sh -c 'cat > /data/data/com.example.myapplication/lldb/bin/ll
cat              27182   27180     0 /system/bin/cat /data/local/tmp/lldb-server
run-as           27183   27180     0 /system/bin/run-as com.example.myapplication sh -c cat > /data/data/com.example.myapplication/lldb/bin/lldb-server && chmod 700 /data/data/com.example.myapplication/lldb/bin/lldb-
sh               27183   27180     0 /system/bin/sh -c cat > /data/data/com.example.myapplication/lldb/bin/lldb-server && chmod 700 /data/data/com.example.myapplication/lldb/bin/lldb-
sh               27184   1253      0 /system/bin/sh -c cat /data/local/tmp/start_lldb_server.sh | run-as com.example.myapplication sh -c 'cat > /data/data/com.example.myapplication/ll
cat              27186   27184     0 /system/bin/cat /data/local/tmp/start_lldb_server.sh
run-as           27187   27184     0 /system/bin/run-as com.example.myapplication sh -c cat > /data/data/com.example.myapplication/lldb/bin/start_lldb_server.sh && chmod 700 /data/data/com.example.myapplication/lldb/
sh               27187   27184     0 /system/bin/sh -c cat > /data/data/com.example.myapplication/lldb/bin/start_lldb_server.sh && chmod 700 /data/data/com.example.myapplication/lldb/
cat              27188   27187     0 /system/bin/cat
chmod            27189   27187     0 /system/bin/chmod 700 /data/data/com.example.myapplication/lldb/bin/start_lldb_server.sh
sh               27190   1253      0 /system/bin/sh -c run-as com.example.myapplication /data/data/com.example.myapplication/lldb/bin/start_lldb_server.sh /data/data/com.example.myapp
run-as           27192   27190     0 /system/bin/run-as com.example.myapplication /data/data/com.example.myapplication/lldb/bin/start_lldb_server.sh /data/data/com.example.myapplication/lldb unix-abstract /com.example.myapplication-0 platform-1737979821597.sock lldb process:gdb-remote packets
start_lldb_serv  27192   27190     0 /data/data/com.example.myapplication/lldb/bin/start_lldb_server.sh /data/data/com.example.myapplication/lldb unix-abstract /com.example.myapplication-0 platform-1737979821597.sock lldb process:gdb-remote packets
chmod            27193   27192     0 /system/bin/chmod 0775 /data/data/com.example.myapplication/lldb
rm               27194   27192     0 /system/bin/rm -r /data/data/com.example.myapplication/lldb/tmp
mkdir            27195   27192     0 /system/bin/mkdir /data/data/com.example.myapplication/lldb/tmp
rm               27196   27192     0 /system/bin/rm -r /data/data/com.example.myapplication/lldb/log
mkdir            27197   27192     0 /system/bin/mkdir /data/data/com.example.myapplication/lldb/log
cat              27198   27192     0 /system/bin/cat
lldb-server      27199   27192     0 /data/data/com.example.myapplication/lldb/bin/lldb-server platform --server --listen unix-abstract:///com.example.myapplication-0/platform-1737979821597.sock --log-file /data/data/com.example.myapplication/lldb/log/platform.log --log-channels lldb process:gdb-remote packets
lldb-server      27201   27200     0 /data/data/com.example.myapplication/lldb/bin/lldb-server gdbserver unix-abstract:///com.example.myapplication-0/gdbserver.9a2200 --native-regs --pipe 7 --log-file=/data/data/com.example.myapplication/lldb/log/gdb-server.log --log-channels=lldb process:gdb-remote packets
sh               27203   1253      0 /system/bin/sh -c getprop
getprop          27203   1253      0 /system/bin/getprop
sh               27205   1253      0 /system/bin/sh -c getprop
getprop          27205   1253      0 /system/bin/getprop

```

