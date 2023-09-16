---
layout: post
title:  "Android: 如何存取"
date:   2022-08-08 16:15:07 +0800
categories: [android, permission]
---

#背景

這篇文章會講述如何使用 Android 的存取相關的 Permission。

因為 Permission 眾多且會不斷更新，因此這文章也會不斷地更新。

另外，為了讓大家知道自己的 SDK 的版本，這裡有 [SDK 平台版本資訊](https://developer.android.com/studio/releases/platforms) 供參考。

目前已包含的 Permission 包括以下：

|Permission|Constants|
|--|:--|
|[Storage](#storage)   | android.permission.<br><b>WRITE_EXTERNAL_STORAGE</b> (API 19)<br><b>READ_EXTERNAL_STORAGE</b> (API 19) <br><b>MANAGE_EXTERNAL_STORAGE</b> (API 33)<br><b>READ_MEDIA_IMAGES</b> (API 33)<br><b>READ_MEDIA_VIDEO</b> (API 33)<br><b>READ_MEDIA_VISUAL_USER_SELECTED</b> (API 34) |


<a id = "storage"></a>
# Storage

Android 資料可以分成兩種：

|種類|屬性|
|:--|:--|
|**App 專屬資料**   | - 可分為 **內存** 與 **外存**<br>- **只能** 由 **目標 App** **寫入** 及 **讀取**<br>- 當 App 被刪除時也會 **一同被刪除**<br>- 且 **空間有限** <br>- 另外，**外存** 可被其他 app 查看 |
| **共享資料**  | - 只存在於 **外存**<br>- 可以被多個 App 寫入或讀取 <br>- 包含 **媒體文件** 及 **其他文件** |

PS: 內存存在於裝置當中。 外存存在於 SD 卡。

[參考官方](https://developer.android.com/training/data-storage)

# Permissions 版本

當 App 需要進行讀寫時，一般我們只會需要 **部分** 以下的許可：

- android.permission.WRITE_EXTERNAL_STORAGE
- android.permission.READ_EXTERNAL_STORAGE
- android.permission.MANAGE_EXTERNAL_STORAGE
- android.permission.READ_MEDIA_IMAGES
- android.permission.READ_MEDIA_VIDEO

但是實際上需要哪些許可就要看你的 Android / API 版本了。

## API 34 或以上 (Upsidedown Cake)
|情況|所需許可|
|:--|:--|
|使用 Picker 點選媒體資料 <br>  |  READ_MEDIA_VISUAL_USER_SELECTED <br> 並配合<br>READ_MEDIA_IMAGES<br>或<br>READ_MEDIA_VIDEO |

## API 33 或以上 (Tiramisu)

|情況|所需許可|
|:--|:--|
|讀取或寫入 **共享** 資料 (全部)  | MANAGE_EXTERNAL_STORAGE  |
|讀取 **共享** 圖檔   |  READ_MEDIA_IMAGES |
|讀取 **共享** 影檔   | READ_MEDIA_VIDEO  |
|修改或刪除 **共享** 媒體資料   | MANAGE_EXTERNAL_STORAGE<br> MANAGE_MEDIA |

## API 19 或以上

|情況|所需許可|
|:--|:--|
|讀取或寫入 **App 專屬** 資料   | 無  |
|讀取或寫入 **共享** 資料 (全部)   |  READ_EXTERNAL_STORAGE<br>WRITE_EXTERNAL_STORAGE |

# Migration


# 如何使用

在了解如何寫入或讀取資料之前，我們需要知道如何取得資料的所在位置。

首先，我們來看看 App 專屬資料到底在哪吧。

## 資料的位置

<center>
<img src = "/images/posts/jekyll/android/storage/mind_graph_storage.jpg"/>
</center>

### 內存

我們可以兩個方法取得內存的絕對路徑：

```kotlin
// 實作位於 ContextImpl.java
context.filesDir // file 的絕對路徑
context.cacheDir // cache 的絕對路徑
```

取得內存的絕對路徑。 他會有以下的格式：

```kotlin
/data/user/0/{package_name}/{files 或 cache}

ie. /data/user/0/com.javanrhinos.lifepic.debug/files
```

如果我們想要知道目前有哪些資料存在於 `filesDir` 中，我們可以直接調用 `Context.fileList()` ：

```Kotlin
var files: Array<String> = context.fileList()
```

如果我們想要創建資料夾的時候，我們則可以使用 `getDir`：

```kotlin
val dirName = "newDir"
val dirF = context.getDir(dirName, Context.MODE_PRIVATE)
dirF.mkdir() // 若沒有這段，雖然下面的 f.list() 依舊會顯示 app_newDir，但卻沒有實質創建出來。
```

經過以上的行為後，我們可以使用 **File** 印出 `/data/user/0/` 中的資料夾：

```kotlin
val f = File(filesDir.parent)

Log.i("內存", "f = " + f)

f.list().forEach {
    Log.i("內存", "inner f = " + it)
}

/*
f = /data/user/0/com.javanrhinos.lifepic.debug
inner f = cache
inner f = code_cache
inner f = app_newDir // getDir 會在名稱前面加上 app_ 字串
inner f = files
*/
```

### 外存

外存還可分為： **App 專屬** 與 **共享**。

雖然兩者皆可被其他 App 任意讀取，但 App 專屬的資料會在卸載 App 後被刪除。

#### App 專屬

```kotlin
context.obbDirs // 取得 App 專屬的外存絕對路徑

// /storage/emulated/0/Android/obb/com.javanrhinos.lifepic.debug
```

外存也會有對應的 `filesDir` 與 `cacheDir` ：

```Kotlin
// filesDir = /storage/emulated/0/Android/data/com.javanrhinos.lifepic.debug/files
context.getExternalFilesDir(null)

// cacheDir = /storage/emulated/0/Android/data/com.javanrhinos.lifepic.debug/cache
context.externalCacheDir
```

以上兩者只會得到 **外存位置** 或 `emulator`。

雖然說我們是得到 *外存位置* 但是此位置依然存在手機內，因此這裡被成為 **Internal shared storage** (內部共享儲存)。

那我們要如何得到 **SD 卡** 的位置呢？

```kotlin
/*
  得到
    ExtFilesDirs (null) = /storage/emulated/0/Android/data/com.javanrhinos.lifepic.debug/files
    ExtFilesDirs (null) = /storage/1E08-1B1B/Android/data/com.javanrhinos.lifepic.debug/files

    IE08-1B1B 是 SD 卡的 FS_UUID。
*/
getExternalFilesDirs(null).forEach {
    Log.i("外存", "ExtFilesDirs (null) = "  + it)
}

/*
  得到
    ExtCacheDirs = /storage/emulated/0/Android/data/com.javanrhinos.lifepic.debug/cache
    ExtCacheDirs = /storage/1E08-1B1B/Android/data/com.javanrhinos.lifepic.debug/cache
*/
externalCacheDirs.forEach {
    Log.i("外存", "ExtCacheDirs = "  + it)
}
```

我們還可以得到媒體專屬資料位置：

```kotlin
/*
  得到
    ExtMediaDirs = /storage/emulated/0/Android/media/com.javanrhinos.lifepic.debug
    ExtMediaDirs = /storage/1E08-1B1B/Android/media/com.javanrhinos.lifepic.debug
*/
externalMediaDirs.forEach {
    Log.i("外存", "ExtMediaDirs = "  + it)
}
```

#### 共享

共享位置的資料不會隨著 App 的卸載被刪除。

另外，與其使用 **Context** 來取得資料位置，這裡需要使用 **Environment** ：

```kotlin
/*
  取得 /storage/emulated/0
*/
Environment.getExternalStorageDirectory()

/*
  取得 SD 卡的狀態，10 種之一：
    MEDIA_UNKNOWN,
    MEDIA_REMOVED,
    MEDIA_UNMOUNTED,
    MEDIA_CHECKING,
    MEDIA_NOFS,
    MEDIA_MOUNTED,
    MEDIA_MOUNTED_READ_ONLY,
    MEDIA_SHARED,
    MEDIA_BAD_REMOVAL, or
    MEDIA_UNMOUNTABLE.

  另外，我們還可以指定 File Path
    val file = File("my_file")
    EnvironmentCompat.getExternalStorageState(file)
*/
Environment.getExternalStorageState()

/*
  取得共享的指定資料夾。 此次得到的是：
    /storage/emulated/0/Movies

  除此之外，我們還可以選：
    DIRECTORY_MUSIC
    DIRECTORY_PODCASTS
    DIRECTORY_RINGTONES
    DIRECTORY_ALARMS
    DIRECTORY_NOTIFICATIONS
    DIRECTORY_PICTURES
    DIRECTORY_MOVIES
    DIRECTORY_DOWNLOADS
    DIRECTORY_DCIM
    DIRECTORY_DOCUMENTS
    DIRECTORY_SCREENSHOTS
    DIRECTORY_AUDIOBOOKS
    DIRECTORY_RECORDINGS
*/
Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_MOVIES)

```

除了共存資料外， **Environment** 還可以取得其他資料位置：

```kotlin
// 系統 root = /system
Environment.getRootDirectory()

// 內部的 root = /data
// 記得之前 /data/user/0/{package_name}/{files 或 cache}
Environment.getDataDirectory()

// /data/cache
// 這檔案中包含了 backup, backup_stage, recovery 三個檔案
Environment.getDownloadCacheDirectory()
```

##### 檢查外存狀態

```kotlin
// Checks if a volume containing external storage is available
// for read and write.
fun isExternalStorageWritable(): Boolean {
    return Environment.getExternalStorageState() == Environment.MEDIA_MOUNTED
}

// Checks if a volume containing external storage is available to at least read.
fun isExternalStorageReadable(): Boolean {
     return Environment.getExternalStorageState() in
        setOf(Environment.MEDIA_MOUNTED, Environment.MEDIA_MOUNTED_READ_ONLY)
}
```

#### 創建虛擬容器

在沒有外接存放容器時，我們可以使用以下 adb 命令來創建虛擬容器並進行測試：

```shell
adb shell sm set-virtual-disk true
```


## App 專屬讀取或寫入

在 **Manifest** 中我們不需要任何許可。
```xml
<!-- API 19 或以上無需任何許可來讀取或寫入 App 專屬資料 -->
```
### 內存

#### 新增檔案

如果我們想要寫入內存，我們可以直接使用 `Context.openFileOutput`

```kotlin
// 當 內存 中沒有 my_file 才新增。
if (!(filesDir.list()?.contains("my_file") ?: false)) {
    val name = "my_file"
    val mode = MODE_PRIVATE
    openFileOutput(name, mode)
}

// 印出 my_file
fileList().forEach { file ->
    Log.i("內存", "Files = " + file)
}
```

當然，除了針對 `files` ，我們還可以創建暫存資料：

```kotlin
/*
public static File createTempFile(String prefix,
                                  String suffix,
                                  File directory)
*/
File.createTempFile(filename, null, context.cacheDir)

// 我們也可以使用 File 來將其讀取
val cacheFile = File(context.cacheDir, filename)
```

#### 寫入資料

```Kotlin
/**
  寫入資料
    在 API 24 (Android 7.0) 若不是使用 Context.MODE_PRIVATE，
    就會得到 SecurityException 。
    也就是說 API 24 或以上都需要使用 Context.MODE_PRIVATE。
**/

// 當 內存 中沒有 my_file 才新增。
// 否則會被覆蓋

val filename = "my_file.txt"

if (!(filesDir.list()?.contains(filename) ?: false)) {
    val fileContents = "Hello world!"
    context.openFileOutput(filename, Context.MODE_PRIVATE).use {
            it.write(fileContents.toByteArray())
    }
}
```

#### 讀取資料

若想要讀取 `my_file.txt` 我們可以這樣寫：

```kotlin
val data = openFileInput(name)
val bytes = ByteArray(data.available())
val read = data.read(bytes)
val readResult = String(bytes)
```

#### 讀取 Raw 資料

若想要在安裝 App 時進行讀取，我們還可以將他存為 Raw 並以 Stream 讀取：
```kotlin
/**
  ie 讀取 database [ https://stackoverflow.com/a/945879/18597115 ]
**/

val ins = resources.openRawResource(R.raw.my_db_file);

val outputStream = ByteArrayOutputStream();

int size = 0;

// Read the entire resource into a local byte buffer.
byte[] buffer = new byte[1024];
while((size=ins.read(buffer,0,1024))>=0){
  outputStream.write(buffer,0,size);
}
ins.close();
buffer=outputStream.toByteArray();

FileOutputStream fos = new FileOutputStream("mycopy.db");

// 將 buffer 寫入 mycopy.db
fos.write(buffer);
fos.close();
```

類似的，我們也可以 [讀取圖案 甚至 gif](https://karnshah8890.blogspot.com/2013/12/get-images-from-raw-folder-using-only.html) ：

```java
/**
  讀取圖
**/
// 從名字來取得 R.id
int rid = context.getResources()
        .getIdentifier("image_name","raw", context.getPackageName());

Resources res = context.getResources();
InputStream in = res.openRawResource(rid);

Drawable image = Drawable.createFromStream(in, "image_name");

/**
  讀取 gif
**/
AnimationDrawable animation = new AnimationDrawable();
for (int count = 0; count < names.length; count++) {
 try {
  int rid = context.getResources().getIdentifier(names[count],
      "raw", context.getPackageName());
  Resources res = context.getResources();
  InputStream in = res.openRawResource(rid);

  byte[] b = new byte[in.available()];
  in.read(b);
  animation.addFrame(Drawable.createFromStream(in, names[count]),
      durationPerFrame);
 } catch (Exception e) {
  e.printStackTrace();
 }
}

animation.setOneShot(false);

```

#### 移除資料

```kotlin
val cacheFile = File(context.cacheDir, filename)
cacheFile.delete()

// 或

context.deleteFile(cacheFileName)
```

#### 其他範例

- [Saving and Reading Bitmaps/Images from Internal memory in Android](https://stackoverflow.com/questions/17674634/saving-and-reading-bitmaps-images-from-internal-memory-in-android)


### 外存


#### 創建檔案

在這裡我們無法使用 `openFileOutput` 來寫入。

我們必須要用 **Intent** 通知系統來進行寫入，因此我們會使用 `Intent.ACTION_CREATE_DOCUMENT` 。

另外，不像 `openFileOutput`，外存是不會被覆寫。
當創建相同名稱的檔案時，系統會在名稱後面加上 `({數字})` 像是 **confirmation(1).pdf** 。

```kotlin
/*
  來源： https://developer.android.com/training/data-storage/shared/documents-files#create-file

  以下方法會跳出 File Picker 讓使用者做兩件事：
    1. 選擇 File Picker 初始位置
    2. 再次顯示 File Picker 並讓使用者存放 invoice.pdf。
*/

// Request code for creating a PDF document.
const val CREATE_FILE = 1

// 1. 開啟 File Picker
//    讓使用者選擇 File Picker 初始的資料位置。
private fun openFilePicker() {
    val request = registerForActivityResult(ActivityResultContracts.StartActivityForResult()) { result: ActivityResult ->
        if (result.resultCode == Activity.RESULT_OK ) {
          result.data?.let {
              // 2. 進行資料的創建
              createFile(it.data)
          }
        }
    }

    // 這是進行 intent 開啟 File Picker
    val intent = Intent(Intent.ACTION_OPEN_DOCUMENT_TREE)

    request.launch(intent)
}

// 3. 開啟 File Picker 並寫入資料
private fun createFile(pickerInitialUri: Uri) {
    val intent = Intent(Intent.ACTION_CREATE_DOCUMENT).apply {
        addCategory(Intent.CATEGORY_OPENABLE)
        type = "application/pdf"
        putExtra(Intent.EXTRA_TITLE, "invoice.pdf")

        // Optionally, specify a URI for the directory that should be opened in
        // the system file picker before your app creates the document.

        // API 26 或以上 才能指定初始資料夾
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            putExtra(DocumentsContract.EXTRA_INITIAL_URI, pickerInitialUri)
        }
    }
    startActivityForResult(intent, CREATE_FILE)
}
```

如同內存，我們也可以創建和移除外存的暫存 ：

```kotlin
val externalCacheFile = File(context.externalCacheDir, filename)

externalCacheFile.delete()
```

#### 創建多媒體資料

之前我們在調用 `getExternalFilesDir` 時都只是帶入 `null`。 這會讓我們取得外存的 `file` ：

```shell
/storage/emulated/0/Android/data/com.javanrhinos.lifepic.debug/files
```

但若我們想要在 `file` 中特定的資料夾中進行存取呢？

我們可以使用 `Environment.DIRECTORY_xxx` ：

```kotlin
fun getAppSpecificAlbumStorageDir(context: Context, albumName: String): File? {
    // Get the pictures directory that's inside the app-specific directory on
    // external storage.
    val file = File(context.getExternalFilesDir(
                      Environment.DIRECTORY_PICTURES),
                    albumName)

    // 2. 創建 dir
    if (!file?.mkdirs()) {
        Log.e(LOG_TAG, "Directory not created")
    }
    return file
}
```


#### 選取目標檔案位置

如果我們想要選擇檔案位置，我們可以調用 `getExternalFilesDirs` 來取得：

```kotlin
val externalStorageVolumes: Array<out File> =
        ContextCompat.getExternalFilesDirs(applicationContext, null)

val primaryExternalStorage = externalStorageVolumes[0]
```

但是，在 **Android 4.3 (API 18) 或以下** 只會有一個 element。

#### 確保容量足夠

當我們要存放資料的時候，我們可以先檢查手機內的容量是否足夠。

[官方]() 給的範例只支援 API 26 或以上。
翻查了一下，找到唯一可信的 [實作](https://stackoverflow.com/questions/66859325/how-to-get-directorys-uuid-using-storagemanager-on-android-api-below-26)。

```kotlin

// 由於 UUID 是在 API 26 才出來的，所以需要別的作法
// 另外，這方法需要在 Work Thread 進行。

@SuppressLint("NewApi")
fun Context.hasFreeSpace(directory: File, requiredStorageSpace: Long): Boolean {
    return try {
        val api = Build.VERSION.SDK_INT
        val availableBytes = when {
            api >= Build.VERSION_CODES.O -> {
                val storageManager = getSystemService<StorageManager>()!!
                val appSpecificInternalDirUuid = storageManager.getUuidForPath(directory)
                val availableBytes: Long = storageManager.getAllocatableBytes(directoryUUID)

                if (availableBytes >= requiredStorageSpace) {
                    storageManager.allocateBytes(
                        appSpecificInternalDirUuid, NUM_BYTES_NEEDED_FOR_MY_APP)
                }
            }
            else -> {
                val stat = StatFs(directory.path)
                stat.availableBlocksLong * stat.blockSizeLong
            }
        }
        availableBytes > requiredStorageSpace
    } catch (e: Exception) {
        e.printStackTrace()
        false
    }
}

// App needs 10 MB within internal storage.
const val NUM_BYTES_NEEDED_FOR_MY_APP = 1024 * 1024 * 10L;

if (!context.hasFreeSpace(directory, NUM_BYTES_NEEDED_FOR_MY_APP)) {
  if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N_MR1) {
    // api 25
    val storageIntent = Intent().apply {
        // To request that the user remove all app cache files instead, set
        // "action" to ACTION_CLEAR_APP_CACHE.
        action = ACTION_MANAGE_STORAGE
    }
  } else {
    // 顯示不夠容量
  }
}

```

當然，我們也不一定需要進行檢查，我們也可以先儲存然後抓住 **IOException** 。

接下來，如果得知空間不足時，我們可以進行以下行為：

1. 建立 app 專屬的空間管理 Activty ：
   - 我們需要在 Manifest 中將此 activity 加在 `android:manageSpaceActivity` 中 。
   - 另外，此 activity 的 `android:exported` 可設為 `false`。
   - 如此一來 File Manager App 可以啟動此 activity。
<br>

2. 要求使用者刪除裝置資料 ( >= API 25)
   - 這裡會使用 `ACTION_MANAGE_STORAGE` 來進行。
   - 另外，我們還可以用以下算法 Toast 出目前所剩空間：
     ```kotlin
     val storageStateManager = applicationContext.getSystemService<StorageStatsManager>()!!

     // uuid 可從上面的 code 取得
     storageStateManager.getFreeBytes(appSpecificInternalDirUuid) / storageStateManager.getTotalBytes(appSpecificInternalDirUuid)
     ```
<br>

3. 要求使用者將 **全部** App 的暫存都清除 ( >= API 30)
   - 我們只需要發出 `ACTION_CLEAR_APP_CACHE` 的 **Intent** 即可。


https://easy-coding.tistory.com/97

[Android之uri、file、path相互转化
](https://blog.csdn.net/hust_twj/article/details/76665294)

[Android 11 Scoped storage permissions](https://stackoverflow.com/questions/62782648/android-11-scoped-storage-permissions)

[一篇文章彻底明白Android文件存储](https://blog.csdn.net/Coo123_/article/details/98485065)


## 共享 : 外存資料

共享資料可以分成 [四種](https://developer.android.com/training/data-storage/shared)：

1. 多媒體 (Media)
   - Android 會提供一個 standard 公開資料位置。
   - 這裡包含照片、音樂、電影等等的資料。 他們都會有個別的資料夾來管理。
   - 我們可以使用 **MediaStore** API 來進行互動

<br>

2. 文件及其他資料 (Document and other files)
   - 系統會有特別的資料夾來存放這些資料，包括 pdf 或 epub。
   - 我們可以使用 **Storage Access Framework** 來讀取

<br>

3. 數據集 (Datasets)
   - 在 Android 11 ( **API 30** ) 以上，系統會將多個 App 共用的資料暫存起來。
   - 這些資料會使用在 **機器學習** 和 **媒體播放**
   - 若想要取得這些資料，我們需要使用 [**BlobManager**](https://developer.android.com/reference/android/app/blob/BlobStoreManager) API 。

<br>

4. 照片挑選 (Photo Picker)
   - 這裡允許我們進行圖像與影片的挑選
   - 我們需要使用 `PickVisualMedia` 與 `PickMultipleVisualMedia` API。






## 讀取其他 App 專屬檔案


# 共享 Media

官方：[Access media files from shared storage](https://developer.android.com/training/data-storage/shared/media)

想要讀取或寫入多媒體，我們可以使用 **PhotoPicker** 與 **MediaStore**。

**PhotoPicker** 如同其名，是一個用來進行選取圖像與影像的 API。

但如果我們想要顯示圖像、影像 與 音檔 時，我們就可以使用 **MediaStore**。

## MediaStore

**MediaStore** 可以讓我們讀取以下四種類型：

|類型|說明|
|:--|:--|
|**圖像**   | 位置： **DCIM/** 與 **Pictures/** 資歷夾的 **照片** 與 **截圖** 。 <br><br>這些檔案資料皆會被系統存放在 **MediaStore.Images** 表格中。  |
|**影像**   | 位置： **Movies/**、 **DCIM/** 與 **Pictures/** 資歷夾的檔案。 <br><br>這些檔案資料皆會被系統存放在 **MediaStore.Video** 表格中。 |
|**音檔**   | 位置： **Alarms/**、 **AudioBooks/** 、 **Music/** 、 **Notifications/**、 **Podcasts/** 與 **Ringtones/** 中的檔案。 <br><br>除此之外<br>被系統存放在 **Music/** 與 **Movies/** 以及 **Recordings/** ( > **API 30** Android 11 ) 中的音檔也會被讀取。 <br><br>這些檔案資料皆會被系統存放在 **MediaStore.Audio** 表格中。  |
|**下載**   | 位置： **Download/** <br><br>  **API 29** (Android 10) 時，資料會被放在 **MediaStore.Downloads** 中。  |






# ...
若當下是 **Android 29+** ，那我們就不需要任何 Permission， 因此我們只需要有以下 `user-permission` ：

```xml
<uses-permission
    android:name="android.permission.WRITE_EXTERNAL_STORAGE"
    android:maxSdkVersion="28"/>
<uses-permission
    android:name="android.permission.READ_EXTERNAL_STORAGE"
    android:maxSdkVersion="28"/>
<!-- 注意，這裡是將 maxSdkVersion 設為 28 -->
```

## 使用 app 所持有的以外檔案

**READ_EXTERNAL_STORAGE** 對 **Android 33+** (Tiramisu) 是 **無效** 的。 取而待之的是 **READ_MEDIA_VIDEO** 與 **READ_MEDIA_IMAGES** 。

```xml
<uses-permission
    android:name="android.permission.READ_MEDIA_IMAGES"/>
<uses-permission
    android:name="android.permission.READ_MEDIA_VIDEO"/>

<uses-permission
    android:name="android.permission.WRITE_EXTERNAL_STORAGE"
    android:maxSdkVersion="28"/>
<uses-permission
    android:name="android.permission.READ_EXTERNAL_STORAGE"
    android:maxSdkVersion="32"/>
```

## 共享檔案的









<br><br>
