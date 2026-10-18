---
layout: post
title: "照着做：天玑 8100 开 KVM 跑 Windows 11 ARM64"
date: 2026-10-18 19:30
author: "磁贴"
header-img: "img/post-bg-kvm-steps.jpg"
header-pos-mobile: "center center"
catalog: true
tags:
    - Android
    - KVM
    - MTK
    - 教程
    - Windows
    - QEMU
---

上一篇写的是这半个月怎么一路错过来的，这篇只讲动手顺序。

想听故事的去看[那篇](/2026/10/18/android-kvm-windows11-xaga/)。这里默认你已经决定要刷，只想把机器弄出来。

## 先花两分钟确认你是哪一批

补丁不是按机型匹配的，是按 `tee` 固件的完整 SHA-256 匹配。同一个型号，不同批次，哈希就不一样。我手上两台 Redmi Note 11T Pro+，一台 `f8f286f1`，一台 `a91f5ded`，差得挺远。

```bash
adb shell su -c 'dd if=/dev/block/by-name/tee_a bs=4096 2>/dev/null | sha256sum'
adb shell su -c 'dd if=/dev/block/by-name/tee_b bs=4096 2>/dev/null | sha256sum'
```

`tee_b` 那个槽从来没被动过，它是判断批次的锚点。如果 `tee_b` 和你查到的成品基座对不上，说明这台机器跟成品不是一个来源，往下走得自己构建。

## 需要的东西

解锁的 Bootloader、能用的 root（KernelSU 或 Magisk）、PC 上的 adb 和 Python 3.10 以上。

没有 root 就别往下看了。市面上那种"免 root 跑虚拟机"的方案走的是 TCG，纯软件模拟，慢十几倍到几十倍，装个系统要几小时。这个项目只做 KVM 路线。

另外建议用 USB 线，别用无线 adb 刷分区。技术上能刷，我心里不踏实。

## 第一件事不是刷，是备份

一键脚本会强制备份六个分区：`tee_a`、`tee_b`、`lk_a`、`lk_b`、`preloader_raw_a`、`seccfg`，每个都校验哈希，坏一个就停。

顺带说一句 `preloader`。原厂 preloader 的 EDL 需要小米售后账号授权，意思是 tee 刷坏、机器起不来的时候，你手上没有免授权的救援通道。工程 preloader 让写入不需要授权，但它不关开机校验，两件事别混。所以理想顺序是先确认能刷工程 preloader，再动 tee。

如果你不确定手上是不是工程版，看哈希：

```bash
adb shell su -c 'dd if=/dev/block/by-name/preloader_raw_a bs=4096 2>/dev/null | sha256sum'
```

`056ed47a...` 开头就是那份已经流出来的工程版，不用再刷。

## 跑一键脚本

```powershell
git clone https://github.com/MT6895-Mainline/mtk-mod-tee-nogz  D:\mtk-mod-tee-nogz
git clone https://github.com/kasnria001/pwnage24mtk           D:\pwnage24mtk
cd D:\mtk-mod-tee-nogz
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt

cd <本仓库>\scripts
.\kvm-oneclick.ps1 -Profile xagapro `
  -TeeFixRepo D:\mtk-mod-tee-nogz `
  -PwnageDir D:\pwnage24mtk
```

`-Profile` 必须是 `xagapro`，不是可选项。上游那个 `xaga` profile 对应的是另一批固件（`bd4b13a7`），拿它去配你的 `f8f286f1`，脚本第一步哈希校验就过不去，直接中止。这是设计如此，不是 bug。

不想真刷就加 `-DryRun`，会一路做到签名和校验，把待刷镜像留在磁盘上。

脚本跑起来先打印型号、序列号、系统版本。录视频的话记得把这行挡掉。

第一次运行会看到它在改上游仓库，加一个 `sign_all_flag` 的定义、往 profiles 里塞 `xagapro`。这是上游确实缺的东西，不改的话走到签名那步必然 NameError。脚本做成了幂等，改过就跳过。

中间会停下来要你在手机上点授权：USB 调试的"一律允许"，还有 KernelSU 给 `adb shell` 的 root。各点一次就行。

如果设备升过 Android 16，`lk_a` 大概率已经被 OTA 换掉了。补丁在真机上照样能用（这点我实测过），但构建工具的离线回归需要配对的那支 lk，脚本会去 `lk-archive` 里找。实在找不到，就别构建了，直接用 `flash-tee.ps1` 刷仓库里的成品。

## 刷完重启，先把手放下

这一步劝退的人最多。

刷完重启，屏幕会停在第二屏，转圈，一两分钟没动静。看起来跟变砖一模一样。

