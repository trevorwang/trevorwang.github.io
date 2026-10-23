---
title: "Flutter Project Structure"
date: 2018-06-09T09:00:00+08:00
draft: false
categories: ["WeChat"]
tags: ["Archive"]
summary: "The project directory layout, and what the main files contain"
---

# Flutter Project Structure
## Main directories and files
![](/images/pic-flutter-project-structure.png)

1. `android` Android platform code
2. `ios` iOS platform code
3. `build` produced by the build, ignore it
4. `lib` cross-platform code, the directory you actually care about in a Flutter project
5. `test` test code
6. `pubspec.yaml` the project manifest, equivalent to `package.json` in a Node.js project. It holds the project description and the libraries it depends on


## The main files

### `pubspec.yaml` 
![](/images/pic-flutter-pubspec.png)


### `main.dart`

Flutter is written in `dart` (later chapters will add more on dart). The image below shows the code after simplification. Three classes are enough for one app screen, and it runs on both iOS and Android at the same time. That simple. And `main.dart` is the entry file for every Flutter project: it cannot be deleted or renamed.
![pic-flutter-main](/images/pic-flutter-main.png)

