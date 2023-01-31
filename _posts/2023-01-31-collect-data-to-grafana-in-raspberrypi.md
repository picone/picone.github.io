---
layout: post
title: 树莓派采集机器数据并上报到 Grafana 云。
description: 本文没有太多的技术含量，只是描述了发现的新世界大门。
---

# 背景

最近玩起了 PT，通过树莓派外挂 USB 移动硬盘盒接一块 16T 的硬盘来下载资源。但是在家的时候也很想知道树莓派跑的怎么样了系统情况怎么样，其实这属于可
观测性的一部分，可以直接撸上现有的各种组件。下面介绍一下使用 Grafana Cloud 接入数据的方法。

# 过程

## 注册 Grafana Cloud

在 Grafana 官网注册一个新账号，目前新账号的都有 15 天的免费 Pro Plan 体验，但是免费 Plan 其实对于个人用户来说足够了。下面是注册链接：

[https://grafana.com/auth/sign-up](https://grafana.com/auth/sign-up?refCode=gr8aqPwaTptKwp8)

![Grafana Plan]({{ "/assets/images/2023-01-31-grafana-plan.png" | relative_url }})

## 安装 Agent

在导航栏选择 Integrations and Connections，服务选择 Linux Server，进入如下界面：

![安装 agent 步骤 1]({{ "/assets/images/2023-01-31-grafana-install-agent-step-1.png" | relative_url }})

架构选择 Armv7。然后翻到下面复制安装命令，执行后就会自动开始采集。

![安装 agent 步骤 2]({{ "/assets/images/2023-01-31-grafana-install-agent-step-2.png" | relative_url }})

首次安装成功后会自动新增 Integration - Linux Node 这个文件夹的 Dashboard，可以查看预先配置好的 Dashboard，离成功不远了。

## 配置 Dashboard

根据自己的个人喜好拖拽对应的 Dashboard

# 成果

![成果]({{ "/assets/images/2023-01-31-grafana-result.png" | relative_url }})

除此之外可以酌情增加一些告警策略。
