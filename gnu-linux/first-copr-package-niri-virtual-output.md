---
title: "第一次维护 COPR 包：niri 虚拟输出 + NVIDIA 翻车实录"
date: 2026-05-17
tags: [fedora, copr, niri, nvidia, wayland]
description: "人生中第一次维护一个 Linux 软件包，就为了给我的 iPad 搞个完美分辨率的虚拟显示器。结果是包跑起来了，显示器也有了，但 NVIDIA 不给力。First COPR packaging experience: niri virtual output works, NVIDIA doesn't."
---

## 📦 起因：我想要一个虚拟显示器

事情是这样的：我有一台 iPad，偶尔用 [Moonlight](https://moonlight-stream.org/) 串流到主力机上玩游戏或者远程桌面。Moonlight 的服务端 [Sunshine](https://github.com/LizardByte/Sunshine) 需要抓取画面 —— 正常情况下它抓你物理显示器的画面，分辨率就是显示器多大就多大。

但我的 iPad 是 4:3 比例，物理显示器是 16:10。串过去要么有黑边，要么拉伸变形，看着难受。

解决方案很明确：**搞一个虚拟显示器，分辨率完美匹配 iPad 屏幕，让 Sunshine 抓这个虚拟输出就行了。**

[niri](https://github.com/YaLTeR/niri) 的 [PR #3800](https://github.com/niri-wm/niri/pull/3800) 正好在做这件事 —— 给 niri 添加虚拟输出（headless display）支持。通过 IPC 命令就能动态创建/删除虚拟显示器：

```bash
niri msg create-virtual-output --name sunshine --width 1920 --height 1440 --refresh-rate 60
```

完美。问题只剩下一个：这个 PR 还没合并进上游。

> [!note] 碎碎念
> PR #3800 是 [willybarret](https://github.com/willybarret) 提交的，功能相当完整：支持命名输出、IPC 控制、config 声明式创建、headless 和 TTY 双后端。社区反响也很好（18 👍 11 ❤️），但截至我写这篇文章的时候还开着，没合。

---

## 🛠️ 于是，我搞了个 COPR

反正平时也用 [obedur-os](https://github.com/sorubedo/obedur-os)，Fedora 生态的东西多少得会一点。干脆把 niri + PR #3800 打包成 COPR，既能自己用，也算第一次真正「维护」一个包。

### 做了什么

1. Fork 了 niri，建了个 [`build` 分支](https://github.com/sorubedo/niri/tree/build)：跟踪上游 main，rebase 后 cherry-pick PR #3800 的提交
2. niri 上游仓库自带 RPM spec 文件，不用自己写 —— 最头疼的一步直接跳过 👍
3. 在 COPR 上创建项目，配好 chroots 构建目标平台，设置 Webhook：`build` 分支一有推送，COPR 自动拉源码编译打包

就这么简单。没有手写 CI 配置，没有依赖地狱，没有"在我机器上能编"。

> **GitHub 托管代码和 Webhook，Red Hat 的 COPR 负责编译分发。** 我做的事：fork → 合并 PR → 配 Webhook。剩下全是自动的。

### Fedora 的基础设施是真的好用

实话实说，之前一直觉得「打包」是个很重的事情。Arch 要写 PKGBUILD（其实还好），Debian 要搞一堆 control 文件，Gentoo 的 ebuild 我看一眼就头疼。

但这次 COPR 给我的体验是这样的：

1. 上游仓库自带 spec，直接用
2. 推到 GitHub —— Webhook 通知 COPR
3. COPR 自动拉源码、编译、打包、发布
4. 用户 `dnf copr enable sorubedo/niri-pull3800-git && dnf install niri`

全程零运维。编译环境、签名、仓库托管 Red Hat 全包了。对于我这种"想用某个未合并 feature 又不想每次手动编译"的人来说，简直是救星。

> [!note] 感慨
> GitHub + COPR 这套组合拳，让一个没维护过任何包的人，花一个下午就能拥有自己的软件源。开源基础设施的强大之处就在这 —— **不是让你少写代码，而是让你只写必要的代码。**

---

## 🎯 理想的使用场景

本来计划是这样的：

> iPad Moonlight 客户端 → 网络串流 → Sunshine wlroots 捕获 → niri 虚拟输出（2160×1620，完美匹配 iPad 分辨率）

- iPad 获得原生分辨率，无黑边无拉伸
- 虚拟输出独立于物理显示器，不会暴露桌面隐私
- Sunshine 可配置为只抓这个虚拟输出

一切看起来都很美好。然后 NVIDIA 登场了。

---

## 💀 Fuck You, NVIDIA

Sunshine 有多种捕获模式：

### 方案 A：wlroots 捕获

```ini
# sunshine.conf
capture = wlr
```

这是推荐做法，niri 虽然是基于 smithay 的而不是wlroots，但也已经支持 wlr-screencopy 协议。Sunshine 通过 wlroots 协议直接抓取合成器画面。理论上应该完美支持虚拟输出。

**实际结果**：

```
[wayland] Frame capture failed
[wayland] Frame capture failed
[wayland] Frame capture failed
[wayland] Frame capture failed
...
```

无限刷屏。日志疯狂滚动，内存一路飙升直到 OOM。而我 ipad 上黑屏啥也看不到，NVIDIA 驱动实现大概是用脚写的。

### 方案 B：KMS 捕获

```ini
# sunshine.conf
capture = kms
```

KMS 模式直接抓显存（DRM），不走 wlroots 协议层。

**实际结果**：KMS 能抓物理显示器，但**抓不到 Wayland 层级的虚拟输出**。虚拟输出只存在于合成器的逻辑层，不经过 DRM/KMS，所以 KMS 捕获根本看不到它。

### 死胡同

| 方案         | 结果                             |
| :--------- | :----------------------------- |
| wlroots 捕获 | Frame capture failed 刷屏 → 内存爆炸 |
| KMS 捕获     | 看不到虚拟输出                        |

两个方案都走不通。NVIDIA 的 Wayland 支持就是一座围城 —— 外面的人想进来，里面的人想出去。

---

## 📌 现在的状态

这个 COPR 包我还在维护，`build` 分支有更新就会自动构建。Fedora 用户想要用 niri + 虚拟输出的话，一个命令就能装上：

```bash
dnf copr enable sorubedo/niri-pull3800-git
dnf install niri
```

但我自己暂时用不上虚拟输出的功能。

等什么时候 NVIDIA 把 Wayland 支持修好了，或者 sunshine/niri 适配了，或者我换 AMD 显卡了（那一天可能不远了），这套组合就能真正派上用场。

---

> [!quote] 最后
> 第一次维护一个包的感觉，有点像第一次把你的代码推上 GitHub —— 不是什么惊天动地的大事，但你确实参与了整个开源生态的运作。
>
> 至于 NVIDIA？
>
> **Fuck you, NVIDIA.** 🖕🐧
