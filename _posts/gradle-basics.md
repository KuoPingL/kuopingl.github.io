---
layout: post
title: "Lesson 1 - What is a Gradle ?"
date: 11/07/23 21:22:20  +0800
categories: [The World of Gradle]
keywords: gradle
---

# 序言

我相信很多人寫了幾年的 Android 卻都不知道 Gradle 到底是什麼。 所以我想借這次機會講解也順便自學什麼是 Gradle。

Gradle 其實是一個 **自動化的工具** 。 通過他，我們可以進行 **編譯、測試、檢查程式碼、打包** 等工作

接下來就讓我們來看看這個神奇的 Gradle 吧。

# build.gradle

想要暸解 Gradle 就從我們時常看到的 build.gradle 著手就是最好選擇了。

我們會先從舊版說起，之後會講解與新版的差異。 希望這樣大家就不會看到 Gradle 時只是覺得「啊官方就這樣寫，我就這樣做了」的感覺。

## 舊版

### Project Gradle

```groovy
//
buildscript {
    //
    ext.kotlin_version = '1.7.20'
    ext.hilt_version = '2.45'
    repositories {
        google()
        mavenCentral()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:7.3.1'
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
        classpath "com.google.dagger:hilt-android-gradle-plugin:$hilt_version"
    }
}

// 這定義了全部專案都會使用到這個命令
allprojects {
    repositories {
        // 這兩個皆是託管開源的 library 供專案使用
        google()
        mavenCentral()
    }
}

tasks.register('clean', Delete) {
    delete rootProject.buildDir
}

```

### Module Gradle

```groovy
plugins {
    id 'com.android.application'
    id 'kotlin-android'
    id 'kotlin-parcelize'
    id 'kotlin-kapt'
    id 'dagger.hilt.android.plugin'
}

android {
    compileSdk 34


    defaultConfig {
        applicationId "com.example.android.hilt"
        minSdkVersion 16
        targetSdkVersion 33
        versionCode 1
        versionName "1.0"

        testInstrumentationRunner 'androidx.test.runner.AndroidJUnitRunner'

        javaCompileOptions {
            annotationProcessorOptions {
                arguments["room.incremental"] = "true"
            }
        }
    }

    compileOptions {
        sourceCompatibility 1.8
        targetCompatibility 1.8
    }
}

dependencies {

    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'androidx.core:core-ktx:1.12.0'
    implementation 'androidx.constraintlayout:constraintlayout:2.1.4'
    implementation 'androidx.recyclerview:recyclerview:1.3.2'

    // Room
    implementation "androidx.room:room-runtime:2.6.0"
    kapt "androidx.room:room-compiler:2.6.0"

    // Testing dependencies
    androidTestImplementation "junit:junit:4.13.2"
    androidTestImplementation "androidx.test:core-ktx:1.5.0"
    androidTestImplementation "androidx.test.ext:junit-ktx:1.1.5"
    androidTestImplementation "androidx.test:rules:1.5.0"
    androidTestImplementation "androidx.test.espresso:espresso-core:3.5.1"

    // Hilt dependencies
    implementation "com.google.dagger:hilt-android:$hilt_version"
    kapt "com.google.dagger:hilt-android-compiler:$hilt_version"
}
```

### gradle.properties

```groovy
# Project-wide Gradle settings.
# IDE (e.g. Android Studio) users:
# Gradle settings configured through the IDE *will override*
# any settings specified in this file.
# For more details on how to configure your build environment visit
# http://www.gradle.org/docs/current/userguide/build_environment.html
# Specifies the JVM arguments used for the daemon process.
# The setting is particularly useful for tweaking memory settings.
org.gradle.jvmargs=-Xmx1536m
# When configured, Gradle will run in incubating parallel mode.
# This option should only be used with decoupled projects. More details, visit
# http://www.gradle.org/docs/current/userguide/multi_project_builds.html#sec:decoupled_projects
# org.gradle.parallel=true
# AndroidX package structure to make it clearer which packages are bundled with the
# Android operating system, and which are packaged with your app's APK
# https://developer.android.com/topic/libraries/support-library/androidx-rn
android.useAndroidX=true
# Automatically convert third-party libraries to use AndroidX
android.enableJetifier=true
# Kotlin code style for this project: "official" or "obsolete":
kotlin.code.style=official
# Use kapt incremental
kapt.incremental.apt=true
```

### gradle-wrapper.properties

```groovy
#Sat Mar 13 20:10:35 CET 2021
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-7.4-bin.zip
```
