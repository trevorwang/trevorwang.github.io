---
title: "flutter_boost Integration on Android: What It Took"
date: 2019-04-10T09:00:00+08:00
draft: false
categories: ["WeChat"]
tags: ["Archive"]
summary: "flutter_boost integration on Android: what it took"
---

> How to add flutter_boost to an existing Android project. This demo was done on macOS. On Windows, change the paths to Windows format.

## The Flutter side

1. Create a new flutter module

	```sh
	flutter create -t module flutter_module
	```
2. Add the flutter_boost dependency in `pubspec.yaml`
	
	```yaml
	dependencies:
  	  flutter_boost:
	```
3. Run `flutter build apk`
	With flutter_boost 0.400 this throws an error. The key part is:
	
	```
	* What went wrong:
	A problem occurred evaluating project ':flutter_boost'.
	> Plugin with id 'kotlin-android-extensions' not found.

	```
	It's not hard to see it's a missing `kotlin` dependency. Let's fix that.

4. From the project root run `flutter make-host-app-editable`
   When it finishes you'll see `android` and `ios` folders appear.
5. Add the `kotlin` dependency in `android/build.gralde`

	```diff
	 buildscript {
	+    ext.kotlin_version = '1.3.21'
		repositories {
			google()
			jcenter()
	@@ -8,6 +9,7 @@ buildscript {

		dependencies {
			classpath 'com.android.tools.build:gradle:3.2.1'
	+        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
		}
	}
	```
	Run `flutter build apk` again and it succeeds.
6. Copy the files under `flutter_boost/example/lib` into the `lib` directory

That's the Flutter side prepared.

## The Android side

1. Create a project with Android Studio
  
   If you already have a project you can skip this step

2. Add kotlin support to the App module, and add the following to `settings.gradle`

	```groovy
	setBinding(new Binding([gradle: this]))
	evaluate(new File('../flutter_module/.android/include_flutter.groovy'))   // keep the Android project and the flutter module in the same directory
	```
   Run `gralde sync`. You should now see three new projects in your build: `Flutter-flutter`, `flutter_boost` and `xservice_kit`

   If you hit the error below, change the gradle version to **4.10.2**

   ```
   ERROR: Unable to resolve dependency for ':flutter@dynamicProfile/compileClasspath': Could not resolve project :xservice_kit.
	Show Details
	Affected Modules: flutter

	ERROR: Unable to resolve dependency for ':flutter@dynamicProfile/compileClasspath': Could not resolve project :flutter_boost.
	Show Details
	Affected Modules: flutter
   ```

3. Add the dependencies below to the `build.gradle` file in the `flutterservice` directory

	```groovy
    implementation project(path: ':flutter')
    implementation project(":flutter_boost")
	```
4. Copy everything from `flutter_boost/example/android` and create a new `MyApplication` class:

	```kotlin
	class MyApplication : FlutterApplication(), IPlatform {
		override fun onCreate() {
			super.onCreate()
			FlutterBoostPlugin.init(this)
		}
		override fun getApplication(): Application {
			return this
		}

		override fun getMainActivity(): Activity? {
			return MainActivity.sRef.get()
		}

		override fun isDebug(): Boolean {
			return true
		}

		override fun startActivity(context: Context?, url: String?, requestCode: Int): Boolean {
			return false
		}

		override fun getSettings(): MutableMap<Any?, Any?> {
			return HashMap()
		}

	}
	```
	Then point the manifest at this Application
5. Add a weak reference in MainActivity to hold `MainActivity`

	```kotlin
	class MainActivity : AppCompatActivity() {
		override fun onCreate(savedInstanceState: Bundle?) {
			super.onCreate(savedInstanceState)
			setContentView(R.layout.activity_main)
			sRef = WeakReference(this)
		}

		companion object {

			var sRef: WeakReference<Activity?> = WeakReference(null)
		}
	}

	```
6. Create FlutterPageActivity, and remember to add it to the manifest

	```kotlin
	class FlutterPageActivity:BoostFlutterActivity() {
		override fun getContainerParams(): MutableMap<Any?, Any?> {
			return HashMap()
		}

		override fun onRegisterPlugins(registry: PluginRegistry?) {
			GeneratedPluginRegistrant.registerWith(registry)
		}

		override fun getContainerName(): String {
			return "flutterPage"
		}
	}
	```
7. Give the textview an id in the layout

	```xml
            android:text="Hello World!"
            android:id="@+id/tvHello"
            app:layout_constraintBottom_toBottomOf="parent"
	```
8. Add a click listener in MainActivity

	```kotlin
	tvHello.setOnClickListener {
		startActivity(Intent(applicationContext, FlutterPageActivity::class.java))
	}
	```
9. Run the Android project. If you hit errors, fix them following the prompts.
10. Click `Hello world` and a Flutter page opens. At this point you've gone from a native page to a Flutter page.

11. Change the startActivity function in MyApplication to the code below:

```
 override fun startActivity(context: Context?, url: String?, requestCode: Int): Boolean {
        if (url?.contains("sample://nativePage") == true) {
            context?.startActivity(Intent(context, MainActivity::class.java))
            return true
        }
        return false
    }
```
Click `open native page` in the Flutter page and the activity reopens.

That's the round trip between both sides done. Welcome to the Flutter pit.
