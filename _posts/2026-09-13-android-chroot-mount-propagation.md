---
layout: post
title: "从 IPv6 外网访问到 Mount Propagation：一次 Android chroot 故障的完整排查"
date: 2026-09-13 00:00
author: "磁贴"
header-img: "img/post-bg-unix-linux.jpg"
catalog: true
tags:
    - Android
    - Linux
    - chroot
    - IPv6
    - mount
    - 排障
---

最开始，我只是想解决一个看起来很普通的问题：

**让家里的 Android 手机服务器，通过福建联通的公网 IPv6 从外网访问。**

最后，这件事却一路从 DDNS、IPv6、Cloudflare Tunnel、OpenFRP、自启动，一直挖到了 Linux mount propagation、Binder、devpts、PTY，甚至把 SSH 一并拖下了水。

真正修复问题的代码其实只有很少几行，但找到这些代码为什么需要存在，花掉了远比修改本身多得多的时间。

这篇就把整个过程记下来，包括那些当时看着合理、后来被新证据推翻的判断。

---

## 一切的起点：IPv6 明明通，为什么外网就是连不上？

我的设备环境有点特殊。

服务机并不是一台传统意义上的 Linux 服务器，而是一台 Android 手机。网络结构大致是：

```text
福建联通
   │
中兴光猫
192.168.1.1
   │
小米路由器
192.168.31.1
   │
Android 手机
192.168.31.78
```

手机上通过 chroot 跑着 Ubuntu，里面有 OpenList、nginx、qBittorrent、OpenSSH、Tailscale、frpc、cloudflared 等服务。

最开始，我的目标非常简单：

> DDNS-GO 更新公网 IPv6 → 外网直接访问手机上的服务。

手机本身确实拿到了公网 IPv6，出站 IPv6 也完全正常。

一开始甚至连手机的公网 IPv6 都没从局域网邻居表里看到，于是我一度怀疑手机根本没有拿到公网地址。

结果换一种方式，从 DNS 解析出来的 IPv6 地址反向测试，反而发现它确实就是手机。

也就是说：

> **公网 IPv6 有，IPv6 出站有，甚至 ICMPv6 入站都能通，但 TCP 就是不通。**

这时候问题就开始变得有意思了。

---

## 从 sshd 一路查到光猫

进一步登录手机检查后，我发现 sshd 最初只监听 IPv4：

```text
ListenAddress 0.0.0.0
```

于是改成 IPv6 后继续测试。

结果又遇到了 Android 特有的奇怪行为：只写 IPv6 后，IPv4 监听反而出了问题。

最后采用双行：

```text
ListenAddress 0.0.0.0
ListenAddress ::
```

才实现真正的双栈监听。

但即使这样，外网 TCP 仍然无法连接。

于是开始从网络最上游一路往下查。

全国多个测试节点的结果很有代表性：

* 大多数节点都能解析出 IPv6
* ICMPv6 可以到达
* TCP 连接全部失败

这已经基本排除了 DNS 和 DDNS。

接着继续进入光猫检查。

光猫的普通 Telnet 权限很有限，但通过隐藏管理入口拿到了更高权限，检查了防火墙与转发规则。看到的结果却相当奇怪：

> IPv6 转发策略并没有明显拒绝入站 TCP。

甚至光猫自己明确开放的 WAN 端口，从外网测试也一样 TCP 不通，但 ICMPv6 依然可以到达。

这时候结论开始清晰起来：

**不是我的 DDNS 错了，不是手机没有公网 IPv6，也不是局域网路由器简单地把端口挡住了。**

而是这条 IPv6 入站链路本身存在"可以 Ping，但 TCP 进不来"的限制。

于是我决定不再和这条链路死磕。

---

# 换一个方向：既然入站进不来，那就主动向外连

既然家庭网络的 IPv6 入站 TCP 走不通，那最自然的方案就是：

> 不让外面的客户端主动打进来，而是让手机主动建立出站隧道。

最终采用了两条路线。

### Web 服务：Cloudflare Tunnel

OpenList 和 OpenSpeedTest 之类的 HTTP/HTTPS 服务通过 Cloudflare Tunnel 暴露：

