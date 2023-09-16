---
layout: post
title:  "Note: LeafPic Clone (Android)"
date:   2022-08-08 16:15:07 +0800
categories: [note taking]
---
;) 這是我個人的筆記，有興趣的可以暸解暸解

### 背景
我想要從 [LeafPic Revive](https://github.com/apcro/leafpicrevived) 中學習更多 Android 的編寫方式。
並以 LeafPic 為例來編寫另一隻更符合我需求的 App。


### New Project (Basic Activity)
以下是從 New Project (Basic Activity) 中所了解的東西。

#### MainActivity

`activity_main.xml` 中可分成四個部分：
- **CoordinatorLayout**
- **AppBarLayout**
- **ConstraintLayout** (include layout)
- **FAB**

之所以會用 **CoordinatorLayout** 是因為 **FAB** 和 **AppBarLayout** 都有 CoordinatorLayout.**AttachedBehavior**。

通過 **AttachedBehavior**，ViewGroup 元件

```Kotlin
final Callback callback = currentSnackbar.callback.get();
if (callback != null) {
  callback.show();
} else {
  // The callback doesn't exist any more, clear out the Snackbar
  currentSnackbar = null;
}
```



```XML
<?xml version="1.0" encoding="utf-8"?>
<androidx.coordinatorlayout.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <com.google.android.material.appbar.AppBarLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:theme="@style/Theme.LeafPicClone.AppBarOverlay">

        <androidx.appcompat.widget.Toolbar
            android:id="@+id/toolbar"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize"
            android:background="?attr/colorPrimary"
            app:popupTheme="@style/Theme.LeafPicClone.PopupOverlay" />

    </com.google.android.material.appbar.AppBarLayout>

    <include layout="@layout/content_main" />

    <com.google.android.material.floatingactionbutton.FloatingActionButton
        android:id="@+id/fab"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="bottom|end"
        android:layout_marginEnd="@dimen/fab_margin"
        android:layout_marginBottom="16dp"
        app:srcCompat="@android:drawable/ic_dialog_email" />

</androidx.coordinatorlayout.widget.CoordinatorLayout>
```
