---
title: Arch Linux 安装全过程
date: 2023-09-28 00:00:00
tags:
---

## 为什么离开 Windows

1. 微软的监控
2. 盗版的风险

## 安装完整步骤

1. 下载 Arch Linux iso 镜像文件
   - 使用 [中科大镜像](https://mirrors.ustc.edu.cn/archlinux/iso/2023.09.01/archlinux-2023.09.01-x86_64.iso)
2. 下载 Rufus 刻录软件
3. 写入镜像到 U 盘
4. 开机狂按 Del 键进入 BIOS，关闭安全启动
5. 开机狂按 F11 进入引导项，选择 U 盘
6. 在启动选择界面选择第一项：`Arch Linux install medium (x86_64, UEFI)`
7. 依次执行以下命令

   ```bash
   # 禁用 reflector 服务
   systemctl stop reflector.service
   # 检查 reflector 服务是否被关闭
   systemctl status reflector.service
   # 确认当前是否运行在 UEFI 模式
   # 如果出现一堆东西已在 UEFI 模式，否则需要确认启动方式是否为 UEFI
   ls /sys/firmware/efi/efivars
   # 连接网络: 使用无线连接才需要进行以下步骤
   iwctl # 进入交互式命令行
   device list # 列出无线网卡设备名，比如无线网卡看到叫 wlan0
   station wlan0 scan # 扫描网络
   station wlan0 get-networks # 列出所有 wifi 网络
   station wlan0 connect wifi-name # 进行连接，注意这里无法输入中文。回车后输入密码即可
   exit # 连接成功后退出
   # 测试网络连通性
   ping www.bilibili.com
   # 更新系统时钟
   # 这步很重要，关系到后面 pgp 密钥环更新环节
   timedatectl set-ntp true # 将系统时间与网络时间进行同步
   timedatectl status # 检查服务状态
   # 修改镜像源为国内，加快下载速度
   # 推荐使用中科大源和清华源
   vim /etc/pacman.d/mirrorlist
   # =========================
   Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch # 中国科学技术大学开源镜像站
   Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch # 清华大学开源软件镜像站
   # =========================
   # 分区和格式化
   lsblk # 显示当前分区情况
   # 使用 cfdisk 对磁盘分区
   cfdisk /dev/nvme0n1
   # 如果不再使用 Windows，则抹掉硬盘上原来的文件系统
   wipefs --all /dev/nvme0n1
   # 创建三个分区: EFI、Swap 和主分区
   # EFI: 512M, Swap: 14G, 剩下的给主分区
   # 分区完成后查看磁盘情况
   lsblk
   # 下面假设
   # EFI: /dev/nvme0n1p1
   # Swap: /dev/nvme0n1p2
   # 主分区: /dev/nvme0n1p3
   # 格式化 Swap 分区
   mkswap /dev/nvme0n1p2
   # 使用 Btrfs 格式化主分区
   mkfs.btrfs -L myArch /dev/nvme0n1p3
   # 将主分区挂载以创建子卷
   mount -t btrfs -o compress=zstd /dev/nvme0n1p3 /mnt
   # 查看挂载情况
   df -h
   # 创建 / 目录子卷
   btrfs subvolume create /mnt/@
   # 创建 /home 目录子卷
   btrfs subvolume create /mnt/@home
   # 查看子卷情况
   btrfs subvolume list -p /mnt
   # 卸载主分区以挂载子卷
   umount /mnt
   # 挂载子卷 (请务必依次执行下列命令)
   mount -t btrfs -o subvol=/@,compress=zstd /dev/nvme0n1p3 /mnt # 挂载 / 目录
   mkdir /mnt/home # 创建 /home 目录
   mount -t btrfs -o subvol=/@home,compress=zstd /dev/nvme0n1p3 /mnt/home # 挂载 /home 目录
   mkdir -p /mnt/boot/efi # 创建 /boot/efi 目录
   mount /dev/nvme0n1p1 /mnt/boot/efi # 挂载 /boot/efi 目录
   swapon /dev/nvme0n1p2 # 挂载交换分区
   # 查看挂载情况
   df -h
   # 查看 Swap 分区挂载情况
   free -h
   # 安装系统
   pacstrap /mnt base base-devel linux linux-firmware btrfs-progs
   # 安装其他必要的功能性软件
   pacstrap /mnt dhcpcd networkmanager vim sudo zsh zsh-completions
   # 生成 fstab 文件
   genfstab -U /mnt > /mnt/etc/fstab
   # 查看 /mnt/etc/fstab 确保没有错误
   cat /mnt/etc/fstab
   # 将系统环境切换到新系统下
   arch-chroot /mnt
   # 设置主机名与分区
   vim /etc/hostname
   # 设置 hosts
   vim /etc/hosts
   # =========================
   127.0.0.1   localhost
   ::1         localhost
   127.0.1.1   myarch.localdomain  myarch
   # =========================
   # 设置时区
   ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
   # 将系统时间同步到硬件
   hwclock --systohc
   # 设置 Locale
   vim /etc/locale.gen
   # 去掉 `en_US.UTF-8 UTF-8` 以及 `zh_CN.UTF-8 UTF-8` 行前的注释符号
   # 生成 locale
   locale-gen
   # 向 /etc/locale.conf 添加内容
   echo 'LANG=en_US.UTF-8'  > /etc/locale.conf
   # 为 root 用户设置密码
   passwd root
   # 安装芯片制造商的微码
   pacman -S intel-ucode # Intel
   pacman -S amd-ucode # AMD
   # 安装引导程序
   pacman -S grub efibootmgr os-prober
   grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ARCH
   # 编辑 GRUB 文件
   vim /etc/default/grub
   # =========================
   # 去掉 GRUB_CMDLINE_LINUX_DEFAULT 一行中最后的 quiet 参数
   # 把 loglevel 的数值从 3 改成 5，为了后续出现系统错误时方便排错
   # 加入 nowatchdog 参数，显著提高开关机速度
   GRUB_CMDLINE_LINUX_DEFAULT="loglevel=5 nowatchdog"
   # =========================
   # 如果需要引导 Windows 10 则需要再加一行
   GRUB_DISABLE_OS_PROBER=false
   # =========================
   # 生成 GRUB 所需的配置文件
   grub-mkconfig -o /boot/grub/grub.cfg
   # 完成安装, 这个时候拔掉 U 盘
   exit # 退回安装环境
   umount -R /mnt # 卸载新分区
   reboot # 重启
   ```

8. 重启后使用 root 和密码登录 Linux 并执行以下命令

   ```bash
   systemctl enable --now NetworkManager # 设置开机自启并立即启动 NetworkManager 服务
   ping www.bilibili.com # 测试网络连接
   # 若使用无线连接则先连接网络
   nmcli dev wifi list # 显示附近的 Wi-Fi 网络
   nmcli dev wifi connect "Wi-Fi名（SSID）" password "网络密码" # 连接指定的无线网络
   # 安装 neofetch
   pacman -S neofetch
   # 使用 neofetch 打印系统信息
   neofetch
   # 关机
   shutdown -h now
   ```

9. 至此，无图形界面的 archlinux 已安装完成

## 图形界面安装和配置

```bash
# 确保当前系统为最新
pacman -Syyu
# 配置 root 账户默认编辑器
vim ~/.bash_profile
# 添加以下内容
# =========================
export EDITOR='vim'
# =========================
# 准备非 Root 用户
useradd -m -G wheel -s /bin/bash myusername
# 设置新用户的密码
passwd myusername
# 同意新用户使用 sudo
vim /etc/sudoers
# 找到下面一行并把前面的 `#` 去掉
# =========================
#%wheel ALL=(ALL) ALL
# =========================
# 开启 32位支持库和Arch Linux 中文社区仓库
vim /etc/pacman.conf
# 去掉 `[multilib]` 一节中的两行注释
# 在末尾添加如下内容
# =========================
[archlinuxcn]
Server = https://mirrors.ustc.edu.cn/archlinuxcn/$arch # 中国科学技术大学开源镜像站
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch # 清华大学开源软件镜像站
# =========================
# 再次更新系统
pacman -Syyu
# 安装 KDE Plasma 桌面环境
pacman -S plasma-meta konsole dolphin
# (可选) 安装 wayland
pacman -S  plasma-wayland-session xdg-desktop-portal
# 配置并启动欢迎屏幕
systemctl enable --now sddm
# 输入新用户密码即可登录桌面
```

## 图形界面的后续配置

```bash
# 测试桌面环境网络连通性
ping www.bilibili.com
# 安装基础功能包
sudo pacman -S ntfs-3g # 使系统可以识别 NTFS 格式的硬盘
sudo pacman -S adobe-source-han-serif-cn-fonts wqy-zenhei # 安装几个开源中文字体。一般装上文泉驿就能解决大多 wine 应用中文方块的问题
sudo pacman -S noto-fonts noto-fonts-cjk noto-fonts-emoji noto-fonts-extra # 安装谷歌开源字体及表情
sudo pacman -S firefox chromium # 安装常用的火狐、chromium 浏览器
sudo pacman -S ark # 压缩软件。在 dolphin 中可用右键解压压缩包
sudo pacman -S packagekit-qt5 packagekit appstream-qt appstream # 确保 Discover（软件中心）可用，需重启
sudo pacman -S gwenview # 图片查看器
sudo pacman -S steam # 游戏商店。稍后看完显卡驱动章节再使用
sudo pacman -S archlinuxcn-keyring # cn 源中的签名（archlinuxcn-keyring 在 archlinuxcn）
sudo pacman -S yay # yay 命令可以让用户安装 AUR 中的软件（yay 在 archlinuxcn）
# 检查家目录下常见目录是否创建, 若未创建则需要手动创建
cd ~
ls -hl
mkdir Desktop Documents Downloads Music Pictures Videos
```

## 遇到的问题

### 出现 PGP signature Error

```bash
pacman-key --init
pacman-key --populate
pacman -S archlinux-keyring
```

### 显卡驱动问题

NVIDIA 驱动未正常运行，具体现象为按步骤完成安装并进入桌面后打开应用很慢，且在终端执行 `nvtop` 显示无法找到显卡

经过排查后发现是由于 Linux 系统自带的 nouveau 比 NVIDIA 官方驱动更早启动，导致 NVIDIA 驱动无法正常运行。

解决方法如下

```bash
# 查看启动时系统内核输出消息
sudo dmesg
# 禁用 Linux 自带的 nouveau 显卡驱动
echo 'blacklist nouveau' > /etc/modprobe.d/blacklist-nvidia-nouveau.conf
echo 'options nouveau modeset=0' >> /etc/modprobe.d/blacklist-nvidia-nouveau.conf
# 重新制作镜像
sudo mkinitcpio -p linux
```

### 蓝牙耳机无法连接

```bash
sudo pacman -S pulseaudio
sudo pacman -S pulseaudio-bluetooth
```

### Dolphin 预览 avif 格式的图片 2

```bash
sudo pacman -S kimageformats
sudo pacman -S shared-mime-info
sudo pacman -S libavif
```

### crontab 报错：`"/usr/bin/vi" exited with status 127`

```bash
sudo vim ~/.profile
# ========================================
EDITOR='/bin/vim'
VISUAL='/bin/vim'
export EDITOR VISUAL
# ========================================
```

### Gwenview 无法显示 webp 图像

```bash
sudo pacman -S qt5-imageformats
```

## 编程环境配置

### OpenCV（C++ 和 Python）

1. 安装依赖包

   ```bash
   sudo pacman -S opencv hdf5 tk
   pip install opencv-python
   ```

2. 编辑项目的 CMake 文件，添加如下两行（仅 C++ 需要）

   ```cmake
   find_package (OpenCV 4.0.0 REQUIRED)
   include_directories ("/usr/include/opencv4/")
   ```

### C++ Boost 库

```bash
sudo pacman -Ss boost
```

## 参考资料

1. [archlinux 简明指南](https://arch-linux.osrc.com/)
2. [NVIDIA - ArchWiki](https://wiki.archlinux.org/title/NVIDIA)
