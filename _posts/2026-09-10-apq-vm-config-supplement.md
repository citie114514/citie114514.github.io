---
layout: post
title: "APQ虚拟机配置补充教程：磁盘与CPU进阶设置"
date: 2026-09-10 13:00:00 +0800
header-img: "img/post-bg-apq-config.jpg"
categories: APQ 教程
description: "补充APQ虚拟机的磁盘挂载、CPU多线程等高级配置方法"
tags: [APQ, 虚拟机, Android, QEMU, 教程]
---

这是上一期 APQ 教程的补充，讲磁盘配置和 CPU 进阶设置这些细节。APQ 是一个在 Android 上运行 QEMU 虚拟机的工具，可以拿来跑 Windows 之类的系统。

## 一、配置文件概览

打开 APQ 的"添加配置文件"页面，能看到完整的虚拟机配置项：

![配置文件概览](/assets/img/apq-supplement/config-overview.jpg)

- 磁盘A：主硬盘镜像（如 win7mini.qcow2）
- 运行内存：虚拟机内存大小（如 1024M）
- 网卡：网络适配器配置（如 `-net user -net nic,model=e1000`）
- 显卡：显示适配器（如 `-vga vmware`）
- 显示方式：VNC 显示（如 `-vnc :0`）
- CPU架构：系统架构（如 `#sys x86_64`）

## 二、磁盘配置详解

### 1. 硬盘A（主磁盘）

硬盘A一般放操作系统镜像，比如 qcow2 格式的 Windows 镜像。

![配置详情](/assets/img/apq-supplement/config-detail.jpg)

### 2. 添加硬盘B

点磁盘区域的 **+** 号可以加新的磁盘设备：

![添加硬盘B](/assets/img/apq-supplement/disk-edit.jpg)

### 3. 硬盘属性设置

新加的硬盘B可以设置这些属性：

- 路径：选择磁盘镜像或目录的位置
- 目录：勾上表示挂载的是目录而不是磁盘镜像
- 只读：建议勾上，免得碰上写权限问题
- 启动优先级：设 `-boot c` 表示从这块盘启动

> **注意**：磁盘设为 fat 模式时默认就是只读挂载。读写模式一般都会出错，容易出权限问题，推荐直接用只读，把手机目录挂进去当磁盘。

### 4. 磁盘路径配置

硬盘B的路径指到手机上哪个目录都行，比如 `/storage/emulated/0/`：

![磁盘路径](/assets/img/apq-supplement/disk-path.jpg)

### 5. IDE硬盘说明

> IDE硬盘属于 IDE 第 0/1 端口，可以添加磁盘镜像/目录，**目录不支持作为启动设备**。

`/dev/` 下的磁盘节点设备也能拿来当磁盘用，不过需要 ROOT 权限，而且很可能把磁盘上原来的数据搞丢。

## 三、CPU进阶设置

### 1. CPU类型

虚拟 CPU 的型号也可以在这里选：

![CPU设置](/assets/img/apq-supplement/cpu-settings.jpg)

### 2. 核心数配置

核心数用 `-smp cpus=2` 设置，多线程加速加一行 `--accel tcg,thread=multi` 就能开。

### 3. 完整CPU配置示例

```
-cpu core2duo
-smp cpus=2
--accel tcg,thread=multi
```

## 四、软盘配置

软盘 0/1 设备只认软盘镜像，路径示例：

`/storage/emulated/0/Android/data/com.tencent.mobileqq/Tencent/QQfile_recv/`

![软盘配置](/assets/img/apq-supplement/disk-usb.jpg)

## 五、配置保存

全部设完点右上角的保存按钮，配置文件就存好了，回主界面选中它就能启动虚拟机。

## 小结

进阶配置主要就这么几块：磁盘支持 qcow2 镜像和目录挂载，只读比读写模式省心；可以同时挂多块盘；CPU 能选型号、设核心数、开多线程加速；软盘也能挂。

> 视频教程来源：[被win11吃掉的磁贴](https://space.bilibili.com/) B站频道
