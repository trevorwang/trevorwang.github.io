---
title: "同一段 C 代码，Flutter 和 KMP 各踩五个坑"
date: 2026-11-13T08:11:41+08:00
draft: false
categories: ["公众号"]
tags: ["Kotlin", "Flutter"]
summary: "Flutter 官方博客在 2026 年 11 月 12 日发了篇公告，作者 Mariam Hasnany，说 FFIgen 的 Obj-C 支持和 JNIgen 都转正了。博客原文写道，Pigeon 底层一直在用这两套机制，只是现在才把它们摆到台面上。"
cover:
    image: "/images/interop/cover.png"
    alt: "同一段 C 代码，Flutter 和 KMP 各踩五个坑"
    relative: false
---

Flutter 官方博客在 2026 年 11 月 12 日发了篇公告，作者 Mariam Hasnany，说 FFIgen 的 Obj-C 支持和 JNIgen 都转正了。博客原文写道，Pigeon 底层一直在用这两套机制，只是现在才把它们摆到台面上。

我当天下午就去试。

第一步就卡住：最新版 ffigen 装不进我的工程。

这不是官方说错了什么。转正说的是这套方案成熟了，跟我这台机器上的 Flutter 版本能不能装最新包，是两件事。但这两件事叠在一起，恰好是所有人上手时会遇到的第一堵墙。

## 公告说 stable，装的时候报错了

我本机是 Flutter 3.41.4，Dart 3.11.1。

pubspec 里写 ffigen 23.0.0，pub get 直接失败，报的是一长串依赖推导：ffigen 22 以上依赖 code_assets，code_assets 依赖 hooks，hooks 依赖 record_use，record_use 要 meta 1.19.0 以上，而 Flutter SDK 把 meta 钉死在 1.17.0，六个包串成一条链，断在第四环，报错信息整整一屏，末行才写着降级建议。

链条到这儿就断了。降级建议是装 ffigen 21。

我把版本改成 21，秒过。

pub.dev 上 ffigen 的版本页写着 23.0.0，发布日期 10 月 6 日。能装和最新，是两回事。

所以第一件事不是「怎么用」，是「你得先确认自己能不能装」。

## 那就退一个版本

ffigen 21 能不能覆盖这次公告说的能力，我没法从版本号上判断。官方描述指出的是这套方案的设计取舍，通篇没提最低版本要求。

但我要测的东西不需要 JNIgen，也不需要绑 Obj-C。我要的是一个最基础的场景：一段 C 代码，两边各自绑一遍，看谁更省事。

C 就够。C 是两边唯一没有分歧的交集。

## 先写一段两边都能用的 C

我写了 34 行 C 加 17 行头文件，四个函数，故意挑了四种数据形状。

一个纯计算，传入数组返回和，用来量调用开销。

一个返回结构体，结构体里带一个 64 字节的内联字符数组。这是最容易出事的形状。

一个往调用方给的缓冲区里写字符串。谁分配内存，两边态度不一样。

一个返回 const char 指针。

这四个函数在 C 层先跑通，输出 sum=10、value=-7 negative、filled:42、interop-lab 1.0。后面两边绑完，必须复现同一组值，一个字符都不能差。

![一段 C 代码，两条绑定路径](/images/interop/fig1.png)

## Flutter 这条路：一个 yaml 出全部绑定

Flutter 侧我只写了 15 行配置和 34 行调用代码。

配置就是一个 yaml，告诉 ffigen 头文件在哪、输出到哪个 dart 文件、哪些函数要、哪些结构体要。

跑起来 0.76 秒，生成 457 行 Dart。

然后 dart run，四条调用全部正确。

但这里有个细节值得说。

457 行里，真正跟我的接口有关的只有 23 行。剩下 434 行是 Darwin 系统头的产物，因为我的头文件 include 了 stdint.h，ffigen 一路展开，把 pthread、mbstate_t 这些全生成了出来，连同它们依赖的一大串内部结构。

能读。但没什么用。

![手写行数与生成代码](/images/interop/fig2.png)

## KMP 这条路：四行 def 加一个任务

KMP 侧要三样东西。

一个 def 文件，四行：头文件路径、编译选项、链接选项、包名。

一个 gradle 配置，二十二行：定义 macosArm64 目标、创建 cinterop、指定 def 文件。

还有 commonMain 里四条 expect 声明，以及 macosArm64Main 里对应的四条 actual 实现，四十二行。

cinterop 生成 1.4 秒。链接可执行文件，debug 2.8 秒，release 5.7 秒。

跑起来的结果跟 C 层、跟 Dart 侧完全一致。

四个函数、三套实现、同一组输出。这一步过了，后面的对比才有意义。如果连值都对不上，比构建时间就是瞎比。

