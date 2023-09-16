---
layout: post
title:  "Interview : Android "
date:   2022-11-14 15:27:40 +08:00
use_math: true
categories: [android, interview]
---

## Background
以下是 Appworks School 校友們回饋的題目與出現的機率。

答案則是我個人的答覆，僅供參考。

## Questions
### Context 相關問題

#### 1. What is Application？ (35.7%)

#### 2. What is Context? (54.8%)

#### 3. What is Activity? (69%)

### Lifecycle Related

### Service, Content Provider, Broadcast Receiver Related

#### 1.

### Thread Related

### Language Related

### Gradle Related


### Algorithem Related



1. What’s Application？ (35.7%)
2. What’s Context？ (54.8%)
3. What’s a BuildType in Gradle？And what can you use it for？ (28.6%)
Gradle 中的 BuildType

4. 試說明什麼是 Activity。 (69%)

5. 試說明 Activity 的 Lifecycle。 (100%)

<details>
<summary><b> 5. 試說明 Activity 的 Lifecycle。 (100%)</b></summary>

Activity 的 Lifecycle 會以以下順序進行：
  1. onCreate - Activity 起動
  2. onStart - 使用者可以看到 UI
  3. onResume - 使用者可以與 UI 互動
  4. onPause - 使用者無法進行互動
  5. onStop - 使用者已看不到當下的 Activity UI
  6. onDestroy - Activity 已被結束

</details>


6. 試說明 onCreate() 與 onStart() 之間的不同。 (73.8%)


7. 試說明在哪種情境之下，Activity 只會呼叫到 onDestroy()，不會呼叫到 onPause()、onStop()。 (42.9%)
When `finish()` is called after `onCreate`


8. Activity 為什麼需要在 onCreate() 呼叫 setContentView()？ (16.7%)


9. 試說明 onSaveInstanceState() 與 onRestoreInstanceState()。 (45.2%)


10. 試說明 Activity 的啟動模式有哪些。什麼是 taskAffinity？ (59.5%)


11. Activtiy 在螢幕旋轉的時候，會發生哪些事？ (81%)
12. 如何避免在螢幕旋轉的時候，資料發生遺失的狀況？ (81%)
13. 請提供兩種方法在使用 Intent 去呼叫 Activity 時，能夠清除 Activity 的 back stack。 (33.3%)
14. FLAG_ACTIVITY_CLEAR_TASK 與 FLAG_ACTIVITY_CLEAR_TOP 的差別？ (23.8%)
15. 試說明 Intent。什麼是 Implicit Intent？ (45.2%)

16. 試說明什麼是 Broadcast Receiver。 (64.3%)
17. 靜態註冊與動態註冊分別如何實作？兩者的差別在哪？ (42.9%)

18. 試說明什麼是 Content Provider。 (54.8%)
19. 試說明 Content Provider 實現數據共享的流程。 (40.5%)

20. 試說明什麼是 Service。 (81%)
21. 試說明 Service 的 Lifecycle。 (47.6%)
22. Service 與 IntentService 的差別？ (42.9%)

23. 請問什麼是 Application Not Responding？在什麼情況下會發生？該如何去避免？ (83.3%)
24. 試說明什麼是 Coroutines。 (83.3%)
25. Coroutines 與 Threads 的差別？ (64.3%)
26. 試說明 Service、 IntentService、Coroutines 與 Threads 的區別。 (28.6%)
27. 試說明什麼是 Handler。 (40.5%)
28. 試說明 Handler、Looper、Message、Message Queue 的關聯。 (33.3%)
29. 什麼是 Dead Lock？ (21.4%)
30. 什麼是 Race Condition？ (14.3%)

31. 試說明 Value Type 與 Reference Type 的差別。 (52.4%)
32. 試說明什麼是 Memory Leak。 (78.6%)
33. 試說明 OOM 是什麼。通常在什麼情況下會發生？該如何防止 OOM 發生？ (78.6%)
34. 試說明 Stack 與 Heap 的差別。 (50%)
35. 試說明 Android 的 GC 運作機制。 (50%)
36. 請問 Android 有哪些 Reference types？ (35.7%)
37. 為什麼該使用 WeakReference 而不使用 SoftReference？ (26.2%)

38. 如果你進到公司，公司的 Project 放在 GitHub 上，公司請你把 code 拉下來看，你要怎麼做？ (31%)
39. 什麼是 Git flow？ (50%)
40. 如果現在 Play Store 上的 app 發生了 Critical 的 issue，我們需要修復並送出一個版本，你要怎麼做？ (26.2%)

41. 使用過純 Programming 動態配置 UI 的方法開發嗎？ (19%)
42. 有用過 LruCache 嗎？它可以用來做什麼？試說明 Lru 的機制。 (21.4%)
43. 有使用任何分析工具嗎？ (59.5%)
44. 為什麼要使用 Lint tool？ (9.5%)
45. 如何在兩個不同的 Instance 之間傳遞資料？ (54.8%)

46. 在 Android 裡，你要如何儲存資料？在所有選項中，彼此有什麼差別？ (73.8%)
47. 請問 Room、Internal/External Storage、SharedPreferences 有什麼差別？個別使用時機是什麼？ (64.3%)
48. 有使用過 SQLite 嗎？你是怎麼使用 SQLite/Room 的？ (50%)
49. Entiny 之間有做過關聯嗎？ (19%)
50. 有做過 Migration 嗎？ (16.7%)
51. 怎麼優化 Database 的效能？ (14.3%)

