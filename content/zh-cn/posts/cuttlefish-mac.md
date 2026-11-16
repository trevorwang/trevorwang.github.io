---
title: "Cuttlefish 上 Mac：加速是真的，500GB 也是真的"
date: 2026-11-16T07:16:05+08:00
draft: false
categories: ["公众号"]
tags: ["Android", "QEMU"]
summary: "Meta 工程博客讲了把 Cuttlefish 搬到 Apple Silicon 的全过程。我照着做了一遍：19 个离线单测、130 个补丁全过、QEMU 编出来、HVF 比 TCG 快 15.4 倍，最后卡在 500GB 磁盘上。"
cover:
    image: "/images/cuttlefish-mac/cover.png"
    alt: "Cuttlefish 上 Mac：加速是真的，500GB 也是真的"
    relative: false
---

11 月 13 日，Meta 工程博客发了篇《Cuttlefish on Mac》，作者是 Hadrien Zalek、Joe Knox、Alex Chong 和 Marcel Apfelbaum。文章讲的是 Reality Labs 怎么把 AOSP 的虚拟设备框架搬到 macOS 和 Apple silicon 上。

Cuttlefish 一直是 Linux 独占的。它和 Android Studio 里那个 Goldfish 不是一回事：Goldfish 面向应用开发者，跑得动就行；Cuttlefish 跑的是和真机同一套系统镜像，面向平台开发，能测蓝牙、Wi-Fi、蜂窝、3D 图形和 OTA 升级流程。

Meta 的处境是，他们的应用和库开发者全在用 Apple silicon 的 Mac，而高保真的那套虚拟设备只在 Linux 上有。于是团队被迫同时维护 Goldfish 和 Cuttlefish 两条线，工具每个都要支持两套平台，基础设施和人力被劈成两半。

博客给出的解法是把 Cuttlefish 本身搬到 Mac 上，而不是让 Mac 去连一个 Linux 虚拟机。他们把代码开源成两个仓库：`qemu-wearables` 是带 Meta 提交的 QEMU 分支，`cuttlefish-wearables` 是一批补丁和打补丁的说明。

我把两边都拉到本机跑了一遍，有些东西博客里没写。

## 他们改了三个只存在于 Linux 的假设

Cuttlefish 本身是编排层，真正跑客户机的是虚拟机监视器。Linux 上默认用 crosvm，QEMU 曾经是个备选，后来那条路径荒废了。为了在 macOS 上拿到原生支持，Meta 把 QEMU 这条线捡回来了。

问题在于，Cuttlefish 的很多虚拟设备走 vhost-user：独立进程通过 socket 向 QEMU 提供 virtio 设备。架构很干净，但代码里写死了 Linux。

第一个假设：vhost-user 那套机制只编译在 Linux 上，libvhost-user 和 vhost-user-blk 这些后端在 macOS 上根本编不过。他们的做法是放宽构建限制，让前端和库在 macOS 上也能编译。

第二个假设：Apple silicon 上 QEMU 靠 HVF 加速，也就是 Hypervisor.framework，而不是 KVM。QEMU 的 HVF 后端给内存区域追踪留的槽位不够，大一点的 Cuttlefish 配置跑不起来。他们把这个槽位数加上去了。

第三个假设最实在：没有可用的原生显示。之前只能通过 WebRTC 界面看虚拟设备。他们改用 QEMU 原生的 Cocoa 显示，做了三件事：支持 BGRA 像素格式消除 Apple silicon 上的视觉伪影，把鼠标点击翻译成触摸事件，给 `graphic_console_init` 加了 `-g` 参数用来配几何。

剩下的就是构建和打包：Darwin 平台支持、给 macOS 编 arm64 的 QEMU、工具链修复、JIT entitlement、代码签名和公证。

![三个只在 Linux 上成立的假设](/images/cuttlefish-mac/fig1.png)

## 补丁仓库的 commit 里写着另一句话