```text
用户浏览器
    ↓
Cloudflare
    ↓
cloudflared
    ↓
localhost:5244 / localhost:3000
```

这样客户端不需要安装任何额外软件，浏览器直接访问域名即可。

### SSH：OpenFRP

SSH 是另外一种情况。

Cloudflare Tunnel 对普通 TCP/SSH 客户端存在额外限制，如果要求外网直接使用标准 SSH 客户端，我最终还是选择了 OpenFRP：

```text
SSH Client
   ↓
OpenFRP
   ↓
frpc
   ↓
sshd :22
```

最后形成了一个很实用的组合：

| 服务            | 外网入口              |
| ------------- | ----------------- |
| OpenList      | Cloudflare Tunnel |
| OpenSpeedTest | Cloudflare Tunnel |
| SSH           | OpenFRP           |
| OpenList 备用入口 | OpenFRP           |

而且 Cloudflare 和 OpenFRP 之间还有一定程度的互备。

至此，外网访问问题基本解决。

但真正的大坑，才刚刚开始。

---

# 接下来，我决定让手机上的服务真正"像服务器一样"开机自启

最开始这台手机并不像服务器。

Android 开机以后，容器并不会自己完整启动。

没有 systemd，没有传统意义上的 rc.local，/data/adb/service.d 也基本是空的。

于是开始建立自己的启动链。

核心结构大致变成：

```text
Android 开机
   ↓
Magisk service.d
   ↓
等待网络
   ↓
挂载 chroot
   ↓
进入 Ubuntu
   ↓
启动 sshd
   ↓
启动 services.sh
   ↓
启动 Tailscale
   ↓
启动 cron
```

OpenList 又比较特殊。

它本身是 Android 原生应用，并不是 chroot 服务，所以只能通过 Android 的 `am` 来启动，而不是在 Ubuntu 里面直接启动。

慢慢地，这台手机开始真的像一台小型服务器了。

---

# 然后，一个"小需求"把事情带到了完全不同的方向

后来只是想做两个小调整：

一个是修复 citie 用户的 sudo。

另一个是把 ddns-go 从主服务里移出去。

结果这两个小需求，最终把整个系统最深层的问题都暴露出来了。

---

# 第一个异常：sudo 怎么都不能正常工作？

最终发现问题不是 sudo 本身，而是：

> Android 的 /data 所在文件系统以 `nosuid` 方式挂载。

因此 /usr/bin/sudo 即便本身是 setuid，也无法正常利用 setuid 机制提权。

最后通过对 chroot 根目录做 bind/remount，使容器这部分重新具备需要的 suid 行为，再配合 sudoers 配置，sudo 才恢复。

问题解决了。

但真正麻烦的东西随后出现。

---

# 第二个异常：OpenCode/Bun 莫名其妙 Segmentation Fault

OpenCode 在容器里面运行时出现：

```text
Segmentation fault
abort
```

一开始看起来像是 Bun、JavaScriptCore 或者 ARM64 环境的问题。

但继续深入后，发现一个非常诡异的事实：

```text
/proc/self/maps
/proc/version
```

这些本来应该一定存在的 /proc 内容，在容器里面竟然直接报：

```text
No such file or directory
```

这意味着：

> **不是 OpenCode 本身坏了，是容器里的 /proc 已经坏了。**

于是开始把宿主 Android 的：

```text
/dev
/proc
/sys
/dev/pts
```

bind 到 chroot 里面。

OpenCode 果然恢复。

当时我以为：

> 好了，问题解决。

后来才发现，这其实只是把真正的问题埋了下去。

---

# 一个长期存在的误判：35930 到底是什么？

这次排查里，我还纠正了一个自己错了很久的判断。

之前经常拿：

```text
:35930
```

来判断 OpenList 是否正常。

后来发现：

> **35930 实际上是 tailscaled 使用的端口。**

所以之前很多"OpenList 已经正常启动"的判断，其实是假的。

这也让我意识到：

监控指标本身选错了，后面所有"服务正常"的结论就都可能建立在错误的基础上。

