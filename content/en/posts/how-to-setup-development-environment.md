---
title: "Setting Up Your Flutter Development Environment"
date: 2018-06-03T09:00:00+08:00
draft: false
categories: ["WeChat"]
tags: ["Archive"]
summary: "Setting up the Flutter development environment"
---

## System requirements
OS: macOS (64-bit). This walkthrough uses macOS as the example.  
Tools: `bash`,`git`,`curl`,`unzip`

## Get the Flutter SDK 
1. Download the latest SDK from the [Flutter SDK Archive](https://flutter.io/sdk-archive/#macos) page.
    Click [flutter_macos_v0.4.4-beta.zip](https://storage.googleapis.com/flutter_infra/releases/beta/macos/flutter_macos_v0.4.4-beta.zip) to download, or
    
    ```shell
curl -o ~/Downloads/flutter_macos_v0.4.4-beta.zip https://storage.googleapis.com/flutter_infra/releases/beta/macos/flutter_macos_v0.4.4-beta.zip
```
    
    **P.S. If the links above give you trouble, use the China mirror.**
    
    ```shell 
curl -o ~/Downloads/flutter_macos_v0.4.4-beta.zip  https://storage.flutter-io.cn/flutter_infra/releases/beta/macos/flutter_macos_v0.4.4-beta.zip
```
2. Unzip into whatever path you want

    ```shell
cd ~/development
unzip ~/Downloads/flutter_macos_v0.4.4-beta.zip
```
3. Add `flutter` to PATH

    ```shell
export PATH=`pwd`/flutter/bin:$PATH
```
4. Run flutter doctor

    ```shell
flutter doctor
```

***If the steps above did not get you a working install, use these commands instead:***

```shell
export PUB_HOSTED_URL=https://pub.flutter-io.cn
export FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn
git clone -b dev https://github.com/flutter/flutter.git
export PATH="$PWD/flutter/bin:$PATH"
cd ./flutter
flutter doctor

####alternate mirror sites####
# export FLUTTER_STORAGE_BASE_URL=https://mirrors.sjtug.sjtu.edu.cn/
# export PUB_HOSTED_URL=https://dart-pub.mirrors.sjtug.sjtu.edu.cn/
```

## Set up a development platform
### iOS
#### Install Xcode
1. Install Xcode from the [App Store](https://itunes.apple.com/us/app/xcode/id497799835).
2. Point the command line at the latest Xcode
    
    ```shell
    sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer
    ```
3. Make sure you have accepted the Xcode license agreement

    ```shell
    sudo xcodebuild -license
    ```
    
#### Set up the simulator
1. Launch the simulator from Launchpad, or with the command below

    ```shell
open -a Simulator
```

2. `flutter run` to run your app


### Android
#### Install Android Studio
Download and install [Android Studio](https://developer.android.com/studio/index.html)
#### Android device (Android 4.1 or newer)
1. Follow [this link](https://developer.android.com/studio/debug/dev-options.html) to enable developer mode and turn debugging on
2. Plug your phone into the computer
3. `flutter run` to run your app
