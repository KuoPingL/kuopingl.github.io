---
layout: post
title:  "Android Study - TextView"
date:   2022-10-23 17:38:00+08:00
categories: [android, textview, study]
use_math: true
---

# 背景
我希望可以讓 TextView 顯示垂直的格式，所以才開始研究它。

> <br>
>
> **提示**： 最後發現只能覆寫 `onDraw`
> <br>

因為想要顯示垂直，我當時想到幾個地方值得去探索：
1. **Android DrawText 的機制**
2. **Skia DrawText 的機制**
3. **TextView 安排字串顯示位置的機制**
4. **TextView 與 Keyboard 的互動**

雖然最後還是只能建立 View 並定義 `onDraw` 的行為，但通過這次的研究，我對 TextView 又有更深的理解了。

所以希望通過這文章記錄自己所學，並分享給大家。


# TextView 的定義

由於 Android 除了 XML 我們還可以通過 Compose 來定義 TextView。

## XML

```groovy
<androidx.appcompat.widget.AppCompatTextView
      android:layout_width="match_parent"
      android:layout_height="wrap_content"
      android:text="Testing"/>
```

## Compose

```kotlin
@Composable
fun Greeting(name: String, modifier: Modifier = Modifier) {
    Text(
        text = "Hello $name!",
        modifier = modifier
    )
}

```