后来改成真正的 HTTP 访问：

```text
localhost:5244 → HTTP 200
localhost:3000 → HTTP 200
localhost:8080 → HTTP 200
localhost:8443 → HTTP 200
```

这才算真正验证服务。

---

# OpenList 又"炸了"

过了一阵子，我再次发现 OpenList 访问异常。

这一次的问题已经不像普通应用故障。

SSH 有时候能连上，但登录后马上断开。

Android 的 `am` 命令偶发异常。

PTY 报：

```text
out of pty devices
```

更严重的是：

```text
/dev/binder
```

开始出现异常。

于是继续排查。

这一次终于碰到了真正的核心：

# Mount Propagation

---

# 原来 chroot 根本没有你想象中的"隔离"

很多人第一次接触 chroot 时，会天然认为：

> "既然进了容器，那里面的挂载就是容器自己的。"

但 chroot 并不自动创建 mount namespace。

这意味着：

**文件系统根目录变了，不代表 mount namespace 也隔离了。**

而我的 Android chroot 就属于这种情况。

宿主的：

```text
/dev
/proc
/sys
/dev/pts
```

本身处于 shared propagation。

例如：

```text
/dev       shared:2
/dev/pts   shared:3
/proc      shared:4
/sys       shared:5
```

问题就从这里开始。

---

# 真正危险的不是 bind，本身是 propagation

表面上看，我执行的是很普通的操作：

```sh
mount -o bind /dev "/dev"
mount -o bind /proc "/proc"
mount -o bind /sys "/sys"
mount -o bind /dev/pts "/dev/pts"
```

从字符串层面完全没问题。

甚至如果你只是做静态审计，很可能会得出：

> "这几个 bind 看起来挺正常。"

但 Linux 真正危险的地方在于：

> **这些 mount 自己属于什么 propagation group？**

因为目标 mount 仍然处于 shared peer。

于是：

```text
bind
  ↓
产生 mount event
  ↓
沿 shared propagation 传播
  ↓
宿主 peer 收到事件
  ↓
宿主对应路径产生新的 mount
  ↓
覆盖原来的 mount
```

然后灾难开始。

---

# 宿主的 mount 开始一层一层堆起来

排查过程中，可以看到宿主的 /dev、/proc、/sys、/dev/pts 上不断出现新的 mount ID。

原本应该是：

```text
/dev      41
/dev/pts  42
/proc     43
/sys      44
```

后来开始出现：

```text
41
104227
67901
...
```

其他路径也是类似情况。

这意味着每次 bind 都没有老老实实停留在 chroot 里。

它实际上在**反向影响宿主 Android**。

---

# 而真正倒霉的是 Binder 和 devpts

Android 的 Binder 本来位于：

```text
/dev/binderfs
```

PTY 则依赖：

```text
/dev/pts
```

当这些目录被新的覆盖层遮掉以后，后果非常直接。

### Binder

原本：

```text
/dev/binder
→ /dev/binderfs/binder
```

后来 Binderfs 被覆盖，/dev/binder 变成异常状态。

结果：

```text
am
ADB
scrcpy
```

都开始出现：

```text
Binder driver '/dev/binder' could not be opened
```

### PTY

/dev/pts 被异常挂载覆盖之后：

```text
/dev/pts/ptmx
```

消失。

然后 sshd 虽然还能监听：

```text
:22
```

但是给用户创建伪终端失败。

于是出现一个非常迷惑的现象：

```text
SSH connection
    ↓
能连上
    ↓
能看到 Ubuntu banner
    ↓
突然 Connection closed
```

实际上不是 SSH 密码，也不是 sshd 配置。

而是：

> **sshd 根本拿不到 PTY。**

---

# 然后又发现了第二个问题：boot 脚本居然执行了两遍

事情已经够复杂了，结果继续查日志又发现：

```text
boot script start
boot script start
```

同一时间出现两次。

后来发现 /data/adb/service.d 里居然放着一个：

```text
99-chroot-boot.sh.bak-propfix
```

而且还是：

```text
0755
```

也就是说，它不是普通备份。

