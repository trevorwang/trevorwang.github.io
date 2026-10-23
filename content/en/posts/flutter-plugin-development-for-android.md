---
title: "Flutter Plugin Development -- The Android Edition"
date: 2018-06-16T09:00:00+08:00
draft: false
categories: ["WeChat"]
tags: ["Archive"]
summary: "A walkthrough of plugin development using QR code scanning as the example"
---

Flutter is not fully featured yet, and many common third-party libraries have no equivalent in Flutter. So how do you use an existing native library from Flutter? This post walks through Flutter plugin development using QR code scanning as the example.
## Creating a Plugin project
```shell
flutter create --org mingsin.scan --template=plugin  -i swift -a kotlin scan
```
Open the newly created project and you will see this code:

`lib/scan.dart`

```dart
import 'dart:async';

import 'package:flutter/services.dart';

class Scan {
  static const MethodChannel _channel =
      const MethodChannel('scan');

  static Future<String> get platformVersion async {
    final String version = await _channel.invokeMethod('getPlatformVersion');
    return version;
  }
}

```
`android/src/..../ScanPlugin.kt`

```kotlin
class ScanPlugin(): MethodCallHandler {
  companion object {
    @JvmStatic
    fun registerWith(registrar: Registrar): Unit {
      val channel = MethodChannel(registrar.messenger(), "scan")
      channel.setMethodCallHandler(ScanPlugin())
    }
  }

  override fun onMethodCall(call: MethodCall, result: Result): Unit {
    if (call.method.equals("getPlatformVersion")) {
      result.success("Android ${android.os.Build.VERSION.RELEASE}")
    } else {
      result.notImplemented()
    }
  }
}
```
From that code you can see clearly that Flutter does cross-language calls by providing a `MethodChannel`.
In the Kotlin code there is a class called `ScanPlugin` that inherits from `MethodCallHandler`. It first registers a channel called scan with Flutter, then in `onMethodCall` checks whether the method name is `getPlatformVersion` and returns the Android OS version if so.

In Dart, get a `MethodChannel` registered under the name `scan`. Then use `_channel.invokeMethod('getPlatformVersion');` to get the result of the corresponding function.

From that analysis you can already see that adding a function called `codescan` is all it takes.

## Adding the `codescan` function
### Run this command first

    ```shell
    cd example; 
    flutter build apk
    ```
### Import the Android Plugin project

1. Open Android Studio 
2. Choose Import project, then select the `example/android/build.gradle` file
3. In the Gradle Sync window, choose OK
4. In the Android Gradle Plugin Update window, choose Don’t remind me again for this project

Now you can run the example project.


### Add the function in `lib/scan.dart`
```dart
  static Future codeScan() async {
    await _channel.invokeMethod('codeScan');
  }
```


### Implement it in `ScanPlugin.kt`

```kotlin
class ScanPlugin(private val registrar: Registrar) : MethodCallHandler {
    companion object {
        @JvmStatic
        fun registerWith(registrar: Registrar) {
            val channel = MethodChannel(registrar.messenger(), "scan")
            channel.setMethodCallHandler(ScanPlugin(registrar))
        }
    }

    override fun onMethodCall(call: MethodCall, result: Result) {
        when (call.method) {
            "getPlatformVersion" -> {
                result.success("Android ${android.os.Build.VERSION.RELEASE}")
            }
            "codeScan" -> {
                IntentIntegrator(registrar.activity())
                        .setDesiredBarcodeFormats(IntentIntegrator.ONE_D_CODE_TYPES)
                        .initiateScan()
            }
            else ->
                result.notImplemented()
        }

    }
}
```

### Call it from `example/lib/main.dart`

```diff
         body: new Center(
-          child: new Text('Running on: $_platformVersion\n'),
+          child: RaisedButton(
+            onPressed: () async {
+              await Scan.codeScan();
+            },
+            child: Text("Scan"),
+          ),
         ),
       ),
```

Reload the project and you should see this screen:
![flutter code scan](/images/flutter-scan-button.png)
Tap the Scan button and the QR code scanning screen appears.

 ![flutter scan camera](/images/flutter-scan-camera.png)

At this point you can call a third-party library from Dart to do real work.

How do you get a return value back from the third-party library? MethodHandler provides a result class to pass values back, and you only need to pass what you need in the success function. If you also need the return value from onActivity, you can do it with PluginRegistry.ActivityResultListener.

```kotlin
    override fun onMethodCall(call: MethodCall, result: Result) {
        when (call.method) {
            "getPlatformVersion" -> {
                result.success("Android ${android.os.Build.VERSION.RELEASE}")
            }
            "codeScan" -> {
                if (listener.result == null) {
                    registrar.addActivityResultListener(listener)
                }
                listener.result = result
                IntentIntegrator(registrar.activity())
                        .setDesiredBarcodeFormats(IntentIntegrator.ONE_D_CODE_TYPES)
                        .initiateScan()
            }
            else ->
                result.notImplemented()
        }

    }

    private val listener = object : PluginRegistry.ActivityResultListener {
        var result: Result? = null
        override fun onActivityResult(requestCode: Int, resultCode: Int, intent: Intent?): Boolean {
            val intentResult = IntentIntegrator.parseActivityResult(requestCode, intent)
            result?.success(intentResult.contents)
            return true
        }

    }
```
