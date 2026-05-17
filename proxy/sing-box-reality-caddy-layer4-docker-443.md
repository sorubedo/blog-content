---
title: Docker Compose 编排 sing-box + Caddy：让 VLESS REALITY 与 HTTPS 共享 443
date: 2026-05-17
tags:
  - docker
  - sing-box
  - caddy
  - proxy
description: 用 Caddy L4 插件做 SNI 分流，让 REALITY 流量和 HTTPS 流量在同一端口上——一次从裸机到容器化的迁移记录。
aliases:
  - test
---

## 为什么要折腾

之前我的 sing-box [[VLESS]] + [[XTLS-REALITY]] 节点一直以裸机进程跑在服务器上。稳定倒是稳定，systemd 也能把进程管得服服帖帖，但那台机器上还跑着 Caddy 做反向代理，443 端口被它独占，REALITY 只能灰溜溜地听在其他端口上。

问题在于——REALITY 或 Naive 这类协议跑在高位端口上其实毫无意义。你伪装成在访问 `www.microsoft.com`，结果端口号却是 8443，本身就挺可疑的。更蛋疼的是，sing-box 的 VLESS 入站不像 [[Xray]] 那样自带 fallback 机制，没法在同一个端口上把非代理流量转给 Web 服务。

最近终于决定容器化，把 sing-box 和 Caddy 塞进同一个 `docker-compose.yml`，目标很明确：

- **sing-box 和 Caddy 共享 443 端口**；
- **REALITY 流量走 sing-box**，普通 HTTPS 流量走 Caddy；
- **一个 `docker compose up -d` 就能起整个代理栈**。

Caddy 的 `layer4` 插件正好就是干这个的。

> [!note] 背景交代
> 如果你还不了解 REALITY——简单说就是它通过模仿一个真实 TLS 会话（比如访问 `www.microsoft.com`）来隐藏代理流量，配合 VLESS 的 `xtls-rprx-vision` 流控消除 TLS-in-TLS 的指纹特征，比传统 TLS 伪装更难被识别。

---

## 架构总览

核心思路一句话：**Caddy 在 L4 层接管 443，根据 TLS ClientHello 中的 SNI 做流量分流**。

```
客户端 → Caddy L4 (:443) ── SNI = www.microsoft.com ──→ sing-box (:443) VLESS + REALITY
                          ── 其他 SNI ──────────────→ Caddy 自身 (:8443) HTTPS / 反向代理
```

- 如果 SNI 是 `www.microsoft.com`（REALITY 的伪装目标），流量**原封不动**转发给 sing-box 做正向代理；
- 其他所有 SNI（比如你自己的域名），交给 Caddy 自己的 TLS 层处理。

同一个 443 端口上，REALITY 回落给 sing-box，HTTPS 回落给 Caddy，互不干扰——这就是 L4 分流的魅力。

---

## 文件结构一览

```
/root/docker/Proxy
├── caddy
│   ├── conf
│   │   └── Caddyfile
│   ├── config/
│   ├── data/
│   ├── Dockerfile
│   └── site/
├── compose.yml
└── sing-box
    └── config.json
```

结构很清爽：`caddy/` 放 Caddy 的配置、数据和自定义 Dockerfile，`sing-box/` 只挂一个 `config.json`，`compose.yml` 统筹全局。

---

## 分步配置

### 1. Caddy：L4 分流 + 反向代理

首先 Caddy 需要 `layer4` 插件，官方镜像不带这个，得自己打个包：

```dockerfile
# caddy/Dockerfile
FROM caddy:builder AS builder
RUN xcaddy build --with github.com/mholt/caddy-l4

FROM caddy:latest
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

两阶段构建，编译完把二进制拷进官方镜像，干净利落。

然后是 Caddyfile，这是整套方案的大脑：

```caddy
{
    http_port 80
    https_port 8443
    layer4 {
        :443 {
            @reality tls sni www.microsoft.com
            route @reality {
                proxy sing-box:443
            }
            route {
                proxy 127.0.0.1:8443
            }
        }
    }
}

