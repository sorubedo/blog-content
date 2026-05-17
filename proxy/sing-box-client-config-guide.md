---
title: sing-box 客户端配置完全解析
date: 2026-05-17
tags:
  - sing-box
  - proxy
  - android
description: 一份脱敏的 sing-box 客户端配置逐段拆解，涵盖 DNS 分流、入站选择（tun vs tproxy）、自定义规则编写，以及在不同平台上的坑。A desensitized sing-box client config walkthrough.
aliases:
  - sing-box-client
---

## 这篇文章是干嘛的

之前写了 [[sing-box-reality-caddy-layer4-docker-443|服务端部署]]，但这只解决了"梯子怎么搭"。更让人头大的是客户端——sing-box 的客户端配置比服务端长好几倍，DNS 分流、规则匹配、入站模式……新手大概率对着 JSON 一脸懵逼。

本文拿我自己的客户端配置做逐段拆解，**已全部脱敏**，直接复制是跑不起来的，但足够让你理解每一块在干什么。

> [!warning] 重要声明
> 下面这份配置**不能直接用**。UUID、密钥、服务器地址、自定义规则中的域名和 IP 均已替换为占位符。请按需生成自己的值。

---

## 配置文件全景

完整 `config.json` 包含六个大块：`log`、`dns`、`http_clients`、`inbounds`、`outbounds`、`route`，外加 `experimental` 杂项。逐个来看。

---

## DNS 分流

```json
"dns": {
  "servers": [
    {
      "type": "fakeip",
      "tag": "FakeDns",
      "inet4_range": "198.18.0.0/15",
      "inet6_range": "fc00::/18"
    },
    {
      "type": "https",
      "tag": "Local-DNS",
      "server": "223.5.5.5"
    },
    {
      "type": "https",
      "tag": "Remote-DNS",
      "detour": "🚀 节点选择",
      "server": "8.8.8.8"
    }
  ],
  "rules": [
    {
      "clash_mode": "direct",
      "server": "Local-DNS"
    },
    {
      "clash_mode": "global",
      "server": "FakeDns"
    },
    {
      "type": "logical",
      "mode": "and",
      "rules": [
        {
          "domain_suffix": [
            ".lan", ".localdomain", ".example",
            ".invalid", ".localhost", ".test",
            ".local", ".home.arpa",
            ".msftconnecttest.com", ".msftncsi.com"
          ],
          "invert": true
        },
        {
          "query_type": ["A", "AAAA"]
        }
      ],
      "server": "FakeDns"
    },
    {
      "rule_set": "CN-Domain",
      "server": "Local-DNS"
    }
  ],
  "final": "Remote-DNS",
  "strategy": "prefer_ipv4"
}
```

三个 DNS 上游各司其职：

- **FakeDns**：fakeip，对于tun入站很有用，不解释；
- **Local-DNS**：用阿里 DoH（`223.5.5.5`）直接解析，给国内域名用；
- **Remote-DNS**：通过代理节点（`detour: "🚀 节点选择"`）访问 Google DNS，确保获得无污染的解析结果。

规则逻辑：
1. Clash 模式 —— `direct` 直连时用本地 DNS，`global` 全局模式全走 FakeDns；
2. 非内网/非系统探测域名的 A/AAAA 查询，统一走 FakeDns；
3. 命中 `CN-Domain` 规则集的域名用本地 DNS；
4. 兜底走 Remote-DNS。

---

## HTTP 客户端

```json
"http_clients": [
  {
    "tag": "direct",
    "engine": "go",
    "version": 2,
    "stream_receive_window": 0,
    "connection_receive_window": 0
  },
  {
    "tag": "proxy",
    "engine": "go",
    "version": 2,
    "detour": "🚀 节点选择",
    "stream_receive_window": 0,
    "connection_receive_window": 0
  }
]
```

定义了两个 HTTP 客户端实例：`direct` 直连，`proxy` 走代理。用处不多，主要是给规则集中需要 HTTP 拉取的远程资源提供通道。

---

## 入站：最关键的抉择

