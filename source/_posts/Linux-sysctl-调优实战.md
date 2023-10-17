---
title: Linux sysctl 调优实战
date: 2023-06-12 00:00:00
tags: Linux 性能优化
---

## sysctl 的使用方式

1. 使用命令 `sysctl -w key=value` 手动调整选项值
2. 使用命令 `sysctl key` 获取选项当前的值
3. （推荐）先编辑文件 `/etc/sysctl.conf` 然后使用命令 `sysctl -p` 一键导入已修改的键值对

## 使用合适的拥塞控制算法

拥塞控制是通过算法来提升或降低数据发送速率以防止网络出现拥堵。常见的拥塞控制算法有 reno、cubic、bbr 等，其中 reno 和 cubic 是基于丢失的算法（出现拥堵后降低发送速率），bbr 则是基于模型（监控延迟是否改变）。

在 Linux 中可以使用 `sysctl net.ipv4.tcp_congestion_control` 查看当前系统使用的拥塞控制算法。使用 `uname -r` 查看 Linux 内核版本，若版本低于 4.9 则未内置 bbr 算法，可以先更新内核版本。

## 优化完成后的 sysctl.conf 文件

```bash
# 文件系统优化
fs.aio-max-nr=1048576
fs.file-max=1000000
fs.inotify.max_user_instances=4096
fs.lease-break-time=30
# 开启崩溃自动重启
kernel.panic=1
kernel.sysrq=0
kernel.pid_max=1000000
net.core.default_qdisc=fq
net.core.netdev_max_backlog=65536
net.core.rmem_default=3400000
net.core.rmem_max=67000000
net.core.somaxconn=65535
net.core.wmem_default=3400000
net.core.wmem_max=67000000
net.core.optmem_max=67000000
# 关闭 ICMP 重定向请求
net.ipv4.conf.default.accept_redirects=0
net.ipv4.conf.default.rp_filter=1
net.ipv4.conf.default.accept_redirects=0
net.ipv4.conf.default.secure_redirects=0
# 路由系统中不要禁用
net.ipv4.conf.default.send_redirects=0
# 禁 ping
net.ipv4.icmp_echo_ignore_all=1
net.ipv6.icmp.echo_ignore_all=1
# 关闭攻击记录日志
net.ipv4.conf.default.log_martians=0
# 禁用 IP 源路由
net.ipv4.conf.default.accept_source_route=0
# 避免放大攻击
net.ipv4.icmp_echo_ignore_broadcasts=1
# 启用恶意错误信息防护
net.ipv4.icmp_ignore_bogus_error_responses=1
# 关闭 IPv4 包转发
net.ipv4.ip_forward=0
# 增大本地端口范围
net.ipv4.ip_local_port_range=1024 65000
# 开启 MTU 嗅探
net.ipv4.ip_no_pmtu_disc=0
# 加快路由缓存刷新频率
net.ipv4.route.gc_timeout=100
net.ipv4.tcp_adv_win_scale=1
net.ipv4.tcp_base_mss=1024
net.ipv4.tcp_comp_sack_delay_ns=1000000
# 调整拥塞控制算法为 BBR
net.ipv4.tcp_congestion_control=bbr
# 关闭 early demux
net.ipv4.tcp_early_demux=0
net.ipv4.tcp_ecn=1
net.ipv4.tcp_ecn_fallback=1
# 开启发送和接收双向的 fastopen
net.ipv4.tcp_fastopen=3
# 缩短处于 FIN_WAIT_2 状态的时间
net.ipv4.tcp_fin_timeout=30
net.ipv4.tcp_frto=2
# TCP 发送心跳包的时间间隔
net.ipv4.tcp_keepalive_time=1200
net.ipv4.tcp_keepalive_intvl=30
net.ipv4.tcp_keepalive_probes=3
# 已记住的连接数
net.ipv4.tcp_max_syn_backlog=2048
# 增大 tw buckets
net.ipv4.tcp_max_tw_buckets=2000000
net.ipv4.tcp_min_rtt_wlen=50
net.ipv4.tcp_min_tso_segs=2
# TCP 包大小自适应
net.ipv4.tcp_moderate_rcvbuf=1
# 打开 TCP MTU 嗅探
net.ipv4.tcp_mtu_probing=1
net.ipv4.tcp_no_metrics_save=1
# 开启 tcp_notsent_lowat
net.ipv4.tcp_notsent_lowat=131072
net.ipv4.tcp_orphan_retries=2
net.ipv4.tcp_recovery=1
net.ipv4.tcp_reordering=6
# 放弃回应一个 TCP 连接请求前需进行多少次重试
net.ipv4.tcp_retries1=3
# 在丢弃已建立通讯状况的 TCP 连接前需进行多少次重试
net.ipv4.tcp_retries2=3
# 开启 RFC1337 标准
net.ipv4.tcp_rfc1337=1
# 启用有选择的应答
net.ipv4.tcp_sack=1
net.ipv4.tcp_fack=1
net.ipv4.tcp_tw_reuse=1
net.ipv4.tcp_max_orphans=3276800
# 加快重发
net.ipv4.tcp_slow_start_after_idle=0
net.ipv4.tcp_early_retrans=1
net.ipv4.tcp_syn_retries=3
net.ipv4.tcp_synack_retries=2
# SYN 等待队列溢出时启用 cookies 处理, 防止 SYN 攻击
net.ipv4.tcp_syncookies=1
# 开启 TCP 时间戳, 防止 TCP 的 RST 攻击
net.ipv4.tcp_timestamps=1
net.ipv4.tcp_tso_win_divisor=2
# 要支持超过 64KB 的窗口大小需要启用
net.ipv4.tcp_window_scaling=1
# tcp 内存调整
net.ipv4.tcp_mem=78643200 104857600 157286400
net.ipv4.tcp_wmem=4096 66560 70254592
net.ipv4.tcp_rmem=4096 89088 70254592
# 关闭 early demux
net.ipv4.udp_early_demux=0
# 增大 UDP 内存限制
net.ipv4.udp_mem=78643200 104857600 157286400
# net.ipv4.udp_rmem_min=94500000
# net.ipv4.udp_wmem_min=94500000
# 不使用 IPv6 时彻底关闭 IPv6
net.ipv6.conf.default.disable_ipv6=1
# 使用 IPv6 时优化
# net.ipv6.conf.all.forwarding=1
# net.ipv6.conf.all.accept_ra=2
# net.ipv6.conf.all.proxy_ndp=1
# net.ipv6.conf.all.hop_limit=128
# net.ipv6.conf.default.hop_limit=128
# net.ipv6.route.min_adv_mss=1024
# net.ipv6.route.mtu_expires=600
# 调整内存写回策略
vm.dirty_background_ratio=2
vm.dirty_expire_centisecs=600
vm.dirty_ratio=5
vm.dirty_writeback_centisecs=100
vm.max_map_count=262144
vm.min_free_kbytes=262144
vm.swappiness=10
vm.vfs_cache_pressure=200
# netfilter 调整
net.netfilter.nf_conntrack_acct=1
net.netfilter.nf_conntrack_checksum=0
net.netfilter.nf_conntrack_max=65535
net.netfilter.nf_conntrack_tcp_timeout_established=7440
net.netfilter.nf_conntrack_udp_timeout=60
net.netfilter.nf_conntrack_udp_timeout_stream=180
```

