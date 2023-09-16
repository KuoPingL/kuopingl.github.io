---
layout: post
title:  "Android: 使用 SharedPreferences 與 DataStore"
date:   2022-08-08 16:15:07 +0800
categories: [android, sharedPreference, datastore]
---

# 背景

在寫 App 時，我們會時常遇到一種情況，那就是我們希望記錄使用者的某些選項。 包括：

  - 登入名稱 與 密碼
  - 是否有勾取「不要顯示」的選項
  - 上次所在的畫面 等等

當我們遇到這些情況，我們一般不會使用 Database 來存放。
因為我們所需要存放的資料其實很小。 取而代之，我們會使用 **SharedPreferences** 或 **DataStore** 來存放。

這篇我們會看看我們如何使用這兩者，以及他們之間的不同。

# SharedPreferences

```java
public interface SharedPreferences
```

<center>
<img src = "/images/posts/jekyll/android/sharedPreference/sharedpreference-mindmap.png"/>
</center>

**SharedPreferences** 是一個介面。
他的功能是讓我們可以通過 `setter` 與 `getter` 存取資料。

其中最為重要的是，他所接受的類型是 `String`, `int`, `long`, `float` 與 `boolean`。

這表示如果我們想要存取物件，我們需要用字串來將其存起來。
最直接的作法就是將物件當中的屬性使用 **JSON** 格式存放，並在讀取時進行解析。


一般來說，我們 App 會在多個地方使用 **SharedPreferences**。 因此如果我們可以用 *Singleton* ，我們就不需要每次都創建他了，如同以下：

```kotlin
object SharedPref {

    private lateinit var mSharedPref: SharedPreferences

    private const val SHARED_PREF_NAME = "lifePic"

    fun init(context: Context) {
        mSharedPref = context.getSharedPreferences(context.packageName, MODE_PRIVATE)
    }

    fun getRequestCountForPermission(key: String) =
        mSharedPref.getInt(key, 0);

    fun setRequestCountForPermission(key: String, value: Int) =
        mSharedPref.edit().putInt(key, value).apply()
}

// 首次創建 SharedPref 時，我們可以在 Application 中設定

fun onCreate() {
  SharedPref.init(this)
}


// 然後在其他地方運用時我們只需要寫

val count = SharedPref.getRequestCountForStoragePermission("Storage")

```

以上的實作其實很簡單，但我們順便看看 **SharedPreferences** 的實作吧。


## 創建

首現，我們要看的是 **SharedPreferences** 的創建。

**SharedPreferences** 的創建需要使用 **Context** 與 對應的 **名稱**：

```java
context.getSharedPreferences(String name, @PreferencesMode int mode)
```

當中的 `mode` 可以是以下 4 種。 但要注意 `MODE_WORLD_READABLE` 與 `MODE_WORLD_WRITEABLE` 無法在 **API 24** 以上使用:

```java
@IntDef(flag = true, prefix = { "MODE_" }, value = {
        MODE_PRIVATE,
        MODE_WORLD_READABLE,  // API 24 以下
        MODE_WORLD_WRITEABLE, // API 24 以下
        MODE_MULTI_PROCESS,
})
@Retention(RetentionPolicy.SOURCE)
public @interface PreferencesMode {}
```

### ContextImpl 中的實作

我們還可以進入 **ContextImpl** 中查看 `getSharedPreferences` 的實作。

```java
@Override
public SharedPreferences getSharedPreferences(String name, int mode) {
    // At least one application in the world actually passes in a null
    // name.  This happened to work because when we generated the file name
    // we would stringify it to "null.xml".  Nice.

    // 1. 在 API 19 以下，會確認 name != null
    if (mPackageInfo.getApplicationInfo().targetSdkVersion <
            Build.VERSION_CODES.KITKAT) {
        if (name == null) {
            name = "null";
        }
    }

    File file;

    // 2. 進行同步化 (每次只能一個線程讀取)
    synchronized (ContextImpl.class) {

        // 3. 創建 ArrayMap<String, File>
        if (mSharedPrefsPaths == null) {
            mSharedPrefsPaths = new ArrayMap<>();
        }

        // 4. 在 dataDir 中建立 shared_prefs.xml
        //    另外，這裡也會檢查 mode
        file = mSharedPrefsPaths.get(name);
        if (file == null) {
            file = getSharedPreferencesPath(name);
            mSharedPrefsPaths.put(name, file);
        }
    }

    // 5. 建立 SharedPreferencesImpl
    //    並存放至 cache : ArrayMap<File, SharedPreferencesImpl>
    //    中
    //
    //    這方法也會對 ContextImpl.class 進行同步化
    return getSharedPreferences(file, mode);
}
```