```json
"inbounds": [
  {
    "type": "mixed",
    "tag": "mixed-in",
    "listen": "0.0.0.0",
    "listen_port": 1080,
    "tcp_fast_open": true,
    "udp_fragment": true
  },
  {
    "type": "socks",
    "tag": "bittorrent-in",
    "listen": "127.0.0.1",
    "listen_port": 1081,
    "tcp_fast_open": true,
    "udp_fragment": true
  },
  {
    "type": "tproxy",
    "tag": "tproxy-in",
    "listen": "::",
    "listen_port": 9898
  }
]
```

三个入站各有用处：

- **mixed-in**（端口 1080）：HTTP/SOCKS 混合入站，给浏览器或单个应用手动设置代理用；
- **bittorrent-in**（端口 1081）：独立的 SOCKS 入站，专门给 BT 客户端，配合路由规则可以让 BT 流量走特定出口；
- **tproxy-in**（端口 9898）：TPROXY 透明代理入站。

### 关于 TPROXY —— 我踩过的坑

> [!warning] TPROXY 劝退警告
> TPROXY 入站**需要 root 权限**，而且你得手动写 iptables/nftables 路由规则把流量导到 `9898` 端口。这在 Linux 桌面端已经够麻烦了——你得处理 IPv4/IPv6 双栈、绕行本机出站流量避免死循环、还要和 NetworkManager 或 systemd-networkd 的路由策略和平共处。
>
> 更惨的是 **Android 端**：各家 ROM 的路由表行为诡异，有些改了规则不生效，有些直接和热点/Wi-Fi 切换干架。我自己使用Tproxy，但我不推荐小白使用它，所以我在这里不做额外教程。

**强烈建议用 [[sing-box-tun|TUN 入站]]替代 TPROXY**：

```json
{
  "type": "tun",
  "tag": "tun-in",
  "interface_name": "SB0",
  "mtu": 9000,
  "address": [
    "172.18.0.1/30",
    "fdfe:dcba:9876::1/126"
  ],
  "auto_route": true,
  "auto_redirect": true,
  "strict_route": true,
  "stack": "mixed"
}
```

TUN 的优势：不需要手写一行路由规则，`auto_route` 自动处理，`auto_redirect` 自动插入nftables规则，提供比 `Tproxy` 性能更好的代理，并且能处理与docker路由规则的冲突

> [!note] Android SFA 用户注意
> 如果你在 Android 上使用 SFA（sing-box for Android），走的是 Android VPN Service，不需要 root。此时**必须移除** `"interface_name"` 和 `"auto_redirect"` 两个字段——它们是 Linux root 模式专属的，放在 VPN Service 模式下会导致启动失败。

---

## 出站：节点与分组

```json
"outbounds": [
  {
    "type": "selector",
    "tag": "🚀 节点选择",
    "outbounds": ["你的节点", "直连"],
    "interrupt_exist_connections": true
  },
  {
    "type": "selector",
    "tag": "🐼 中国大陆",
    "outbounds": ["直连", "🚀 节点选择"],
    "interrupt_exist_connections": true
  },
  {
    "type": "selector",
    "tag": "🧲 Bittorrent",
    "outbounds": ["直连", "🚀 节点选择", "🐼 中国大陆"],
    "interrupt_exist_connections": true
  },
  {
    "type": "direct",
    "tag": "直连"
  }
]
```

出站的核心思路是 **selector 级联**：

- `🚀 节点选择`：最外层选择器，决定"代理流量走哪个节点"；
- `🐼 中国大陆`：国内流量优先直连，不行再走代理；
- `🧲 Bittorrent`：BT 流量的选择器；
- `🐟 漏网之鱼`、`📺 Bilibili`、`🛰 Telegram` 等：按应用/服务细分的选择器，方便对不同服务采用不同策略。

每个 selector 的 `outbounds` 列表就是候选出口，`interrupt_exist_connections: true` 表示切换节点时断开已有连接。

实际节点配置（已脱敏）：

```json
{
  "type": "vless",
  "tag": "你的节点",
  "server": "your-server.example.com",
  "server_port": 443,
  "uuid": "生成你自己的-uuid-替换这里",
  "flow": "xtls-rprx-vision",
  "tls": {
    "enabled": true,
    "server_name": "www.microsoft.com",
    "utls": {
      "enabled": true,
      "fingerprint": "chrome"
    },
    "reality": {
      "enabled": true,
      "public_key": "你的-public-key-替换这里",
      "short_id": "你的-short-id"
    }
  }
}
```

