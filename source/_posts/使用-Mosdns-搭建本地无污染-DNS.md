---
title: 使用 Mosdns 搭建本地无污染 DNS
date: 2023-01-17 00:00:00
tags:
---

## 前言

国内 DNS 污染越来越严重，特别是移动宽带，运营商分配的 DNS 在解析很多境外网站时会得到污染的结果或直接返回 127.0.0.1（例如 aka.ms）从而导致代理软件失效。虽然上述问题可以在代理软件中使用分流解决，但在本地搭建 DNS 不仅可以实现分流，更可实现部分广告过滤和缓存功能，极大提升了网页浏览体验。

![](1673951344719.png)

## 无污染 DNS 实现原理

全部 DNS 请求均使用境外无污染 DNS 会导致本地域名解析速度偏慢，且在使用 ECS 的情况下准确度不如国内 DNS，若使用本地 DNS 服务端进行分流则可以兼备快速和准确的优点。

## 要实现的目标

1. DNS 解析结果无污染
2. 国内外按域名和 IP 分流
3. 支持 ECS，能解析到距离最近的 CDN 服务器
4. 延迟尽可能低
5. 最好全程使用加密 DNS（DoT、DoH、DoQ 等）

## 上游 DNS 选择

国内 DNS 的选择方面，经过测试发现，使用腾讯 DNS 在国内的出口在广东，解析出的地址不是我所在地区的最优地址，而阿里 DNS 就没有这个问题。

国外的 DNS 选择 GoogleDNS 和 EasymosDNS 作为上游，EasymosDNS 的上游使用国内阿里 DNS、腾讯的 DNSPod 和国外的 GoogleDNS、OpenDNS 等，搭配 ECS 可实现国外正确解析到距离最近的 CDN。

## 分流原理

由于 mosdns 支持 v2ray 的 dat 文件，因此可直接使用 geoip.dat 匹配 IP 地址，使用 geosite.dat 匹配域名。

在广告过滤方面，可直接使用 `geosite:category-ads-all` 进行过滤，也可添加常见的广告过滤规则。这里广告过滤规则只支持域名，示例文件如下：

```txt
example-ad1.com
example-ad2.com
```

