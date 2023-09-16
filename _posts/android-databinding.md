---
layout: post
title: Android - Data Binding
categories: [Android]
use_math: true
keywords: android, databinding, jetpack
---

# 概要
**Date Binding** 是 Android Jetpack 中的一個函數庫。其主要功能就是建立 XML 與 Data 之間的互動橋樑。

><br>
>並不適用於 <b>Compose</b>
><br><br>

而這個橋樑，我們可以通過不同的方式來連接，包括通過以下方式：
- **Layout Expression**
- **Binding Adapter**
- **Two Ways Binding**

我們都將一一示範與說明。

# Layout Expression

Layout Expression 其實就是通過 **One Way Binding** 來對 UI 進行更新。

最簡單的 **One Way Binding** 便是物件或方法的連接。
當然，我們還可以通過不同的 **運算子** 來得到較為複雜的結果。

這裡提到的運算子有以下 [ [資料來源](https://proandroiddev.com/android-data-binding-layout-expressions-69ee6a0a3d2a) ]：

- Mathematical `+ - / * %`
- String concatenation `+`
- Logical `&& ||`
- Binary `& | ^`
- Unary `+ - ! ~`
- Shift `>> >>> <<`
- Comparison `== > < >= <=` (Note that `<` needs to be escaped as `&lt;`)
- `instanceof`
- Grouping `()`
- Literals — character, String, numeric, `null`
- Cast
- Method calls
- Field access
- Array access `[]`
- Ternary operator `?:`

接下來，我們就看看這些運算子的範例吧。

## 範例

在正式 App 開發中，一般來說我們會將 **ViewModel** 放入 XML 中來進行 Data Binding。 為了展示 Data Binding，我們在此就使用 **登入頁面** 來當作範例吧。

首先，我們先準備 **ViewModel** 吧：

```kotlin

class LayoutExpressionViewModel: ViewModel() {

  private val _username = MutableLiveData<String>()
  val username: LiveData<String>
      get() = _username

  private val _password = MutableLiveData<String>()
  val password: LiveData<String>
      get() = _password

  companion object {
      val Factory: ViewModelProvider.Factory = object: ViewModelProvider.Factory {
          override fun <T : ViewModel> create(modelClass: Class<T>): T {
              return with(modelClass) {
                  when {
                      isAssignableFrom(LayoutExpressionViewModel::class.java) -> LayoutExpressionViewModel()
                      else -> throw IllegalArgumentException(
                          "Unknown ViewModel class: ${modelClass.name}")
                  }
              } as T
          }
      }
  }
}

```

而我們的 UI 也是很單純的只有兩個 **EditText** 以及數個提示 **TextView** 。

<center>
<img src = "/images/posts/jekyll/android/databinding/login_page_0.png" style="width:40%"/>
</center>

<details>
<summary>XML</summary>

```groovy
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">


    <androidx.appcompat.widget.AppCompatImageView
            android:id="@+id/img"
            android:layout_width="0dp"
            android:layout_height="0dp"
            app:layout_constraintTop_toTopOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintDimensionRatio="1"
            app:layout_constraintBottom_toTopOf="@id/tv_feedback"
            android:layout_marginTop="20dp"
            android:layout_marginBottom="20dp"
            android:src="@drawable/baseline_face_4_24" />

        <androidx.appcompat.widget.AppCompatTextView
            android:id="@+id/tv_feedback"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            app:layout_constraintTop_toBottomOf="@id/img"
            app:layout_constraintBottom_toTopOf="@id/spacer"
            app:layout_constraintVertical_bias="1"
            android:layout_marginTop="8dp"
            android:paddingTop="8dp"
            android:paddingBottom="8dp"
            android:paddingStart="20dp"
            android:paddingEnd="20dp"
            android:textSize="16sp"
            android:textStyle="bold"
            android:textColor="@color/black"
            android:text="Welcome" />

        <View
            android:id="@+id/spacer"
            android:layout_width="match_parent"
            android:layout_height="1dp"
            android:background="@color/black"
            app:layout_constraintBottom_toTopOf="@id/layout_form"
            android:layout_marginStart="20dp"
            android:layout_marginEnd="20dp"
            android:layout_marginBottom="20dp"/>

        <androidx.appcompat.widget.LinearLayoutCompat
            android:id="@+id/layout_form"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            app:layout_constraintBottom_toTopOf="@id/btn_login"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            android:layout_marginTop="8dp"
            android:layout_marginBottom="40dp"
            android:orientation="vertical"
            android:paddingStart="20dp"
            android:paddingEnd="20dp">

            <androidx.appcompat.widget.AppCompatTextView
                android:id="@+id/tv_name"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:lines="1"
                android:textSize="20sp"
                android:text="User Name"
                android:textStyle="bold" />

            <androidx.appcompat.widget.AppCompatEditText
                android:id="@+id/edit_name"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:hint="Type your name here"
                android:textSize="20sp"
                android:textStyle="bold"
                android:textColorHint="#999" />

            <View
                android:layout_width="match_parent"
                android:layout_height="10dp" />

            <androidx.appcompat.widget.AppCompatTextView
                android:id="@+id/tv_password"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:lines="1"
                android:textSize="20sp"
                android:text="Password"
                android:textStyle="bold" />

            <androidx.appcompat.widget.AppCompatEditText
                android:id="@+id/edit_password"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:hint="Type your password here"
                android:inputType="textPassword"
                android:textSize="20sp"
                android:textStyle="bold"
                android:textColorHint="#999"
                tools:text="Test"/>

            <View
                android:layout_width="match_parent"
                android:layout_height="10dp" />

        </androidx.appcompat.widget.LinearLayoutCompat>

        <androidx.appcompat.widget.AppCompatButton
            android:id ="@+id/btn_login"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            android:layout_marginStart="20dp"
            android:layout_marginEnd="20dp"
            android:layout_marginBottom="40dp"
            android:textSize="20sp"
            android:text="Login"
            app:layout_constraintBottom_toBottomOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

</details>

<br>

另外，別忘記我們需要將 XML 包裹在 `layout` 標籤中。

><br>
> 若我們只使用 <b>ViewBinding</b>，那這一步則不需要。
> <br><br>

在 Mac 我們通過 `option + enter` 就可以叫喚出以下視窗。

<center>
<img src ="/images/posts/jekyll/android/databinding/add_layer_tag_in_xml.png"/>
</center>

<br>

點選 **Convert to data binding layout** 就會將原本的 XML 包裹在 `<layout>` 中：

```groovy
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>

    </data>

    <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <!-- 我們的 UI -->
    </androidx.constraintlayout.widget.ConstraintLayout>
</layout>
```


### Casting
一切就緒了，我們便可將 **ViewModel** *cast* 入 XML 中：

```groovy
<data>
    <import type="com.anushka.bindingdemo1.LayoutExpressionViewModel"/>
    <variable
        name="vm"
        type="LayoutExpressionViewModel" />
</data>
```

但因為我們已經需要用到 `vm` 這物件，所以我們可以將兩者合二為一：

```groovy
<data>
    <variable
        name="vm"
        type="com.anushka.bindingdemo1.LayoutExpressionViewModel" />
</data>
```

除了 **ViewModel**，我們另一個最常需要 `import` 的就是 **View** 了。 因為我們有時候會通過參數來判斷 View 是否應該顯示：

```groovy
<data>
  <import type="android.view.View"/>
  <variable
      name="vm"
      type="com.anushka.bindingdemo1.LayoutExpressionViewModel" />
</data>

<androidx.constraintlayout.widget.ConstraintLayout
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <View
      android:layout_width="match_parent"
      android:layout_height="match_parent"
      android:visibility="@{vm.username == null ? View.GONE : View.VISIBLE}"
    />

</androidx.constraintlayout.widget.ConstraintLayout>

```


### Field Access
接下來，我們希望在使用者輸入 user name 的同時在 `tv_feedback` 中顯示回應。

所以我們先將 `edit_name` 與 ViewModel 中的 _username 進行連結：


```groovy
<androidx.appcompat.widget.AppCompatEditText
        android:id="@+id/edit_name"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="Type your name here"

        android:text="@{vm.username}"

        android:textSize="20sp"
        android:textStyle="bold"
        android:textColorHint="#999" />
```

當然，這還不夠。 因為 DataBinding 需要知道什麼時候才進行更新。 而這就取決於 **生命週期** 了。

所以我們需要設定 DataBinding 的 `lifecycleOwner` 。

```kotlin

// Activty 中
binding.lifecycleOwner = this

// Fragment 中我們只需要 Fragment 更新畫面的生命週期
binding.lifecycleOwner = viewLifecycleOwner

```

最後我們還需要將 **ViewModel** 傳如 `binding` 中：

```kotlin
viewModel = LayoutExpressionViewModel.Factory.create(LayoutExpressionViewModel::class.java)
binding.vm = viewModel
```
如此一來 UI 就會監聽著 `username` 了。當然，我們還需要從 ViewModel 中更新 `username`：

```kotlin
// ViewModel 中新增方法
fun setUsername(name: String) {
  _username.value = name
}

// Activity 中調用此方法
viewModel.setUsername("Joe")

```

如此一來，當我們進入此頁面時就會看到更新後的 UI。

<center>
<img src = "/images/posts/jekyll/android/databinding/login_page_username.png" style="width:40%"/>
</center>

<br>


接下來，如果我們想讓歡迎字串顯示 "Welcome {username}" 的話，那該怎麼做呢？ 第一個想到的作法應該會是：

```groovy
android:text = "Welcome + @{vm.username}"
```

但 <span style="color:#f00">這是錯的</span>。

我們其實有幾種作法：
1. 固定的字串
   ```groovy
   android:text = "@{`Welcome` + vm.username}"
   ```
2. 資料中的字串
   ```groovy
   android:text = "@{vm.username + vm.username}"
   ```
3. 字串格式
   ```groovy
   // strings.xml (en)
   <string name = "welcome">Welcome</string>
   // strings.xml (es)
   <string name = "welcome">Bienvenida</string>
   // strings.xml
   <string name = "msg_wel">%s %s</string>

   // ui
   android:text = "@{@strings/msg_wel(@strings/welcome, vm.username)}"
   ```

如果我們想要把字串與數字連接在一起，我們也可以使用字串格式來達成。或者使用 `String.valueOf()` 來進行轉換：

```groovy
android:text = "@{String.valueOf(user.id)}"
```




### 方法
以上說了如何使用 DataBinding 搭配著 `TextChangedListener` 來更新參數值並顯示在畫面上。 接下來，我們要看的是 DataBinding 如何通過 **Button** 與使用者進行基本的互動。

我們就做一個簡單的 `submit` 按鈕。當使用者按下 **Button** 後就會做出反應。

當我們希望在 Button 被按下時進行某種方法，我們有幾種方法：

1. 通過 `setOnClickListener` 來建立按下時的反應：
   <br>
   ```kotlin
   _binding.btnLogin.setOnClickListener { /*Do Something*/ }
   ```
2. 在 XML 中定義 `onClick` 所調用的方法：
   <br>
   ```groovy
   android:onClick="doSomething"
   ```
   並在對應的 Activity 與 Fragment 中建立此法：
   ```kotlin
   fun doSomething(view: View) {}
   ```
3. 搭配 DataBinding 來調用 **ViewModel** 中的方法：
   首先，我們在 **ViewModel** 中定義一個方法：
   ```kotlin
   fun action() {
     Log.d(TAG, "Button Pressed")
   }
   ```

   然後我們就可以通過 XML 中的 Button 的 `android:onClick` 的行為調用 `action` 來調用這個方法：

   ```groovy
   android:onClick="@{() -> vm.action()}"
   ```

這裡我們來看看 `onClick` 做了些什麼吧。
`onClick` 所需要的是 **String**，而這個字串會在 **View** 的創建時讀取。 讀取時的源碼如下：

```java
case R.styleable.View_onClick:
    if (context.isRestricted()) {
        throw new IllegalStateException("The android:onClick attribute cannot "
                + "be used within a restricted context");
    }

    final String handlerName = a.getString(attr);
    if (handlerName != null) {
        setOnClickListener(new DeclaredOnClickListener(this, handlerName));
    }
    break;
```

當 **DeclaredOnClickListener** 的 `onClick` 被調用時，就會通過 `resolveMethod` 來找到對應的方法。 最後就會通過 `mResolvedMethod.invoke` 來調用指定的方法。

雖然 DataBinding 也是通過 `onClick` 指定方法的調用，但實作其實與上述的不同。

當我們使用 DataBinding 時， Android 會動態地未我們創建出對應的行為。 而 `onClick = @{() -> vm.action()}` 其實會為我們創建一個 **OnClickListener**，然後將我們定義的行為變成 Listener 的實作來調用。

#### 多個參數
那如果我們想要在按下按鈕時，將使用者名稱或密碼傳入呢？

我們自然可以按照 [Field Access](#field-access) 那樣寫：
```kotlin
// ViewModel
fun action(name: String) {
    Log.d(TAG, "Button Pressed $name")
}

// XML
android:onClick="@{() -> vm.action(vm.username ?? `NO NAME`)}"

```

以上就是幾個簡單的範例，其他的 Operator 如果有用到，我之後會再補充。

# Two Way Binding

Two Way Binding 其實就是讓我們可以只通過 XML 來進行資料的更新。 我們接下來看看要如何做到。

## Field

目前當我們輸入 username 時都是通過 `TextChangedListener` 來更新 `vm.username`，但這並非最終方法。

```kotlin
viewModel.username = "Joe"
```

如果希望自動更新的話，那我們需要先把 `username` 從 **LiveData** 改成 **MutableLiveData** ：

```kotlin
val username = MutableLiveData<String>()
```

再將 XML 中的 **EditText** 的 `text` 改為 ：

```groovy
android:text="@={vm.username}"
```

如此即可移除 `TextChangedListener`。


除此之外，我們也可能會遇到以下情況：

1. 若目標字串存在於舉列之中，我們可以通過 `[ ]` 來得到：
    ```groovy
    android:text = "@{`Welcome` + vm.usernames[0]}"
    ```
2. 那如果 `username` 是 **null** 呢？ 此時我們便可以使用 `??` 或 `?:` 來判斷 `username` 的狀態。
   <br>
   ```groovy
   android:text = "@{`Welcome` + vm.username ?? "" }"

   <!-- 或是 -->

   android:text = "@{`Welcome` + vm.username == null ? `` : vm.username }"
   ```
3. 如果我們使用的參數並非字串呢？ 也許是 `user.id: Int`。 <br>
   此時若我們照 `username` 那樣傳入 `android:text`，那我們就會出現 **Int 無法轉換成 String** 的錯誤。
   <br>所以我們可以通過字串格式來傳入，但也可以通過 [Data Binding Adapter](#data-binding-adapter) 來進行型別的轉換。

那如果我們不想要使用 **LiveData** 呢？ 是否也可以進行更新呢？ 這是一定的，但我們需要使用 **Bindable** 並與 **BaseObservable** 進行配合才行。

主要原因是因為 **LiveData** 可ㄧ通過 ViewBinding 來創建 **LiveDataListener** 來監聽 LiveData 的改變。

### 不使用 ViewModel 與 LiveData 的情況

如果我們不使用 **LiveData** 與 **ViewModel**，我們其實依舊可以使用一般類別來得到相同結果。

首先，我們先創建一個 **User** 類別：
```kotlin
class User {
  var password: String
  var name: String
}
```

並將 XML 中的 `data` 加入：
```groovy
<variable
    name="user"
    type="com.anushka.bindingdemo1.User" />
```

以及將 `user` 運用在 XML 中：
```groovy
<androidx.appcompat.widget.AppCompatEditText
    android:id="@+id/edit_name"
    ...
    android:text="@={user.name}"
/>

<androidx.appcompat.widget.AppCompatTextView
            android:id="@+id/tv_feedback"
            ...
            android:text="@{@string/msg_wel(@string/welcome, userpassword.name == null || userpassword.name == `` ? `Mr/Mrs`:userpassword.name)}" />
...

<androidx.appcompat.widget.AppCompatEditText
    android:id="@+id/edit_password"
    ...
    android:text="@={user.password}"
/>

<androidx.appcompat.widget.AppCompatButton
    android:id ="@+id/btn_login"
    ...
    android:onClick="@{() -> vm.submit(user.name, user.password)}"
/>
```

最後在 **Activty** 中將 **User** 傳入：
```kotlin
_binding.user = User().apply { password = "TEST" }
```

如此一來，畫面出來時就會看到密碼欄位已被填入 `Password`。而且在輸入完字串後，通過點擊 `submit` 按鈕，我們可以看見其實資料還是有更新。

但我們會發現原本該一起更新的 **TextView** `tv_feedback` 並沒有隨著 `user.name` 的更新而更新。這是因為每次更新後，`user.name` 並沒有進行通知。

為了要有自動更新的能力，**User** 必須要能被監聽與通知。所以我們需要使用 **BaseObservable** ：

```java
public class BaseObservable implements Observable
```

**BaseObservable** 會提供以下方法來進行監聽：

```java
public void addOnPropertyChangedCallback(@NonNull OnPropertyChangedCallback callback)
public void removeOnPropertyChangedCallback(@NonNull OnPropertyChangedCallback callback)
public void notifyChange() // 通知 callbacks
```

我們需要讓 **User** 繼承 **BaseObservable** ：

```kotlin
class User : BaseObservable() {
  var password: String = ""
}
```

並使用 **Bindable** annotation 來讓 **BaseObservable** 知道該監聽哪些參數：

```kotlin
class User(): BaseObservable() {

    var password: String = ""

    @get: Bindable
    var name: String = ""
        set(value) {
            field = value
            notifyPropertyChanged(BR.name)
            // 如果希望全部資料都更改，那可以調用 notifiedChange()
        }
}
```

其中的 `BR` 是由 **DataBinding** 自動創建的類別，其中包含了 DataBinding 所使用的參數：

```java
public class BR {
  public static final int _all = 0;

  public static final int name = 1;

  public static final int student = 2;

  public static final int user = 3;

  public static final int vm = 4;
}

```

><br>
>
>現在跑起來你會發現 ... 出現錯誤訊息
><b style="color:red">Unresolved reference: BR</b>
><br>

[解決方法](https://stackoverflow.com/questions/50956111/unresolved-reference-br-android-studio) 便是在 `build.gradle(app)` 中加上：

```groovy
plugins {
    /* ... */
    id 'kotlin-kapt'
}
```

然後如果遇到 `'compileDebugJavaWithJavac' task (current target is 1.8) and 'kaptGenerateStubsDebugKotlin' task (current target is 17) jvm target compatibility should be set to the same Java version.` 那就需要將 Java 版本統一一下：

```groovy
compileOptions {
    // 從 1_8 改為 17
    sourceCompatibility JavaVersion.VERSION_17
    targetCompatibility JavaVersion.VERSION_17
}
kotlinOptions {
    // 從 1.8 改為 17
    jvmTarget = '17'
}
```

最後，我們只需要更改 XML 的 **EditText** 部分為：
```groovy
android:text="@={userpassword.password}"
```

這樣就完成了。

# 高級用法

以上就是 DataBinding 的簡單用法。 接下來我們要看看如何進行客製化的作法。

客製化需要通過以下三種 **annotations** ：
- **BindingMethod** / **InverseBindingMethod**
- **BindingAdapter** / **InverseBindingAdapter**
- **BindingConversion** / **InverseBindingConversion**

## Demo 設定
這次我們就來使用 Google 提供的 [範例](https://github.com/android/databinding-samples/tree/main/TwoWaySample) 來學習吧。

首先，我們建立以下 UI ：

<center>
<img src = "/images/posts/jekyll/android/databinding/advanced_binding_ui.png" style="width:70%"/>
</center>

<br>

<details>
<summary>XML 源碼</summary>

```groovy
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>

    </data>

    <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:focusable="true"
        android:focusableInTouchMode="true">

        <ToggleButton
            android:id="@+id/startPause"
            android:layout_width="wrap_content"
            android:layout_height="42dp"
            android:layout_marginBottom="4dp"
            android:layout_marginTop="4dp"
            android:textOff="@string/start"
            android:textOn="@string/pause"
            android:focusable="true"
            app:layout_constraintBottom_toTopOf="@+id/displayWorkTimeLeft"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintHorizontal_bias="1"
            app:layout_constraintStart_toEndOf="@+id/stop"
            app:layout_constraintTop_toTopOf="parent"/>

        <Button
            android:id="@+id/stop"
            android:layout_width="wrap_content"
            android:layout_height="42dp"
            android:layout_marginBottom="4dp"
            android:layout_marginStart="8dp"
            android:layout_marginTop="4dp"
            android:text="@string/stop"
            app:layout_constraintBottom_toTopOf="@+id/displayWorkTimeLeft"
            app:layout_constraintEnd_toStartOf="@+id/startPause"
            app:layout_constraintHorizontal_bias="1"
            app:layout_constraintHorizontal_chainStyle="packed"
            app:layout_constraintStart_toEndOf="@+id/setsIncrease"
            app:layout_constraintTop_toTopOf="parent"/>

        <androidx.appcompat.widget.AppCompatTextView
            android:id="@+id/displayWorkTimeLeft"
            android:layout_width="0dp"
            android:layout_height="0dp"
            android:layout_marginEnd="8dp"
            android:layout_marginStart="8dp"
            android:gravity="center"
            android:maxLines="1"
            android:textAlignment="center"
            android:textColor="@color/secondaryDarkColor"
            app:autoSizeTextType="uniform"
            app:layout_constraintBottom_toTopOf="@+id/setWorkTime"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintHorizontal_bias="1.0"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toBottomOf="@+id/startPause"
            app:layout_constraintVertical_bias="0.0"
            app:layout_constraintVertical_chainStyle="spread_inside"
            tools:text="15:55"/>

        <androidx.appcompat.widget.AppCompatTextView
            android:id="@+id/displayRestTimeLeft"
            android:layout_width="0dp"
            android:layout_height="0dp"
            android:layout_marginEnd="8dp"
            android:layout_marginStart="8dp"
            android:textColor="@color/secondaryDarkColor"
            android:gravity="center"
            android:maxLines="1"
            android:textAlignment="center"
            app:autoSizeTextType="uniform"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintBottom_toTopOf="@+id/setRestTime"
            app:layout_constraintTop_toBottomOf="@+id/workoutBar"
            tools:text="5:55"/>

        <EditText
            android:id="@+id/setWorkTime"
            android:layout_width="64dp"
            android:layout_height="wrap_content"
            android:layout_marginTop="0dp"
            android:layout_marginBottom="0dp"
            android:digits=",.:0123456789"
            android:imeOptions="actionDone"
            android:inputType="time"
            android:maxLines="1"
            android:textAlignment="center"
            tools:text="15:34"
            app:layout_constraintBottom_toTopOf="@+id/workoutBar"
            app:layout_constraintTop_toBottomOf="@+id/displayWorkTimeLeft"
            app:layout_constraintStart_toEndOf="@+id/workminus"
            app:layout_constraintEnd_toStartOf="@+id/workplus"/>

        <EditText
            android:id="@+id/setRestTime"
            android:layout_width="64dp"
            android:layout_height="wrap_content"
            android:layout_marginEnd="8dp"
            android:layout_marginTop="0dp"
            android:layout_marginBottom="0dp"
            android:digits=",.:0123456789"
            android:ems="10"
            android:imeOptions="actionDone"
            android:inputType="time"
            android:maxLines="1"
            android:textAlignment="center"
            tools:text="15:50"
            app:layout_constraintBottom_toTopOf="@+id/restBar"
            app:layout_constraintTop_toBottomOf="@+id/displayRestTimeLeft"
            app:layout_constraintStart_toEndOf="@+id/restminus"
            app:layout_constraintEnd_toStartOf="@+id/restplus"
            />

        <Button
            android:id="@+id/workplus"
            android:layout_width="42dp"
            android:layout_height="42dp"
            android:layout_marginEnd="8dp"
            android:layout_marginStart="8dp"
            android:text="@string/plus_sign"
            app:layout_constraintEnd_toEndOf="@+id/workoutBar"
            app:layout_constraintStart_toEndOf="@+id/setWorkTime"
            app:layout_constraintBottom_toTopOf="@+id/workoutBar"
            app:layout_constraintTop_toBottomOf="@+id/displayWorkTimeLeft"
            app:layout_constraintHorizontal_chainStyle="packed"/>

        <Button
            android:id="@+id/workminus"
            android:layout_width="42dp"
            android:layout_height="42dp"
            android:layout_marginEnd="8dp"
            android:text="@string/minus_sign"
            app:layout_constraintBottom_toTopOf="@+id/workoutBar"
            app:layout_constraintEnd_toStartOf="@+id/setWorkTime"
            app:layout_constraintHorizontal_chainStyle="packed"
            app:layout_constraintStart_toStartOf="@id/workoutBar"
            app:layout_constraintTop_toBottomOf="@+id/displayWorkTimeLeft"/>

        <Button
            android:id="@+id/restplus"
            android:layout_width="42dp"
            android:layout_height="42dp"
            android:layout_marginEnd="8dp"
            android:layout_marginTop="0dp"
            android:layout_marginBottom="0dp"
            android:text="@string/plus_sign"
            app:layout_constraintEnd_toEndOf="@+id/restBar"
            app:layout_constraintStart_toEndOf="@+id/setRestTime"
            app:layout_constraintBottom_toTopOf="@+id/restBar"
            app:layout_constraintHorizontal_chainStyle="packed"
            app:layout_constraintTop_toBottomOf="@+id/displayRestTimeLeft"
            />

        <Button
            android:id="@+id/restminus"
            android:layout_width="42dp"
            android:layout_height="42dp"
            android:layout_marginEnd="8dp"
            android:text="@string/minus_sign"
            app:layout_constraintBottom_toTopOf="@+id/restBar"
            app:layout_constraintEnd_toStartOf="@+id/setRestTime"
            app:layout_constraintHorizontal_chainStyle="packed"
            app:layout_constraintStart_toStartOf="@id/restBar"
            app:layout_constraintTop_toBottomOf="@+id/displayRestTimeLeft"/>

        <ProgressBar
            android:id="@+id/restBar"
            style="?android:attr/progressBarStyleHorizontal"
            android:layout_width="0dp"
            android:layout_height="16dp"
            android:layout_marginBottom="8dp"
            android:layout_marginEnd="8dp"
            android:layout_marginStart="8dp"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            />

        <ProgressBar
            android:id="@+id/workoutBar"
            style="?android:attr/progressBarStyleHorizontal"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_marginBottom="8dp"
            android:layout_marginEnd="8dp"
            android:layout_marginStart="8dp"
            android:layout_marginTop="8dp"
            app:layout_constraintBottom_toTopOf="@+id/restBar"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toBottomOf="@+id/startPause"/>

        <EditText
            android:id="@+id/numberOfSets"
            android:layout_width="0dp"
            android:layout_height="42dp"
            android:layout_marginStart="4dp"
            android:layout_marginTop="4dp"
            android:digits="0123456789"
            android:ems="10"
            android:imeOptions="actionDone"
            android:inputType="number"
            android:textAlignment="center"
            android:textSize="16sp"
            tools:text="Sets: 8/29"
            app:layout_constraintEnd_toStartOf="@+id/setsIncrease"
            app:layout_constraintHorizontal_bias="0.5"
            app:layout_constraintStart_toEndOf="@+id/setsDecrease"
            app:layout_constraintTop_toTopOf="parent"/>

        <Button
            android:id="@+id/setsIncrease"
            android:layout_width="42dp"
            android:layout_height="42dp"
            android:layout_marginStart="4dp"
            android:layout_marginTop="4dp"

            android:text="@string/plus_sign"
            app:layout_constraintStart_toEndOf="@+id/numberOfSets"
            app:layout_constraintTop_toTopOf="parent"/>

        <Button
            android:id="@+id/setsDecrease"
            android:layout_width="42dp"
            android:layout_height="42dp"
            android:layout_marginStart="4dp"
            android:layout_marginTop="4dp"

            android:text="@string/minus_sign"
            app:layout_constraintEnd_toStartOf="@+id/numberOfSets"
            app:layout_constraintHorizontal_bias="0.5"
            app:layout_constraintHorizontal_chainStyle="spread_inside"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toTopOf="parent"/>


    </androidx.constraintlayout.widget.ConstraintLayout>
</layout>
```

</details>

<br>

這個 UI 有以下功能：
1.

## BindingMethod
><br>
>
> **BindingMethod** 的功能是將 XML 上的某 `type` 中所使用的屬性 `attribute` 與方法 `method` 進行連結。
><br>

```java
@Target(ElementType.ANNOTATION_TYPE)
public @interface BindingMethod {

    /**
     * @return the View Class that the attribute is associated with.
     */
    Class type();

    /**
     * @return The attribute to rename. Use android: namespace for all android attributes or
     * no namespace for application attributes.
     */
    String attribute();

    /**
     * @return The method to call to set the attribute value.
     */
    String method();
}

```




### BindingMethods
若我們在一個類別中有多個方法需要進行連結，我們就可以使用 **BindingMethods** 。

```java
@Target({ElementType.TYPE})
public @interface BindingMethods {
    BindingMethod[] value();
}
```



## BindingAdapter


## BindingConversion







<br><br><br><br><br><br><br><br><br><br><br><br>
