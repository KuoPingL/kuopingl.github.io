---
layout: post
title:  "Android - Compose Camp 2022"
date:   2022-08-08 16:15:07 +0800
categories: [jetpack compose, android, beginner]
---

## Background
In this article, let's take a look at what we can learn from the [Jetpack Compose Pathway](https://developer.android.com/courses/android-basics-compose/course?authuser=1) by Google.

Be sure to sign in to your Google account and create a profile to get badges :

<center>
<img src = "/images/posts/jekyll/jetpack_compose/android_camp_badge.png" style = "width:30%"/>
</center>
<br>

><br>
>
>Just a heads up, this article **DOES NOT** just cover the the study path, but also contains notes and things that I dig up while learning Jetpack Compose.
><br><br/>

## WHAT is a Jetpack Compose and WHY do we use it ?

Here's a statement on Jetpack Compose given by Android developer course :

><br>
>
>**Jetpack Compose** is a modern toolkit for building Android UIs. Compose **simplifies and accelerates** UI development on Android with less code, powerful tools, and intuitive Kotlin capabilities. With Compose, you can build your UI by defining a set of functions, called **composable functions**, that **take in data and emit UI elements**.
><br><br/>

Traditionally, we uses **imperative** programming to write our code. Meaning that we need to explicitly specify each instruction step by step and mutating the internal state of the widget, such as :

```java
button.setText(String);
container.addChild(View);
img.setImageBitmap(Bitmap);
```

In order to update the view, we also need to *walk* through the tree using `findViewById`.

However, **manually** manipulating the state is
error-prone. Such as updating a node that has been removed from the UI.

As the application become more complex, it is more likely to encounter these errors and the amount of work to maintain the application will also increased.

To **simplify the engineering associated with building and updating user interfaces**, the entire industry has shifted to **declarative** UI model .

**Declarative** programming is a paradigm describing WHAT the program does, without explicitly specifying its control flow.

And **Compose** is a declarative UI framework.

Next, we will get a feeling on how **Compose** looks by creating an empty compose activity.

## Empty Compose Activity
The empty activity in Compose is quite different from non-compose one, especially how the code is structed.

Let's take a peek at how **MainActivity.kt** looks like :

<details>
<summary>MainActivity.kt</summary>

```Kotlin
  package com.javanrhinos.greetingcard

  import android.os.Bundle
  import androidx.activity.ComponentActivity
  import androidx.activity.compose.setContent
  import androidx.compose.foundation.layout.fillMaxSize
  import androidx.compose.material.MaterialTheme
  import androidx.compose.material.Surface
  import androidx.compose.material.Text
  import androidx.compose.runtime.Composable
  import androidx.compose.ui.Modifier
  import androidx.compose.ui.tooling.preview.Preview
  import com.javanrhinos.greetingcard.ui.theme.GreetingCardTheme

  class MainActivity : ComponentActivity() {
      override fun onCreate(savedInstanceState: Bundle?) {
          super.onCreate(savedInstanceState)
          setContent {
              GreetingCardTheme {
                  // A surface container using the 'background' color from the theme
                  Surface(
                      modifier = Modifier.fillMaxSize(),
                      color = MaterialTheme.colors.background
                  ) {
                      Greeting("Android")
                  }
              }
          }
      }
  }

  @Composable
  fun Greeting(name: String) {
      Text(text = "Hello $name!")
  }

  @Preview(showBackground = true)
  @Composable
  fun DefaultPreview() {
      GreetingCardTheme {
          Greeting("Android")
      }
  }
```

</details>

<br>

If you look carefully you can see the following differences :

   1. **Inheritage**
      MainActivity is now inherited from **ComponentActivity** instead of **AppCompatActivity**, which has the following additional implementations :
   <br>
       ```kotlin
         AppCompatCallback,
         TaskStackBuilder.SupportParentable,
         ActionBarDrawerToggle.DelegateProvider

         ActivityCompat.OnRequestPermissionsResultCallback,
         ActivityCompat.RequestPermissionsRequestCodeValidator
       ```
  2. **setContent**
     Normally, we pass in an XML id to `setContent` to inflate our view. But with compose, we don't need to use XML, but instead, we need to use **Composable** compoents.
<br>
  3. The way to define **Style**
     If we look at the **themes.xml** file :
     <br>
     ```XML
       <resources>
           <style name="Theme.GreetingCard" parent="android:Theme.Material.Light.NoActionBar">
               <item name="android:statusBarColor">@color/purple_700</item>
           </style>
       </resources>
     ```

     it looks just like normal. However, if we look into the style **Theme.GreetingCard** :

     ```kotlin
       @Composable
       fun GreetingCardTheme(darkTheme: Boolean = isSystemInDarkTheme(), content: @Composable () -> Unit) {
           val colors = if (darkTheme) {
               DarkColorPalette
           } else {
               LightColorPalette
           }

           MaterialTheme(
               colors = colors,
               typography = Typography,
               shapes = Shapes,
               content = content
           )
       }
     ```

     It is actually composed of **Composable** and **Stable**, the color, components.

Now those are just some of the changes you will find in MainActivity. Let's take a look at the changes in dependencies inside **build.gradle**.

### build.gradle(app)

<details>
<summary>build.gradle(app)</summary>

```groovy
  plugins {
      id 'com.android.application'
      id 'org.jetbrains.kotlin.android'
  }

  android {
      namespace 'com.javanrhinos.greetingcard'
      compileSdk 33

      defaultConfig {
          applicationId "com.javanrhinos.greetingcard"
          minSdk 21
          targetSdk 33
          versionCode 1
          versionName "1.0"

          testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
          vectorDrawables {
              useSupportLibrary true
          }
      }

      buildTypes {
          release {
              minifyEnabled false
              proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
          }
      }
      compileOptions {
          sourceCompatibility JavaVersion.VERSION_1_8
          targetCompatibility JavaVersion.VERSION_1_8
      }
      kotlinOptions {
          jvmTarget = '1.8'
      }
      buildFeatures {
          compose true
      }
      composeOptions {
          kotlinCompilerExtensionVersion '1.1.1'
      }
      packagingOptions {
          resources {
              excludes += '/META-INF/{AL2.0,LGPL2.1}'
          }
      }
  }

  dependencies {

      implementation 'androidx.core:core-ktx:1.9.0'
      implementation 'androidx.lifecycle:lifecycle-runtime-ktx:2.5.1'
      implementation 'androidx.activity:activity-compose:1.6.1'
      implementation "androidx.compose.ui:ui:$compose_ui_version"
      implementation "androidx.compose.ui:ui-tooling-preview:$compose_ui_version"
      implementation 'androidx.compose.material:material:1.3.1'
      testImplementation 'junit:junit:4.13.2'
      androidTestImplementation 'androidx.test.ext:junit:1.1.4'
      androidTestImplementation 'androidx.test.espresso:espresso-core:3.5.0'
      androidTestImplementation "androidx.compose.ui:ui-test-junit4:$compose_ui_version"
      debugImplementation "androidx.compose.ui:ui-tooling:$compose_ui_version"
      debugImplementation "androidx.compose.ui:ui-test-manifest:$compose_ui_version"
  }
```

</details>

<br>

The build.gradle in a compose project does share few dependencies with a non-compose project :

```groovy
  implementation 'androidx.core:core-ktx:1.9.0'

  testImplementation 'junit:junit:4.13.2'
  androidTestImplementation 'androidx.test.ext:junit:1.1.4'
  androidTestImplementation 'androidx.test.espresso:espresso-core:3.5.0'
```

However, to better suit a compose project, many dependencies are being replaced or added, as listed below :

