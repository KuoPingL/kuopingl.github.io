---
layout: post
title: Simple Player 2 - UI 解說
categories: [Projects]
keywords: android, project, mediaPlayer
---

# 專案簡介
不知道你是否跟我一樣，平時會從不同地方 (TikTok, Line ... ) 下載喜歡的影片然後希望之後可以慢慢回味。 但當我們使用相簿 App 開啟時才發現，我們無法無限地播放影片，更無法將影片歸入播放清單。
Â
為了解決這問題，我決定建立一個簡單的 Media Player App ( 取名 **SMP Simple Media mediaPlayer** )來提供使用者一個可以好好享受音樂的平台。

在上一單元中我們順利地取得本地的多媒體，這單元我們將要看看 UI 的設計。

## UI
UI 將採取 TikTok Vibe：

<center>
<img src = "/images/posts/jekyll/projects/smp/tiktok-logo-color-palette.jpg" style="width:50%"/>
</center>

而影片與音樂的播放也會如此。

整個 UI 大致長這樣

<center>
<img src = "/images/posts/jekyll/projects/smp/ui_sketch.png" style="width:70%"/>
</center>

我將在這裡使用 Android XML 的寫法，之後會再將這些轉換成 Compose。 如此一來就能更加熟悉了。

# Splash Screen
首先，我們要建立一個 SplashScreen。

這時就可以看看官方的 [文件](https://developer.android.com/develop/ui/views/launch/splash-screen)。

我們需要做幾件事：
1. 增加 dependencies
    ```groovy
    implementation 'androidx.core:core-splashscreen:1.0.0-beta02'
    ```
2. 設定 style
   ```xml
   <style name="Theme.App.Starting" parent="Theme.SplashScreen">

        <!-- Set the splash screen background, animated icon, and animation duration. -->
        <item name="windowSplashScreenBackground">@color/black</item>

        <!-- Use windowSplashScreenAnimatedIcon to add either a drawable or an
             animated drawable. One of these is required. -->
        <item name="windowSplashScreenAnimatedIcon">@drawable/splash_screen</item>
        <!-- Required for animated icons -->
        <item name="windowSplashScreenAnimationDuration">200</item>

        <!-- Set the theme of the Activity that directly follows your splash screen. -->
        <!-- Required -->
        <item name="postSplashScreenTheme">@style/Theme.SMP</item>
    </style>
   ```

然後我們有 [兩種實作方法](https://developer.android.com/develop/ui/views/launch/splash-screen/migrate)：
1. 在 Manifest 中將 application 的 android:theme 指向 Theme.App.Starting
2. 建立一個 Activity，並將其 android:theme 指向 Theme.App.Starting


我兩種都試過，但第一種會產生錯誤訊息：

```
unable to start activity you need to use a theme.appcompat theme after splash screen
```

並導致整個 app 無法開啟。

所以我選擇第二種。不僅簡單，且適合 Android 12 以下的機子。
我只需要在 Activity 的 `onCreate` 調用 `installSplashScreen` 就好：

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    val splashScreen = installSplashScreen()
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_splash)
}
```

`installSplashScreen` 其實就是建立一個 **SplashScreen** 物件並讓他進行動畫：

```kotlin

@JvmStatic
public fun Activity.installSplashScreen(): SplashScreen {
    val splashScreen = SplashScreen(this)
    splashScreen.install()
    return splashScreen
}

open fun install() {
    val typedValue = TypedValue()
    val currentTheme = activity.theme
    if (currentTheme.resolveAttribute(
            R.attr.windowSplashScreenBackground,
            typedValue,
            true
        )
    ) {
        backgroundResId = typedValue.resourceId
        backgroundColor = typedValue.data
    }
    if (currentTheme.resolveAttribute(
            R.attr.windowSplashScreenAnimatedIcon,
            typedValue,
            true
        )
    ) {
        icon = currentTheme.getDrawable(typedValue.resourceId)
    }

    if (currentTheme.resolveAttribute(R.attr.splashScreenIconSize, typedValue, true)) {
        hasBackground =
            typedValue.resourceId == R.dimen.splashscreen_icon_size_with_background
    }
    setPostSplashScreenTheme(currentTheme, typedValue)
}

protected fun setPostSplashScreenTheme(
    currentTheme: Resources.Theme,
    typedValue: TypedValue
) {
    if (currentTheme.resolveAttribute(R.attr.postSplashScreenTheme, typedValue, true)) {
        finalThemeId = typedValue.resourceId
        if (finalThemeId != 0) {
            activity.setTheme(finalThemeId)
        }
    }
}

```

><br>
>
> 對了，我原本是希望有文字的，但 Android 是無法使用 \<text\> 的，所以我們必須將所有文字用點與線來表達。 這就難倒我了。 所以就只畫出 Play Button 了。
><br><br>

## 讀取資料
除此之外，如果我們想在 SplashScreen 跑的時候進行資料的讀取，其實也是可以的。這是官方給的 [範例](https://developer.android.com/develop/ui/views/launch/splash-screen#suspend-drawing)：

```kotlin
// Create a new event for the activity.
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    // Set the layout for the content view.
    setContentView(R.layout.main_activity)

    // Set up an OnPreDrawListener to the root view.
    val content: View = findViewById(android.R.id.content)
    content.viewTreeObserver.addOnPreDrawListener(
        object : ViewTreeObserver.OnPreDrawListener {
            override fun onPreDraw(): Boolean {
                // Check whether the initial data is ready.
                return if (viewModel.isReady) {
                    // The content is ready. Start drawing.
                    content.viewTreeObserver.removeOnPreDrawListener(this)
                    true
                } else {
                    // The content isn't ready. Suspend.
                    false
                }
            }
        }
    )
}
```

## 動畫
我們有兩種方法讓 SplashScreen 擁有動畫：
1. 在 Drawable 中建立一個以 **animation-list** 為根的 xml
2. 在 onCreate 時對 iconView 進行 [動畫的設定](https://developer.android.com/develop/ui/views/launch/splash-screen#customize-animation)
  ```kotlin
  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    // ...

    // Add a callback that's called when the splash screen is animating to the
    // app content.
    splashScreen.setOnExitAnimationListener { splashScreenView ->
        // Create your custom animation.
        val slideUp = ObjectAnimator.ofFloat(
            splashScreenView,
            View.TRANSLATION_Y,
            0f,
            -splashScreenView.height.toFloat()
        )
        slideUp.interpolator = AnticipateInterpolator()
        slideUp.duration = 200L

        // Call SplashScreenView.remove at the end of your custom animation.
        slideUp.doOnEnd { splashScreenView.remove() }

        // Run your animation.
        slideUp.start()
    }
}
  ```

## Permission

由於 SplashScreen 並不會一直在畫面上，所以不建議在這裡進行 pemission 的請求。

這應當由 MainActivity 或下一個 Activity 來負責。


# 首頁

這裡會有四個 UI 元件：
1. 顯示主要 Playlist 的 ViewPager2 或 RecyclerView
2. 顯示音檔/影片 RecyclerView
3. 顯示音檔與影片的 ViewPager2
4. 下面的 Tabbar





<br><br><br><br><br><br><br><br><br>
