---
title: "Cuttlefish on the Mac: The Speedup Is Real, and So Is the 500GB"
date: 2026-11-16T07:16:05+08:00
draft: false
categories: ["WeChat"]
tags: ["Android", "QEMU"]
summary: "Meta's engineering blog walked through porting Cuttlefish to Apple silicon. I followed it on my own machine: 19 offline tests, all 130 patches verified, QEMU built, HVF 15.4x faster than TCG, and then the 500GB disk requirement stopped me cold."
cover:
    image: "/images/cuttlefish-mac/cover.png"
    alt: "Cuttlefish on the Mac: The Speedup Is Real, and So Is the 500GB"
    relative: false
---

On November 13, Meta's engineering blog published *Cuttlefish on Mac*, by Hadrien Zalek, Joe Knox, Alex Chong and Marcel Apfelbaum. It is the story of how Reality Labs moved AOSP's virtual device framework onto macOS and Apple silicon.

Cuttlefish had always been Linux-only. It is not the Goldfish emulator you get inside Android Studio: Goldfish is built for app developers and only has to run fast enough to be usable. Cuttlefish runs the same system images as real hardware. It targets platform work, and it can exercise Bluetooth, Wi-Fi, cellular, 3D graphics and the OTA update flow.

Meta's problem was that all of their app and library developers sit on Apple silicon Macs, while the high-fidelity virtual device only existed on Linux. The team ended up maintaining both Goldfish and Cuttlefish, every tool had to support two platforms, and both infrastructure and headcount were split down the middle.

The fix the blog describes is to move Cuttlefish itself to the Mac, rather than having the Mac reach out to a Linux VM. They open-sourced two repositories: `qemu-wearables`, a QEMU branch carrying Meta's commits, and `cuttlefish-wearables`, a set of patches plus instructions for applying them.

I pulled both down and ran them on my own machine. Some of it the blog does not mention.

## Three assumptions that only held on Linux

Cuttlefish is an orchestration layer. The thing that actually runs the guest is a hypervisor. On Linux that defaults to crosvm; QEMU was once an alternative, then that path rotted. To get native support on macOS, Meta revived the QEMU route.

The trouble is that many Cuttlefish virtual devices go through vhost-user: a separate process hands QEMU virtio devices over a socket. Clean architecture, but the code had Linux baked into it.

Assumption one: vhost-user only compiles on Linux, and backends like libvhost-user and vhost-user-blk simply do not build on macOS. Their fix was to relax the build constraints so the frontend and the libraries compile on macOS too.

Assumption two: on Apple silicon, QEMU accelerates through HVF, which is Hypervisor.framework, not KVM. QEMU's HVF backend did not reserve enough slots for tracking memory regions, so larger Cuttlefish configurations would not start. They raised the slot count.

Assumption three is the most concrete: there was no usable native display. Viewing the virtual device meant going through the WebRTC UI. They switched to QEMU's native Cocoa display and did three things: support the BGRA pixel format to kill visual artifacts on Apple silicon, translate mouse clicks into touch events, and add a `-g` argument to `graphic_console_init` for geometry.

The rest is build and packaging work: Darwin platform support, compiling an arm64 QEMU for macOS, toolchain fixes, the JIT entitlement, code signing and notarization.

![Three assumptions that only held on Linux](/images/cuttlefish-mac/fig1.png)

## The patch repo's commit message says something else

I started by cloning `cuttlefish-wearables`. 1.6MB, 130 patch files, spanning 35 AOSP projects.

Then I read the commit message. The whole repository is a single commit, titled "Add native macOS support for Cuttlefish". The body is detailed, and the last line is: full AOSP sync, native macOS builds, and boot remain unvalidated.

In plain terms: the source-only check passes all 130 patches across 35 project series, the 19 offline tests and lint pass too, and full AOSP sync, native macOS builds, and boot remain unvalidated.

The blog describes the achievement. The repository records the actual state. That gap matters, because it determines what you will run into if you follow along.

## Step one: the official check script, run twice

The README ships a `check_patches.py`, and the idea behind it is refreshingly practical. You do not download 100GB of AOSP. It shallow, blobless clones only the 35 projects that were touched, then applies each patch to the Git index without checking out any source files, and returns non-zero on failure.

Offline unit tests first: 19 tests in 6.903s, all passing.

Then the patch check. That step needs the network to fetch 35 repositories, and it ends with 35/35 project series passed.

130 patches, 35 projects, not one failure. The whole run took 3 minutes 58 seconds with CPU at 8%, which is to say almost all of it was spent waiting on the network.

That is the first layer of evidence I could get locally: Meta's patches are clean and line up against the `android-16.0.0_r1` baseline.

![The three steps I could reach locally](/images/cuttlefish-mac/fig2.png)

## Step two: the QEMU half builds in 49 seconds

`qemu-wearables` clones in 7.4 seconds at 197MB. The branch is `stable-9.2-meta-external`, based on QEMU 9.2, and the dependencies are the usual trio: ninja, meson and pkgconf, all a brew install away.

The first configure failed outright. It picked up `/usr/bin/python3`, reported Python 3.9.6, and then a large Ouch followed by found no usable tomli, please install it.

The system Python 3.9 has no tomli, so QEMU's mkvenv cannot be created. Point it at Homebrew's Python and it goes through.

