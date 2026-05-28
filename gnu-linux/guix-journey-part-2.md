---
title: "Guix 一日体验：我放弃了"
date: 2026-05-25
tags:
  - linux
  - guix
  - arch
  - distro
description: "花了一天了解 Guix，结论是：还是 Arch 香。A one-day Guix investigator's report on why functional package management isn't worth the pain (for me)."
---

## 上集回顾：我要去 Guix 了！

昨天的 [[gnu-linux/guix-journey-part-1|告别 Fedora Atomic，我决定试试 Guix]] 里，我信誓旦旦地说要拥抱 GNU Guix，开启"从零开始的 Guix 生活"。写那篇文章的时候，我已经把 Guix 的 ISO 下好了，Virtual Machine Manager 也打开了。

然后我花了一天时间认真了解 Guix。

结论：**我不去了。**

> [!note] 这不算打脸
> 昨天那篇本来就是连载预告，第一篇就说了"先在虚拟机里试水"。我试了，水太冰了，我上岸了。这叫及时止损，不叫打脸。

## Guix 到底哪里劝退了我

先说清楚：Guix 的技术理念没有问题。函数式包管理、声明式系统配置、事务性升级、可复现构建——这些都很美。问题出在**生态与现实的摩擦面**。

### 严格到底的 GNU FSDG

