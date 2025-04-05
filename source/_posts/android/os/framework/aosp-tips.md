---
title: Aosp小tips
tags:
  - aosp
date: 2025-03-23 23:41:17
cover: https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/aosp-andvanced-development-tricks.png
---


# 关于本文



> 主要记录在AOSP调试学习过程中遇到的坑～

![aosp-andvanced-development-tricks](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/aosp-andvanced-development-tricks.png)



# Android 14_r73



## userdebug 编译out of space? out of inodes? the tree size XXX 



- 问题：

> out of space? out of inodes?XX
>
> 这个报错包含有3个原因。
>
> 1. 内存耗尽
> 2. inodes耗尽
> 3. 生成的文件超过文件系统的大小限制
>
> 
>
> ***这里的报错主要是因为#3导致，原因如下：***
>
> img文件生成目前主要有两种方式ex4和f2fs, 图中的报错主要是由于userdata默认采用了f2fs导致大小超过了f2fs的最大限制，需要***调整文件类型为ext4***并且将***文件的大小限制设置为合适的大小***

![Image_410772202840384](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/Image_410772202840384.jpg)

- 解决方式

https://juejin.cn/post/7484089347156295714



> note：
>
> 1. 报错原因有三个！！需要明确原因。不能盲目抄
> 2. 需要明确报错的文件，每个img类型都有不同的参数控制
>
> 3. 需要明确lunch type,不同的lunch type可能配置的参数不一样！！



## userdebug无法调试的问题





- 问题：

1.现象一

``` shell
# 当我们调用adb jdwp查看可调式的应用时会一直卡住。（）
➜  ~  adb jdwp
^C
```

2.现象二

> 设备能被识别，当时attach process找不到设备（右上角可见设备名称）

<img src="https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/Screenshot from 2025-03-23 23-23-12.png" alt="Screenshot from 2025-03-23 23-23-12" />



- 解决方案：

https://www.bilibili.com/opus/911162975264440345





## Winscope编译问题



问题描述：

>  依据README给出的构建流程最终会有一个报错

