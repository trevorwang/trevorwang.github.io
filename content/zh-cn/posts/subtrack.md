---
title: "从一个订阅账本开始，用 CMP 跑通 Android、iOS 和鸿蒙"
date: 2026-11-19T15:30:00+08:00
draft: false
categories: ["公众号"]
tags: ["Kotlin", "Android", "鸿蒙"]
summary: "同一份订阅账本，三端界面和计算共享，各平台只接自己的存储和系统能力。KMP 共享模型，CMP 共享界面。"
cover:
    image: "/images/subtrack/cover.png"
    alt: "从一个订阅账本开始，用 CMP 跑通 Android、iOS 和鸿蒙"
    relative: false
---

做一个订阅管理应用，第一版很容易想到三个页面：把正在付费的服务列出来，提醒即将结束的试用，再算一算接下来要花多少钱。随着页面增加，日期选择、金额格式、浅色与深色主题也会不断调整。把这些改动同步到三个平台，就成了持续的开发工作。

Subtrack 用 Kotlin Multiplatform 共享订阅模型和账本逻辑，用 Compose Multiplatform（CMP）共享界面，再由 Android、iOS 和鸿蒙各自接入存储、系统提醒与应用生命周期。这篇文章以这份工程为例，从准备环境、认识目录开始，逐步运行三端应用，并用同一份虚构账本对照实际效果。

先看跑出来的概览。三张图片均来自实际运行的模拟器，使用同样的六条订阅和八条付款记录。

![Android、iOS、鸿蒙三端的订阅概览，从左到右](/images/subtrack/overview.png)

图中未来 30 天预计扣款为人民币 201.00 元及美元 9.99 元，历史实付为人民币 123.00 元及美元 19.98 元。两种币种分别累计，方便核对数据在三个平台上是否一致。本文的金额、服务名称和付款记录均为演示数据。

## 先固定一次能复现的版本

跨端项目的第一次构建，最容易花时间的地方是环境。Subtrack 的鸿蒙部分使用 CPF-KMP-CMP 提供的适配版本。先保持仓库中的版本组合，等三个目标都运行起来，再单独尝试升级。

本文截图于 2026 年 11 月 19 日采集，应用源码基线为 `06cad3d`，所在分支为 `codex/subtrack-cmp-example`。鸿蒙截图使用临时测试数据加载，加载代码在拍摄后恢复。完整图片尺寸、文件校验和与采集方式见截图记录。

| 项目 | 本文工程或实际环境 |
| --- | --- |
| Kotlin | 2.2.21-1.2.0 |
| Compose Multiplatform | 1.9.2-1.2.0 |
| 鸿蒙 OHPM Compose 包 | 1.9.2-1.2.0 |
| OHOS Skiko | 0.9.22.2-1.2.0 |
| Gradle Wrapper / Android Gradle Plugin | 8.9 / 8.6.0 |
| JDK | 21.0.12.1 |
| Android compile / target / min SDK | 36 / 36 / 24 |
| Xcode / iOS 最低部署版本 | 27.2 / 18.2 |
| DevEco Studio / 本机 native SDK | 26.0.0.821 / 26.0.0.105 |
| deveco-cli / Node.js | 1.3.0-stable / 26.11.1 |

截图设备分别为 Pixel 9 Pro、iPhone 18 Pro 和 Mate 70 Pro 模拟器，对应 Android 16（API 36）、iOS 27.2（24B8117）和 HarmonyOS 6.1.0（API 23）。本机安装的 SDK 与截图设备的系统版本分列记录，便于复现时选择目标。

Android 应用版本为 1.0（versionCode 1），iOS 为 1.0（build 1），鸿蒙为 1.0.0（versionCode 1000000）。它们来自各平台的构建配置，界面代码对应同一个源码基线。

下面是本次设备的软件版本页面，可以和上面的版本表一起核对。

![三端的系统版本页面，从左到右](/images/subtrack/system.png)


## 认清共享界面从哪里进入

取得配套工程后，先在根目录找到 `composeApp`、`iosApp` 和 `harmonyApp`。我们后面会分别用 Android Studio、Xcode 和 DevEco Studio 打开相应部分。

```text
subtrack/
├── composeApp/
│   └── src/
│       ├── commonMain/   共享界面、账本、金额与日期计算
│       ├── androidMain/  Android 入口、SQLite、通知
│       ├── iosMain/      iOS 控制器、SQLite、Keychain
│       └── ohosMain/     鸿蒙 Compose 控制器与平台桥接
├── iosApp/               Xcode 宿主
├── harmonyApp/           ArkTS 宿主、RDB 与日历接入
└── supabase/             可选同步后端的迁移与测试
```

