---
layout: post
title: "手机上用 APQ 运行 Windows：QEMU 虚拟机 + VNC 连接"
date: 2026-09-10 20:00
author: "磁贴"
header-img: "img/post-bg-android.jpg"
catalog: true
tags:
    - APQ
    - QEMU
    - Windows
    - Android
    - 虚拟机
---

APQ 是一个基于 QEMU 4.2.0 的 Android 虚拟机应用，思路是在手机里跑一个 x86 虚拟机，再用 VNC 客户端连上它的显示输出。这期视频全程没有解说（BGM 是《勾指起誓》），纯靠画面演示，这里把流程整理成文字版。

视频原版在 B 站：[BV1YV4y177Ri](https://www.bilibili.com/video/BV1YV4y177Ri/)。**先说一句重要的：视频简介里明确写了"该方法较为老旧，新版本安卓可能无法正常使用"**，新机器翻车概不负责，想试的有个心理预期。

## 下载两个应用

视频开头的便签里给了两个蓝奏云地址：

- VNC Viewer 汉化版（3.6.1.42089，12.2MB）
- APQ 测试版（1.6，71.9MB）

![下载地址便签](/assets/img/apq/01-download-links.jpg)

视频简介里还有一个打包的网盘链接（citie.lanzoul.com，密码 8ck5）和 QQ 群号 960262360，镜像文件在群文件里。两个 APK 依次下载安装，安装过程中系统推荐的"去应用商店下载"不用理。

![VNC Viewer 下载](/assets/img/apq/02-vnc-download.jpg)

## 安装与权限

APQ 首次打开会申请权限：**全部同意**。包括联网权限（虚拟机联网用）、储存权限（读写镜像和配置文件），以及悬浮窗权限——悬浮窗透明像素是用来保持后台运行的，不给的话切出去虚拟机可能就没了。

![APQ 安装与权限](/assets/img/apq/03-apq-install.jpg)

## 添加虚拟机配置

打开 APQ（界面标题栏显示 1.6 QEMU-4.2.0），主界面底部有"添加配置 / 镜像工厂 / ROOT 运行 / 关于"四个入口。点**添加配置**，按视频里的参数填：

![APQ 主界面](/assets/img/apq/04-apq-main.jpg)

- **磁盘 A**：点文件夹图标，进入 `/storage/emulated/0/APQ` 选镜像。视频演示用的是 `win7mini.qcow2`（Windows 7 精简镜像，在 QQ 群下载）
- **运行内存**：`-m size=1024M`。默认只有 128M 肯定不够，视频里手机剩余 1231MB / 总共 3826MB，给虚拟机分了 1024M。注意参数说明里也提示了运存给太大反而会导致闪退和卡顿，量力而行
- **网卡**：`-net user -net nic,model=rtl8139`
- **显卡**：`-vga vmware`
- **显示方式**：`-vnc :0`——这就是后面要用 VNC 连的原因，虚拟机不直接出画面，而是通过 VNC 协议输出
- **CPU 架构**：`#sys:x86_64`
- **CPU**：`-cpu coreduo`

![配置文件参数](/assets/img/apq/05-config-file.jpg)

### CPU 设置

CPU 部分选择 x86_64 架构和 coreduo 处理器型号。界面下方有提示：该选项在跨架构时并无多大用处，在这种情况下一般只能用到单线程。

![CPU 设置](/assets/img/apq/06-cpu-settings.jpg)

填完点保存，提示"配置文件保存成功"后返回主界面。

## 用 VNC Viewer 连接

APQ 启动虚拟机后，切到 VNC Viewer，点右下角**加号**新建连接：

- **地址**：`127.0.0.1`（虚拟机就在本机，直接连回环地址）
- **名字**：随意，视频里填的"我的电脑"，也可以不取

![APQ 运行中](/assets/img/apq/07-apq-running.jpg)

创建后点连接，会弹出"**未加密连接**"的红色警告——这是正常的，本机 VNC 没有加密，点**确定**继续，Windows 桌面就出来了。可以把"每次警告我"关掉省得每次都弹。

![VNC 连接 127.0.0.1](/assets/img/apq/08-vnc-connect.jpg)

## 和 lbochs 方案的对比

同样是手机跑 Windows，APQ 背靠 QEMU，配置项是标准的 QEMU 参数写法，镜像格式用 qcow2，整体比 Bochs 那套更接近现代虚拟机的玩法；lbochs 胜在配置全是图形界面点选，门槛更低。两条路线我都写了记录，按需取用。再次提醒：APQ 这个方法年代较早，新安卓版本大概率跑不起来，纯属考古向折腾。
