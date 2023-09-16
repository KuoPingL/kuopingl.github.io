---
layout: post
title: Simple Player 1 - Query 解說
categories: [Projects]
use_math: true
keywords: android, project, mediaPlayer
---

# 專案簡介
不知道你是否跟我一樣，平時會從不同地方 (TikTok, Line ... ) 下載喜歡的影片然後希望之後可以慢慢回味。 但當我們使用相簿 App 開啟時才發現，我們無法無限地播放影片，更無法將影片歸入播放清單。
Â
為了解決這問題，我決定建立一個簡單的 Media Player App ( 取名 **SMP Simple Media mediaPlayer** )來提供使用者一個可以好好享受音樂的平台。

## 功能介紹
1. 播放有效的影音檔 ( midi, wav, mp3, mp4, ogg, 3gp, amr, aac, flac, mkv, webm )
2. 記錄使用者最近的播放內容
3. 使用者可以建立 PlayList
4. 讓使用者可以通過 BlueTooth 分享音樂
5. ( optional ) 建立 Server 讓使用者可以存放影音檔，並可以隨時下載與上傳。
6. ( optional ) 讓使用者之間進行串流。
7. ( optional ) 提供免費的網站音樂讓使用者串流與下載。

## UI
UI 將才取 TikTok Vibe：

<center>
<img src = "/images/posts/jekyll/projects/smp/tiktok-logo-color-palette.jpg" style="width:50%"/>
</center>

而影片與音樂


# 解說
## 存取影片功能

**Manifest**

```groovy
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

**Permission Request**
在這裡我將 Permission 相關的行為寫入 singleton PermissionUtils。 如此一來我們就可以通過 PermissionUtils 進行 Permission 的請求、顯示 Permission 的 Rationale、以及導向 Settings 的功能。

另外還搭配 **SharedPreferences** 來記錄請求次數。
因為 Google 要求請求次數在 2 次之後就無法再次進行 Permission 的請求。此時我們就必須讓使用者導向 Settings 來手動更改。

<details>
<summary>PermissionUtils</summary>

```kotlin
object PermissionUtils {

    /**
     * Request Code for Storage
     */
    const val STORAGE_REQUEST_CODE = 100

    /**
     * check if storage permission is granted
     * @return true if permission is granted. You can call #requestStoragePermission afterward
     */
    fun isStoragePermissionGranted(context: Context): Boolean {
        return if (Build.VERSION.SDK_INT <= Build.VERSION_CODES.P) {
            checkPermissions(context, Manifest.permission.READ_EXTERNAL_STORAGE, Manifest.permission.WRITE_EXTERNAL_STORAGE)
        }
        else if (Build.VERSION.SDK_INT < Build.VERSION_CODES.TIRAMISU) {
            checkPermissions(context, Manifest.permission.READ_EXTERNAL_STORAGE)
        } else {
            checkPermissions(context, Manifest.permission.READ_MEDIA_VIDEO)
        }
    }

    private fun checkPermissions(context: Context, vararg permissions: String): Boolean {
        for (permission in permissions) {
            if (ContextCompat.checkSelfPermission(context, permission) == PackageManager.PERMISSION_DENIED) {
                return false
            }
        }
        return true
    }

    fun handleStoragePermissionResult(permissions: Array<out String>,
                                      grantResults: IntArray,
                                      context: AppCompatActivity) {
        val nonGrantedPermissions = arrayListOf<String>()

        for (i in 0 until grantResults.size) {
            if (grantResults[i] != PackageManager.PERMISSION_GRANTED) {
                nonGrantedPermissions.add(permissions[i])
            }
        }

        if (nonGrantedPermissions.size > 0) {

            val shouldRedirectToSettings = SharedPref.getRequestCountForStoragePermission(
                STORAGE_PERMISSION_REQUEST_COUNT) >= 1

            showRationaleForStoragePermission(context, shouldRedirectToSettings, onPositiveClicked = {

                if (shouldRedirectToSettings) {
                    // direct to setting
                    (context.application as SMPApplication).goToSettings()
                } else {
                    SharedPref.incrementStoragePermissionCounter(STORAGE_PERMISSION_REQUEST_COUNT)
                    requestStoragePermission(context)
                }

                it.dismiss()
            }, onNegativeClicked = {
                it.dismiss()
            })

        } else {
//                    val t = MediaStore.Images.Media.INTERNAL_CONTENT_URI
            val t = MediaStore.Files.getContentUri("external")
            Log.i("TAG", t.path?:"NONE")
        }
    }

