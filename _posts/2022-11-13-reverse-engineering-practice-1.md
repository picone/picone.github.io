---
layout: post
title: 逆向工程实战（一）
description: 从一个简单的安卓游戏入手，旨在介绍一些常用的逆向工具以及分析思路。
---

> 这里介绍的逆向的是一个安卓 APP，它被混淆了因此阅读起来有点困难。这里会介绍一些调试的小技巧。项目的源码
> [snake-ad-block](https://github.com/picone/snake-ad-block)。

# Requirement

- [Magisk](https://github.com/topjohnwu/Magisk)，用于安卓的 Root 权限管理。
- [LSPosed](https://github.com/LSPosed/LSPosed)、[EdXposed](https://github.com/ElderDrivers/EdXposed) 等 Xposed 框架，用于
可以 hook 入所需的函数。
- [dex2jar](https://github.com/pxb1988/dex2jar)，用于反编译 dex 文件到 jar，方便我们阅读。
- [JD-GUI](http://java-decompiler.github.io/)，jar 文件阅读，当然也可以使用 IDEA 等工具。

# 反编译 APK

1. 先获取 .apk 安装包，这个安装包务必保证版本和手机正在运行的一致。最准确的办法是直接拉取手机内的 apk 路径并 pull 到本地：
    ```shell
    # 获取完整的包名
    adb shell pm list packages | grep xxx
    # 通过包名获取完整的路径
    adb shell pm path xxx
    # 通过路径 pull apk 文件到本地
    adb pull xxx .
    ```
2. 反编译 apk 文件
    ```shell
    d2j-dex2jar.sh -f apk_to_decompile.apk
    ```
3. 打开 jar 文件。执行 d2j-dex2jar 后会生成一个 jar 文件，可以使用 JD-GUI 等工具打开。

# 功能实践

## 增加去广告功能

实现一个功能首先要找到这个功能的切入点才能沿着他的调用路径分析。通过 Charles 等抓包工具能定位到这个游戏使用的是小米广告联盟 SDK，对应 SDK 的文档
可以通过[官网](https://t5.a.market.xiaomi.com/download/AdCenter/0d3a369516ee146e8a9d5c290985939da4624fe0a/AdCenter0d3a369516ee146e8a9d5c290985939da4624fe0a.html)
找到，甚至能找到这个 SDK 的源码。熟读这个广告 SDK 后我们自然能找到他的使用方法。对于这个 SDK，根据广告样式能定位到这是个视频激励广告。使用方法：

```java
// 请求广告
RewardVideoAd rewardVideoAd = new RewardVideoAd();
rewardVideoAd.loadAd(PORTRAIT_POS_ID, new RewardVideoAd.RewardVideoLoadListener() {
    @Override
    public void onAdRequestSuccess() {
        //广告网络请求成功
    }

    @Override
    public void onAdLoadSuccess() {
        //广告素材缓存成功，可调用show展示广告
    }
    
    @Override
    public void onAdLoadFailed(int errorCode, String errorMsg) {
        //广告加载失败
    }
});
// 展示广告
rewardVideoAd.showAd(activity,new RewardVideoAd.RewardVideoInteractionListener());
// 销毁广告
rewardVideoAd.recycle();
```

所以，能直接 hook 进入这个方法的实现，屏蔽 showAd 方法，并主动执行 listener 的回调函数即可。

## 自定义长度

这个功能比较难调，因为没有现有可以参考的第三方插件作为切入点，只能通过代码一点点去调试。通过包名分析、猜测等方法大概定位到一些范围，还需要验证自己的
猜测。有一些小技巧：

### 验证是否执行到当前代码

只能通过最传统的方法，Hook 进行并打日志，然后使用 `Log.d()` 等方法打日志，使用 logcat 打印。

### 获取当前方法的调用堆栈

```
Log.d(LOG_TAG, Log.getStackTraceString(new Throwable()));
```

综上，拥有这些技能后，只需要耐心去分析代码，大部分 apk 都能分析。
