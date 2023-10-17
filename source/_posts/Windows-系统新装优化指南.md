---
title: Windows 系统新装优化指南
date: 2023-05-08 00:00:00
tags: Windows 性能优化
---

## 安装驱动和基本软件：驱动装完再装软件

驱动安装顺序：

1. 主板相关驱动
2. BIOS 更新（如果有）
3. 显卡、网卡、声卡
4. 其他驱动
5. 配套软件
6. 杀软（如果需要）

## 使用 Dism++ 卸载系统无用软件并关闭系统更新

若软件在用户应用和预装应用里都有，那么先在用户应用中删除再在预装应用中删除

- 有用的 Appx
  - 应用商店相关
  - UI 相关
  - 各种 VCLibs
  - .NET 环境
  - Webp、VP9、HEIF 等格式扩展支持
  - 画图、截图等

## 关闭无用的服务

![](禁用服务_1.webp)

![](禁用服务_2.webp)

![](禁用服务_3.webp)

![](禁用服务_4.webp)

对需要禁用的服务说明如下：

- AVCTP 服务：蓝牙耳机相关
- Hyper-V 相关服务：HV 主机相关
- Edge 更新服务：新版 Edge 相关
- Smart Card 开头的服务：智能卡相关
- SSDP Discovery、Function Discovery Provider Host、UPnP Device Host、Function Discovery Resource Publication：Upnp、SSDP 服务
- Shell Hardware Detection：自动播放功能
- Quality Windows Audio Video Experience：可禁用
- Windows Search：务必关闭
- SysMain：使用固态硬盘的请关闭
- IP Helper：提供 IPv6 到 IPv4 的转换，不使用 IPv6 的可以关闭
- TCP/IP NetBIOS Helper：提供 NetBIOS 相关功能
- Print 开头的三项服务：提供打印相关服务，不使用打印机的可以关闭
- 无线网卡搜不到网络的，一般都是因为无线电管理服务被禁用了
- 带 `_xxxxx` 后缀的服务可能在服务里调整不了，需到注册表的 `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\` 目录下调整（2 自动，3 手动，4 禁用）
  - BcastDVRUserService_xxx Xbox 游戏录制，根据需求选择 3/4
  - BluetoothUserService_xxx 蓝牙服务，建议 3
  - ConsentUxUserSvc_xxx、DevicesFlowUserSvc_xxx、DevicePickerUserSvc_xxx、DeviceAssociationBrokerSvc_xxx、UdkUserSvc_xxx 连接设备时可能会用到，建议 3
  - CaptureService_xxx 截图工具可能会用到的 api，建议 3
  - cbdhsvc_xxx 剪贴板，建议 2/3
  - CDPUserSvc_xxx 同步用户设置、隐私等数据用的，想保护隐私的选 3，需要这项功能的选 2
  - CredentialEnrollmentManagerUserSvc_xxx 凭据管理器，存储密码用的，建议 2/3
  - MessagingService_xxx 需要短信功能的 3，不用的 4
  - UnistoreSvc_xxx、UserDataSvc_xxx、储存、访问联系人、邮件等用户数据，需要用的 3，不用的 4
  - PimIndexMaintenanceSvc_xxx 联系人相关，用的 3，不用的 4
  - PrintWorkflowUserSvc_xxx 打印机相关，选 3
  - WpnUserService_xxx 推送系统通知用的，选 2
  - OneSyncSvc、OneSyncSvc_xxxx 不需要同步的可以 4

## 释放系统预留的 7G 空间

`DISM.exe /Online /Set-ReservedStorageState /State:Disabled`

## 避免微软输入法卡顿

右键输入法，设置 -> 常规，在兼容性里选择使用以前版本的微软输入法

## 解决提示需要新应用打开此 ms-screenclip 链接

`DISM /Online /Add-Capability /CapabilityName:Windows.Client.ShellComponents~~~~0.0.1.0`

## 推荐精简版系统

远景论坛 **不忘初心** 大神的各类精简版系统

## 右键菜单管理

[GitHub - BluePointLilac/ContextMenuManager: 🖱️ 纯粹的 Windows 右键菜单管理程序](https://github.com/BluePointLilac/ContextMenuManager)

## 解决 AMD CPU 卡顿的问题

关闭 BIOS 中的 `AMD Platform Security Processor`

## Windows 11 改回 Windows 10 的右键菜单

打开 cmd 并输入 `reg add "HKCU\Software\Classes\CLSID\{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}\InprocServer32" /f /ve` 后重启资源管理器即可生效。要恢复为 Windows 11 右键菜单，则使用命令：`reg delete "HKCU\Software\Classes\CLSID\{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}" /f`

## CPU 响应速度优化

下载软件 QuickCPU，将左下角 Core Parking、Frequency Scaling、Turbo boost、Performance 全部拉满即可。

## 加速系统响应速度

注册表找到 `计算机\HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management`，根据屏幕分辨率修改 SessionPoolSize 和 SessionViewSize

- SessionPoolSize：1080P 改为 62K 改为 12，4K 改为 24
- SessionViewSize：1080P 改为 62，2K 改为 144，4K 改为 288

## 针对 Ryzen 笔记本的优化

### 风扇转速

1. 使用软件 atrofac
2. 点击 Edit Configuration 配置，常用的配置如下

   ```conf
   # 极度安静
   cpu_curve:"30c:0%,40c:0%,50c:0%,60c:0%,70c:10%,80c:20%,90c:60%,100c:99%"
   # 中度安静
   cpu_curve: "30c:5%,40c:5%,50c:5%,60c:5%,70c:30%,80c:55%,90c:78%,100c:99%"
   # 玩游戏时
   cpu_curve:"30c:30%,40c:30%,50c:30%,60c:30%,70c:99%,80c:99%,90c:99%,100c:99%"
   ```

### 开启性能模式

1. 运行 regedit 打开注册表
2. 找到 `计算机\HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Power\PowerSettings\54533251-82be-4824-96c1-47b60b740d00\be337238-0d82-4146-a960-4f3749d470c7`，修改右侧 Attributes 修改值为 2
3. 在电源管理中可以看到处理器性能提升模式，启用可以开启睿频，关闭可关闭睿频

### 调整 AMD CPU 核显的 TDP 修正周期

1. 使用软件 `AMD APUTuning Utility`
2. 在 `Pre-Made Presets` 中找到自己的处理器应用即可

### 延长电池寿命

1. 使用软件 `MyASUS`
2. 设置笔记本充满电电量为 60%

## FPS 禁用全屏优化

打开注册表，进入 `计算机\HKEY_CURRENT_USER\System\GameConfigStore`

- GameDVR_FSEBehaviorMode：2
- GameDVR_HonorUserFSEBehaviorMode：1
- GameDVR_FSEBehavior：2
- GameDVR_DXGIHonorFSEWindowsCompatible：1

## 笔记本配置触控板手势

1. Windows 10 进入 设置 -> 设备 -> 触摸板 -> 高级手势配置 设置自定义手势
2. 个人使用的设置
   - 三指点击：关闭 (Ctrl+W)
   - 三指向上轻扫：向后导航
   - 三指向下轻扫：回到桌面 (Win+D)
   - 三指向左轻扫：切换应用 (Alt+Tab)
   - 三指向右轻扫：切换应用 (Alt+Tab)
   - 四指点击：播放 / 暂停
   - 四指向上轻扫：添加新的标签页 (Ctrl+T)
   - 四指向下轻扫：关闭当前页面 (Ctrl+W)
   - 四指向左轻扫：上一个标签 (Ctrl+Shift+Tab)
   - 四指向右轻扫：下一个标签 (Ctrl+Tab)

## 关闭 Virtualization-based Security

1. 按 Win+S，输入 System Information 或 msinfo32，如果 Virtualization-based security 为 Running 则已启用
2. 按 Win+S，输入 Core Isolation，将内存完整性保护关闭
3. 运行 regedit，进入 `计算机\HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\DeviceGuard`，新建 DWORD `EnableVirtualizationBasedSecurity` 并赋值 0

## 彻底关闭 Hyper-V 虚拟机

```powershell
bcdedit /set hypervisorlaunchtype off
```

## 重置所有系统应用

```powershell
# 输入 Y 并回车
Set-ExecutionPolicy Unrestricted
Get-AppXPackage -AllUsers | Foreach {Add-AppxPackage -DisableDevelopmentMode -Register "$($_.InstallLocation)\AppXManifest.xml"}
Get-AppxPackage Microsoft.SecHealthUI -AllUsers | Reset-AppxPackage
```

## 关闭笔记本 S0 睡眠状态

```powershell
# 查询系统固件支持哪些睡眠状态
powercfg /a
# 关闭 S0 睡眠状态
reg add "HKLM\System\CurrentControlSet\Control\Power" /v PlatformAoAcOverride /t REG_DWORD /d 0
# 重新开启 S0 睡眠状态
reg delete "HKLM\System\CurrentControlSet\Control\Power" /v PlatformAoAcOverride /f
```

## 优化系统 MTU

```powershell
# 查看当前使用的 MTU 大小
netsh interface ipv4 show subinterfaces
# 使 -l 后的参数最大但可 ping 通
ping baidu.com -f -l 1464
# 将得到的数值加 28
# 本机设置, 也可在路由器设置
netsh int ipv4 set subinterface "本地连接" mtu=1492 store=persistent
```