    /**
     *
     */
    fun requestStoragePermission(activity: Activity): Boolean {

        val permissions = arrayListOf(Manifest.permission.READ_EXTERNAL_STORAGE, Manifest.permission.WRITE_EXTERNAL_STORAGE)


        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q /*29*/) {
            permissions.clear()

            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
                permissions.add(Manifest.permission.READ_MEDIA_VIDEO)
            } else {
                permissions.add(Manifest.permission.READ_EXTERNAL_STORAGE)
            }
        }

        for (permission in permissions) {
            if (ContextCompat.checkSelfPermission(activity, permission) == PackageManager.PERMISSION_DENIED) {
                ActivityCompat.requestPermissions(activity, permissions.toTypedArray(), STORAGE_REQUEST_CODE)
                return false;
            }
        }

        return true;
    }

    /**
     *
     */
    private fun showRationaleForStoragePermission(context: AppCompatActivity,
                                          shouldRedirectToSettings: Boolean = false,
                                          onPositiveClicked:(DialogInterface)->Unit, onNegativeClicked: (DialogInterface)->Unit ) {
        // create a dialog
        val dia = AlertDialog.Builder(context).apply {
            setCancelable(false)
            setTitle(context.resources.getString(R.string.permission_required_title))
            setMessage(context.resources.getString( if (shouldRedirectToSettings) R.string.msg_storage_permission_denied_warning else R.string.msg_storage_permission_required))

            setPositiveButton(
                context.resources.getString(if (shouldRedirectToSettings) R.string.btn_settings
                else R.string.btn_ok)) { dialog: DialogInterface, which: Int ->
                onPositiveClicked(dialog)
            }

            setNegativeButton(context.resources.getString(R.string.btn_cancel)) {dialog: DialogInterface, which: Int ->
                onNegativeClicked(dialog)
            }
        }

        dia.show()
    }

}
```

</details>

<br>



## Media Query
想要進行多媒體的讀取，我們可以從兩個地方著手：
- **MediaStore** (官方推薦)
- **File** (主要用來開啟 App 專屬檔案，但也可以開啟 非專屬檔案)

><br>
>
>**MediaStore** is the contract between the media provider and applications.
>
> 所以 App 可以通過 **MediaStore** 與 **MediaProvider** 溝通。 其中的 **MediaProvider** 其實就是 **ContentProvider**。
><br>

在這因為我其實不了解這兩者的差別，所以我都會去試試看。

### 基礎流程
在說 **MediaStore** 與 **File** 的使用方法之前，我們需要先了解 Query 的流程。

1. 取得資料夾的位置 ( **Uri** )。
2. 定義想要取得的資訊 (`String[] projection`)
3. 定義針對資訊的要求 (`String selection`)
4. 定義排序 (`String sortOrder`)
5. 通過 **ContentResolver** 以及以上的資訊跟系統的 **ContentProvider** 拿資料 (query)。
6. 得到資料後就使用 **Cursor** 取得對應的資料並寫入客製化的類別。

那我們接下來就要看如何使用 **MediaStore** 以及 **File** 得到我們想要的資料。

### MediaStore

#### 取得 Uri
假設我們需要取得 App 外部或非專屬或共享的多媒體，我們就可以通過以下方式取得：

```kotlin
// 官方源碼
val collection =
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        MediaStore.Video.Media.getContentUri(
            MediaStore.VOLUME_EXTERNAL
        )
    } else {
        MediaStore.Video.Media.EXTERNAL_CONTENT_URI
    }