它是一个：

> **可执行的完整 Magisk service.d 脚本。**

Magisk 看到以后，自然把它也执行。

于是：

```text
正式 boot 脚本
      +
备份 boot 脚本
      ↓
执行两遍
      ↓
bind 两遍
      ↓
propagation 两遍
      ↓
宿主 overlay 增长得更快
```

这就解释了为什么问题会被放大得如此严重。

---

# 最终的修复方向终于明确

这时候已经有三个明确的根因：

### 根因一

mount propagation 没隔离。

### 根因二

boot 脚本被可执行备份文件重复执行。

### 根因三

之前使用的 mount --make-private 工具选错了。

Android 的 toybox：

```text
/system/bin/mount
```

不支持我们需要的那种语法。

而 BusyBox：

```text
/system/xbin/mount
```

虽然支持 private，但：

```sh
mount -o private /path
```

会因为参数形式进入 fstab 解析，同样失败。

最终实测下来，当前设备真正可用的是：

```sh
/system/xbin/mount --make-private "/path"
```

返回：

```text
rc=0
```

并且 shared: 计数实际下降。

---

# 最后的修复其实非常简单

对于每个目标：

```text
/dev
/proc
/sys
/dev/pts
```

在 bind 前后做 private：

```sh
/system/xbin/mount --make-private "/X" 2>/dev/null
mount -o bind /宿主/X "/X"
/system/xbin/mount --make-private "/X"
```

也就是：

```text
bind 前：
先把嫁接点设为 private

bind：

再把新 mount 设为 private
```

同时把可执行的：

```text
99-chroot-boot.sh.bak-propfix
```

移出：

```text
/data/adb/service.d
```

---

# 为什么必须"前后各一次"？

这里最关键的一点不是"命令成功了"。

而是：

> **bind 前的那个 mount 和 bind 后新产生的 mount，都必须确保 propagation 属性符合预期。**

所以最终方案不是简单地把一条失败命令换掉，而是形成：

```text
目标 mount
    ↓
private
    ↓
bind
    ↓
新 mount
    ↓
private
```

这样才能确保 mount event 不再继续向宿主传播。

---

# 真正的验收：重启

这种问题最怕的是：

> 当前看着好了，下一次开机又复发。

所以最后不是简单地看：

```text
rc=0
```

而是直接重启。

然后分别检查：

```text
boot + 1 min
+120s
+360s
```

结果非常漂亮。

修复前：

```text
/dev      41 → 104227 → 67901 → ...
/proc     43 → 102553 → 49150 → ...
/sys      44 → 103204 → 57972 → ...
/dev/pts  42 → 大量 tmpfs
```

修复后：

```text
/dev      41
/dev/pts  42
/proc     43
/sys      44
```

三次采样：

> **没有任何增长。**

---

# 然后进行真正的功能验证

### Binder

恢复：

```text
binder
binder-control
hwbinder
vndbinder
```

`am` get-current-user：

```text
0
```

### PTY

测试：

```text
pty.openpty()
→ PTY_OK
```

不再出现：

```text
out of pty devices
```

### SSH

最终真的建立：

```text
/dev/pts/0
```

交互式 SSH 会话成功。

### 服务

同时：

```text
nginx       → HTTP 200
OpenList    → HTTP 200
qBittorrent → HTTP 200
Tailscale   → Online
frpc        → 正常
cloudflared → 正常
```

至此，才真正可以说：

> 修复完成。

---

# 还有一个很有意思的后续：frpc 到底是不是"双实例故障"？

稳定性审计过程中，又发现：

```text
frpc PID 7732
frpc PID 9005
```

第一反应很容易是：

> "怎么又启动两份了？"

结果只读检查后才发现：

```text
7732
→ SSH tunnel

9005
→ OpenList tunnel
```

它们：

* PID 不同
* PPID 都是 1
* 配置来源不同
* 日志不同
* 各自只负责一个 tunnel

而 `frpc.sh` 本身就明确写着：

```text
INSTANCES="ssh openlist"
```

所以这里的结论恰恰是：

> **两个 frpc 进程不是异常，而是设计如此。**

