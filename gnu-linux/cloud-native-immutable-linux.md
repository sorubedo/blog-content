---
title: "云原生不可变 Linux 发行版，为什么是未来"
date: 2026-05-17
tags: [linux, immutable, fedora-atomic, distro]
description: "我造了个没人用的发行版，但我依然坚信不可变基础设施 + 容器化构建是 Linux 桌面发行版的未来。A nobody's take on why cloud-native immutable distros are the next big thing."
---

## 🤔 先回答一个问题：这玩意是啥？

简单说，**[obedur-os](https://github.com/sorubedo/obedur-os)** 是我用 [BlueBuild](https://blue-build.org/) 搓出来的一个 Fedora Atomic 不可变发行版。它基于 OCI 容器镜像构建，系统核心只读，更新原子化，出了问题一个命令就能回滚到上一个版本。

桌面环境是 **[[Niri]]**（一个 Wayland 下的滚动平铺合成器）+ **[[Noctalia Shell]]**，内核换成了 CachyOS 的 LTO 优化版。还顺手做了 SecureBoot 支持、每日 GH Actions 自动构建、cosign 签名验证。

> [!note] 碎碎念
> 说白了就是把 Fedora Silverblue 拿过来，换了层皮，塞了点私货。没什么高深的技术含量，但我觉得这个**方向**是对的。

---

## 🧬 什么叫「云原生不可变原子容器化构建」？

别被这一串 buzzword 吓到，拆开来看其实很简单：

### 不可变（Immutable）

传统 Linux 发行版，系统文件和用户数据混在一起，你装个依赖可能就把 `/usr/lib` 搞得一团糟。哪天某个包的 post-install 脚本抽风了，系统就崩了。

不可变系统的思路是：**系统核心只读挂载，和用户数据物理隔离**。你想改系统？不行。你想装个包？用 Flatpak 或者容器。

> 这就像把操作系统的「C 盘」变成了固件 —— 你往手机上刷固件的时候会去改 `/system` 吗？不会。那桌面 Linux 凭什么还能随便改？

### 原子化更新（Atomic Updates）

传统 `dnf update` 的问题：如果更新到一半断电了，系统可能处于一个薛定谔的状态 —— 半新不旧，既不是旧版本也不是新版本。

原子化更新保证：**要么全部生效，要么完全不生效**。类似于数据库事务，没有中间态。Fedora Atomic 底层用的是 [[Ostree]]，每次更新都是一次完整的树切换。

### 容器化构建（Container-native Build）

这就是 BlueBuild 做的事情：**用 Dockerfile 的方式构建整个操作系统镜像**。

```dockerfile
# 伪代码，但大概就是这么个意思
FROM fedora:44
RUN rpm-ostree install niri ghostty fcitx5
RUN 塞 CachyOS 内核
RUN 我的各种私货配置
BUILD & PUSH → ghcr.io/sorubedo/obedur-os:latest
```

构建产物是一个 OCI 容器镜像，推到 GitHub Container Registry。用户只需要：

```bash
bootc switch ghcr.io/sorubedo/obedur-os:latest
```

就把整个系统切过来了。比你重装系统还快，而且随时能回滚。

---

## 🚀 为什么说这是未来？

### 1. 传统包管理模型已经到极限了

Debian 的 apt、Fedora 的 dnf、Arch 的 pacman……这些包管理器本质上都是**状态机**。系统的最终状态取决于你**按什么顺序**装了什么包。同一套包列表，不同的安装顺序可能导致不同的结果。

[[NixOS]] 和 [[Guix]] 用声明式配置解决了这个问题，但学习曲线陡峭得堪比悬崖。不可变 + 容器化模型则提供了一个折中：**用容器构建时的声明式定义保证可复现，用 ostree 的原子切换保证运行时安全**。

### 2. 桌面 Linux 需要「消费者化」

说句得罪人的话：普通用户不应该需要关心 `libcrypto.so` 的 ABI 兼容性。

macOS 有 sealed system volume，ChromeOS 用 dual-partition A/B 更新，Android 有 dm-verity。桌面 Linux 在这方面的落后不是技术问题，而是没人愿意迈出那一步。

不可变系统让桌面 Linux 变得**更像一个产品，而不是一个拼装玩具**。

### 3. 容器化构建 = 无限可复现

我的 obedur-os 构建流程完全在 GitHub Actions 上跑。每次 push 触发构建，生成的镜像推送到 ghcr.io。这意味着：

- **任何人**都能复现我的系统
- **任何时间**我都能回滚到之前的已发布的任意版本
- 构建日志公开透明，出问题能追溯

这就是 [[DevOps]] 理念在操作系统层的应用：Infrastructure as Code，操作系统镜像也是 Infrastructure。

### 4. 分发就是一次 `docker pull`

传统发行版要维护镜像站、做种、担心 ISO 下载速度。OCI 容器镜像呢？GitHub 帮你托管，全球 CDN 加速，还自带签名验证。

> 你甚至可以用 `skopeo copy` 把镜像搬到任何 registry，真正的去中心化分发。

---

## 🦗 但是，没人在意它

实话实说，obedur-os 目前只有我一个用户。

但这不重要。重要的是**方向是对的**。Fedora Silverblue / Kinoite、openSUSE MicroOS、Vanilla OS、ChromeOS……越来越多人开始拥抱不可变系统。我只是在这个趋势里自己搓了一个满足我个人需求的版本。

> [!warning] 避坑警告
> 如果你想试试 obedur-os：你的 CPU 必须支持 **x86_64-v3**（Intel Haswell / AMD Excavator 及以上）。CachyOS 内核砍掉了老指令集的支持，用老 CPU 启动会直接 kernel panic。

---

## 📖 想了解更多？

完整的文档、安装指南、桌面操作说明都在项目主页：

**[👉 sorubedo.github.io/obedur-os](https://sorubedo.github.io/obedur-os/)**

源代码和构建配置在 GitHub：

**[👉 github.com/sorubedo/obedur-os](https://github.com/sorubedo/obedur-os)**

如果你想自己搓一个类似的发行版，推荐看看 [[BlueBuild]] 的[文档](https://blue-build.org/)，上手比想象中简单很多。你只需要写几个 YAML 文件，剩下的交给 GitHub Actions。

---

> [!quote] 最后
> 造轮子不可耻，可耻的是造了轮子还不写博客吹一下。
> —— 我，一个自己的发行版只有自己用的开发者 🫡
