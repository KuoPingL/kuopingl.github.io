---
layout: post
title: "Commands for abd and gradle "
date: 11/07/23 21:22:20  +0800
categories: [Collections]
keywords: gradle, abd, terminal
---

# gradlew

以下指令適用在 Gradle Wrapper 上。這裡不會有任何說明，但請參考 ... 。

## 常用指令

| command                   | prepare | description                                                        | source                                                          |
| :------------------------ | :------ | :----------------------------------------------------------------- | :-------------------------------------------------------------- |
| `./gradlew tasks`         | --      | 打印出所有可用的任务列表                                           | [Gradle 的基本构建任务](https://www.jianshu.com/p/de779844219c) |
| `./gradlew tasks --all`   | --      | 打印出所有可用的任务列表，並將每个任务对应依赖的详细介绍也会被打印 | [Gradle 的基本构建任务](https://www.jianshu.com/p/de779844219c) |
| `./gradlew assembleDebug` | --      | 打包一个 debug 版本的 APK                                          | [Gradle 的基本构建任务](https://www.jianshu.com/p/de779844219c) |
| `./gradlew check`         | --      | 运行所有的检查（在一个正在连接的设备上运行测试）                   | [Gradle 的基本构建任务](https://www.jianshu.com/p/de779844219c) |
| `./gradlew build`         | --      | 触发 assemble 和 check                                             | [Gradle 的基本构建任务](https://www.jianshu.com/p/de779844219c) |
| `./gradlew clean`         | --      | 清除项目的输出                                                     | [Gradle 的基本构建任务](https://www.jianshu.com/p/de779844219c) |

## 特別指令

| command                              | prepare                                             | description                                                  | source                                                                                                                                                                    |
| :----------------------------------- | :-------------------------------------------------- | :----------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `./gradlew aDeb -Pkapt.verbose=true` | add `kapt.use.worker.api=true` in gradle.properties | detect non-incremental annotation processors in your project | [Making incremental KAPT work (Speed Up your Kotlin projects!)](https://medium.com/@daniel_novak/making-incremental-kapt-work-speed-up-your-kotlin-projects-539db1a771cf) |
