---
layout: post
title: Simple Player 1.5 - Query 修正
categories: [Projects]
keywords: android, project, mediaPlayer
---

#更正公告
我在 [Simple Player 1](https://kuopingl.github.io//2023/09/16/project-simple-player-1-query/) 中敘述了不少與 Query 相關的資料並指出如何取得音樂與影片。

但在編寫 Simple Player 2 時，我才發現原來有不少錯誤觀念以及錯誤的 permission 請求。

我在這篇裡將會一一敘述。

# Permission

## Manifest
錯誤：
```groovy
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
<uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />
```

更正：
```groovy
<uses-permission android:name="android.permission.READ_MEDIA_AUDIO" />
<uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />
```

解釋：
這裡應該要取得 **READ_MEDIA_AUDIO** 才是，而不是 **READ_MEDIA_IMAGES**。

當你使用錯誤的 permission 時，儘管你想要取得音檔，系統也不會跟你說。 所以千萬要注意這個細節。

## PermissionUtils

### 更正方法

<u>更正 `requestStoragePermission`</u>

```kotlin
fun requestStoragePermission(activity: Activity): Boolean {

  val permissions = arrayListOf(Manifest.permission.READ_EXTERNAL_STORAGE, Manifest.permission.WRITE_EXTERNAL_STORAGE)

  if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
      permissions.clear()
      if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
          // >>>>>>>>>> 新增此行
          permissions.add(Manifest.permission.READ_MEDIA_VIDEO)
          // <<<<<<<<<<<
          permissions.add(Manifest.permission.READ_MEDIA_AUDIO)
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
```

<br>

解釋：
我們在 **Manifest** 中新增了 **READ_MEDIA_AUDIO** 的 permission，所以我們也可以在這裡新增需要 request 的 permission 了。

<u>更正 `handleStoragePermissionResult`</u>

```kotlin
fun handleStoragePermissionResult(permissions: Array<out String>,
                                grantResults: IntArray,
                                context: AppCompatActivity,
                                // >>>>>>> 新增 lambda action
                                action: () -> Unit
                                // <<<<<<<<
                                ) {
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
      // >>>>>>>> 調用 action
      action.invoke()
      // <<<<<<<<
  }
}
```
<br>

解釋：
當我們進行請求時，我們應當在請求達成的時候進行下一個步驟。
想要有這種行為，我們有兩種選擇：
1. Callback
2. return value

由於我們這個方法會需要顯示 Dialog，所以無法進行回傳。因此我選擇使用 Callback。

使用的時候就會變成如此：

```kotlin
override fun onRequestPermissionsResult(
    requestCode: Int,
    permissions: Array<out String>,
    grantResults: IntArray
) {

    when (requestCode) {
        PermissionUtils.STORAGE_REQUEST_CODE -> PermissionUtils.handleStoragePermissionResult(
            permissions,
            grantResults,
            this
        ) {
            // >>>>>> 進行 fetch
        }

        else -> super.onRequestPermissionsResult(requestCode, permissions, grantResults)
    }

}
```

<br>

如同 `requestStoragePermission` 一樣， `isStoragePermissionGranted` 也需要進行更正：

```kotlin
fun isStoragePermissionGranted(context: Context): Boolean {
    return if (Build.VERSION.SDK_INT <= Build.VERSION_CODES.P) {
        checkPermissions(context, Manifest.permission.READ_EXTERNAL_STORAGE, Manifest.permission.WRITE_EXTERNAL_STORAGE)
    }
    else if (Build.VERSION.SDK_INT < Build.VERSION_CODES.TIRAMISU) {
        checkPermissions(context, Manifest.permission.READ_EXTERNAL_STORAGE)
    } else {
        checkPermissions(context, Manifest.permission.READ_MEDIA_VIDEO,
          // >>>>>>>> 新增此行
           Manifest.permission.READ_MEDIA_AUDIO
           // <<<<<<<<
           )
    }
}
```

## Request 的流程
在上一篇中，我的請求流程是在 `onStart` 中調用，

錯誤示範：
```kotlin
override fun onStart() {
    super.onStart()
    val allowed = PermissionUtils.requestStoragePermission(this)

    if (allowed) {
        _viewModel.fetchMedias(
            MediaStore.Files.getContentUri("external"),
            Media.getProjection(),
            Media.sortByDate(),
            arrayOf(Query.SortOrder.DESC),
            contentResolver)
    }
}
```

但其實這樣寫是錯的，因為以下幾點：
1. 在 `onStart` 進行 fetchMedias 會導致每次使用者回到這個頁面時都得重新讀取與顯示資料。 這會很廢勁。
2. 在 `onStart` 我們理應通過 **PermissionUtils** 的 `isStoragePermissionGranted` 方法來檢測是否擁有 permission。若無，才進行請求才是。

因此，**MainActivity** 的請求流程會從：

<center>
<img src = "/images/posts/jekyll/projects/smp/1_5_previous_permission_flow.png" style="width:50%"/>
</center>

變成

<center>
<img src = "/images/posts/jekyll/projects/smp/1_5_new_permission_flow.png" style="width:50%"/>
</center>

為此，我在 activity_main.xml 中新增一個錯誤訊息畫面

```groovy
<androidx.constraintlayout.widget.ConstraintLayout
      android:id="@+id/container_error"
      android:visibility="gone"
      android:layout_width="match_parent"
      android:layout_height="match_parent"
      android:background="@color/white">

      <androidx.appcompat.widget.AppCompatTextView
          android:id="@+id/tv_error_msg"
          android:layout_width="match_parent"
          android:layout_height="wrap_content"
          app:layout_constraintTop_toTopOf="parent"
          app:layout_constraintBottom_toBottomOf="parent"
          app:layout_constraintVertical_bias="0.6"
          android:textAlignment="center"
          android:paddingStart="20dp"
          android:paddingEnd="20dp"
          android:paddingTop="8dp"
          android:paddingBottom="8dp"
          android:text="Oops, Looks like someone \ndidn't grant permission for storage"
          android:textStyle="bold"
          android:textSize="20sp"/>

      <androidx.appcompat.widget.AppCompatButton
          android:id="@+id/btn_request"
          android:layout_width="wrap_content"
          android:layout_height="wrap_content"
          app:layout_constraintTop_toBottomOf="@id/tv_error_msg"
          app:layout_constraintStart_toStartOf="@id/tv_error_msg"
          app:layout_constraintEnd_toEndOf="@id/tv_error_msg"
          android:backgroundTint="@color/black"
          android:textColor="@color/white"
          android:textStyle="bold"
          android:text="Request Permission"/>

  </androidx.constraintlayout.widget.ConstraintLayout>

```

如此一來， **MainActivity** 中針對 permission 的請求流程會是如下：

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)

    // ... basic setup

    // Requesting Permission Button
    _binding.btnRequest.setOnClickListener {
        PermissionUtils.requestStoragePermission(this)
    }

    // check if permission is granted
    val granted = PermissionUtils.isStoragePermissionGranted(this)


    if (granted) {
        // fetch media
    } else {
        PermissionUtils.requestStoragePermission(this)
    }
}

