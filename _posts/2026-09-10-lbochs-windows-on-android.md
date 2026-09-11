---
layout: post
title: "用手机 lbochs 运行 Windows XP：安装与配置全过程"
date: 2026-09-10 19:00
author: "磁贴"
header-img: "img/post-bg-android.jpg"
catalog: true
tags:
    - lbochs
    - Bochs
    - Windows XP
    - Android
    - 虚拟机
---

lbochs 是把 x86 模拟器 Bochs 移植到 Android 上的应用，不靠任何虚拟化硬件加速，纯软件指令翻译，所以慢是慢了点，但好处是什么手机都能跑。这期视频演示了从安装到在手机上点亮 Windows XP 经典蓝天草地桌面的完整过程。

视频原版在 B 站：[BV1ZP4y1y7YX](https://www.bilibili.com/video/BV1ZP4y1y7YX/)，镜像和 lbochs 安装包都在 QQ 群（960262360）里。下面是文字版流程。

![QQ 群号与下载说明](/assets/img/lbochs/01-download-qq.jpg)

## 安装

群文件里下载 **lBochs PC Emulator**（视频里装的是 2.1 版，安装包 8.45MB），正常安装。首次打开申请的权限**全部允许**。进入应用后如果需要返回上一级，用手机的全面屏手势触发返回即可，用虚拟按键或物理按键按返回也一样。

![权限全给](/assets/img/lbochs/02-permissions.jpg)

## 配置虚拟机

点进设置，按视频里的顺序过一遍：

![启动顺序设置](/assets/img/lbochs/03-boot-order.jpg)

- **启动顺序**：floppy（软盘）、cdrom（光盘）、disk（硬盘）三选一，你用哪种镜像就勾哪种
- **GUI 线程**：打开
- **CPU IPS**：建议填 **30**（界面默认 15，单位 MIPS）。这个值决定模拟速度的上限
- **RAM**：内存可以自由设置，但**别调太大**，视频里演示填的是 1024MB。给多了手机自己先撑不住
- **PC hardware → CPU**：处理器位数和型号按需选，列表里有酷睿 T2400（1.83GHz）、阿童木 N270 等，底下有 Limit CPUID 选项
- **BIOS**：默认即可，不用动
- **显示**：Vga 更新频率、实时频率、PCI 连接、Voodoo 3D Accel 这些保持默认
- **Sound**：不用管
- **端口**：串行/并行/USB 端口维持默认，不映射 APM shutdown

![PC Hardware 设置](/assets/img/lbochs/04-pc-hardware.jpg)

### BIOS 选项

BIOS 可以选择 BOCHS LEGACY、BOCHS LATEST、SEABIOS 1.7.0 / 1.8.0 / 1.13.0 或自定义。视频里用的是 BOCHS LATEST。

![BIOS 设置](/assets/img/lbochs/05-bios-settings.jpg)

### 声卡设置

Sound 部分可以配置声卡、DMA 定时器、动态定时器和启用扬声器。视频里保持默认即可。

![声卡设置](/assets/img/lbochs/06-sound-settings.jpg)

每改完一项会提示"配置参数已更改，请重新启动应用程序"，全部设完之后**退出应用重新进入**让配置生效。

## 挂载镜像并启动

重新进入后设置**镜像路径**，找到你从 QQ 群下载好的 Windows XP 镜像文件（如果在 QQ 里下载的，路径一般在 `Android/data/com.tencent.mobileqq/cache` 这类 QQ 下载目录里）。

![选择镜像路径](/assets/img/lbochs/07-image-path.jpg)

确认退出、再进应用点运行，屏幕上会跑过 Bochs 2.6.11 的 BIOS 自检画面，然后就开始进系统了。

![Windows XP 启动中](/assets/img/lbochs/08-xp-booting.jpg)

视频结尾的镜头里，Windows XP 的 Bliss 桌面完整显示在手机屏幕上。

![Windows XP 经典桌面](/assets/img/lbochs/09-xp-desktop.jpg)

## 体验预期

纯软件模拟的性能不要有太多幻想：XP 这个级别的系统能动、能点、能打开程序，但每一步都有明显的迟滞。它适合折腾和怀旧，不适合当正经工具。想跑得更像样，可以看看另一篇用 APQ（QEMU）方案的记录，思路不同但同样在手机上跑 Windows。
