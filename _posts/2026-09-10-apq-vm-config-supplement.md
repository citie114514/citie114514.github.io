---
layout: post
title: "APQ虚拟机配置补充教程：磁盘与CPU进阶设置"
date: 2026-09-10 13:00:00 +0800
categories: APQ 教程
description: "补充APQ虚拟机的磁盘挂载、CPU多线程等高级配置方法"
tags: [APQ, 虚拟机, Android, QEMU, 教程]
---

## 前言

这是对上一期APQ教程的补充，主要讲解磁盘配置和CPU进阶设置等细节。APQ 是一个在 Android 上运行 QEMU 虚拟机的工具，可以用来运行 Windows 等系统。

---

## 一、配置文件概览

打开APQ的"添加配置文件"页面，可以看到完整的虚拟机配置项：

![配置文件概览](/assets/img/apq-supplement/config-overview.jpg)

配置项包括：
- **磁盘A**：主硬盘镜像（如 win7mini.qcow2）
- **运行内存**：虚拟机内存大小（如 1024M）
- **网卡**：网络适配器配置（如 `-net user -net nic,model=e1000`）
- **显卡**：显示适配器（如 `-vga vmware`）
- **显示方式**：VNC 显示（如 `-vnc :0`）
- **CPU架构**：系统架构（如 `#sys x86_64`）

---

## 二、磁盘配置详解

### 1. 硬盘A（主磁盘）

硬盘A通常用于存放操作系统镜像，如 qcow2 格式的 Windows 镜像。

![配置详情](/assets/img/apq-supplement/config-detail.jpg)

### 2. 添加硬盘B

点击磁盘区域的 **+** 号可以添加新的磁盘设备：

![添加硬盘B](/assets/img/apq-supplement/disk-edit.jpg)

### 3. 硬盘属性设置

新添加的硬盘B可以设置以下属性：

- **路径**：选择磁盘镜像或目录的路径
- **目录**：勾选后表示挂载的是目录而非磁盘镜像
- **只读**：建议勾选，避免写权限问题
- **启动优先级**：设置 `-boot c` 表示从该磁盘启动

> ⚠️ **注意**：设定磁盘为 fat 模式时，默认挂载为只读。注意读写模式一般都会出错，可能会导致权限问题，推荐使用只读模式，挂载手机的目录到磁盘。

### 4. 磁盘路径配置

硬盘B的路径可以指向手机上的任意目录：

![磁盘路径](/assets/img/apq-supplement/disk-path.jpg)

例如可以挂载手机的 `/storage/emulated/0/` 目录。

### 5. IDE硬盘说明

> IDE硬盘属于 IDE第0/1端口，可以添加磁盘镜像/目录，**目录不支持作为启动设备**。

你也可以让 `/dev/` 下的磁盘节点设备作为磁盘，不过这样很可能会让你的磁盘原来的数据丢失，不过这样需要ROOT权限。

---

## 三、CPU进阶设置

### 1. CPU类型

APQ 支持设置虚拟 CPU 的类型：

![CPU设置](/assets/img/apq-supplement/cpu-settings.jpg)

### 2. 核心数配置

- **核心数**：通过 `-smp cpus=2` 设置 CPU 核心数
- **CPU多线程**：通过 `--accel tcg,thread=multi` 启用多线程加速

### 3. 完整CPU配置示例

```
-cpu core2duo
-smp cpus=2
--accel tcg,thread=multi
```

---

## 四、软盘配置

APQ 还支持挂载软盘设备：

- **软盘0/1设备**，仅支持软盘镜像
- 路径示例：`/storage/emulated/0/Android/data/com.tencent.mobileqq/Tencent/QQfile_recv/`

![软盘配置](/assets/img/apq-supplement/disk-usb.jpg)

---

## 五、配置保存

完成所有设置后，点击右上角的保存按钮即可保存配置文件。之后可以在主界面选择该配置文件启动虚拟机。

---

## 总结

APQ 的进阶配置主要涉及：
1. **磁盘挂载**：支持 qcow2 镜像、目录挂载、只读/读写模式
2. **多磁盘**：可以同时挂载多个磁盘设备
3. **CPU配置**：支持设置 CPU 类型、核心数、多线程加速
4. **软盘支持**：可以挂载软盘镜像

合理配置这些参数，可以让虚拟机运行更加流畅。

---

> 视频教程来源：[被win11吃掉的磁贴](https://space.bilibili.com/) B站频道
