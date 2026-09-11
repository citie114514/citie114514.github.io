---
layout: post
title: "终末地手机端解锁极高画质：iUnlocker 模块实操记录"
subtitle: "Root + Zygisk-Next，让非旗舰机也能开 1036P"
date: 2026-09-10 21:20
author: "磁贴"
header-img: "img/post-bg-android.jpg"
catalog: true
tags:
    - 明日方舟终末地
    - Android
    - Root
    - 模块
    - 教程
---

相信很多人的手机配置都比不上 8 Elite 和 iPhone 17 Pro Max。《明日方舟：终末地》的"极高画质"（1036P）默认只对旗舰芯片开放，中低端机型在设置里根本看不到这个选项。这期视频就是教你怎么用 iUnlocker 模块把它强开出来，演示机是一台 Redmi Note 11T Pro+。

视频原版在 B 站：[BV1xdctzkEsc](https://www.bilibili.com/video/BV1xdctzkEsc/)，这篇博文把流程整理成文字版方便照着做。

![视频开头 - 问题引出](/assets/img/endfield/01-intro.jpg)

## 前置条件

- **iUnlocker 模块**：QQ 群（960262360）群文件里有，也可以从视频简介的网盘下载（iUnlocker 模块和 Zygisk-Next 模块打包在一起）
- **Root 权限**：KernelSU 或 Magisk（面具）都可以，视频里用的是 KernelSU
- **Zygisk 环境**：KernelSU 用户需要单独刷入 Zygisk-Next 模块；Magisk 用户不用装，在面具设置里把 Zygisk 打开即可

两个模块我在演示机上都已提前装好，视频里不重复刷机过程。装好模块后**重启手机**，桌面上会多出一个 iUnlocker 的应用图标。

![手机桌面与模块](/assets/img/endfield/02-ksu-modules.jpg)

### KernelSU 模块管理

在 KernelSU 的模块页面可以看到已安装的模块列表，包括 Pandora Kernel 附加模块、GyroSensor、音效模块等。确保 Zygisk-Next 模块已启用。

![KernelSU 模块列表](/assets/img/endfield/03-modules-list.jpg)

## 配置 iUnlocker

打开这个应用，操作顺序是：

1. 点下方的**加号**，点 **SEARCH** 搜索"终末地"，把游戏添加进来（视频里因为已经添加过，选项界面会略有不同）
2. 点上方出现的**终末地图标**——就是那个眼睛和爱心形状的图标
3. 剩下的选项**都不用改**，把右边的开关按钮打开就行
4. **GPU 型号**填 Adreno **740~830** 都可以，视频里用的是 **830**；天玑平台的机器也是一样填
5. 按返回键，点 **Save**，再点 **OK**

![iUnlocker 配置界面](/assets/img/endfield/04-iunlocker-config.jpg)

### GPU 型号确认

在 Graphics 信息页面可以确认当前 GPU 型号。视频中显示 OpenGL 和 Vulkan 均为 Adreno (TM) 830，填入 iUnlocker 的 GPU 型号栏即可。

![GPU 信息确认](/assets/img/endfield/05-iunlocker-gpu.jpg)

到这里配置就完成了。

## 验证效果

直接打开终末地进游戏看。判断成功的标准有两个，同时出现才算成：

- 画质选项里出现了**"极高画质"**
- 选项前面有**"编译着色器"**的提示

![启动终末地](/assets/img/endfield/06-launch-game.jpg)

视频后半段是实机演示，画面确实跑在极高画质上。如果设置里还是只有原来的几档，回头检查模块是否生效、开关是否打开、Save 是否点了。

![终末地启动画面](/assets/img/endfield/07-endfield-loading.jpg)

![终末地主菜单](/assets/img/endfield/08-endfield-menu.jpg)

## 补充说明

这个方法的本质是模块绕过了游戏的机型白名单检测，画质解锁后对 SoC 的压力是实打实的，中端芯跑 1036P 帧率不会好看，发热也会上来，图画质还是图流畅自己权衡。模块下载链接和 QQ 群号都在视频简介里，这里就不重复贴了，以原视频为准。