## 调整 Linux 系统限制

```bash
# 修改系统安全限制
echo "* soft nofile 51200" >> /etc/security/limits.conf
echo "* hard nofile 51200" >> /etc/security/limits.conf
echo "session required pam_limits.so" >> /etc/pam.d/common-session
# 配置开机修改限制
echo "ulimit -n 1000000" >> /etc/profile
echo "ulimit -i 193045" >> /etc/profile
echo "ulimit -l 64" >> /etc/profile
echo "ulimit -u 1000000" >> /etc/profile
```

## 关闭网卡 NAPI 功能

网卡的 NAPI 功能可能会带来延迟的波动和其他问题，可使用以下命令将其关闭：

```bash
ethtool -C eth0 rx-usecs 0
```

## 调整网卡的接收和发送队列

```bash
# 查看当前网卡的队列长度
ethtool -g eth0
# 修改 eth0 接收和发送队列长度为 4096
ethtool -G eth0 rx 4096
ethtool -G eth0 tx 4096
# 增加 txqueuelen
ip link set dev eth0 txqueuelen 4096
# 如果在运行旧版本内核, 应手动增加 txqueuelen 到 1000
ifconfig eth0 txqueuelen 1000
```

## 增大 Swap 分区

推荐设置的 Swap 分区大小和内存大小的对应关系如下表：

| 内存大小 | 推荐的 Swap 分区大小 |
| -------- | -------------------- |
| 小于 4g  | 4G                   |
| 4-16G    | 8G                   |
| 16-64G   | 16G                  |
| 64-256G  | 32G                  |

可使用如下命令改变创建新的 Swap 文件并设置开启自动启用：

```bash
# 查看当前分区状况
free -m
# 关闭所有 Swap
swapoff -a
# 创建新的 Swap 文件: 块大小为 1MB (1048576 bytes)，数量为 4096
dd if=/dev/zero of=/var/swapfile bs=1M count=4096
# 设置文件为 Swap
mkswap /var/swapfile
# 启用 Swap 分区
swapon /var/swapfile
# 查看当前分区状况
free -m
# 设置开启启动
vim /etc/fstab
# ========================================
/var/swapfile swap swap defaults 0 0
# ========================================
```

## 参考资料

1. [IP Sysctl — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html)
2. [Linux 如何增大、缩小 swap 分区](https://blog.csdn.net/MssGuo/article/details/120586496)
3. [Sysctl configuration for high performance](https://gist.github.com/voluntas/bc54c60aaa7ad6856e6f6a928b79ab6c)
4. [Red Hat Enterprise Linux Network Performance Tuning Guide](https://access.redhat.com/sites/default/files/attachments/20150325_network_performance_tuning.pdf)
5. [networking:napi](https://wiki.linuxfoundation.org/networking/napi#disadvantages)