role|Non-Compose|Compose|
|:--|:--|:--|
|Activity| `androidx.appcompat:appcompat`  | `androidx.activity:activity-compose`  |
|Theme| `com.google.android.material:material`  | `androidx.compose.material:material`  |
|Lifecycle <br>(without ViewModel or LiveData)   |  --- |  `androidx.lifecycle:lifecycle-runtime-ktx` |
|UI   | `androidx.constraintlayout:constraintlayout`  |  `androidx.compose.ui:ui` |
|UI Preview   | ---  | `androidx.compose.ui:ui-tooling-preview`  |
|UI Test<br> (androidTestImplementation)  | ---  | `androidx.compose.ui:ui-test-junit4`  |
|UI Tooling<br> (debugImplementation)  |  --- |  `androidx.compose.ui:ui-tooling` |
| UI Test manifest<br> (debugImplementation)  |   ---| `androidx.compose.ui:ui-test-manifest`  |

For a full list of dependencies needed in compose, checkout this [link](https://developer.android.com/jetpack/compose/setup?authuser=1#compose-compiler).

### DefaultPreview
Despite all the differences mentioned above, a cool feature in Compose is the preview feature.

By defining a **Composable** function with **Preview** annotation as follows :

```kotlin
  @Preview(showBackground = true)
  @Composable
  fun DefaultPreview() {
      GreetingCardTheme {
          Greeting("Android")
      }
  }
```

We can preview the result on the design screen :

<center>
<img src = "/images/posts/jekyll/jetpack_compose/composable_preview.png" style = "width:80%"/>
</center>

<br>

There are also other arguments that can be used when previewing :

```kotlin
  @MustBeDocumented
  @Retention(AnnotationRetention.BINARY)
  @Target(
      AnnotationTarget.ANNOTATION_CLASS,
      AnnotationTarget.FUNCTION
  )
  @Repeatable
  annotation class Preview(
      val name: String = "",
      val group: String = "",
      @IntRange(from = 1) val apiLevel: Int = -1,
      // TODO(mount): Make this Dp when they are inline classes
      val widthDp: Int = -1,
      // TODO(mount): Make this Dp when they are inline classes
      val heightDp: Int = -1,
      val locale: String = "",
      @FloatRange(from = 0.01) val fontScale: Float = 1f,
      val showSystemUi: Boolean = false,
      val showBackground: Boolean = false,
      val backgroundColor: Long = 0,
      @UiMode val uiMode: Int = 0,
      @Device val device: String = Devices.DEFAULT
  )
```

Now that we see their differences, let's take some time to understand them.


## What is Composable ?
A **Composable** is an *annotation* that allows a function to become a **building block** of an application built with Compose.

This is done by informing the **Compose compiler** that this function is intended to **convert data into UI**.

```kotlin
  @MustBeDocumented
  @Retention(AnnotationRetention.BINARY)
  @Target(
    AnnotationTarget.FUNCTION,
    AnnotationTarget.TYPE,
    AnnotationTarget.TYPE_PARAMETER,
    AnnotationTarget.PROPERTY_GETTER
  )
  annotation class Composable
```

Through the code, we understand that any function, type and getter can be annotated with `Composable` and become a **Composable Function**.

Examples :

```kotlin
  // function
  @Composable fun Foo() { ... }
  val foo = @Composable { ... } // lambda

  // type
  var foo: @Composable () -> Unit = { ... } // same as lambda
  foo: @Composable () -> Unit // parameter type

  // type parameter
  foo: (@Composable () -> Unit) -> Unit

  // getter
  val foo: Int @Composable get() { ... } // also wtih var

```

A **Composable** also has the following properties :

1.  **Composable** functions can only be called from within another **Composable** function and **cannot return anything**.
2. By passing a **Composable** into a **Composable** function, a **composable context** is also implicitly passed into it when it is being called within the function.
3. The **composable content** can be used to store information from previous executions of the function that happened at the same logical point of the tree.

The advantages of a **Composable** function are that it is **fast**, **idempotent** and **free of side-effects**, meaning :

- The function behaves the same way when called multiple times with the same argument, and it does not use other values such as global variables or calls to `random()`.
- The function describes the UI without any side-effects, such as modifying properties or global variables.
- Free of side-effects also implies Composable function can be **executed in any order and even in parallel**.


<details>
<summary>Definition of Idempotent and Side-effect</summary>

><br>
>
>**Idempotent**
> "The property of certain operations in mathematics and computer science whereby they can be applied multiple times without changing the result beyond the initial application."[[wiki](https://en.wikipedia.org/wiki/Idempotence#Computer_science_meaning)]
><br>
>
>**Side Effect**
>"When an operation, function or expression is said to have a side effect if it modifies some state variable value(s) outside its local environment, which is to say if it has any observable effect other than its primary effect of returning a value to the invoker of the operation.
>
>Example side effects include modifying a non-local variable, modifying a static local variable, modifying a mutable argument passed by reference, performing I/O or calling other functions with side-effects. In the presence of side effects, a program's behaviour may depend on history; that is, the order of evaluation matters. Understanding and debugging a function with side effects requires knowledge about the context and its possible histories." [[wiki](https://en.wikipedia.org/wiki/Side_effect_(computer_science))]
><br/>

</details>

<br>

Enough said, let's dive into the code and see how can we create and modify a simple TextView.

## Creating a TextView (Text)
To create a TextView in Compose, instead of using **TextView**, we can simply use **Text** :

```kotlin
  Text(text = "Hello $name!")

  // wrap in a composable function
  @Composable
  fun Greeting(name: String) {
      Text(text = "Hello $name!")
  }

  // if we want use material design,
  // we can also wrap it in a Surface
  @Composable
  fun Greeting(name: String) {
      Surface(color = Color.Magenta) {
          Text(text = "Hello $name!")
      }
  }
```

<center>
<img src = "/images/posts/jekyll/jetpack_compose/text_default.png" style="width:80%"/>
</center>

<br>

A **Text** is a composable function that takes in variables that are text related.

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
  )
```

Among them, a **Modifier** is the common parameter that is seen all over composable functions.

A **Modifier** is a **Stable** interface, it acts as an ordered, immutable collection of modifier elements (`Modifier.Element`) that decorate or add behavior to Compose UI elements.

```kotlin
  @Suppress("ModifierFactoryExtensionFunction")
  @Stable
  @JvmDefaultWithCompatibility
  interface Modifier {
      fun <R> foldIn(initial: R, operation: (R, Element) -> R): R
      fun <R> foldOut(initial: R, operation: (Element, R) -> R): R

      fun any(predicate: (Element) -> Boolean): Boolean
      fun all(predicate: (Element) -> Boolean): Boolean

      infix fun then(other: Modifier): Modifier =
          if (other === Modifier) this else CombinedModifier(this, other)

      companion object : Modifier {
          override fun <R> foldIn(initial: R, operation: (R, Element) -> R): R = initial
          override fun <R> foldOut(initial: R, operation: (Element, R) -> R): R = initial
          override fun any(predicate: (Element) -> Boolean): Boolean = false
          override fun all(predicate: (Element) -> Boolean): Boolean = true
          override infix fun then(other: Modifier): Modifier = other
          override fun toString() = "Modifier"
      }

      @JvmDefaultWithCompatibility
      interface Element : Modifier {
          override fun <R> foldIn(initial: R, operation: (R, Element) -> R): R =
              operation(initial, this)

          override fun <R> foldOut(initial: R, operation: (Element, R) -> R): R =
              operation(this, initial)

          override fun any(predicate: (Element) -> Boolean): Boolean = predicate(this)

          override fun all(predicate: (Element) -> Boolean): Boolean = predicate(this)
      }
  }
```

A **Stable** is an annotation that is used to communicate some guarantees to the compose compiler about how a certain type or function will behave.

```kotlin
  @MustBeDocumented
  @Target(
      AnnotationTarget.CLASS,
      AnnotationTarget.FUNCTION,
      AnnotationTarget.PROPERTY_GETTER,
      AnnotationTarget.PROPERTY
  )
  @Retention(AnnotationRetention.BINARY)
  @StableMarker
  annotation class Stable
```

When applying **Stable** to a class or an interface, **Stable** indicates the following to be true :

- The result of `equals` will always return the same result for the same two instances.
- When a **public property** of the type changes, composition will be notified.
- All public property types are **stable**.

An example of using **Modifier** is to add padding on Textview :

```kotlin
  Text(text = "Hello $name!", Modifier.padding(24.dp))
```

<center>
<img src = "/images/posts/jekyll/jetpack_compose/text_padding_24.png" style="width:80%"/>
</center>

<br>

And the definition of `Modifier.padding` is as follow :

```kotlin
  @Stable
  fun Modifier.padding(all: Dp) =
      this.then(
          PaddingModifier(
              start = all,
              top = all,
              end = all,
              bottom = all,
              rtlAware = true,
              inspectorInfo = debugInspectorInfo {
                  name = "padding"
                  value = all
              }
          )
      )
```

which gets a **PaddingModifer** in return :

```kotlin
  private class PaddingModifier(
      val start: Dp = 0.dp,
      val top: Dp = 0.dp,
      val end: Dp = 0.dp,
      val bottom: Dp = 0.dp,
      val rtlAware: Boolean,
      inspectorInfo: InspectorInfo.() -> Unit
  ) : LayoutModifier, InspectorValueInfo(inspectorInfo)
```

A **PaddingModifer** is a **LayoutModifier** that overrides `MeasureScope.measure` that will return an interface holding the size and alignment lines of the measured layout, as well as the children positioning logic :

```kotlin
  override fun MeasureScope.measure(
      measurable: Measurable,
      constraints: Constraints
  ): MeasureResult {

      val horizontal = start.roundToPx() + end.roundToPx()
      val vertical = top.roundToPx() + bottom.roundToPx()

      val placeable = measurable.measure(constraints.offset(-horizontal, -vertical))

      val width = constraints.constrainWidth(placeable.width + horizontal)
      val height = constraints.constrainHeight(placeable.height + vertical)
      return layout(width, height) {
          if (rtlAware) {
              placeable.placeRelative(start.roundToPx(), top.roundToPx())
          } else {
              placeable.place(start.roundToPx(), top.roundToPx())
          }
      }
  }
```

## Basic Layout Elements
A simply layout can be created using 3 different composable components, including :

1. **Column**

```kotlin
  @Composable
  inline fun Column(
      modifier: Modifier = Modifier,
      verticalArrangement: Arrangement.Vertical = Arrangement.Top,
      horizontalAlignment: Alignment.Horizontal = Alignment.Start,
      content: @Composable ColumnScope.() -> Unit
  ) {
      val measurePolicy = columnMeasurePolicy(verticalArrangement, horizontalAlignment)
      Layout(
          content = { ColumnScopeInstance.content() },
          measurePolicy = measurePolicy,
          modifier = modifier
      )
  }

```
<br>

**Example**

```kotlin
// Using Trailing Lambda Syntax
  @Composable
  fun Greeting(name: String) {
      Column(Modifier.padding(vertical = 8.dp, horizontal = 10.dp)) {
          Text(text = "First Row ")
          Text(text = "Second Row")
      }
  }

// or using explicit parameter
  @Composable
  fun Greeting(name: String) {
      Column(Modifier.padding(vertical = 8.dp, horizontal = 10.dp),
      content = {
        Text(text = "First Row ")
        Text(text = "Second Row")
        })
  }
```

<center>
<img src = "/images/posts/jekyll/jetpack_compose/column_two_text.png" style="width:80%"/>
</center>

<br>


2. **Row** :

```kotlin
  @Composable
  inline fun Row(
      modifier: Modifier = Modifier,
      horizontalArrangement: Arrangement.Horizontal = Arrangement.Start,
      verticalAlignment: Alignment.Vertical = Alignment.Top,
      content: @Composable RowScope.() -> Unit
  ) {
      val measurePolicy = rowMeasurePolicy(horizontalArrangement, verticalAlignment)
      Layout(
          content = { RowScopeInstance.content() },
          measurePolicy = measurePolicy,
          modifier = modifier
      )
  }
```

<br>

**Example**

```kotlin
  @Composable
  fun Greeting(name: String) {
      Row(Modifier.padding(vertical = 8.dp, horizontal = 10.dp)) {
          Text(text = "First Column ")
          Text(text = "Second Column")
      }
  }
```
<center>
<img src = "/images/posts/jekyll/jetpack_compose/row_two_text.png" style="width:80%"/>
</center>

<br>


3. **Box**

```kotlin
  @Composable
  inline fun Box(
      modifier: Modifier = Modifier,
      contentAlignment: Alignment = Alignment.TopStart,
      propagateMinConstraints: Boolean = false,
      content: @Composable BoxScope.() -> Unit
  ) {
      val measurePolicy = rememberBoxMeasurePolicy(contentAlignment, propagateMinConstraints)
      Layout(
          content = { BoxScopeInstance.content() },
          measurePolicy = measurePolicy,
          modifier = modifier
      )
  }
```

Unlike **Column** and **Row**, a **Box** will overlay components on top of each other.

Example :

```kotlin
  @Composable
  fun Greeting(name: String) {
      Box(Modifier.padding(vertical = 8.dp, horizontal = 10.dp)) {
          Text(text = "First Column ")
          Text(text = "Second Column")
      }
  }
```

<center>
<img src = "/images/posts/jekyll/jetpack_compose/box_two_text.png" style="width:80%"/>
</center>

<br>

From the source code of these components, you can tell that main component that affects the way `content` is being displayed is the **MeasurePolicy**.


### MeasurePolicy

```kotlin
  @Stable
  @JvmDefaultWithCompatibility
  fun interface MeasurePolicy {
    fun MeasureScope.measure(
        measurables: List<Measurable>,
        constraints: Constraints
    ): MeasureResult
  }
```

><br>
>
>A **MeasurePolicy** is an **interface** that is used to define the **measure and layout behavior** of a Layout.
><br/>


In the function `MeasureScope.measure`, it takes in a list of composition that is measurable and a **Constraints** that defines a range, in pixels, within which the measured layout should choose a size.

Here are the **MeasurePolicy** implementations from **Column**, **Row** and **Box**.

<details>
<summary><b>columnMeasurePolicy</b></summary>

```kotlin
  // columnMeasurePolicy
  @PublishedApi
  @Composable
  internal fun columnMeasurePolicy(
      verticalArrangement: Arrangement.Vertical,
      horizontalAlignment: Alignment.Horizontal
  ) = remember(verticalArrangement, horizontalAlignment) {
      if (verticalArrangement == Arrangement.Top && horizontalAlignment == Alignment.Start) {
          DefaultColumnMeasurePolicy
      } else {
          rowColumnMeasurePolicy(
              orientation = LayoutOrientation.Vertical,
              arrangement = { totalSize, size, _, density, outPosition ->
                  with(verticalArrangement) { density.arrange(totalSize, size, outPosition) }
              },
              arrangementSpacing = verticalArrangement.spacing,
              crossAxisAlignment = CrossAxisAlignment.horizontal(horizontalAlignment),
              crossAxisSize = SizeMode.Wrap
          )
      }
  }

  @PublishedApi
  internal val DefaultColumnMeasurePolicy = rowColumnMeasurePolicy(
      orientation = LayoutOrientation.Vertical,
      arrangement = { totalSize, size, _, density, outPosition ->
          with(Arrangement.Top) { density.arrange(totalSize, size, outPosition) }
      },
      arrangementSpacing = Arrangement.Top.spacing,
      crossAxisAlignment = CrossAxisAlignment.horizontal(Alignment.Start),
      crossAxisSize = SizeMode.Wrap
  )

```


</details>


<details>
<summary><b>rowMeasurePolicy</b></summary>

```kotlin
  @PublishedApi
  @Composable
  internal fun rowMeasurePolicy(
      horizontalArrangement: Arrangement.Horizontal,
      verticalAlignment: Alignment.Vertical
  ) = remember(horizontalArrangement, verticalAlignment) {
      if (horizontalArrangement == Arrangement.Start && verticalAlignment == Alignment.Top) {
          DefaultRowMeasurePolicy
      } else {
          rowColumnMeasurePolicy(
              orientation = LayoutOrientation.Horizontal,
              arrangement = { totalSize, size, layoutDirection, density, outPosition ->
                  with(horizontalArrangement) {
                      density.arrange(totalSize, size, layoutDirection, outPosition)
                  }
              },
              arrangementSpacing = horizontalArrangement.spacing,
              crossAxisAlignment = CrossAxisAlignment.vertical(verticalAlignment),
              crossAxisSize = SizeMode.Wrap
          )
      }
  }

  @PublishedApi
  internal val DefaultRowMeasurePolicy = rowColumnMeasurePolicy(
      orientation = LayoutOrientation.Horizontal,
      arrangement = { totalSize, size, layoutDirection, density, outPosition ->
          with(Arrangement.Start) { density.arrange(totalSize, size, layoutDirection, outPosition) }
      },
      arrangementSpacing = Arrangement.Start.spacing,
      crossAxisAlignment = CrossAxisAlignment.vertical(Alignment.Top),
      crossAxisSize = SizeMode.Wrap
  )
```

</details>



<details>
<summary><b>rowColumnMeasurePolicy</b></summary>

```kotlin
  internal fun rowColumnMeasurePolicy(
      orientation: LayoutOrientation,
      arrangement: (Int, IntArray, LayoutDirection, Density, IntArray) -> Unit,
      arrangementSpacing: Dp,
      crossAxisSize: SizeMode,
      crossAxisAlignment: CrossAxisAlignment
  ): MeasurePolicy {
      fun Placeable.mainAxisSize() =
          if (orientation == LayoutOrientation.Horizontal) width else height

      fun Placeable.crossAxisSize() =
          if (orientation == LayoutOrientation.Horizontal) height else width

      return object : MeasurePolicy {
          override fun MeasureScope.measure(
              measurables: List<Measurable>,
              constraints: Constraints
          ): MeasureResult {
              @Suppress("NAME_SHADOWING")
              val constraints = OrientationIndependentConstraints(constraints, orientation)
              val arrangementSpacingPx = arrangementSpacing.roundToPx()

              var totalWeight = 0f
              var fixedSpace = 0
              var crossAxisSpace = 0
              var weightChildrenCount = 0

              var anyAlignBy = false
              val placeables = arrayOfNulls<Placeable>(measurables.size)
              val rowColumnParentData = Array(measurables.size) { measurables[it].data }

              // First measure children with zero weight.
              var spaceAfterLastNoWeight = 0
              for (i in measurables.indices) {
                  val child = measurables[i]
                  val parentData = rowColumnParentData[i]
                  val weight = parentData.weight

                  if (weight > 0f) {
                      totalWeight += weight
                      ++weightChildrenCount
                  } else {
                      val mainAxisMax = constraints.mainAxisMax
                      val placeable = child.measure(
                          // Ask for preferred main axis size.
                          constraints.copy(
                              mainAxisMin = 0,
                              mainAxisMax = if (mainAxisMax == Constraints.Infinity) {
                                  Constraints.Infinity
                              } else {
                                  mainAxisMax - fixedSpace
                              },
                              crossAxisMin = 0
                          ).toBoxConstraints(orientation)
                      )
                      spaceAfterLastNoWeight = min(
                          arrangementSpacingPx,
                          mainAxisMax - fixedSpace - placeable.mainAxisSize()
                      )
                      fixedSpace += placeable.mainAxisSize() + spaceAfterLastNoWeight
                      crossAxisSpace = max(crossAxisSpace, placeable.crossAxisSize())
                      anyAlignBy = anyAlignBy || parentData.isRelative
                      placeables[i] = placeable
                  }
              }

              var weightedSpace = 0
              if (weightChildrenCount == 0) {
                  // fixedSpace contains an extra spacing after the last non-weight child.
                  fixedSpace -= spaceAfterLastNoWeight
              } else {
                  // Measure the rest according to their weights in the remaining main axis space.
                  val targetSpace =
                      if (totalWeight > 0f && constraints.mainAxisMax != Constraints.Infinity) {
                          constraints.mainAxisMax
                      } else {
                          constraints.mainAxisMin
                      }
                  val remainingToTarget =
                      targetSpace - fixedSpace - arrangementSpacingPx * (weightChildrenCount - 1)

                  val weightUnitSpace = if (totalWeight > 0) remainingToTarget / totalWeight else 0f
                  var remainder = remainingToTarget - rowColumnParentData.sumOf {
                      (weightUnitSpace * it.weight).roundToInt()
                  }

                  for (i in measurables.indices) {
                      if (placeables[i] == null) {
                          val child = measurables[i]
                          val parentData = rowColumnParentData[i]
                          val weight = parentData.weight
                          check(weight > 0) { "All weights <= 0 should have placeables" }
                          // After the weightUnitSpace rounding, the total space going to be occupied
                          // can be smaller or larger than remainingToTarget. Here we distribute the
                          // loss or gain remainder evenly to the first children.
                          val remainderUnit = remainder.sign
                          remainder -= remainderUnit
                          val childMainAxisSize = max(
                              0,
                              (weightUnitSpace * weight).roundToInt() + remainderUnit
                          )
                          val placeable = child.measure(
                              OrientationIndependentConstraints(
                                  if (parentData.fill && childMainAxisSize != Constraints.Infinity) {
                                      childMainAxisSize
                                  } else {
                                      0
                                  },
                                  childMainAxisSize,
                                  0,
                                  constraints.crossAxisMax
                              ).toBoxConstraints(orientation)
                          )
                          weightedSpace += placeable.mainAxisSize()
                          crossAxisSpace = max(crossAxisSpace, placeable.crossAxisSize())
                          anyAlignBy = anyAlignBy || parentData.isRelative
                          placeables[i] = placeable
                      }
                  }
                  weightedSpace = (weightedSpace + arrangementSpacingPx * (weightChildrenCount - 1))
                      .coerceAtMost(constraints.mainAxisMax - fixedSpace)
              }

              var beforeCrossAxisAlignmentLine = 0
              var afterCrossAxisAlignmentLine = 0
              if (anyAlignBy) {
                  for (i in placeables.indices) {
                      val placeable = placeables[i]!!
                      val parentData = rowColumnParentData[i]
                      val alignmentLinePosition = parentData.crossAxisAlignment
                          ?.calculateAlignmentLinePosition(placeable)
                      if (alignmentLinePosition != null) {
                          beforeCrossAxisAlignmentLine = max(
                              beforeCrossAxisAlignmentLine,
                              alignmentLinePosition.let {
                                  if (it != AlignmentLine.Unspecified) it else 0
                              }
                          )
                          afterCrossAxisAlignmentLine = max(
                              afterCrossAxisAlignmentLine,
                              placeable.crossAxisSize() -
                                  (
                                      alignmentLinePosition.let {
                                          if (it != AlignmentLine.Unspecified) {
                                              it
                                          } else {
                                              placeable.crossAxisSize()
                                          }
                                      }
                                      )
                          )
                      }
                  }
              }

              // Compute the Row or Column size and position the children.
              val mainAxisLayoutSize = max(fixedSpace + weightedSpace, constraints.mainAxisMin)
              val crossAxisLayoutSize = if (constraints.crossAxisMax != Constraints.Infinity &&
                  crossAxisSize == SizeMode.Expand
              ) {
                  constraints.crossAxisMax
              } else {
                  max(
                      crossAxisSpace,
                      max(
                          constraints.crossAxisMin,
                          beforeCrossAxisAlignmentLine + afterCrossAxisAlignmentLine
                      )
                  )
              }
              val layoutWidth = if (orientation == Horizontal) {
                  mainAxisLayoutSize
              } else {
                  crossAxisLayoutSize
              }
              val layoutHeight = if (orientation == Horizontal) {
                  crossAxisLayoutSize
              } else {
                  mainAxisLayoutSize
              }

              val mainAxisPositions = IntArray(measurables.size) { 0 }
              return layout(layoutWidth, layoutHeight) {
                  val childrenMainAxisSize = IntArray(measurables.size) { index ->
                      placeables[index]!!.mainAxisSize()
                  }
                  arrangement(
                      mainAxisLayoutSize,
                      childrenMainAxisSize,
                      layoutDirection,
                      this@measure,
                      mainAxisPositions
                  )

                  placeables.forEachIndexed { index, placeable ->
                      placeable!!
                      val parentData = rowColumnParentData[index]
                      val childCrossAlignment = parentData.crossAxisAlignment ?: crossAxisAlignment

                      val crossAxis = childCrossAlignment.align(
                          size = crossAxisLayoutSize - placeable.crossAxisSize(),
                          layoutDirection = if (orientation == Horizontal) {
                              LayoutDirection.Ltr
                          } else {
                              layoutDirection
                          },
                          placeable = placeable,
                          beforeCrossAxisAlignmentLine = beforeCrossAxisAlignmentLine
                      )

                      if (orientation == Horizontal) {
                          placeable.place(mainAxisPositions[index], crossAxis)
                      } else {
                          placeable.place(crossAxis, mainAxisPositions[index])
                      }
                  }
              }
          }

          override fun IntrinsicMeasureScope.minIntrinsicWidth(
              measurables: List<IntrinsicMeasurable>,
              height: Int
          ) = MinIntrinsicWidthMeasureBlock(orientation)(
              measurables,
              height,
              arrangementSpacing.roundToPx()
          )

          override fun IntrinsicMeasureScope.minIntrinsicHeight(
              measurables: List<IntrinsicMeasurable>,
              width: Int
          ) = MinIntrinsicHeightMeasureBlock(orientation)(
              measurables,
              width,
              arrangementSpacing.roundToPx()
          )

          override fun IntrinsicMeasureScope.maxIntrinsicWidth(
              measurables: List<IntrinsicMeasurable>,
              height: Int
          ) = MaxIntrinsicWidthMeasureBlock(orientation)(
              measurables,
              height,
              arrangementSpacing.roundToPx()
          )

          override fun IntrinsicMeasureScope.maxIntrinsicHeight(
              measurables: List<IntrinsicMeasurable>,
              width: Int
          ) = MaxIntrinsicHeightMeasureBlock(orientation)(
              measurables,
              width,
              arrangementSpacing.roundToPx()
          )
      }
  }
```
</details>

<details>
<summary><b>rememberBoxMeasurePolicy</b></summary>

```kotlin
  @PublishedApi
  @Composable
  internal fun rememberBoxMeasurePolicy(
      alignment: Alignment,
      propagateMinConstraints: Boolean
  ) = if (alignment == Alignment.TopStart && !propagateMinConstraints) {
      DefaultBoxMeasurePolicy
  } else {
      remember(alignment, propagateMinConstraints) {
          boxMeasurePolicy(alignment, propagateMinConstraints)
      }
  }

  internal val DefaultBoxMeasurePolicy: MeasurePolicy = boxMeasurePolicy(Alignment.TopStart, false)

  internal fun boxMeasurePolicy(alignment: Alignment, propagateMinConstraints: Boolean) =
  MeasurePolicy { measurables, constraints ->
      if (measurables.isEmpty()) {
          return@MeasurePolicy layout(
              constraints.minWidth,
              constraints.minHeight
          ) {}
      }

      val contentConstraints = if (propagateMinConstraints) {
          constraints
      } else {
          constraints.copy(minWidth = 0, minHeight = 0)
      }

      if (measurables.size == 1) {
          val measurable = measurables[0]
          val boxWidth: Int
          val boxHeight: Int
          val placeable: Placeable
          if (!measurable.matchesParentSize) {
              placeable = measurable.measure(contentConstraints)
              boxWidth = max(constraints.minWidth, placeable.width)
              boxHeight = max(constraints.minHeight, placeable.height)
          } else {
              boxWidth = constraints.minWidth
              boxHeight = constraints.minHeight
              placeable = measurable.measure(
                  Constraints.fixed(constraints.minWidth, constraints.minHeight)
              )
          }
          return@MeasurePolicy layout(boxWidth, boxHeight) {
              placeInBox(placeable, measurable, layoutDirection, boxWidth, boxHeight, alignment)
          }
      }

      val placeables = arrayOfNulls<Placeable>(measurables.size)
      // First measure non match parent size children to get the size of the Box.
      var hasMatchParentSizeChildren = false
      var boxWidth = constraints.minWidth
      var boxHeight = constraints.minHeight
      measurables.fastForEachIndexed { index, measurable ->
          if (!measurable.matchesParentSize) {
              val placeable = measurable.measure(contentConstraints)
              placeables[index] = placeable
              boxWidth = max(boxWidth, placeable.width)
              boxHeight = max(boxHeight, placeable.height)
          } else {
              hasMatchParentSizeChildren = true
          }
      }

      // Now measure match parent size children, if any.
      if (hasMatchParentSizeChildren) {
          // The infinity check is needed for default intrinsic measurements.
          val matchParentSizeConstraints = Constraints(
              minWidth = if (boxWidth != Constraints.Infinity) boxWidth else 0,
              minHeight = if (boxHeight != Constraints.Infinity) boxHeight else 0,
              maxWidth = boxWidth,
              maxHeight = boxHeight
          )
          measurables.fastForEachIndexed { index, measurable ->
              if (measurable.matchesParentSize) {
                  placeables[index] = measurable.measure(matchParentSizeConstraints)
              }
          }
      }

      // Specify the size of the Box and position its children.
      layout(boxWidth, boxHeight) {
          placeables.forEachIndexed { index, placeable ->
              placeable as Placeable
              val measurable = measurables[index]
              placeInBox(placeable, measurable, layoutDirection, boxWidth, boxHeight, alignment)
          }
      }
  }

```

</details>

<br>

We will discuss about this in the future.


### Basic Layout : Creating a Greeting Card
In this section, we will take a look at how the three components can be used.

#### Adding Image

Lesson : [Add images to your Android app](https://developer.android.com/codelabs/basic-android-kotlin-compose-add-images?authuser=1&continue=https%3A%2F%2Fdeveloper.android.com%2Fcourses%2Fpathways%2Fandroid-basics-compose-unit-1-pathway-3%3Fauthuser%3D1%23codelab-https%3A%2F%2Fdeveloper.android.com%2Fcodelabs%2Fbasic-android-kotlin-compose-add-images#2)

In order to create an ImageView, we need to get the image from `/res`.

```kotlin
  val image = painterResource(R.drawable.androidparty)
```

`painterResource` is a composable function that returns a **Painter** from `androidx.compose.ui.graphics.painter` package. It can either be a **VectorPainter** or **BitmapPainter** depending on the resource file type :

```kotlin
  @Composable
  fun painterResource(@DrawableRes id: Int): Painter {
      val context = LocalContext.current
      val res = resources()
      val value = remember { TypedValue() }
      res.getValue(id, value, true)
      val path = value.string
      // Assume .xml suffix implies loading a VectorDrawable resource
      return if (path?.endsWith(".xml") == true) {
          val imageVector = loadVectorResource(context.theme, res, id, value.changingConfigurations)
          rememberVectorPainter(imageVector)
      } else {
          // Otherwise load the bitmap resource
          val imageBitmap = remember(path, id, context.theme) {
              loadImageBitmapResource(res, id)
          }
          BitmapPainter(imageBitmap)
      }
  }
```

However, a **Painter** is only stating that `image` is something that can be drawn. To display it, we need to use **Image** :

```kotlin
  @Composable
  fun Image(
      painter: Painter,
      contentDescription: String?,
      modifier: Modifier = Modifier,
      alignment: Alignment = Alignment.Center,
      contentScale: ContentScale = ContentScale.Fit,
      alpha: Float = DefaultAlpha,
      colorFilter: ColorFilter? = null
  )
```

By passing in the `image` and a `contentDescription` into **Image**, we can display the image on the preview :

```kotlin
  @Composable
  fun GreetingWithImage(message: String, from: String) {
      val painter = painterResource(R.drawable.androidparty)
      Image(painter = painter, contentDescription = "Party")
  }
```

<center>
<img src = "/images/posts/jekyll/jetpack_compose/image_default.png" style="width:80%"/>
</center>

<br>

><br>**Note** :
> Image resources can be retrieved using **`painterResource`**.
> String resources can be retrieved using **`stringResource`**
><br/>

#### Adding Captions via Box and Column

Knowing that a **Box** displays multiple components by stacking one on top another, we can also add text on top of the image by wrapping **Image** and **Column** inside a **Box**.

```kotlin
  @Composable
  fun GreetingWithImage(message: String, from: String) {
      val painter = painterResource(R.drawable.androidparty)
      Box() {
          Image(painter = painter, contentDescription = "Party")
          Column(Modifier.padding(bottom = 20.dp).align(Alignment.BottomCenter)) {
              Text(text = message, fontSize = 18.sp, style =
              TextStyle(
                  fontStyle = FontStyle.Italic,
                  fontWeight = FontWeight.ExtraBold))
              Text(text = from)
          }
      }
  }
```

<center>
<img src = "/images/posts/jekyll/jetpack_compose/camp2022/greeting_caption.png" style="width:80%"/>
</center>

<br>

By default, **Image** is showing the content as fit (`ContentScale.Fit`). In order to fill the whole page with image, we can simply set `contentScale` as `ContentScale.FillHeight` :


```kotlin
  Image(painter = painter,
      contentDescription = "Party",
      contentScale = ContentScale.FillHeight)
```

If we want to place image at the center and perform crop, we can combine the usage of **Modifier** and **ContentScale** to achieve the result :

```kotlin
  Image(painter = painter,
        contentDescription = "Party",
        modifier = Modifier.fillMaxHeight().fillMaxWidth(),
        contentScale = ContentScale.Crop)
```

#### Aligning Text

Through the UI seems alright, but let's make it better by aligning the `from` text to the right of the `message`.

To align the text, we can simply use a **Modifier** :

```kotlin
Text(text = from, Modifier.align(Alignment.End))
```

<center>
<img src = "/images/posts/jekyll/jetpack_compose/camp2022/greeting_caption_align_1.png" style="width:80%"/>
</center>

<br>

But, what if I want to align `from` at the center of `message` ?

Well, we just need to make the text fill the whole width followed by align to center, right ?

```kotlin
  Text(text = from, Modifier.fillMaxWidth()
                            .wrapContentWidth(Alignment
                            .CenterHorizontally))
```

Unfortunately, by applying `fillMaxWidth` causes the parent container to expand to match_parent as well :

<center>
<img src = "/images/posts/jekyll/jetpack_compose/camp2022/greeting_caption_fillMaxWidth.png" style="width:80%"/>
</center>

<br>

Here are some of the ways to solve it :

```kotlin
// 1. apply the same code to `message` textview
  Text(text = message, fontSize = 18.sp, style =
            TextStyle(
                fontStyle = FontStyle.Italic,
                fontWeight = FontWeight.ExtraBold),
                modifier = Modifier.fillMaxWidth()
                        .wrapContentWidth(Alignment
                        .CenterHorizontally))
  Text(text = from, Modifier.fillMaxWidth()
                            .wrapContentWidth(Alignment
                            .CenterHorizontally))

// 2. make Column fillMaxWidth and align the texts at the center
  Column(Modifier.padding(bottom = 20.dp).align(Alignment.BottomCenter).fillMaxWidth()) {
    Text(text = message,
         fontSize = 18.sp,
         style = TextStyle(
                  fontStyle = FontStyle.Italic,
                  fontWeight = FontWeight.ExtraBold),
                  modifier = Modifier.align(Alignment.CenterHorizontally))
              Text(text = from, Modifier.align(Alignment.CenterHorizontally))
          }

```

<center>
<img src = "/images/posts/jekyll/jetpack_compose/camp2022/greeting_caption_fillMaxWidth_center.png" style="width:80%"/>
</center>

<br>

### Practice : Compose Basic
After creating greeting card, we now have a better understanding on how Compose works.

Next, we can try to create [some basic UI](https://developer.android.com/codelabs/basic-android-kotlin-compose-composables-practice-problems?authuser=1&continue=https%3A%2F%2Fdeveloper.android.com%2Fcourses%2Fpathways%2Fandroid-basics-compose-unit-1-pathway-3%3Fauthuser%3D1%23codelab-https%3A%2F%2Fdeveloper.android.com%2Fcodelabs%2Fbasic-android-kotlin-compose-composables-practice-problems#1) just to see how much we've learned.

Here are something that you might need to know to complete these exercises :

- We can arrange the layout in **Column** and **Row** using **Arrangement**. We can even use `Density.arrange` method to create more complicated arrangement. (Note: Equal weight means developers define equal weights for all subcomponents )
<center>
<img src = "/images/posts/jekyll/jetpack_compose/camp2022/column_arrangement_visualization.gif" style="width:80%"/>
</center>

<br>

- We can also set the proportion of components in **Column** and **Row** using `weight` variable. For instance :

  <br>
  The following code will generate a view that is filled with blue color :

  ```kotlin
    Column(Modifier.background(Color.Blue)) {
      Row(Modifier.background(Color.Red)) {}
      Row(Modifier.background(Color.Yellow)) {}
    }
  ```

  To display the colors in **Row**s, we need to expand it using weight :

  ```kotlin
    Column(Modifier.background(Color.Blue)) {
      Row(Modifier.background(Color.Red)
                  .weight(1f)) {}
      Row(Modifier.background(Color.Yellow)
                  .weight(1f)) {}
    }
  ```

  By changing the proportion among the weight values, we can display other complex structures.
  <br>
  ><br> **Note**
  > `weight` is only accessible when the component is wrapped within a **Column** or **Row**.
  ><br/>

  One more thing, remember how we can use **Modifier** to generate paddings ?

  <br>If you think you can create **margin** using **Modifier**, then you won't find what you are looking for.
  <br>Instead, you can use **Padding** or **Spacer** :

  <br>

  ```kotlin
    Spacer(modifier = Modifier.height(16.dp))
  ```
  <br>**Spacer** is nothing but a composable that represents an empty space:

  <br>

  ```kotlin
    @Composable
    fun Spacer(modifier: Modifier) {
        Layout({}, modifier) { _, constraints ->
            with(constraints) {
                val width = if (hasFixedWidth) maxWidth else 0
                val height = if (hasFixedHeight) maxHeight else 0
                layout(width, height) {}
            }
        }
    }
  ```

  As for **Padding**, it can either be padding or margin depends on the order it is called by **Modifier**.

  <br>For Instance :
  <br>
  ```kotlin
    @Composable
    fun MarginWall() {
        Column() {
            Box(
                Modifier
                    .background(Color.Cyan)
                    .fillMaxWidth()
                    .height(100.dp)
                    .padding(10.dp)
                    .background(Color.LightGray)
                    .padding(start = 5.dp, top = 5.dp, end = 5.dp)){
                Text(text = "Inner Text", modifier = Modifier.fillMaxWidth().background(Color.Green), textAlign = TextAlign.Start)
            }
        }
    }
  ```

  The code from above will create a **Box** that holds a **Text** with a margin of 5dp at start, top and end :

  <br>
  <center>
  <img src = "/images/posts/jekyll/jetpack_compose/camp2022/margin_example.png" style="width:80%"/>
  </center>

  <br>


## Making Interactive App

A new concept introduced in **Jetpack Compose** is *state*.

><br>A <i>**state**</i> is any value in an app that can change overtime.
><br>

By default, a **Composable** is *stateless*, because they will not automatically remember state.

As a result, a **Composable** will reset to its initial value whenever a **Compose** reset the UI (**recompose**).

To make a component interact with the user, we need to think of some way to remember the *state* which leads **recomposition** to occur and updating the display.


The first method that we will see is the `remember` API.

### `remember`

```kotlin
  @Composable
  inline fun <T> remember(calculation: @DisallowComposableCalls () -> T): T =
      currentComposer.cache(false, calculation)
```

From the code, we can tell that it is used to store objects in the Composition.

><br>**However**, also note that when the composable that called `remember` is removed from the Composition, the cached objects **will be forgotten**.
><br><br/>

Here's an example of using `remember` through delegation :

```kotlin
  import androidx.compose.runtime.mutableStateOf
  import androidx.compose.runtime.remember
  // these are needed when using `by`
  // to delegate for a `var`
  import androidx.compose.runtime.getValue
  import androidx.compose.runtime.setValue
  var result by remember { mutableStateOf(1) }

  Image(
    painter = painterResource(
        when(result) {
            1 -> R.drawable.dice_1
            2 -> R.drawable.dice_2
            3 -> R.drawable.dice_3
            4 -> R.drawable.dice_4
            5 -> R.drawable.dice_5
            else -> R.drawable.dice_6
        }
    ), contentDescription = "$result"
  )
  // ...
  Button(onClick = { result = (1..6).random()
  }) {
    Text(stringResource(R.string.roll))
  }
```

This code displays a dice image and a button.
When the button is clicked, it will update `result` which leads to the update the value of the `MutableState`, `result`.

><br>**`mutableStateOf`** wraps the initial value in a **State** object, which makes the value observable, allowing **Compose** to observe  changes and trigger recomposition.
><br>And it has to be worked with `remember`, otherwise we will get an error saying  **Creating a state object during composition without using remember**
><br>

We've seen how `remember` is used on a local variable that is used in a single **Composable**.

However, there are times when we need to share state among other **Composable**s as well.

To do so, we need learn about the **State Hoisting**, a fancy word for **state injection**.

#### State Hoisting

><br>**State Hoisting** is the process of moving a *state* outside of a **Composable**.
><br><br/>

<center>
<img src = "/images/posts/jekyll/jetpack_compose/camp2022/screenshot_hoisting_state.png" style="width:80%"/>
</center>

<br>

**State hoisting** is needed in two situations :

1. When we want to share a *state* among composables.
2. When we wish to **reuse** the state.


To move a *state* out of composable we just need to turn the *state* and whatever is acting on it into paramters :

```kotlin
  @Composable
  fun EditNumberField() {
    var input by remember {mutableStateOf("")}

    TextField(value = input,
              onValueChange = {input = it})
  }

// after hoisting the state
@Composable
fun EditNumberField( input: String,
                     onValueChange: (String) -> Unit) {

  TextField(
      value = input,
      onValueChange = onValueChange)
}
```

This way, we moved the *state* **to its parent**.

And just a heads up, to remove keyboard and focus off from the keyboard, we need to use **LocalFocusManager** :

```kotlin
val focusManager = LocalFocusManager.current

TextField(
    ...
    keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Number, imeAction = ImeAction.Done),
    keyboardActions = KeyboardActions {
        focusManager.clearFocus(true)
    }
)
```

which is similar to the traditional way :

```kotlin
fun View.hideKeyboard() {
    val imm = context.getSystemService(Context.INPUT_METHOD_SERVICE) as InputMethodManager
    imm.hideSoftInputFromWindow(windowToken, 0)

    if (this is EditText) clearFocus()
}
```

and we also **need to set the parent view to be focusable**, because once the **EditText** is cleared of focus, window will try to locate the next view that it can focus on.

Here is a great video on [A Compose state of mind: Using Jetpack Compose's automatic state observation](https://www.youtube.com/watch?v=rmv2ug-wW4U). We will take a closer look inside this video in another article.

### Automatic Test
There are two types of Automatic Tests :

1. Local tests (Unit Test)
   - they are executed on a **Java virtual machine**
   - a type of automated test that directly test a **small piece of code** to ensure that it functions properly
<br>
2. Instrumentation tests (UI Test)
   - UI tests **launch an app or part of an app**, simulate user interactions, and check whether the app reacted appropriately
   - When you run an instrumentation test on Android, the test code is **actually built into its own Android Application Package (APK)** like a regular Android app
   - The test APK is installed on the device or emulator along with the regular app APK. The test APK then runs its tests against the app APK.

Writing a **Unit Test** is simple, all you need is the annotation `@Test` and make sure the function to be tested can be seen by the Class doing the Testings. This can be done by either making the function `public` or make it an `internal` with `@VisibleForTesting` at the top.

For example :
```kotlin
  class TipCalculatorTests {
      @Test
      fun calculate_20_percent_tip_no_roundup() {
          val amount = 10.00
          val tipPercent = 20.00
          val expectedTip = "$2.00"
          val actualTip = calculateTip(amount = amount, tipPercent = tipPercent, false)
          assertEquals(expectedTip, actualTip)
      }
  }
```

Creating a **UI Test** is a bit different form Unit Test.

In a View-based UI Testing, we have **[Espresso](https://developer.android.com/training/testing/espresso)** framework to help us. And since a View is a well defined object with specific properties, we can create an UI Test as follows :

```java
  @Test
  public void greeterSaysHello() {
      onView(withId(R.id.name_field)).perform(typeText("Steve"));
      onView(withId(R.id.greet_button)).perform(click());
      onView(withText("Hello Steve!")).check(matches(isDisplayed()));
  }
```

However, unlike View-based UI, in Compose only some composables emit UI into the UI hierarchy, therefore a different approach to matching UI elements is needed. [[link](https://developer.android.com/jetpack/compose/testing?authuser=1)]

In Compose UI Test, we need to define a **ComposeContentTestRule**, as shown below :

```kotlin
  class TipUiTest {
      @get:Rule
      val composeTestRule = createComposeRule()
  }
```

A **ComposeContentTestRule** allows us to set content without the necessity to provide a host (Activty) for the content :

```kotlin
interface ComposeContentTestRule : ComposeTestRule {
    fun setContent(composable: @Composable () -> Unit)
}
```

Here is how we use it :

```kotlin
@Test
fun calculate_20_percent_tip() {
  composeTestRule.setContent {
      TipCalculatorTheme {
          CustomTipTimeScreen()
      }
  }
}
```

Once the content is prepared, we can find composables we need and perform actions on them :

```kotlin
@Test
fun calculate_20_percent_tip() {
    composeTestRule.setContent {
        TipCalculatorTheme {
            CustomTipTimeScreen()
        }
    }
    composeTestRule.onNodeWithText("Bill Amount")
        .performTextInput("10")
    composeTestRule.onNodeWithText("Tip (%)").performTextInput("20")
    composeTestRule.onNodeWithText("Tip Amount: $2.00").assertExists()
}
```

If we want to see the structure of the content, we can simply print out all the composables from `Root` :

```kotlin
@Test
fun listAll() {
    composeTestRule.setContent {
        TipCalculatorTheme {
            CustomTipTimeScreen()
        }
    }
    composeTestRule.onRoot().printToLog("TAG")
}
```

With this structure :

<center>
<img src = "/images/posts/jekyll/jetpack_compose/camp2022/customize_tip_cal.png" style="width:80%"/>
</center>

<br>

it will print the followings :

```
12-09 22:12:17.806 25179 25197 D TAG     : printToLog:
12-09 22:12:17.806 25179 25197 D TAG     : Printing with useUnmergedTree = 'false'
12-09 22:12:17.806 25179 25197 D TAG     : Node #1 at (l=0.0, t=88.0, r=1440.0, b=1441.0)px
12-09 22:12:17.806 25179 25197 D TAG     :  |-Node #2 at (l=477.0, t=200.0, r=964.0, b=312.0)px
12-09 22:12:17.806 25179 25197 D TAG     :  | Text = '[Calculate Tip]'
12-09 22:12:17.806 25179 25197 D TAG     :  | Actions = [GetTextLayoutResult]
12-09 22:12:17.806 25179 25197 D TAG     :  |-Node #3 at (l=112.0, t=424.0, r=1328.0, b=620.0)px
12-09 22:12:17.806 25179 25197 D TAG     :  | ImeAction = 'Next'
12-09 22:12:17.806 25179 25197 D TAG     :  | EditableText = ''
12-09 22:12:17.806 25179 25197 D TAG     :  | TextSelectionRange = 'TextRange(0, 0)'
12-09 22:12:17.806 25179 25197 D TAG     :  | Focused = 'false'
12-09 22:12:17.806 25179 25197 D TAG     :  | Text = '[Bill Amount]'
12-09 22:12:17.806 25179 25197 D TAG     :  | Actions = [GetTextLayoutResult, SetText, SetSelection, OnClick, OnLongClick, PasteText]
12-09 22:12:17.806 25179 25197 D TAG     :  | MergeDescendants = 'true'
12-09 22:12:17.806 25179 25197 D TAG     :  |-Node #7 at (l=112.0, t=648.0, r=1328.0, b=844.0)px
12-09 22:12:17.806 25179 25197 D TAG     :  | ImeAction = 'Done'
12-09 22:12:17.806 25179 25197 D TAG     :  | EditableText = ''
12-09 22:12:17.806 25179 25197 D TAG     :  | TextSelectionRange = 'TextRange(0, 0)'
12-09 22:12:17.806 25179 25197 D TAG     :  | Focused = 'false'
12-09 22:12:17.806 25179 25197 D TAG     :  | Text = '[Tip (%)]'
12-09 22:12:17.806 25179 25197 D TAG     :  | Actions = [GetTextLayoutResult, SetText, SetSelection, OnClick, OnLongClick, PasteText]
12-09 22:12:17.806 25179 25197 D TAG     :  | MergeDescendants = 'true'
12-09 22:12:17.806 25179 25197 D TAG     :  |-Node #11 at (l=112.0, t=974.0, r=465.0, b=1050.0)px
12-09 22:12:17.806 25179 25197 D TAG     :  | Text = '[Round up tip ?]'
12-09 22:12:17.806 25179 25197 D TAG     :  | Actions = [GetTextLayoutResult]
12-09 22:12:17.806 25179 25197 D TAG     :  |-Node #12 at (l=1178.0, t=970.0, r=1311.0, b=1054.0)px
12-09 22:12:17.806 25179 25197 D TAG     :  | Role = 'Switch'
12-09 22:12:17.806 25179 25197 D TAG     :  | ToggleableState = 'Off'
12-09 22:12:17.806 25179 25197 D TAG     :  | Focused = 'false'
12-09 22:12:17.806 25179 25197 D TAG     :  | Actions = [OnClick]
12-09 22:12:17.806 25179 25197 D TAG     :  | MergeDescendants = 'true'
12-09 22:12:17.806 25179 25197 D TAG     :  |-Node #14 at (l=429.0, t=1236.0, r=1012.0, b=1329.0)px
12-09 22:12:17.806 25179 25197 D TAG     :    Text = '[Tip Amount: $0.00]'
12-09 22:12:17.806 25179 25197 D TAG     :    Actions = [GetTextLayoutResult]
```

You can find Testing Cheat Sheet on this [page](https://developer.android.com/jetpack/compose/testing-cheatsheet?authuser=1).

### Practice : Create an Art Space app
You can find the practice at this [link](https://developer.android.com/codelabs/basic-android-kotlin-compose-art-space?authuser=1&continue=https%3A%2F%2Fdeveloper.android.com%2Fcourses%2Fpathways%2Fandroid-basics-compose-unit-2-pathway-3%3Fauthuser%3D1%23codelab-https%3A%2F%2Fdeveloper.android.com%2Fcodelabs%2Fbasic-android-kotlin-compose-art-space#0)

In order for you to fully complete this exercise, you will need to do some diggings on these topics :

1. **Gesture**
   A gesture can be detected in several different ways, including : **pointerInput**, **pointerInteropFilter** and other experimental API such as **swipeable**.

2. **SubcomposeLayout**
   This API is used when you wish to get the size of certain composable and use it to layout another composable based on the size of the first one.

Since it took me hours of digging to finally get things straight, here is my [source code](https://github.com/KuoPingL/ArtSpaceCompose) for you when you find yourself hitting a brick of wall.

Of course, if you have other alternative solutions or questions, feel free to share.


## Unit 3 : Displaying Scrollable List

In traditional Android, to create a RecyclerView, we need to ...


However, when we are using Compose, we simply need `LazyColumn` :

```kotlin
@Composable
fun LazyColumn(
    modifier: Modifier = Modifier,
    state: LazyListState = rememberLazyListState(),
    contentPadding: PaddingValues = PaddingValues(0.dp),
    reverseLayout: Boolean = false,
    verticalArrangement: Arrangement.Vertical =
        if (!reverseLayout) Arrangement.Top else Arrangement.Bottom,
    horizontalAlignment: Alignment.Horizontal = Alignment.Start,
    flingBehavior: FlingBehavior = ScrollableDefaults.flingBehavior(),
    content: LazyListScope.() -> Unit
) 
```



### How does MutableState works ?




3. In order to add an `onClickListener` in compose, instead of using a **Button**, we can simply use **`Modifier.clickable`** :

  ```kotlin
    Image(paint,
          contentDescription = str,
          modifier = Modifier.border(1.dp, Color(105, 205, 216))
                             .clickable {
          when(step) {
              2 -> {
                  if (rand == 0) step ++
                  else rand--
              }
              4 -> {
                  step = 1
              }
              else -> step++
          }
      })

  ```

4. To create an Random Value with a specific range of values, simply do :

  ```kotlin
    val rand = (1..100).random()
  ```

More on [State and Jetpack Compose](https://developer.android.com/jetpack/compose/state?authuser=1) in the future.





## Naming a Composable Function
[ref](https://github.com/androidx/androidx/blob/androidx-main/compose/docs/compose-api-guidelines.md#naming-unit-composable-functions-as-entities)

- MUST be a noun: DoneButton()
- NOT a verb or verb phrase: DrawTextField()
- NOT a nouned preposition: TextFieldWithLink()
- NOT an adjective: Bright()
- NOT an adverb: Outside()
- Nouns MAY be prefixed by descriptive adjectives: RoundIcon()




## Reference
1. [Declarative vs imperative](https://dev.to/ruizb/declarative-vs-imperative-4a7l)
2. [Thinking in Compose](https://developer.android.com/jetpack/compose/mental-model)







<br><br><br><br><br><br><br><br><br><br><br>
