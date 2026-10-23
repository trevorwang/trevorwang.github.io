---
title: "Using JaCoCo to Check Test Coverage"
date: 2019-06-20T09:00:00+08:00
draft: false
categories: ["WeChat"]
tags: ["Testing", "JaCoCo", "Unit Test", "Manual Testing"]
summary: "Enabling jacoco during manual testing to check test coverage"
---

​    The team had no automated test coverage at the time, so the QA team wanted a sense of what manual testing actually covered. That is how this post came about. Plenty of articles online use Android's instrumentation test framework and start the app from a command. The report is also generated after the launched activity finishes, which in complex scenarios cannot capture function calls on every screen.

   The approach here overrides the Application code under a new `build type`, used only during manual testing, with no impact on existing code. Hope it helps. Sample code for this post is available here: [trevorwang/jacoco-manual-coverage](https://github.com/trevorwang/jacoco-manual-coverage)

1. In your project's `buildscripts` directory, create a `jacoco.gradle` file and add the following code

   ```groovy
   apply plugin: 'jacoco'  // enable jacoco
   
   jacoco {	
       toolVersion = "0.8.3"  // set the jacoco version
   }
   
   // If you use Robolectric, you must add the following
   tasks.withType(Test) {
       jacoco.includeNoLocationClasses = true
   }
   ```

2. In the `build.gradle` under the app directory, add code to apply the script

   ```groovy
   apply from: '../buildscripts/jacoco.gradle'
   ```

3. After running `testDebugUnitTest`, you will see `testDebugUnitTest.exec` under `app/build/jococo/`. Remember that file — we will use it later to generate the report.

   ![jacoco-executedata](/images/jacoco-executedata.png)

4. Create a `jacoco` task

   The Android gradle plugin generates different `variant`s, so we need a different task per `variant` to generate reports.

   ```groovy
   project.afterEvaluate {
       android.applicationVariants.all { variant ->
           def variantName = variant.name
           def testTaskName = "test${variantName.capitalize()}UnitTest"
           tasks.create(name: "${testTaskName}Coverage", type: JacocoReport, dependsOn: "$testTaskName") {
             // TODO implement later
           }
       }
   }
   ```

   After clicking `Sync Gradle`, the gradle tasks gain two entries, ` testDebugUnitTestCoverage`, `testReleaseUnitTestCoverage`. Next we add the implementation that generates the report.

5. Replace the `TODO` from the previous step with this code

   ```groovy
    group = "Reporting"
      description = "Generate Jacoco coverage reports for the ${variantName.capitalize()} build."
    	// set report formats
      reports {
          html.enabled = true
          xml.enabled = true
      }
    
  	 // exclude classes we do not want counted
      def excludes = [
              '**/R.class',
              '**/R$*.class',
              '**/BuildConfig.*',
              '**/Manifest*.*',
              '**/*Test*.*',
              'android/**/*.*',
        	  'androidx/**/*.*'
      ]
  	  // Java class files
      def javaClasses = fileTree(dir: variant.javaCompiler.destinationDir, excludes: excludes)
  	  // Kotlin files
      def kotlinClasses = fileTree(dir: "${buildDir}/tmp/kotlin-classes/${variantName}", excludes: excludes)
      classDirectories = files([javaClasses, kotlinClasses])
  	  // source files
      sourceDirectories = files([
              "$project.projectDir/src/main/java",
              "$project.projectDir/src/${variantName}/java",
              "$project.projectDir/src/main/kotlin",
              "$project.projectDir/src/${variantName}/kotlin"
      ])
    	// the file we generated at the beginning
      executionData = files("${project.buildDir}/jacoco/${testTaskName}.exec")
   ```

6. Run the `testDebugUnitTestCoverage` task and the report shows up in the build directory

   ![](/images/jacoco-reports.png)

    With that, we have finished generating a jacoco report.

Now the key step: how do you enable jacoco in the packaged app?

7. Create a new `staging` build type

   ```groovy
       buildTypes {
           release {...}
   
           staging {
               initWith(debug)
               matchingFallbacks = ["debug"]
               testCoverageEnabled true    // packages the jacoco runtime into the app
           }
       }
   ```

   

8. Under `src`, at the same level as `main`, create a `staging` directory

9. Under `staging`, create a `java` directory, and add a `StagingApp.kt` file in the `com.example.staging` package, with this code:

   ```kotlin
   package com.example.staging
   
   import android.Manifest
   import android.app.Activity
   import android.app.Application
   import android.os.Bundle
   import android.os.Environment
   import android.util.Log
   import android.widget.Toast
   import androidx.fragment.app.FragmentActivity
   import com.tbruyelle.rxpermissions2.RxPermissions
   import java.io.File
   import java.io.FileOutputStream
   import java.io.IOException
   
   
   class StagingApp : Application() {
       override fun onCreate() {
           super.onCreate()
           Log.d(TAG, "StagingApp")
           registerActivityLifecycleCallbacks(object : ActivityLifecycleCallbacks {
               var activitySize = 0
               override fun onActivityPaused(activity: Activity?) {
               }
   
               override fun onActivityResumed(activity: Activity?) {
                   // ask the first activity for SD card directory access
                   if (activitySize == 1) {
                       (activity as? FragmentActivity)?.let {
                           val rxPerm = RxPermissions(it)
                           rxPerm.request(Manifest.permission.WRITE_EXTERNAL_STORAGE).subscribe({ result ->
                               if (!result) {
                                   Toast.makeText(
                                       it,
                                       "You have to grant the permission to save coverage file",
                                       Toast.LENGTH_SHORT
                                   ).show()
                               }
                           }, { e ->
                               e.printStackTrace()
                           })
                       }
                   }
               }
   
               override fun onActivityStarted(activity: Activity?) {
               }
   
               override fun onActivityDestroyed(activity: Activity?) {
                   activitySize -= 1
   
                   if (activitySize <= 0) {
                     //once all activities are destroyed, write the report file
                       generateCoverageReport(createFile())
                   }
               }
   
               override fun onActivitySaveInstanceState(activity: Activity?, outState: Bundle?) {
               }
   
               override fun onActivityStopped(activity: Activity?) {
               }
   
               override fun onActivityCreated(activity: Activity?, savedInstanceState: Bundle?) {
                   activitySize += 1
               }
   
           })
   
       }
   
   
       private fun generateCoverageReport(file: File) {
           Log.d(TAG, "generateCoverageReport():${file.absolutePath}")
           FileOutputStream(file, false).use {
               val agent = Class.forName("org.jacoco.agent.rt.RT")
                   .getMethod("getAgent")
                   .invoke(null)
   
   
               Log.d(TAG, agent.toString())
               it.write(
                   agent.javaClass.getMethod("getExecutionData", Boolean::class.javaPrimitiveType)
                       .invoke(agent, false) as ByteArray
               )
           }
       }
   
       fun createFile(): File {
           // under the SD card
           val file = File(Environment.getExternalStorageDirectory(), "jacoco/$DEFAULT_COVERAGE_FILE_PATH")
           if (!file.exists()) {
               try {
                   file.parentFile?.mkdirs()
                   file.createNewFile()
               } catch (e: IOException) {
                   Log.d(TAG, "Exception : $e")
                   e.printStackTrace()
               }
           }
           return file
       }
   
       companion object {
           const val DEFAULT_COVERAGE_FILE_PATH = "jacoco-coverage.ec"
           const val TAG = "StagingApp"
       }
   }
   ```

   

10. Create an `AndroidManifest.xml` file in the `staging` directory with this content

    ```xml
    <?xml version="1.0" encoding="utf-8"?>
    <manifest
            xmlns:android="http://schemas.android.com/apk/res/android" package="com.example.jacocomanual">
        <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
        <application android:name="com.example.staging.StagingApp"/>
    
    </manifest>
    ```

    The result looks like this:

    ![jacoco-staging-build-type](/images/jacoco-staging-build-type.png)

11. In the IDE, select `staging` under `Build Variants`

    ![jacoco-build-variants](/images/jacoco-build-variants.png)

12. Run the app and install it on a device or emulator, click around a bit, then press back to close every screen. This generates a `jacoco/jacoco-coverage.ec` file under the SD card directory.

    ![jacoco-sd-location](/images/jacoco-sd-location.png)

13. Copy `jacoco-coverage.ec` into the `jacoco` folder at the project root

14. Now modify the jacoco task to generate the final report

    ```groovy
                // the file we generated at the beginning
                executionData = files([
                        "${project.buildDir}/jacoco/${testTaskName}.exec",
                        "${rootDir}/jacoco/jacoco-coverage.ec"   // add a second data source
                ])
    ```

14. Run `testStagingUnitTest` and the report is there

    ![jacoco-report-html](/images/jacoco-report-html.png)

Good stuff, honestly.~~