> [!warning] 脱敏说明
> `server`、`uuid`、`public_key`、`short_id` 均已替换为占位符。这些值需要和服务端配置一一对应，用 `sing-box generate reality-keypair` 和 `sing-box generate uuid` 自行生成。

`utls.fingerprint: "chrome"` 让 sing-box 模拟 Chrome 的 TLS 指纹，Reality协议必须配置它。sing-box开发者认为utls不够安全且维护质量差，但不用太在意，想要使用更安全的协议请去部署 `naive` 节点，`naive` 完全复用了 chrome 网络栈，这里不做教程。

---

## 路由规则

路由是整个配置的灵魂，决定了"什么流量走哪"：

```json
"route": {
  "rules": [
    { "action": "sniff" },
    { "protocol": "dns", "action": "hijack-dns" },
    { "ip_is_private": true, "outbound": "直连" },
    { "clash_mode": "direct", "outbound": "直连" },
    { "clash_mode": "global", "outbound": "GLOBAL" },
    {
      "type": "logical",
      "mode": "or",
      "rules": [
        { "protocol": "bittorrent" },
        { "inbound": "bittorrent-in" }
      ],
      "outbound": "🧲 Bittorrent"
    },
    { "rule_set": "Custom-Domain", "outbound": "🛠 Custom" },
    { "rule_set": "Bilibili-Domain", "outbound": "📺 Bilibili" },
    { "rule_set": "Telegram-Domain", "outbound": "🛰 Telegram" },
    { "rule_set": "Google-Domain", "outbound": "🌈 Google" },
    { "rule_set": "CN-Domain", "outbound": "🐼 中国大陆" },
    { "rule_set": "!CN-Location-Domain", "outbound": "🚀 节点选择" },
    { "action": "resolve" },
    { "rule_set": "Custom-IP", "outbound": "🛠 Custom" },
    { "rule_set": "Telegram-IP", "outbound": "🛰 Telegram" },
    { "rule_set": "Google-IP", "outbound": "🌈 Google" },
    { "rule_set": "CN-IP", "outbound": "🐼 中国大陆" }
  ],
  "final": "🐟 漏网之鱼",
  "auto_detect_interface": true,
  "default_domain_resolver": {
    "server": "Local-DNS",
    "strategy": "prefer_ipv4"
  },
  "default_http_client": "direct"
}
```

规则**从上到下依次匹配**，命中即停止：

1. **`sniff`**：嗅探流量协议类型；
2. **`hijack-dns`**：劫持 DNS 请求到 sing-box 内置 DNS 模块，防止 DNS 泄露；
3. **私有 IP** 直连；
4. **Clash 模式** 路由（`direct` → 直连，`global` → 全局走代理）；
5. **BT 流量** 走 `🧲 Bittorrent` 选择器；
6. **域名规则集**（Custom、Bilibili、Telegram、Google、CN 等）匹配到对应出站；
7. **`!CN-Location-Domain`**：非中国地区的域名，走代理；
8. **`action: resolve`**：在此之上的规则用域名匹配，之下的规则用 IP 匹配——防止无必要的本地dns解析。但是域名是点，IP是面，用IP规则兜底还是很重要的；
9. **IP 规则集**：按解析后的 IP 做二次分流；
10. **兜底**：`🐟 漏网之鱼` 处理所有未匹配的流量。

### 规则集定义

规则集主体是远程拉取的二进制 SRS 文件，由 [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat) 维护：

```json
"rule_set": [
  {
    "type": "remote",
    "tag": "CN-Domain",
    "url": "https://ghfast.top/https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/sing/geo/geosite/cn.srs",
    "update_interval": "2h0m0s"
  },
  {
    "type": "remote",
    "tag": "Google-Domain",
    "url": "https://ghfast.top/https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/sing/geo/geosite/google.srs",
    "update_interval": "2h0m0s"
  },
  {
    "type": "local",
    "tag": "Custom-Domain",
    "path": "./rules/custom-domain.json"
  },
  {
    "type": "local",
    "tag": "Custom-IP",
    "path": "./rules/custom-ip.json"
  }
]
```

远程规则每两小时自动更新。重点说说两个**本地自定义规则**。

---

## 自定义规则

