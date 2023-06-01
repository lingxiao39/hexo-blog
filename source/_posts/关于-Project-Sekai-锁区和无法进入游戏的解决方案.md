---
title: 关于 Project Sekai 锁区和无法进入游戏的解决方案
date: 2023-04-30 00:00:00
tags:
---

## 前言

`Project Sekai: Colorful Stage! feat. Hatsune Miku`（以下简称 `PJSK`）是世嘉开发的一款音乐游戏，在进入游戏的过程中会检测玩家 IP 是否在允许列表中，如果不在则会禁止玩家登录。无法登录游戏的常见情况有如下几种：

1. 提示 `会话已过期，返回标题界面`（如图），这种情况一般是 IP 不在允许登录的范围内，即 **被锁区**
   ![](1682833256196.webp)
2. 依次出现两个弹窗，第一个提示 `发生通信错误`，第二个提示 `回到标题界面`，一般是网络不稳定造成的，不能确定是否被锁区
   ![](1682833260822.webp)
   ![](1682833264335.webp)
3. 进入游戏登录界面并点击后又返回登录界面，并且无任何弹窗提示，这种和第二种情况类似，一般为网络原因

## 锁区原理探究

对于以上的第二和第三种情况，请检查网络问题并重试，这里重点讨论第一种情况（被锁区）的解决方案。
在进入游戏的过程中主要向以下几个域名发起连接：

- cdp.cloud.unity3d.com
- config.uca.cloud.unity3d.com
- production-xxxxxxxx-assetbundle.sekai.colorfulpalette.org
- production-xxxxxxxx-assetbundle-info.sekai.colorfulpalette.org
- issue.sekai.colorfulpalette.org
- game-version.sekai.colorfulpalette.org
  以下域名是 unity 用于收集分析数据的（见 [https://stackoverflow.com/questions/46045805/why-my-game-is-trying-to-connect-to-external-server](https://stackoverflow.com/questions/46045805/why-my-game-is-trying-to-connect-to-external-server)），这两个域名在进入游戏 Rizline 时也会出现
- cdp.cloud.unity3d.com
- config.uca.cloud.unity3d.com
  以下域名为游戏中分发静态资源（例如音乐、谱面文件和图片等）的域名，其中 `xxxxxxxx` 为八位字母和数字组合
- production-xxxxxxxx-assetbundle.sekai.colorfulpalette.org
- production-xxxxxxxx-assetbundle-info.sekai.colorfulpalette.org
  以下域名用于 **游戏结束** 上传成绩时和服务器通信：
- production-web.sekai.colorfulpalette.org
- production-game-api.sekai.colorfulpalette.org
  以下域名用于在 **登录游戏** 时验证玩家 IP 是否在允许范围内（是否解锁），如果不是则会出现无法进入游戏的情况
- issue.sekai.colorfulpalette.org
- game-version.sekai.colorfulpalette.org
  解决锁区问题的核心就是 **以上两个域名** 需要通过官方允许的 IP 向服务器发起连接。

## 解决方案

1. 懒人方案：直接将后缀为 `sekai.colorfulpalette.org` 的域名分流到解锁了 PJSK 的节点，对应 Clash 分流规则如下：

   ```yaml
   rules:
     # プロセカ 为解锁了 PJSK 的代理组
     - DOMAIN-SUFFIX,sekai.colorfulpalette.org,プロセカ
   ```

2. 完美方案：将解锁域名 `issue.sekai.colorfulpalette.org` 和 `game-version.sekai.colorfulpalette.org` 分流到解锁了 PJSK 的节点，其他域名分流到延迟最低的节点（提升游戏内加载速度），对应 Clash 分流规则如下：

   ```yaml
   rules:
     # プロセカ 为解锁了 PJSK 的代理组
     - DOMAIN,issue.sekai.colorfulpalette.org,プロセカ
     - DOMAIN,game-version.sekai.colorfulpalette.org,プロセカ
     # 匹配其他游戏需要用到的域名
     - DOMAIN-SUFFIX,sekai.colorfulpalette.org,延迟最低的代理组
   ```