### SharedPreferencesImpl 的實作

由於實作的源碼太長，我們就看重點即可。

因為 **SharedPreferences** 其實就是進行 xml 的存取與修改，所以 **SharedPreferencesImpl** 中必須會調用 `synchronized` 來確保 data racing 不會發生。

```Java

final class SharedPreferencesImpl implements SharedPreferences {
    private final Object mLock = new Object();

    // 我們看看 getter 吧
    @GuardedBy("mLock")
    private Map<String, Object> mMap;

    @Override
    public Map<String, ?> getAll() {
        // 1. 進行同步
        synchronized (mLock) {

            // 2. 等待資料準備好
            //    等待期間會不斷調用 mLock.wait()
            awaitLoadedLocked();

            //noinspection unchecked

            // 3. 回傳 mMap
            return new HashMap<String, Object>(mMap);
        }
    }

    // 取得 Editor
    @Override
    public Editor edit() {
        synchronized (mLock) {
            awaitLoadedLocked();
        }

        return new EditorImpl();
    }
}
```

你應該發現 **SharedPreferencesImpl** 的主要功能是進行讀取 (getters)。

如果我們想要寫入，那就需要暸解 **SharedPreferences.Editor** 了。

## EditorImpl 的實作

**EditorImpl** 與 **SharedPreferencesImpl** 中的實作都會需要對物件同步化，這是為了要確保資料的準確性。

```java
public final class EditorImpl implements Editor {
    private final Object mEditorLock = new Object();

    @GuardedBy("mEditorLock")
    private final Map<String, Object> mModified = new HashMap<>();

    @GuardedBy("mEditorLock")
    private boolean mClear = false;

    // 我們來看看 setter 吧
    @Override
    public Editor putString(String key, @Nullable String value) {
         synchronized (mEditorLock) {
             mModified.put(key, value);
             return this;
         }
     }
}
```

當我們將資料寫入 `mModified` 後，我們需要將資料寫入 xml 中。 因此，我們需要調用 `apply` 或 `commit` ：

```java
@Override
public void apply() {
    final long startTime = System.currentTimeMillis();

    /*
        1. 這裡會同步化 SharedPreferencesImpl.this.mLock
           如此一來便確保寫入無法與讀取一同進行。

           這方法會回傳一個 MemoryCommitResult
           當中會包含 更新資料 與 Listeners
    */
    final MemoryCommitResult mcr = commitToMemory();

    /*
        2. 建立一個 Runnable 來等待 writtenToDisk 已完成
    */
    final Runnable awaitCommit = new Runnable() {
            @Override
            public void run() {
                try {
                    //  這是一個 CountDownLatch
                    //  當 countdown == 0 時
                    //  就表示可以往下進行
                    mcr.writtenToDiskLatch.await();
                } catch (InterruptedException ignored) {
                }

                if (DEBUG && mcr.wasWritten) {
                    Log.d(TAG, mFile.getName() + ":" + mcr.memoryStateGeneration
                            + " applied after " + (System.currentTimeMillis() - startTime)
                            + " ms");
                }
            }
        };

    // 3. QueuedWork 會以 LinkedList 的方式存放 Runnable
    //    而且是存在 static final 的 LinkedList<Runnable> 中
    QueuedWork.addFinisher(awaitCommit);

    /*
      4. 建立 Runnable 來進行兩件事：
         a. 進行 commit
         b. 將 awaitCommit 從 LinkedList 中移除
    */
    Runnable postWriteRunnable = new Runnable() {
            @Override
            public void run() {
                awaitCommit.run();
                QueuedWork.removeFinisher(awaitCommit);
            }
        };

    /*
      5. 建立 writeToDiskRunnable 來進行 MemoryCommitResult 的寫入。
         而 postWriteRunnable 會在最後進行 run
         並讓下一個更改進行

         另外， writeToDiskRunnable 會經由 QueuedWork.queue 放入另一個 LinkedList<Runnable> 中。

         並由 handler :
         HandlerThread("queued-work-looper",
                        Process.THREAD_PRIORITY_FOREGROUND);
        sHandler = new QueuedWorkHandler(handlerThread.getLooper());

        來進行
    */
    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

    // Okay to notify the listeners before it's hit disk
    // because the listeners should always get the same
    // SharedPreferences instance back, which has the
    // changes reflected in memory.

    // 6. 最後便是通知註冊的 OnSharedPreferenceChangeListener 了
    notifyListeners(mcr);
}
```

