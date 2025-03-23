---
title: 记一次在AOSP上装Google Play的尝试
tags:
 - andorid
 - aosp
cover: https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/google-play.1024x1024.png
date: 2024-12-14 16:45:30
---





# 记一次在AOSP上装Google Play的尝试





## 结果

![失败GIF动图免费下载- 爱给网](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/46b680f2f72a429797a6e17098e9b7a9.gif)



# 背景

> 最近在工作中有需要用到 Google Play 来验证一些事情。
>
> 但是我手中的测试机Pixel5刷了 AOSP 是不具备 Google Play 的环境的。

> 经过多次调研 & 尝试在 AOSP 上安装 Google Play都已失败告终
>
> 1.NikeGapps（Pixel 5 不能刷 Recovery）
>
> 2.[bitgapps](https://bitgapps.io/)(Pixel 5 不能刷 Recovery)
>
> 3.[microGapps](https://microg.org/)（存在一些兼容问题）

> 在无意间发现了一篇文章[StackOverflow](https://stackoverflow.com/questions/41695566/install-google-apps-on-aosp-build)
>
> 文章大致内容是说可以通过提取官方镜像中的 APK，导入到 AOSP 中。
>
> 虽然这篇文章很早之前都看过，但是觉得比较麻烦，就没尝试。
>
> 本文章也算是最终尝试的记录。（要是还不行我真撂挑子了 这个 GP 谁爱装谁装了 : ）



# 过程



## 12.14日——失败



### 确认官方镜像的版本

通过Settings -> About Phone -> Build Number 可知AOSP 的版本

<img src="https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20241211232934789.png" alt="image-20241211232934789" style="zoom:25%;" />

我的是 TQ3A.230901.001.C2 其实也就是 android-13.0.0_r78 (Pixel 5)

经过确认可以下载的官方镜像版本是https://dl.google.com/dl/android/aosp/redfin-tq3a.230901.001.c2-factory-ca20bd02.zip?hl=zh-cn







### 提取APK



1.下载后我们得到一个压缩包，压缩包解压后如下。

<img src="https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20241211233127868.png" alt="image-20241211233127868" style="zoom: 50%;" />

2.再解压后如下

<img src="https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20241211233228358.png" alt="image-20241211233228358" style="zoom: 50%;" />

3.挂载 system 镜像

a. 查看文件类型

```shell
➜  image-redfin-tq3a.230901.001.c2 file system.img 
system.img: Linux rev 1.0 ext2 filesystem data, UUID=4f99442d-5b10-59c7-bafe-1ee733580bd7 (extents) (large files) (huge files)
```

b.挂载镜像(最好用 Linux 挂载)

```shell
sudo mount -o ro,loop system.img /mnt/system
```

> Note：
>
> 这里小小的踩了一个坑。Mac 针对于 ext文件系统支持不太好https://v2ex.com/t/658836
>
> “Linux 是最好的学习设备，不接受反驳”

c.提取

```shell
~ cd /mnt/system #进入挂载的系统镜像路径
~ cd system/ #进入系统路径
~ tar -cvf ~/Downloads/priv-app.tar priv-app #提取 apk
```

> 如下是system.img内的priv-app集合。but我找了以后发现没有我想要的google 3件套

![image-20241212001952936](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20241212001952936.png)



> 最后通过mnt后发现三件套主要在
>
> system_ext，product 分区。
>
> GoogleServicesFramework Phonesky PrebuiltGmsCore
>
> 提取流程跟上面是一样的



### 安装



1.remount

```shell
adb remount # syetem 分区是 readonly。需要 remount 才能修改
```

2.push

```shell
adb push GoogleServicesFramework Phonesky PrebuiltGmsCore /system/priv-app/
```



### bootloop



> 最终的结果很显然，bootloop了（循环死机）

> 很无奈。通过重新flash system分区修复了这个问题

```shell
fastboot flashing system system.img
```





### 最后



第一点从理论上我知道，google play肯定是可以刷的。但是结果上我失败了。（就是菜的缘故）

理论上应该是漏掉了一些东西，[priv-app特殊权限](https://source.android.com/docs/core/permissions/perms-allowlist?hl=zh-cn)

但是也给了我一些启发吧。毕竟实践出真理（追求真理的同时需要planB，实验过程中差点手机就成砖了，好不容易救回来了。haha～）

**“越是菜越是爱折腾，我觉得我很励志。”**



## 12.15日——成功



### pre



1. 首先总结之前失败的原因。主要是一点[priv-app特殊权限](https://source.android.com/docs/core/permissions/perms-allowlist?hl=zh-cn)

核心其实就一句话：在android 8.1以下特殊权限声明在system分区priv-app下，而在android 9以后特殊权限在system，product, vendor分区的priv-app下

特权应用是位于系统映像某个分区上 `priv-app` 目录下的系统应用。在各 Android 版本中，相应分区为：

- Android 9 及更高版本：`/system, /product, /vendor`
- Android 8.1 及更低版本：`/system`

2. 那接下来我们需要怎么做解决这个问题

其实很简单，我们bootloop是因为我们的gms三件套没有权限，我们把出厂镜像的权限加进去就好了。



### 提取permission权限文件



1.挂载system_ext，product img



2.进入etc/permissions

```shell
➜  product ls etc/permissions 
android.hardware.telephony.euicc.xml   com.android.sdm.plugins.connmo.xml   com.android.sdm.plugins.sprintdm.xml         com.google.android.apps.dreamliner.xml  com.google.android.odad.xml       com.verizon.apn.xml       lpa.xml                             RemoteSimlockManager.xml      UimGbaManager.xml    uimremoteserver.xml
com.android.imsserviceentitlement.xml  com.android.sdm.plugins.dcmo.xml     com.android.sdm.plugins.usccdm.xml           com.google.android.dialer.support.xml   com.google.omadm.trigger.xml      com.verizon.services.xml  privapp-permissions-google-p.xml    RemoteSimlock.xml             UimGba.xml           UimService.xml
com.android.omadm.service.xml          com.android.sdm.plugins.diagmon.xml  com.google.android.apps.diagnosticstool.xml  com.google.android.hardwareinfo.xml     com.google.SSRestartDetector.xml  features-verizon.xml      qti_telephony_hidl_wrapper_prd.xml  split-permissions-google.xml  uimremoteclient.xml  vendor.qti.hardware.data.connection-V1.0-java.xml
```



3.提取privapp-permissions-x.xml文件



### 修改& 上传



1.稍加改造

上述提到的permission文件是所有的app权限，我们肯定是用不到全部的。我们只需要使用几个就行了。（主要是gms三件套）

```java
// privapp-permissions-google-p.xml 

<permissions>

    <privapp-permissions package="com.android.vending">
        <permission name="android.permission.ALLOCATE_AGGRESSIVE"/>
        <permission name="android.permission.BACKUP"/>
        <permission name="android.permission.BATTERY_STATS"/>
        <permission name="android.permission.CHANGE_COMPONENT_ENABLED_STATE"/>
        <permission name="android.permission.CHANGE_DEVICE_IDLE_TEMP_WHITELIST"/>
        <permission name="android.permission.CHANGE_OVERLAY_PACKAGES"/>
        <permission name="android.permission.CLEAR_APP_CACHE"/>
        <permission name="android.permission.CONNECTIVITY_INTERNAL"/>
        <permission name="android.permission.DELETE_PACKAGES"/>
        <permission name="android.permission.DUMP"/>
        <permission name="android.permission.FORCE_STOP_PACKAGES"/>
        <permission name="android.permission.GET_ACCOUNTS_PRIVILEGED"/>
        <permission name="android.permission.GET_APP_OPS_STATS"/>
        <permission name="android.permission.INSTALL_PACKAGES"/>
        <permission name="android.permission.INTERACT_ACROSS_USERS"/>
        <permission name="android.permission.LOADER_USAGE_STATS"/>
        <permission name="android.permission.MANAGE_CLOUDSEARCH"/>
        <permission name="android.permission.MANAGE_ROLLBACKS"/>
        <permission name="android.permission.MANAGE_USERS"/>
        <permission name="android.permission.PACKAGE_USAGE_STATS"/>
        <permission name="android.permission.PACKAGE_VERIFICATION_AGENT"/>
        <permission name="android.permission.READ_PRIVILEGED_PHONE_STATE"/>
        <permission name="android.permission.READ_RUNTIME_PROFILES"/>
        <permission name="android.permission.REAL_GET_TASKS"/>
        <permission name="android.permission.REBOOT"/>
        <permission name="android.permission.SEND_DEVICE_CUSTOMIZATION_READY"/>
        <permission name="android.permission.SEND_SAFETY_CENTER_UPDATE"/>
        <permission name="android.permission.SEND_SMS_NO_CONFIRMATION"/>
        <permission name="android.permission.SET_PREFERRED_APPLICATIONS"/>
        <permission name="android.permission.START_ACTIVITIES_FROM_BACKGROUND"/>
        <permission name="android.permission.STATUS_BAR"/>
        <permission name="android.permission.SUBSTITUTE_NOTIFICATION_APP_NAME"/>
        <permission name="android.permission.UPDATE_DEVICE_STATS"/>
        <permission name="android.permission.WRITE_SECURE_SETTINGS"/>
        <permission name="com.android.permission.USE_INSTALLER_V2"/>
        <permission name="android.permission.OVERRIDE_COMPAT_CHANGE_CONFIG_ON_RELEASE_BUILD"/>
        <permission name="com.google.android.settings.setup.dock.RUN_DOCK_SETUP"/>
    </privapp-permissions>


    <privapp-permissions package="com.google.android.gms">
        <permission name="android.permission.ACCESS_BROADCAST_RESPONSE_STATS"/>
        <permission name="android.permission.ACCESS_CACHE_FILESYSTEM"/>
        <permission name="android.permission.ACCESS_CONTEXT_HUB"/>
        <permission name="android.permission.ACCESS_FPS_COUNTER"/>
        <permission name="android.permission.ACCESS_NETWORK_CONDITIONS"/>
        <permission name="android.permission.ACCESS_VIBRATOR_STATE"/>
        <permission name="android.permission.ACTIVITY_EMBEDDING"/>
        <permission name="android.permission.ALLOCATE_AGGRESSIVE"/>
        <permission name="android.permission.BACKUP"/>
        <permission name="android.permission.BLUETOOTH_PRIVILEGED"/>
        <permission name="android.permission.BROADCAST_CLOSE_SYSTEM_DIALOGS"/>
        <permission name="android.permission.CALL_PRIVILEGED"/>
        <permission name="android.permission.CAPTURE_AUDIO_HOTWORD"/>
        <permission name="android.permission.CAPTURE_AUDIO_OUTPUT"/>
        <permission name="android.permission.CAPTURE_SECURE_VIDEO_OUTPUT"/>
        <permission name="android.permission.CAPTURE_VIDEO_OUTPUT"/>
        <permission name="android.permission.CHANGE_COMPONENT_ENABLED_STATE"/>
        <permission name="android.permission.CHANGE_DEVICE_IDLE_TEMP_WHITELIST"/>
        <permission name="android.permission.COMPANION_APPROVE_WIFI_CONNECTIONS"/>
        <permission name="android.permission.CONNECTIVITY_USE_RESTRICTED_NETWORKS"/>
        <permission name="android.permission.CONTROL_INCALL_EXPERIENCE"/>
        <permission name="android.permission.CONTROL_DISPLAY_SATURATION"/>
        <permission name="android.permission.CONTROL_KEYGUARD_SECURE_NOTIFICATIONS"/>
        <permission name="android.permission.DISPATCH_PROVISIONING_MESSAGE"/>
        <permission name="android.permission.DOMAIN_VERIFICATION_AGENT"/>
        <permission name="android.permission.DUMP"/>
        <permission name="android.permission.GET_APP_OPS_STATS"/>
        <permission name="android.permission.INSTALL_LOCATION_TIME_ZONE_PROVIDER_SERVICE" />
        <permission name="android.permission.INTENT_FILTER_VERIFICATION_AGENT"/>
        <permission name="android.permission.INTERACT_ACROSS_USERS"/>
        <permission name="android.permission.INVOKE_CARRIER_SETUP"/>
        <permission name="android.permission.LOCAL_MAC_ADDRESS"/>
        <permission name="android.permission.LOCATION_BYPASS"/>
        <permission name="android.permission.LOCATION_HARDWARE"/>
        <permission name="android.permission.LOCK_DEVICE"/>
        <permission name="android.permission.MANAGE_DEVICE_ADMINS"/>
        <permission name="android.permission.MANAGE_FACTORY_RESET_PROTECTION" />
        <permission name="android.permission.MANAGE_GAME_ACTIVITY" />
        <permission name="android.permission.MANAGE_GAME_MODE" />
        <permission name="android.permission.MANAGE_ROLLBACKS"/>
        <permission name="android.permission.MANAGE_SOUND_TRIGGER"/>
        <permission name="android.permission.MANAGE_SUBSCRIPTION_PLANS"/>
        <permission name="android.permission.MANAGE_TIME_AND_ZONE_DETECTION"/>
        <permission name="android.permission.MANAGE_USB"/>
        <permission name="android.permission.MANAGE_USERS"/>
        <permission name="android.permission.MANAGE_VOICE_KEYPHRASES"/>
        <permission name="android.permission.MANAGE_WIFI_AUTO_JOIN"/>
        <permission name="android.permission.MANAGE_WIFI_NETWORK_SELECTION"/>
	<permission name="android.permission.MANAGE_WIFI_INTERFACES"/>
        <permission name="android.permission.MASTER_CLEAR"/>
        <permission name="android.permission.MEDIA_CONTENT_CONTROL"/>
        <permission name="android.permission.MODIFY_AUDIO_ROUTING"/>
        <permission name="android.permission.MODIFY_DAY_NIGHT_MODE"/>
        <permission name="android.permission.MODIFY_DEFAULT_AUDIO_EFFECTS"/>
        <permission name="android.permission.MODIFY_NETWORK_ACCOUNTING"/>
        <permission name="android.permission.MODIFY_PHONE_STATE"/>
        <permission name="android.permission.NOTIFY_PENDING_SYSTEM_UPDATE"/>
        <permission name="android.permission.OBSERVE_GRANT_REVOKE_PERMISSIONS"/>
        <permission name="android.permission.OVERRIDE_WIFI_CONFIG"/>
        <permission name="android.permission.PACKAGE_USAGE_STATS"/>
        <permission name="android.permission.PROVIDE_RESOLVER_RANKER_SERVICE" />
        <permission name="android.permission.PROVIDE_TRUST_AGENT"/>
        <permission name="android.permission.READ_DREAM_STATE"/>
        <permission name="android.permission.READ_LOGS"/>
        <permission name="android.permission.READ_NEARBY_STREAMING_POLICY"/>
        <permission name="android.permission.READ_NETWORK_USAGE_HISTORY"/>
        <permission name="android.permission.READ_OEM_UNLOCK_STATE"/>
        <permission name="android.permission.READ_PRIVILEGED_PHONE_STATE"/>
        <permission name="android.permission.READ_WIFI_CREDENTIAL"/>
        <permission name="android.permission.REAL_GET_TASKS"/>
        <permission name="android.permission.REBOOT"/>
        <permission name="android.permission.RECEIVE_DATA_ACTIVITY_CHANGE"/>
        <permission name="android.permission.RECOVER_KEYSTORE"/>
        <permission name="android.permission.RECOVERY"/>
        <permission name="android.permission.REGISTER_CALL_PROVIDER"/>
        <permission name="android.permission.REMOTE_DISPLAY_PROVIDER"/>
        <permission name="android.permission.RENOUNCE_PERMISSIONS"/>
        <permission name="android.permission.REQUEST_COMPANION_PROFILE_COMPUTER"/>
        <permission name="android.permission.REQUEST_COMPANION_SELF_MANAGED"/>
        <permission name="android.permission.RESET_PASSWORD"/>
        <permission name="android.permission.SCHEDULE_PRIORITIZED_ALARM"/>
        <permission name="android.permission.SCORE_NETWORKS"/>
        <permission name="android.permission.SEND_SAFETY_CENTER_UPDATE"/>
        <permission name="android.permission.SEND_SMS_NO_CONFIRMATION"/>
        <permission name="android.permission.SET_TIME"/>
        <permission name="android.permission.SET_TIME_ZONE"/>
        <permission name="android.permission.START_ACTIVITIES_FROM_BACKGROUND"/>
        <permission name="android.permission.START_TASKS_FROM_RECENTS"/>
        <permission name="android.permission.STATUS_BAR"/>
        <permission name="android.permission.SUBSTITUTE_NOTIFICATION_APP_NAME"/>
        <permission name="android.permission.SUBSTITUTE_SHARE_TARGET_APP_NAME_AND_ICON"/>
        <permission name="android.permission.TETHER_PRIVILEGED"/>
        <permission name="android.permission.UPDATE_APP_OPS_STATS"/>
        <permission name="android.permission.UPDATE_DEVICE_STATS"/>
        <permission name="android.permission.UPDATE_FONTS" />
        <permission name="android.permission.USE_RESERVED_DISK"/>
        <permission name="android.permission.USER_ACTIVITY"/>
        <permission name="android.permission.UWB_PRIVILEGED"/>
        <permission name="android.permission.WRITE_GSERVICES"/>
        <permission name="android.permission.WRITE_SECURE_SETTINGS"/>
        <permission name="com.android.voicemail.permission.READ_VOICEMAIL"/>
    </privapp-permissions>

</permissions>
    
// privapp-permissions-google-se.xml
    <permissions>

    <privapp-permissions package="com.google.android.gsf">
        <permission name="android.permission.ACCESS_CACHE_FILESYSTEM"/>
        <permission name="android.permission.BACKUP"/>
        <permission name="android.permission.CHANGE_COMPONENT_ENABLED_STATE"/>
        <permission name="android.permission.DUMP"/>
        <permission name="android.permission.INTERACT_ACROSS_USERS"/>
        <permission name="android.permission.INVOKE_CARRIER_SETUP"/>
        <permission name="android.permission.MANAGE_USERS"/>
        <permission name="android.permission.MASTER_CLEAR"/>
        <permission name="android.permission.READ_DREAM_STATE"/>
        <permission name="android.permission.READ_LOGS"/>
        <permission name="android.permission.READ_NETWORK_USAGE_HISTORY"/>
        <permission name="android.permission.REBOOT"/>
        <permission name="android.permission.RECEIVE_DATA_ACTIVITY_CHANGE"/>
        <permission name="android.permission.RECOVERY"/>
        <permission name="android.permission.SET_TIME"/>
        <permission name="android.permission.STATUS_BAR"/>
        <permission name="android.permission.UPDATE_DEVICE_STATS"/>
        <permission name="android.permission.WRITE_GSERVICES"/>
        <permission name="android.permission.WRITE_SECURE_SETTINGS"/>
    </privapp-permissions>


</permissions>
```



2.拉取aosp xml文件修改上传到/etc/permissions



3.reboot



## 2025/3/23——补充



> 最近几天在我自己的Pixel6手机上刷入了aosp 14.0.0_r71系统。
>
> 提取指定BuildId的Pixel 镜像中的GMS后发现。虽然不会死机了
>
> but会出现崩溃。主要是权限的问题。

![image-20250323183244572](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20250323183244572.png)

``` plaintText
Process: com.android.vending:background, PID: 17274
                                                                                                    java.lang.RuntimeException: Unable to start service com.google.android.finsky.downloadservice.DownloadService@cf51d31 with Intent { act=com.google.android.finsky.BIND_DOWNLOAD_SERVICE pkg=com.android.vending (has extras) }: java.lang.SecurityException: Starting FGS with type systemExempted callerApp=ProcessRecord{9ada412 17274:com.android.vending:background/u0a128} targetSDK=35 requires permissions: all of the permissions allOf=true [android.permission.FOREGROUND_SERVICE_SYSTEM_EXEMPTED] any of the permissions allOf=false [android.permission.SCHEDULE_EXACT_ALARM, android.permission.USE_EXACT_ALARM, android:activate_vpn] 
                                                                                                    	at android.app.ActivityThread.handleServiceArgs(ActivityThread.java:5100)
                                                                                                    	at android.app.ActivityThread.-$$Nest$mhandleServiceArgs(Unknown Source:0)
                                                                                                    	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2432)
                                                                                                    	at android.os.Handler.dispatchMessage(Handler.java:107)
                                                                                                    	at android.os.Looper.loopOnce(Looper.java:232)
                                                                                                    	at android.os.Looper.loop(Looper.java:317)
                                                                                                    	at android.app.ActivityThread.main(ActivityThread.java:8592)
                                                                                                    	at java.lang.reflect.Method.invoke(Native Method)
                                                                                                    	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:580)
                                                                                                    	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:878)
                                                                                                    Caused by: java.lang.SecurityException: Starting FGS with type systemExempted callerApp=ProcessRecord{9ada412 17274:com.android.vending:background/u0a128} targetSDK=35 requires permissions: all of the permissions allOf=true [android.permission.FOREGROUND_SERVICE_SYSTEM_EXEMPTED] any of the permissions allOf=false [android.permission.SCHEDULE_EXACT_ALARM, android.permission.USE_EXACT_ALARM, android:activate_vpn] 
                                                                                                    	at android.os.Parcel.createExceptionOrNull(Parcel.java:3183)
                                                                                                    	at android.os.Parcel.createException(Parcel.java:3167)
                                                                                                    	at android.os.Parcel.readException(Parcel.java:3150)
                                                                                                    	at android.os.Parcel.readException(Parcel.java:3092)
                                                                                                    	at android.app.IActivityManager$Stub$Proxy.setServiceForeground(IActivityManager.java:6960)
                                                                                                    	at android.app.Service.startForeground(Service.java:776)
                                                                                                    	at com.google.android.finsky.downloadservice.DownloadService.onStartCommand(PG:37)
                                                                                                    	at android.app.ActivityThread.handleServiceArgs(ActivityThread.java:5082)
                                                                                                    	at android.app.ActivityThread.-$$Nest$mhandleServiceArgs(Unknown Source:0) 
                                                                                                    	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2432) 
                                                                                                    	at android.os.Handler.dispatchMessage(Handler.java:107) 
                                                                                                    	at android.os.Looper.loopOnce(Looper.java:232) 
                                                                                                    	at android.os.Looper.loop(Looper.java:317) 
                                                                                                    	at android.app.ActivityThread.main(ActivityThread.java:8592) 
                                                                                                    	at java.lang.reflect.Method.invoke(Native Method) 
                                                                                                    	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:580) 
                                                                                                    	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:878) 
                                                                                                    Caused by: android.os.RemoteException: Remote stack trace:
                                                                                                    	at com.android.server.am.ActiveServices.validateForegroundServiceType(ActiveServices.java:2842)
                                                                                                    	at com.android.server.am.ActiveServices.setServiceForegroundInnerLocked(ActiveServices.java:2530)
                                                                                                    	at com.android.server.am.ActiveServices.setServiceForegroundLocked(ActiveServices.java:1806)
                                                                                                    	at com.android.server.am.ActivityManagerService.setServiceForeground(ActivityManagerService.java:13795)
                                                                                                    	at android.app.IActivityManager$Stub.onTransact(IActivityManager.java:3483)
```



解决方法如下。

- 通过xargs需要新增权限

``` xml
// privapp-permissions-google-p.xml   
<permission name="android.permission.USE_EXACT_ALARM"/>
<permission name="android.permission.SCHEDULE_EXACT_ALARM"/>
<permission name="android.permission.FOREGROUND_SERVICE_SYSTEM_EXEMPTED"/>
```

- 通过appops赋权

``` shell
# 赋予ACTIVATE_VPN权限。
adb shell appops set com.android.vending ACTIVATE_VPN  allow
```

- 清空Google play & GMS 应用数据 & 强杀进程

  







### 其他问题



安装后进入google play会有一个问题。：“[this device isn't play protect certified](https://support.google.com/android/answer/7165974?hl=en#zippy=%2Cdevice-isnt-certified)“

简单来说就是我们的设备没有认证（这里的认证是厂商认证）



Google Play需要厂商认证才能用，这里是厂商认证的列表[厂商认证的列表](https://support.google.com/android/answer/7165974?hl=en#zippy=%2Cdevice-isnt-certified)



我们开发者自己打的AOSP有机会用到吗？答案是有的

google play给开发者留了一个空子，注册设备Id后就可以使用啦～

[注册地址](https://www.google.com/android/uncertified)





# 最后

> La fanfare frémit au carrefour de ta forme
>
> 当你打开束缚你的牢笼
>
> Martellant sa poésie diforme 
>
> 我们将去乌托邦
>
> ——2024-12/14 

# refs



[挂载 system 文件](https://blog.csdn.net/netwalk/article/details/140108965)

[android官方文档-priv-apps](https://source.android.com/docs/core/permissions/perms-allowlist?hl=zh-cn)