example.com {
    respond "Hello World!" 200
}
```

逐行拆解：

- **`https_port 8443`**：因为 443 被 `layer4` 占用了，Caddy 自己的 TLS 终止挪到内部 8443，不影响自动签发证书等正常功能；
- **`@reality tls sni www.microsoft.com`**：定义一个匹配器，抓到 SNI 为 `www.microsoft.com` 的 TLS 握手；
- **`route @reality { proxy sing-box:443 }`**：匹配到的流量直接透传给 sing-box 容器（Docker Compose 内部 DNS 会把 `sing-box` 解析成容器 IP）；
- **默认 route `proxy 127.0.0.1:8443`**：不匹配 REALITY 的流量，回送给 Caddy 自己的 HTTPS 端口，走正常的 TLS 终止和反向代理逻辑。

> [!warning] 避坑警告
> `layer4` 块里的 `proxy` 是 L4 透明代理，不解析 HTTP，不修改 TLS。这意味着 sing-box 拿到的就是客户端原始 TLS ClientHello，REALITY 才能正常工作。千万别在中间加任何 TLS 终止。

最后那段 `example.com` 是你自己的站点配置，用来验证 Caddy 的反向代理能力——你可以替换成自己的域名和 `reverse_proxy` 指令。

### 2. sing-box：VLESS + XTLS-REALITY

`sing-box/config.json` 的核心在 inbound 配置：

```json
{
  "log": {
    "level": "info"
  },
  "http_clients": [
    {
      "tag": "direct",
      "engine": "go",
      "version": 2,
      "disable_version_fallback": false
    }
  ],
  "inbounds": [
    {
      "type": "vless",
      "tag": "vless-in",
      "listen": "0.0.0.0",
      "listen_port": 443,
      "users": [
        {
          "name": "sorubedo",
          "uuid": "",
          "flow": "xtls-rprx-vision"
        }
      ],
      "tls": {
        "enabled": true,
        "server_name": "www.microsoft.com",
        "reality": {
          "enabled": true,
          "handshake": {
            "server": "www.microsoft.com",
            "server_port": 443
          },
          "private_key": "",
          "short_id": [""],
          "max_time_difference": "1m"
        }
      }
    }
  ],
  "outbounds": [
    {
      "type": "direct",
      "tag": "direct"
    }
  ],
  "route": {
    "rules": [
      {
        "ip_is_private": true,
        "action": "reject"
      }
    ],
    "final": "direct",
    "auto_detect_interface": true
  }
}
```

几个要点：

- **`listen_port: 443`**，与 Caddy L4 转发的目标端口一致；
- **`flow: "xtls-rprx-vision"`** 是 VLESS 的流控模式，核心作用是消除 TLS-in-TLS 的流量指纹——内层 TLS 的握手特征不再暴露在外层 TLS 的记录长度中，比直接套两层 TLS 安全得多；
- **`tls.server_name` 和 `reality.handshake.server`** 都指向 `www.microsoft.com`——这个域名必须和 Caddyfile 里 `@reality` 匹配的 SNI 一致，不然分流就乱了。示例用微软仅作演示，请根据 VPS 地理位置选择当地大厂的域名，延迟低还不违和；
- **`uuid`、`private_key` 和 `short_id`** 已清空，部署前需自行生成；
- **`route.rules`** 中 `ip_is_private: true → reject` 是安全措施，防止代理被用来扫荡内网资源。

> [!warning] 安全提醒
> 示例中的 `uuid`、`private_key` 和 `short_id` 均为空占位。**部署前请务必替换成自己生成的值并妥善保存**，否则等于把节点拱手送人。

### 3. Docker Compose 编排

`compose.yml` 把两个服务缝在一起：

```yaml
version: "3.8"
services:
  sing-box:
    image: ghcr.io/sagernet/sing-box:latest-testing
    container_name: sing-box
    restart: always
    volumes:
      - ./sing-box:/workdir
    command: -D /workdir run

  caddy:
    build:
      context: ./caddy
      dockerfile: Dockerfile
    container_name: caddy
    restart: always
    cap_add:
      - NET_ADMIN
    ports:
      - "80:80"
      - "80:80/udp"
      - "443:443"
      - "443:443/udp"
    volumes:
      - ./caddy/conf:/etc/caddy
      - ./caddy/site:/srv
      - ./caddy/data:/data
      - ./caddy/config:/config
```

值得注意的细节：

- sing-box 用的是 **`latest-testing`** 镜像——sing-box 迭代快、经常引入破坏式更新，建议关注官方发布说明，别 `docker compose pull` 完才发现配置语法变了。
- sing-box **没有暴露任何端口**到宿主机——流量全部通过 Caddy L4 转发进来，容器之间走 Docker 内部网络，sing-box 对公网完全不可见，安全性 +1。
- Caddy 的 `data` 和 `config` 目录挂载出来，证书和自动续期状态不会因为容器重建而丢失。

---

## 部署

1. **生成密钥和 UUID**（在服务器上）：
   ```bash
   docker run --rm ghcr.io/sagernet/sing-box:latest-testing generate reality-keypair
   docker run --rm ghcr.io/sagernet/sing-box:latest-testing generate uuid
   ```

   把输出的 `private_key`、`public_key` 和 `uuid` 分别填到 `config.json` 和客户端配置中。

2. **启动服务栈**：
   ```bash
   cd /root/docker/Proxy
   docker compose up -d
   ```

---

## 日常维护与更新

部署完不是终点，保持软件版本更新同样重要：

```bash
cd /root/docker/Proxy

# 删除容器，停止服务
docker compose down

# sing-box 拉取最新 testing 镜像
docker compose pull sing-box

# Caddy 需要重新构建（因为有自定义 Dockerfile 编译 L4 插件）
docker compose build caddy

# 启动有变化的服务
docker compose up -d
```

> [!warning] sing-box 更新提醒
> sing-box 的 `latest-testing` 镜像几乎周周都有更新，且**时不时引入破坏式配置变更**。`docker compose pull` 之前建议先看一眼 [sing-box Release Notes](https://github.com/SagerNet/sing-box/releases)，确认没有改你用到的那部分配置语法。别问我怎么知道的。

### 日常运维常用命令

```bash
docker compose logs -f          # 实时查看两个服务的日志
docker compose restart caddy    # 改了 Caddyfile 后单独重启 Caddy
docker exec -w /etc/caddy caddy caddy reload                          #不重启caddy容器，重载caddy配置
docker compose down             # 停掉所有服务
docker compose up -d            # 重新启动
```

### 备份与迁移

整套配置都在 `/root/docker/Proxy` 一个目录下，备份迁移就是一句 `scp` 或 `tar` 的事——这也是容器化最爽的地方：**状态就是文件，文件就是状态**。

---

> [!note] 免责声明
> 本文所述技术方案仅用于**学习 L4 网络分流、Docker 容器编排及反向代理配置**等正当用途。搭建任何网络服务前，请确保你已充分了解并遵守所在地区及服务器所在地的法律法规。本文作者不对任何滥用行为承担责任。用于学习，用于学习，用于学习——重要的事情说三遍。