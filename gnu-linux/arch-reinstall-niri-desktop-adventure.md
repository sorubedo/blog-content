---
title: 重回 Arch Linux
date: 2026-05-28
tags:
  - arch
  - niri
  - wayland
  - wm
  - virt-manager
description: 装个 Arch 不难，难的是选一个能用的桌面环境。在 river 和 niri 之间折腾了一圈。Reinstalling Arch is easy. Picking a working DE is the real boss fight.
---

前两天重装了 Arch Linux。其实是前两天就装好了系统本体，但接下来的时间全花在了桌面上 —— 我试了**三个桌面/窗管环境**才最终定下来。这篇文章算是这次折腾的记录。
![[niri.png]]
## river 0.4.5：曾经的王，现在变成了框架

最开始我是想用 [river](https://github.com/riverwm/river) 0.4.5 的。我知道它 0.4.x 大重构过，已经变成了一个 **"窗管框架"** 而非开箱即用的窗管 —— 按键绑定不管了，窗口布局也不管了，riverctl也没了，这些全部交给实现了 `river-window-management-v1` 协议的第三方工具来做。
### canoe：随便装了个，堆叠布局还行

先随便找了一个实现了这个协议的 WM 叫 [canoe](hhttps://github.com/roblillack/canoe)，是个堆叠布局的窗管，先整 waybar 配置，把状态栏搞起来再说。

![[Screenshot_2026-05-28_15-33-29.png]]

### waybar

waybar 仓库的示例配置 —— 没有完美适配的。这个倒也正常，毕竟协议层全变了，之前 0.3.x 时代的配置全废了。

不过在翻 waybar 配置的时候看到个有意思的窗管：[rhine](https://codeberg.org/sivecano/rhine)，它兼容了 **hyprland IPC call**。这意味着 Hyprland 社区的 waybar 配置大概率能直接用。

但说实话，我已经懒得折腾了。我需要赶紧弄出个**能干活的工作环境**，不是来当发行版鉴赏家的，我早重装过Arch n次了。

> [!note] 碎碎念
> river 的架构方向我是认可的 —— 把协议层和实现层分开，让生态自己生长。但目前社区还不够成熟，处于"理念很先进，拿来用还早"的阶段。下次有空再回来玩。

---

## 回到 niri：Dotfiles 的含金量

于是我又装回了 [niri](https://github.com/YaLTeR/niri)。
![[satty-20260528-14 54 01.png]]
![[satty-20260528-14 56 01.png]]
### 丝滑得不像话

安装流程简单到离谱：

1. `pacman -S niri noctalia-shell` 和相关依赖
2. 把之前的 dotfiles 搬过来
3. 重启

**简直完美。** 跟我重装前一模一样，什么都没有丢，什么也不用重新配。

> 这就是把配置当代码管理的意义 —— 只要 dotfiles 还在，换一台机器你依然是你。

[noctalia-shell](https://github.com/tahmidazam/noctalia-shell) 作为 niri 的 shell 层做得相当不错，状态栏、通知、启动器都有了，开箱即用程度在 Wayland 窗管里排得上前列。

![[satty-20260528-14 56 57.png]]

---

## Arch 打包的粗糙之处：xdg-desktop-portal-gnome 的依赖问题

Arch 的打包质量总体来说不错，但总有些地方让人挠头。这次遇到的是 `xdg-desktop-portal-gnome` 依赖 `nautilus`。

### 问题链

```
xdg-desktop-portal-gnome → nautilus → libadwaita + 一堆 GNOME 套件
```

我这次换成了 [pcmanfm-qt](https://github.com/lxqt/pcmanfm-qt)，比 nautilus 好用不少，轻量、Qt 原生、没有 libadwaita 依赖。我是真不想装 GNOME 的东西（nautilus、libadwaita 这些）。

> [!warning] 但有些东西逃不掉
> `xdg-desktop-portal-gnome` 本身是可以留的 —— niri 兼容且推荐这个门户。GNOME 的密钥环也得用，这个是真的绕不开。

其实可以用 `xdg-desktop-portal-wlr` 替代 portal-gnome。niri 也实现了 `wlr-screencopy` 协议，portal-wlr 能用。

**但是**：portal-wlr 模式下，OBS 没法单独捕获窗口。Screencast 功能受影响。

![[satty-20260528-15 00 23.png]]
ps: 我这里用了fuzzel给portal-wlr做选择器
### 写了一个空包

解决方案：给 nautilus 写了个空包。

```
pkgname=nautilus-dummy
epoch=1
pkgver=1
pkgrel=1
pkgdesc="Dummy empty package to satisfy nautilus dependency (provides/conflicts)"
arch=('any')
url="https://apps.gnome.org/Nautilus/"
license=('GPL-3.0-or-later')
depends=()
makedepends=()
provides=('nautilus')
conflicts=('nautilus')
replaces=()
source=()
sha256sums=()

package() {
    # 空包：只放一个标识文件，不安装 nautilus 的任何实际文件
    install -dm755 "${pkgdir}/usr/share/doc/${pkgname}"
    cat > "${pkgdir}/usr/share/doc/${pkgname}/README" << 'EOF'
nautilus-dummy
==============

This is a dummy/empty package that replaces the official nautilus.
It provides "nautilus" to satisfy dependency requirements without
installing GNOME Files (nautilus).

No actual files from nautilus are installed. If you need the real
file manager, remove this package and install the official nautilus:

    pacman -S nautilus
EOF
}
```

Arch 写包是真的方便，十来行就能搞定一个依赖占位包。

### 文件选择器别忘了配

portal-gnome 的文件选择对话框确实是依赖 nautilus 的。不装 nautilus 的话，需要在 portal 配置里加上：

```ini
[preferred]
default=gnome;gtk;
org.freedesktop.impl.portal.Access=gtk;
org.freedesktop.impl.portal.Notification=gtk;
org.freedesktop.impl.portal.Secret=gnome-keyring;
org.freedesktop.impl.portal.FileChooser=gtk;
```

这样文件选择器就走 GTK 的通用实现了，不会调 nautilus。

![[satty-20260528-15 03 58.png]]

---

## 屏幕共享：

niri + noctalia-shell 日常使用相当顺手，但**屏幕共享这块确实有问题**。

### 腾讯会议的黑屏

[wemeet](https://meeting.tencent.com/)（腾讯会议 Linux 版）官方已经实现了 Wayland 的屏幕捕获。但在 niri 下共享出去的画面是**纯黑屏**。

### OBS 虚拟摄像头也是黑屏

我用 OBS 虚拟摄像头喂给 wemeet —— 也是黑屏。Grok 分析说是 `v4l2loopback` 版本太高了导致兼容问题。社区确实有低版本的 fork，但我懒得折腾了。

> [!note] 不是 niri 的锅
> Wayland 的屏幕捕获协议本身就还处于"谁都想当标准"的阶段。`wlr-screencopy`、`pipewire`、`portal` 各种协议搅在一起，应用适配情况参差不齐。腾讯会议能在 Wayland 下跑起来已经算不错的了。

### 退路：plasma-desktop 最小安装

直接装了个 `plasma-desktop` 最小包当备胎：

```bash
pacman -S plasma-desktop
```

有问题就切到 Plasma，至少在 KDE 下屏幕共享是经过充分验证的。双桌面不是妥协，是工程上的务实。

![[屏幕截图_20260528_152106.png]]

---

## 虚拟机：回到 virt-manager

虚拟机方案这次也换回来了。
本来想偷懒用 [quickemu](https://github.com/quickemu-project/quickemu)，一个命令就能起虚拟机确实爽。但这学期有实验课要用 **Windows XP** 这种上古系统 —— quickemu 不支持。
### libvirt 用户会话模式

换回 [virt-manager](https://virt-manager.org/) + libvirt。但这次学聪明了，用了 **libvirt 的用户会话模式**。

不需要 `libvirtd` 这个常驻后台守护服务。我启用了 `libvirtd.socket`，有 root 级 QEMU/KVM 会话需求时由 systemd 按需唤醒，不用的时候不占后台资源：

```bash
systemctl enable --now libvirtd.socket
```

virt-manager 里把自动连接关了就好。systemd 的 socket activation 真是好东西 —— **按需启动，零空闲开销**。
![[satty-20260528-15 06 39.png]]

> [!note] 拥抱红帽恩情
> systemd、libvirt、virt-manager 、flatpak、pipewire全是 Red Hat 主导的项目。去年前我还在骂 systemd 臃肿，今年我已经离不开 socket activation、timer、user service 了。人终究会活成自己讨厌的样子。

---

## 总结
折腾了一大圈，最后发现**最适合自己的还是已经配好的那套**。river 的新架构很有潜力，但社区还需时间。niri + noctalia-shell/dms目前就是 Wayland 窗管里对"开箱即用 + 可定制"平衡得最好的选择之一 —— 尤其当你已经有了成熟的 dotfiles。

下次大概率不会这么折腾了。大概。

> 骗你的，下周可能又换一个。