我先把 `cuttlefish-wearables` 拉下来，1.6MB，130 个 patch 文件，横跨 35 个 AOSP 项目。

然后我翻了 commit message。整个仓库只有一个提交，标题是「Add native macOS support for Cuttlefish」，正文写得很详细，末尾那句是 full AOSP sync, native macOS builds, and boot remain unvalidated。

翻译过来就是：源码级检查全过，19 个离线测试和 lint 也全过，但完整的 AOSP 同步、macOS 原生构建、以及开机，都还没有验证。

博客讲的是成果，仓库留的是实情。这个差别很重要，因为它决定了你照着做会撞到什么。

## 第一步：官方给的检查脚本，我跑了两遍

仓库 README 里有个 `check_patches.py`，思路很实在：不用把 100GB 的 AOSP 全下下来，只对 35 个被改过的项目做浅层 blobless 克隆，然后把每个 patch 应用到 Git 的索引里，不检出源码文件，失败就返回非零。

先跑离线单测，19 个测试 6.903 秒跑完，全过。

再跑补丁检查，这一步要联网拉 35 个仓库，末尾输出是 35/35 project series passed。

130 个补丁，35 个项目，一个都没失败。整个过程 3 分 58 秒，CPU 只占 8%，时间基本花在等网络上。

这是我在本机能拿到的第一层证据：Meta 的补丁是干净的，能对上 `android-16.0.0_r1` 的基线。

![本机能跑到的三步](/images/cuttlefish-mac/fig2.png)

## 第二步：QEMU 那半，编译只用了 49 秒

`qemu-wearables` 克隆下来 197MB，7.4 秒。分支叫 `stable-9.2-meta-external`，基于 QEMU 9.2，依赖是 ninja、meson 和 pkgconf 三件套，brew 装上就行。

第一次 configure 直接失败，它挑中了 `/usr/bin/python3`，报这一行 Python 3.9.6，然后一个大写的 Ouch 加 found no usable tomli, please install it。

系统自带的 Python 3.9 缺 tomli，QEMU 的 mkvenv 建不起来。指一下 Homebrew 的 Python 就过了。

第二次 configure 建出三个关键开关，全是绿的：cocoa 是 enabled，hvf 是 enabled，libvhost-user 是 YES。

`libvhost-user` 这条最值得看，它正是博客里说的「把 vhost-user 从 Linux 限制里解出来」的结果。上游 QEMU 在 macOS 上这一项是 NO。

然后 `make -j`，2900 个目标，49 秒，CPU 占用 779%。产物 34MB，叫 `qemu-system-aarch64-unsigned`。

## 编出来的东西，第一件事是先签名

看到那个 `-unsigned` 后缀我还没反应过来发生了什么事。

直接运行 `--version` 是好的，报 QEMU 9.2.3。查加速器：`hvf` 和 `tcg` 都在。查显示后端，列出来的是 none、sdl、curses 三个，没有 cocoa。但 `-display cocoa` 传进去是不报错的，显示模块也确实编译了（`build/libcommon.a.p/ui_cocoa.m.o` 在），只是不在 help 列表里。

真正的问题在加速器。我让 QEMU 起一台 4 核 2G 的虚拟机：

```
qemu-system-aarch64-unsigned: -accel hvf: Error: ret = HV_DENIED
(0xfae94007, at ../accel/hvf/hvf-accel-ops.c:334)
```

HV_DENIED。二进制是未签名的，拿不到 Hypervisor.framework 的权限。

写一个 entitlements 文件，把 `com.apple.security.hypervisor` 和 `com.apple.security.cs.allow-jit` 打开，然后用 `codesign -s -` 给同一个二进制重新签了一次，输出是 replacing existing signature。

再跑同一条命令，不再报错，虚拟机直接起来了。

这就是博客里那句「macOS 构建还需要 JIT entitlement，并且要支持代码签名和公证」的具体后果。它不是一个打包细节，是你从零到能跑必须过的一关，而 README 里没写这一步。