```

雖然 MediaStore.class 中顯示 `MediaStore.Video.Media.EXTERNAL_CONTENT_URI` 為 null，但運作時會回傳：

| **content://media/external/video/media**

也就是以一個 Uri 的形式顯示：

| **{scheme} : // {host or authority} / {path}**

這也是我們需要用的，因為 SMP 並不會創造新的 Media，所以沒必要讀取專屬資料。

| **那如果想要取得圖檔或是圖跟影片檔呢？**

我們可以使用：
```kotlin
MediaStore.Files.getContentUri("external")
```

我們會取得：

| **content://media/external/file**

#### 定義 Projection
接下來我們需要定義我們想要在這些檔案中取得的資訊，像是名稱、 ID、影片長度、音效是否為鬧鐘、等等。

我們可以從 MediaStore 取得資料類型所支援的 Columns 或 資料名稱。

- [Audio](https://developer.android.com/reference/android/provider/MediaStore.Audio.AudioColumns) - `MediaStore.Audio.AudioColumns `
- [Video](https://developer.android.com/reference/android/provider/MediaStore.Video.VideoColumns) - `MediaStore.Video.VideoColumns`
- [Image](https://developer.android.com/reference/android/provider/MediaStore.Images.ImageColumns) - `MediaStore.Images.ImageColumns`
- [File](https://developer.android.com/reference/android/provider/MediaStore.Files.FileColumns) - `MediaStore.File.FileColumns`

若是從 MediaStore.File 取得，那我們可以通過 **selection** 來指定要取出哪些類型的資料，包括： Document, Audio, Video, Image, Subtitle, PlayList (31 deprecated) 還有 None，也就是除之前所提到的類型之外的類型。

因為我們想要取得影片與音樂，所以我們就使用 MediaStore.File。

另外，由於 `MediaStore.Media.MediaColumns` 是上述全部類型 Columns 的父類別。 所以我們的 FileColumns 可以寫成：

```kotlin

fun getFileColumns(): Array<String> {
    val columns = getMediaColumns().toMutableSet()

    columns.add(MediaStore.Files.FileColumns.MEDIA_TYPE)
    columns.add(MediaStore.Files.FileColumns.MIME_TYPE)
    columns.add(MediaStore.Files.FileColumns.PARENT) // The index of the parent directory of the file

    return columns.toTypedArray()
}
```

當然，如果我們不需要全部的資訊時，我們可以隨意挑選。

以下是其他 Columns 的方法：

<details>
<summary>其他 Columns 的取得方法 </summary>

```kotlin
enum class TargetTypes {
    VIDEO,
    AUDIO,
    IMAGE,
    ALL
}

val imageInternalUri: Uri = MediaStore.Images.Media.INTERNAL_CONTENT_URI
val imageExternalUri: Uri = MediaStore.Images.Media.EXTERNAL_CONTENT_URI
val videoInternalUri: Uri = MediaStore.Video.Media.INTERNAL_CONTENT_URI
val videoExternalUri: Uri = MediaStore.Video.Media.EXTERNAL_CONTENT_URI