共享部分的核心是 `App.kt` 和 `SubscriptionBook.kt`。前者组合概览、订阅列表、支出和设置页面，后者描述订阅、实付记录、计费周期与状态。对同一条订阅的金额展示、日期推进和未来扣款计算，三个平台都调用这些共享代码。

![共享界面与三个平台的入口、系统能力对应关系](/images/subtrack/arch.png)

到了系统能力这一层，工程保留了各平台的适配。Android 使用 SQLiteOpenHelper，iOS 调用系统 SQLite，鸿蒙由 ArkTS 使用 ArkData RDB。理解这条分界后，修改共享布局时去 commonMain，排查系统日历授权或宿主启动问题时去对应平台目录，定位会明确很多。

## 第一步，准备工具和本机路径

本文用 macOS 演示三端。iOS 的构建依赖 Xcode，所以准备一台能运行 Xcode 的 Mac，可以在同一台机器上完成三端对照。

先安装 JDK 21，在 Android Studio 的 Gradle 设置中选择同一份 JDK。终端执行下面的命令确认环境，并始终使用仓库附带的 Wrapper。

终端跑 `java -version` 和 `./gradlew --version` 确认环境，并始终使用仓库附带的 Wrapper。

接着安装 Android SDK Platform 36。通过 Android Studio 的 SDK Manager 可以找到 SDK 的安装位置。在项目根目录创建本机配置文件 `local.properties`，填入实际路径。

文件里写两行：`sdk.dir=/path/to/Android/sdk`，以及 `local.deveco.path=/Applications/DevEco-Studio.app`。

这里两个值都需要和自己的安装位置对应。该文件已被 Git 忽略，适合保存机器相关的路径。路径配置完成后再进行 Gradle Sync，遇到 SDK location 相关错误时先回到这里核对。

鸿蒙还要准备 DevEco Studio 和 HarmonyOS SDK。随后在 Android Studio 中通过 Install Plugin from Disk 安装 CPF 的 KMP 工具链插件，重启并同步工程。

插件下载入口、配套版本及界面步骤以 CPF-KMP-CMP 官方的快速开始文档为准，项目内还带一份鸿蒙环境说明。

Gradle 脚本同时声明了三端目标，因此建议第一次同步前就准备好鸿蒙环境。这样后面切换目标时，可以沿用同一份工程配置。

## 第二步，先运行 Android 并添加一条记录

在 Android Studio 中打开项目根目录，等待 Gradle Sync 完成。创建或启动 Android 模拟器，选择 `composeApp` 运行配置，再点击 Run。也可以从终端先构建 APK。

从终端构建的话，先跑 `./gradlew :composeApp:assembleDebug`，再用 `adb devices` 拿到设备序列号，然后 `adb -s <设备序列号> install -r` 装 `composeApp/build/outputs/apk/debug/composeApp-debug.apk`。

这里的设备序列号使用 `adb devices` 的结果。连接多台设备时明确指定 `-s`，可以避免把调试包安装到第二台设备。生成的 APK 位于 `composeApp/build/outputs/apk/debug/`。

新安装的应用从空账本开始。点击右上角加号，填写服务名称、日期、计费周期和金额，再保存。先用“云盘空间”这条记录试一次，金额填 18.00，选择人民币和每月续费。保存后回到列表，打开详情，再重启应用查看记录是否仍在，可以同时检查录入、导航与本地持久化。

![Android 上的订阅列表、试用详情与编辑页面，从左到右](/images/subtrack/android-flow.png)

正式录入金额时，界面展示的小数金额会转换成币种的最小单位。例如 18.00 元对应 1800 分，美元 9.99 对应 999 美分。项目用整数保存这些值，汇总时也按币种分开处理。测试数据文件里的 `amount` 因此使用整数，和编辑框中的小数形式需要对应起来。

## 第三步，让同一套界面进入 iOS

用 Xcode 打开 `iosApp/iosApp.xcodeproj`，选择 `iosApp` scheme 和一个符合最低部署版本的 iPhone 模拟器，执行 Run。

这个工程通过 SwiftUI 承载 Compose 控制器。当前 `ContentView.swift` 中的桥接代码如下，控制器由 Kotlin 导出。

