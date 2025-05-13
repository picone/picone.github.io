---
layout: post
title: Carlink 协议分析
description: 通过反编译某个手机端支持 Carlink  协议的 APP 研究其通讯协议。
---

# 链接通路

有 USB over AOA 和 蓝牙+WiFi 两种通讯方式。

# 链接认证

1. 读取 8 个字节。这 8 个字节结构如下：
    ```c
    struct {
        int32_t channel_id;
        int32_t message_len; // 接下来接收的消息长度
        char* message; // 接收的消息，长度为 message_len
    }
    ```
   channel_id 范围为 1-240，message_len 为 64MB。
2. 对于 channel_id = 1，message_len = 10 的消息，是特殊的建连消息，即结构如下：这是通用的端口创建 payload
    ```c
    struct {
        int32_t channel_id;
        int32_t message_len;
        int32_t port;
        int32_t new_channel_id; // 后续传输的 channel_id，终端需要记录下不同的 channel_id
        int8_t socket_type;
        int8_t channel_msg_type; // 1 为 ucar 协议，2 为 raw 类型
    }
    ```

    | port  | enum    | description                                                                                     |
    |-------|---------|-------------------------------------------------------------------------------------------------|
    | 0     | CUSTOM  |                                                                                                 |
    | 4321  | UIBC    | 用户输入反向控制通道，定义可以参考[Miracast](https://github.com/albfan/miraclecast/wiki/Miracast#specifications) |
    | 7236  | RTSP    |                                                                                                 |
    | 15550 | RTP     |                                                                                                 |
    | 57209 | AUTH    |                                                                                                 |
    | 57219 | CONTROL |                                                                                                 |
    | 57229 | MEDIA   |                                                                                                 |
    | 57239 | SENSOR  |                                                                                                 |
    | 57249 | CERT    |                                                                                                 |
    | 10113 | SHARE   |                                                                                                 |

   | socket_type | enum   | description                  |
   |-------------|--------|------------------------------|
   | 1           | SERVER | 看起来只有 channel_id = 1 的才是这个类型 |
   | 2           | CLIENT |                              |