@SuppressLint("ObsoleteSdkInt")
fun getMediaColumns(): Array<String> {
    val columns = mutableSetOf(
        BaseColumns._ID,
        MediaStore.MediaColumns.DISPLAY_NAME,
        MediaStore.MediaColumns.RELATIVE_PATH, MediaStore.MediaColumns.DATA,
        MediaStore.MediaColumns.DATE_ADDED,
        MediaStore.MediaColumns.DATE_MODIFIED,
        MediaStore.MediaColumns.MIME_TYPE,
        MediaStore.MediaColumns.SIZE,
        MediaStore.MediaColumns.TITLE,
    )

    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN) {
        columns.add(MediaStore.MediaColumns.HEIGHT)
        columns.add(MediaStore.MediaColumns.WIDTH)
    }

    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        columns.apply {
            add(MediaStore.MediaColumns.BUCKET_DISPLAY_NAME)
            add(MediaStore.MediaColumns.BUCKET_ID)
            add(MediaStore.MediaColumns.DATE_EXPIRES)
            add(MediaStore.MediaColumns.DATE_TAKEN)
            add(MediaStore.MediaColumns.DOCUMENT_ID)
            add(MediaStore.MediaColumns.DURATION)
            add(MediaStore.MediaColumns.INSTANCE_ID)
            add(MediaStore.MediaColumns.IS_PENDING)
            add(MediaStore.MediaColumns.ORIENTATION)
            add(MediaStore.MediaColumns.ORIGINAL_DOCUMENT_ID)
            add(MediaStore.MediaColumns.OWNER_PACKAGE_NAME)
            add(MediaStore.MediaColumns.RELATIVE_PATH)
        }
    }

    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
        columns.apply {
            add(MediaStore.MediaColumns.ALBUM)
            add(MediaStore.MediaColumns.ALBUM_ARTIST)
            add(MediaStore.MediaColumns.ARTIST)
            add(MediaStore.MediaColumns.AUTHOR)
            add(MediaStore.MediaColumns.BITRATE)
            add(MediaStore.MediaColumns.CAPTURE_FRAMERATE)
            add(MediaStore.MediaColumns.CD_TRACK_NUMBER)
            add(MediaStore.MediaColumns.COMPILATION)
            add(MediaStore.MediaColumns.COMPOSER)
            add(MediaStore.MediaColumns.DISC_NUMBER)
            add(MediaStore.MediaColumns.GENERATION_ADDED)
            add(MediaStore.MediaColumns.GENERATION_MODIFIED)
            add(MediaStore.MediaColumns.GENRE)
            add(MediaStore.MediaColumns.IS_DOWNLOAD)
            add(MediaStore.MediaColumns.IS_DRM)
            add(MediaStore.MediaColumns.IS_FAVORITE)
            add(MediaStore.MediaColumns.IS_TRASHED)
            add(MediaStore.MediaColumns.NUM_TRACKS)
            add(MediaStore.MediaColumns.XMP)
            add(MediaStore.MediaColumns.WRITER)
            add(MediaStore.MediaColumns.VOLUME_NAME)
            add(MediaStore.MediaColumns.RESOLUTION)
            add(MediaStore.MediaColumns.YEAR)

        }
    }

    return columns.toTypedArray()
}

fun getImageColumns(): Array<String> {
    val columns = getMediaColumns().toMutableSet()

    columns.add(MediaStore.Images.ImageColumns.DESCRIPTION)
    columns.add(MediaStore.Images.ImageColumns.IS_PRIVATE)
    columns.add(MediaStore.Images.ImageColumns.LATITUDE)
    columns.add(MediaStore.Images.ImageColumns.LONGITUDE)
    columns.add(MediaStore.Images.ImageColumns.MINI_THUMB_MAGIC)
    columns.add(MediaStore.Images.ImageColumns.PICASA_ID)

    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        columns.remove(MediaStore.Images.ImageColumns.LATITUDE)
        columns.remove(MediaStore.Images.ImageColumns.LONGITUDE)
        columns.remove(MediaStore.Images.ImageColumns.MINI_THUMB_MAGIC)
        columns.remove(MediaStore.Images.ImageColumns.PICASA_ID)
    }

    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
        columns.addAll(
            arrayListOf(
                // ImageColumns
                MediaStore.Images.ImageColumns.EXPOSURE_TIME,
                MediaStore.Images.ImageColumns.F_NUMBER,
                MediaStore.Images.ImageColumns.ISO,
                MediaStore.Images.ImageColumns.SCENE_CAPTURE_TYPE,
            )
        )
    }
    return columns.toTypedArray()
}

fun getVideoColumns(): Array<String> {
    val columns = getMediaColumns().toMutableSet()

    // VideoColumns
    columns.add(MediaStore.Video.VideoColumns.BOOKMARK)
    columns.add(MediaStore.Video.VideoColumns.CATEGORY)
    columns.add(MediaStore.Video.VideoColumns.DESCRIPTION)
    columns.add(MediaStore.Video.VideoColumns.IS_PRIVATE)
    columns.add(MediaStore.Video.VideoColumns.LANGUAGE)
    columns.add(MediaStore.Video.VideoColumns.LATITUDE)
    columns.add(MediaStore.Video.VideoColumns.LONGITUDE)
    columns.add(MediaStore.Video.VideoColumns.MINI_THUMB_MAGIC)
    columns.add(MediaStore.Video.VideoColumns.TAGS)

    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        columns.remove(MediaStore.Video.VideoColumns.LATITUDE)
        columns.remove(MediaStore.Video.VideoColumns.LONGITUDE)
        columns.remove(MediaStore.Video.VideoColumns.MINI_THUMB_MAGIC)
    }

    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
        columns.addAll(
            arrayListOf(
                // VideoColumns
                MediaStore.Video.VideoColumns.COLOR_RANGE,
                MediaStore.Video.VideoColumns.COLOR_STANDARD,
                MediaStore.Video.VideoColumns.COLOR_TRANSFER,
            )
        )
    }
    return columns.toTypedArray()
}