```swift
struct ComposeView: UIViewControllerRepresentable {
    func makeUIViewController(context: Context) -> UIViewController {
        MainViewControllerKt.MainViewController()
    }

    func updateUIViewController(
        _ uiViewController: UIViewController,
        context: Context
    ) {}
}
```

Xcode 的构建阶段会调用 `:composeApp:embedAndSignAppleFrameworkForXcode`，把共享 Kotlin 框架接进宿主。

第一次运行时，终端能找到 JDK、Gradle 能取得依赖，是这一阶段顺利完成的前提。排查错误时可以展开 Compile Kotlin Framework 的日志，找到真正失败的 Gradle 任务。

模拟器跑通以后，再准备真机签名。把自己的团队和可用的 Bundle ID 放进 Git 忽略的 `iosApp/Configuration/Local.xcconfig`，然后在 Xcode 中确认 Signing 配置。

文件里两行：`TEAM_ID=YOUR_APPLE_TEAM_ID` 和 `PRODUCT_BUNDLE_IDENTIFIER=com.yourcompany.subtrack`。

团队和 Bundle ID 会影响设备安装、签名及 Keychain 使用。文章的账本截图来自离线页面。云端登录、钥匙串和后台任务还需要各自的验证步骤，项目的项目的验证记录保留了当前测试结果。

![iOS 上的同一份账本：列表、详情与编辑页面，从左到右](/images/subtrack/ios-flow.png)

对照 Android，可以检查卡片状态、金额、日期和操作文案。两端由相同的 Composable 生成这些内容，系统状态栏、安全区域和宿主行为则来自各自的平台环境。截图保留了各自的完整屏幕，便于观察这些差异。

## 第四步，生成鸿蒙共享库，再构建 HAP

鸿蒙这条链路先由 Gradle 编译 Kotlin，再由 DevEco 构建 ArkTS 宿主。在仓库根目录运行共享库发布任务。

在仓库根目录跑 `./gradlew :composeApp:publishDebugBinariesToHarmonyApp`。

任务会编译 arm64 和 x64 两个目标，把 `libkn.so`、对应的 `libkn_api.h` 以及 Compose 资源复制到 harmonyApp。CMake 接着把 `libkn.so` 链接进原生入口，ArkTS 的 `Compose` 组件通过控制器承载共享页面。

这也解释了为什么应用需要这些 .so 文件。它们是 Kotlin/Native 的编译产物，打包阶段必须存在。仓库已经忽略新生成的 .so，开发者可以通过上述任务从源码生成。共享 Kotlin 代码变化后再次执行发布任务，可以让鸿蒙宿主使用本次修改对应的库和头文件。

共享库准备好后，用 DevEco Studio 打开 `harmonyApp` 目录，完成 OHPM 依赖安装，并构建 `entry` 模块。使用命令行时，需要 Node.js 22 或以上及本文采用的 CLI。

命令行先 `npm install -g @deveco/deveco-cli@1.3.0-stable` 装 CLI，进 `harmonyApp` 目录跑 `ohpm install`，再 `devecocli build --modules entry`。

设备安装还需要自己的鸿蒙签名。在 DevEco Studio 中登录开发者账号，为模拟器或设备配置签名，再构建并运行。签名配置会写入 `build-profile.json5`，提交前检查文件差异，把个人证书路径和签名材料保存在本机。

本文运行设备为 Mate 70 Pro 模拟器，系统为 HarmonyOS 6.1.0、API 23。当前宿主的 `Index.ets` 使用 `libraryName: 'entry'`，它和 native 入口模块名对应。排查启动崩溃时，可以沿着“共享库是否更新、头文件是否匹配、native 初始化是否完成、ArkTS 控制器是否创建”这条顺序检查。

![鸿蒙上的同一组页面，从左到右](/images/subtrack/harmony-flow.png)

## 第五步，用同一份账本检查三端行为

空页面能帮助确认应用启动。加入不同状态的数据后，还能检查排序、标签、金额、累计和关闭记录的展示。本文使用下面六条虚构订阅，并额外录入八条付款。完整数据保存在演示数据文件，金额字段采用币种最小单位。

