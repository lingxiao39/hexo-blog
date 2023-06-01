---
title: WPD：阻止 Windows 收集个人信息的利器
date: 2022-12-05 20:04:23
tags:
---

[WPD](https://wpd.app) 是一款免费的 Windows 系统隐私设置工具，包含隐私管理、IP 拦截和 Appx 卸载等功能，通过调用系统自带 API 实现禁止相关隐私功能、阻止 Windows 遥测、关闭 Windows Defender 防火墙、阻止 Windows 更新等操作。
从 Windows 10 开始，操作系统对于个人数据的搜集能力大大加强。虽然微软保证不泄露个人信息，但是总让人不放心。如果你不想让微软得到这些数据，可以在系统设置、服务、组策略、注册表等中关闭，WPD 实现了多种隐私设置一处管理。其可以实现的功能有：禁用 UAC、Windows Defender、隐私设置、监控计划任务等；移除 Cortana、One Drive、遥测与数据搜集、遥测主服务、位置服务、Wi-Fi 共享、广告 ID、Defender 数据报告、数据共享、DRM 互联网访问、系统反馈请求、系统搜索、IE/Edge 请勿跟踪等。下面介绍主要的设置方法并分享我自己的设置。
![](1670228218069.png)
软件主要分为 Privacy、Blocker 和 Apps 三个模块，主要用到前两个模块。

## Privacy

首先打开 Privacy 模块，可根据自己的需要关闭对应的选项。对于会影响系统功能的选项在右边会有黄色的感叹号按钮，点击可查看会影响系统的哪些功能。例如 Services 中的 **[连接设备平台用户服务]** 会影响系统的 **夜灯** 功能，**Windows Push Notifications User Service** 会影响操作中心和网络设置。

![](1670228850059.png)

对于我来说，我保留了 **Allow Clipboard History**（使用剪切板历史，方便的小功能）和 **Windows Push Notifications User Service**。

![](1670229057887.png)

## Blocker

来到第二个大项 Blocker，这里建议使用 Windows Filtering Platform 进行过滤，以下是 [Microsoft 的描述](https://learn.microsoft.com/en-us/windows/win32/fwp/windows-filtering-platform-start-page)。

> Windows Filtering Platform (WFP) is a set of API and system services that provide a platform for creating network filtering applications. The WFP API allows developers to write code that interacts with the packet processing that takes place at several layers in the networking stack of the operating system.
> 下面的三个规则为绿色则说明已开启屏蔽，分别对应着 **Windows 遥测**、**第三方应用跟踪** 和 **阻止 Windows Update**，推荐全部开启。
>
> ![](1670229454968.png)