若想要使用 Compose，那就得設定 [Gradle](https://developer.android.com/jetpack/compose/setup?hl=zh-tw) 喔！

<details><summary>Gradle 的設定</summary>


```groovy
android {
    buildFeatures {
        compose = true
    }

    composeOptions {
        kotlinCompilerExtensionVersion = "1.4.2"
    }
}
```

以及 Dependency

```groovy
dependencies {

    val composeBom = platform("androidx.compose:compose-bom:2023.01.00")
    implementation composeBom
    androidTestImplementation composeBom

    // Choose one of the following:
    // Material Design 3
    implementation("androidx.compose.material3:material3")
    // or Material Design 2
    implementation("androidx.compose.material:material")
    // or skip Material Design and build directly on top of foundational components
    implementation("androidx.compose.foundation:foundation")
    // or only import the main APIs for the underlying toolkit systems,
    // such as input and measurement/layout
    implementation("androidx.compose.ui:ui")

    // Android Studio Preview support
    implementation("androidx.compose.ui:ui-tooling-preview")
    debugImplementation("androidx.compose.ui:ui-tooling")

    // UI Tests
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
    debugImplementation("androidx.compose.ui:ui-test-manifest")

    // Optional - Included automatically by material, only add when you need
    // the icons but not the material library (e.g. when using Material3 or a
    // custom design system based on Foundation)
    implementation("androidx.compose.material:material-icons-core")
    // Optional - Add full set of material icons
    implementation("androidx.compose.material:material-icons-extended")
    // Optional - Add window size utils
    implementation("androidx.compose.material3:material3-window-size-class")

    // Optional - Integration with activities
    implementation("androidx.activity:activity-compose:1.6.1")
    // Optional - Integration with ViewModels
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.5.1")
    // Optional - Integration with LiveData
    implementation("androidx.compose.runtime:runtime-livedata")
    // Optional - Integration with RxJava
    implementation("androidx.compose.runtime:runtime-rxjava2")

}
```

我們也可以通過 AS 自動產生 Compose Activity。 最後 Gradle 便會新增以下 Dependency ：

```groovy
implementation 'androidx.lifecycle:lifecycle-runtime-ktx:2.6.1'
implementation 'androidx.activity:activity-compose:1.7.2'
implementation platform('androidx.compose:compose-bom:2022.10.00')
implementation 'androidx.compose.ui:ui-graphics'
implementation 'androidx.compose.material3:material3'
```
</details>

<br>


# TextView 的形成

無論是使用 XML 還是 Compose，最後還是會由 **TextView.java** 來實作。

不過，Compose 還需要一些額外的流程。

## Compose

Compose 在建立 **Text** 時會有以下流程：

<details><summary>建立 <b>Text</b> 源碼</summary>

```kotlin
@Composable
fun Text(
    text: String,
    modifier: Modifier = Modifier,
    color: Color = Color.Unspecified,
    fontSize: TextUnit = TextUnit.Unspecified,
    fontStyle: FontStyle? = null,
    fontWeight: FontWeight? = null,
    fontFamily: FontFamily? = null,
    letterSpacing: TextUnit = TextUnit.Unspecified,
    textDecoration: TextDecoration? = null,
    textAlign: TextAlign? = null,
    lineHeight: TextUnit = TextUnit.Unspecified,
    overflow: TextOverflow = TextOverflow.Clip,
    softWrap: Boolean = true,
    maxLines: Int = Int.MAX_VALUE,
    onTextLayout: (TextLayoutResult) -> Unit = {},
    style: TextStyle = LocalTextStyle.current
) {

    val textColor = color.takeOrElse {
        style.color.takeOrElse {
            LocalContentColor.current
        }
    }
    // NOTE(text-perf-review): It might be worthwhile writing a bespoke merge implementation that
    // will avoid reallocating if all of the options here are the defaults
    val mergedStyle = style.merge(
        TextStyle(
            color = textColor,
            fontSize = fontSize,
            fontWeight = fontWeight,
            textAlign = textAlign,
            lineHeight = lineHeight,
            fontFamily = fontFamily,
            textDecoration = textDecoration,
            fontStyle = fontStyle,
            letterSpacing = letterSpacing
        )
    )
    BasicText(
        text,
        modifier,
        mergedStyle,
        onTextLayout,
        overflow,
        softWrap,
        maxLines,
    )
}
```

</details>

<br>

1. 建立 **Text** 時會需要取得 **TextStyle**。 此時會將 **DynamicProvidableCompositionLocal** 以 key 的方式記錄在 **SlotTable** 中。
2. 繼續將預設的 **TextStyle** 與客製化的 TextStyle 合在一起。
3. 建立 **BasicText**

<details><summary>建立 <b>BasicText</b></summary>

```kotlin
@OptIn(InternalFoundationTextApi::class)
@Composable
fun BasicText(
    text: String,
    modifier: Modifier = Modifier,
    style: TextStyle = TextStyle.Default,
    onTextLayout: (TextLayoutResult) -> Unit = {},
    overflow: TextOverflow = TextOverflow.Clip,
    softWrap: Boolean = true,
    maxLines: Int = Int.MAX_VALUE,
) {
    // NOTE(text-perf-review): consider precomputing layout here by pushing text to a channel...
    // something like:
    // remember(text) { precomputeTextLayout(text) }
    require(maxLines > 0) { "maxLines should be greater than 0" }

    // selection registrar, if no SelectionContainer is added ambient value will be null
    val selectionRegistrar = LocalSelectionRegistrar.current
    val density = LocalDensity.current
    val fontFamilyResolver = LocalFontFamilyResolver.current

    // The ID used to identify this CoreText. If this CoreText is removed from the composition
    // tree and then added back, this ID should stay the same.
    // Notice that we need to update selectable ID when the input text or selectionRegistrar has
    // been updated.
    // When text is updated, the selection on this CoreText becomes invalid. It can be treated
    // as a brand new CoreText.
    // When SelectionRegistrar is updated, CoreText have to request a new ID to avoid ID collision.

    // NOTE(text-perf-review): potential bug. selectableId is regenerated here whenever text
    // changes, but it is only saved in the initial creation of TextState.
    val selectableId = if (selectionRegistrar == null) {
        SelectionRegistrar.InvalidSelectableId
    } else {
        rememberSaveable(text, selectionRegistrar, saver = selectionIdSaver(selectionRegistrar)) {
            selectionRegistrar.nextSelectableId()
        }
    }

    val controller = remember {
        TextController(
            TextState(
                TextDelegate(
                    text = AnnotatedString(text),
                    style = style,
                    density = density,
                    softWrap = softWrap,
                    fontFamilyResolver = fontFamilyResolver,
                    overflow = overflow,
                    maxLines = maxLines,
                ),
                selectableId
            )
        )
    }
    val state = controller.state
    if (!currentComposer.inserting) {
        controller.setTextDelegate(
            updateTextDelegate(
                current = state.textDelegate,
                text = text,
                style = style,
                density = density,
                softWrap = softWrap,
                fontFamilyResolver = fontFamilyResolver,
                overflow = overflow,
                maxLines = maxLines,
            )
        )
    }
    state.onTextLayout = onTextLayout
    controller.update(selectionRegistrar)
    if (selectionRegistrar != null) {
        state.selectionBackgroundColor = LocalTextSelectionColors.current.backgroundColor
    }

    Layout(modifier.then(controller.modifiers), controller.measurePolicy)
}
```

</details>

<br>


原本我只是單純想要暸解 **TextView** 是如何將字顯示出來的。但發現原來 TextView 有三種 **Layout**，包括
1. **StaticLayout**
2. **BoringLayout**
3. **DynamicLayout**




# 顯示垂直的文字




<br><br><br><br><br><br><br><br>