| 服务 | 状态 | 预计日期 | 金额 | 周期 |
| --- | --- | --- | --- | --- |
| 云盘空间 | 使用中 | 2026-11-24 | CNY 18.00 | 每月 |
| 视频会员 | 使用中 | 2026-11-22 | CNY 25.00 | 每月 |
| 写作助手 | 使用中 | 2026-11-28 | USD 9.99 | 每月 |
| 在线课程 | 免费试用 | 2026-11-20 | CNY 59.00 | 每月 |
| 健身课程 | 计划取消 | 2026-11-21 | CNY 99.00 | 每月 |
| 旧版音乐会员 | 已关闭 | 2026-11-05 | CNY 12.00 | 每月 |

按照正常录入流程，可以先创建三条使用中的记录，再打开“免费试用”开关创建在线课程，接着用详情页的取消和关闭操作建立其余两种状态。支出页面可以通过右上角加号补录付款。为每条付款选择订阅、实付日期、币种和金额后，观察概览与按服务统计是否同步更新。

截图以 2026-11-19 为观察日期，所以在线课程显示“明天扣款”，健身课程显示“后天扣款”。过几天再复现时，这些相对日期会自然变化。核对数据时以记录中的绝对日期为准。

支出页累计的是已经录入的付款。订阅的预计金额服务于未来扣款提示，两者各有数据来源。试用或计划取消的记录仍保留预计扣款日期，用户完成相应操作后再更新状态，因此截图中的未来预测也会包含这些记录。

![三端的支出统计，从左到右](/images/subtrack/spending.png)

为了批量准备截图，附带的 Python 工具会在新目录里生成测试数据库和文本账本。

跑 `python3 docs/articles/cmp-subtrack/tools/prepare_demo_ledger.py --output /tmp/subtrack-article-demo`，它会在新目录里生成测试数据库和文本账本。

该工具只向指定的新目录写文件。日常体验可以按表格手动录入。本文采集时，Android 使用只读模拟器会话，iOS 使用专用模拟器，鸿蒙则先备份账本，再通过临时数据加载版本写入虚构记录。拍摄完成后恢复原账本和源码。这样既能得到内容一致的截图，也能保留已有开发数据。

## 第六步，再接入系统能力与实验性同步

三端的设置页提供账号、外观、提醒和本机数据入口。第一次体验时，可以先完成本地账本的增加、编辑、重启和支出核对，再检查提醒权限与后台执行。

![三端的设置页面，从左到右](/images/subtrack/settings.png)

Subtrack 默认离线可用，构建步骤不要求注册账号。

需要体验同步时，把 `supabase.local.properties.example` 复制成 `supabase.local.properties`，填写自己的项目地址和 publishable key，并按文件名顺序执行 SQL 迁移。邮箱验证码模板、Google OAuth 回调和后端权限的具体配置见同步说明文档。

这些客户端配置会进入应用安装包，服务端密钥和 SMTP 凭据应配置在服务端。鸿蒙修改同步配置后还需要重新发布共享库和配置文件，再构建 HAP。

当前账号页已标记云同步为实验性功能。跨设备往返、断网重试、冲突处理、账号切换和三端系统后台调度仍需设备验收。本文配图展示本地账本的实际界面，并使用本地测试数据核对展示结果。

已有构建检查中，Android 的 34 项单元测试通过，iOS 独立测试通过 32 项，Keychain 测试仍报 `-25291`。图中的离线页面已经在实际模拟器上运行。继续完善账号功能时，可以沿用这些测试，再补充有签名应用宿主和真实账号的验证记录。

## 第七步，保留一套可重复检查的流程

第一次把三端运行起来以后，可以固定一个小的回归顺序。打开概览，核对预计扣款和历史实付；进入列表，打开试用详情；编辑一条记录并重启；补录付款，回到支出页检查累计。每次修改共享界面后，按这个顺序在三个平台都走一遍。

构建和现有测试从项目根目录执行。

```bash
./gradlew :composeApp:testDebugUnitTest
./gradlew :composeApp:assembleDebug
./gradlew :composeApp:iosSimulatorArm64Test
./gradlew :composeApp:publishDebugBinariesToHarmonyApp
```

iOS 测试任务需要可用的 Apple Silicon 模拟器环境。鸿蒙 HAP 则在共享库发布后，进入 harmonyApp 继续构建。保存每轮对应的源码提交、依赖版本、设备系统和截图，可以让后续问题有明确的比较依据。

这个订阅账本把共享界面、金额与日期逻辑，以及三种原生宿主放在了一个可以逐步运行的工程里。继续开发时，可以先在 commonMain 完成一条完整的业务路径，再分别验证持久化、授权和生命周期。每一轮都用相同测试数据走通三端，跨端开发的收益才会随着功能增长持续积累。