自定义规则放在 **sing-box 的工作区目录**（即 `-D` 参数指定的目录）下的 `rules/` 文件夹中。

路径取决于你的运行方式：

| 平台                          | 工作区路径                | 规则文件位置                     |     |
| :-------------------------- | :------------------- | :------------------------- | --- |
| Linux（包管理器安装自带的systemd服务单元） | `/var/lib/sing-box/` | `/var/lib/sing-box/rules/` |     |
| SFA（Android）                | 设置 → 核心 → 打开工作区      | `工作区/rules/`               |     |
| 手动命令行                       | `-D` 指定的任意路径         | `<指定路径>/rules/`            |     |

### 域名规则示例 `rules/custom-domain.json`

```json
{
  "version": 5,
  "rules": [
    {
      "domain": [
        "my-blog.example.com"
      ],
      "domain_suffix": [
        ".my-blog.example.com"
      ]
    }
  ]
}
```

> [!note] 脱敏说明
> 示例中的域名已替换。你可以填入任何你需要自定义路由的域名，比如自己的网站、小众服务等。

### IP 规则示例 `rules/custom-ip.json`

```json
{
  "version": 5,
  "rules": [
    {
      "ip_cidr": [
        "192.0.2.1"
      ]
    }
  ]
}
```

> [!note] 脱敏说明
> 示例中的 IP 为 RFC 5737 文档保留地址。按需替换成你自己需要指定路由的 IP 或 CIDR 段。

这两个文件会被 `route.rules` 中的 `Custom-Domain` 和 `Custom-IP` 规则引用，匹配到的流量走 `🛠 Custom` 选择器，方便你随时增删自定义条目而不动主配置。

---

## 实验性功能

```json
"experimental": {
  "cache_file": {
    "enabled": true,
    "path": "cache.db",
    "store_fakeip": true,
    "rdrc_timeout": "168h0m0s",
    "store_dns": true
  },
  "clash_api": {
    "external_controller": "127.0.0.1:23333",
    "external_ui": "./ui",
    "external_ui_download_url": "https://ghfast.top/https://github.com/Zephyruso/zashboard/releases/latest/download/dist.zip",
    "secret": "你的密码",
    "default_mode": "rule",
    "access_control_allow_origin": "*"
  }
}
```

- **cache_file**：缓存 FakeDNS 映射和 DNS 记录到 `cache.db`，重启不丢，`rdrc_timeout: 168h` 表示缓存一周；
- **clash_api**：开启 REST API，配合 Zashboard 面板做可视化管理。`external_controller` 监听 `127.0.0.1:23333`，面板通过它切换节点、查看连接。

> [!warning] 安全提醒
> `secret` 已替换为占位符。这个密码是面板登录凭证，请设为强密码并妥善保管。如果要从其他设备访问请把 `external_controller` 绑到 `0.0.0.0` 上。

---

## 总结

这份客户端配置的核心思路：

1. **DNS**：FakeDns 做域名映射，国内走阿里 DoH，国外走 Google DNS + 代理；
2. **入站**：推荐 TUN 入站，省去手动路由的折磨；Android 无 root 用 VPN Service 模式；
3. **出站**：selector 级联，按服务粒度灵活调度；
4. **路由**：域名规则 → Dns解析 → IP 规则，层层过滤，兜底走 `漏网之鱼`；
5. **自定义规则**：`rules/` 目录下的 JSON 文件，独立维护，不污染主配置。

> [!note] 碎碎念
> 折腾代理三年，我的心得是：**配置越简洁，出问题越容易排查**。不要为了"全覆盖"把规则写得比正则还密，留一些漏网之鱼反而活得更舒服。毕竟漏网之鱼兜底的 `final` 出站选得好，其实也没什么影响。
>
> 如果你还没看服务端那篇，可以移步 [[sing-box-reality-caddy-layer4-docker-443]]。

---

## 完整配置参考

以下为脱敏后的完整 `config.json`，供对照查阅：

