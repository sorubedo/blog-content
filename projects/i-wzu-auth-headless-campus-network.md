---
title: "i-wzu-auth：给旧手机写了个校园网 CLI，然后它继续吃灰了"
date: 2026-05-18
tags: [rust, headless, campus-network]
description: "用 Rust + Gemini CLI 搓了个无头校园网认证工具，cargo-dist 一键发布全平台，结果栽在了手机网卡上。A headless campus network auth CLI built with Rust, released via cargo-dist — defeated by terrible WiFi hardware."
---

## 起因：一台吃灰的旧手机

我有一台 Redmi Note 4X (mido)，骁龙 625，金属机身，2017 年的老家伙。给它刷上了 [postmarketOS](https://postmarketos.org/)，没装图形环境，想让它当个低功耗小服务器跑着玩。

问题来了：校园网是开放 WiFi，没有 WPA 加密，但连接后需要进网页登录认证。没 GUI = 没浏览器 = 登不了。

原本的土办法是：在另一台设备上把自己的 MAC 改成手机的 MAC，浏览器登录后再改回去。每次断线重连都得来一遍，麻烦得想死。

> [!note] 碎碎念
> 温大的校园网用的是深澜 (Srun) 认证系统。这玩意很多高校都在用，协议本身不复杂，但官方不给 CLI 客户端——毕竟正常人谁会拿命令行上网？

## 方案：搓个 CLI 工具

需求很明确：

1. 能在终端里一键登录/注销
2. 能查看当前在线状态（流量、时长、在线设备）
3. 最好能支持 `crontab` 定时检查断线自动重连

语言选了 Rust——编译出来单二进制，扔进 `/usr/local/bin` 就能跑，不需要运行时依赖。而且顺便练练 Rust 手感。

> [!warning] 避坑警告
> 深澜的 Srun 协议不是标准的 HTTP Basic Auth。它涉及 XXTEA 加密 + 自定义 Base64 编码 + HMAC-MD5 签名，外加 JSONP 回调解析。你得先 `get_challenge` 拿 token，再用 token 加密用户名密码拼成 `info` 参数，最后算 `chksum` 签名一起发过去。缺一步都登不上。

## 实现：Gemini CLI 当副驾

整个项目大概 500 行 Rust，核心就三个文件：

- `main.rs` — CLI 入口，[clap](https://crates.io/crates/clap) 解析参数，三个子命令 `login` / `logout` / `status`
- `api.rs` — Srun 协议实现，`get_challenge` → `login` / `check_info` / `logout`
- `crypto.rs` — XXTEA 加解密、自定义 Base64、HMAC-MD5、SHA1

加密这块是最恶心的。XXTEA 的正确实现、深澜魔改的 Base64 字符表 `LVoJPiCN2R8G90yg+...`，一步步对着 Wireshark 抓包和浏览器 JS 源码调，Gemini CLI 帮了不少忙——虽然偶尔也会胡说八道，但在这种「对着已知算法写 Rust」的场景下还算靠谱。

### 几个有意思的设计点

**自动检测在线状态**：`check_online()` 用 `ureq` GET `http://www.google.cn/generate_204`，超时 3 秒。

**注销逻辑分两步**：先是 `rad_user_dm` 解除 MAC 无感绑定，再 `srun_portal` 标准 Session 注销。缺第一步会导致「服务器明明返回注销成功但马上又登录了」的玄学 bug。

**友好的中文输出 + emoji**：毕竟是给自己用的，状态查询把累计流量/时长换算成可读格式，在线设备列表也做了 IP + OS 解析。彩色的 `colored` crate 加持，看着比 `curl` 舒服多了。

```bash
# 登录
i-wzu-auth login -u 学号 -p 密码

# 查状态
i-wzu-auth status

# 注销
i-wzu-auth logout
```

要注意的是，bash 等 shell 会把命令记录写到文件里，这会泄露密码，以后也许改成 stdin 交互式输入。还支持环境变量 `SRUN_USER` / `SRUN_PASS` / `SRUN_URL`
## cargo-dist + GitHub Actions: 一键全平台发布

这是整个项目最爽的部分。

用 [cargo-dist](https://opensource.axo.dev/cargo-dist/) 配置好 `dist-workspace.toml`，写好目标平台：

```toml
targets = [
  "aarch64-apple-darwin", "x86_64-apple-darwin",
  "aarch64-unknown-linux-gnu", "x86_64-unknown-linux-gnu",
  "x86_64-unknown-linux-musl", "aarch64-unknown-linux-musl",
  "x86_64-pc-windows-msvc"
]
```

然后 `dist init` 自动生成 GitHub Actions workflow。之后每次发布只需：

```bash
git tag v0.1.3
git push origin v0.1.3
```

剩下的全自动——GitHub Actions 拉起来，并行编译 Linux (glibc/musl) + macOS (Intel/Apple Silicon) + Windows，生成 tar.gz/zip + 安装脚本，最后 `gh release create` 发布到 Release 页面。

> [!note] 体验
> 以前手动搞跨平台发布：每个系统装一次编译工具链，打包，生成 checksum，写 Release Notes，上传……现在打个 tag 推上去就完了。cargo-dist 这套流程对于 Rust CLI 项目来说基本是标配级体验了。

## 结局：硬件不给力

工具写好了，Release 也发布了，mido 也能跑了——然后发现：**这破手机的 WiFi 模块太垃圾了。**

mido 的 WCNSS 网卡配合金属机身，信号本来就差。连接校园网之后，出站请求能发出去，入站延迟巨大——ping 网关能到 2000ms+，SSH 都卡成幻灯片。根本没法用。

> [!note] 当代西西弗斯
> 花了两天写工具，省下了每次重连的 30 秒手动操作，然后发现设备本身不适合这个用途。这就是折腾的真谛啊朋友们：为了解决一个 30 秒的问题，花了两天时间，最后问题没解决，但学会了一堆东西。

## 收获

虽然手机继续吃灰，但这段经历不亏：

- 完整走了一遍 **Rust CLI 项目从零到发布**的流程
- **cargo-dist + GitHub Actions** 这套 CI/CD 是真的香，以后所有 CLI 项目都会沿用
- 逆向理解了 Srun 认证协议（XXTEA + 魔改 Base64 + JSONP）
- 再次确认了一件事：**硬件不行，软件再好也白搭**

项目地址：[github.com/sorubedo/i-wzu-auth](https://github.com/sorubedo/i-wzu-auth)，MIT 协议。温大的同学自取，如果你的设备网卡没那么烂的话应该挺好用的 🙃
