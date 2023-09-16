---
layout: post
title:  "Android - AppBarLayout"
date:   2022-11-14 15:27:40 +08:00
use_math: true
categories: [android, layout]
---

這篇我們將花時間來理解 **AppBarLayout** 以及他的運用。

首先，我們先來看看 **AppBarLayout** 的架構吧。

# 架構

我們可以從 [material](https://m2.material.io/components/app-bars-top/#anatomy) 中看到他的架構：

<center>
<img src = "/images/posts/jekyll/android/ui/appbarlayout/anatomy.png"/>
</center>

<br>

這只是給我們一個大略 UI 的樣子。 至於實作，我們會在另一節細談。接下來，我們就研究源碼吧。

# 源碼

原來 **AppBarLayout** 是一個實作 **CoordinatorLayout.AttachedBehavior** 的 **LinearLayout** ：
```java
public class AppBarLayout extends LinearLayout implements CoordinatorLayout.AttachedBehavior
```

**AttachedBehavior** 會通過 `getBehavior` 回傳 **AppBarLayout.Behavior** ：

```java
public static class Behavior extends BaseBehavior<AppBarLayout>
```

這個 **Behavior** 可以在 [這篇]() 中找到。




# 範例

## 官方範例

```xml
<androidx.coordinatorlayout.widget.CoordinatorLayout
      xmlns:android="http://schemas.android.com/apk/res/android"
      xmlns:app="http://schemas.android.com/apk/res-auto"
      android:layout_width="match_parent"
      android:layout_height="match_parent">

  <androidx.core.widget.NestedScrollView
          android:layout_width="match_parent"
          android:layout_height="match_parent"
          app:layout_behavior="@string/appbar_scrolling_view_behavior">

      <!-- Your scrolling content -->

  </androidx.core.widget.NestedScrollView>

  <com.google.android.material.appbar.AppBarLayout
          android:layout_height="wrap_content"
          android:layout_width="match_parent">

      <androidx.appcompat.widget.Toolbar
              ...
              app:layout_scrollFlags="scroll|enterAlways"/>

      <com.google.android.material.tabs.TabLayout
              ...
              app:layout_scrollFlags="scroll|enterAlways"/>

  </com.google.android.material.appbar.AppBarLayout>

</androidx.coordinatorlayout.widget.CoordinatorLayout>
```

<br><br><br><br><br><br><br><br>