The second configure produces three switches that matter, all green: cocoa enabled, hvf enabled, libvhost-user YES.

`libvhost-user` is the one worth looking at. That is exactly what the blog means by lifting vhost-user out of its Linux-only restriction. Upstream QEMU reports NO for this on macOS.

Then `make -j`: 2900 targets, 49 seconds, CPU at 779%. The output is 34MB and it is called `qemu-system-aarch64-unsigned`.

## The first thing the build needs is a signature

That `-unsigned` suffix did not register at first.

Running `--version` works fine and reports QEMU 9.2.3. The accelerators list has both `hvf` and `tcg`. The display backends list none, sdl, curses, and no cocoa. Yet passing `-display cocoa` does not error, and the display module clearly got built (`build/libcommon.a.p/ui_cocoa.m.o` is right there), it is just missing from the help list.

The real problem is the accelerator. I asked QEMU for a 4-core, 2G VM:

```
qemu-system-aarch64-unsigned: -accel hvf: Error: ret = HV_DENIED
(0xfae94007, at ../accel/hvf/hvf-accel-ops.c:334)
```

HV_DENIED. The binary is unsigned, so it cannot get Hypervisor.framework permission.

I wrote an entitlements file turning on `com.apple.security.hypervisor` and `com.apple.security.cs.allow-jit`, then re-signed the same binary with `codesign -s -`. The output said replacing existing signature.

The same command again: no error, the VM came up.

This is the concrete consequence of the blog's line about the macOS build needing the JIT entitlement, plus code signing and notarization. It is not a packaging detail. It is a gate you have to get through between zero and a running VM, and the README does not mention it.

![HV_DENIED versus after signing](/images/cuttlefish-mac/fig3.png)

## Signed, so let's measure how much faster it is

"No error" on its own proves nothing. I downloaded an ARM64 Alpine kernel and initramfs, 9.7MB plus 8.7MB, about 35 seconds each, and raced them.

Same QEMU, same `-cpu max`, same 4 cores and 2G, same image. The only variable is the accelerator:

```
HVF: 0.126s to reach "Mounting boot media"
TCG: 1.944s to the same point
```

That works out to 15.4x. Both sides stop at the same place (I had no install media attached, so Alpine looks for boot media and waits there), which makes the comparison clean. The guest kernel's own timestamps do not lie: under hardware acceleration Linux gets through kernel init, mounts the initramfs and loads the boot drivers in 0.126 seconds; pure software emulation takes 1.94 seconds.

One more detail: `-cpu host` and `-cpu max` land within noise of each other under HVF, 0.1255 and 0.1260, which says the gap comes from the accelerator and not the CPU model.

![HVF versus TCG boot time](/images/cuttlefish-mac/fig4.png)

## To actually run Android, set aside 500GB

Up to here the QEMU half has my respect. It builds, it signs, it accelerates, and the numbers hold up.

Then the README tells me the next step is a 500GB case-sensitive volume:

```bash
hdiutil create -type SPARSEBUNDLE -fs HFS+ -fsargs -s -size 500g \
  -volname 'AOSP' "${HOME}/AOSP.sparseimage"
```

500GB, case-sensitive, sparse. AOSP contains filenames that differ only by case, so a normal APFS volume cannot hold them.

This machine has 25GB free. That is off by a factor of twenty. Even with the space, what follows is `repo sync -c -j8` for 274 projects, then sourcing `build/envsetup.sh`, lunching to `aosp_cf_arm64_phone-bp2a-userdebug`, and building `out/host/darwin-x86/cvd-host_package.tar.gz` with m.

Note the output path: `darwin-x86`. This is building a macOS package on Apple silicon and the directory still says x86. Historical baggage, harmless, faintly funny.

The README also has a more honest line: "Application is not proof of a successful build or boot." Patches applied does not mean it built. Built does not mean it boots.

![500GB against 25GB](/images/cuttlefish-mac/fig5.png)

## How it compares to the emulator already on my machine

I have Android emulator 36.6.11.0 installed, and the first line of its startup banner reads: "This program is a derivative of the QEMU CPU emulator."

Both sides are QEMU. The difference is that Goldfish only emulates enough to run apps, while Cuttlefish has to emulate Bluetooth, Wi-Fi, cellular, graphics and the whole OTA chain. The former comes free with Android Studio. The latter needs 500GB and a build chain nobody has validated end to end.

Meta's post says they want to "contribute these changes upstream to QEMU and AOSP", and that this is still exploratory. So for now the only route is a fork plus patches.

## When it's worth the trouble

If all you want is to run an Android app and look at the UI, the Android Studio emulator is enough. Do not bother. Goldfish wins on both startup time and ease of use, and it is not close.

If you work at the platform layer, building for non-phone devices — glasses, watches, head units — and what you need to verify is Bluetooth pairing, cellular registration, OTA updates, then Cuttlefish operates in a different class of fidelity. Meta went to this trouble because their developers need to test things on AI glasses, and Goldfish cannot emulate that.

But go in prepared: 500GB of space, a set of Xcode command line tools, the ability to write entitlements and sign a binary yourself, and the willingness to accept that the build may fail with nobody to help (Meta has not run the full flow either).

The 15x speedup is real. So is the 500GB.