``` plaintText
➜  winscope git:(android-14.0.0_r73) ✗ npm run build:prod

> winscope@0.0.0 build:prod
> npm run build:trace_processor && npm run build:protos && rm -rf dist/prod/ && webpack --config webpack.config.prod.js --progress && cp deps_build/trace_processor/to_be_served/* src/adb/winscope_proxy.py dist/prod/


> winscope@0.0.0 build:trace_processor
> PERFETTO_TOP=../../../external/perfetto; (cd $PERFETTO_TOP && tools/install-build-deps --ui && ui/node ui/build.js --out trace_processor_build) && rm -rf deps_build/trace_processor && mkdir -p deps_build/trace_processor && rsync -ar $PERFETTO_TOP/trace_processor_build/ deps_build/trace_processor && mkdir deps_build/trace_processor/to_be_served && cp deps_build/trace_processor/ui/dist_version/engine_bundle.js deps_build/trace_processor/to_be_served/ && cp deps_build/trace_processor/wasm/trace_processor.wasm deps_build/trace_processor/to_be_served/

INFO:root:Running `pnpm install --shamefully-hoist --frozen-lockfile` in /Users/rose/aosp/android-14.0.0_r73/external/perfetto/ui
Lockfile is up to date, resolution step is skipped
Packages: +1
+
Progress: resolved 1, reused 0, downloaded 1, added 1, done
Done in 433ms
rm /Users/rose/aosp/android-14.0.0_r73/external/perfetto/trace_processor_build/ui
Entering /Users/rose/aosp/android-14.0.0_r73/external/perfetto/trace_processor_build
[00.487] 1/63   Waiting for first build to complete... 1 s
[00.487] 2/63   ../tools/gn gen --args=is_debug=false 
Done. Made 1083 targets from 226 files in 33ms
[00.596] 3/63   ../tools/ninja -C  trace_processor_wasm traceconv_wasm
ninja: Entering directory `/Users/rose/aosp/android-14.0.0_r73/external/perfetto/trace_processor_build'
ninja: no work to do.
[00.709] 4/63   cp wasm/trace_processor.wasm ui/dist/v42.0-7c79c3ab0/trace_processor.wasm
[00.714] 5/63   cp wasm/trace_processor.js ui/tsc/gen/trace_processor.js
[00.714] 6/63   cp wasm/trace_processor.d.ts ui/tsc/gen/trace_processor.d.ts
[00.714] 7/63   cp wasm/traceconv.wasm ui/dist/v42.0-7c79c3ab0/traceconv.wasm
[00.719] 8/63   cp wasm/traceconv.js ui/tsc/gen/traceconv.js
[00.720] 9/63   cp wasm/traceconv.d.ts ui/tsc/gen/traceconv.d.ts
[00.720] 10/63  cp ../ui/src/assets/brand.png ui/dist/v42.0-7c79c3ab0/assets/brand.png
[00.720] 11/63  sass --quiet ../ui/src/assets/perfetto.scss ui/dist/v42.0-7c79c3ab0/perfetto.css
[01.072] 12/63  cp ../ui/src/assets/favicon.png ui/dist/v42.0-7c79c3ab0/assets/favicon.png
[01.073] 13/63  cpHtml ../ui/src/assets/index.html index.html
[01.073] 14/63  cp ../ui/src/assets/logo-128.png ui/dist/v42.0-7c79c3ab0/assets/logo-128.png
[01.073] 15/63  cp ../ui/src/assets/logo-3d.png ui/dist/v42.0-7c79c3ab0/assets/logo-3d.png
[01.073] 16/63  cp ../ui/src/assets/rec_atrace.png ui/dist/v42.0-7c79c3ab0/assets/rec_atrace.png
[01.073] 17/63  cp ../ui/src/assets/rec_battery_counters.png ui/dist/v42.0-7c79c3ab0/assets/rec_
[01.074] 18/63  cp ../ui/src/assets/rec_board_voltage.png ui/dist/v42.0-7c79c3ab0/assets/rec_boa
[01.074] 19/63  cp ../ui/src/assets/rec_cpu_coarse.png ui/dist/v42.0-7c79c3ab0/assets/rec_cpu_co
[01.074] 20/63  cp ../ui/src/assets/rec_cpu_fine.png ui/dist/v42.0-7c79c3ab0/assets/rec_cpu_fine
[01.074] 21/63  cp ../ui/src/assets/rec_cpu_freq.png ui/dist/v42.0-7c79c3ab0/assets/rec_cpu_freq
[01.074] 22/63  cp ../ui/src/assets/rec_cpu_voltage.png ui/dist/v42.0-7c79c3ab0/assets/rec_cpu_v
[01.074] 23/63  cp ../ui/src/assets/rec_frame_timeline.png ui/dist/v42.0-7c79c3ab0/assets/rec_fr
[01.074] 24/63  cp ../ui/src/assets/rec_ftrace.png ui/dist/v42.0-7c79c3ab0/assets/rec_ftrace.png
[01.074] 25/63  cp ../ui/src/assets/rec_gpu_mem_total.png ui/dist/v42.0-7c79c3ab0/assets/rec_gpu
[01.074] 26/63  cp ../ui/src/assets/rec_java_heap_dump.png ui/dist/v42.0-7c79c3ab0/assets/rec_ja
[01.075] 27/63  cp ../ui/src/assets/rec_lmk.png ui/dist/v42.0-7c79c3ab0/assets/rec_lmk.png
[01.075] 28/63  cp ../ui/src/assets/rec_logcat.png ui/dist/v42.0-7c79c3ab0/assets/rec_logcat.png
[01.075] 29/63  cp ../ui/src/assets/rec_long_trace.png ui/dist/v42.0-7c79c3ab0/assets/rec_long_t
[01.075] 30/63  cp ../ui/src/assets/rec_mem_hifreq.png ui/dist/v42.0-7c79c3ab0/assets/rec_mem_hi
[01.075] 31/63  cp ../ui/src/assets/rec_meminfo.png ui/dist/v42.0-7c79c3ab0/assets/rec_meminfo.p
[01.075] 32/63  cp ../ui/src/assets/rec_native_heap_profiler.png ui/dist/v42.0-7c79c3ab0/assets/
[01.076] 33/63  cp ../ui/src/assets/rec_one_shot.png ui/dist/v42.0-7c79c3ab0/assets/rec_one_shot
[01.076] 34/63  cp ../ui/src/assets/rec_profiling.png ui/dist/v42.0-7c79c3ab0/assets/rec_profili
[01.076] 35/63  cp ../ui/src/assets/rec_ps_stats.png ui/dist/v42.0-7c79c3ab0/assets/rec_ps_stats
[01.076] 36/63  cp ../ui/src/assets/rec_ring_buf.png ui/dist/v42.0-7c79c3ab0/assets/rec_ring_buf
[01.076] 37/63  cp ../ui/src/assets/rec_syscalls.png ui/dist/v42.0-7c79c3ab0/assets/rec_syscalls
[01.076] 38/63  cp ../ui/src/assets/rec_vmstat.png ui/dist/v42.0-7c79c3ab0/assets/rec_vmstat.png
[01.076] 39/63  cp ../ui/src/assets/scheduling_latency.png ui/dist/v42.0-7c79c3ab0/assets/schedu
[01.077] 40/63  cp ../ui/src/assets/logo-128.png ui/chrome_extension/logo-128.png
[01.077] 41/63  cp ../ui/src/chrome_extension/manifest.json ui/chrome_extension/manifest.json
[01.077] 42/63  cp ../ui/src/test/diff_viewer/index.html ui-test-artifacts/index.html
[01.077] 43/63  cp ../ui/src/test/diff_viewer/script.js ui-test-artifacts/script.js
[01.077] 44/63  cp ../buildtools/typefaces/._MaterialSymbolsOutlined.woff2 ui/dist/v42.0-7c79c3a
[01.077] 45/63  cp ../buildtools/typefaces/MaterialSymbolsOutlined.woff2 ui/dist/v42.0-7c79c3ab0
[01.079] 46/63  cp ../buildtools/typefaces/Roboto-100.woff2 ui/dist/v42.0-7c79c3ab0/assets/Robot
[01.079] 47/63  cp ../buildtools/typefaces/Roboto-300.woff2 ui/dist/v42.0-7c79c3ab0/assets/Robot
[01.079] 48/63  cp ../buildtools/typefaces/Roboto-400.woff2 ui/dist/v42.0-7c79c3ab0/assets/Robot
[01.079] 49/63  cp ../buildtools/typefaces/Roboto-500.woff2 ui/dist/v42.0-7c79c3ab0/assets/Robot
[01.079] 50/63  cp ../buildtools/typefaces/RobotoCondensed-Light.woff2 ui/dist/v42.0-7c79c3ab0/a
[01.079] 51/63  cp ../buildtools/typefaces/RobotoCondensed-Regular.woff2 ui/dist/v42.0-7c79c3ab0
[01.079] 52/63  cp ../buildtools/typefaces/RobotoMono-Regular.woff2 ui/dist/v42.0-7c79c3ab0/asse
[01.079] 53/63  cp ../buildtools/catapult_trace_viewer/catapult_trace_viewer.html ui/dist/v42.0-
[01.080] 54/63  cp ../buildtools/catapult_trace_viewer/catapult_trace_viewer.js ui/dist/v42.0-7c
[01.081] 55/63  python3 ../tools/gen_ui_imports ../ui/src/tracks --out ../ui/src/gen/all_tracks.
[01.157] 56/63  python3 ../tools/gen_ui_imports ../ui/src/plugins --out ../ui/src/gen/all_plugin
[01.231] 57/63  pbjs --no-beautify --force-number --no-delimited --no-verify -t static-module -w
[02.366] 58/63  pbts --no-comments -p .. -o ui/tsc/gen/protos.d.ts ui/tsc/gen/protos.js
[16.294] 59/63  python3 ../tools/write_version_header.py --ts_out ui/tsc/gen/perfetto_version.ts
[16.388] 60/63  tsc --project ../ui
[23.376] 61/63  tsc --project ../ui/src/service_worker
[23.896] 62/63  rollup -c ../ui/config/rollup.config.js --no-indent --silent
[50.313] 63/63  makeManifest

