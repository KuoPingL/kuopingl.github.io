---
layout: post
title: "LFA 2 - Unit 與 UI 測試"
date: 2023-11-07 21:22:20  +0800
categories: [Learning From Android]
keywords: android, sample, architecture, learn, testing
---

# 序言

歡迎來到「跟著官方學程式」 ( Learning From Android, **LFA** ) 的第二章。

我們將從 [architecture sample views branch](https://github.com/android/architecture-samples/tree/views) 學習如何用 View ( 不是 Compose 喔 ) 來做到官方推薦的架構。

當然，官方並沒有一步一步地教大家如何寫這個 App，但我會從無到有來建立出來。 至於教大家嘛？ 看看是否有需要吧！ 哈哈哈

不過我會盡量將需要或不需要的資訊都寫出來，哈哈哈。

另外，這個系列不太會有圖檔。 因為我的目標是從官方的程式中有效且快速地複習與更新我的 Android 知識。希望能早日回到職場上。

這篇我們是基於 [上一篇](2023-09-26-lfa-architecture-1.md) 繼續看看官方是如何做測試的。

官方也有提供 [codelab](https://developer.android.com/codelabs/advanced-android-kotlin-training-testing-basics/) 可以讓大家跟著做喔。

# 暸解一下 Android 專案

Android 的專案中，一般會有三個 sources，分別是 ：

- **main**
  main 包含著 app 主要的源碼。 它會由各個 build variants 共享。
- **androidTest**
  這裡放的是 Instrumental Test。也就是需要用實體或虛擬機來跑的測試，像是 UI 測試。
- **test**
  這裡放的是本地端的測試， Unit Test。像是一般方法的測試。這些都不需要依賴 Android Framework 或 OS 。

## 針對性的 Implementation

由於 Android 有至少三種不同的 sources，我們也可以通過 Gradle 進行針對性地載入。
所以，載入方法也有分成三種：

- **implementation**
  這些函式庫會被大家所共用
- **androidTestImplementation**
  這些函式庫只能被 `androidTest` 所使用
- **testImplementation**
  這些則只能被 `test` 所使用

以下是建立專案時會自帶的基本 dependencies：

```groovy
// basic libraries
testImplementation("junit:junit:4.13.2")
androidTestImplementation("androidx.test.ext:junit:1.1.5") // AndroidX libraries ext
androidTestImplementation("androidx.test.espresso:espresso-core:3.5.1") //AndroidX libraries
```

# The Testing Pyramid

想要進行測試之前，我們需要暸解我們有 3 種不同的 **測試策略** 包括：

- **Scope** ： 覆蓋率
- **Speed** ：測試的速度
- **Fidelity** ： 測試的真實性

這三者之間各有取捨。如果我們希望測試能加速，那通常就會偏向失真。
為此，測試會分成三種：

- **Unit Test**：這裡通常只會測試單一類型的方法，像是 **ViewModel**、 **Utils** 或 **Repository**。可想而知，這些測試並不複雜，所以速度必然很快。但一般情況不會只會調用一個類別的方法，所以會導致失真。
  這些測試會放在 `test` 檔案中。

- **Integration Test**：這裡的測試會需要通過多個類別之間的配合來完成某個 feature (特點)。 像是 **ViewModel** 搭配 **Repository** 進行資料的更新。所以這些測試覆蓋率大、速度快也會較偏向真實狀況。
  按不同情況，這些測試會出現在 `test` 或 `androidTest` 中。

- **End to end tests (E2e)**：這裡的測試會針對多個特點 或 主要功能 一同測試。所以這些測試相對的慢，但卻是最接近真實情況。
  這些測試都會是 Instrumental Test，所以會寫在 `androidTest` 中。

這三種測試各有利弊，官方建議他們的比例為：

- **Unit Test** - 70%
- **Integration Test** - 20%
- **E2e** - 10%

<center>
<img src = "https://developer.android.com/static/codelabs/advanced-android-kotlin-training-testing-test-doubles/img/7017a2dd290e68aa_1920.png" style="width=0.7"/>
</center>

接下來我們來看看看如何做不同的測試吧。

# Codelab

首先，我們會看看 [codelab](https://developer.android.com/codelabs/advanced-android-kotlin-training-testing-basics/) 是如何實作測試的。

想要暸解 architecture sample 如何測試的可以自行研究。

請注意， codelab 中的源碼與 architecture sample 有所不同，所以測試的項目也會有所不同。

## UnitTest : StatisticsUtils

我們先測試 **StatisticsUtils** 的 `getActiveAndCompletedStats` ：

```kotlin
internal fun getActiveAndCompletedStats(tasks: List<Task>?): StatsResult {

    return if (tasks == null || tasks.isEmpty()) {
        StatsResult(0f, 0f)
    } else {
        val totalTasks = tasks.size
        val numberOfActiveTasks = tasks.count { it.isActive }
        StatsResult(
            activeTasksPercent = 100f * numberOfActiveTasks / tasks.size,
            completedTasksPercent = 100f * (totalTasks - numberOfActiveTasks) / tasks.size
        )
    }
}

data class StatsResult(val activeTasksPercent: Float, val completedTasksPercent: Float)
```

我們可以用右鍵點選這個方法，並選擇 `generate` \> `Test`。 直接點選 OK 並挑選 `/app/src/test/...` 。
如此一來 AS 就會幫我們創建一個位於 `test` 的測試檔。

現在我們在裡面寫入：

```kotlin
class StatisticsUtilsTest {
    @Test
    fun getActiveAndCompletedStats_noCompleted_returnsHundredZero() {
        // Create an active task
        val tasks = listOf<Task>(
            Task("title", "desc", isCompleted = false)
        )

        // Call your function
        val result = getActiveAndCompletedStats(tasks)

        // Check the result
        assertEquals(result.completedTasksPercent, 0f)
        assertEquals(result.activeTasksPercent, 100f)
    }
}
```

這個測試很簡單，我們只是定義一個只有單一未完成的 `task` 的 **Task** 的舉列。
通過 `getActiveAndCompletedStats` 我們理應得到 `completedTasksPercent` 為 0f 而 `activeTasksPercent` 為 100f。

也許這個寫法並非那麼容易看得懂，我們可以搭配 [hamcrest](https://hamcrest.org/JavaHamcrest) 或是 [truth](https://truth.dev/)。

### 搭配 hamcrest

```groovy
val hamcrestVersion = "2.2"
testImplementation ("org.hamcrest:hamcrest:$hamcrestVersion")
testImplementation ("org.hamcrest:hamcrest-library:$hamcrestVersion")
```

然後我們就可以將：

```kotlin
assertEquals(result.completedTasksPercent, 0f)
assertEquals(result.activeTasksPercent, 100f)
```

改為：

```kotlin
assertThat(result.completedTasksPercent, `is`(0f))
assertThat(result.activeTasksPercent, `is` (100f))
```

要注意，後面是改為 `assertThat` 而不是 `assertEquals`。 另外，這是需要導入：

```kotlin
import org.hamcrest.Matchers.`is`
import org.junit.Assert.*
```

### 或搭配 truth

```groovy
testImplementation ("com.google.truth:truth:1.1.4")
```

然後我們便可以將上面的 code 改為：

```kotlin
assertThat(result.completedTasksPercent)
    .isEqualTo(0f)
assertThat(result.activeTasksPercent)
    .isEqualTo(100f)
```

要注意的是，現在的 `assertThat` 並非來自 **org.junit.Assert.\*** 而是來自 **com.google.common.truth.Truth.assertThat**。

所以我們的 import 也需要修改：

```kotlin
import com.google.common.truth.Truth.assertThat
```

## UnitTest : TasksViewModel

上面講的是針對方法進行測試，現在我們要談的是針對 **ViewModel** 的測試。

```kotlin
class TasksViewModel(application: Application) : AndroidViewModel(application)
```

雖然說 ViewModel 通常會與 Lifecycle 有關聯，但如果只是測試內部的方法其實也不需要使用到 Android Framework 或 OS。所以我們還是可以在 `test` 中建立 **TasksViewModel** 的測試。

但是從建構子可以看到一個致命問題，那就是想要測試 **TasksViewModel** 我們就需要得到 `application`。

但是 **Application** 理論上只會跟著 App 的啟動才會啟動。那我們該如何在 `test` 取得 **Application** 呢？

這時我們就需要用到 **AndroidX Test libraries** 。

### AndroidX Test libraries

這個函式庫可以為我們模擬 Android Framework。如此一來，我們就可以通過函式庫取得測試版的 **Application Context**，包括 Application 與 Activity。

這個函式庫通常都是專案中預設的：

```groovy
// module gradle, but now it's usually located at settings.gradle
allprojects {
  repositories {
    jcenter()
    google()
  }
}
```

Dependencies 如下：

```groovy
androidTestImplementation('androidx.test.espresso:espresso-core:$espressoVersion')
```

以下是其他相關的 dependencies：

```groovy
dependencies {
    // Core library
    androidTestImplementation("androidx.test:core:$androidXTestVersion")

    // AndroidJUnitRunner and JUnit Rules
    androidTestImplementation("androidx.test:runner:$testRunnerVersion")
    androidTestImplementation("androidx.test:rules:$testRulesVersion")

    // Assertions
    androidTestImplementation("androidx.test.ext:junit:$testJunitVersion")
    androidTestImplementation("androidx.test.ext:truth:$truthVersion")

    // Espresso dependencies
    androidTestImplementation( "androidx.test.espresso:espresso-core:$espressoVersion")
    androidTestImplementation( "androidx.test.espresso:espresso-contrib:$espressoVersion")
    androidTestImplementation( "androidx.test.espresso:espresso-intents:$espressoVersion")
    androidTestImplementation( "androidx.test.espresso:espresso-accessibility:$espressoVersion")
    androidTestImplementation( "androidx.test.espresso:espresso-web:$espressoVersion")
    androidTestImplementation( "androidx.test.espresso.idling:idling-concurrent:$espressoVersion")

    // The following Espresso dependency can be either "implementation",
    // or "androidTestImplementation", depending on whether you want the
    // dependency to appear on your APK"s compile classpath or the test APK
    // classpath.
    androidTestImplementation( "androidx.test.espresso:espresso-idling-resource:$espressoVersion")
}
```

#### 我們的設定

根據我們的需求，我們所需要的函式庫是：

```groovy
// Core library
testImplementation("androidx.test:core-ktx:$androidXTestVersion")
testImplementation("androidx.test.ext:junit-ktx:$testJunitVersion")
```

> 注意：我們將 **androidTestImplementation** 改為 **testImplementation**。另外，我們需要的是 `ktx` 版本。

實際版本可以參考 [官方資料](https://developer.android.com/jetpack/androidx/releases/test)

#### 如何使用？

如此一來，我們就可以這樣寫：

```kotlin
val tasksViewModel = TasksViewModel(ApplicationProvider.getApplicationContext())
```

#### 尚有不足

但這還不夠，因為 **ApplicationProvider** 的 `getApplicationContext` 需要通過 **InstrumentationRegistry** 取得 **Application** ：

```java
public static <T extends Context> T getApplicationContext() {
return (T)
    InstrumentationRegistry.getInstrumentation().getTargetContext().getApplicationContext();
}
```

這時我們需要另一個函式庫 [**Robolectric**](https://robolectric.org/)。

### Robolectric

另外，我們還需要 **[Robolectric](https://robolectric.org/)**，一個能幫我們通過 JVM 跑模擬測試的函式庫。
它能為我們模擬 Android 環境，從而免去使用實機或模擬器並加快測試速度。

由於它是模擬 Android Framework，所以會使用到 Android 的資源。 因此，我們需要定義以下：

```groovy
android {
  testOptions {
    unitTests {
      includeAndroidResources = true
    }
  }
}

dependencies {
  testImplementation 'junit:junit:4.13.2'
  testImplementation 'org.robolectric:robolectric:4.9'
}
```

如果使用 `kts`，我們需要寫：

```groovy
android {
  // ...
  testOptions.unitTests.isIncludeAndroidResources = true
}

dependencies {
  // ...
  testImplementation ("org.robolectric:robolectric:4.9")
}
```

#### 如何使用？

現在有了 **Robolectric**，我們需要在 **TasksViewModelTest** 上加上這兩行：

```kotlin
@Config(sdk = [30]) // Remove when Robolectric supports SDK 31
@RunWith(AndroidJUnit4::class)
class TasksViewModelTest
```

之所以需要 **AndroidJUnit4** 這個 **Runner** 是為了使用 **Robolectric**。
因為 **AndroidJUnit4** 會將實作會委派 **Robolectric** 來進行，也就是 delegate。

### 目前的 Gradle

#### build.gradle (Project)

```groovy
buildscript {
    ext.kotlinVersion = '1.9.10'
    ext.navigationVersion = '2.7.3' // '2.5.0'
    repositories {
        google()
        mavenCentral()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:8.1.1'
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlinVersion"
        classpath "androidx.navigation:navigation-safe-args-gradle-plugin:$navigationVersion"

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

plugins {
    id("com.google.devtools.ksp") version "1.9.10-1.0.13" apply false
}

allprojects {
    repositories {
        google()
        mavenCentral()
    }
}

// Define versions in a single place
ext {
    // Sdk and tools
    minSdkVersion = 21
    targetSdkVersion = 33
    compileSdkVersion = 34

    // App dependencies
    androidXVersion = '1.0.0'
    androidXTestCoreVersion = '1.3.0'
    androidXTestExtKotlinRunnerVersion = '1.1.5' // '1.1.3'
    androidXTestRulesVersion = '1.2.0'
    androidXAnnotations = '1.3.0'
    appCompatVersion = '1.6.1' //'1.4.0'
    archLifecycleVersion = '2.4.0'
    coroutinesVersion = '1.5.2'
    cardVersion = '1.0.0'
    espressoVersion = '3.5.1' // '3.4.0'
    fragmentKtxVersion = '1.4.0'
    junitVersion = '4.13.2'
    materialVersion = '1.9.0' // '1.4.0'
    recyclerViewVersion = '1.2.1'
    roomVersion = '2.5.2' // 2.3.0
    rulesVersion = '1.0.1'
    swipeRefreshLayoutVersion = '1.1.0'
    timberVersion = '5.0.1' // '4.7.1'
}
```

#### build.gradle (Module)

```groovy
apply plugin: 'com.android.application'
apply plugin: 'kotlin-android'
apply plugin: 'kotlin-kapt'
apply plugin: "androidx.navigation.safeargs.kotlin"
apply plugin: "com.google.devtools.ksp"

android {

    namespace  "com.example.android.architecture.blueprints.todoapp"

    compileSdk rootProject.compileSdkVersion

    defaultConfig {
        applicationId "com.example.android.architecture.blueprints.reactive"
        minSdkVersion rootProject.minSdkVersion
        targetSdkVersion rootProject.targetSdkVersion
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }

    buildFeatures {
        buildConfig true
    }

    dataBinding {
        enabled = true
        enabledForTests = true
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_1_8
        targetCompatibility = JavaVersion.VERSION_1_8
    }

    kotlinOptions {
        jvmTarget = "1.8"
    }

    testOptions {
        unitTests {
            includeAndroidResources = true
        }
    }
}

dependencies {

    // App dependencies
    implementation "androidx.core:core-ktx:1.12.0"
    implementation "androidx.appcompat:appcompat:$appCompatVersion"
    implementation "androidx.swiperefreshlayout:swiperefreshlayout:$swipeRefreshLayoutVersion"
    implementation "com.google.android.material:material:$materialVersion"
    implementation "org.jetbrains.kotlinx:kotlinx-coroutines-android:$coroutinesVersion"
    implementation "com.jakewharton.timber:timber:$timberVersion"

    // Architecture Components
    implementation "androidx.room:room-runtime:$roomVersion"
    ksp "androidx.room:room-compiler:$roomVersion"
    implementation "androidx.room:room-ktx:$roomVersion"

    implementation "androidx.navigation:navigation-fragment-ktx:$navigationVersion"
    implementation "androidx.navigation:navigation-ui-ktx:$navigationVersion"

    // Dependencies for local unit tests
    testImplementation "junit:junit:$junitVersion"

    // AndroidX Test - Instrumented testing
    androidTestImplementation "androidx.test.ext:junit:$androidXTestExtKotlinRunnerVersion"
    androidTestImplementation "androidx.test.espresso:espresso-core:$espressoVersion"

    // Kotlin
    implementation "org.jetbrains.kotlin:kotlin-stdlib-jdk7:$kotlinVersion"
    implementation "androidx.fragment:fragment-ktx:$fragmentKtxVersion"

    // Core
    def androidXTestVersion = '1.5.0'
    testImplementation "androidx.test:core-ktx:$androidXTestVersion"
    testImplementation "androidx.test.ext:junit-ktx:$androidXTestExtKotlinRunnerVersion"
    // Simulate JVM robolectric
    testImplementation 'junit:junit:4.13.2'
    testImplementation 'org.robolectric:robolectric:4.9'
    // livedata test
    def archTestingVersion = "2.2.0"
    testImplementation "androidx.arch.core:core-testing:$archTestingVersion"
}
```

### addNewTask 的測試 (1)

現在我們 Gradle 已經設定完成，所以現在可以進行 **TasksViewModelTest** 了。

```kotlin
@Test
fun addNewTask_setsNewTaskEvent() {

    // Given a fresh ViewModel
    val tasksViewModel = TasksViewModel(ApplicationProvider.getApplicationContext())

    // When adding a new task
    tasksViewModel.addNewTask()

    // Then the new task event is triggered
    // TODO test LiveData
}
```

此時我們又遇到問題了。當我們調用 `addNewTask` 時，我們需要對 **LiveData** 進行監聽才行。那該怎麼辦呢？

#### LiveData

我們需要使用到另一個函式庫 [AndroidX Arch](https://developer.android.com/jetpack/androidx/releases/arch-core)：

```groovy
def archTestingVersion = "2.2.0"
testImplementation "androidx.arch.core:core-testing:$archTestingVersion"

// kts
val archTestingVersion = "2.2.0"
testImplementation ("androidx.arch.core:core-testing:$archTestingVersion")
```

這個函式庫可以提供測試 **LiveData** 所需要的 **Rule**。

#### Rule

> <br>
> Annotates fields that reference rules or methods that return a rule.
> <br>

這裡所謂的 **Rule** 其實是 **org.junit.rules.TestRule** 或 **org.junit.rules.MethodRule** 的子類別。

##### TestRule

> <br>
> A TestRule is an alteration in how a test method, or set of test methods, is run and reported
> A TestRule may add additional checks that cause a test that would otherwise fail to pass, or it may perform necessary setup or cleanup for tests, or it may observe test execution to report it elsewhere.
> <br><br/>

```java
public interface TestRule {
    Statement apply(Statement base, Description description);
}
```

**TestRule** 雖然可以做 **Before** 與 **After** 可以做的事，但這兩者比想像中更強大，且更容易在不同的類別與專案間共享。另外，**JUnit Runner** 除了可以通過 **Rule** 取得方法層級的 **TestRule** 還可以通過 **ClassRule** 取得類別層級的 Rule。

我們甚至可以設定多個 **Rule** 並以階層的方式執行。

以下是函式庫提供的 **TestRule** 子類別：

- **ErrorCollector**: collect multiple errors in one test method
- **ExpectedException**: make flexible assertions about thrown exceptions
- **ExternalResource**: start and stop a server, for example
- **TemporaryFolder**: create fresh files, and delete after test
- **TestName**: remember the test name for use during the method
- **TestWatcher**: add logic at events during method execution
- **Timeout**: cause test to fail after a set time
- **Verifier**: fail test if object state ends up incorrect

##### MethodRule

```java
public interface MethodRule {
    Statement apply(Statement base, FrameworkMethod method, Object target);
}
```

這與 **TestRule** 差不多，而函式庫所提供的子類別為：

- **JUnitRule**
- **MockitoRule**

##### InstantTaskExecutorRule

在這個範例中，我們需要使用的 **Rule** 是 **InstantTaskExecutorRule**。

```kotlin
@get:Rule
var instantExecutorRule = InstantTaskExecutorRule()
```

**InstantTaskExecutorRule** 所提供的 **Executor** 會讓事件通過 synchronously (同步) 進行。也就是說，他會將 Executor 取代原本的 Background Executor。

```java
public class InstantTaskExecutorRule extends TestWatcher {
    @Override
    protected void starting(Description description) {
        super.starting(description);
        ArchTaskExecutor.getInstance().setDelegate(new TaskExecutor() {
            @Override
            public void executeOnDiskIO(@NonNull Runnable runnable) {
                runnable.run();
            }

            @Override
            public void postToMainThread(@NonNull Runnable runnable) {
                runnable.run();
            }

            @Override
            public boolean isMainThread() {
                return true;
            }
        });
    }

    @Override
    protected void finished(Description description) {
        super.finished(description);
        ArchTaskExecutor.getInstance().setDelegate(null);
    }
}
```

從源碼可以看出他並不會在對應的 Thread 中調用 runnable，不像 **DefaultTaskExecutor** 那樣會讓 Executor 在不同的縣城執行：

```java

private final ExecutorService mDiskIO = Executors.newFixedThreadPool(4, new ThreadFactory() {
    private static final String THREAD_NAME_STEM = "arch_disk_io_";

    private final AtomicInteger mThreadId = new AtomicInteger(0);

    @Override
    public Thread newThread(Runnable r) {
        Thread t = new Thread(r);
        t.setName(THREAD_NAME_STEM + mThreadId.getAndIncrement());
        return t;
    }
});

@Override
public void executeOnDiskIO(@NonNull Runnable runnable) {
    mDiskIO.execute(runnable);
}
```

如此一來，**LiveData** 的資料傳遞都會通過相同的 Thread 進行。 這樣我們才可以監控 **LiveData** 的更新。

### addNewTask 的測試 (2) -- LiveData

現在有了 **Rule** 後，我們可以正式針對 **LiveData** 進行測試了。

現在我們想要測試是否可以通過 `addNewTask` 真的創建新的 **Event**。
由於 `addNewTask` 更新的是 `newTaskEvent: LiveData<Event<Unit>>`。所以我們需要看看如何最 **LiveData** 進行測試。

通常我們在使用 **LiveData** 時，我們都會需要用到 **LifecycleObserver**。 但在測試環境中，我們可能無法取得的。
取而代之，我們可以創建一個 **Observer** 並通過 `observeForever` 來不停監控 **LiveData** 的更新：

```kotlin
@Test
fun addNewTask_setsNewTaskEvent() {

    // Given a fresh ViewModel
    val tasksViewModel = TasksViewModel(ApplicationProvider.getApplicationContext())


    // Create observer - no need for it to do anything!
    val observer = Observer<Event<Unit>> {}
    try {

        // Observe the LiveData forever
        tasksViewModel.newTaskEvent.observeForever(observer)

        // When adding a new task
        tasksViewModel.addNewTask()

        // Then the new task event is triggered
        val value = tasksViewModel.newTaskEvent.value
        assertThat(value?.getContentIfNotHandled(), (not(nullValue())))

    } finally {
        // Whatever happens, don't forget to remove the observer!
        tasksViewModel.newTaskEvent.removeObserver(observer)
    }
}
```

當中的 `observer` 只是為了讓 **LiveData** 知道有監控的人，這樣才會收到 `data` 的 `onChanged`。

#### 簡化 boilerplate

如果每次 **LiveData** 的測試都需要如此冗長，那真的很累。
所以為了簡化，我們可以寫一個 **LiveData** 的 extension， `getOrAwaitValue` ：

```kotlin
@VisibleForTesting(otherwise = VisibleForTesting.NONE)
fun <T> LiveData<T>.getOrAwaitValue(
    time: Long = 2, // timeout
    timeUnit: TimeUnit = TimeUnit.SECONDS,
    afterObserve: () -> Unit = {}
): T {
    var data: T? = null
    val latch = CountDownLatch(1)
    val observer = object : Observer<T> {
        override fun onChanged(value: T) {
            data = value
            latch.countDown()
            this@getOrAwaitValue.removeObserver(this)
        }
    }
    this.observeForever(observer)

    try {
        afterObserve.invoke()

        // Don't wait indefinitely if the LiveData is not set.
        // therefore, there is a timeout
        // await(long timeout, TimeUnit unit)
        if (!latch.await(time, timeUnit)) {
            throw TimeoutException("LiveData value was never set.")
        }

    } finally {
        this.removeObserver(observer)
    }

    @Suppress("UNCHECKED_CAST")
    return data as T
}
```

這裡的源碼看似複雜但其實行為很簡單。
`getOrAwaitValue` 會有以下行為：

1. 建立一個監控 LiveData 的 **Observer**。
   他會在收到新的 data 時將自己移除。
2. 建立 **CountDownLatch**，並預設 count 為 1。
   在 count 尚未是 0 之前， **CountDownLatch** 都會通過 `await` 阻止任意 thread 的進行。
3. 通過 `observeForever` 讓 `Observer` 開始對 **LiveData** 的監聽
4. 開始監聽後，就會調用 `afterObserve.invoke()` 來進行資料的更新
5. 此時理應會通過 `observer` 調用 `latch.countDown()`
6. 通過調用 `latch.await` 可以查看 data 是否有被更新。
   若沒有更新，就會拋出 **TimeoutException**
7. 再次移除 `observer`，因為若 data 沒有更新，那 `observer` 依舊沒被移除。
8. 最後回傳 `data`

通過 `getOrAwaitValue` 便可以將之前的源碼改成：

```kotlin
@Config(sdk = [30]) // Remove when Robolectric supports SDK 31
@RunWith(AndroidJUnit4::class)
class TasksViewModelTest {
    // Test code

    // Replace Background Executor with InstantTaskExecutor
    // This will simply execute the runnable instead of executing it on different threads
    @get:Rule
    var instantExecutorRule = InstantTaskExecutorRule()

    @Test
    fun addNewTask_setsNewTaskEvent() {

        // Given a fresh ViewModel
        val tasksViewModel = TasksViewModel(ApplicationProvider.getApplicationContext())
        tasksViewModel.addNewTask()
        val value = tasksViewModel.newTaskEvent.getOrAwaitValue()
        assertThat(value.getContentIfNotHandled(), (not(nullValue())))
    }
}
```

## setFilterAllTasks 的測試

這次我們需要測試 **Filter** 的更改是否會會成功將 `_tasksAddViewVisible` 設為 `true`

```kotlin
private fun setFilter(
    @StringRes filteringLabelString: Int, @StringRes noTasksLabelString: Int,
    @DrawableRes noTaskIconDrawable: Int, tasksAddVisible: Boolean
) {
    _currentFilteringLabel.value = filteringLabelString
    _noTasksLabel.value = noTasksLabelString
    _noTaskIconRes.value = noTaskIconDrawable
    _tasksAddViewVisible.value = tasksAddVisible
}
```

我們依樣畫葫蘆可以寫成：

```kotlin
@Test
fun setFilterAllTasks_tasksAddViewVisible() {

    // Given a fresh ViewModel
    val tasksViewModel = TasksViewModel(ApplicationProvider.getApplicationContext())

    // When the filter type is ALL_TASKS
    tasksViewModel.setFiltering(TasksFilterType.ALL_TASKS)

    // Then the "Add task" action is visible
    assertThat(tasksViewModel.tasksAddViewVisible.getOrAwaitValue(), `is`(true))
}
```

## 移除 TasksViewModel 的創建

你會發現這裡個測試中，我們都需要重複地建立 **TasksViewModel**。 那該如何做呢？

我們可以將它設為 `lateinit` 並通過 `@Before` 定義在測試之前該做的事：

```kotlin
// Subject under test
private lateinit var tasksViewModel: TasksViewModel

@Before
fun setupViewModel() {
    tasksViewModel = TasksViewModel(ApplicationProvider.getApplicationContext())
}
```

通過這個方設定，我們可以移除此行：

```kotlin
val tasksViewModel = TasksViewModel(ApplicationProvider.getApplicationContext())
```

所以變成以下：

```kotlin
@Config(sdk = [30]) // Remove when Robolectric supports SDK 31
@RunWith(AndroidJUnit4::class)
class TasksViewModelTest {
    // Test code

    // Replace Background Executor with InstantTaskExecutor
    // This will simply execute the runnable instead of executing it on different threads
    @get:Rule
    var instantExecutorRule = InstantTaskExecutorRule()

    // Subject under test
    private lateinit var tasksViewModel: TasksViewModel

    @Before
    fun setupViewModel() {
        tasksViewModel = TasksViewModel(ApplicationProvider.getApplicationContext())
    }

    @Test
    fun addNewTask_setsNewTaskEvent() {
        tasksViewModel.addNewTask()
        val value = tasksViewModel.newTaskEvent.getOrAwaitValue()
        assertThat(value.getContentIfNotHandled(), (not(nullValue())))
    }

    @Test
    fun setFilterAllTasks_tasksAddViewVisible() {
        tasksViewModel.setFiltering(TasksFilterType.ALL_TASKS)
        assertThat(tasksViewModel.tasksAddViewVisible.getOrAwaitValue(), `is`(true))
    }
}
```

## 架構與測試

> <br>
> 一個好的測試環境通常需要分工清晰，如此一來每次都可以針對某個特點進行測試。
> 而分工的清晰則是由專案的架構決定。 所以，一個好的架構才能寫出好的測試。
> <br>

接下來，我們會進行以下的測試：

- **repository** 的 Unit Test
- **viewModel** 的 Unit 與 Integration Test
- **fragments 與 viewModel** 的 Integration Test
- **navigation** 的 Integration Test

## Repository

這部分我們要針對 **DefaultTasksRepository** 進行測試：

```kotlin
class DefaultTasksRepository private constructor(application: Application) {
    private val tasksRemoteDataSource: TasksDataSource
    private val tasksLocalDataSource: TasksDataSource
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO
}
```

我們所定義的 **Repository** 包含兩個來源： **Remote** 與 **Local**。**Local** 就是 本地端的**Database**，而 **Remote** 則是網路上的資料。

由於我們未必可以隨時取得 **Remote** 資料，也可能尚未有 **Local** 資料，所以為了減少這種不定性。我們需要建立一個 **FakeDataSource** 來充當測試的資料來源 。這也是所謂的 **Test Double**。

### Test Double

**Test Double** 有以下的類別：

| Test Double | Description                                                                                                                                                                                                                    |
| :---------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Fake**    | A test double that has a **"working" implementation** of the class, but it's implemented in a way that makes it good for tests but unsuitable for production.                                                                  |
| **Mock**    | A test double that **tracks which of its methods were called**. It then _passes or fails a test depending on whether it's methods were called correctly_.                                                                      |
| **Stub**    | A test double that **includes no logic and only returns what you program it to return**. A StubTaskRepository **could be programmed to return certain combinations of tasks from getTasks** for example.                       |
| **Dummy**   | A test double that is **passed around but not used**, such as if you _just need to provide it as a parameter_. If you had a NoOpTaskRepository, it would just implement the TaskRepository with no code in any of the methods. |
| **Spy**     | A test double which also **keeps tracks of some additional information**; for example, if you made a SpyTaskRepository, it might _keep track of the number of times the addTask method was called_.                            |

更多的資訊可以看看這個 [Blog](https://testing.googleblog.com/2013/07/testing-on-toilet-know-your-test-doubles.html)

在 Android 中，通常會使用 **Mock** 與 **Fake** 進行測試。

### FakeDataSource

我們在 `test` 檔案的 `data.source` 中建立一個 **FakeDataSource** ：

```kotlin
class FakeDataSource(var tasks: MutableList<Task>? = mutableListOf()) : TasksDataSource {
    override fun observeTasks(): LiveData<Result<List<Task>>> {
        TODO("Not yet implemented")
    }

    override suspend fun getTasks(): Result<List<Task>> {
        tasks?.let { return Result.Success(ArrayList(it)) }
        return Result.Error(
            Exception("Tasks not found")
        )
    }

    override suspend fun deleteAllTasks() {
        tasks?.clear()
    }

    override suspend fun saveTask(task: Task) {
        tasks?.add(task)
    }

    override suspend fun refreshTasks() {
        TODO("Not yet implemented")
    }

    override fun observeTask(taskId: String): LiveData<Result<Task>> {
        TODO("Not yet implemented")
    }

    override suspend fun getTask(taskId: String): Result<Task> {
        TODO("Not yet implemented")
    }

    override suspend fun refreshTask(taskId: String) {
        TODO("Not yet implemented")
    }

    override suspend fun completeTask(task: Task) {
        TODO("Not yet implemented")
    }

    override suspend fun completeTask(taskId: String) {
        TODO("Not yet implemented")
    }

    override suspend fun activateTask(task: Task) {
        TODO("Not yet implemented")
    }

    override suspend fun activateTask(taskId: String) {
        TODO("Not yet implemented")
    }

    override suspend fun clearCompletedTasks() {
        TODO("Not yet implemented")
    }

    override suspend fun deleteTask(taskId: String) {
        TODO("Not yet implemented")
    }

}
```

### Dependency Injection

雖然定義了 **FakeDataSource**，但我們目前無法更換 `tasksRemoteDataSource`：

```kotlin
class DefaultTasksRepository private constructor(application: Application) {
  companion object {
      @Volatile
      private var INSTANCE: DefaultTasksRepository? = null

      fun getRepository(app: Application): DefaultTasksRepository {
          return INSTANCE ?: synchronized(this) {
              DefaultTasksRepository(app).also {
                  INSTANCE = it
              }
          }
      }
  }
}
```

為了要更換，我們需要通過 **Dependency Injection** 才行。 所以我們需要修改一下 **DefaultTasksRepository** 的建構子：

```kotlin
class DefaultTasksRepository(
    private val tasksRemoteDataSource: TasksDataSource,
    private val tasksLocalDataSource: TasksDataSource,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO)
```

並且修改 `getRepository` 為：

```kotlin
companion object {
    @Volatile
    private var INSTANCE: DefaultTasksRepository? = null

    fun getRepository(app: Application): DefaultTasksRepository {
        return INSTANCE ?: synchronized(this) {
            val database = Room.databaseBuilder(app,
                ToDoDatabase::class.java, "Tasks.db")
                .build()
            DefaultTasksRepository(TasksRemoteDataSource, TasksLocalDataSource(database.taskDao())).also {
                INSTANCE = it
            }
        }
    }
}
```

現在可以進行 DI 了，我們可以進行測試了。

#### 導入 FakeDataSource

首先，我們需要建立一個 **DefaultTasksRepositoryTest** ：

```kotlin
class DefaultTasksRepositoryTest {
    private val task1 = Task("Title1", "Description1")
    private val task2 = Task("Title2", "Description2")
    private val task3 = Task("Title3", "Description3")
    private val remoteTasks = listOf(task1, task2).sortedBy { it.id }
    private val localTasks = listOf(task3).sortedBy { it.id }
    private val newTasks = listOf(task3).sortedBy { it.id }

    private lateinit var tasksRemoteDataSource: FakeDataSource
    private lateinit var tasksLocalDataSource: FakeDataSource

    // Class under test
    private lateinit var tasksRepository: DefaultTasksRepository
}
```

並在 `@Before` 中建立 **DefaultTasksRepository** ：

```kotlin
@Before
fun createRepository() {
    tasksRemoteDataSource = FakeDataSource(remoteTasks.toMutableList())
    tasksLocalDataSource = FakeDataSource(localTasks.toMutableList())
    // Get a reference to the class under test
    tasksRepository = DefaultTasksRepository(
        // TODO Dispatchers.Unconfined should be replaced with Dispatchers.Main
        //  this requires understanding more about coroutines + testing
        //  so we will keep this as Unconfined for now.
        tasksRemoteDataSource, tasksLocalDataSource, Dispatchers.Unconfined
    )
}
```

我們此時可以建立 `getTasks` 的測試：

```kotlin
@Test
fun getTasks_requestsAllTasksFromRemoteDataSource() {
    // When tasks are requested from the tasks repository
    val tasks = tasksRepository.getTasks(true) as Result.Success

    // Then tasks are loaded from the remote data source
    assertThat(tasks.data, IsEqual(remoteTasks))
}
```

但這會出現問題，由於 `getTasks` 本身是 **suspend** 方法，所以要嘛調用他的方法也是 **suspend** 否則就需要通過 **Coroutine** 進行調用。

##### Coroutine

為了要加上 **Coroutine** 來測試，我們需要 `testImplementation` **kotlinx-coroutines-test**：

```groovy
testImplementation "org.jetbrains.kotlinx:kotlinx-coroutines-test:$coroutinesVersion"
```

#### getTasks 測試

設定好後，我們便可以使用 `runBlockingTest` 來進行測試了。
這裡我們要測試的是當我們調用 `getTasks` 時，他會不會從 remote 取得最新資料？

```kotlin
@Test
fun getTasks_requestsAllTasksFromRemoteDataSource() = runBlockingTest {
    // When tasks are requested from the tasks repository
    val tasks = tasksRepository.getTasks(true) as Success

    // Then tasks are loaded from the remote data source
    assertThat(tasks.data, IsEqual(remoteTasks))
}
```

`runBlockingTest` 在新的版本中被標示 **deprecated**，所以我們可以使用試驗版的 `runTest` ：

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
@Test
fun getTasks_requestsAllTasksFromRemoteDataSource() = runTest {
    // When tasks are requested from the tasks repository
    val tasks = tasksRepository.getTasks(true) as Result.Success

    // Then tasks are loaded from the remote data source
    assertThat(tasks.data, IsEqual(remoteTasks))
}
```

##### 創建 TasksRepository 介面

想要測試 DefaultTasksRepository 我們就要先創建一個 DefaultTasksRepository 介面。 我們右鍵點選 DefaultTasksRepository 類別名稱，然後點選 Refactor \> Extract Interface \> Extract to Separate File 並更改名稱為 TasksRepository 且只點選 public 方法，不包含 companion。

```kotlin
interface TasksRepository {
    suspend fun getTasks(forceUpdate: Boolean = false): Result<List<Task>>

    suspend fun refreshTasks()
    fun observeTasks(): LiveData<Result<List<Task>>>

    suspend fun refreshTask(taskId: String)
    fun observeTask(taskId: String): LiveData<Result<Task>>

    /**
     * Relies on [getTasks] to fetch data and picks the task with the same ID.
     */
    suspend fun getTask(taskId: String, forceUpdate: Boolean = false): Result<Task>

    suspend fun saveTask(task: Task)

    suspend fun completeTask(task: Task)

    suspend fun completeTask(taskId: String)

    suspend fun activateTask(task: Task)

    suspend fun activateTask(taskId: String)

    suspend fun clearCompletedTasks()

    suspend fun deleteAllTasks()

    suspend fun deleteTask(taskId: String)
}
```

如此一來， DefaultTasksRepository 就會實作 TasksRepository：

```kotlin
class DefaultTasksRepository(
    private val tasksRemoteDataSource: TasksDataSource,
    private val tasksLocalDataSource: TasksDataSource,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO) : TasksRepository
```

##### 創建 FakeRepository

在 test \> data.source 中創建 FakeRepository.kt 並讓它實作 TasksRepository：

```kotlin
class FakeTestRepository: TasksRepository {

    override suspend fun getTasks(forceUpdate: Boolean): Result<List<Task>> {
        TODO("Not yet implemented")
    }

    override suspend fun refreshTasks() {
        TODO("Not yet implemented")
    }

    override fun observeTasks(): LiveData<Result<List<Task>>> {
        TODO("Not yet implemented")
    }

    override suspend fun refreshTask(taskId: String) {
        TODO("Not yet implemented")
    }

    override fun observeTask(taskId: String): LiveData<Result<Task>> {
        TODO("Not yet implemented")
    }

    override suspend fun getTask(taskId: String, forceUpdate: Boolean): Result<Task> {
        TODO("Not yet implemented")
    }

    override suspend fun saveTask(task: Task) {
        TODO("Not yet implemented")
    }

    override suspend fun completeTask(task: Task) {
        TODO("Not yet implemented")
    }

    override suspend fun completeTask(taskId: String) {
        TODO("Not yet implemented")
    }

    override suspend fun activateTask(task: Task) {
        TODO("Not yet implemented")
    }

    override suspend fun activateTask(taskId: String) {
        TODO("Not yet implemented")
    }

    override suspend fun clearCompletedTasks() {
        TODO("Not yet implemented")
    }

    override suspend fun deleteAllTasks() {
        TODO("Not yet implemented")
    }

    override suspend fun deleteTask(taskId: String) {
        TODO("Not yet implemented")
    }
}
```

接下來我們就要準備資料：

```kotlin
// All the tasks
var tasksServiceData: LinkedHashMap<String, Task> = LinkedHashMap()

// Tasks that are being fetched
private val observableTasks = MutableLiveData<Result<List<Task>>>()
```

##### 實作 getTasks, refreshTasks, observeTasks

如此一來，我們就可以實作 `getTasks`：

```kotlin
override suspend fun getTasks(forceUpdate: Boolean): Result<List<Task>> {
    return Result.Success(tasksServiceData.values.toList())
}
```

通過 `getTasks` 我們也可以實作 `refreshTasks`：

```kotlin
override suspend fun refreshTasks() {
    observableTasks.value = getTasks(true)
}
```

通過這兩個方法，我們可以通過 `runBlocking` 實作 `observeTasks`：

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
override fun observeTasks(): LiveData<Result<List<Task>>>  {
    runBlocking {
        refreshTasks()
    }
    return observableTasks
}
```

如果這個方法是 `@Test` 那就可以使用 `runTest` 或 `runBlockingTest` 來得到固定的行為。 但 `runBlocking` 會與現實更貼切。

##### 新增 addTasks

一般來說，如果我們要新增 Tasks，repository 裡面最好就是有一些資料，這可以通過調用多次的 `saveTask`。 但為了簡化，我們直接通過新增方法， `addTasks`， 來更新 `tasksServiceData` ：

```kotlin
fun addTasks(vararg tasks: Task) {
    for (task in tasks) {
        tasksServiceData[task.id] = task
    }
    runBlocking { refreshTasks() }
}
```

##### 更新 TasksViewModel 建構子

想要測試就先將 **TasksViewModel** 建構子更新 DI：

```kotlin
class TasksViewModel(application: Application) : AndroidViewModel(application) {
    private val tasksRepository = DefaultTasksRepository.getRepository(application)
    /* ... */
}

// 更新成

class TasksViewModel( private val tasksRepository: TasksRepository ) : ViewModel() { /* ... */ }
```

然後將建設一個 **TasksViewModelFactory** ：

```kotlin
@Suppress("UNCHECKED_CAST")
class TasksViewModelFactory (
    private val tasksRepository: TasksRepository
) : ViewModelProvider.NewInstanceFactory() {
    override fun <T : ViewModel> create(modelClass: Class<T>) =
        (TasksViewModel(tasksRepository) as T)
}
```

最後在 **TasksFragment** 中更改 TasksViewModel 的創建：

```kotlin
private val viewModel by viewModels<TasksViewModel>()

// 改為

private val viewModel by viewModels<TasksViewModel> {
    TasksViewModelFactory(DefaultTasksRepository.getRepository(requireActivity().application))
}
```

##### 使用 FakeTestRepository

在 **TasksViewModelTest** 中定義 **FakeTestRepository** ：

```kotlin
@RunWith(AndroidJUnit4::class)
class TasksViewModelTest {

    // Use a fake repository to be injected into the viewmodel
    private lateinit var tasksRepository: FakeTestRepository

    // Rest of class
}
```

更新 `setupViewModel`：

```kotlin
@Before
fun setupViewModel() {
    tasksViewModel = TasksViewModel(ApplicationProvider.getApplicationContext())
}

// 更新為

@Before
fun setupViewModel() {
    // We initialise the tasks to 3, with one active and two completed
    tasksRepository = FakeTestRepository()
    val task1 = Task("Title1", "Description1")
    val task2 = Task("Title2", "Description2", true)
    val task3 = Task("Title3", "Description3", true)
    tasksRepository.addTasks(task1, task2, task3)

    tasksViewModel = TasksViewModel(tasksRepository)
}
```

此時因為我們不再需要調用 **ApplicationProvider**， 所以可以將 `@RunWith(AndroidJUnit4::class)` 移除。

##### 更新 TaskDetailViewModel

再次地依樣畫葫蘆，我們更新 **TaskDetailViewModel** 為：

```kotlin
// REPLACE
class TaskDetailViewModel(application: Application) : AndroidViewModel(application) {

    private val tasksRepository = DefaultTasksRepository.getRepository(application)

    // Rest of class
}

// WITH

class TaskDetailViewModel(
    private val tasksRepository: TasksRepository
) : ViewModel() { // Rest of class }
```

並創建 **TaskDetailViewModelFactory**：

```kotlin
@Suppress("UNCHECKED_CAST")
class TaskDetailViewModelFactory (
    private val tasksRepository: TasksRepository
) : ViewModelProvider.NewInstanceFactory() {
    override fun <T : ViewModel> create(modelClass: Class<T>) =
        (TaskDetailViewModel(tasksRepository) as T)
}
```

最後在 **TaskDetailFragment** 中更新：

```kotlin
private val viewModel by viewModels<TaskDetailViewModel>()

// 改為

private val viewModel by viewModels<TaskDetailViewModel>() {
    TaskDetailViewModelFactory(DefaultTasksRepository.getRepository(requireActivity().application))
}
```

當然，我們也可以寫出一個統一的 TasksViewModelFactory ：

```kotlin
@Suppress("UNCHECKED_CAST")
class TaskDetailViewModelFactory (
    private val tasksRepository: TasksRepository
) : ViewModelProvider.NewInstanceFactory() {
    override fun <T : ViewModel> create(modelClass: Class<T>) {
        return with(modelClass) {
          when {
            isAssignableFrom(TaskDetailViewModel::class.java) -> TaskDetailViewModel(tasksRepository)
            isAssignableFrom(TasksViewModel::class.java) -> TasksViewModel(tasksRepository)
            else -> throw IllegalArgumentException("TaskDetailViewModelFactory: Unsupported model class $modelClass")
          }
        } as T
    }
}
```

## Integration Test

> Integration tests 主要目的是將多個類別一同測試來確保它們之間互動正常。
> 測試內容可以是在 **test** 或 **androidTest** 中
> <br>

我們此次的目的是測試 Fragment 與 ViewModel。

### androidTest Dependency

由於我們依然需要使用到 **Coroutine** 與 **JUnit**， 所以我們除了原本的 `testImplementation` 外，現在外加 `androidTestImplementation`：

```groovy
// android instrumented unit test, we also have these in testImplementation
androidTestImplementation "org.jetbrains.kotlinx:kotlinx-coroutines-test:$coroutinesVersion"
androidTestImplementation "junit:junit:$junitVersion"

// Testing code should not be included in the main code.
// Once https://issuetracker.google.com/128612536 is fixed this can be fixed.
implementation "androidx.test:core:$androidXTestCoreVersion"
debugImplementation "androidx.fragment:fragment-testing:$fragmentVersion"
```

他們的作用如下：

| dependency                                                                                     | function                                                                        |
| :--------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------ |
| `androidTestImplementation "org.jetbrains.kotlinx:kotlinx-coroutines-test:$coroutinesVersion"` | The coroutines testing library                                                  |
| `androidTestImplementation "junit:junit:$junitVersion"`                                        | JUnit, which is necessary for writing basic test statements.                    |
| `implementation "androidx.test:core:$androidXTestCoreVersion"`                                 | Core AndroidX test library                                                      |
| `implementation "androidx.fragment:fragment-testing:$fragmentVersion"`                         | AndroidX test library for creating fragments in tests and changing their state. |

### TaskDetailFragmentTest

右鍵點選 TaskDetailFragment \> Generate \> Test \> OK \> androidTest

在 androidTest 中創建一個 TaskDetailFragmentTest：

```kotlin
class TaskDetailFragmentTest {}
```

然後因為我們需要使用到 AndroidX Test，所以也需要定義 `@RunWith(AndroidJUnit4::class)`。 然後我們也定義一下這個測試的 group 為 `MediumTest`：

```kotlin
@MediumTest
@RunWith(AndroidJUnit4::class) // Used in any class using AndroidX Test
class TaskDetailFragmentTest {}
```

除了 `MediumTest` 它還有以下大小：

|                                          Group                                          | Usage                                                  |
| :-------------------------------------------------------------------------------------: | :----------------------------------------------------- |
|  [@SmallTest](https://developer.android.com/reference/androidx/test/filters/SmallTest)  | Unit Test                                              |
| [@MediumTest](https://developer.android.com/reference/androidx/test/filters/MediumTest) | Marks the test as a "medium run-time" integration test |
|  [@LargeTest](https://developer.android.com/reference/androidx/test/filters/LargeTest)  | end-to-end tests                                       |

#### FragmentScenario 來展現 Fragment

這裡我們需要使用到 AndroidX Test 中的 [FragmentScenario](https://developer.android.com/reference/kotlin/androidx/fragment/app/testing/FragmentScenario.html) 類別。 通過 FragmentScenario，我們可以將 fragment 包裹起來並提供我們操控 Fragment 生命週期的能力。

我們可以通過 `launchFragmentInContainer` 來得到 FragmentScenario：

```kotlin
public inline fun <reified F : Fragment> launchFragmentInContainer(
    fragmentArgs: Bundle? = null,
    @StyleRes themeResId: Int = R.style.FragmentScenarioEmptyFragmentActivityTheme,
    initialState: Lifecycle.State = Lifecycle.State.RESUMED,
    factory: FragmentFactory? = null
): FragmentScenario<F> = FragmentScenario.launchInContainer(
    F::class.java, fragmentArgs, themeResId, initialState,
    factory
)
```

預設中他的生命週期是 **RESUMED**，當然我們也可以改。 不過我們就用最簡單的用法，並且用 Bundle 將 `activeTask` 傳入：

```kotlin
@Test
fun activeTaskDetails_DisplayedInUi() {
    // GIVEN - Add active (incomplete) task to the DB
    val activeTask = Task("Active Task", "AndroidX Rocks", false)

    // WHEN - Details fragment launched to display task
    val bundle = TaskDetailFragmentArgs(activeTask.id).toBundle()
    launchFragmentInContainer<TaskDetailFragment>(bundle, R.style.AppTheme)

    // add this if you want to see the screen
    // Thread.sleep(2000)
}
```

以上的測試會直接顯示 TaskDetailFragment 但會顯示 No Data。 這是因為我們沒有將資料存放至 Repository 中。 所以當 TaskDetailFragment 顯示時，它無法在 Repository 找到。以下是 TaskDetailFragment 顯示 Task 的流程：

```kotlin
// TaskDetailFragment
private val args: TaskDetailFragmentArgs by navArgs()

override fun onCreateView(
    inflater: LayoutInflater,
    container: ViewGroup?,
    savedInstanceState: Bundle?
): View? {
    val view = inflater.inflate(R.layout.taskdetail_frag, container, false)
    viewDataBinding = TaskdetailFragBinding.bind(view).apply {
        viewmodel = viewModel
    }
    viewDataBinding.lifecycleOwner = this.viewLifecycleOwner

    // take the args.taskId into start
    viewModel.start(args.taskId)

    setHasOptionsMenu(true)
    return view
}

// TaskDetailViewModel
private val _taskId = MutableLiveData<String>()

private val _task = _taskId.switchMap { taskId ->
    tasksRepository.observeTask(taskId).map { computeResult(it) }
}

fun start(taskId: String) {
   // If we're already loading or already loaded, return (might be a config change)
   if (_dataLoading.value == true || taskId == _taskId.value) {
       return
   }
   // Trigger the load, and fetch data from tasksRepository
   _taskId.value = taskId
}

```

由於我們無法通過建構子將 Repository 帶入 Fragment / Activity 中，所以我們無法將 ViewModel 中的 Repository 從 DefaultTasksRepository 改為 FakeTestRepository。

為此，我們需要使用 Service Locator Pattern 。

#### Service Locator

Service Locator 是一個 Singleton 並使用 DI 傳入所需類別中。 通過這方法，我們可以統一更新 Repository。

接下來我們就設計一個 ServiceLocator 類別：

```kotlin
object ServiceLocator {

    private var database: ToDoDatabase? = null
    @Volatile
    var tasksRepository: TasksRepository? = null

}
```

> 我原本是覺得只需要一個 Repository 即可，所以不知道為什麼還需要有一個 ToDoDatabase。

接下來， ServiceLocator 需要知道以下行為：

| Method                      | Purpose                                                                            |
| :-------------------------- | :--------------------------------------------------------------------------------- |
| `provideTasksRepository`    | 提供新的或已存在的 Repository (需要使用 `synchronize(this)` 來避免 race condition) |
| `createTasksRepository`     | 創建新的 Repository，這裡指的是 TasksRepository                                    |
| `createTaskLocalDataSource` | 通過 `createDataBase` 創建新的 Local Data Source                                   |
| `createDataBase`            | 創建新的 database                                                                  |

以下是他們的實作：

```kotlin
object ServiceLocator {

    private var database: ToDoDatabase? = null
    @Volatile
    var tasksRepository: TasksRepository? = null

    fun provideTasksRepository(context: Context): TasksRepository {
        synchronized(this) {
            return tasksRepository ?: createTasksRepository(context)
        }
    }

    private fun createTasksRepository(context: Context): TasksRepository {
        val newRepo = DefaultTasksRepository(TasksRemoteDataSource, createTaskLocalDataSource(context))
        tasksRepository = newRepo
        return newRepo
    }

    private fun createTaskLocalDataSource(context: Context): TasksDataSource {
        val database = database ?: createDataBase(context)
        return TasksLocalDataSource(database.taskDao())
    }

    private fun createDataBase(context: Context): ToDoDatabase {
        val result = Room.databaseBuilder(
            context.applicationContext,
            ToDoDatabase::class.java, "Tasks.db"
        ).build().also {
            database = it
        }
        return result
    }
}
```

另外為了要測試 ServiceLocator，我們還需要加上 `visible`：

```kotlin
@Volatile
var tasksRepository: TasksRepository? = null
    @VisibleForTesting set

private val lock = Any()

@VisibleForTesting
fun resetRepository() {
    synchronized(lock) {
        runBlocking {
            TasksRemoteDataSource.deleteAllTasks()
        }
        // Clear all data to avoid test pollution.
        database?.apply {
            clearAllTables()
            close()
        }
        database = null
        tasksRepository = null
    }
}
```

這是為了讓我們可以重新設定 ServiceLocator 的狀態： `taskRepository`。

> <br>
>
> 這也就是使用 Singleton 的缺點。 除了在測試完之後要重設之外，還不能進行平行測試。
> <br>

接下來我們需要將 **ServiceLocator** 放在 Todoapplication 中，這樣在 App 啟動之後就可以設定好：

```kotlin
class TodoApplication : Application() {

    val taskRepository: TasksRepository
        get() = ServiceLocator.provideTasksRepository(this)

    override fun onCreate() {
        super.onCreate()
        if (BuildConfig.DEBUG) Timber.plant(DebugTree())
    }
}
```

接下來我們要先將 DefaultTasksRepository 中創建 Database 的部分去掉：

```kotlin
// 移除
/* companion object {
    @Volatile
    private var INSTANCE: DefaultTasksRepository? = null

    fun getRepository(app: Application): DefaultTasksRepository {
        return INSTANCE ?: synchronized(this) {
            val database = Room.databaseBuilder(app,
                ToDoDatabase::class.java, "Tasks.db")
                .build()
            DefaultTasksRepository(TasksRemoteDataSource, TasksLocalDataSource(database.taskDao())).also {
                INSTANCE = it
            }
        }
    }
} */
```

由於我們移除了 companion object，所以所有調用 `getRepository` 的都需要從 `Application.taskRepository` 取得：

```kotlin
private val tasksRepository = DefaultTasksRepository.getRepository(application)

// 改為

private val tasksRepository = (application as TodoApplication).taskRepository

// 或

private val viewModel by viewModels<TaskDetailViewModel>() {
    /* TaskDetailViewModelFactory(DefaultTasksRepository.getRepository(requireActivity().application)) */

    // 改為

    TaskDetailViewModelFactory((requireActivity().application as TodoApplication).taskRepository)
}
```

接下來，我們要新增一個 **FakeAndroidRepository。** 雖然我們已經建設了一個 FakeTestRepository 在 test 中，但由於我們無法 test 與 androidTest 共用，所以我們還是要在 androidTest 中建設。 其中， FakeAndroidRepository 跟 FakeTestRepository 相比， FakeAndroidRepository 會多出一個 `shouldReturnError` 來進行錯誤測試：

```kotlin
class FakeAndroidTestRepository : TasksRepository {

    var tasksServiceData: LinkedHashMap<String, Task> = LinkedHashMap()

    private var shouldReturnError = false

    private val observableTasks = MutableLiveData<Result<List<Task>>>()

    fun setReturnError(value: Boolean) {
        shouldReturnError = value
    }

    // ... unimplemented functions
}
```

> <br>
>
> 但其實我們是可以分享 test 與 androidTest 之間的檔案的。 我們需要在 Gradle 中更改：
>
> ```groovy
>   android {
> ```

        sourceSets {
            String sharedTestDir = 'src/sharedTest/java'
            test {
                java.srcDir sharedTestDir
            }
            androidTest {
                java.srcDir sharedTestDir
            }
        }
    }

> ```
> 這樣就可以使用 `sharedTestDir` 來存放共享檔案。
> <br>
> ```

接下來就實作其中方法，我們先看看 FakeAndroidRepository 與 FakeTestRepository 的實作差別：

```kotlin
// ============= getTask ==========

// FakeTestRepository
override suspend fun getTasks(forceUpdate: Boolean): Result<List<Task>> {
    return Result.Success(tasksServiceData.values.toList())
}

// FakeAndroidRepository
override suspend fun getTasks(forceUpdate: Boolean): Result<List<Task>> {
    if (shouldReturnError) {
        return Result.Error(Exception("Test exception"))
    }
    return com.example.android.architecture.blueprints.todoapp.data.Result.Success(
        tasksServiceData.values.toList()
    )
}

// ============= refreshTasks ==========
// FakeTestRepository
override suspend fun refreshTasks() {
    observableTasks.value = getTasks(true)
}

// FakeAndroidRepository
override suspend fun refreshTasks() {
    observableTasks.value = getTasks()
}

override suspend fun refreshTask(taskId: String) {
    refreshTasks()
}

// ============= refreshTasks ==========
// FakeTestRepository
fun addTasks(vararg tasks: Task) {
    for (task in tasks) {
        tasksServiceData[task.id] = task
    }
    runBlocking { refreshTasks() }
}

// FakeAndroidRepository
fun addTasks(vararg tasks: Task) {
    for (task in tasks) {
        tasksServiceData[task.id] = task
    }
    runBlocking { refreshTasks() }
}

```

接著就是 FakeAndroidRepository 完整的實作：

```kotlin
class FakeAndroidTestRepository : TasksRepository {

    var tasksServiceData: LinkedHashMap<String, Task> = LinkedHashMap()
    private val observableTasks = MutableLiveData<Result<List<Task>>>()
    private var shouldReturnError = false


    fun setReturnError(value: Boolean) {
        shouldReturnError = value
    }

    fun addTasks(vararg tasks: Task) {
        for (task in tasks) {
            tasksServiceData[task.id] = task
        }
        runBlocking { refreshTasks() }
    }

    override suspend fun getTasks(forceUpdate: Boolean): Result<List<Task>> {
        if (shouldReturnError) {
            return Result.Error(Exception("Test exception"))
        }
        return Result.Success(tasksServiceData.values.toList())
    }

    override suspend fun getTask(taskId: String, forceUpdate: Boolean): Result<Task> {
        if (shouldReturnError) {
            return Result.Error(Exception("Test exception"))
        }
        tasksServiceData[taskId]?.let {
            return Result.Success(it)
        }
        return Result.Error(Exception("Could not find task"))
    }

    // update observableTasks
    override suspend fun refreshTasks() {
        observableTasks.value = getTasks()
    }

    override suspend fun refreshTask(taskId: String) {
        refreshTasks() // update all tasks
    }

    override suspend fun saveTask(task: Task) {
        tasksServiceData[task.id] = task
    }

    // upate target task
    override suspend fun completeTask(task: Task) {
        val completedTask = Task(task.title, task.description, true, task.id)
        tasksServiceData[task.id] = completedTask
    }

    override suspend fun completeTask(taskId: String) {
        // Not required for the remote data source.
        throw NotImplementedError()
    }

    // update task to activate
    override suspend fun activateTask(task: Task) {
        val activeTask = Task(task.title, task.description, false, task.id)
        tasksServiceData[task.id] = activeTask
    }

    override suspend fun activateTask(taskId: String) {
        throw NotImplementedError()
    }

    // fetch only isCompleted
    override suspend fun clearCompletedTasks() {
        tasksServiceData = tasksServiceData.filterValues {
            !it.isCompleted
        } as LinkedHashMap<String, Task>
    }

    // remove task from tasksServiceData
    // and refetch data from tasksServiceData to observableTasks
    override suspend fun deleteTask(taskId: String) {
        tasksServiceData.remove(taskId)
        refreshTasks()
    }

    override suspend fun deleteAllTasks() {
        tasksServiceData.clear()
        refreshTasks()
    }

    // usually takes time to refreshTask
    override fun observeTasks(): LiveData<Result<List<Task>>> {
        runBlocking { refreshTasks() }
        return observableTasks
    }

    // observe specific task
    override fun observeTask(taskId: String): LiveData<Result<Task>> {
        runBlocking { refreshTasks() }
        return observableTasks.map { tasks ->
            when (tasks) {
                is Result.Loading -> Result.Loading
                is Result.Error -> Result.Error(tasks.exception)
                is Result.Success -> {
                    val task = tasks.data.firstOrNull() { it.id == taskId }
                        ?: return@map Result.Error(Exception("Not found"))
                    Result.Success(task)
                }
            }
        }
    }

}
```

##### 更新 TaskDetailFragmentTest (ServiceLocator)

現在我們更新 TaskDetailFragmentTest，新增 ServiceLocator 的測試：

```kotlin
private lateinit var repository: TasksRepository

@Before
fun initRepository() {
    repository = FakeAndroidTestRepository()
    ServiceLocator.tasksRepository = repository
}

@OptIn(ExperimentalCoroutinesApi::class)
@After
fun cleanupDb() = runTest {
    ServiceLocator.resetRepository()
}
```

現在我們將創建的 Task 放入 Repository：

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
@Test
fun activeTaskDetails_DisplayedInUi() = runTest {
    // GIVEN - Add active (incomplete) task to the DB
    val activeTask = Task("Active Task", "AndroidX Rocks", false)
    repository.saveTask(activeTask)

    // WHEN - Details fragment launched to display task
    val bundle = TaskDetailFragmentArgs(activeTask.id).toBundle()
    launchFragmentInContainer<TaskDetailFragment>(bundle, R.style.AppTheme)

    // add this if you want to see the screen
    Thread.sleep(2000)
}
```

### Espresso

接下來我們要進行 UI 測試。 這時我們需要動用到 **Espresso** 因為它可以讓我們與 UI 進行互動。為此，我們所需要的 dependency 是：

```groovy
dependencies {

  // ALREADY in your code
    androidTestImplementation "androidx.test.espresso:espresso-core:$espressoVersion"

 // Other dependencies
}
```

#### 關掉 Animation

由於 Espresso 是在實體機上運作的，所以我們會遇到動畫導致的 delay。 因此，當我們通過 Espresso 檢查畫面時，在進行動畫的 View 就會被無視，導致錯誤的出現。 也因為如此，我們需要將 Animation 給關掉。

#### 基本語法

```kotlin
onView(withId(R.id.task_detail_complete_checkbox)) // findViewById
    .perform(click())                              // click
    .check(matches(isChecked()))                   // determine is it is checked
```

Espresso 語法會包含四個部分：

- **Static Method**
- **ViewMatcher**
- **ViewAction**
- **ViewAssertion**

```kotlin
// Static Method
// https://developer.android.com/reference/androidx/test/espresso/Espresso.html#onView(org.hamcrest.Matcher%3Candroid.view.View%3E)
onView(
      // ViewMatcher
      // https://developer.android.com/reference/androidx/test/espresso/matcher/ViewMatchers.html
      withId(R.id.task_detail_title_text)
)
    .perform(
        // ViewAction
        // https://developer.android.com/reference/androidx/test/espresso/ViewAction.html
        click()
    )

    .check(
        // ViewAssertion
        // https://developer.android.com/reference/androidx/test/espresso/assertion/ViewAssertions#matches
        matches(
          // also a ViewMatcher
          isChecked()
        )
    )
```

#### addTask 測試

```kotlin
// TaskDetailFragmentTest

import androidx.test.espresso.Espresso.onView
import androidx.test.espresso.assertion.ViewAssertions.matches
import androidx.test.espresso.matcher.ViewMatchers.isChecked
import androidx.test.espresso.matcher.ViewMatchers.isDisplayed
import androidx.test.espresso.matcher.ViewMatchers.withId
import androidx.test.espresso.matcher.ViewMatchers.withText
import org.hamcrest.core.IsNot.not

@OptIn(ExperimentalCoroutinesApi::class)
@Test
fun activeTaskDetails_DisplayedInUi() = runTest {
    // GIVEN - Add active (incomplete) task to the DB
    val activeTask = Task("Active Task", "AndroidX Rocks", false)
    repository.saveTask(activeTask)

    // WHEN - Details fragment launched to display task
    val bundle = TaskDetailFragmentArgs(activeTask.id).toBundle()
    launchFragmentInContainer<TaskDetailFragment>(bundle, R.style.AppTheme)

    // THEN - Task details are displayed on the screen
    // make sure that the title/description are both shown and correct
    onView(withId(R.id.task_detail_title_text)).check(matches(isDisplayed()))
    onView(withId(R.id.task_detail_title_text)).check(matches(withText("Active Task")))

    onView(withId(R.id.task_detail_description_text)).check(matches(isDisplayed()))
    onView(withId(R.id.task_detail_description_text)).check(matches(withText("AndroidX Rocks")))

    // and make sure the "active" checkbox is shown unchecked
    onView(withId(R.id.task_detail_complete_checkbox)).check(matches(isDisplayed()))
    onView(withId(R.id.task_detail_complete_checkbox)).check(matches(not(isChecked())))

    // add this if you want to see the screen
     Thread.sleep(3000)
}

```

再次依樣畫葫蘆，創建一個 `completedTaskDetails_DisplayedInUi`：

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
@Test
fun completedTaskDetails_DisplayedInUi() = runTest {
    // GIVEN - Add completed task to the DB
    val title = "Completed Second Test"
    val description = "Espression is pretty cool"
    val completedTask = Task(title, description, true)

    repository.saveTask(completedTask)

    // WHEN - Details fragment launched to display task
    val bundle = TaskDetailFragmentArgs(completedTask.id).toBundle()
    launchFragmentInContainer<TaskDetailFragment>(bundle, R.style.AppTheme)

    // THEN - Task details are displayed on the screen
    // make sure that the title/description are both shown and correct
    onView(withId(R.id.task_detail_title_text)).check(matches(isDisplayed()))
    onView(withId(R.id.task_detail_title_text)).check(matches(withText(title)))
    onView(withId(R.id.task_detail_description_text)).check(matches(isDisplayed()))
    onView(withId(R.id.task_detail_description_text)).check(matches(withText(description)))
    // and make sure the "active" checkbox is shown unchecked
    onView(withId(R.id.task_detail_complete_checkbox)).check(matches(isDisplayed()))
    onView(withId(R.id.task_detail_complete_checkbox)).check(matches(isChecked()))

    // add this if you want to see the screen
    Thread.sleep(3000)
}
```

### Mockito - 導覽測試

接下來我們要進行導覽的測試，這時我們就需要用到 [Mockito](https://site.mockito.org/) 還有一個叫做 **mock** 的 Test Double 。 但是， Mockito 除了能做 mock 還可以使用 **stubs** 與 **spies**。

以下是 TasksFragment 中進行導向的源碼：

```kotlin
private fun openTaskDetails(taskId: String) {
    val action = TasksFragmentDirections.actionTasksFragmentToTaskDetailFragment(taskId)
    findNavController().navigate(action)
}
```

導向本身是個很複雜的行為，所以我們能做的就是檢查調用 `navigate()` 時所使用的變數是否正確。 為此，我們會通過 Mockito 建構一個 mock **NavigationController** 讓我們進行檢測。

#### Dependency

```groovy
// Dependencies for Android instrumented unit tests

// This is the Mockito dependency
androidTestImplementation "org.mockito:mockito-core:$mockitoVersion"

/*
    This library is required to use Mockito in an Android project.
    Mockito needs to generate classes at runtime.
    On Android, this is done using dex byte code, and so this library enables Mockito to generate objects during runtime on Android.
*/
androidTestImplementation "com.linkedin.dexmaker:dexmaker-mockito:$dexMakerVersion"

/*
    This library is made up of external contributions (hence the name) which contain testing code for more advanced views, such as DatePicker and RecyclerView.

    It also contains Accessibility checks and class called CountingIdlingResource that is covered later
*/
androidTestImplementation "androidx.test.espresso:espresso-contrib:$espressoVersion"

```

#### 創建 TasksFragmentTest

> 在 TasksFragment 名稱點選右鍵 \> Generate \> Test \> 選擇 androidTest

再來進行與 Instrumental Unit Test 一樣的設定：

```kotlin
@RunWith(AndroidJUnit4::class)
@MediumTest
@ExperimentalCoroutinesApi
class TasksFragmentTest {

    private lateinit var repository: TasksRepository

    @Before
    fun initRepository() {
        repository = FakeAndroidTestRepository()
        ServiceLocator.tasksRepository = repository
    }

    @After
    fun cleanupDb() = runBlockingTest {
        ServiceLocator.resetRepository()
    }

}
```

#### 測試 navigate To TaskDetailFragmentOne

```kotlin
@Test
fun clickTask_navigateToDetailFragmentOne() = runTest {
    // PREPARE - Create Tasks and stores it
    repository.saveTask(Task("TITLE1", "DESCRIPTION1", false, "id1"))
    repository.saveTask(Task("TITLE2", "DESCRIPTION2", true, "id2"))

    // GIVEN - On the home screen
    val scenario = launchFragmentInContainer<TasksFragment>(Bundle(), R.style.AppTheme)

    // CREATE NAV
    val navController = mock(NavController::class.java)

    // SET NavController
    scenario.onFragment {it: TasksFragment ->
        Navigation.setViewNavController(it.view!!, navController)
    }

    // WHEN - Click on the first list item
    onView(withId(R.id.tasks_list))
            .perform(RecyclerViewActions.actionOnItem<RecyclerView.ViewHolder>(
                hasDescendant(withText("TITLE1")), click()))

    // THEN - Verify that we navigate to the first detail screen
    // verify method is what makes this a mock
    // from this, we can confirm the mocked navController called a specific method (navigate)
    // with a parameter (actionTasksFragmentToTaskDetailFragment with the ID of "id1").
    verify(navController).navigate(
        TasksFragmentDirections.actionTasksFragmentToTaskDetailFragment( "id1")
    )
}
```

#### 依樣畫葫蘆

接下來要測試這個方法：

```kotlin
private fun navigateToAddNewTask() {
    val action = TasksFragmentDirections
        .actionTasksFragmentToAddEditTaskFragment(
            null,
            resources.getString(R.string.add_task)
        )
    findNavController().navigate(action)
}
```

依樣畫葫蘆就成這樣了：

```kotlin
@Test
fun clickAddTaskButton_navigateToAddEditFragment() {
    // GIVEN - On the home screen
    val scenario = launchFragmentInContainer<TasksFragment>(Bundle(), R.style.AppTheme)
    val navController = mock(NavController::class.java)
    scenario.onFragment {
        Navigation.setViewNavController(it.view!!, navController)
    }

    // WHEN - Click on the "+" button
    onView(withId(R.id.add_task_fab)).perform(click())

    // THEN - Verify that we navigate to the add screen
    verify(navController).navigate(
        TasksFragmentDirections.actionTasksFragmentToAddEditTaskFragment(
            null, getApplicationContext<Context>().getString(R.string.add_task)
        )
    )
}
```

# 錯誤訊息

## Database_Impl not Found

記住要將 [所需要的 dependencies](https://developer.android.com/jetpack/androidx/releases/room#feedback) 都加上並更新。

```groovy
val roomVersion = "2.5.2"
implementation("androidx.room:room-ktx:$roomVersion")
ksp("androidx.room:room-compiler:$roomVersion")
implementation("androidx.room:room-runtime:$roomVersion")
```

另外，一定要更新 `implementation "androidx.core:core-ktx:1.12.0"`

<br><br><br><br><br><br><br><br><br><br>