fun getAudioColumns(): Array<String> {
    val columns = getMediaColumns().toMutableSet()

    columns.add(MediaStore.Audio.AudioColumns.ALBUM_ID)
    columns.add(MediaStore.Audio.AudioColumns.ARTIST_ID)
    columns.add(MediaStore.Audio.AudioColumns.BOOKMARK)
    columns.add(MediaStore.Audio.AudioColumns.IS_ALARM)
    columns.add(MediaStore.Audio.AudioColumns.IS_MUSIC)
    columns.add(MediaStore.Audio.AudioColumns.IS_NOTIFICATION)
    columns.add(MediaStore.Audio.AudioColumns.IS_PODCAST)
    columns.add(MediaStore.Audio.AudioColumns.IS_RINGTONE)
    columns.add(MediaStore.Audio.AudioColumns.TRACK)
    columns.add(MediaStore.Audio.AudioColumns.YEAR)
    columns.add(MediaStore.Audio.AudioColumns.TITLE_KEY)


    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        columns.add(MediaStore.Audio.AudioColumns.IS_AUDIOBOOK)
    }

    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
        columns.addAll(
            arrayListOf(
                MediaStore.Audio.AudioColumns.GENRE,
                MediaStore.Audio.AudioColumns.GENRE_ID,
                MediaStore.Audio.AudioColumns.TITLE_RESOURCE_URI // The resource URI of a localized title, if any. https://developer.android.com/reference/android/provider/MediaStore.Audio.AudioColumns#TITLE_RESOURCE_URI
            )
        )
        columns.remove(MediaStore.Audio.AudioColumns.TITLE_KEY)
    }

    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        columns.add(MediaStore.Audio.AudioColumns.IS_RECORDING)
    }

    return columns.toTypedArray()
}
```

</details>

<br>

值得注意的是：
雖然 **MediaStore.MediaColumns.DATA** 在 API 29後被棄用，但目前 (33) 卻還是可以使用。

><br>
>
> 你可能會想： 竟然我們需要影片與音樂檔案，那為什麼我們不順便將 **MediaStore.Audio.AudioColumns** 與 **MediaStore.Video.VideoColumns** 加入 projection 呢？
><br>

其實答案很簡單。因為在查詢時，我們只能針對固定類型的 Columns。 也就是說， MediaStore.File 只能使用 FileColumns。

如果有無法找到的 projection 就會得到 **java.lang.IllegalArgumentException: Invalid column** 。

至於我們要如何取得所需要的資料，之後會提到。

#### 定義 Selection

接下來我們需要通過 SQL 語法定義我們要哪些資料。

假設我們只想要影片超過某長度的話，就可以寫：

```kotlin
val sel = MediaStore.Video.Media.DURATION + " >= ?"
```

在通過 Selection Args 來定義 `?` 是多少。這裡我們定義 5 分鐘：

```kotlin
val selectionArgs = arrayOf(
                TimeUnit.MILLISECONDS.convert(5, TimeUnit.MINUTES).toString()
            )
```

那如果我們要定義 File 的類別呢？

我們可以使用：

```kotlin
val sel = "${MediaStore.Files.FileColumns.MEDIA_TYPE} = ${MediaStore.Files.FileColumns.MEDIA_TYPE_AUDIO} || ${MediaStore.Files.FileColumns.MEDIA_TYPE} = ${MediaStore.Files.FileColumns.MEDIA_TYPE_VIDEO}"

val selectionArgs = null
```

或

```kotlin
val sel = "${MediaStore.Files.FileColumns.MEDIA_TYPE} = ? OR ${MediaStore.Files.FileColumns.MEDIA_TYPE} = ?"

