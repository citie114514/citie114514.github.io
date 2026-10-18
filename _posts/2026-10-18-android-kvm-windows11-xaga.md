---
layout: post
title: "给天玑 8100 开 KVM：一次把 Windows 11 ARM64 塞进 Android 的过程"
date: 2026-10-18 18:00
author: "磁贴"
header-img: "img/post-bg-kvm-xaga.jpg"
header-pos-mobile: "left center"
catalog: true
tags:
    - Android
    - KVM
    - MTK
    - Windows
    - QEMU
    - tee
---

这台 Redmi Note 11T Pro+ 现在能跑 Windows 11 ARM64，CPU-Z 里处理器写着 virt-10.0，系统信息里设备型号直接就是 "KVM Virtual Machine"。不是模拟，是硬件加速。

但过程比结果难看多了：我误判过一次「变砖」，把成品的方案判死过一轮，还因为有段时间录屏突然恢复正常，重新推翻了自己写的结论。这篇把过程按发生顺序写下来，包括那些走错的地方。

## 手机为什么没有 /dev/kvm

先看现象。Android 内核里没有这个节点，不是内核没编，是拿不到虚拟化扩展。

ARM 有 EL0 到 EL3 四层权限，Linux 内核跑在 EL1，KVM 要求它把 hypervisor 架在 EL2。MTK 平台上 EL2 被 GenieZone 占着，这是联发科自己的一套虚拟化框架，跑在处理器的安全固件里。

所以问题不在 Android，也不在 QEMU，在于那段固件。

`ls /dev/gz_kree` 有、`/dev/kvm` 没有，就是这个状态的体现。

## 换个角度看启动链

我一开始想的是改内核，后来发现方向错了。真正决定 EL2 归谁的是 `tee_a` 分区里的一段 ATF 固件，它在内核起来之前就已经把交接做完了。

动它之前先确认了两件事。

一是 Secure Boot 到底开没开。preloader 会把自己的判定结果写进 `expdb` 分区，把那个分区拖出来 grep 一下：

```bash
adb shell su -c 'dd if=/dev/block/by-name/expdb of=/data/local/tmp/expdb.img bs=1M'
adb pull /data/local/tmp/expdb.img
grep -a -o "sbc_en = [01]" expdb.img | sort | uniq -c
```

结果是 440 条 `sbc_en = 1`。这个值来自 eFuse，一次性烧写，改不了。意思是每次开机都会对 ATF 做证书链校验，改过的固件没有合法签名就过不去。

二是签名怎么绕。这里用的不是私钥，是 MTK 证书解析的一个漏洞，工具叫 pwnage24mtk。它签出来的镜像能让校验通过，代价是你必须自己承担「我在修改设备信任链」这件事。

## 具体做了什么

流程本身不复杂，麻烦的是每一步的校验。

先从设备本身 dump 出 `tee_a`、`lk_a`、`preloader` 三个文件。不用网上找的同型机固件包，因为补丁工具是按完整 SHA-256 匹配的，差一个字节都不认。

然后构建。改的是 `tee_a` 里 `atf` 成员的启动交接逻辑，让 LK 进 EL1h、内核交接到 EL2h，同时把 ATF 和 LK 共享的那块 GZ-info tag 同步过去。工具会跑 14 项离线回归，检查 PC/SPSR 状态、两边的 tag parser、GZ gate，还有几条故意做错的反例得被拒绝。

签名之后要求看到两个 `Result: VALID`，ATF 组一个、TEE 组一个。少一个都不能刷。

最后是个很蠢但差点卡住我的细节：签名会在 ATF 后面插一段证书包装，产物比分区大 1072 字节。`tee` 分区是 5 MiB，正好被原始镜像占满。解决办法是裁掉尾部 1072 字节，但前提是得逐字节确认那段全是 0x00。我写脚本强制先验证再裁，因为如果布局和预期不一样，硬裁就是把分区写坏。

写进去之后回读比对 sha256，然后重启。

## 那次「变砖」

重启之后屏幕停在第二屏（logo 转圈那个），一分半没有任何变化。我以为是补丁没被接受，进了 fastboot，把备份刷了回去。

后来才知道那是个误判。补丁生效后每次开机都会在第二屏卡一到两分钟，原因大概是启动链里某处在等 GZ 的回应，等不到就超时继续。ATF 的签名校验发生在 preloader 阶段的 `bl2_ext`，远早于内核，所以能看见第二屏本身就说明补丁已经被接受了。