以下是 `commit`。 從源碼可以看出其實與 `apply` 差不多。 主要不同在於 `awaitCommit` 與 `postWriteRunnable` 了。

注意這裡再調用 `SharedPreferencesImpl.this.enqueueDiskWrite` 時所傳入的 **Runnable** 是 `null` 。

當我們將 `null` 帶入時， `writeToDiskRunnable` 便只會在 **UI Thread** 進行。

```java
@Override
public boolean commit() {
    long startTime = 0;

    if (DEBUG) {
        startTime = System.currentTimeMillis();
    }

    MemoryCommitResult mcr = commitToMemory();

    SharedPreferencesImpl.this.enqueueDiskWrite(
        mcr, null /* sync write on this thread okay */);
    try {
        mcr.writtenToDiskLatch.await();
    } catch (InterruptedException e) {
        return false;
    } finally {
        if (DEBUG) {
            Log.d(TAG, mFile.getName() + ":" + mcr.memoryStateGeneration
                    + " committed after " + (System.currentTimeMillis() - startTime)
                    + " ms");
        }
    }

    notifyListeners(mcr);
    return mcr.writeToDiskResult;
}
```

接下來，我們就要看看 **DataStore** 了。

# DataStore

根據官方， **DataStore** 是以非同步、一致且交易方式儲存資料，並克服 **SharedPreferences** 的 **某些缺點**

我們可以從 [codelabs](https://developer.android.com/codelabs/android-preferences-datastore#0) 了解。

||**PreferencesDataStore**|**ProtoDataStore**|
|:--|:--|:--|
|存取方式   | 如同 **SharedPreferences**， 使用 `key` 來存取資料  | 需要定義 [**Protocol Buffers**](https://protobuf.dev/) 來存取物件 |


## Manifest

想要使用 **DataStore** 會需要定義 Manifest

[PreferencesDataStore](https://developer.android.com/jetpack/androidx/releases/datastore?hl=zh-tw#preferences-datastore-dependencies)


```groovy
// Preferences DataStore (SharedPreferences like APIs)
dependencies {
    implementation("androidx.datastore:datastore-preferences:1.0.0")

    // optional - RxJava2 support
    implementation("androidx.datastore:datastore-preferences-rxjava2:1.0.0")

    // optional - RxJava3 support
    implementation("androidx.datastore:datastore-preferences-rxjava3:1.0.0")
}

// Alternatively - use the following artifact without an Android dependency.
dependencies {
    implementation("androidx.datastore:datastore-preferences-core:1.0.0")
}

```

[ProtoDataStore](https://developer.android.com/jetpack/androidx/releases/datastore?hl=zh-tw#proto-datastore-dependencies)

```groovy
// Typed DataStore (Typed API surface, such as Proto)
dependencies {
    implementation("androidx.datastore:datastore:1.0.0")

    // optional - RxJava2 support
    implementation("androidx.datastore:datastore-rxjava2:1.0.0")

    // optional - RxJava3 support
    implementation("androidx.datastore:datastore-rxjava3:1.0.0")
}

// Alternatively - use the following artifact without an Android dependency.
dependencies {
    implementation("androidx.datastore:datastore-core:1.0.0")
}

```

## 使用方式

1. 使用 Singleton 來使用 **DataStore**，否則會得到 **IllegalStateException**。
2. **DataStore** 的泛型必須不為不變的。
3. 當針對一個檔案運用時，不要混搭 **SingleProcessDataStore** 和 **MultiProcessDataStore**。 如果想要由多個 process 來存取 **DataStore**，那就使用 **MultiProcessDataStore** 即可。



### PreferencesDataStore

這類別使用 **DataStore** 與 **Preferences** 來運行。




### ProtoDataStore


```kotlin
```




<br><br><br><br><br><br><br>