override fun onStart() {
    super.onStart()

    // if containerError is visible, then we need to do a request again.
    // this will not conflict with the onCreate request,
    // because the system will take care of it.
    //
    // Also, this is a must, because user might turn off the permission when returning to the page.
    when(_binding.containerError.visibility) {
        View.VISIBLE -> {
            PermissionUtils.requestStoragePermission(this)
        }
        else -> {
          if (!PermissionUtils.isStoragePermissionGranted(this)) {
            _binding.containerError.visibility = View.VISIBLE
          }

        }
    }
}


override fun onRequestPermissionsResult(
    requestCode: Int,
    permissions: Array<out String>,
    grantResults: IntArray
) {

    when (requestCode) {
        PermissionUtils.STORAGE_REQUEST_CODE -> PermissionUtils.handleStoragePermissionResult(
            permissions,
            grantResults,
            this
        ) {
            // fetch media

            _binding.containerError.visibility = View.GONE
        }

        else -> super.onRequestPermissionsResult(requestCode, permissions, grantResults)
    }

}
```

# Querying
關於查詢也是有不少的問題，我在這也必須要一一糾正。

## 自作聰明的 inline 方法

還記得我為了檢查 Uri、 Selection 與 Projection 是否為 null 而創建的 inline 方法嗎？

不記得沒關係，因為問題不是出在這方法，問題是出在 Query 上。因為在 Query 的時候，只要 Uri 不為 null 一切都可以。

所以我們需要將 Query 方法從：

```kotlin
fun getCursor(cr: ContentResolver): Cursor? {
    return withNoNulls(uri, projection, selection) {uri, projection, selection ->
        cr.query(uri, projection, selection, selectionArgs, getSortString())
    }
}
```

改為

```kotlin
fun getCursor(cr: ContentResolver): Cursor? {
    return uri?.let {
        cr.query(it, projection, selection, selectionArgs, getSortString())
    }
}
```

即可。


## Uri
我在上一節說，當我們要找資料的時候我們可以使用 `MediaStore.File.getContentUri("external")`

並配合 `${MediaStore.Files.FileColumns.MEDIA_TYPE} = ?` 與 `arrayOf("${MediaStore.Files.FileColumns.MEDIA_TYPE_AUDIO}")` 來作為 selection 與其 args。

通過這些方法以及 `getCursor` 的修改，我們可以取得所有音樂以及其他相關的檔案，包括：

- App 專屬的通知音檔
- 以及我們最需要的 Download 與 Music 中的音檔

> 那如果我們只想要讀取 Download 與 Music 中的音檔呢？

我們可以通過 selection 與 selectArgs 來過濾：

```kotlin
Query.Builder()
        .uri(MediaStore.Files.getContentUri("external"))
        .projection(arrayOf(MediaStore.Files.FileColumns.DATA))
        .sortByColumns(null)
        .sortArgs(null)
        .selection("${MediaStore.Files.FileColumns.MEDIA_TYPE} = ? AND (${MediaStore.Audio.AudioColumns.DATA} like ? OR ${MediaStore.Audio.AudioColumns.DATA} like ?)")
        .selectionArgs(arrayOf("${MediaStore.Files.FileColumns.MEDIA_TYPE_AUDIO}", "%" + "music" + "%", "%" + "download" + "%"))
        .build()
```

以上的方法雖然可以得到音檔，但這些音檔也有可能是語音檔而已。
那我們要如何判斷音檔是否為音樂呢？

( PS: 我的這 App 只會將全部都撈出來，所以不在乎是否為音樂 )

我們可以使用 `MediaStore.Audio.Media.IS_MUSIC` 來進行判斷。

```kotlin
val selection = "${MediaStore.Audio.Media.IS_MUSIC} != 0"
```

如果你使用 **File** 的 Uri，你會遇到錯誤訊息：

```
invalid token is_music
```

這是因為 **MediaStore.Audio.Media.IS_MUSIC** 並不適用於 **File** 而是專屬於 **MediaStore.Audio.Media** 的。 像是：**MediaStore.Audio.Media.EXTERNAL_CONTENT_URI** 。


如果你打算用 **IS_MUSIC** 的方法，那要記住，API 30 之後是無法取得 Download 的資料。 要使用 30 推出的 **Mediastore.Downloads** uri 才行。

為此，我們還需要使用 [**MergeCursor**](https://developer.android.com/reference/android/database/MergeCursor) 將多個 uri 一同被查詢。








<br><br><br><br><br><br><br><br><br>