![HV_DENIED 与签名后的差别](/images/cuttlefish-mac/fig3.png)

## 签名之后，测一下到底快多少

光看「没报错」没有说服力。我下了 Alpine 的 ARM64 内核和 initramfs，9.7MB 加 8.7MB，各跑 35 秒，比它能跑多远。

同一个 QEMU、同一个 `-cpu max`、同样 4 核 2G、同一个镜像，唯一变量是加速器：

```
HVF: 0.126 秒 到达 Mounting boot media
TCG: 1.944 秒 到达同一点
```

算下来 15.4 倍。两边都停在同一点（因为我没有挂安装介质，Alpine 找不到 boot media 就等在那里），所以这个对比是干净的。guest 内核自己的时间戳不会说谎：硬件加速下 Linux 在 0.126 秒里跑完了内核初始化、挂上 initramfs、加载完启动驱动；纯软件模拟要 1.94 秒。

顺带一个细节，`-cpu host` 和 `-cpu max` 在 HVF 下结果几乎一样，0.1255 和 0.1260，说明这个差距来自加速器本身，不是 CPU 模型。

![HVF 与 TCG 的启动耗时对比](/images/cuttlefish-mac/fig4.png)

## 想真跑 Android，先准备 500GB

到这里 QEMU 那半我是服气的：能编、能签、能加速，数据也对得上。

然后 README 告诉我下一步要建一个 500GB 的 case-sensitive 卷：

```bash
hdiutil create -type SPARSEBUNDLE -fs HFS+ -fsargs -s -size 500g \
  -volname 'AOSP' "${HOME}/AOSP.sparseimage"
```

500GB 的 case-sensitive 稀疏卷。因为 AOSP 里有只差大小写的文件名，普通 APFS 卷装不下。

我这台机器只剩 25GB，差了二十倍。就算空间够，后面还有 `repo sync -c -j8` 拉 274 个项目，再 source `build/envsetup.sh`、lunch 到 `aosp_cf_arm64_phone-bp2a-userdebug`，再用 m 构建出 `out/host/darwin-x86/cvd-host_package.tar.gz`。

注意那个输出路径，`darwin-x86`。这是在 Apple silicon 上构建 macOS 包，目录名还留着 x86。历史包袱，不影响功能，但看着有点好笑。

README 还写了句更实在的话：「Application is not proof of a successful build or boot.」补丁打上了不等于构建成功，构建成功不等于能开机。

![500GB 对 25GB](/images/cuttlefish-mac/fig5.png)

## 和我手上那个官方模拟器比

我本机装着 Android emulator 36.6.11.0，启动 banner 第一句就写着 This program is a derivative of the QEMU CPU emulator。

两边都是 QEMU。差别在于 Goldfish 只模拟到「够跑应用」，Cuttlefish 要把蓝牙、Wi-Fi、蜂窝、图形、OTA 整条链路都模拟出来。前者你随手装个 Android Studio 就有，后者得先有 500GB 和一套没被验证过的构建链。

Meta 的博客里说要「contribute these changes upstream to QEMU and AOSP」，目前在探索阶段。所以现在想用，只能走 fork 加补丁这条路。

## 什么时候值得动手

如果你只是想跑个 Android 应用看界面，Android Studio 的模拟器够用，别折腾。Goldfish 的启动速度和易用性都不是 Cuttlefish 能比的。

如果你在做平台层、要给非手机设备（眼镜、手表、车机）做开发，需要验证的是蓝牙配对、蜂窝注册、OTA 升级这些流程，那 Cuttlefish 的保真度确实是另一个量级的东西。Meta 之所以费这个劲，是因为他们的开发者要测 AI 眼镜上的东西，Goldfish 模拟不出来。

但你要做好准备：500GB 空间、一套 Xcode 命令行工具、会自己写 entitlements 并给二进制签名、接受构建可能失败且没人能帮你（Meta 自己都没跑通过完整流程）。

那 15 倍的加速是真的。500GB 也是真的。
