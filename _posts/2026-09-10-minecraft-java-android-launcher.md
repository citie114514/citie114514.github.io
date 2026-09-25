---
layout: post
title: "手机玩Java版Minecraft：PojavLauncher与HMCL-PE启动器教程"
date: 2026-09-10 12:00:00 +0800
header-img: "img/post-bg-mc-java.jpg"
categories: Minecraft 教程
description: "教你如何在Android手机上通过PojavLauncher和HMCL-PE启动器运行Java版Minecraft"
tags: [Minecraft, PojavLauncher, HMCL-PE, Android, Java版]
---

想在 Android 手机上玩 Java 版 Minecraft 的玩家不少，但很多人不知道从哪下手。这篇把两期视频的内容合在一起，分别讲 PojavLauncher 和 HMCL-PE 两个启动器怎么用。

## 一、PojavLauncher 启动器

PojavLauncher 是个开源的 Android 端 Java 版 Minecraft 启动器，多版本和模组加载都支持。

### 1. 下载与安装

先要把这几个资源下齐：

| 资源 | 下载地址 |
|------|---------|
| PojavLauncher | [GitHub Releases](https://github.com/PojavLauncherTeam/PojavLauncher/releases) |
| Forge | [files.minecraftforge.net](https://files.minecraftforge.net/) |
| Fabric | [fabricmc.net](https://fabricmc.net/use/) |
| Optifine | [optifine.net](https://www.optifine.net/home) |
| Open4es 光影 | [GitHub](https://github.com/Open4Es/Open4Es-Shader) |

![下载页面](/assets/img/mc-launcher/pojav-download.jpg)

下载完装 APK 就行。

### 2. 登录账号

打开 PojavLauncher，登录方式有三种：

- 微软账号：推荐，能过正版验证
- 离线模式：输个用户名就行（只能玩单人）
- 选择账号：管理已经登录的账号

![登录界面](/assets/img/mc-launcher/pojav-login.jpg)

### 3. 基本设置

设置页面里有几个值得看的参数：

- Java 运行时环境：需要重装的话点"卸载 Java 运行时环境"
- 长按触发时长：改长按的触发时长（默认 500ms）
- 分辨率缩放倍率：在开销和画面之间自己找平衡

选好版本点**启动游戏**就行。

![设置界面](/assets/img/mc-launcher/pojav-settings.jpg)

### 4. 进入游戏

启动后进的就是 Minecraft Java 版主界面，Singleplayer、Multiplayer、Minecraft Realms 都在老地方。

![游戏主菜单](/assets/img/mc-launcher/pojav-mc-menu.jpg)

### 5. 操控方式

PojavLauncher 用虚拟按键操控：

- 左侧：方向键（主/副方向）、视角控制
- 右侧：GUI、背包、键盘、TAB
- 顶部：鼠标开关

![游戏画面1](/assets/img/mc-launcher/pojav-gameplay1.jpg)
![游戏画面2](/assets/img/mc-launcher/pojav-gameplay2.jpg)

### 6. 安装 Forge

要用 Forge 模组的话，先在手机浏览器里下载 Forge 安装器，再回 PojavLauncher 里装对应版本。

![Forge下载](/assets/img/mc-launcher/pojav-forge-download.jpg)

### 7. 光影设置

装好 Optifine 就能用光影包了，推荐 Open4es，画面设置里各项参数都能调。

![光影设置](/assets/img/mc-launcher/pojav-shader.jpg)

## 二、HMCL-PE 启动器

HMCL-PE（Hello Minecraft! Launcher Pocket Edition）是 HMCL 的 Android 移动端版本，功能更全一些。

### 1. 安装

下载 HMCL-PE 的 APK 直接装。

![安装APK](/assets/img/mc-launcher/hmclpe-install.jpg)

### 2. 添加离线模式账户

打开 HMCL-PE，点"添加离线模式账户"，用户名随便填，离线账号就建好了。

![添加账户](/assets/img/mc-launcher/hmclpe-account.jpg)

### 3. 安装 OptiFine

HMCL-PE 自带版本管理，OptiFine 这类东西直接在启动器里装，不用额外折腾。

![安装OptiFine](/assets/img/mc-launcher/hmclpe-optifine.jpg)

### 4. 启动器主界面

主界面分成这几块：

- 左侧导航栏：新闻、开发控制台、崩溃日志、设置
- 版本选择和启动按钮
- 账户管理

![主界面](/assets/img/mc-launcher/hmclpe-main.jpg)

### 5. 自定义控制

控制布局能自定义的地方不少：悬浮窗、侧边栏、侧滑手势、不锁定悬浮窗、联机模块菜单、高级输入都支持。

![控制设置](/assets/img/mc-launcher/hmclpe-controls.jpg)

### 6. 进入游戏

启动后进的是 Minecraft Java Edition 主界面，和电脑版一模一样。

![游戏主菜单](/assets/img/mc-launcher/hmclpe-mc-menu.jpg)

### 7. 游戏画面

操控同样是虚拟按键加鼠标模拟，玩起来没什么卡顿。

![游戏画面](/assets/img/mc-launcher/hmclpe-gameplay.jpg)

### 8. 光影与画面设置

一样支持 OptiFine 光影，画面设置里各项参数都能调。

![光影设置](/assets/img/mc-launcher/hmclpe-shader.jpg)

## 三、两款启动器对比

| 特性 | PojavLauncher | HMCL-PE |
|------|--------------|---------|
| 开源 | √ | √ |
| Forge/Fabric 支持 | √ | √ |
| Optifine 安装 | 需手动 | 内置安装 |
| 控制自定义 | 基础 | 丰富 |
| 光影支持 | √ | √ |
| 多账号管理 | √ | √ |

## 四、常见问题

**Q: 手机配置要求是什么？**
A: 建议 4GB 以上内存、64 位处理器。视频里测试用的是腾讯黑鲨游戏手机3。

**Q: 可以玩模组吗？**
A: 可以，PojavLauncher 和 HMCL-PE 都支持 Forge 和 Fabric 模组加载。

**Q: 光影会卡吗？**
A: 看手机性能和光影包复杂度，Open4es 算比较轻量的选择。

> 视频教程来源：[被win11吃掉的磁贴](https://space.bilibili.com/) B站频道