这又是一次很典型的"不能只看数量判断系统是否异常"。

---

# 最终系统状态

经过完整审计之后，当前系统已经运行了超过 20 小时，核心状态稳定。

整个系统大概变成这样：

```text
                    Android
                       │
               ┌───────┴───────┐
               │               │
          Magisk boot       Android 服务
               │
         chroot mount
               │
        ┌──────┴──────┐
        │             │
     private       宿主 shared
        │             │
     Ubuntu        Android
        │
   ┌────┼──────────────┐
   │    │              │
 sshd  services      cron
   │    │
   │    ├─ nginx
   │    ├─ OpenList
   │    ├─ qBittorrent
   │    ├─ frpc ×2
   │    └─ cloudflared
   │
   └── 外部 SSH
```

而 mount 层则保持：

```text
宿主：
/dev      41
/dev/pts  42
/proc     43
/sys      44

chroot：
/dev      private
/dev/pts  private
/proc     private
/sys      private
```

没有继续产生覆盖层。

---

# 这次故障最值得记住的几个教训

## 1. chroot ≠ mount namespace 隔离

这是整次事件最大的教训。

很多人看到：

```text
chroot
```

就会下意识认为"容器里的 mount 不会影响外面"。

实际上不是。

如果 mount namespace 没有隔离，传播机制依然可能把你的操作带回宿主。

---

## 2. 静态代码看不懂 mount propagation

单纯搜索：

```text
mount -o bind
```

很难发现真正的问题。

因为：

```text
mount -o bind /dev "/dev"
```

从语法上完全正常。

真正的问题是：

> **这个目标 mount 是 shared 还是 private？**

这属于运行时内核语义，而不是字符串层面的 bug。

---

## 3. 不要把"端口开着"当成服务健康

35930 当时确实是 UP。

但：

> 35930 属于 tailscaled。

所以：

```text
端口 UP ≠ OpenList 正常
```

正确做法应该是：

```text
HTTP 200
实际 API 响应
真实 SSH 登录
真实 PTY
```

---

## 4. 备份文件绝对不要放进自动执行目录

这一点几乎可以写进每一个 Magisk 脚本项目的 README：

> **service.d 不是普通文件夹。**

一个：

```text
99-chroot-boot.sh.bak
```

如果还带着：

```text
0755
```

那它就不再是备份。

它是：

> **另一个启动脚本。**

---

## 5. 满盘问题经常比真正的 bug 更危险

这次排查过程中还出现过：

* 脚本被写成全 \0
* 服务启动失败
* 下载任务刚加就失败
* 开机异常

后来发现 /data 曾经一度接近甚至达到 100%。

所以我越来越觉得：

> **磁盘空间应该和 CPU、内存一样，属于基础健康指标。**

---

# 最后的回顾

现在再回头看整件事情，会发现最有意思的地方是：

我最初只是想解决：

> "为什么我的公网 IPv6 从外面访问不了？"

真正走到最后，却变成了：

```text
IPv6
 ↓
光猫
 ↓
Cloudflare / OpenFRP
 ↓
Android 自启动
 ↓
chroot
 ↓
/proc
 ↓
/dev
 ↓
mount propagation
 ↓
Binder
 ↓
devpts
 ↓
PTY
 ↓
SSH
```

最后真正解决问题的代码，甚至没有多少。

但这也是系统排障最有意思的地方之一：

> **真正困难的，从来不是最后写出那几行代码，而是确定那几行代码为什么必须存在。**

从"OpenList 又炸了"，到发现 /dev/binder 消失；从 SSH banner 后断开，到发现根本没有 PTY；从 mount ID 不断增长，到最终定位 shared propagation。

每一个看似独立的问题，最后其实都是同一条因果链上的不同表现。

而现在，这台原本只是"拿来折腾"的 Android 手机，已经真正变成了一台跑着 Ubuntu chroot、反向隧道、Web 服务、SSH、定时任务和自动证书续期的小型家庭服务器。

这次故障算是正式结案。

> 有时候，最深的 bug，藏在最普通的一行 mount -o bind 后面。