val selectionArgs = arrayOf(
  ${MediaStore.Files.FileColumns.MEDIA_TYPE_AUDIO},
  ${MediaStore.Files.FileColumns.MEDIA_TYPE_VIDEO}
  )
```

當然，我們還可以通過其他運算子來設定 Selection，可以去 [w3school](https://www.w3schools.com/sql/default.asp) 來學習。


#### 定義 Sort Order

一般來說，我們會按加入的時間來排列，從最新到最晚：

```kotlin
val sortOrder = "${MediaStore.Files.FileColumns.DATE_TAKEN} DESC"

// or

val sortOrder = "${MediaStore.MediaColumns.DATE_TAKEN} DESC"
```

我們還可以按多個 Column 來進行排列：

```kotlin
val sortOrder = "${MediaStore.Files.FileColumns.DATE_TAKEN} DESC , ${MediaStore.Files.FileColumns.SIZE} ASC"
```

#### 進行 Query

Query (查詢) 時，你可以想像有一個表格提供給我們：

|index (optional)|Column_A|Column_B|...|
|--|--|--|--|
|0   | a  | b  | ...  |

這是因為在電腦中，檔案的資訊會以表格的方式存取起來。所以我們必須使用 **Cursor** 來讀取。

我們需要先記錄 Column 的 index ：

| **這些 index 是由我們的 `projection` 順序來訂的**

```kotlin
val query = contentResolver.query(
    collection,
    projection,
    selection,
    selectionArgs,
    sortOrder
)

query.use {cursor ->
    // Cache column indices.
    cursor?.let {
        val mediaTypeCol = it.getColumnIndex(MediaStore.Files.FileColumns.MEDIA_TYPE)
        val mimiTypeCol = it.getColumnIndex(MediaStore.Files.FileColumns.MIME_TYPE)
        val parentTypeCol = it.getColumnIndex(MediaStore.Files.FileColumns.PARENT)

        val displayNameCol = it.getColumnIndex(MediaStore.MediaColumns.DISPLAY_NAME)
        val durationCol = it.getColumnIndex(MediaStore.MediaColumns.DURATION)
        val pathCol = it.getColumnIndex(if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) MediaStore.MediaColumns.RELATIVE_PATH else MediaStore.MediaColumns.DATA)

    }
}
```

取得 Column Index 後就需要讀取對應的值：

```kotlin
while (it.moveToNext()) {
    val path = it.getString(pathCol)
    val type = it.getInt(mediaTypeCol)
    val name = it.getString(displayNameCol)
    Log.d("SMP", "PATH = $path , TYPE = $type, name = $name")
}
```

如果我們有錄影，那就會得到 **PATH = DCIM/Camera/ , TYPE = 3, name = {name} . mp4**。

以上就是 API 29 以上得到的答案。

但若是 29 以前， **PATH** 會由 **MediaStore.MediaColumns.DATA** 取得：
| **/storage/emulated/0/DCIM/Camera/VID_20230910_090407.mp4**

><br>
>
>切記： 若 column 並不存在，我們就會得到 **-1** 喔 !
><br>

以上就是我們目前取得資料的方法。接下來，我們看看使用 File 時要如何取得。

### File
想要取得 **/storage/emulated/0/DCIM/Camera/VID_20230910_090407.mp4** 我們必須要進入 */storage/emulated/0/DCIM/Camera* 中。

這個路徑可以由 **Environment.DIRECTORY_DCIM** 取得：

```kotlin
val file = Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_DCIM + "/Camera")
```

由此我們可以取得這個檔案夾的全部檔案：

```kotlin
file.listFiles()
```

但這當然不是我們所要的，我們需要使用 **FileFilter** 來取得特定類別：

```kotlin
val filter = object: FileFilter {

    val acceptedFileExtensions = ".(mp4|mkv|webm|avi)\$"
    val pattern = Pattern.compile(acceptedFileExtensions, Pattern.CASE_INSENSITIVE)

    override fun accept(pathname: File?): Boolean {
        if (pathname == null) return false
        return pathname.isFile && pattern.matcher(pathname.name).find()
    }
}
```

以上 filter 會讓我們取得 **map4, mkv, webm, avi** 的影片檔案。 如果也想要拿圖檔，那就更改為以下即可：

```kotlin
".(jpg|png|gif|jpe|jpeg|bmp|webp|mp4|mkv|webm|avi)$"
```

那也就是說，**File** 有以下的好處：
- 可以讓我們搜索 **特定的資料夾**
- 可以讓我們搜索 **特定的檔案類型**

**Environment** 除了能讓我們找到照相機的資料，還能夠提供多個共享及系統資料夾的位置 ( **/system, /data 與 /storage** ) ：

```kotlin
// 取得 /system
Environment.getRootDirectory()