我第一次就是这么判断的，进 fastboot 把备份刷了回去，白折腾一轮。后来才知道 ATF 的签名校验发生在 preloader 阶段的 `bl2_ext`，远早于内核。**能看到第二屏，就说明补丁已经被接受了**，后面只是启动链里某处在等 GZ 的回应，等不到就超时往下走。

所以判据就两条：

停在第二屏，`adb devices` 能看到设备，等，等到满三分钟。停在第一屏，或者黑屏之后自己跳进 fastboot，那才是真出问题。

顺便说个我没搞清的现象：备用机刷完用了一段时间之后，这个等待明显变短了，现在几乎不卡。可能是最初几次开机叠了什么一次性的事，也可能跟服务重试的退避有关，我没验证。文档里还是按"每次都要等"写的，保守一点没坏处。

## 验证

```bash
adb shell su -c 'ls -l /dev/kvm'
```

看到 `crw-rw-rw- 10, 232` 就成了。

想确认不是假象，用系统自带的 crosvm 拉一个 microdroid 起来，能看到 guest 内核打印 `Booting Linux on physical CPU` 就是整条链闭环了。命令在仓库 docs/01 里。

## 想要录屏也正常

裸刷 tee 之后，硬件编解码会失效：录屏 0 字节，Moonlight 无响应，QQ 图片不显示。原因是 venc/vdec 在 open 时要通过 IPI 问 VCP 支持哪些帧尺寸，而 VCP 的握手依赖 EL2 那条链。

解决办法是把 `vendor_boot` 的 dtb 里两个 compatible 字符串各改一个字符，让驱动匹配不上、不绑定，框架找不到硬编只能走软编：

```
mediatek,mt6895-vcodec-dec → mediatek,mt6895-vcodec-de0
mediatek,mt6895-vcodec-enc → mediatek,mt6895-vcodec-en0
```

一共 2 个字节，长度不变。改完重启，`screenrecord` 出来 1.8 MB 的完整 H.264，logcat 里编码器是 `c2.android.avc.encoder`。

这个我写成脚本了：

```powershell
cd <本仓库>\scripts
.\flash-vendor-boot.ps1
```

它会备份原厂 `vendor_boot_a`，基座对得上就用仓库成品，对不上就用你本机的备份现场改那 2 个字节，写完回读校验。回滚镜像的路径也会打印出来。

有件事得提醒：软编能不能用，取决于你的 ROM。我主力机那台 A15 的 ROM 里，`libmedia.so` 缺 `MetaDataBase::writeToParcel` 这个符号，软编根本起不来；备用机的 A16 正常。同一套方案两台机器两个结果，所以刷完一定要自己跑一次 `screenrecord` 看字节数，别信文档。

## 出问题的时候

卡在第一屏，或者黑屏之后进 fastboot：补丁没被接受。先确认回读的 `tee_a` 哈希是不是你想要的那个，再确认基座对不对。救回来很简单，进 fastboot：

```bash
fastboot flash tee_a tee_a_backup.img
fastboot reboot
```

只动 `tee_a` 一个分区，用户数据和系统都不受影响。这条路我自己用过。

卡在第二屏超过五分钟没有任何变化：也回滚，换正确基座的成品再试。

系统起来了但 QEMU 起不来：一般是三个地方。`-cpu host` 必须配 `taskset f0` 绑到 A78 簇，不然 big.LITTLE 上会随机失败；`-vnc 127.0.0.1:0` 里的 0 是 display 号不是端口，端口是 5900；DroidVM 的 QEMU 需要 `export LD_LIBRARY_PATH=/system/lib64`。

## 几件别做的事

别用 SP Flash 深刷 tee，那会把 BL 重新锁上，之后 fastboot 就不方便了。

别拿别人博客里的哈希硬套。这套工具是固件样本级别的，差一个字节都不认。跨基座的补丁实测能启动，但那属于运气，不属于方法。

别把开了 KVM 的机器当主力机。软编吃 CPU，1080p 还行，4K 或者高帧率会卡；4 GB 内存分给虚拟机之后手机自己也紧张。我这台现在就是台实验机。

## 最后

刷完的样子：Windows 11 ARM64 能进桌面，能上网，CPU-Z 单核 247、多核 709，录屏和串流都正常。

至于银行 App 和各种检测，跟刷不刷 tee 没关系。补丁只改了 EL2 归谁，没碰 TEE。KeyMint、Gatekeeper、Widevine、指纹、安全元件，我刷前刷后逐项对过，一模一样。BL 解锁本来就是 Play Integrity 过不去的原因，那是另一件事。

代码、文档、成品镜像、还有那份记了十几条坑的清单，都在 [xaga-windows-arm64-kvm](https://github.com/citie114514/xaga-windows-arm64-kvm)。真卡住了，提 issue 比在评论区问快。