52. 試解釋 RESTful API。 (59.5%)
53. 在 RESTful API 中，GET 跟 POST 有什麼差別？ (54.8%)
54. 請描述在一個 Request 中，含有哪些部分？ (40.5%)
55. HTTP 是什麼？與 HTTPS 有什麼差別？在 Android 怎麼處理他們？ (38.1%)
56. 如果我想要發出一個 Request，會用到哪些 Class？ (19%)

57. 在 Android 裡，如何處理 JSON 資料？ (64.3%)

58. 有寫過 Unit Test 嗎？利用到哪些技巧？ (78.6%)
59. 什麼是 Mock？如何 Mock？ (40.5%)

60. 試說明什麼是 Job Scheduling。 (14.3%)
61. ArrayList v.s. LinkedList (45.2%)
62. ConstraintLayout 的優點是什麼？ (66.7%)
63. 試說明 Kotlin 中的 Visibility Modifiers。 (23.8%)
64. 試說明 Interface 與 Abstract Class 的差別。 (64.3%)
65. 試說明 Override 與 Overload 的差別。 (26.2%)
66. 試說明 Serializable 與 Parcelable 的差別。 (31%)
67. 試說明在 Kotlin 中 Data Class 與 Class 的差別。 (71.4%)
68. 試說明 Kotlin 中的 Companion Object 是什麼？與 static 的差別在哪？想在 Kotlin 中使用 static 的話要怎麼辦？ (57.1%)
69. 為什麼要使用 Kotlin 開發 Android，Kotlin 的優勢在哪？ (61.9%)
70. 請問什麼是 Functional Programing？ (31%)
71. 系統化出 View 時需要被調用的方法順序？View 的 post 與 Handler 的 post 有什麼不同？ (19%)
72. 請說明 final、finally、finalize 的差異在哪？ (4.8%)

73. 試說明什麼是 SOLID 原則。 (59.5%)
74. 試說明 SRP 單一職責原則。 (50%)
75. 試說明 OCP 開放封閉原則。 (50%)
76. 試說明 LSP 里氏替換原則。 (52.4%)
77. 試說明 ISP 介面隔離原則。 (47.6%)
78. 試說明 DIP 依賴反轉原則。 (50%)


79. 試著按照執行先後，說明 Android 的 Lifecycle。另請說明來電時的 Lifecycle 變化。 (66.7%)
80. 試說明 Model-View-ViewModel architecture pattern 的特性。另請說明與 Model-View-Controller architecture pattern 的差異。 (88.1%)
81. 請問什麼是 Application Not Responding？在哪些情況下會發生？如何去避免？ (73.8%)
82. 試說明 Activity 四種啟動模式的差別。 (71.4%)
83. 試說明 Observer Design Pattern 的用途及設計方式。 (71.4%)
84. 試說明 Singleton 是什麼。有什麼優缺點？使用上有什麼需要注意的地方？ (85.7%)
85. 試說明 要如何實作一個 Notification 功能？ (38.1%)
86. 試問 Service 是否能執行耗時的操作？為什麼？ (57.1%)
87. 請試著詳細描述 Fragment 的 Lifecycle。 (83.3%)
88. 請問什麼是 Scope Functions？ (57.1%)
89. Kotlin 中在 Property initialization 使用 lazy 以及 lateinit 的差別？ (47.6%)
90. 什麼是 OOP？ (66.7%)
91. 什麼是封裝 Encapsulation？ (57.1%)
92. 什麼是繼承 Inheritance？ (61.9%)
93. 什麼是多型 Polymorphism？ (50%)
94. 什麼是抽象 Abstraction？ (66.7%)
95. Companion object 跟 const 差別在哪裡？ (42.9%)
96. Why use LiveData with MutableLiveData in ViewModel？ (71.4%)
97. 分別說明 dp、dpi、sp、px 是什麼。 (31%)
98. 請說明 Gradle 中 Project 以及 Module 的差異。 (21.4%)
99. 請問為什麼要使用 ProGuard 做程式碼混淆？ (28.6%)
100. 請說明 RecyclerView 的回收、緩存原理、數量計算。 (59.5%)

101. 校友追加題庫
    a. 在不是 Main Thread 對 LiveData setValue 會發生甚麼事？
    b. data class 的 ==、equal 和 === 的差異？
    c. 什麼是 inline function？
    d. Coroutine 和 Flow 的差異？
    e. ConstraintLayout 的 guideline 和 barrier？
    f. 如何不用 library 實作 pagination？
    g. Process 間怎麼溝通？
    h. Broadcast Receiver 的接收順序會和發送 Broadcast 的順序一樣嗎？
    i. 請說明什麼是 synchronize 以及 volatile，我們如何使用，為何要使用？
    j. What’s SDK？
    k. What’s Socket？
    l. 費式數列相關題型。
    m. RxJava and Coroutines Flow。
    n. RxJava 有哪些 stream type，說明一下 type 的差別？
    o. 有使用過 JNI 開發嗎，何謂 JNI 呢？
    p. Repository 的用意是什麼？
    q. map 跟 flatMap 之間有什麼差異？
    r. 是否有使用過 DI？是否使用過 DI 套件，能否簡述其底層實作方式(是如何注入)？
    s. 在 Activity-Activity 、fragment-fragment 與 fragment-Activity 間傳遞訊息的方式？
    t. Kotlin 哪些地方會用到 object ？各代表什麼功能？
    u. DiffUtil
    v. 什麼是 thread-safe, thread 跟 coroutine 差異是什麼？
    w. View 怎麼畫出來的？
    x. 從 notification 啟動的 activity launch mode。
    y. 如果要作連續圖片的動畫功能，會如何實作？
    z. 如果頁面要從兩支 API 上拿回資料，合併成直的 RecyclerView 中嵌套一個橫的 RecyclerView，會如何實作。