// 取得 /data
Environment.getDataDirectory()

// 取得 /data/cache
// 這檔案中包含了 backup, backup_stage, recovery 三個檔案
Environment.getDownloadCacheDirectory()

// 取得 /storage
Environment.getStorageDirectory()

// 取得 /storage/emulated/0
Environment.getExternalStorageDirectory()

// 取得 /storage/emulated/0/Alarms
// 我們還可以通過傳入不同的 DIRECTORY_{TYPE} 來取得
// /storage/emulated/0/{TYPE} 的路徑
// 包括： DIRECTORY_MUSIC , DIRECTORY_PODCASTS, DIRECTORY_RINGTONES
//      DIRECTORY_ALARMS, DIRECTORY_NOTIFICATIONS, DIRECTORY_PICTURES
//      DIRECTORY_MOVIES, DIRECTORY_DOWNLOADS, DIRECTORY_DCIM
//      DIRECTORY_DOCUMENTS, DIRECTORY_SCREENSHOTS, DIRECTORY_AUDIOBOOKS
//      DIRECTORY_RECORDINGS
Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_ALARMS)

```


### 取得檔案更多訊息

由於我們只取得針對 File 相關的資訊，所以就算我們知道檔案類型，我們依舊無法取得對方的相關資訊。

對此，我們有兩種做法：
1. 使用 **ContentProvider** 相關的 Uri 進行查詢
2. 使用 **完整路徑** 讀取檔案

#### ContentProvider

所謂 **ContentProvider** 相關的 Uri 其實就是從 **MediaStore.Files.getContentUri** 取得：

```kotlin
public static Uri getContentUri(String volumeName)
public static Uri getContentUri(String volumeName, long rowId)
```

我們可以傳入不同的 `volumeName` 包括： `external`, `external_primary` 與 `internal`。 **API 30** 之後也可以從 **MediaStore** 參數取得：

```kotlin
// content://media/external/file/
public static final String VOLUME_EXTERNAL = "external";

// content://media/external_primary/file
public static final String VOLUME_EXTERNAL_PRIMARY = "external_primary";

// content://media/internal/file
public static final String VOLUME_INTERNAL = "internal";
```

如果我們傳入了 `rowId`，所得到的 Uri 會是：

```kotlin
content://media/{location}/file/{rowId}

// ie content://media/external/file/0
```

我們也可以通過 Uri 取得 rowId。但若是 Uri 中沒有數字，Long.Parser 就會失敗並拋出錯誤訊息。

```kotlin
val row: Long = ContentUris.parseId(uri);
```

當我們想要取得某檔案的更多訊息時，我們可以通過資料中的 **MediaStore.MediaColumns._ID** 來建立新的 Uri，並進行查詢 ：

```kotlin
query.use {cursor ->
    // Cache column indices.
    cursor?.let {
      val idCol = it.getColumnIndex(MediaStore.MediaColumns._ID)
      while (it.moveToNext()) {
        val uri = ContentUris.withAppendedId(MediaStore.Files.getContentUri("external"), id)

        contentResolver.query(uri, arrayOf(MediaStore.Video.Thumbnails.DATA), null, null, null).use {
              it?.let {
                  val thumbDataCol =
                      it.getColumnIndex(MediaStore.Video.VideoColumns.Thumbnails.DATA)
                  while (it.moveToNext()) {
                      val c = it.getInt(thumbDataCol)
                      Log.d("SMP", "C = $c")
                  }
              }
          }
      }
    }
}
```

如此一來就可以在  API 29 之前取得 Thumbnails 了。

在 API 29 及以上需要用到 **contentResolver.loadThumbnail** 來進行：

```kotlin
contentResolver.loadThumbnail(uri, Size(100, 100), null)
```

由於想要顯示就需要知道大小。 所以他們會在 View 相關的類型中調用。像是： RecyclerView.ViewHolder。


#### 完整路徑
想要取得完整路徑，我們有兩種方式取得：
1. 查詢 **MediaStore.MediaColumns.DATA**
2. 通過 **Environment**

如之前所說， 官網宣稱 **DATA** 在 API 29 就廢棄了，但測試時 API 33 還是可用的。 也有人說 API 32 又可以用了。

**|** 無論如何，我們就先 **假設** 官方沒錯吧 :upside_down_face:	。

API 29 之後就只能通過查詢 RELATIVE_PATH 與 NAME 來取得相關資訊。

經由測試，我們知道 **/storage/emulated/0/DCIM/Camera/VID_20230910_090407.mp4** 其實可以看成：

**/storage/emulated/0 {RELATIVE_PATH} {NAME}**

問題就在如何得到 **/storage/emulated/0/** 這部分了。

我們可以通過

```kotlin
Environment.getStorageDirectory() // 取得 /storage/emulated/0

