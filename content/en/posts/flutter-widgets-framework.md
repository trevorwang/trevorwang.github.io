---
title: "An Introduction to the Flutter Widgets Framework"
date: 2018-06-11T09:00:00+08:00
draft: false
categories: ["WeChat"]
tags: ["Archive"]
summary: "A quick introduction to the Flutter widgets framework, and how Flutter builds complex UI by composing widgets."
---

## Introduction

Flutter widgets were inspired by [React](http://facebook.github.io/react/), and turned into the reactive framework everyone uses today. The core idea is that all UI is created with widgets. Every widget provides its current configuration and state. When a widget changes, the system computes the smallest change in the render tree behind that widget from the previous state to the current one, and rebuilds accordingly. So in Flutter, all UI is built by composing widgets. **[See all the widgets](https://flutter.io/widgets/)**

## A Hello World
Copy the code below into `lib/main.dart` in a new project and run it to see the result. **(If you have not created a project yet, follow [the steps here](/en/posts/how-to-setup-development-environment/) to create one)**

```dart
import 'package:flutter/material.dart';

void main() {
  runApp(
    new Text(
      'Hello, world!',
      textDirection: TextDirection.ltr,
    ),
  );
}
```

![Hello world](/images/flutter-hello-world-1.png)
In the main function we pass a text widget to `runApp` and get the result above.
What if we want `Hello, world!` centered? As I said earlier, in Flutter we compose widgets. Here we need a widget called `Center`, and whatever component sits inside it shows up in the middle of that component. Let's change the code:

```dart
import 'package:flutter/material.dart';

void main() {
  runApp(
   Center(
      child: Text(
        'Hello, world!',
        textDirection: TextDirection.ltr,
      ),
    ),
  );
}
```
The result:
![Hello world 2](/images/flutter-hello-world-2.png)
*Sharp-eyed readers may have noticed that the `new` keyword on `Center` and `Text` disappeared in this snippet. That is Dart 2 syntactic sugar letting us drop `new`.*

We put `Text` in the `child` property of the `Center` widget, and now the text is centered. Could not be simpler. At this point we have touched two kinds of widgets: `Text`, which displays content, and `Center`, which handles layout. In Flutter development we use this kind of composition heavily to get the effect we want. Next we want a white background and blue text. **[More layout widgets](https://flutter.io/widgets/layout/)**

```dart
import 'package:flutter/material.dart';

void main() {
  runApp(
    Container(
      color: Colors.white, // background color
      child: Center(
        child: Text(
          'Hello, world!',
          textDirection: TextDirection.ltr,
          style: TextStyle(color: Colors.red),  // text color
        ),
      ),
    ),
  );
}
```
![Hello world 3](/images/flutter-hello-world-3.png)
To get a white background we add a `Container` widget and set its color to white. As you can see, Text has no color property but it does have style, so we build a TextStyle to turn the text red. TextStyle can change other font properties too — dig into it yourself.

How do we put a button under the text?

```dart
import 'package:flutter/material.dart';

void main() {
  runApp(
    Container(
      color: Colors.white,
      child: Center(
          child: Column( // vertical layout
        mainAxisAlignment: MainAxisAlignment.center,
        mainAxisSize: MainAxisSize.min,
        children: <Widget>[
          Text(
            'Hello, world!',
            textDirection: TextDirection.ltr,
            style: TextStyle(color: Colors.red),
          ),
          RaisedButton( // button
            child: Text(
              'Say Hello',
              textDirection: TextDirection.ltr,
            ),
            onPressed: null,
          )
        ],
      )),
    ),
  );
}
```
![Hello world 4](/images/flutter-hello-world-4.png)
From the code you can see we added a `Column` to stack multiple widgets vertically, which is why its child property becomes a List called `children`. Then the button: we want a label on it, so we add a `Text` widget for that too. If you have read this far patiently, you basically have a foothold in Flutter development. Even more complex pages are built the same way, by composing widgets — maybe with more complicated widgets.

Notice anything? We did not use an `xml` or `xib` file, and we got this interface built with no effort. If we built the same screen in Java or Swift, how much code would that take?