> winscope@0.0.0 build:protos
> node protos/build.js

node:internal/process/promises:279
            triggerUncaughtException(err, true /* fromPromise */);
            ^

[UnhandledPromiseRejection: This error originated either by throwing inside of an async function without a catch block, or by rejecting a promise which was not handled with .catch(). The promise rejected with the reason "Failed to execute command

command: npx pbjs --force-long --target json-module --wrap es6 --out /Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/protos/../deps_build/protos/windowmanager/latest/json.js --root windowmanager_latest --path /Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/protos/../../../../../external/perfetto --path /Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/protos/.. --path /Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/protos/../../../../.. /Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/protos/../../../../frameworks/base/core/proto/android/server/windowmanagertrace.proto

stdout: 

stderr: /Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/node_modules/protobufjs-cli/pbjs.js:254
            throw err;
            ^

Error: no such Type or Enum 'WindowManagerServiceDumpProto' in Type .com.android.server.wm.WindowManagerTraceProto
    at Type.lookupTypeOrEnum (/Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/node_modules/protobufjs/src/namespace.js:410:15)
    at Field.resolve (/Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/node_modules/protobufjs/src/field.js:268:94)
    at Type.set (/Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/node_modules/protobufjs/src/type.js:177:38)
    at Type.get (/Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/node_modules/protobufjs/src/type.js:155:45)
    at Field.resolve (/Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/node_modules/protobufjs/src/field.js:320:21)
    at Type.resolveAll (/Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/node_modules/protobufjs/src/type.js:304:21)
    at Namespace.resolveAll (/Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/node_modules/protobufjs/src/namespace.js:307:25)
    at Namespace.resolveAll (/Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/node_modules/protobufjs/src/namespace.js:307:25)
    at Namespace.resolveAll (/Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/node_modules/protobufjs/src/namespace.js:307:25)
    at Namespace.resolveAll (/Users/rose/aosp/android-14.0.0_r73/development/tools/winscope/node_modules/protobufjs/src/namespace.js:307:25)
".] {
  code: 'ERR_UNHANDLED_REJECTION'
```



> 解决方法：
>
> https://blog.csdn.net/fromair/article/details/145961344