现在我的判断标准就两条：停在第二屏、并且 `adb devices` 能看到设备，就等满三分钟，别碰电源键。停在第一屏或者黑屏后自己跳进 fastboot，那才是真出问题。

这个教训写在文档里，因为我自己就踩过一次。

## 能开机之后，录屏坏了

跑起来第一件事是试录屏，5 秒的 screenrecord，文件 0 字节。

原因在 MTK 的编解码器上。venc 和 vdec 在 open 的时候要用 IPI 向 VCP 协处理器查询支持的帧尺寸，而 VCP 的 READY 握手依赖 EL2 那条链。内核被抬到 EL2，握手就断了，尺寸表读不出来，Codec2 的 configure 直接返回 EINVAL。框架拿到这个错误不会降级到软编，它就直接放弃了。

Moonlight、UU 远程、QQ 聊天图片，全是这一个原因。

我一开始的结论是「编解码和 KVM 互斥，没有两全的办法」。这个结论错了。

## 改两个字节

`venc` 和 `vdec` 在 device tree 里靠 compatible 字符串和驱动配对。把 `vendor_boot` 的 dtb 里那两个字符串各改一个字符：

```
mediatek,mt6895-vcodec-dec → mediatek,mt6895-vcodec-de0
mediatek,mt6895-vcodec-enc → mediatek,mt6895-vcodec-en0
```

长度不变，总共只动 2 个字节。驱动匹配不上就不绑定，MTK 的编解码器不再注册，框架找不到硬编就只能走软编。

改完重启，`screenrecord` 出来 1.8 MB 的完整 H.264，logcat 里编码器是 `c2.android.avc.encoder`，软编。

这里还有个插曲。这个方案我一度判过「无效」，因为第一次试的时候 HAL 崩了。后来发现那次测的状态组合不对，而且「软编也废」这个结论本身是拿主力机的 A15 ROM 外推出来的：那台机器上 `libmedia.so` 缺 `MetaDataBase::writeToParcel` 这个符号，软编确实起不来；但备用机的 A16 ROM 上配套正常。

同一个方案，两台机器两个结果。我原来把单台机器的结论当成了普遍规律，这是错的。

## 现在的状态

刷完之后这台机器的能耐：

Windows 11 ARM64 能进桌面，能上网，能跑 CPU-Z，单核 247 多核 709。录屏、相机录像、QQ 图片、Moonlight 串流都正常，走软编。

代价也真实存在。软编吃 CPU，1080p 没什么问题，4K 或者高帧率会卡。4 GB 内存分给虚拟机之后，手机自己也紧张。所以这台机器现在就是台实验机，不是主力机。

至于应用检测，跟刷不刷 tee 没什么关系。补丁只改了 EL2 归谁，没碰 TEE。KeyMint 硬件密钥、Gatekeeper、Widevine、指纹、安全元件，实测刷前刷后逐项一致。BL 解锁本来就是 Play Integrity 过不去的原因，那是另一码事。

## 几个命令，留给想动手的人

```bash
# 补丁前先看自己是什么固件批次
adb shell su -c 'dd if=/dev/block/by-name/tee_a bs=4096 2>/dev/null | sha256sum'
adb shell su -c 'dd if=/dev/block/by-name/tee_b bs=4096 2>/dev/null | sha256sum'

# 刷回原厂（系统还能起来的时候用 dd，起不来就进 fastboot）
adb push tee_a_backup.img /data/local/tmp/tee_stock.img
adb shell su -c 'dd if=/data/local/tmp/tee_stock.img of=/dev/block/by-name/tee_a bs=4096 && sync'
adb reboot
```

`tee_b` 从来没被动过，切到 B 槽就是没打补丁的状态。这是天然的后备。

最后说一句可能不太受欢迎的话：别照抄别人博客里的哈希值。这套工具按固件样本精确匹配，同一个型号不同批次的 `tee_a` 都可能不一样，我手上两台设备就差着（`f8f286f1` 和 `a91f5ded`）。拿错基座的补丁有可能开得起来，但那属于运气，不属于方法。

代码、文档、成品镜像都在 [xaga-windows-arm64-kvm](https://github.com/citie114514/xaga-windows-arm64-kvm)，包括那个让我误判过两次的坑清单。
