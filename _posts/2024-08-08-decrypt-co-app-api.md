---
layout: post
title: 破解某克 APP 的 API 接口
description: 某克的 APP 使用了一套 API 接口，抓包查看有 signature 参数，这篇文章将介绍如何破解这个参数
---

# 抓包

我使用的是安卓 APP，对应接口使用的是 HTTPS，因此抓包会有些许麻烦。

1. 从 Android 7 开始，应用就可以在 Manifest 文件声明就可以禁止自签证书了，这就拦下 99% 的人了。为了抵制这个方法，可以 root，然后把自签
  证书放到系统根证书目录蒙混过关，这样就可以跳过这个限制，能绕过安卓系统自身判断系统证书的限制。

2. 然而，有些应用会使用 SSL Pinning，这个是应用自己写的代码，会检查证书的公钥，如果不是预期的公钥，就会拒绝连接。这个时候最简单的方法应该是
  使用 Xposed 框架，使用 [JustTrustMe](https://github.com/Fuzion24/JustTrustMe) 模块即可绕过 SSL Pinning。

3. 然而，鸡贼的某克还加了一个限制，动态去检测 Xposed，如果使用了 Xposed 则整个 App 会 crash。我使用的 Xposed 方案是 Magisk，尝试使用
  Magisk hide 等方法无果，他依然可以检测出来。这时候找到一个叫 [Shamiko](https://www.magiskmodule.com/shamiko-magisk-module/)
  的模块可以隐藏 Magisk 的各种特征，尝试后终于可以正常打开某克 APP 了。

经过这些叠加的限制，我们终于可以抓包了。抓出的包如这个例子：

```shell
curl -X GET 'https://app-services.xxx.com.cn/auth/login/refresh?deviceId=xxx&refreshToken=xxx' \
 -H 'User-Agent: ALIYUN-ANDROID-UA' \
 -H 'Accept: application/json; charset=utf-8' \
 -H 'Accept-Encoding: gzip' \
 -H 'date: Wed, 07 Aug 2024 10:37:12 GMT' \
 -H 'x-ca-signature: KCFQR8i+VA6A0O5U7pLyT6dMTpWnzxAnHe04BNVX9r0=' \
 -H 'x-ca-nonce: edeb2fae-0633-4a43-bd5a-39329b636f7f' \
 -H 'x-ca-key: xxx' \
 -H 'ca_version: 1' \
 -H 'x-requiretoken: false' \
 -H 'x-ca-timestamp: 1723027032430' \
 -H 'x-ca-signature-headers: x-ca-nonce,x-ca-timestamp,x-ca-key' \
 -H 'content-type: application/x-www-form-urlencoded; charset=utf-8' \
 -H 'oauth: false' -H 'appversioncode: 3.4.6' \
 -H 'appversionname: xxx' \
 -H 'publicplatform: android' \
 -H 'if-modified-since: Wed, 07 Aug 2024 10:37:12 GMT'
```

我们发现，UA 是 `ALIYUN-ANDROID-UA`，且有一个 `x-ca-signature` 参数，这个参数是签名的结果，我们需要找到签名的算法。按照目前已知的信息，
我们把一些相关的参数放到搜索引擎里搜一下，看看是不是已有的开源的产品。在搜索 `x-ca-signature` 时，发现这个和阿里云网关的签名算法非常相像。
相关文档：[使用摘要签名认证方式调用API](https://www.alibabacloud.com/help/zh/api-gateway/traditional-api-gateway/user-guide/use-digest-authentication-to-call-an-api)。

我们先大胆假设它使用的就是阿里云网关，比较某克开发在杭州，杭州那边肯定很多阿里的人过去，所以大概率用阿里的产品，一切猜测都很合理。

# 签名算法

根据这个假设，我们看文档，最主要是需要 `x-ca-signature` 参数，这个参数是签名的结果。签名算法如下：

```java
Mac hmacSha256 = Mac.getInstance("HmacSHA256");
byte[] appSecretBytes = appSecret.getBytes(Charset.forName("UTF-8"));
hmacSha256.init(new SecretKeySpec(appSecretBytes, 0, appSecretBytes.length, "HmacSHA256"));
byte[] md5Result = hmacSha256.doFinal(stringToSign.getBytes(Charset.forName("UTF-8")));
String signature = Base64.encodeBase64String(md5Result);
```

这里面有 2 个重点：

- 计算出 `stringToSign`，即签名原文；
- 获取到 `appSecret`。

签名原文的构造方法其实文档里面都有，甚至有不同语言的 SDK，这里就不再赘述。我们重点关注 `appSecret` 怎么获取。

## 获取 appSecret

最初的想法是直接反编译 APP 找相关的字符串。可是发现某克直接把 APP 的代码放到 so 里并加密了，运行时再动态解密，因此静态去分析会比较困难。

![Java decompiler]({{ "/assets/images/2024-08-08-java-decompiler.png" | relative_url }})

因此，考虑别的办法，动态分析就是很好的办法。由于我们前面已经铺平了道路可以在 Magisk 的环境下启动某克 APP 了，因此我们可以尝试找切入点 Hook
并把对应的私钥打出来。

这里我们继续做一个假设，它使用的是官方的 Java SDK，通过看官方 SDK 的代码，在构造 Signer 的时候会传入 `appSecret` 
[permalink](https://github.com/aliyun/apigateway-sdk-core/blob/13fb5ee76a3c9241972a023193a6f3381ecf18e3/src/main/java/com/alibaba/cloudapi/sdk/signature/HMacSHA256SignerFactory.java#L45)。
这会是一个很好的切入点。

之所以说是很好的切入点，是因为它真的很好切入。这里需要引入一些背景知识。Xposed 的原理其实是类似 Java agent，最终可以实现在任意指定的函数执行
前或者后插入自己的代码，从而实现修改函数入参，出参。捕获函数入参这些当然不在话下。这里 Hook 难点其实是要找到对应的函数，函数名一般来说是混淆了，
但是有个盲区，一般标准库的函数是不会混淆的，不然整个程序跑不起来。利用这个特性，我们可以捕获它调用标准库函数的方法从而找到我们想要的密钥。退一
万步来说，我们可以捕获 `java.lang.String` 的构造函数，因为这个函数是一定会调用的，这样我们就可以捕获到所有的字符串了，我就不信他们不适用
字符串来存储密钥。但是这个输出量太大了，回到我们刚找到的切入点，它调用了 `javax.crypto.spec.SecretKeySpec`，这也是标准库的函数，我们直接
写个简单的 Hook 就可以捕获到了。

```java
XposedHelpers.findAndHookConstructor("javax.crypto.spec.SecretKeySpec", lpparam.classLoader, "byte[]", "int", "int", String.class, new XC_MethodHook() {
    @Override
    protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
        super.beforeHookedMethod(param);
        Object secret = param.args[0];
        byte[] buf = (byte[])secret;
        String secret = new String(buf);
        Log.w(LOG_TAG, "secret:"+secret);
    }
});
```

最后我们就可以在 logcat 里面看到输出的密钥了。并且成功地发出去包，拿到响应。

# 总结

这里我们已经成功破解了这个 APP。这个 APP 的防范比较多，混淆、加密、Xposed 检测、SSL 证书检测等等手段都上了，但我们还是见招拆招找到突破点。
毫无疑问地，对于一个车企来说安全其实是生命线，谁也不希望装了一个第三方插件第二天车被人开走了，到时候被反讹一下，所以这么多手段是能理解的。