我使用了 oisd 的 [basic 过滤列表](https://dbl.oisd.nl/basic)，更多列表可在 <https://oisd.nl/downloads> 中查看。

## DNS 请求处理流程

1. 查询 hosts 文件匹配域名
2. 查询缓存
3. 屏蔽类型为 65 的 DNS 请求：返回 127.0.0.1（[为什么要屏蔽？](https://www.v2ex.com/t/712074)）
4. 屏蔽广告域名：返回 NXDOMAIN
5. 通过域名匹配网址，若匹配到则可第一次分流
6. 从无污染服务器获取域名 IP，通过 IP 分流，此时可能出现以下情况
   - 获取到的 IP 是国外的 IP 则直接返回结果即可
   - 获取的 IP 是国内 IP 或获取到国内有服务器的 CDN 厂商：将请求通过本地服务器再次解析并返回结果，此时又分为两种情况
     - 从国内服务器获取到了被污染的 IP 或使用国内服务器解析出的 IP 不是境内的 IP 则将请求再次交给国外无污染服务器处理
     - 获取到的 IP 不满足上一条的则直接返回

## 配置文件

个人使用的配置文件如下：

```yaml
######### 日志文件存放位置 #########
log:
  file: "./mosdns.log"
  level: error
######### 数据源, 在这里声明规则文件 #########
data_providers:
  - tag: geosite
    file: ./rule/geosite.dat
    auto_reload: true
  - tag: gfwip
    file: ./rule/gfw_ip_list.txt
    auto_reload: true
  - tag: cdncn
    file: ./rule/cdn_domain_list.txt
    auto_reload: true
  - tag: cn
    file: ./rule/cn.dat
    auto_reload: true
  - tag: force_cn
    file: ./rule/ecs_cn_domain.txt
    auto_reload: true
  - tag: force_us
    file: ./rule/ecs_us_domain.txt
    auto_reload: true
  - tag: hosts
    file: ./rule/hosts.txt
    auto_reload: true
  - tag: oisd_dbl_basic
    file: ./rule/oisd_dbl_basic.txt
    auto_reload: true
######### 插件 #########
plugins:
  # 缓存
  - tag: cache
    type: cache
    args:
      size: 102400
      lazy_cache_ttl: 86400
      cache_everything: true
  # hosts
  - tag: hosts
    type: hosts
    args:
      hosts:
        - "provider:hosts"
  # 调整 ECS
  - tag: ecs_remote
    type: ecs
    args:
      auto: false
      ipv4: "代理服务器公网 IP"
      mask4: 24
      force_overwrite: true
  - tag: ecs_local
    type: ecs
    args:
      auto: false
      ipv4: "本地公网 IP"
      mask4: 24
      force_overwrite: true
  # 调整 TTL
  - tag: ttl_long
    type: ttl
    args:
      minimal_ttl: 300
      maximum_ttl: 3600
  # TYPE65 类型请求
  - tag: qtype65
    type: query_matcher
    args:
      qtype: [65]
  # PTR 类型请求
  - tag: query_is_ptr
    type: query_matcher
    args:
      qtype: [12]
  # 屏蔽请求
  - tag: black_hole
    type: blackhole
    args:
      rcode: 0
      ipv4: "127.0.0.1"
      ipv6: "::1"
  # 转发请求给阿里 DNS
  - tag: forward_ali
    type: fast_forward
    args:
      upstream:
        - addr: "tls://223.5.5.5"
          trusted: true
          enable_pipeline: true
        - addr: "tls://223.6.6.6"
          trusted: true
          enable_pipeline: true
  # 转发请求给 Google DNS
  - tag: forward_google
    type: fast_forward
    args:
      upstream:
        - addr: "tls://8.8.4.4"
          trusted: true
          enable_pipeline: true
        - addr: "https://dns.google/dns-query"
          dial_addr: "2001:4860:4860::8888"
          trusted: true
          enable_pipeline: true
          enable_http3: true
  # 转发请求给 iQDNS
  - tag: forward_iqdns
    type: fast_forward
    args:
      upstream:
        - addr: "https://cn-east.lele233.com/dns-query"
          dial_addr: "112.132.212.159"
          trusted: true
          enable_pipeline: true
  # 转发请求给 EasyMos DNS
  - tag: forward_easymos
    type: forward
    args:
      upstream:
        - addr: "https://doh.apad.pro/cn-query"
      bootstrap:
        - "119.29.29.29"
  # 并行转发至本地服务器
  - tag: forward_local
    type: sequence
    args:
      exec:
        - parallel:
            - - "forward_ali"
            - - "forward_iqdns"
  # 并行转发至远程无污染服务器
  - tag: forward_remote
    type: sequence
    args:
      exec:
        - parallel:
            - - "forward_google"
            - - "forward_easymos"
  # 匹配 CDN 域名
  - tag: query_is_cdn_cn_domain
    type: query_matcher
    args:
      domain:
        - "provider:cdncn"
  # 匹配 本地域名
  - tag: query_is_local_domain
    type: query_matcher
    args:
      domain:
        - "provider:geosite:cn,private,apple-cn,steam@cn,google-cn,category-games@cn"
  # 匹配 地址非中国的域名
  - tag: query_is_non_local_domain
    type: query_matcher
    args:
      domain:
        - "provider:geosite:geolocation-!cn"
  # 匹配 广告域名
  - tag: query_is_ad_domain
    type: query_matcher
    args:
      domain:
        - "provider:oisd_dbl_basic"
        - "provider:geosite:category-ads-all"
  # 强制本地解析: 可在 ./rule/ecs_cn_domain.txt 文件中添加域名
  - tag: query_is_cn_domain
    type: query_matcher
    args:
      domain:
        - "provider:force_cn"
  # 强制非本地解析: 可在 ./rule/ecs_us_domain.txt 文件中添加域名
  - tag: query_is_us_domain
    type: query_matcher
    args:
      domain:
        - "provider:force_us"
  # 结果有本地 IP
  - tag: response_has_local_ip
    type: response_matcher
    args:
      ip:
        - "provider:cn:cn"
  # 匹配 Akamai 的 CDN 域名
  - tag: response_cname_akamai
    type: response_matcher
    args:
      cname:
        - "akamaiedge.net"
        - "akadns.net"
        - "edgekey.net"
  # 匹配被污染 IP
  - tag: response_has_gfw_ip
    type: response_matcher
    args:
      ip:
        - "provider:gfwip"
  # 主要运行逻辑
  - tag: main_sequence
    type: sequence
    args:
      exec:
        - hosts # 域名映射 IP
        - cache # 缓存
        # 屏蔽 TYPE65 类型请求并拦截广告
        - if: "qtype65"
          exec:
            - black_hole
            - _return
        # 屏蔽广告域名
        - if: query_is_ad_domain
          exec:
            - _new_nxdomain_response
            - _return
        # 本地解析: 强制本地解析、PTR 请求、本地域名、CDN 域名、akamai 节点
        - if: "(query_is_cn_domain) || [query_is_ptr] || (query_is_local_domain) || (query_is_cdn_cn_domain) || (response_cname_akamai)"
          exec:
            - ecs_local
            - forward_local
            - _return
        # 非本地解析: 强制非本地解析、位置非中国的域名
        - if: "(query_is_us_domain) || (query_is_non_local_domain)"
          exec:
            - _prefer_ipv4
            - ecs_remote
            - forward_remote
            - ttl_long
            - _return
        # 未知域名 IP 分流
        # 远程获取应答, 丢弃本地 IP 结果
        - primary:
            - _prefer_ipv4
            - ecs_remote
            - forward_remote
            - if: "(response_has_local_ip) || (response_cname_akamai)"
              exec:
                - _drop_response
          # 本地获取应答
          secondary:
            - ecs_local
            - forward_local
            - if: "(! response_has_local_ip) && [_response_valid_answer] || (response_has_gfw_ip)"
              exec:
                - _prefer_ipv4
                - ecs_remote
                - forward_remote
                - ttl_long
          # 避免 DNS 泄漏
          fast_fallback: 2500
          always_standby: false
######### 监听地址 #########
servers:
  - exec: main_sequence
    timeout: 5
    listeners:
      - protocol: udp
        addr: "127.0.0.1:53"
      - protocol: udp
        addr: "[::1]:53"
```

## 测试本地 DNS 运行情况

使用 [q](https://github.com/natesales/q) 作为测试工具，支持 UDP、TCP、DoT、DoH、DoQ 等协议，同时支持设置 ECS。

### 国内网站解析测试

选用浙江电信 202.101.172.35 作为 ECS 地址进行测试。

1. [爱奇艺](https://www.iqiyi.com)
   ![](1673953706420.png)

   - 仅用 32ms 返回结果，解析到了武汉机房

2. bilibili 负载均衡域名：`i0.hdslb.com`、`i1.hdslb.com`、`s1.hdslb.com`
   ![](1673953829048.png)

   ![](1673953833412.png)

   ![](1673953836246.png)

   - 可以看到均能正确解析，且解析时间均在 30ms 左右，非常快
   - IPv4 解析结果均在浙江邻省，可认为 ECS 正常发挥作用

3. [开源中国](https://www.oschina.net)
   ![](1673953912020.png)

   - 解析到了上海，且解析速度较快

### 国外网站解析测试

1. Google 系网站，以 google.com 为例
   ![](1673951541023.png)

   ![](1673951613279.png)

   - 均在 180ms 以内得到正确的解析结果，且解析到了距离洛杉矶代理服务器最近的地点

2. [维基百科](https://www.reddit.com)
   ![](1673951686762.png)

   ![](1673951689851.png)

   - 解析延迟在可接受的范围内

3. [GitHub](https://github.com)
   ![](1673951957703.png)

   ![](1673951964166.png)

   - 200ms 以内得出正确的解析结果，ping 延迟可接受

4. CloudFlare CDN：以 [v2ex 论坛](https://www.v2ex.com) 为例
   ![](1673951994369.png)

   ![](1673952002358.png)

   - 解析用时可接受，由于 Anycast 技术，在各个地方延迟均非常低

## 参考资料

1. [mosdns 官方文档](https://irine-sistiana.gitbook.io/mosdns-wiki/mosdns-v4/ru-he-pei-zhi-mosdns)
2. [Jasper 的博客](https://jasper1024.com/jasper/20211223034622)
3. [EasymosDNS](https://github.com/pmkol/easymosdns)