```json
{
  "log": {
    "level": "info"
  },
  "dns": {
    "servers": [
      {
        "type": "fakeip",
        "tag": "FakeDns",
        "inet4_range": "198.18.0.0/15",
        "inet6_range": "fc00::/18"
      },
      {
        "type": "https",
        "tag": "Local-DNS",
        "server": "223.5.5.5"
      },
      {
        "type": "https",
        "tag": "Remote-DNS",
        "detour": "🚀 节点选择",
        "server": "8.8.8.8"
      }
    ],
    "rules": [
      {
        "clash_mode": "direct",
        "server": "Local-DNS"
      },
      {
        "clash_mode": "global",
        "server": "FakeDns"
      },
      {
        "type": "logical",
        "mode": "and",
        "rules": [
          {
            "domain_suffix": [
              ".lan",
              ".localdomain",
              ".example",
              ".invalid",
              ".localhost",
              ".test",
              ".local",
              ".home.arpa",
              ".msftconnecttest.com",
              ".msftncsi.com",
              ".market.xiaomi.com",
              ".wotgame.cn",
              ".wggames.cn",
              ".wowsgame.cn",
              ".wargaming.net",
              ".steamcontent.com"
            ],
            "invert": true
          },
          {
            "query_type": [
              "A",
              "AAAA"
            ]
          }
        ],
        "server": "FakeDns"
      },
      {
        "rule_set": "CN-Domain",
        "server": "Local-DNS"
      }
    ],
    "final": "Remote-DNS",
    "strategy": "prefer_ipv4"
  },
  "http_clients": [
    {
      "tag": "direct",
      "engine": "go",
      "version": 2,
      "stream_receive_window": 0,
      "connection_receive_window": 0
    },
    {
      "tag": "proxy",
      "engine": "go",
      "version": 2,
      "detour": "🚀 节点选择",
      "stream_receive_window": 0,
      "connection_receive_window": 0
    }
  ],
  "inbounds": [
    {
      "type": "mixed",
      "tag": "mixed-in",
      "listen": "0.0.0.0",
      "listen_port": 1080,
      "tcp_fast_open": true,
      "udp_fragment": true
    },
    {
      "type": "socks",
      "tag": "bittorrent-in",
      "listen": "127.0.0.1",
      "listen_port": 1081,
      "tcp_fast_open": true,
      "udp_fragment": true
    },
    {
      "type": "tproxy",
      "tag": "tproxy-in",
      "listen": "::",
      "listen_port": 9898
    }
  ],
  "outbounds": [
    {
      "type": "selector",
      "tag": "🚀 节点选择",
      "outbounds": [
        "你的节点",
        "SSH-备用",
        "直连"
      ],
      "interrupt_exist_connections": true
    },
    {
      "type": "selector",
      "tag": "🐼 中国大陆",
      "outbounds": [
        "直连",
        "🚀 节点选择"
      ],
      "interrupt_exist_connections": true
    },
    {
      "type": "selector",
      "tag": "🧲 Bittorrent",
      "outbounds": [
        "直连",
        "🚀 节点选择",
        "🐼 中国大陆"
      ],
      "interrupt_exist_connections": true
    },
    {
      "type": "selector",
      "tag": "🐟 漏网之鱼",
      "outbounds": [
        "🚀 节点选择",
        "🐼 中国大陆",
        "直连"
      ],
      "interrupt_exist_connections": true
    },
    {
      "type": "selector",
      "tag": "📺 Bilibili",
      "outbounds": [
        "🐼 中国大陆",
        "🚀 节点选择",
        "直连"
      ],
      "interrupt_exist_connections": true
    },
    {
      "type": "selector",
      "tag": "🛰 Telegram",
      "outbounds": [
        "🚀 节点选择",
        "🐼 中国大陆",
        "直连"
      ],
      "interrupt_exist_connections": true
    },
    {
      "type": "selector",
      "tag": "💻 Github",
      "outbounds": [
        "🚀 节点选择",
        "🐼 中国大陆",
        "直连"
      ],
      "interrupt_exist_connections": true
    },
    {
      "type": "selector",
      "tag": "🪟 Microsoft",
      "outbounds": [
        "🐼 中国大陆",
        "🚀 节点选择",
        "直连"
      ],
      "interrupt_exist_connections": true
    },
    {
      "type": "selector",
      "tag": "🎥 Youtube",
      "outbounds": [
        "🚀 节点选择",
        "🐼 中国大陆",
        "直连"
      ],
      "interrupt_exist_connections": true
    },
    {
      "type": "selector",
      "tag": "🌈 Google",
      "outbounds": [
        "🚀 节点选择",
        "🐼 中国大陆",
        "直连"
      ],
      "interrupt_exist_connections": true
    },
    {
      "type": "selector",
      "tag": "🍎 Apple",
      "outbounds": [
        "🐼 中国大陆",
        "🚀 节点选择",
        "直连"
      ],
      "interrupt_exist_connections": true
    },
    {
      "type": "selector",
      "tag": "GLOBAL",
      "outbounds": [
        "🚀 节点选择",
        "直连",
        "🐼 中国大陆"
      ],
      "interrupt_exist_connections": true
    },
    {
      "type": "selector",
      "tag": "🛠 Custom",
      "outbounds": [
        "🚀 节点选择",
        "直连",
        "🐼 中国大陆"
      ],
      "interrupt_exist_connections": true
    },
    {
      "type": "direct",
      "tag": "直连"
    },
    {
      "type": "vless",
      "tag": "你的节点",
      "server": "your-server.example.com",
      "server_port": 443,
      "uuid": "生成你自己的-uuid-替换这里",
      "flow": "xtls-rprx-vision",
      "tls": {
        "enabled": true,
        "server_name": "www.microsoft.com",
        "utls": {
          "enabled": true,
          "fingerprint": "chrome"
        },
        "reality": {
          "enabled": true,
          "public_key": "你的-public-key-替换这里",
          "short_id": "你的-short-id"
        }
      }
    },
    {
      "type": "ssh",
      "tag": "SSH-备用",
      "server": "your-server.example.com",
      "server_port": 22,
      "user": "root",
      "password": "你的密码"
    }
  ],
  "route": {
    "rules": [
      {
        "action": "sniff"
      },
      {
        "protocol": "dns",
        "action": "hijack-dns"
      },
      {
        "ip_is_private": true,
        "outbound": "直连"
      },
      {
        "clash_mode": "direct",
        "outbound": "直连"
      },
      {
        "clash_mode": "global",
        "outbound": "GLOBAL"
      },
      {
        "type": "logical",
        "mode": "or",
        "rules": [
          {
            "protocol": "bittorrent"
          },
          {
            "inbound": "bittorrent-in"
          }
        ],
        "outbound": "🧲 Bittorrent"
      },
      {
        "rule_set": "Custom-Domain",
        "outbound": "🛠 Custom"
      },
      {
        "rule_set": "Bilibili-Domain",
        "outbound": "📺 Bilibili"
      },
      {
        "rule_set": "Telegram-Domain",
        "outbound": "🛰 Telegram"
      },
      {
        "rule_set": "Github-Domain",
        "outbound": "💻 Github"
      },
      {
        "rule_set": "Microsoft-Domain",
        "outbound": "🪟 Microsoft"
      },
      {
        "rule_set": "Youtube-Domain",
        "outbound": "🎥 Youtube"
      },
      {
        "rule_set": "Google-Domain",
        "outbound": "🌈 Google"
      },
      {
        "rule_set": "Apple-Domain",
        "outbound": "🍎 Apple"
      },
      {
        "rule_set": "CN-Domain",
        "outbound": "🐼 中国大陆"
      },
      {
        "rule_set": "!CN-Location-Domain",
        "outbound": "🚀 节点选择"
      },
      {
        "action": "resolve"
      },
      {
        "rule_set": "Custom-IP",
        "outbound": "🛠 Custom"
      },
      {
        "rule_set": "Bilibili-IP",
        "outbound": "📺 Bilibili"
      },
      {
        "rule_set": "Telegram-IP",
        "outbound": "🛰 Telegram"
      },
      {
        "rule_set": "Google-IP",
        "outbound": "🌈 Google"
      },
      {
        "rule_set": "Apple-IP",
        "outbound": "🍎 Apple"
      },
      {
        "rule_set": "CN-IP",
        "outbound": "🐼 中国大陆"
      }
    ],
    "rule_set": [
      {
        "type": "local",
        "tag": "Custom-Domain",
        "path": "./rules/custom-domain.json"
      },
      {
        "type": "local",
        "tag": "Custom-IP",
        "path": "./rules/custom-ip.json"
      },
      {
        "type": "remote",
        "tag": "Github-Domain",
        "url": "https://ghfast.top/https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/sing/geo/geosite/github.srs",
        "update_interval": "2h0m0s"
      },
      {
        "type": "remote",
        "tag": "Microsoft-Domain",
        "url": "https://ghfast.top/https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/sing/geo/geosite/microsoft.srs",
        "update_interval": "2h0m0s"
      },
      {
        "type": "remote",
        "tag": "Youtube-Domain",
        "url": "https://ghfast.top/https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/sing/geo/geosite/youtube.srs",
        "update_interval": "2h0m0s"
      },
      {
        "type": "remote",
        "tag": "Google-Domain",
        "url": "https://ghfast.top/https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/sing/geo/geosite/google.srs",
        "update_interval": "2h0m0s"
      },
      {
        "type": "remote",
        "tag": "Google-IP",
        "url": "https://ghfast.top/https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/sing/geo/geoip/google.srs",
        "update_interval": "2h0m0s"
      },
      {
        "type": "remote",
        "tag": "Bilibili-Domain",
        "url": "https://ghfast.top/https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/sing/geo/geosite/bilibili.srs",
        "update_interval": "2h0m0s"
      },
      {
        "type": "remote",
        "tag": "Bilibili-IP",
        "url": "https://ghfast.top/https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/sing/geo-lite/geoip/bilibili.srs",
        "update_interval": "2h0m0s"
      },
      {
        "type": "remote",
        "tag": "Telegram-Domain",
        "url": "https://ghfast.top/https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/sing/geo/geosite/telegram.srs",
        "update_interval": "2h0m0s"
      },
      {
        "type": "remote",
        "tag": "Telegram-IP",
        "url": "https://ghfast.top/https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/sing/geo/geoip/telegram.srs",
        "update_interval": "2h0m0s"
      },
      {
        "type": "remote",
        "tag": "CN-Domain",
        "url": "https://ghfast.top/https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/sing/geo/geosite/cn.srs",
        "update_interval": "2h0m0s"
      },
      {
        "type": "remote",
        "tag": "CN-IP",
        "url": "https://ghfast.top/https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/sing/geo/geoip/cn.srs",
        "update_interval": "2h0m0s"
      },
      {
        "type": "remote",
        "tag": "!CN-Location-Domain",
        "url": "https://ghfast.top/https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/sing/geo/geosite/geolocation-!cn.srs",
        "update_interval": "2h0m0s"
      },
      {
        "type": "remote",
        "tag": "Apple-Domain",
        "url": "https://ghfast.top/https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/sing/geo/geosite/apple.srs",
        "update_interval": "2h0m0s"
      },
      {
        "type": "remote",
        "tag": "Apple-IP",
        "url": "https://ghfast.top/https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/sing/geo-lite/geoip/apple.srs",
        "update_interval": "2h0m0s"
      }
    ],
    "final": "🐟 漏网之鱼",
    "find_process": true,
    "auto_detect_interface": true,
    "default_domain_resolver": {
      "server": "Local-DNS",
      "strategy": "prefer_ipv4"
    },
    "default_http_client": "direct"
  },
  "experimental": {
    "cache_file": {
      "enabled": true,
      "path": "cache.db",
      "store_fakeip": true,
      "rdrc_timeout": "168h0m0s",
      "store_dns": true
    },
    "clash_api": {
      "external_controller": "127.0.0.1:23333",
      "external_ui": "./ui",
      "external_ui_download_url": "https://ghfast.top/https://github.com/Zephyruso/zashboard/releases/latest/download/dist.zip",
      "secret": "你的密码",
      "default_mode": "rule",
      "access_control_allow_origin": "*"
    }
  }
}
```

> [!note] 关于完整配置
> 1. SSH 节点配置最简单、协议也足够安全，缺点是速度慢，这里用作备用。
> 2. 所有 `rule_set` 中的远程 URL 使用了 `ghfast.top` 作为 GitHub 加速代理，在国内网络环境下可直连拉取。如果网络环境不同，可以去掉前缀直接使用 `https://raw.githubusercontent.com/...`。
> 3. 这份配置**仍然不能直接用**——把tproxy入站换成自己需要的入站，如tun入站，把带中文占位符的值替换成你自己的，再用 `sing-box check -c /path/to/ur/config.json` 验证一遍。
