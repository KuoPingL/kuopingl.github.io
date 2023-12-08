---
layout: post
title: "Lesson 1 - 從專案暸解 Gradle"
date: 2023-12-08 20:50:54  +0800
categories: [The World of Gradle]
keywords: gradle, beginner
---

你是否時常跟 Gradle 乾瞪眼呢？ Look no further，這篇就是你需要的。一篇帶你看懂 Gradle。

# 序言

我相信很多人寫了幾年的 Android 卻都不知道 Gradle 到底是什麼。 所以我想借這次機會講解也順便自學什麼是 Gradle。

[Gradle](https://docs.gradle.org/current/userguide/userguide.html) 其實是一個 **自動化的工具** 。 通過他，我們可以進行 **編譯、測試、檢查程式碼、打包** 等工作

目前可以使用 Gradle 的語言包括 : Android, Java, Kotlin Multiplatform, Groovy, Scala, Javascript, 以及 C/C++。

而支援 Gradle 的 IDE 則包括 : Android Studio, IntelliJ IDEA, Visual Studio Code, Eclipse, 以及 NetBeans 。

我們甚至可以使用 命令指令 (CLI) 和 CI 伺服器中的終端機中啟動 Gradle。

對 Android 工程師而言，想學會 Gradle 與其去翻他的官方文件還不如先搞懂什麼是 `build.gradle` 什麼是 `settings.gradle`。 但是想要暸解他們之前，我們需要有一些基本 Gradle 常識。

# Gradle 小常識

## Gradle 是如何看待專案的 ?

在 Gradle 開始支援 Java 專案時，由於這些專案通常會將 **Sources** 與 **Resources** 分類擺放，所以 Gradle 也就開始把專案看成多個 **source set**。 不同的 source set 便可以一起被編譯與執行。

通過 Source Set 的觀念，Gradle 可以通過 Task 將編譯相關的三者連接在一起 :

- 源碼所在位置
- 編譯所需要的類別 (compilation classpath) 包括 dependencies
- 編譯後檔案的所在位置

<center>
    <a href="https://docs.gradle.org/current/userguide/building_java_projects.html#sec:java_source_sets">
        <img src = "https://docs.gradle.org/current/userguide/img/java-sourcesets-compilation.png">
    </a>
</center>

其中 :

- [`sourceSetCompileOnly`](https://docs.gradle.org/current/userguide/java_plugin.html#java_source_set_configurations) 和 [`sourceSetImplementation`](https://docs.gradle.org/current/userguide/java_plugin.html#java_source_set_configurations) 是 JavaPlugin 所提供的 dependency configuration
- [`compileSourceSetJava`](https://docs.gradle.org/current/userguide/java_plugin.html#java_source_set_configurations) 則是 JavaPlugin 中負責 Source Set 編譯的 task。 他會自動或被動使用在指定的 Source Set 之中

另外， resource 資源也會通過 JavaPlugin 的 [`processSourceSetResources`](https://docs.gradle.org/current/userguide/java_plugin.html#java_source_set_configurations) 進行複製與修改並產出最後被打包的產物 :

<center>
    <a href="https://docs.gradle.org/current/userguide/building_java_projects.html#sec:java_source_sets">
        <img src = "https://docs.gradle.org/current/userguide/img/java-sourcesets-process-resources.png">
    </a>
</center>

所謂的 **Source Set** 指的是擁有 _共同目標_ 的 Java 源碼與資源檔案的集合。就像是 Android 新的專案中所預設的 `src/main` 、 `src/androidTest` 與 `src/test` 都是 [source set](https://developer.android.com/build#sourcesets) 。

<details markdown=1>
<summary markdown='span'>如果我們想要看到目前 Android 專案中的 Source Set，我們可以在 <code>build.gradle (Module)</code> 中這樣做 :</summary>

在 `build.gradle (Moduel)` 中的 `android`

```groovy
android {
    ...
    sourceSets.each {
        println(it.name)
    }
}
```

最後通過 `./gradlew build` 看到目前的 source set :

```shell
debug  ----------------+
main                 default source sets
release ---------------+

androidTest -----------+
androidTestDebug     androidTest{VariantName} source sets
androidTestRelease ----+

test ------------------+
testDebug            test{VariantName} source sets
testRelease -----------+

testFixtures ----------+
testFixturesDebug    textFixtures{VariantName} source sets
testFixturesRelease ---+
```

這些都是預設的 Source sets，但 Gradle 預設並不會自動為他們創建資料夾。

<br>
</details>

### 如何新增與設定 Source Set ?

#### 新增 Source Set

如果我們想要新增 Android 的 source set 我們可以通過兩種方式來做到 :

1. 設定 `buildType`，像是預設的 `debug` 與 `release`
   ```kotlin
   buildTypes {
       release {
           //...
       }
   }
   ```
2. 設定 `productFlavor`，像是 `android` block 中的 `defaultConfig`
   ```groovy
   android {
       compileSdk 34
       defaultConfig {
           // ...
       }
   }
   ```

`buildType` 與 `productFlavor` 的主要差別在於他們的目的。

> Build Types model your development lifecyle (debug, "dogfood", release, etc.).
> Product flavors model your distribution strategy (Google IAP vs. Amazon IAP vs. BlackBerry IAP, etc.).
> <a href = "https://stackoverflow.com/questions/27905934/why-are-build-types-distinct-from-product-flavors"> -- 子曰 </a>

也就是說應用程式在不同的 buildTypes 時會有 _相同的功能_。 但在不同的 productFlavors 下，_功能則會有所限制_，像是 `free` 與 `paid` 。

在新版的 `build.gradle (Module)` 我們可以看到 `buildType` 的範例 :

```kotlin
buildTypes {
    release {
        isMinifyEnabled = false
        proguardFiles(
            getDefaultProguardFile("proguard-android-optimize.txt"),
            "proguard-rules.pro"
        )
    }
    /*
    // 預設不會顯示，但會設定為 debuggable true (https://developer.android.com/build/build-variants?utm_source=android-studio#build-types)
    // This lets you debug the app on secure Android devices and configures app signing with a generic debug keystore.
    // debug.keystore is merely for developing and testing purposes, so using that you can't release your app to Google Play using that only. [https://stackoverflow.com/a/21880423/18597115]

        debug {
            debuggable true
        }
    */

    // 我們也可以設定一個 staging build type
    staging {
        // 並設定他為 debug
        initWith debug
        // Inject build variables into the manifest (https://developer.android.com/build/manage-manifests#inject_build_variables_into_the_manifest)
        manifestPlaceholders = [hostName:"internal.example.com"]
        applicationIdSuffix ".debugStaging"
    }
}
```

而 `productFlavor` 也很類似的設定，但每個 productFlavor 都需要指定的 `flavorDimensions` :

```groovy
// All flavors must now belong to a named flavor dimension. Learn more at https://d.android.com/r/tools/flavorDimensions-missing-error-message.html

flavorDimensions = ['version']

productFlavors {
    free {
        applicationIdSuffix ".free"
        versionNameSuffix "-free"
        dimension "version"
    }
    paid {
        applicationIdSuffix ".paid"
        versionNameSuffix "-paid"
        dimension "version"
    }
}
```

經過以上的設定，當我們同步後， 我們可以看到這個專案就會多出好幾種 source sets :

```shell
paid
testPaid
androidTestPaid
testFixturesPaid

free
testFree
androidTestFree
testFixturesFree

staging
testStaging
androidTestStaging
testFixturesStaging
```

如果我們新增幾個 `flavorDimension` 和 productFlavor :

```groovy
flavorDimensions = ['version', 'server', 'local']

productFlavors {
    free {
        applicationIdSuffix ".free"
        versionNameSuffix "-free"
        dimension "version"
    }
    paid {
        applicationIdSuffix ".pro"
        versionNameSuffix "-pro"
        dimension "version"
    }

    client1 {
        dimension 'local'
    }

    client2 {
        dimension 'local'
    }

    server1 {
        dimension 'server'
    }

    server2 {
        dimension 'server'
    }
}
```

然後我們在 AS IDE 中點選 `Build > Select Build Variant (or View > Tool Windows > Build Variants`

我們就能看到目前所有的 **Build Variants** 也就是全部可以 Build 的方式。 這些 Build 會以 `{productFlavor(s)}{buildType}` 的方式展現 :

<center>
    <img src = "/images/posts/jekyll/gradle/build_variant.png"/>
</center>

如此一來我們就可以讓 Gradle 創建出不同功能與應用的應用程式了。

#### 設定 Source Set

想知道我們能如何設定 buildType 或 productFlavor 就要看看當下他們繼承了什麼介面了。

當我們設定 buildType 時，其實我們是在設定繼承 [**BuildType**](https://developer.android.com/reference/tools/gradle-api/8.1/com/android/build/api/dsl/BuildType) 或其子類型的物件 :

```kotlin
interface BuildType : Named, VariantDimension, ExtensionAware
interface ApplicationBuildType : BuildType, ApplicationVariantDimension
interface DynamicFeatureBuildType : BuildType, DynamicFeatureVariantDimension
interface LibraryBuildType : BuildType, LibraryVariantDimension
interface TestBuildType : BuildType, TestVariantDimension
```

其中我們最常見的 buildType `debug` 與 `release` 就是 [**ApplicationBuildType**](https://developer.android.com/reference/tools/gradle-api/8.1/com/android/build/api/dsl/ApplicationBuildType)。

而 [**TestBuildType**](https://developer.android.com/reference/tools/gradle-api/8.1/com/android/build/api/dsl/TestBuildType) 則決定了 <i>Instrumental Test</i> 要以哪個 buildType 為測試目標。

```groovy
// https://developer.android.com/studio/test/advanced-test-setup#change-the-test-build-type
android {
    testBuildType "stage" // 預設為 debug
}
```

相同的，當我們在設定 ProductFlavor 時，我們其實是在對實作以下介面的物件進行設定 :

```kotlin
interface ProductFlavor : Named, BaseFlavor, ExtensionAware, HasInitWith
interface ApplicationProductFlavor : ApplicationBaseFlavor, ProductFlavor
interface DynamicFeatureProductFlavor : DynamicFeatureBaseFlavor, ProductFlavor
interface LibraryProductFlavor : LibraryBaseFlavor, ProductFlavor
interface TestProductFlavor : TestBaseFlavor, ProductFlavor
```

其中 `defaultConfig` 與我們創建的 productFlavor ( `free` 、 `paid` 、 `client`、 `server` ) 都是在定義 [**ApplicationProductFlavor**](https://developer.android.com/reference/tools/gradle-api/8.1/com/android/build/api/dsl/ApplicationProductFlavor)。

另外， [**TestProductFlavor**](https://developer.android.com/reference/tools/gradle-api/8.1/com/android/build/api/dsl/TestProductFlavor) 也是用來指定要測試的 build 有哪些。

### SourceSet 與 Google 產品的範例

我們最常會在使用 Google 產品時使用到 Build Set。 所以在此特別提供了 [官方教學](https://firebase.blog/posts/2016/08/organizing-your-firebase-enabled-android-app-builds)

## 何謂 Gradle Build ?

Gradle Build 的意思其實就是針對一個或多個 (多重) 專案執行我們設定的自動化程序。而 Gradle 會為每個專案都創建一個 [**Project**](https://docs.gradle.org/current/dsl/org.gradle.api.Project.html) 物件。

Gradle 本身是一個基于依賴編程的語言。因此想要讓 Project 執行這些自動化程序，他們內部就必須要有多個 [**Task**](https://docs.gradle.org/current/dsl/org.gradle.api.Task.html) 並且設定這些 Task 所依賴的 Dependencies。 通過 Gradle，這些 Tasks 會按照 Dependencies 的順序形成一個 [Directed Acyclic Graph](http://en.wikipedia.org/wiki/Directed_acyclic_graph) (DAG) :

<center>
    <a href="https://docs.gradle.org/current/userguide/build_lifecycle.html#sec:build_phases">
        <img src = "https://docs.gradle.org/current/userguide/img/task-dag-examples.png"/>
    </a>
</center>

最後再由 [**Build Script**](https://docs.gradle.org/current/userguide/tutorial_using_tasks.html) 與 [**Plugin**](https://docs.gradle.org/current/javadoc/org/gradle/api/Plugin.html) 通過 [**task dependencies 運作方式**](#task-dependencies-mechanism-範例) 對這個圖表進行設定。

> 人話 : 其實也就是對 tasks 的執行順序與依賴關係進行改變。

> 注意 : 這裡的 task dependencies 與我們 `buildScript` 中使用的 `dependencies` block 是不同的。Task dependencies 只會影響 tasks 被執行的順序。 `buildScript` 中的 `dependencies` 則是 Task 或是專案所需要的函式庫。

雖然可能還看不懂這些詞彙的意思，但之後都會一一說明。

### Build Lifecycle

當 Gradle 進行 build 時，他會經歷 [3 個狀態](https://docs.gradle.org/current/userguide/build_lifecycle.html#sec:build_phases) :

1. **Initialization**
   - Gradle 會尋找 `settings.gradle` 並決定是否要創建 Multi-Project 還是 Single Project。
   - 在設定 Settings 時，Gradle 還會做以下事情 :
     - 將函式庫加在 `buildScript` 的 `classPath` 中
     - 定義哪個 **included builds** 會參與在 **composite build** 中
     - 定義哪個專案會參與在多重專案中
2. **Configuration**
   - 此時 Gradle 會將 tasks 與 properties 傳入剛被創建的 Project 中
   - 在這階段，我們針對 Project、 DAG 與 Task 的創建進行監控 :
     - 通過 `gradle.beforeProject`、 `gradle.afterProject` 或 [**ProjectEvaluationListener**](https://docs.gradle.org/current/javadoc/org/gradle/api/ProjectEvaluationListener.html) 監聽 Project 的分析並在 Project 分析之前與之後進行 [其他設定](https://docs.gradle.org/current/userguide/build_lifecycle.html#react_to_project_evaluation)。
     - 通過 [**TaskExecutionGraphListener**](https://docs.gradle.org/current/javadoc/org/gradle/api/execution/TaskExecutionGraphListener.html) 監聽 Task DAG 何時創建
     - 通過 [`taskContainer.whenTaskAdded` 對 Task 進行設定](https://docs.gradle.org/current/userguide/build_lifecycle.html#sec:task_creation)
3. **Execution**
   - 這裡 Gradle 會執行 tasks
   - 我們也可以使用 [**TaskExecutionListener**](https://docs.gradle.org/current/javadoc/org/gradle/api/execution/TaskExecutionListener.html) 來監聽 Gradle [何時開始執行 Task 與 何時結束執行 Task](https://docs.gradle.org/current/userguide/build_lifecycle.html#react_to_task_execution_notifications)。

<center>
<img src = "/images/posts/jekyll/gradle/gradle_build_lifecycle.png"/>
</center>

也可以看官方版本的 :

<center>
    <a href="https://docs.gradle.org/current/userguide/build_lifecycle.html#build_lifecycle"><img src = "https://docs.gradle.org/current/userguide/img/build-lifecycle-example.png"/></a>
</center>

如果我們是多重專案，他也是有類似的生命週期 :

<center>
    <a href="https://docs.gradle.org/current/userguide/intro_multi_project_builds.html"><img src = "https://docs.gradle.org/current/userguide/img/gradle-basic-9.png"/></a>
</center>

<details markdown=1>
<summary markdown='span'> 如果我們在 <code>settings.gradle</code> 與 <code>build.gradle</code> 中進行列印，我們可以看出他們的 <a href="https://docs.gradle.org/current/userguide/build_lifecycle.html#example">執行順序</a>。 </summary>

```groovy
// settings.gradle
rootProject.name = 'basic'
println 'This is executed during the initialization phase.'

// ====================================================== //

// build.gradle
println 'This is executed during the configuration phase.'

tasks.register('configured') {
    println 'This is also executed during the configuration phase, because :configured is used in the build.'
}

tasks.register('test') {
    doLast {
        println 'This is executed during the execution phase.'
    }
}

tasks.register('testBoth') {
	doFirst {
	  println 'This is executed first during the execution phase.'
	}
	doLast {
	  println 'This is executed last during the execution phase.'
	}
	println 'This is executed during the configuration phase as well, because :testBoth is used in the build.'
}
```

通過 `gradle test testBoth` 我們會得到 :

```shell
> gradle test testBoth
# 進入 settings.gradle
This is executed during the initialization phase.

# 進入 build.gradle，並建立 DAG
> Configure project :
This is executed during the configuration phase.
This is executed during the configuration phase as well, because :testBoth is used in the build.

# 開始執行 Task
> Task :test
This is executed during the execution phase.

> Task :testBoth
This is executed first during the execution phase.
This is executed last during the execution phase.

BUILD SUCCESSFUL in 0s
2 actionable tasks: 2 executed
```

</details>

## 何謂 Task ?

Task 其實就是實作 [Task 介面](https://docs.gradle.org/current/javadoc/org/gradle/api/Task.html) 的類別。他會通過 [Action 列舉](https://docs.gradle.org/current/javadoc/org/gradle/api/Action.html) 來定義他的行為。

Task 的設定通常會放在以下三個位置之一 :

- build script
  這裡的 Task 都會被 **自動編譯並放入 classPath 中**。 但這些 tasks 並 **無法被共享**。
- `buildsrc` 資料夾 或 專案
  Task 會被放在 `rootProjectDir/buildSrc/src/main/{DSL Language}` 中。 這裡的 tasks 也會被 Gradle 自動編譯與放入 classPath 中， 但是他們也會被所有參與 build 的 build script 所看到，所以他們可以在同一個 build 中共用 。

  由於 `buildsrc` 通常會出現在多重專案中，所以這篇不會細談。 若有興趣可以看看官方的 [管理 Gradle 專案頁面](https://docs.gradle.org/current/userguide/organizing_gradle_projects.html#organizing_gradle_projects)。

  另外，Gradle 也推出新的多重 Build 管理方式 : [Composite Build](https://docs.gradle.org/current/userguide/composite_builds.html#header)。

- 單獨專案 (JAR)
  這個專案會產生並發佈 JAR 來讓多個 build 使用。
  JAR 通常會有使用客製化的 plugins 或 多個 tasks 。

### 簡單 Task 的創建

我們從 [Gradle Build](#何謂-gradle-build) 章節中知道 Task 是可以通過 [TaskContainer](https://docs.gradle.org/current/dsl/org.gradle.api.tasks.TaskContainer.html) 的 `register`、`create`、 `mayCreate` 與 `replace` 創建。

<details markdown=1>
<summary markdown='span'>在 build script 中創建 Task</summary>

```groovy
tasks.register('myTask') {
    /* actions */
}

// 或是用 create
tasks.create('myTask') {

}
```

</details>

如果我們可以取得 Project，像是 `build.gradle`，那我們便可以使用 **Project** 的方法。

```groovy
// https://docs.gradle.org/current/dsl/org.gradle.api.Project.html#org.gradle.api.Project:task(java.lang.String)
task('myTask') {

}

// 或是
task myTask {

}

// 如果有指定的 Task 類別，那就可以學預設在 build.gradle 中的寫法
/**
 * 其中的 Delete 是一個類別
 * public class Delete extends ConventionTask implements DeleteSpec
*/
task clean(type: Delete) {
    delete rootProject.buildDir // {path}/build
}
```

通過追蹤 ktx 的源碼， 我們會發現 TaskContainer 會通過 **TaskIdentity** 來紀錄 Task 的相關資訊再由 **ITaskFactory** 創建所需要的 Task。

TaskFactory 會確認 Task 是 **DefaultTask** 的子類別後才會創建 Task。

<details markdown=1>
<summary markdown='span'>除了通過 TaskContainer 外，我們也可以自己定義 Task。</summary>

```groovy
abstract class GreetingTask extends DefaultTask {

    // 定義參數
    @Input
    abstract Property<String> getGreeting()

    GreetingTask() {
        // 初始化參數
        greeting.convention('hello from GreetingTask2')
    }

    // 定義 Action
    @TaskAction
    def greet() {
        println 'hello from GreetingTask'
    }

    @TaskAction
    def greet2() {
        println greeting.get()
    }
}
```

並再次通過 `TaskContainer#register` 將他註冊起來 :

```groovy
tasks.register('hello', GreetingTask)

// 更改 greeting
tasks.register('greeting',GreetingTask) {
    greeting = 'greetings from GreetingTask'
}
```

當我們輸入 `gradle -q hello greeting` ， 我們會得到 :

```shell
# hello Task
hello from GreetingTask2
hello from GreetingTask
 # gretting Task
greetings from GreetingTask
hello from GreetingTask
```

方法被調用的順序是由 Gradle 決定的。

</details>

<details markdown=1>
<summary markdown='span'>我們甚至可以指令可輸入的參數</summary>

```groovy
public class UrlVerify extends DefaultTask {
    private String url;

    @Option(option = "url", description = "Configures the URL to be verified.")
    public void setUrl(String url) {
        this.url = url;
    }

    @Input
    public String getUrl() {
        return url;
    }

    @TaskAction
    public void verify() {
        getLogger().quiet("Verifying URL '{}'", url);

        // verify URL by making a HTTP call
    }
}

tasks.register('verifyUrl', UrlVerify)

// 如果可以取得 Project
task verifyUrl(type: UrlVerify)
```

我們通過 `gradle -q verifyUrl --url=http://www.google.com/` 可以得到 :

```shell
Verifying URL 'http://www.google.com/'
```

</details>

如果我們想要創建一個 **可被發佈** 的 Task，那我們還需要設定所需要的 plugins 與 dependencies :

```groovy
build.gradle
plugins {
    id 'groovy'
}

dependencies {
    // 通過 groovy plugin，一個繼承 JavaPlugin 的 plugin， 我們可以使用 implementation
    implementation gradleApi()
}
```

由於我們需要用到 `implementation` ，所以我們需要 JavaPlugin 或他的繼承者 `groovy`。 通過 implementation 我們才可以使用 `gradleApi()` 來取得目前 Gradle 版本的 API 。

最後我們便可以與之前一樣地定義 Task :

```groovy
package org.gradle

import org.gradle.api.DefaultTask
import org.gradle.api.tasks.TaskAction
import org.gradle.api.tasks.Input

class GreetingTask extends DefaultTask {

    @Input
    String greeting = 'hello from GreetingTask'

    @TaskAction
    def greet() {
        println greeting
    }
}
```

以上我們也只是談了 Task 的創建與定義，但卻沒有談到 Task 共享時會遇到的問題與解決方法。 如果有興趣的可以造訪 [官方頁面](https://docs.gradle.org/current/userguide/custom_tasks.html) 。

另外， Task 其實也可以進行非同步運算，但由於這些一般開發者不太會遇到，所以想暸解的可以在這 [頁面](https://docs.gradle.org/current/userguide/worker_api.html) 找到。

在我們定義了 tasks 之後，他們會由 Gradle 分析並以 DAG 方式連結在一起。 如果想要改變他們被執行的順序，我們就需要暸解 [Task Dependencies Mechanism](#task-dependencies-mechanism-範例) 了。

### [Task Dependencies Mechanism 範例](https://docs.gradle.org/current/userguide/tutorial_using_tasks.html#sec:task_dependencies)

<details markdown=1>
<summary markdown = 'span'>
想要影響 Task DAG，我們可以通過 Script 的方式插入 Tasks 並且通過依賴關係將 tasks 連接在一起。
</summary>

```groovy
tasks.register('hello') { // 創建 hello Task
    doLast {
        println 'Hello world!'
    }
}
tasks.register('intro') { // 創建 intro Task
    dependsOn tasks.hello // 會在 hello Task 之後才執行
    doLast {
        println "I'm Gradle"
    }
}

//
// Hello world
// I'm Gradle
```

通過 `gradle -q intro` 我們會得到 :

```shell
> gradle -q intro
Hello world
I'm Gradle
```

<br>
</details>

<details markdown=1>
<summary markdown = 'span'>
或是使用 [TaskProvider]() 來做到相同的事。
</summary>

```groovy
def hello = tasks.register('hello') {
    doLast {
        println 'Hello world!'
    }
}

def intro = tasks.register('intro') {
    doLast {
        println "I'm Gradle"
    }
}

intro.configure {
    dependsOn hello
}
```

<br>
</details>

<details markdown=1>
<summary markdown = 'span'>
我們甚至可以 [跨 projects 來設定](https://docs.gradle.org/current/userguide/more_about_tasks.html#sec:adding_dependencies_to_tasks) 。
</summary>

```groovy
project('project-a') {
    tasks.register('intro')  {
        dependsOn ':project-b:hello'
        doLast {
            println "I'm Gradle"
        }
    }
}

project('project-b') {
    tasks.register('hello') {
        doLast {
            println 'Hello world!'
        }
    }
}
```

<br>
</details>

<details markdown=1>
<summary markdown = 'span'>
我們還可以同時依賴多個 tasks 並通過 loop 的方式在 TaskProvider 中尋找。
</summary>

```groovy
def taskX = tasks.register('intro') {
    doLast {
        println "I'm Gradle"
    }
}

// Using a Gradle Provider
taskX.configure {
    dependsOn(provider {
        tasks.findAll { task -> task.name.startsWith('hello') }
    })
}

tasks.register('hello1') {
    doLast {
        println('Hello World!')
    }
}

tasks.register('hello2') {
    doLast {
        println('Hello Mars!')
    }
}

tasks.register('goodbye') {
    doLast {
        println('Good Bye')
    }
}

// gradle -q intro
// Hello World!
// Hello Mars!
// I'm Gradle
```

<br>
</details>

<br>

雖然依賴關係可以控制 tasks 的執行順序，但我們其實還可以強制設定執行的順序。

以下是一些會使用強制設定執行順序的情形 :

- 當我們希望 clean 之後才 build。
- 當我們想要在 release build 之前進行 build 的檢測，以保全部證件都齊全。
- 讓快速的先執行 (Unit Test 先執行，後執行 Integrated Test)。
- 當某個 Task 需要其他 tasks 的結果時也需要強制設定順序。

至於要如何設定 tasks 的執行順序，我們需要制定兩個規則 : _“must run after”_ 與 _“should run after”_。

<details markdown=1> 
<summary markdown='span'> <code>mustRunAfter</code> 範例 </summary>

```groovy
def intro = tasks.register('intro') {
    doLast {
        println "I'm Gradle"
    }
}
def hello = tasks.register('hello') {
    doLast {
        println 'Hello World!'
    }
}
intro.configure {
    mustRunAfter hello
}
```

<br>
</details>

<details markdown=1> 
<summary markdown='span'> <code>shouldRunAfter</code> 範例。 (這並非強制性的)  </summary>

```groovy
def intro = tasks.register('intro') {
    doLast {
        println "I'm Gradle"
    }
}
def hello = tasks.register('hello') {
    doLast {
        println 'Hello World!'
    }
}
intro.configure {
    shouldRunAfter hello
}
```

<br>
</details>

<br>
> 切記 : 
> <code>mustRunAfter</code> 與 <code>shouldRunAfter</code> 並不表示兩個 tasks 會互相依賴。 順序設定只有在兩者都被執行時才會生效。
> 我們甚至可以通過 <a href = "https://docs.gradle.org/current/userguide/command_line_interface.html#sec:continue_build_on_failure"><code>--continue</code></a> 指令讓 Gradle 在沒有依賴性錯誤時，無視其他錯誤繼續執行 tasks。

竟然我們可以新增 Task 以及調整 tasks 的執行順序，我們自然也可以做出以下行為 :

<details markdown=1><summary markdown='span'>1. 跳過 Task 的執行</summary>

```groovy
def hello = tasks.register('hello') {
    doLast {
        println 'hello world'
    }
}

hello.configure {
    // 如果有一個叫做 skipHello 的參數，就不會執行 hello
    def skipProvider = providers.gradleProperty("skipHello")
    // onlyIf(onlyIfReason: String, onlyIfSpec: Spec)
    // hello will only execute only if "skipHello" is not present ... hence !skipProvider.present
    onlyIf("there is no property skipHello") {
        !skipProvider.present
    }
}
```

如此一來，當我們輸入 :

```shell
> gradle hello -PskipHello --info # -P{property name}

> Task :hello SKIPPED
Skipping task ':hello' as task onlyIf 'there is no property skipHello' is false.
:hello (Thread[included builds,5,main]) completed. Took 0.018 secs.
```

<br>
</details>
<details markdown=1><summary markdown='span'><a href="https://docs.gradle.org/current/userguide/more_about_tasks.html#sec:using_stopexecutionexception">2. 通過 exception 來停止 Task</a></summary>

```groovy
def compile = tasks.register('compile') {
    doLast {
        println 'We are doing the compile.'
    }
}

compile.configure {
    doFirst {
        // Here you would put arbitrary conditions in real life.
        if (true) {
            throw new StopExecutionException()
        }
    }
}
tasks.register('myTask') {
    dependsOn('compile')
    doLast {
        println 'I am not affected'
    }
}
```

這裡比較有趣的是，當 **StopExecutionException** 發生在 `compile` 時，他只會終止 `compile`，並不會影響其他的 tasks 包括依賴著 `compile` 的 `myTask`。

```shell
> gradle -q myTask
I am not affected
```

我們也可以使用 `dependsOn` 讓多個 Task 一同執行 :

```groovy
tasks.register('groupPing') {
    dependsOn 'pingServer1', 'pingServer2'
}
```

<br>
</details>

<details markdown=1><summary markdown='span'>3. Enable / Disable Task</summary>

```groovy
def disableMe = tasks.register('disableMe') {
    doLast {
        println 'This should not be printed if the task is disabled.'
    }
}

disableMe.configure {
    enabled = false
}
```

由於 `enabled = false`，所以 `disableMe` 會被跳過。

</details>

<details markdown=1> 
<summary markdown='span'><a href="https://docs.gradle.org/current/userguide/more_about_tasks.html#sec:task_timeouts">4. Timeout</a></summary>

```groovy
tasks.register("hangingTask") {
    doLast {
        Thread.sleep(100000)
    }
    timeout = Duration.ofMillis(500) // By default, tasks never time out.
}
```

<br>
</details>

<details markdown=1>
<summary markdown='span'>5. 定義 Task Rule</summary>

這裡所謂的 Task Rule 其實相當於 tasks 的 for-loop。 每個被執行的 Task 都需要符合 Rule :

```groovy
// 定義 rule 名稱或 description 為 "Pattern: ping<ID>"
tasks.addRule("Pattern: ping<ID>") { String taskName ->

    // 當遇到 taskName 為 ping* 的就會印出他的名稱 減去 'ping' 後的字串
    if (taskName.startsWith("ping")) {
        task(taskName) {
            doLast {
                println "Pinging: " + (taskName - 'ping')
            }
        }
    }
}
```

通過 `gradle -q pingServer1` (範例中假設有一個名為 `pingServer1` 的 Task)，他會印出 :

```shell
> gradle -q pingServer1
Pinging: Server1
```

如果我們讓多個 `ping*` tasks 一同進行 :

```groovy
tasks.register('groupPing') {
    dependsOn 'pingServer1', 'pingServer2'
}
```

結果便會顯示多個 Task 進入 `Pattern: ping<ID>` 中 :

```shell
> gradle -q groupPing
Pinging: Server1
Pinging: Server2
```

</details>

<details markdown=1>
<summary markdown='span'>6. 設定最後需要執行的 tasks aka <a href="https://docs.gradle.org/current/userguide/more_about_tasks.html#sec:finalizer_tasks"><b>Finalizer Tasks</b></a></summary>

目前我們看到的只能改變 DAG 中 tasks 之間的依賴性、 task 是否要執行 以及某些執行後的設定或條件。但 Finalizer Tasks 可以讓我們將某些 tasks 在最後一個 task 執行後自動執行。

```groovy
def taskX = tasks.register('taskX') {
    doLast {
        println 'taskX'
    }
}
def taskY = tasks.register('taskY') {
    doLast {
        println 'taskY'
    }
}

taskX.configure { finalizedBy taskY }
```

通過 `gradle -q taskX` 我們可以得到 :

```shell
> gradle -q taskX
taskX
taskY
```

當你看到這個範例時，你有沒有想過如過 `taskX` _依賴著_ `taskY` 時會發生什麼事呢？

如果我們寫

```groovy
def taskX = tasks.register('taskX') {
    dependsOn 'taskY' // 依賴 taskY
    doLast {
        println 'taskX'
    }
}
```

再次輸入 `gradle -q taskX` 我們就可以得到 :

```shell
FAILURE: Build failed with an exception.

* What went wrong:
Circular dependency between the following tasks:
:taskX
\--- :taskY
     \--- :taskX (*)

(*) - details omitted (listed previously)


* Try:
> Run with --stacktrace option to get the stack trace.
> Run with --info or --debug option to get more log output.
> Run with --scan to get full insights.
> Get more help at https://help.gradle.org.
```

很有趣吧？

</details>

### 如何使用共享的 Task ?

如果我們的 Task 是定義在 `buildsrc` 中，我們只需要使用 `import` 即可。

若他是一個 JAR，那我們需要在 `repository` 通過 `repository` 定義下載 url 並在 `dependencies` 指定所需要的 `{group:name:version}` :

```groovy
buildscript {
    repositories {
        maven {
            url = uri(repoLocation)
        }
    }
    dependencies {
        // https://stackoverflow.com/a/50441077/18597115
        // 我們使用 classPath 因為這是 Project.buildscript 需要的
        // 如果我們是在設定 Project.dependencies 那就需要使用 implementation 了
        classpath 'org.gradle:task:1.0-SNAPSHOT'
    }
}

tasks.register('greeting', org.gradle.GreetingTask) {
    greeting = 'howdy!'
}
```

### Task 的作用

我們目前只知道如何創建與共享 Task，但 Task 到底是用來做什麼的呢？

想知道 Task 是如何使用，我們可以參考 [**JavaPlugin**](https://docs.gradle.org/current/userguide/java_plugin.html) 中所定義的 Task。

JavaPlugin 會提供以下元件 :

1. Task
2. Configuration
3. Object Extension

其中 Task 則包含以下範疇的 tasks :

- [一般的 Task](https://docs.gradle.org/current/userguide/java_plugin.html#sec:java_tasks)
  這些 tasks 會負責編譯、測試 Java 源碼、 創建 Java Doc 、 打包成 JAR 、 清除 build directory 等等
- [SourceSet Task](https://docs.gradle.org/current/userguide/java_plugin.html#sec:java_tasks)
  JavaPlugin 會將這些 tasks 放入指定的 SourceSet 中

## 何謂 Plugins ？

我們在 [何謂 Gradle Build ?](#何謂-gradle-build) 中提到通過 plugins 我們可以對 tasks 的 DAG 進行改變。這個行為其實是通過實作 [**Plugin<T>** 介面](https://docs.gradle.org/current/javadoc/org/gradle/api/Plugin.html) 來達成的， ie Gradle 的 core plugin [JavaPlugin](https://docs.gradle.org/current/javadoc/org/gradle/api/plugins/JavaPlugin.html) ( 這是他的 [源碼](https://github.com/gradle/gradle/blob/master/subprojects/plugins/src/main/java/org/gradle/api/plugins/JavaLibraryPlugin.java) ) 。

Plugins 可以分成兩種 : **Binary** 與 **Script**。

| Plugins | 簡介                                                                                                        | 所在位置                                                       |
| :------ | :---------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------- |
| Binary  | 這是使用程式來實作 **Plugin** ( 當然也可以使用 Gradle DSL ) 來創建的類別。 他是當作專案基礎 plugin 的使用。 | 可以在 `buildscript` 中也可以是在專案之外以 **JAR** 方式存在。 |
| Script  | Script 是對當下 build script 中的額外設定                                                                   | 一般都會在 `buildscript` 中，但也是可以遠程取得。              |

> 由於在創建 tasks DAG 前 plugins 就需要被設定，所以 <code>plugins {}</code> 一定會在檔案的最上面定義。
> 如果無法定義 `plugins{}` 我們也可以在 `buildscript.plugins{}` 中定義。

### Plugin 的創建

Plugin 與 [Task 一樣](#何謂-task) 都可以被放在 3 個地方 :

- build script
- `buildsrc`
- JAR

但與 Task 不同的是，Plugin 的創建只能通過實作 **Plugin** 才行。而且由於 **Plugin\<T\>** 是個泛型，所以我們可以選擇要對哪種類型進行設定。

<details markdown=1>
<summary markdown='span'>這裡我們來看一個 **Plugin<Project>** 的範例 </summary>

```groovy
class GreetingPlugin implements Plugin<Project> {
    void apply(Project project) {
        // 創建 'hello' task
        project.task('hello') {
            doLast {
                println 'Hello from the GreetingPlugin'
            }
        }
    }
}

// Apply the plugin
apply plugin: GreetingPlugin
```

雖然我們沒有通過 `tasks` 定義 Task，但當 Gradle build 時，Gradle 會先執行 Plugin。 因此 GreetingPlugin 會在 Project 中創建一個 Task `hello`。

所以當我們調用 `gradle -q hello`，我們會得到 :

```shell
> gradle -q hello
Hello from the GreetingPlugin
```

<br>
</details>

除了可以對 Project 進行調整外，我們還可以對以下類別進行設為 :

| T            | 作用                              |
| :----------- | :-------------------------------- |
| **Project**  | 對 Project 進行設定               |
| **Settings** | 對 Settings Script 進行設定       |
| **Gradle**   | 對 Initialization Script 進行設定 |

> 當然，我們也可以創建與發佈 Plugin 。 但目前不會談到，如果你想要發佈 Plugin，那就看這個 <a href="https://docs.gradle.org/current/userguide/implementing_gradle_plugins.html">頁面</a> 與 <a href="https://docs.gradle.org/current/userguide/custom_plugins.html#sec:publishing_your_plugin">這裡</a> 吧。

在我們創建或取得 Plugin 後，我們其實還可以通過 `extension objects` 對 Plugin 進行調整。這都要感謝 Project 中的 [**ExtensionContainer**](https://docs.gradle.org/current/javadoc/org/gradle/api/plugins/ExtensionContainer.html)。 由於他會存放所有 plugins 對 Project 進行的 **設定** 與 **屬性**，所以我們也可以通過他進行額外的修改 :

```groovy
interface GreetingPluginExtension {
    Property<String> getMessage()
}

class GreetingPlugin implements Plugin<Project> {
    void apply(Project project) {
        // Add the 'greeting' extension object
        def extension = project.extensions.create('greeting', GreetingPluginExtension)
        // (optional) 初始化 greeting
        extension.message.convention('Hello from GreetingPlugin')
        // Add a task that uses configuration from the extension object
        project.task('hello') {
            doLast {
                println extension.message.get()
            }
        }
    }
}

apply plugin: GreetingPlugin

// Configure the extension
greeting.message = 'Hi from Gradle' // 也可以寫 greeting {message = 'Hi from Gradle' }
```

通過 `gradle -q hello` 我們得到 :

```shell
> gradle -q hello
Hi from Gradle
```

### 如何加入 Plugins ?

當我們創建 Android 專案時， Gradle 會將 plugins 的設定分布在

Plugin 的加入方法取決於 [4 種狀況](https://docs.gradle.org/current/userguide/plugins.html#sec:binary_plugins):

1. 如果 plugin 是從 Plugins Portal 或 某資源庫中的 Plugins DSL 取得，我們可以使用 [以下方式](https://docs.gradle.org/current/userguide/plugins.html#sec:plugins_block) :
   ```groovy
   plugins {
       id `java`   // core plugin ; id «plugin id»
       // 若不是 sub-project 需要用到，那就設為 false
       id 'com.jfrog.bintray' version '1.8.5' // binary ; id «plugin id» version «plugin version» [apply «false»]
       // 這可以用 def helloPluginVersion=1.0.0 也可以在 gradle.properties 中寫 helloPluginVersion=1.0.0
       // 最後便在需要用到的 module 中的 buildscript 寫 plugins { id 'com.example.hello' }
       id `com.example.hello` version '${helloPluginVersion}'
   }
   ```
2. 如果 plugin 是已發佈的 JAR，我們 [可以在 `buildscript` 中加入](https://docs.gradle.org/current/userguide/plugins.html#sec:applying_plugins_buildscript)。 這個行為其實是對 [ScriptHandler](https://docs.gradle.org/current/javadoc/org/gradle/api/initialization/dsl/ScriptHandler.html) 進行設定 :

   ```groovy
   buildscript {
       repositories {
           gradlePluginPortal()
       }
       dependencies {
           classpath 'com.jfrog.bintray.gradle:gradle-bintray-plugin:1.8.5'
           // 也可以寫 classpath group: 'om.jfrog.bintray.gradle', name: 'gradle-bintray-plugin', version: '1.8.5'
       }
   }

   apply plugin: 'com.jfrog.bintray'
   ```

3. 我們可以將 plugin 在 `buildScr` 中定義為一個來源，那就用 [以下方法](https://docs.gradle.org/current/userguide/plugins.html#sec:buildsrc_plugins_dsl)。

   ```groovy
   // buildSrc/build.gradle
   // 舊版會寫 apply plugin : 'java-gradle-plugin'
   plugins {
       id 'java-gradle-plugin' // core plugin
   }

   gradlePlugin {
       plugins {
           // customized plugin
           myPlugins {
               id = 'my-plugin'
               implementationClass = 'my.MyPlugin' // implementationClass 指的是 Jar 中的類別
           }
       }
   }

   // build.gradle
   plugins {
       id 'my-plugin'
   }
   ```

   通過 `gradlePlugin`， Gradle 會自動

   - 生成 Jar 中 `META-INF` 的 descriptor
   - 調整 plugin 的 Plugin Marker Artifact
   - 如果有設定發佈 Gradle 也會自動發佈至 Gradle Plugin Portal
     <br>

4. 直接在 `buildscript` 中定義 Plugin 類別

   ```groovy
   // build.gradle
   class GreetingPlugin implements Plugin<Project> {
       void apply(Project project) {
           project.task('hello') {  // 建立名為 hello 的 Task
               // hello Task 最後的 Action 是 println
               doLast {
                   println 'Hello from the GreetingPlugin'
               }
           }
       }
   }

   // Apply the plugin
   apply plugin: GreetingPlugin
   ```

   如果我們調用 `gradle -q hello` 我們會得到 :

   ```shell
   # -q suppresses Gradle’s log messages, so that only the output of the tasks is shown
   Hello from the GreetingPlugin
   ```

   我們甚至可以建立類別的延伸 :

   ```groovy
   interface GreetingPluginExtension {
       Property<String> getMessage()
       Property<String> getGreeter()
   }

   class GreetingPlugin implements Plugin<Project> {
       void apply(Project project) {
           def extension = project.extensions.create('greeting', GreetingPluginExtension)
           project.task('hello') {
               doLast {
                   println "${extension.message.get()} from ${extension.greeter.get()}"
               }
           }
       }
   }

   <!-- 若在 Kotlin DSL 那就使用 apply<GreetingPlugin>() -->
   apply plugin: GreetingPlugin

   // Configure the extension using a DSL block
   greeting {
       message = 'Hi'
       greeter = 'Gradle'
   }
   ```

   通過 `gradle -q hello` 我們會得到 :

   ```shell
   Hi from Gradle
   ```

### Plugin 的作用

我們在 [一開始時](#何謂-plugins) 就提到 plugins 是為了影響 tasks 的 DAG 架構。 但如果我們看 [**JavaPlugin**](https://docs.gradle.org/current/userguide/java_plugin.html) 時，我們會發現原來 Plugin 還可以新增 Task。

JavaPlugin 中的 Task 包含以下三種 :

- 一般的 Task
  負責編譯、測試、清理 與 生成 javadoc
- Sourceset Task
  這些 tasks 只針對不同的 [Sourceset](#如何新增與設定-source-set)
- Lifecycle Task
  針對 Gradle Lifecycle 的行為，像是 check 與 build。

JavaPlugin 的 tasks 之間會有以下依賴關係 :

<center>
    <a href="https://docs.gradle.org/current/userguide/java_plugin.html"><img src = "https://docs.gradle.org/current/userguide/img/javaPluginTasks.png"/></a>
</center>

### Plugin 的測試

想要 [測試 Plugin](https://docs.gradle.org/current/userguide/custom_plugins.html#sec:writing_tests_for_your_plugin) 我們需要在測試 Java 檔中通過 **ProjectBuilder** 來取得 Project :

```java
public class GreetingPluginTest {
    @Test
    public void greeterPluginAddsGreetingTaskToProject() {
        Project project = ProjectBuilder.builder().build();
        project.getPluginManager().apply("org.example.greeting");

        assertTrue(project.getTasks().getByName("hello") instanceof GreetingTask);
    }
}
```

## 何謂 Project ?

對 Gradle 而言， Project 除了是「專案 或 項目」外，也是指 Gradle 的其中一個核心 API 介面 : [**Project**](https://docs.gradle.org/current/dsl/org.gradle.api.Project.html)。 因為每個專案都會有一個對應的 Project，所以他們的解釋可以互換。

Project 的創建則是通過 Gradle Build 的過程產生的，可以看看 [Gradle Build 的部分](#何謂-gradle-build)。

### 如何設定 Multi-Project ?

當我們建立新的 Android 時，他的結構預設就是 **Multi-Project Build**。 若我們擁有更多 project 時，他的資料夾會放在 `root` 中並預設會有獨立的 `build.gradle`:

```groovy
.                                               .
├── app                                         ├── app
│   ...                                         │   ...
│   └── build.gradle    - multi-projects ->     │   └── build.gradle
└── settings.gradle                             ├── lib
                                                │   ...
                                                │   └── build.gradle
                                                └── settings.gradle
```

如果我們想讓 Gradle 將兩個 projects 都進行 build，那我們則需要在 `settings.gradle` 中定義 :

```groovy
rootProject.name = 'basic-multiproject' // the name of 'app'
include 'app'
include 'lib'
```

當我們使用多個 projects 時，我們有時候會遇到以下情況 :

1. 共用 Build Logic aka [**Convention Plugins**](https://docs.gradle.org/current/samples/sample_convention_plugins.html)
2. 共用 Dependencies

### 如何共享 Build Logic ?

我們直接看 Gradle 官方的 [範例](https://docs.gradle.org/current/samples/sample_convention_plugins.html) 吧。

假設我們的專案結構如下 :

```groovy
├── internal-module
│   └── build.gradle
├── library-a
│   ├── build.gradle
│   └── README.md
├── library-b
│   ├── build.gradle
│   └── README.md
└── settings.gradle
```

其中

- `library-a` 與 `library-b` 需要用到 `internal-module`
- `library-a` 與 `library-b` 都是需要發佈的函式庫
- `internal-module` 是一個 Java Project

由於這個專案中的 projects 有 Java 與 Library 兩種，所以我們也需要兩種 plugins。 因此，我們可以建立以下結構 :

```groovy
├── buildSrc
│   ├── build.gradle
│   ├── settings.gradle
│   ├── src
│   │   ├── main
│   │   │   └── groovy // (若是 kts 這檔案名稱會是 kotlin )
│   │   │       ├── myproject.java-conventions.gradle
│   │   │       └── myproject.library-conventions.gradle
...
```

> gradle 的命名並不限制

我們新增的 gradles 有以下功能 (範例) :

- `java-conventions` : 負責進行 Java 的編譯與檢查
- `library-conventions` : 負責設定發佈的 repository 和檢查是否有 README.md
- `build.gradle` : 負責 conventions 共用的 plugins 與 dependencies

之後我們需要在 `internal-module` 中的 `build.gradle` 新增 `java-conventions` 的 plugin

```groovy
plugins {
    id 'myproject.java-conventions'
}

dependencies {
    // internal module dependencies
}
```

相似地，`library-{a or b}` 中的 `build.gradle` 也需要新增 `library-conventions` plugin :

```groovy
plugins {
    id 'myproject.library-conventions'
}

dependencies {
    implementation project(':internal-module')
}
```

接下來我們需要設定 `java-conventions` 與 `library-conventions`。

我們需要先設定 [**Precompiled Script Plugins**](https://docs.gradle.org/current/userguide/custom_plugins.html#sec:precompiled_plugins)，也就是 `buildsrc/build.gradle` 中的設定。

#### 設定 Precompiled Script Plugins

Precompiled Script Plugins 的命名有以下限制 :

- cannot start with `org.gradle`
- 不能與 built-in plugin id 同名

當 Gradle build 時，這些 plugins 會被編譯成 class 檔案並打包入 JAR。

接下來我們可以通過 `groovy-gradle-plugin` 來讓我們設定 precompiled script plugins :

```groovy
// buildsrc/build.gradle
plugins {
    id 'groovy-gradle-plugin'
}
```

如果需要用到遠端的 plugins 我們也可以設定 dependencies :

```groovy
// buildsrc/build.gradle
repositories {
    mavenCentral()
}

dependencies {
    implementation 'com.bmuschko:gradle-docker-plugin:6.4.0'
}
```

<details markdown=1>
<summary markdown='span'>如此一來我們便可以定義 `java-conventions` 與 `library-conventions` 的 plugins</summary>

```groovy
// java-conventions
plugins {
    id 'java-library'
    id 'checkstyle'
}

java {
    sourceCompatibility = JavaVersion.VERSION_11
    targetCompatibility = JavaVersion.VERSION_11
}

checkstyle {
    maxWarnings = 0
    // ...
}

tasks.withType(JavaCompile) {
    options.warnings = true
    // ...
}

dependencies {
    testImplementation("junit:junit:4.13")
    // ...
}
```

<br>
</details>

### 如何共享 Dependencies ?

> Project Dependency 是一種 「執行依賴」。 這表示當 模組 A 依賴 模組 B 時，Gradle 在 build 模組 A 的時候會先 build 模組 B 並將他的 class 放入 模組 A 的 JAR 中。

與 Build Logic 一樣，我們先假設目前的架構 :

```groovy
.
├── buildSrc
│   ├── build.gradle
│   ├── settings.gradle
│   ├── src
│   │   ├── main
│   │   │   └── groovy // (若是 kts 這檔案名稱會是 kotlin )
│   │   │       ├── myproject.java-conventions.gradle
│               └── myproject.library-conventions.gradle
| ...
├── api
│   ├── src
│   │   └──...
│   └── build.gradle
├── services
│   └── person-service
│       ├── src
│       │   └──...
│       └── build.gradle
├── shared
│   ├── src
│   │   └──...
│   └── build.gradle
└── settings.gradle
```

其中 `person-service` 依賴著 `shared` 和 `api`。 而 `api` 則需要 `shared` 專案。

首先，為了要定義多個專案，我們需要在 `settings.gradle` 設定 `include` :

```groovy
rootProject.name = 'dependencies-java'
include 'api', 'shared', 'services:person-service'
```

> 注意 : 當我們指向 <code>person-service</code> 時，我們使用了 <code>:</code> 來定義 class path。

<details markdown=1>
<summary markdown='span'>接下來我們需要定義 <code>shared</code>、 <code>api</code> 與 <code>person-service</code> 之間的關係以及他們所需要的 plugins

```groovy
// shared/build.gradle
plugins {
    id 'myproject.java-conventions'
}

// api/build.gradle
plugins {
    id 'myproject.java-conventions'
}

dependencies {
    implementation project(':shared')
}

// services/person-service/build.gradle
plugins {
    id 'myproject.java-conventions'
}

dependencies {
    implementation project(':shared')
    implementation project(':api')
}
```

<details markdown=1>
<summary markdown='span'>接下來我們需要設定 `myproject.java-conventions.gradle` </summary>

```groovy
plugins {
    id 'java' // JavaPlugin
}

repositories {
    mavenCentral()
}

dependencies {
    testImplementation "junit:junit:4.13"
}
```

<br>
</details>

目前我們只有談到模組的共享，但有的時候我們也 [需要用到對方所設定的 Task 的結果](https://docs.gradle.org/current/userguide/declaring_dependencies_between_subprojects.html#sec:depending_on_output_of_another_project)。

#### 共享 Task 的結果

假設我們架構如下 :

```groovy
.
├── buildSrc
│   ├── build.gradle
│   ├── settings.gradle
│   ├── src
│   │   ├── main
│   │   │   └── java // (若是 kts 這檔案名稱會是 kotlin )
│   │   │       ├── BuildInfo.java
| ...
├── producer
│   ├── src
│   │   └──...
│   └── build.gradle
├── consumer
│   ├── src
│   │   └──...
│   └── build.gradle
└── settings.gradle
```

這個範例共享的是一個叫做 **BuildInfo** 的 抽象 Task 類別。 他的用處是將 `version` 存放在一個 `outputFile` 中 :

```groovy
// buildSrc/src/main/java/BuildInfo.java
public abstract class BuildInfo extends DefaultTask {

    @Input
    public abstract Property<String> getVersion();

    @OutputFile
    public abstract RegularFileProperty getOutputFile();

    @TaskAction
    public void create() throws IOException {
        Properties prop = new Properties();
        prop.setProperty("version", getVersion().get());
        try (OutputStream output = new FileOutputStream(getOutputFile().getAsFile().get())) {
            prop.store(output, null);
        }
    }
}
```

通過共用， `producer` 定義了一個叫做 `buildInfo` 的 Task。 他的作用是將 project 版本寫入 `generated-resources/build-info.properties` 中 :

```groovy
plugins {
    id 'java-library'
}

version = '1.0'

// BuildInfo Task
def buildInfo = tasks.register("buildInfo", BuildInfo) {
    version = project.version
    outputFile = layout.buildDirectory.file('generated-resources/build-info.properties')
}

sourceSets {
    main {
        output.dir(buildInfo.map { it.outputFile.asFile.get().parentFile })
    }
}
```

最後 `consumer` 為了要在 runtime 時讀取 `generated-resources/build-info.properties` 中的值，他需要使用 [JavaPlugin 的 Dependency configurations](https://docs.gradle.org/current/userguide/java_plugin.html#tab:configurations) 中的 `runtimeOnly` :

```groovy
dependencies {
    runtimeOnly project(':producer')
}
```

如果還想暸解其他共享成果的運用可以參考 [官方頁面](https://docs.gradle.org/current/userguide/cross_project_publications.html#cross_project_publications)。

## 何謂 Repository ?

Repository 即我們需要尋找 Plugin 或 Dependency 的遠端或本地位置。 Android 專案預設為 :

```groovy
repositories {
    google()
    mavenCentral()
}
```

一般而言，你會看到多個地方使用到 `repositories` block。

<details markdown=1>

<summary  markdown='span'> 
在新版中， Gradle 會在 <code>settings.gradle(Project)</code> 中的 <a href="https://docs.gradle.org/current/dsl/org.gradle.plugin.management.PluginManagementSpec.html#org.gradle.plugin.management.PluginManagementSpec"><code>pluginManagement</code></a> 與 <a href="https://docs.gradle.org/current/userguide/declaring_repositories.html#sub:centralized-repository-declaration"><code>dependencyResolutionManagement</code></a> 設定 repositories :
</summary>

```kotlin
pluginManagement {
    // repositories for Plugin
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
dependencyResolutionManagement {
    // repository for all projects (Centralizing repositories declaration)
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS /* make sure repository is based on settings only */ )
    repositories {
        google()
        mavenCentral()
    }
}
```

<br>
</details>

<details markdown=1>

<summary  markdown='span'> 
在舊的版本中， Gradle 會在 <code>build.gradle(Project)</code> 中設定 repositories :
</summary>

```groovy
buildscript {
    repositories {
        google()
        mavenCentral()
    }
}

allprojects {
    repositories {
        google()
        mavenCentral()
    }
}
```

</details>

### 指定遠端 Repositories

上面我們只是使用了 `google` 跟 `mavenCentral` 兩種函式庫管理者。 但如果我們想要使用別的管理者呢？ 像是公司內部的呢？那我們該怎麼做？

我們可以改用 `marven` 或 `ivy` 並提供相關的 url 、 username 與 password :

```groovy
repositories {
    maven {
        url "sftp://repo.mycompany.com:22/maven2" // 或 url "http://repo.mycompany.com/maven2"
        // optional : 有需要才用
        credentials {
            username "user"
            password "password"
        }
    }

    ivy {
        url "sftp://repo.mycompany.com:22/repo" // 或 url "http://repo.mycompany.com/maven2"
        credentials {
            username "user"
            password "password"
        }
    }
}
```

> 切記 : 每個 url 都需要單獨的 maven 或 ivy。

為了防止 url 找不到需要的 Jar，我們還可以 [指定替代 Jar 的位置](https://docs.gradle.org/current/userguide/declaring_repositories.html#sub:custom-maven-repo) :

```groovy
repositories {
    maven {
        // Look for POMs and artifacts, such as JARs, here
        url "http://repo2.mycompany.com/maven2"
        // Look for artifacts here if not found at the above location
        artifactUrls "http://repo.mycompany.com/jars"
        artifactUrls "http://repo.mycompany.com/jars2"
    }
}
```

除了上述的範例外，Gradle 還有更多的 [選項](https://docs.gradle.org/current/userguide/declaring_repositories.html#sec:authentication_schemes) 包括使用 Header、 AWS 、 不同的 authentication schemes 等等。

### 指定本地 Repositories

如果想要指定本地端的 Repository，我們可以使用 [**Flat Directory**](https://docs.gradle.org/current/userguide/declaring_repositories.html#sub:flat_dir_resolver)

```groovy
repositories {
    flatDir {
        dirs 'lib'
    }
    flatDir {
        dirs 'lib1', 'lib2'
    }
}
```

通過 `flatDir` 我們可以將本端的 libs。

> 留意 : 這個範例中 <code>lib</code> 的所在位置會在 `app/lib`

但如果在別的位置，你的 dirs 就不會是 `lib` 了。 譬如我們想要指定 [`myLibrary/libs/myaarfile.aar`](https://stackoverflow.com/questions/39079931/gradle-flatdir-path-to-module) :

```groovy
.
├── app
│   ├── src
│   │   └──...
│   └── build.gradle
├── myLibrary
|   ├── libs
|   |   └── myaarfile.aar
│   └── build.gradle
└── settings.gradle
```

此時，我們需要將 dirs 指向 `{root}/myLibary/libs`。 所以我們有以下方式做到 :

- 直接寫 `{path to your application}/{project name}/myLibrary/libs`
  但這個寫法並不好，因為路徑可能會被改變
- 使用 `${rootProject.projectDir}/myLibrary/libs` 取得專案路徑
- 使用 `${project(':mylipbrary').projectDir}/libs` 取得相同路徑

```groovy
repositories {
    flatDirs {
        dirs '${project(':mylipbrary').projectDir}/libs'
    }
}
```

最後 `myaarfile.aar` 便可以在 `build.gradle (Module)` 中的 `dependencies` 使用 :

```groovy
dependencies {
    ...
    implementation(name: 'myaarfile', ext: 'aar')
}
```

如果我們不需要在 `repositories` 中定義 `dirs` ，我們還可以 [直接在 `dependencies` 中直接定義](https://blog.csdn.net/fengyulinde/article/details/79989813) :

```groovy
dependencies {
    ...
    implementation fileTree(dir: '${project(':mylipbrary').projectDir}/libs', include: ['myaarfile.aar'])
}
```

## 暸解 Dependencies

當我們談到 Dependency 大家第一個會想到的應該就是以下的結構吧 :

```groovy
implementation "androidx.core:core-ktx:1.9.0"
classpath group: 'commons-codec', name: 'commons-codec', version: '1.2'
```

這個結構其實是通過不同元件組合起來的 :

> <code>{<i>configurationName</i>} '{group} : {name} : {version}' </code>

當我們定義 Dependency 時，我們其實是通過 [**DependencyHandler**](https://docs.gradle.org/current/dsl/org.gradle.api.artifacts.dsl.DependencyHandler.html) 來定義，並使用 [**Configuration**](https://docs.gradle.org/current/dsl/org.gradle.api.artifacts.Configuration.html) 來指定 dependencies 的 _使用範疇_， Gradle 稱他們為 **dependency configurations**。

Gradle 本身會有一些 buildin 的 Configuration，但我們需要通過 `plugin` 讓我們可以使用，譬如 :

```groovy
plugins {
    id 'java' // 這是 JavaPlugin 我們也可以通過他的子類別 'application' 或 'groovy' 來使用相同 Configuration
}
```

這是因為 `java` 內就有預設的 Configurations 像是 `implementation` 、 `runtimeOnly` 、 `compileOnly` 等等的 [JavaPlugin Configuration](https://docs.gradle.org/current/userguide/java_plugin.html#tab:configurations) 。

通過 Configuration ， Gradle 便知道在什麼情況之下使用哪些 Dependencies 來建立專案 :

<center>
    <a href = https://docs.gradle.org/current/userguide/declaring_dependencies.html"><img src = "https://docs.gradle.org/current/userguide/img/dependency-management-configurations.png"/></a>
</center>

相同的，在 Android 專案中的 `build.gradle (Module)` 也有類似的 plugin : [`com.android.application`](https://mvnrepository.com/artifact/com.android.application/com.android.application.gradle.plugin)

如果我們將 `com.android.application` 註解掉，同步之後 AS 便會出現錯誤訊息，告訴我們 `implementation`, `testImplementation`, `androidTestImplementation` 等等 Configuration 皆無法解析。 甚至 `android` block 也不知何物。 如果我們將 `org.jetbrains.kotlin.android` 移除，AS 也無法找到 `kotlinOptions`。

從這些結果，我們就大致知道這些 plugins 負責哪些 Configuration。

> 我原本想要追源碼的，但花了一整天都找不到，所以就用反証法了。 想要試圖追源碼的可以看 <a href="https://android.googlesource.com/platform/tools/build/+/master/gradle/src/main/groovy/com/android/build/gradle/AppPlugin.groovy">這裡</a> 。 另外，<code>com.android.application</code> 名稱是從 <a href="https://android.googlesource.com/platform/tools/base/+/master/build-system/gradle/src/main/resources/META-INF/gradle-plugins/com.android.application.properties"><code>com.android.application.properties</code></a> 中定義。

接下來我們會看看什麼是 **Configurations** 以及目前有哪幾種 **Dependencies**。

### 何謂 Configuration ?

> A <a href="https://docs.gradle.org/current/javadoc/org/gradle/api/artifacts/Configuration.html"><b>Configuration</b></a> represents <i>a group of artifacts and their dependencies</i>. They are a fundamental part of dependency resolution in Gradle.
>
> 補充 : artificate is file or directory produced by a build, such as a JAR, a ZIP distribution, or a native executable.

從源碼上， [Configuration](https://docs.gradle.org/current/javadoc/org/gradle/api/artifacts/Configuration.html) 是一個介面，而且他還是一個 [FileCollection](https://docs.gradle.org/current/javadoc/org/gradle/api/file/FileCollection.html)。 他的主要功能就是存放所需要的 dependencies 但不會存放 artifacts。而這些 configurations 都會被存放在 Project 的 ConfigurationContainer 中。

如果我們在 `build.gradle (Project)` 中加上這段 :

```kotlin
tasks.register("showConfigs") {
    configurations.forEach {
        println(it.name)
    }
}
```

再通過 `gradle -q showConfigs` 我們便可以列出當下可以使用的 `configurations` :

```shell
> gradle -q showConfigs
androidApis
androidJdkImage
androidTestAnnotationProcessor
androidTestApi
androidTestApiDependenciesMetadata
androidTestCompileOnly
androidTestCompileOnlyDependenciesMetadata
androidTestDebugAnnotationProcessor
...
implementation
...
default
...
```

但更好的作法就是在 terminal 輸入 :

```shell
./gradlew app:dependencies
```

這個命令會顯示 `app` 中的 configurations 與對應的 dependencies :

```shell
------------------------------------------------------------
Project ':app'
------------------------------------------------------------

androidApis - Configuratio
...
androidTestImplementation - Implementation only dependencies for 'androidTest' sources. (n)
+--- androidx.test.ext:junit:1.1.5 (n)
\--- androidx.test.espresso:espresso-core:3.5.1 (n)
...

```

> 切記 : 這裡並不會顯示你在 <code>build.gradle(Project)</code> 中定義的 Configuration。

#### Configuration 的角色

Configuration 共有 **四種角色** 。 而扮演哪個角色則取決於 Configuration 中的兩個變數 `isCanBeResolved` 與 `isCanBeConsumed`。 而 Android 會使用 [DefaultConfiguraton](https://chromium.googlesource.com/external/github.com/gradle/gradle/+/refs/tags/v5.1.0-RC1/subprojects/dependency-management/src/main/java/org/gradle/api/internal/artifacts/configurations/DefaultConfiguration.java) 來設定預設角色。

> <code>isCanBeResolved</code> true if it is allowed to query or resolve this configuration.
> <code>isCanBeConsumed</code> true if this configuration can be consumed from another project, or published.
> Android 的預設是兩個都是 true，但這會讓 Configuration 被視作 Legacy 而不該被使用。

| Configuration role                                           | `isCanBeResolved` | `isCanBeConsumed` | Android 範例                                                                                                              | 作用                                                                              |
| :----------------------------------------------------------- | :---------------: | :---------------: | :------------------------------------------------------------------------------------------------------------------------ | :-------------------------------------------------------------------------------- |
| Dependency Scope <br>(一般角色)                              |       false       |       false       | `implementation`<br> `compileOnly`<br> `runtimeOnly`<br> `api`                                                            | 這角色只是用來定義 Configuration 所需要的 dependencies                            |
| Resolve for certain usage <br>(消費者 、 app 方 或 INCOMING) |       true        |       false       | `debugAndroidTestCompileClasspath`<br>`debugAndroidTestCompileOnlyDependenciesMetadata`<br>`debugApiDependenciesMetadata` | 這角色的主要功能是在所在的 Project 內進行某些分析。<br>它的行為會由 Task 來實作。 |
| Exposed to consumers <br>(供應者、 lib 方 或 OUTGOING)       |       false       |       true        | [`default`](https://stackoverflow.com/a/19941894/18597115)<br>`releaseApiElements`                                        | 這個 Configuration 不是用在所在的 Project 中，而是為了讓其他 Project 使用的。     |
| Legacy, don’t use                                            |       true        |       true        | `compileClasspath` <br>我們創建的 configurations                                                                          | Android 中的預設角色                                                              |

<details markdown = 1>
<summary markdown = 'span'>
我們也可以在 <code>DefaultProjectDependency.java</code> 中看到 Gradle 在 <code>findProjectConfiguration</code> 時，如果遇到 <code>isCanBeConsumed() == false</code> 時就會拋出 Exception。</summary>

```java
// package org.gradle.api.internal.artifacts.dependencies
// DefaultProjectDependency
@Override
public void resolve(DependencyResolveContext context) {
    boolean transitive = isTransitive() && context.isTransitive();
    if (transitive) {
        Configuration projectConfiguration = findProjectConfiguration();
        for (Dependency dependency : projectConfiguration.getAllDependencies()) {
            context.add(dependency);
        }
        for (DependencyConstraint dependencyConstraint : projectConfiguration.getAllDependencyConstraints()) {
            context.add(dependencyConstraint);
        }
    }
}

@Override
public Configuration findProjectConfiguration() {
    ConfigurationContainer dependencyConfigurations = getDependencyProject().getConfigurations();
    String declaredConfiguration = getTargetConfiguration();
    Configuration selectedConfiguration = dependencyConfigurations.getByName(GUtil.elvis(declaredConfiguration, Dependency.DEFAULT_CONFIGURATION));

    // 確保 configuration 是可被 consumed ... 供應者
    if (!selectedConfiguration.isCanBeConsumed()) {
        throw new ConfigurationNotConsumableException(dependencyProject.getDisplayName(), selectedConfiguration.getName());
    }
    warnIfConfigurationIsDeprecated((DeprecatableConfiguration) selectedConfiguration);
    return selectedConfiguration;
}
```

<br>
</details>

> 如果你有興趣也可以自己去追喔～ 我花太多時間在這了。

我們可以從 [官方範例](https://docs.gradle.org/current/userguide/declaring_dependencies.html#sec:resolvable-consumable-configs) 暸解這些角色是如何被定義的。

<details markdown=1> 
<summary markdown='span'> 第一步，我們先定義 Configuration 與某項目的依賴性 </summary>

```groovy
// 創建一個叫做 "someConfiguration" 的 "configuration"。
// 由於 Android 中的 default Configuration 是 Legacy，所以我們要設定正確的 isCanBeResolved 與 isCanBeConsumed
// kts :
// val someConfiguration = create("someConfiguration") {
//      isCanBeResolved = false
//      isCanBeConsumed = false
// }
configurations {
    someConfiguration  {
        // 因為 Android 的預設為 true, true，所以我需要手動修改
        setCanBeResolved(false)
        setCanBeConsumed(false)
    }
}

dependencies {
    // 這只設定 someConfiguration 依賴著 lib
    // 但並沒有說明 someConfiguration 該如何使用
    someConfiguration project(":lib")
}
```

<br>
</details>

<details markdown=1> 
<summary markdown='span'> 第二步，創建兩個繼承 <code>someConfiguration</code> 的 configurations。 這兩個 configurations 會與 someConfiguration 一樣。 唯一不同的是他們的名稱讓使用者知道它的作用。 </summary>

```groovy
    // 創建一個叫做 "someConfiguration" 的 "configuration"
    // kts : val someConfiguration by configurations.creating
    configurations {
        someConfiguration
    }

    dependencies {
        someConfiguration project(":lib")

        // declare a configuration that is going to resolve the compile classpath of the application
        compileClasspath.extendsFrom(someConfiguration) {
            setCanBeResolved(true)
        }
        // declare a configuration that is going to resolve the runtime classpath of the application
        runtimeClasspath.extendsFrom(someConfiguration) {
            setCanBeResolved(true)
        }

        /** kts 寫法
        * create("compileClasspath") {
        *    extendsFrom(someConfiguration)
        *    isCanBeResolved = false
        * }
        */
    }
```

<br>
</details>

此時 `someConfiguration`、 `compileClassPath` 與 `runtimeClasspath` 都會有基本設定 :

- `isCanBeResolved = true`
- `isCanBeConsumed = false`

以上就是 `consumer` 或 app 的 Configuration 設定。 接下來就是要看 `lib` 是如何為我們提供功能。

<details markdown=1>
<summary markdown='span'> 在 <code>lib</code> 中，我們把想要讓他人使用的 Configuration 的 <code>canBeResolved</code> 設為 <code>false</code> 並將 <code>canBeConsumed</code> 設為 <code>true</code> 即可。
</summary>

```java
configurations {
    // A configuration meant for consumers that need the API of this component
    exposedApi {
        // This configuration is an "outgoing" configuration, it's not meant to be resolved
        canBeResolved = false
        // As an outgoing configuration, explain that consumers may want to consume it
        assert canBeConsumed
    }
    // A configuration meant for consumers that need the implementation of this component
    exposedRuntime {
        canBeResolved = false
        assert canBeConsumed
    }
}
```

<br>
</details>

現在知道 Configuration 是如何創建與定義了，接下來我們看看 Dependencies 吧。

### 有哪些 Dependencies ?

Dependencies 其實有很多種類，包括 :

- [**Module Dependencies** (模組依賴)](https://docs.gradle.org/current/userguide/declaring_dependencies.html#sub:module_dependencies)
  Gradle 會從 Repository 中尋找 `.module`, `.pom` or `ivy.xml` 檔案。 一旦找到，Gradle 便會對他進行解析，並下載他所需要的 Artifacts (`.jar` 或 `.zip`) 以及 Dependencies。
  如果沒找到，你也可以通過 [metadata (元數據)](https://docs.gradle.org/current/userguide/declaring_repositories.html#sec:supported_metadata_sources) 指定 repository。
  > 元數據會紀錄的資訊包括 : 傳遞依賴 (transition) 、 代碼來源、作者訊息 等等
- [**File Dependencies** (文件依賴)](https://docs.gradle.org/current/userguide/declaring_dependencies.html#sub:file_dependencies)
  有時我們並不會依賴 Binary Dependency，而是使用或依賴專案或地區網內的文件。 由於他們存在在專案中，所以他們也不需要元數據。
- [**Project Dependencies** (項目依賴)](https://docs.gradle.org/current/userguide/declaring_dependencies.html#sub:project_dependencies)
  當我們專案中有多個項目時，我們也可以對他們依賴。
- [**Gradle distribution-specific Dependencies** (Gradle 内嵌依赖)](https://docs.gradle.org/current/userguide/declaring_dependencies.html#sub:gradle_distribution_dependencies)
  如我你想要客製化 Gradle tasks 或 plugins，我們可以將 Gradle API 通過依賴方式傳入。

這是因為我們可以從不同地方取得和運用。

#### Module Dependencies

模組依賴都是從 Repository 中取得的模組。 我們可以通過 [DependencyHandler](https://docs.gradle.org/current/dsl/org.gradle.api.artifacts.dsl.DependencyHandler.html) 來暸解如何定義他們。

另外，我們還可以通過 [**ExternalModuleDependency**](https://docs.gradle.org/current/javadoc/org/gradle/api/artifacts/ExternalModuleDependency.html) 來對他們進行設定。

```groovy
dependencies {
    // 我們可以用字串 或 字典方式進行定義
    runtimeOnly group: 'org.springframework', name: 'spring-core', version: '2.5'
    runtimeOnly 'org.springframework:spring-core:2.5',
            'org.springframework:spring-aop:2.5'

    // 還可以將多個模組用陣列定義
    runtimeOnly(
        [group: 'org.springframework', name: 'spring-core', version: '2.5'],
        [group: 'org.springframework', name: 'spring-aop', version: '2.5']
    )

    // 若想要設定，我們也可以通過 block 或 字典進行設定。
    runtimeOnly('org.hibernate:hibernate:3.0.5') {
        transitive = true
    }
    runtimeOnly group: 'org.hibernate', name: 'hibernate', version: '3.0.5', transitive: true
    runtimeOnly(group: 'org.hibernate', name: 'hibernate', version: '3.0.5') {
        transitive = true
    }
}
```

#### File Dependencies

當 Gradle 需要文件依賴時，他可以從專案中 (Local File System) 或 地區網路 (Shared Drive) 中取得並存放在 Gradle Cache 中 :

<center>
    <a href = "https://docs.gradle.org/current/userguide/declaring_dependencies.html#sub:file_dependencies"><img src = "https://docs.gradle.org/current/userguide/img/dependency-management-file-dependencies.png"/></a>
</center>

以下是官方範例 :

```groovy
// 創建多個 Configuration
configurations {
    antContrib
    externalLibs
    deploymentTools
}

// 通過 files 或 fileTree 取得所需要的 Dependencies
// Gradle 提醒 : 盡量使用 files 而不是 fileTree
dependencies {
    antContrib files('ant/antcontrib.jar') // 位於專案中的檔案
    externalLibs files('libs/commons-lang.jar', 'libs/log4j.jar') // 多個檔案
    deploymentTools(fileTree('tools') { include '*.exe' }) // 相當於 fileTree(dir: 'tools', include : [*.exe])

    // 其他 plugin 中提供的 Configuration
    runtimeOnly files('libs/a.jar', 'libs/b.jar')
    runtimeOnly fileTree('libs') { include '*.jar' }
}
```

我們甚至可以通過 Task 創建這些文件依賴 :

```groovy
dependencies {
    // 文件依賴會位於 buildDirectory 的 'classA' 中 (build/classA)
    implementation files(layout.buildDirectory.dir('classA')) {
        // 指定 'myCompile' Task 來建立文件
        builtBy 'myCompile'
    }
}

// 這裡你可以進行檔案的創建或取得
// 想知道更多如何在 Gradle 中使用檔案的話，可以去 https://docs.gradle.org/current/userguide/working_with_files.html 看看。
// 這裡
tasks.register('myCompile') {
    doLast {
        println 'compiling classes'
    }

    /**
    // 範例 : 如果是複製檔案，我們可以讓 task 繼承 Copy，並定義其行為 :
    from layout.buildDirectory.file("reports/my-report.pdf")
    into layout.buildDirectory.dir("classes")
    */

    /**
    // 範例 : 如果想要創建檔案
    // https://stackoverflow.com/questions/52972830/programmatically-create-a-file-in-gradle-build-script
    doLast {
        // 這裡會創建 build/classA
        def resourcesDir = "$buildDir/classA" // sourceSets.main.output.resourcesDir // 會創建 build/resources
        new File(resourcesDir).mkdirs() // 如果是使用 sourceSets.main.output.resourcesDir 這裡改成 resourcesDir.mkdirs()
        def contents = "projectInfo.project=$project.name"
        new File(resourcesDir, "project-info.properties").text = contents
        println 'compiling classes'
    }
    */
}

tasks.register('list') {
    // 取得 Configurations 中的編譯類 (compile) 路徑 (https://juejin.cn/s/gradle%20configurations.compileclasspath)
    // Android 中並沒有 compileClasspath 所以會出錯。
    FileCollection compileClasspath = configurations.compileClasspath
    // 讓編譯類跑一遍，通過這裡才會調用 `myCompile`
    dependsOn compileClasspath
    // 最後你會發現這裡的 classPath 除了原本的還會有 build/classA
    // build/classA 只有當 implementation files(layout.buildDirectory.dir('classA')) 存在時才會出現
    doLast {
        println "classpath = ${compileClasspath.collect { File file -> file.name + ' : ' +  file.path + '\n' }}"
    }
}
```

通過 `gradle -q list` 我們便可看到 :

```shell
$ gradle -q list
compiling classes
classpath = [classes]
```

#### Project Dependencies

假設我們有多個項目，而且他們之間會有依賴關係 :

<center>
<a href = "https://docs.gradle.org/current/userguide/declaring_dependencies.html#sub:project_dependencies"><img src = "https://docs.gradle.org/current/userguide/img/dependency-management-project-dependencies.png"/></a>
</center>

我們該如何設定呢？

在 `Project A` 中，因為我們只需要 `Project B`，所以在 `dependencies` 中我們可以寫 :

```groovy
// build.gradle (Project A)
dependencies {
    implementation project(':projectB')
}
```

而 `Project C` 則需要依賴 `Project A` 與 `Project B` :

```groovy
// build.gradle (Project C)
dependencies {
    implementation project(':projectA') // Gradle 7 之後可以寫 projects.projectA
    implementation project(':projectB')
}
```

> 我原本以為 Project C 只需要依賴 Project A 即可。 但因為如果 Project C 沒有將對 Project B 的依賴告知 Gradle ， Gradle 並不會知道原來 Project C 也依賴著 Project B。 另一個更好的範例可參看 : <a href = "https://docs.gradle.org/current/userguide/declaring_dependencies_between_subprojects.html#declaring_dependencies_between_subprojects">Declaring Dependencies between Subprojects</a> 。

#### Gradle distribution-specific dependencies

想要使用 Gradle API 來客製化 Gradle tasks 或 plugins，我們可以在 `dependencies` 中設定 :

```groovy
dependencies {
    implementation gradleApi() // 用來創建 tasks 或 plugin
    implementation localGroovy() // 用來創建 tasks 或 plugin 但是特別使用 Groovy DSL 的
    testImplementation gradleTestKit() // 測試 Gradle plugins 和 build script
}
```

> 如果對測試 Gradle plugins 或 build script 有興趣可以看看 <a href="https://docs.gradle.org/current/userguide/test_kit.html#test_kit">這頁</a>。

### 如何指定 Dependencies Version ?

Dependency Version 有以幾種 [定義方法](https://docs.gradle.org/current/userguide/single_versions.html) :

1. 特別指定一種
   ie `1.3`, `1.3.0-beta3` , `1.0-20150201.131010-1`
2. 指定範圍
   ie `[1.0,)` , `[1.1, 2.0)` , `(1.2, 1.5]` , `]1.0, 2.0[`, `[1.0, 2.0[`
   `[]` : inclusive (包含)
   `()` : exclusive (不包含)
3. prefix
   ie `1.+`, `1.3.+`
4. 某狀態最新的版本
   ie `latest.integration`, `latest.release`
   Gradle 會通過 [**ComponentMetadata**](https://docs.gradle.org/current/javadoc/org/gradle/api/artifacts/ComponentMetadata.html#getStatus--) 找到最新版本
5. SNAPSHOT
   ie `1.0-SNAPSHOT`, `1.4.9-beta1-SNAPSHOT`

我們還可以用不同的 keywoard 來定義我們限制的版本。 [以下就是使用多種方式來做相同的事](https://docs.gradle.org/current/userguide/single_versions.html#simple_version_declaration_semantics) :

```groovy
dependencies {

    // 範例 1 :
    implementation('org.slf4j:slf4j-api:1.7.15')
    // short-hand notation with !!
    implementation('org.slf4j:slf4j-api:1.7.15!!')
    // is equivalent to
    implementation("org.slf4j:slf4j-api") {
        version {
           strictly '1.7.15'
        }
    }

    // or 範例 2 :
    implementation('org.slf4j:slf4j-api:[1.7, 1.8[!!1.7.25')
    // is equivalent to
    implementation('org.slf4j:slf4j-api') {
        version {
           strictly '[1.7, 1.8['
           prefer '1.7.25'
        }
    }
}
```

我們還可以通過 **Dependency Constraints** 來設定版本 :

```groovy
// build.gradle
dependencies {
    implementation 'org.springframework:spring-web'
    constraints {
        implementation 'org.springframework:spring-web:5.0.2.RELEASE' {
            // 我們還可以記錄選擇這個版本的原因
            because 'previous versions have a bug impacting this application'
        }
    }
}
```

Dependency Constraints 除了能用來指定直接依賴的版本，還可以指定傳遞依賴 (transitive dependency) 的版本。這在多重專案中是一個很不錯的幫手。

#### 版本的順序

當我們定義出依賴所需要的版本範圍後，Gradle 會按照版本的順序來找出最符合的版本。而這個順序則會遵循 [以下規則](https://docs.gradle.org/current/userguide/single_versions.html#version_ordering) :

1. 版本參數之間可用 `[. - _ +]` 來分隔，也可用 `{數字}{英文}{數字}` 來進行分隔。
   `1.a.1 == 1-a+1 == 1.a-1 == 1a1`，若有疑慮請 [參考這頁](https://docs.gradle.org/current/userguide/dependency_resolution.html#sec:base-version-comparison) 。
2. 當比較兩個 prefix 一樣的版本時，數字較大的為較新，字母小寫的為較新，數字比字母較新，版本較長且為數字的的為較新 :
   `1.A < 1.B < 1.a < 1.b < 1.1 < 1.2.a < 1.2 < 1.2.0`
3. 若出現字串，他們的排序會無視大小寫，並以 a 最大 z 最小來排。但是也有特例 : `dev` 會被視作最小。 另外， `rc`, `snapshot`, `final`, `ga`, `release` 和 `sp` 則會無視大小，都會按這個順序被視作最大。
   `1.0-dev < 1.0-zeta < 1.0-snapshot < 1.0-rc < 1.0-final < 1.0-ga < 1.0-release < 1.0-sp < 1.0`

### 如何在項目中共享 Dependencies ?

共享依賴，我們有兩種方式 :

1. [buildSrc](#buildsrc)
   這個方法會通過創建另一個 `buildSrc` 模組來分享給其他項目使用。 主要缺點是每次 `buildSrc` 中有更改，全部的項目都需要重新建立一次，因為他會將 cache 無效化。除此之外， `buildSrc` 必須放置在目標項目的 root 。
2. version catalog (Preferred)
   與其繼承 build cache ，Gradle 推薦的方式是 **version catalog**。

#### buildSrc

一般擁有 [`buildSrc`](https://docs.gradle.org/current/userguide/organizing_gradle_projects.html#sec:build_sources) 的專案中有以下 [結構](https://docs.gradle.org/current/userguide/organizing_gradle_projects.html#sec:build_sources) :

```groovy
.
├── buildSrc                                            // -------------------- Logic
│   ├── build.gradle                                    //                        |
│   └── src                                             //                        |
│       ├── main                                        //                        |
│       │   └── java                                    //                        |
│       │       └── com                                 //                        |
│       │           └── enterprise                      //                        |
│       │               ├── Deploy.java                 //                        |
│       │               └── DeploymentPlugin.java       //                        |
│       └── test                                        //                        |
│           └── java                                    //                        |
│               └── com                                 //                        |
│                   └── enterprise                      //                        |
│                       └── DeploymentPluginTest.java   // -------------------- Layer
├── settings.gradle
├── subproject-one
│   └── build.gradle
└── subproject-two
    └── build.gradle
```

然後我們可以在 Gradle 中將 `buildSrc` 用 `includeBuild` 讓 Gradle 知道 :

```groovy
// settings.gradle
pluginManagement {
    includeBuild 'buildSrc'
}
```

通過 `includeBuild`， `buildSrc` 也會繼承上層的 [build cache](https://docs.gradle.org/current/userguide/build_cache.html)。 也是因為如此，所以對 `buildSrc` 的修改會導致整個 cache 無效，導致整個專案需要重新建立。

> A change in buildSrc causes the whole project to become out-of-date. Thus, when making small incremental changes, the <code>--no-rebuild</code> command-line option is often helpful to get faster feedback. Remember to run a full build regularly or at least when you’re done, though.

#### Version Catalog

[](https://github.com/android/nowinandroid/blob/main/gradle/libs.versions.toml)

### 針對依賴的補充說明

有的時候我們會因為某種原因而使用較舊版本的 Dependencies。 這時為了不讓自己或同事不小心改成錯誤的版本，Gradle 允許我們對 Dependency 作出補充說明 :

```groovy
dependencies {
    implementation('org.ow2.asm:asm:7.1') {
        because 'we require a JDK 9 compatible bytecode generator'
    }
}
```

這些補充說明可以通過 `gradle -q dependencyInsight --dependency asm` 列印出來 :

```shell
> gradle -q dependencyInsight --dependency asm
org.ow2.asm:asm:7.1
  Variant compile:
    | Attribute Name                 | Provided | Requested    |
    |--------------------------------|----------|--------------|
    | org.gradle.status              | release  |              |
    | org.gradle.category            | library  | library      |
    | org.gradle.libraryelements     | jar      | classes      |
    | org.gradle.usage               | java-api | java-api     |
    | org.gradle.dependency.bundling |          | external     |
    | org.gradle.jvm.environment     |          | standard-jvm |
    | org.gradle.jvm.version         |          | 11           |
   Selection reasons:
      - Was requested: we require a JDK 9 compatible bytecode generator

org.ow2.asm:asm:7.1
\--- compileClasspath

A web-based, searchable dependency report is available by adding the --scan option.
```

#### Direct 與 Transitive Dependencies

Dependencies 還可以分成兩種 :

1. 我們指定的 Dependencies，也被稱為 **Direct Dependencies**，因為他們是我們直接指定的 Dependencies。
2. Direct Dependencies 所需要的 Dependencies，我們稱之為 **Transitive Dependencies**，因為他們是從 Direct Dependencies 傳遞到項目中的。

這時我們可能會遇到 Dependencies 衝突的問題 :

```
dependencyA -> dependencyB_v1.0
dependencyC -> dependencyB_v1.1
```

這個問題有多種解決方案 :

- 設定 `transitive = false`
  這個作法必須要確定這個 transitive dependency 會在其他依賴中加入才行。
  [推薦文章](https://www.devsbedevin.net/android-understanding-gradle-dependencies-and-resolving-conflicts/)
- 設定 [resolveStragtegy](https://docs.gradle.org/current/userguide/resolution_rules.html)
  這個方法讓我們對 Configuration 進行設定。通過這些設定我們可以處理版本衝突與限制、模組的偏好與取代以及動態版本包存與更新的時間長度。
- 設定 **Dependency Constraints**
  通過設定 dependencies 中的 constraints，我們可以指定要用哪個版本。而且我們還可以使用 [Rich Versions](https://docs.gradle.org/current/userguide/rich_versions.html#rich-version-constraints) 像 `strictly`, `prefer` 與 `require` 關鍵詞來設定版本的寬鬆度。

<details markdown=1> 
<summary markdown='span'>
Transitive 需要在 dependencies 中使用。
</summary>

```groovy
dependencies {
    implementation ('dependencyA') {
        transitive = false
    }
}
```

<br>
</details>

<details markdown=1>
<summary markdown='span'>
<code>resolveStragtegy</code> 需要在 <code>configurations</code> 或 <code>pluginManagement</code> 中使用。
</summary>

```groovy
// https://docs.gradle.org/current/dsl/org.gradle.api.artifacts.ResolutionStrategy.html
configurations.all {
  resolutionStrategy {
    // 1. 一旦有衝突就會報錯
    //      其他類似方法 failOnDynamicVersions
    //
    // fail eagerly on version conflict (includes transitive dependencies)
    // e.g. multiple different versions of the same dependency (group and name are equal)
    failOnVersionConflict()

    // 2. 告知如果有衝突，請使用我們專案中的
    //
    // prefer modules that are part of this build (multi-project or composite build) over external modules
    preferProjectModules()

    // 3. 我們可以通過更改 Forced Modules 來指定需要強制使用的版本
    //
    // force certain versions of dependencies (including transitive)
    //  *append new forced modules:
    force 'asm:asm-all:3.3.1', 'commons-io:commons-io:1.4' // forcedModule = default force modules + asm + commons
    //  *Replaces existing forced modules with the input :
    forcedModules = ['asm:asm-all:3.3.1'] // forceModule = asm

    // 4. 讓本地項目與遠端模組進行替換
    //
    // add dependency substitution rules
    dependencySubstitution {
      substitute module('org.gradle:api') using project(':api') // 遇到 org.gradle:api 便使用本地的 :api
      substitute project(':util') using module('org.gradle:util:3.0') // 當需要本地的 :util 時，就使用遠端的 org.gradle:util:3.0
    }

    // 5. cacheDynamicVersionsFor 定義目前版本要保存多久 (預設是 24, 'hours')。 這期間的 sync 後的版本都會一樣。
    //    cacheChangingModulesFor 定義要多久之後會自動更新
    //
    //    PS : 這設定只針對動態版本 : 1.+
    //
    // https://www.jianshu.com/p/289354394328
    //
    // 如果兩個接設為 0，那就相當於這個指令 : gradle build --refresh-dependencies
    //
    // cache dynamic versions for 10 minutes
    cacheDynamicVersionsFor 10*60, 'seconds'
    // don't cache changing modules at all
    cacheChangingModulesFor 0, 'seconds'
  }
}
```

我們也可以寫 :

```groovy
resolutionStrategy.dependencySubstitution {
    substitute project(":api") using module("org.utils:api:1.3") because "we use a stable version of org.utils:api"
}
```

而針對我們的範例，我們可以寫 :

```groovy
configurations.all {
  resolutionStrategy {
    force 'dependencyB_v1.1' // 強制使用 v1.1 版本
  }
}
```

<br>
</details>

<details markdown=1>
<summary markdown='span'>
<code>constraints</code> 會在 <code>dependencies</code> block 中使用。
</summary>

```groovy
dependencies {
    implementation 'org.apache.httpcomponents:httpclient'
    constraints {
        implementation('org.apache.httpcomponents:httpclient:4.5.3') {
            because 'previous versions have a bug impacting this application'
        }
        implementation('commons-codec:commons-codec:1.11') {
            because 'version 1.9 pulled from httpclient has bugs affecting this application'
        }
    }
}
```

我們的範例可以通過以下方法來設定 :

```groovy
dependencies {
    implementation 'dependencyA'
    implementation 'dependencyC'

    constraints {
        implementation('dependencyB_v1.1') {
            because 'we prefer this version'
        }
    }
}
```

<br>
</details>

基本上這些就是 Gradle 可以如何處理 Direct 或 Transitive 依賴衝突的方法了。

#### 如何解析非 JAR Artifact ？

當我們在設定 Dependencies 時，Gradle 會試圖從 repositories 中取得 JAR 並進行解析。 但如果

1. 我們想要解析的不是 JAR 呢？ (ie ZIP)
2. metadata 中存在多個 artifacts 呢？
3. 你只想要取得某 artifact 但不想要他所需要的 dependencies 呢？ (也許其他 dependencies 存在其他版本相同的 artifacts)

對 Maven 而言，我們可以使用客製化的 Configuration :

```groovy
// ref : https://jiraaya.wordpress.com/2014/06/05/download-non-jar-dependency-in-gradle/
configurations {
 mrsWar
}

dependencies {
 mrsWar "org.openmrs.web:openmrs-webapp:${openmrsVersion}@war"
}

task downloadMRSWar {
   new File("${buildDir}/resources/main").mkdirs();
      configurations.mrsWar.resolve().each { file ->
       //Copy the file to the desired location
      }
   }
```

如果你使用的是 Ivy，那你可以使用他的 [`patternLayout`](https://ant.apache.org/ivy/history/master/concept.html#patterns) :

```groovy
repositories {
    ivy {
        url 'https://ajax.googleapis.com/ajax/libs'
        patternLayout {
            artifact '[organization]/[revision]/[module].[ext]'
        }
        metadataSources {
            artifact()
        }
    }
}

configurations {
    js
}

dependencies {
    js 'jquery:jquery:3.2.1@js' // {dependency’s coordinates} @ {artifact file extension}
}
```

如果想要針對特定的 Flavor，那還可以增加 `classifier` :

```groovy
repositories {
    ivy {
        url 'https://ajax.googleapis.com/ajax/libs'
        patternLayout {
            artifact '[organization]/[revision]/[module](.[classifier]).[ext]'
        }
        metadataSources {
            artifact()
        }
    }
}

configurations {
    js
}

dependencies {
    js 'jquery:jquery:3.2.1:min@js'
}
```

## 如何設定 Gradle Build 環境 ?

所謂 Build Environment 指的是在 Build 時的行為，像是 要跑哪個 Task、 連接 Proxy 所需要的使用者與密碼、Gradle 在 build 時使用的 JVM 設置 等等的設定。

我們可以通過以下 [四種方式](https://docs.gradle.org/current/userguide/build_environment.html) 進行設定，而他們的排序即他們執行的優先順序 :

| Order | Method                                                                                                              | 如何設定                                                                                                                                                                                                                                                                                                          |
| :---- | :------------------------------------------------------------------------------------------------------------------ | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1     | [指令](https://docs.gradle.org/current/userguide/command_line_interface.html)                                       | 終端機                                                                                                                                                                                                                                                                                                            |
| 2     | [系統屬性](https://docs.gradle.org/current/userguide/build_environment.html#sec:gradle_system_properties)           | Root 中的 [`gradle.properties`](<(https://docs.gradle.org/current/userguide/organizing_gradle_projects.html#declare_properties_in_gradle_properties_file)>)                                                                                                                                                       |
| 3     | [Gradle 屬性](https://docs.gradle.org/current/userguide/build_environment.html#sec:gradle_configuration_properties) | 這也是存在 `gradle.properties` 中，但這個檔案的位置可以是 : <br> - [GRADLE_USER_HOME](https://docs.gradle.org/current/userguide/directory_layout.html#dir:gradle_user_home)<br> - Root 內 <br> - [GRADLE_HOME](https://docs.gradle.org/current/userguide/build_environment.html#sec:gradle_environment_variables) |
| 4     | 環境變數                                                                                                            | 系統中                                                                                                                                                                                                                                                                                                            |

### 指令與 Gradle Build

如果你有在寫 Android， 你也許已經有看過這類的指令了 :

```shell
gradle build
gradle run
gradle check
gradle clean
```

這些方法都會遵循以下格式 :

```shell
gradle [taskName...] [--option-name...]
# or
gradle [--option-name...] [taskName...]

# 多個 Tasks
gradle [taskName1 taskName2...] [--option-name...]
```

除此之外，我們還可以通過以下範例做出其他設定 :

| 功能                  | 指令                                                                                                                                                                                       |
| :-------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 設定變數              | <code>gradle [...] --console=plain</code><br><code>gradle [...] --build-cache</code><br><code>gradle [...] --no-build-cache</code><br><code>gradle --help</code><br><code>gradle -h</code> |
| 設定客製化的變數      | <code>gradle exampleTask --exampleOption=exampleValue</code>                                                                                                                               |
| 調用 root 上的 Task   | <code>gradle :myTask</code>                                                                                                                                                                |
| 調用多重專案中的 Task | <code>gradle :subproject:taskName</code> 或 <br><code>gradle subproject:taskName</code><br><br> 如果我們在 root 上跑 <code>gradle test</code>，這會跑全部專案中的 test Task                |
| 禁止 Task 被調用      | <code>gradle dist --exclude-task test</code>                                                                                                                                               |
| 強制 Task 運行        | <code>gradle test --rerun-tasks</code><br><br>通過 <code>-rerun-tasks</code>，Gradle 不會檢查是否需要更新，便會強制執行 test。                                                             |
| 無視某 Task 的失敗    | <code>gradle test --continue</code>                                                                                                                                                        |

以上是其中一些用法，有興趣或有需要可以看[官方範例](https://docs.gradle.org/current/userguide/command_line_interface.html)。

### 系統變數

我們其實可以通過指令 `gradle build -D 或 --system-prop` 修改系統變數 :

```shell
gradle build -D gradle.wrapperUser=myuser -D gradle.wrapperPassword=mypassword
```

但我們也可以在 `gradle.properties` 中預設 :

```groovy
systemProp.gradle.wrapperUser=myuser
systemProp.gradle.wrapperPassword=mypassword
```

當然，這些設定變數也只是冰山一角，更多的可以去 [官方網站](https://docs.gradle.org/current/userguide/build_environment.html#sec:gradle_system_properties) 查看。

### gradle.properties

我們在 [如何設定 Gradle Build 環境 ?](#如何設定-gradle-build-環境) 的表格中提到 `gradle.properties` 可以被存放在多個地方。 但 Gradle 只會使用其中一個而已。 Gradle 會按以下順序尋找，第一個被找到的就會被使用 :

| 優先順序 | 所在位置                                                                              |
| :------: | :------------------------------------------------------------------------------------ |
|    1     | 指令<br> 通過 -D 設置                                                                 |
|    2     | GRADLE_USER_HOME<br><br>這個資料夾的位置可以通過 <code>-Dgradle.user.home</code> 設定 |
|    3     | 專案資料夾 <br>然後 父專案資料夾 <br> 最後 Root                                       |
|    4     | Gradle installation directory                                                         |

我們可以在 Android 的 `root/gradle.properties` 看到一些設定 :

```groovy
org.gradle.jvmargs=-Xmx1536m // 設定 Gradle 運行在 JVM 最大分配的內存為 1536 MB
                            // -Xms1536m 則是最小為 1536 MB
                            // -Xmn1536m 則是設定 new generation 內存，越大越少 GC
                            // https://blog.csdn.net/cm_pq/article/details/120183572

android.useAndroidX=true    // 告知 Gradle Android Plugin 會用 AndroidX，而非 Support 函式庫。預設為 false。
                            // https://developer.android.com/jetpack/androidx
android.enableJetifier=true // 這如同 userAndroiX，但是針對的是第三方函式庫。 預設為 false。

kotlin.code.style=official  // 使用官方風格，也可使用 obsolete 來無視風格。
kapt.incremental.apt=true   // 讓使用 kapt 升級的 Library 像是 Room 不會每次都完整的 recompile。
                            // 這樣便有效加速整個程式開發。
                            //
                            // 以下供參考 :
                            // https://docs.gradle.org/current/userguide/java_plugin.html#sec:incremental_annotation_processing
                            // https://kotlinlang.org/docs/kapt.html#incremental-annotation-processing
                            // https://medium.com/@daniel_novak/making-incremental-kapt-work-speed-up-your-kotlin-projects-539db1a771cf
```

除了以上的範例外，你也可以去 [官方網站](https://docs.gradle.org/current/userguide/build_environment.html#sec:gradle_configuration_properties) 去尋找其他的參數喔。

# Android 中的 Gradle

上面我們談完了 Gradle 的一些「小小小」常識，接下來就看看 Android 中 Gradle 的架構與運作吧。

每次創建新的 Android 專案中，你會發現他會自動幫我們建立一系列的檔案。 這裡面有些檔案是 Android IDE 提供的，有些則是由 Gradle 幫我們創建的。

其中我們最常見的 Gradle 檔案有以下這些 :

| File                                        | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| :------------------------------------------ | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `.gitignore`                                | Git 不會上傳檔案類別與名稱                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| `gradlew`<br>`gradlew.bat`                  | Unix 與 Window 的 start script                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| `\gradle/wrapper/gradle-wrapper.jar`        | 內有 Gradle Wrapper 需要的類別檔案， 他會通過 gradlew 啟動。                                                                                                                                                                                                                                                                                                                                                                                                      |
| `/gradle/wrapper/gradle-wrapper.properties` | 內部有 Gradle Wrapper 的設定，像是需要哪個 Gradle 的版本來建立項目                                                                                                                                                                                                                                                                                                                                                                                                |
| `settings.gradle`                           | 這個檔案有 [三個作用](https://docs.gradle.org/current/userguide/organizing_gradle_projects.html#sec:settings_file) : <br> 1. 每次 build 執行時，為了尋找 `setttings.gradle` Gradle 都會走過每個檔案直到找到為止。 所以為了更流暢的 build， Gradle 會建議至少放一個空的 `settings.gradle` 在 root 中。<br> 2. 在 multi-project (多重專案) build 中，他是用來定義參與的專案有哪些。因此他是必須有的檔案。<br> 3. 除了定義專案外，他還可以設定有哪些函式庫會被用到。 |
| `build.gradle` (Project)                    | 其實這檔案 [可有可無](https://docs.gradle.org/current/userguide/intro_multi_project_builds.html#sec:project_structure) 的，我們甚至可以把這裡的設定寫在 `settings.gradle` 中。 但這個檔案會在 build 時第二個被執行的，所以我們也可以把他視為一個設定模組們共用的 `plugins`, `buildscript`, `dependencies` ...。                                                                                                                                                   |
| `build.gradle` (Module: app)                | 這是 Android 模組中的核心。 裡面會定義這個模組的 configuration, `plugins`, `dependencies`, `buildType` ...                                                                                                                                                                                                                                                                                                                                                        |

我們在 [Gradle Build 生命週期](#build-lifecycle) 中有提到 Gradle build 時執行的順序。若想要手動驗證，我們只需要在他們裡面寫 `println {msg}` 或 Kotlin `println({msg})` 即可。

<details markdown=1>

<summary  markdown='span'> 這是我的測試結果 </summary>

```shell
Hello Settings!

> Configure project :
Hello Project Gradle!

> Configure project :app
Hello Module Gradle!
```

<br>
</details>

其實大部分的東西我們都已經在第一節中介紹過，接下來我們就簡單地聊聊 `settings.gradle` 的架構吧。

### settings.gradle

舊版的 `settings.gradle (old)` 預設很簡單，他只會告知 Gradle 目前只有一個叫做 `app` 的模組需要被 build。

```groovy
include ':app'
```

新版的 `settings.gradle (new)` 豐富多了，除了有 `include` 外，還設定了 `rootProject.name` (預設名稱與 `@string/app_name` 一樣)。 最特別的是多了 `pluginManagement` 與 `dependencyResolutionManagement`。

```kotlin
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}

rootProject.name = "SpinnerDemo"
include(":app")
```

你應該會注意到 `pluginManagement` 與 `dependencyResolutionManagement` 與舊版的 `build.gradle (Project)` 中的 `buildscript` 與 `allprojects` 為很像 :

```groovy
buildscript {
    // ...
    repositories {
        google()
        mavenCentral()
    }
    // ...
}

allprojects {
    repositories {
        google()
        mavenCentral()
    }
}
```

但其實他們的作用也差不多。

#### pluginManagement

`pluginManagement` 是用來管理 Project 與 subProject 所需要的 plugins。 也因此，他必須要在整個 build 開始前就準備好。 所以他只能存在 `settings.gradle` 或 `init.gradle` 中。

> 為什麼會有 sub-project ? 這個問題可以由 Google <a href="https://developer.android.com/topic/modularization"  target="_blank">官方回答</a> 。

以下是 `settings.gradle` 與 `init.gradle` 中的寫法 :

```groovy
// settings.gradle
pluginManagement {
    plugins {
    }
    resolutionStrategy {
    }
    repositories {
    }
}

// init.gradle
settingsEvaluated { settings ->
    settings.pluginManagement {
        plugins {
        }
        resolutionStrategy {
        }
        repositories {
        }
    }
}
```

如果我們點進去看 `pluginManagement`，你會發現它其實是 **Settings** 或 **KotlinSettingsScript** 的一個方法 :

```java
void pluginManagement(Action<? super PluginManagementSpec> var1);
```

以及 Kotlin DSL :

```kotlin
open fun pluginManagement(@Suppress("unused_parameter") block: PluginManagementSpec.() -> Unit): Unit
```

這裡顯示我們 block 中的設定都是在設定 [**PluginManagementSpec**](https://docs.gradle.org/current/dsl/org.gradle.plugin.management.PluginManagementSpec.html)。

從 API 中，我們能設定的包括 :

| API                  | 作用                                                                                                            |
| :------------------- | :-------------------------------------------------------------------------------------------------------------- |
| `includeBuild`       | 定義要將某資料夾涵蓋在 build 裡面。 這通常是在多重專案時用到: [<code>includeBuild 'buildsrc'</code>](#buildsrc) |
| `repositories`       | 定義要 [從哪裡取得 plugins](#repositories)                                                                      |
| `resolutionStrategy` | 定義 [plugin resolution rules](#pluginresolutionstrategy)                                                       |
| `plugins`            | 設定 [plugins](#plugins)                                                                                        |

# Version Catalog : TOML

我們在 [version catalog](#version-catalog) 中有提到如何在 `build.gradle` 中定義我們的 version catalog。 但你知道其實我們也可以通過別的檔案來定義嗎？

[**TOML**](https://toml.io/en/) aka Tom's Obvious Minimal Language， 他是一種使用簡單明瞭的寫法來創建我們所需要的 Hash Table。

以下是其中一個範例 :

```
# This is a TOML document

title = "TOML Example"

[owner]
name = "Tom Preston-Werner"
dob = 1979-05-27T07:32:00-08:00

[database]
enabled = true
ports = [ 8000, 8001, 8002 ]
data = [ ["delta", "phi"], [3.14] ]
temp_targets = { cpu = 79.5, case = 72.0 }

[servers]

[servers.alpha]
ip = "10.0.0.1"
role = "frontend"

[servers.beta]
ip = "10.0.0.2"
role = "backend"
```

當然，我不會在這說明如何定義參數，這我們可以直接參考 [Android 官方的寫法](https://github.com/android/architecture-samples/blob/main/gradle/libs.versions.toml)。

我想要說的是要如何在專案中使用 TOML。

我們只需要在 `root/gradle` 中創建 `lib.versions.toml` 並定義我們所需要的 versions、 dependencies 和 plugins。之後就可以在 `build.gradle` 或 `settings.gradle` 中調用。

```groovy
libs.servers.beta.ip // 便可取得上方的 ip = "10.0.0.2"
```

> 你可能會在想 <code>lib.versions.toml</code> 是什麼時候被 import 的呢？

這其實是因為 [Gradle 自動就會去讀取](https://docs.gradle.org/current/userguide/platforms.html#sub:conventional-dependencies-toml) 這個名稱的 TOML 檔案。

如果我們改了名字，那我們就需要 [手動引入](https://discuss.gradle.org/t/migrating-to-gradle-catalogs-from-buildsrc-in-android-studio-unknown-property-libraries/43729)。

```groovy
dependencyResolutionManagement {
    versionCatalogs {
        libs {
            from(files("../gradle/someother.toml"))
        }
    }
}
```

如果想知道 `lib.versions.toml` 被讀取後會發生什麼事，可以從這 [回答](https://stackoverflow.com/a/76170383/18597115) 開始。

# References

## BuildType

1. [Advanced Android Flavors](https://proandroiddev.com/advanced-android-flavors-part-1-building-white-label-apps-on-android-ade16af23bcf)

2. [codelab: Create different versions of your app using build variants](https://developer.android.com/codelabs/build-variants#0)

## Task

1. [Gradle Task Inputs and Outputs](https://gradlehero.com/gradle-task-inputs-and-outputs/)
2. [Gradle: What is the difference between classpath and compile dependencies?](https://stackoverflow.com/a/50441077/18597115)

## Plugins

1. [Streamline Android App Dependencies with buildSrc](https://blogs.halodoc.io/streamline-android-app-dependencies-with-buildsrc/)
2. [Using :buildSrc Kotlin Extensions From Groovy-based Gradle Build Scripts](https://engineering.premise.com/using-buildsrc-kotlin-extensions-from-groovy-based-gradle-build-scripts-eb79679ab0ff)
3. [Using :buildSrc Kotlin Extensions From Groovy-based Gradle Build Scripts Github Sample](https://github.com/premisedata/mobile-tutorials/tree/main/001-buildsrc-kotiln-extensions-in-groovy)

4. [Writing a Simple Plugin](https://docs.gradle.org/current/userguide/custom_plugins.html#sec:writing_a_simple_plugin)

5. [Use buildSrc to abstract imperative logic](https://docs.gradle.org/current/userguide/organizing_gradle_projects.html#sec:build_sources)

## Repository

1. [Declaring repositories](https://docs.gradle.org/current/userguide/declaring_repositories.html)

## Dependencies

1. [重磅 - Gradle 功能介绍](https://www.cnblogs.com/wellcherish/p/17071454.html)
2. [Chapter 51. Dependency Management](http://sorcersoft.org/project/site/gradle/userguide/dependency_management.html)
3. [Working With Files](https://docs.gradle.org/current/userguide/working_with_files.html)
4. [Android - Understanding and dominating gradle dependencies](https://www.devsbedevin.net/android-understanding-gradle-dependencies-and-resolving-conflicts/)
5. [依赖的更新与缓存](https://www.jianshu.com/p/289354394328)
6. [Manage Gradle version conflicts with resolution strategy](https://proandroiddev.com/manage-gradle-version-conflicts-with-strategy-611ac3f6ce19)

## Build Environment

1. [-Xmx512m -Xms256m -Xmn256m 都是什么意思](https://blog.csdn.net/cm_pq/article/details/120183572)
2. [kapt compiler plugin](https://kotlinlang.org/docs/kapt.html)
3. [Making incremental KAPT work (Speed Up your Kotlin projects!)](https://medium.com/@daniel_novak/making-incremental-kapt-work-speed-up-your-kotlin-projects-539db1a771cf)
4. [简单几招提速 Kotlin Kapt 编译](https://droidyue.com/blog/2019/08/18/faster-kapt/)
5. [Incremental Annotation Processing](https://docs.gradle.org/current/userguide/java_plugin.html#sec:incremental_annotation_processing)

## TOML

1. [TOML: The Future of Gradle Dependency Declarations](https://medium.com/@dawinderapps/toml-the-future-of-gradle-dependency-declarations-14b72676c71f)
