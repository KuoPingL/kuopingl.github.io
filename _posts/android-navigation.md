---
layout: post
title: Android - Navigation
categories: [Android]
use_math: true
keywords: android, navigation, jetpack, compose
---

# 概要
這篇文章中，我們會看到 Android 是如何實現 Navigation，包括 Jetpack 與 Compose。

而其中的議題包括：
- 簡單的頁面轉換
- 頁面之間的資料傳遞
- Fragment 的轉換
- DeepLink
- Bottom Navigation Bar 的運用
- 測試 Navigation

# Jetpack



# Compose

想要使用 Compose 就需要先將 [Compose 的 dependencies](https://developer.android.com/jetpack/compose/setup) 準備好。

而在 Compose 中，我們可以通過不同的方法來執行 navigation，包括：

- 使用 **Context**
- 使用 **NavHost** 與 **NavController**


## 使用 Context
如果我們想要進入 **Activity**，那我們在 Composable 中可以通過 `LocalContext.current` 取得當下的 **Context**。

我們可以這樣使用：

```kotlin
val context = LocalContext.current

Button(onClick = {
    context.startActivity(Intent(context, SecondActivity::class.java))
}, modifier = btnModifier) {
    Text(text = "UdemyDemo", modifier.padding(horizontal = Dp(10f), vertical = Dp(8f)))
}
```

## 使用 NavHost 與 NavController

我們還需要加上 [以下的 dependencies](https://developer.android.com/jetpack/compose/navigation) 來使用 Compose 的 Navigation ：

```groovy
dependencies {
    def nav_version = "2.6.0"

    implementation "androidx.navigation:navigation-compose:$nav_version"
}
```

接下來我們需要創建一個 **NavigationController** 來記錄瀏覽頁面的狀態以及順序 。

```kotlin
val navController = rememberNavController()
```

其中的 `rememberNavController` 是 **Composable**

```kotlin
@Composable
public fun rememberNavController(
    vararg navigators: Navigator<out NavDestination>
): NavHostController {
    val context = LocalContext.current
    return rememberSaveable(inputs = navigators, saver = NavControllerSaver(context)) {
        createNavController(context)
    }.apply {
        for (navigator in navigators) {
            navigatorProvider.addNavigator(navigator)
        }
    }
}

private fun createNavController(context: Context) =
    NavHostController(context).apply {
        navigatorProvider.addNavigator(ComposeNavigator())
        navigatorProvider.addNavigator(DialogNavigator())
    }
```



因為每個 **NavigationController** 都需要一個對應的 **NavHost**。

Compose **NavHost** 有兩種建立方法：

```kotlin
@Composable
NavHost(
    navController: NavHostController,
    startDestination: String,
    modifier: Modifier,
    contentAlignment: Alignment,
    route: String?,
    enterTransition: AnimatedContentTransitionScope<NavBackStackEntry>.() -> EnterTransition,
    exitTransition: AnimatedContentTransitionScope<NavBackStackEntry>.() -> ExitTransition,
    popEnterTransition: AnimatedContentTransitionScope<NavBackStackEntry>.() -> EnterTransition,
    popExitTransition: AnimatedContentTransitionScope<NavBackStackEntry>.() -> ExitTransition,
    builder: NavGraphBuilder.() -> Unit
)

```

與

```kotlin
// NavGraph 相對於 route、 startDestination 與 NavGraphBuilder
@Composable
NavHost(
    navController: NavHostController,
    graph: NavGraph,
    modifier: Modifier,
    contentAlignment: Alignment,
    enterTransition: AnimatedContentTransitionScope<NavBackStackEntry>.() -> EnterTransition,
    exitTransition: AnimatedContentTransitionScope<NavBackStackEntry>.() -> ExitTransition,
    popEnterTransition: AnimatedContentTransitionScope<NavBackStackEntry>.() -> EnterTransition,
    popExitTransition: AnimatedContentTransitionScope<NavBackStackEntry>.() -> ExitTransition
)
```











<br><br><br><br><br><br><br><br><br><br><br><br>