## 构建：KMP 慢四倍，产物小十倍

生成绑定，Flutter 0.76 秒，KMP 1.4 秒。

编译到能跑，Dart AOT 1.3 秒，KMP release 5.7 秒。KMP 慢四倍多。

但产物反过来。Dart 编译出的可执行文件 5.3 MB，KMP 的 kexe 只有 532 KB。

十倍。这不是谁优化得好，是运行时要不要整个装进去。Dart 那个二进制里带着整个 Dart 运行时，KMP 只带用到的部分。

KMP 的 debug 版是 1.3 MB，也不大，但那是因为它没优化，不是因为小得漂亮。

![构建耗时与产物体积](/images/interop/fig3.png)

## 运行：普通调用打平，一碰结构体就拉开

各跑两万次取均值。

普通函数调用，Dart 0.095 微秒，KMP 0.102 微秒，差 7%，基本是噪声。结构体返回，Dart 0.146 微秒，KMP 0.349 微秒，差 2.4 倍。

四类调用全套走一遍，Dart 56 微秒，KMP 129 微秒。

启动更快的是 KMP，22 毫秒三次一致。Dart 是 25、25、29。

所以差距不是全局的。它只在特定形状上冒头：你传数组、传指针、拿字符串，两边一样快。你一让它按值返回结构体，Kotlin 那边就要多付一次 CValue 的拷贝。

![release 对照下的调用开销](/images/interop/fig4.png)

## debug 差点让我写错结论

这里差点翻车。

我第一次量 KMP，用的是 debug 构建。结构体返回那一项，9.817 微秒，而 Dart 侧同一项 0.146 微秒，我当时的草稿已经写下「差 67 倍」。

后来想，debug 是不优化的，拿它跟 AOT 比不成立。换成 release 重新量，同一项变成 0.349 微秒。

28 倍的差距，全在优化开关上。

要是当时就那么发出去了，结论会完全反过来，而且看起来特别有说服力。

## 五个坑对五个坑

Flutter 这边五个。

第一，最新版 ffigen 装不上，被 meta 版本钉死，只能退到 21。

第二，配置里的 entry-points 写相对路径，ffigen 不报错，静默生成一个空绑定文件。我一开始看到 229 字节的输出，还以为是自己配置写错了。改成绝对路径才正常。

第三，结构体里的内联 char 数组，ffigen 生成的是 Array 类型，没有 cast 方法，要取字符串只能逐字节读。

第四，ffi 包里的 Char 和 Utf8 是两种扩展类型，互相之间得来回转。

第五，457 行生成代码里 434 行跟你无关。

KMP 这边也是五个。

第一，首次构建时 cinterop 和编译的任务顺序会撞车，第一次编译必然报符号找不到，第二次就好了。

第二，整个文件要 opt-in ExperimentalForeignApi，方法上逐个标不如文件头标一次。

第三，ByteVar、IntVar 在 kotlinx.cinterop 里，不在 platform.posix 里。我第一版从 posix 导入，全红。

第四，按值返回的结构体在 Kotlin/Native 里是 CValue，不能直接点字段，必须 useContents。

第五，链接完运行时报 dyld 找不到 dylib，得给 DYLD_LIBRARY_PATH 或者自己加 rpath。

![两边各五个坑](/images/interop/fig5.png)

## 两边坑的区别在哪

数量一样，性质不一样。

Flutter 那五个里，有两个是静默的：空绑定不报错，版本墙只在 pub get 那一步拦你一次，你写完代码跑不通，得自己去猜哪里空。KMP 那五个全是编译期报错，一个都躲不过去，但也一个都不用猜，报错凶，改起来反而快。这个差别来自两边绑定的时机：一个在运行期找符号，一个在链接期定死。

如果你更怕「程序不报错但结果不对」，Flutter 这边的坑更难受。如果你更怕反复调构建，KMP 这边更折腾。

## 该选哪条

如果目标只有一个平台，而且在 Apple 生态里，KMP 的 cinterop 更省事。绑定在编译期就定死，产物体积小十倍，运行时开销也没输。

但要跨平台，尤其是要落到 Android、Windows、JS 上，cinterop 那条路就断了。它只在 Kotlin/Native 支持的 target 上存在，一到 JVM 就得换成 JNI 那一套。

Flutter 的 ffigen 反而是通用的，同一个 dylib、so、dll，换平台只换加载路径。

这也是我做完之后最想说的一点：这两个东西不是同一层的替代品。cinterop 是 Kotlin/Native 的编译器能力，ffigen 是跨平台的绑定生成器。比快慢没意义，比的是「你的项目形态让哪条路走得通」。

至于那个版本墙，等你哪天想试最新的 ffigen，先看看自己 Flutter 是哪一版，别像我一样先写 pubspec 再发现装不上。