Environment.getStorageDirectory() // 取得 /storage
```

畢竟我們能取得的資料夾位置也就三種： **/system**、**/data** 與 **/storage** 。 應該不難知道應該調用哪個方法。

## Query 的實作
為了要讓我們更容易地進行查詢，我們參考 [leafpicrevived](https://github.com/apcro/leafpicrevived) 中的 **Query.java** 來定義一個 **Builder**。

但首先我們需要暸解 SQL 查詢時的 [語法](https://www.fooish.com/sql/)。

### SQL 語法

根據 Android 的 Query 寫法，SQL 可以得到以下格式：

```sql
SELECT {projection}
FROM {uri}
WHERE {selection + selectionArgs}
ORDER BY {sortOrder}
```

雖然如此，但 SQL 還有很多語法可以使用，包括：
**DISTINCT, LIMIT, GROUP BY, HAVING, BETWEEN** 等等。

我們再次參考 leafpicrevived 中的 [CPHelper.java](https://github.com/apcro/leafpicrevived/blob/master/leafpicrevived/src/main/java/com/alienpants/leafpicrevived/data/provider/CPHelper.java) ：

```java
private static String getHavingClause(int excludedCount) {

    if (excludedCount == 0)
        return "(";

    StringBuilder res = new StringBuilder();
    res.append("HAVING (");

    res.append(MediaStore.Images.Media.DATA).append(" NOT LIKE ?");

    for (int i = 1; i < excludedCount; i++)
        res.append(" AND ")
                .append(MediaStore.Images.Media.DATA)
                .append(" NOT LIKE ?");

    // NOTE: dont close ths parenthesis it will be closed by ContentResolver
//        res.append(")");

    return res.toString();

}
```

而這個方法的用法為：

```java
query.selection(String.format("%s=? or %s=?) GROUP BY (%s) %s ",
    MediaStore.Files.FileColumns.MEDIA_TYPE,
    MediaStore.Files.FileColumns.MEDIA_TYPE,
    MediaStore.Files.FileColumns.PARENT,
    getHavingClause(excludedAlbums.size())));
```

所以其實我們可以直接連接在 `selection` 的字串中。

**| 但其實這方法已經無法使用了**

我會在另一篇中建立兩個 App 來做範例。

# Error

## Gradle
### Java version
```
Execution failed for task ':app:kaptGenerateStubsDebugKotlin'.
> 'compileDebugJavaWithJavac' task (current target is 1.8) and 'kaptGenerateStubsDebugKotlin' task (current target is 17) jvm target compatibility should be set to the same Java version.
```

我們只需要將 kotlin options 與 Compile Option 中的 Java Target 從 `1.8` 改為 `17`。

```groovy
compileOptions {
    sourceCompatibility JavaVersion.VERSION_17
    targetCompatibility JavaVersion.VERSION_17
}
kotlinOptions {
    jvmTarget = '17'
}
```

# Reference


# Next
下一篇： 

<br><br><br><br><br><br><br><br><br>