Guix 是 GNU 官方项目，严格遵守 [GNU Free System Distribution Guidelines](https://www.gnu.org/distros/free-system-distribution-guidelines.html)。这意味着官方 channel **只包含自由软件**。

这本身不是坏事。问题在于，FSDG 的"自由"标准非常激进：

- **Prism Launcher**：本身开源的 Minecraft 启动器，但因为用途服务于非自由软件（Minecraft），永远不会被纳入官方 channel。而 NixOS 的 nixpkgs 有它。
- 很多日常软件的依赖链里夹杂了非自由组件，在 Guix 的世界里，你只能自己打包。

> [!warning] 自己打包"几个"软件？
> 一开始你觉得：不就自己写几个 package definition 嘛，Scheme 嘛，看着还挺好玩的。
>
> 然后你发现你要用的十个软件里有六个不在官方 channel。再然后你发现每个软件的构建依赖也是个问题。再再然后你发现你在手动维护一个迷你 nixpkgs。
>
> 这不是不可逾越的障碍，但这**不值得我的时间**。

### AppImage：NixOS 优雅，Guix 原始

我的工作流里 AppImage 使用频率不低。NixOS 对此的处理堪称模范：

```nix
# NixOS: 开箱即用
programs.appimage = {
  enable = true;
  binfmt = true;
};
```

或者 `appimage-run`，直接 `appimage-run ./some-app.AppImage` 就行。`nix-ld` 让不支持 NixOS 的预编译二进制也能找到动态链接器。

Guix 呢？目前没有等价的 `guix-ld`，没有 `appimage-run`。你想跑一个 AppImage，得这样：

```bash
guix shell -F -C <凑齐所有依赖> -- ./some-app.AppImage
```

`-F` 是模拟 FHS 文件系统布局，`-C` 是创建隔离容器。每次跑 AppImage 都等于手动构建一个迷你容器环境。另一种方案是开一个 distrobox 容器，在那里面跑——但到了这个地步，我为什么不用 distrobox 在 Fedora Atomic 上跑呢？也能解决依赖地狱问题但我跑路就是为了不折腾这个啊。

### 虚拟机试水：Plasma 配 GDM？

理论调研不爽，我还是在虚拟机里装了一遍 Guix System。选了 Plasma KDE 桌面，安装流程本身没问题——Guix 的安装脚本挺顺的。

重启，进登录界面。

**GDM。**

我愣了五秒。我选的明明是 Plasma KDE，为什么显示管理器是 GNOME 的 GDM？不是 SDDM？

> [!note] 也不是不能用
> GDM + Plasma 在技术上能工作，登录后 Plasma 会话正常启动。但这就像是买了一套精装修的房子，结果大门是隔壁小区的保安亭——功能上你进得去，但怎么看怎么别扭。

我知道 Plasma 有个新的 `plasma-login-manager`，但我之前看到文章说它强依赖 systemd，Guix 用的是 Shepherd，所以估计没戏。但 SDDM 呢？SDDM 不依赖 systemd 啊，KDE 自家的显示管理器，为什么默认不配这个？

当然，我知道我可以自己写 `config.scm` 把 GDM 换成 SDDM。Guix 的声明式配置本来就是为了让你改这些的。问题是：

> [!warning] 最近真的没时间
> 折腾 Guix 的 system config 不是改一行 `display-manager = sddm` 就完事的。你要理解 Guix 的 service 体系、shepherd 的配置方式、可能还要处理 sddm 的 theme 路径在 Guix store 里的位置。这些东西值得学，但我最近没有一整块时间来跟它耗。

第一印象很重要。如果安装完桌面环境的第一眼就让我皱眉，那后面的坑只会更多。事实证明我的判断是对的。

### 生态落差是真实存在的

社区规模和包数量这种东西，平时看数字觉得无所谓，实际用起来才知道痛：

- nixpkgs：超过 10 万个包
- Guix 官方 channel： 3 万多个

缺的不是什么冷门工具，是一些你**默认它肯定有**的东西。每缺一个，你就多一份手工劳动。

## 回到 Arch Linux

所以我做出了一个清醒的决定：**回到 Arch Linux。**

诚然，Arch 有自己的问题：

- systemd 大而全（虽然我其实不太在意 init 系统的圣战）
- KISS 原则导致打包粒度很粗
- 滚动更新的风险是真实存在的（虽然我用 btrfs + snapper，心里有底）
- 没有声明式系统配置（但谁说我想要声明式系统配置了？）

但 Arch 给了我一样 Guix 和 Fedora Atomic 都给不了的东西：**我不需要和系统的设计哲学搏斗。**

我想装什么，`pacman -S` 或 `paru -S`。想跑 AppImage，直接 `./app`。想运行预编译二进制，它找得到 `/lib64/ld-linux-x86-64.so.2`。这些都是理所当然的事情，在 FHS 的世界里它们确实是理所当然的。是 Guix 和 NixOS 让我重新理解了"理所当然"这四个字的分量。

> [!note] 而且我还能在 Arch 上装 Guix/Nix
> Guix 和 Nix 作为包管理器是可以装在**任何 Linux 发行版**上的。我完全可以在 Arch 上 `pacman -S guix`，然后用 Guix 管理开发环境和特定工具链，同时用 Arch 的生态作为主体。
>
> 反过来呢？在 Fedora Atomic 上装 Guix 或 Nix？不好说，那个不可变根文件系统会跟你闹别扭。所以我做的其实不是"选择 Arch 而不是 Guix"，而是"选择 Arch 作为基底，让 Guix/Nix 作为可选增强"。

## 兜了一个大圈，回到了原点

总结一下我今年的发行版精神分裂史：

1. 用着 obedur-os（Fedora Atomic），觉得不可变 + 容器化构建是未来 ✨
2. 云端构建被依赖冲突炸了，开始怀疑"半吊子不可变"的意义 🤔
3. 写文章反思，决定投奔 Guix 这个"真正的不可变" 🚀
4. 花一天了解 Guix，被 FSDG + 生态差距劝退 💀
5. 回到 Arch Linux，心态平和，世界美好 🕊️

> [!quote] 最后
> 每次回到 Arch 我都有一种"回家了"的感觉。
> 不是因为它最好，是因为它最不跟你讲道理。
> 你要装什么？AUR大概率有。你要删什么？paru -Rns。

> 下个月会不会又跑路？不好说。但今天是 Arch 的形状。

