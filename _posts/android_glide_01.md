---
layout: post
title:  "從 code 暸解 Glide 的運作"
date:   2022-08-08 16:15:07 +0800
categories: [glide, android]
---

# 簡介
**Glide** 是其中一個最常用來顯示圖像的函式庫。

這篇我們將從 [codelab : advanced-kotlin-coroutines](https://developer.android.com/codelabs/advanced-kotlin-coroutines#0) 的程式碼來暸解 Glide 的運作。

```kotlin
/*
  xml 會通過
  app:imageFromUrl="@{plant.imageUrl}"
  調用此方法
*/

@BindingAdapter("imageFromUrl")
fun bindImageFromUrl(view: ImageView, imageUrl: String?) {
    if (!imageUrl.isNullOrEmpty()) {
        Glide.with(view.context)
                .load(imageUrl)
                .transition(DrawableTransitionOptions.withCrossFade())
                .into(view)
    }
}
```

# 從創建到顯示
## Glide.with

```kotlin
Glide.with(view.context) // 取得 RequestManager
```

通過 `Glide.with` 我們會創建 **Glide**，並將以下變數一同設定：

```java
// 擁有不同功能的 ExecutorService
GlideExecutor sourceExecutor    // 4 or number of processor threads
GlideExecutor diskCacheExecutor // 1 thread
GlideExecutor animationExecutor // 1 or 2 threads

MemorySizeCalculator memorySizeCalculator
DefaultConnectivityMonitorFactory connectivityMonitorFactory

BitmapPool bitmapPool  // 可以是 LruBitmapPool 或 BitmapPoolAdapter
LruArrayPool arrayPool // 預設且最大為 4 MB，但會通過 MemorySizeCalculator 計算當下最合適的容量
LruResourceCache memoryCache

// 預設為 250 MB
InternalCacheDiskCacheFactory diskCacheFactory

// 用來載入、管理與暫存資源。
Engine engine
```

最終我們會得到 **RequestManager**。

## RequestManager.load

```kotlin
Glide.with(view.context)
     .load(imageUrl)
```

在 **RequestManager** 中，`load` 方法除了可以傳入 **String** 外，還傳入 **Uri** , **File** , **Object** , **byte [ ]**, **Bitmap**, **Integer** (resourceId) 與 **Drawable**。

當調用 `load` 時，他會創建 **RequestBuilder** ：

```java
new RequestBuilder<>(glide, this, resourceClass, context);
```

在創建的同時， `load` 除了會將傳入的變數設為 **RequestBuilder** 的 `model`， `load` 還會設定 **RequestBuilder** 的 **DiskCacheStrategy**。

一旦創建，就會調用 `RequestBuilder.load` 方法。 而這個方法有以下不同的實作：

```java
/*
    Uri, File, Object, String,
*/
loadGeneric({any type above})

/*
    Bitmap, Drawable
*/
loadGeneric({bitmap or drawable})
    .apply(diskCacheStrategyOf(DiskCacheStrategy.NONE));

//  byte[]
public RequestBuilder<TranscodeType> load(@Nullable byte[] model) {
  RequestBuilder<TranscodeType> result = loadGeneric(model);
  if (!result.isDiskCacheStrategySet()) {
    result = result.apply(diskCacheStrategyOf(DiskCacheStrategy.NONE));
  }
  if (!result.isSkipMemoryCacheSet()) {
    result = result.apply(skipMemoryCacheOf(true /*skipMemoryCache*/));
  }
  return result;
}
```

當 `loadGeneric` 被調用後，**Bitmap** 、 **Drawable** 與 **byte [ ]** 都會對 **DiskCacheStrategy** 進行設定。 而 Glide 提供了以下種類：

|DiskCacheStrategy|作用|
|:--|:--|
|AUTOMATIC   | 自動使用最好策略  |
|DATA   | 在資料解析 **之前** 將資料存入 disk cache 中  |
|RESOURCE   | 在資料解析 **之後** 將資料存入 disk 中 |
|ALL   | DATA 與 RESOURCE 都會執行  |
|NONE   | 不會進行任何存檔  |


如果我們想要設定 **DiskCacheStrategy**，我們也可以寫：

```kotlin
Glide.with(view.context)
     .load(imageUrl)
     .diskCacheStrategy(DiskCacheStrategy.ALL)
```


## RequestManager.transition

```kotlin
Glide.with(view.context)
     .load(imageUrl)
     .transition(DrawableTransitionOptions.withCrossFade())
```

`transition` 是用來設定 **RequestBuilder** 的 **TransitionOptions** `transitionOptions`。




在此，我們調用了 **DrawableTransitionOptions** 的 `withCrossFade` 來取得 **TransitionFactory**。

```java
/*
    1. 建立 DrawableTransitionOptions 並調用 crossFade()
*/
@NonNull
public static DrawableTransitionOptions withCrossFade() {
  return new DrawableTransitionOptions().crossFade();
}

/*
    2. 建立 DrawableCrossFadeFactory.Builder
       並傳入 crossFade(DrawableCrossFadeFactory.Builder)
*/
@NonNull
public DrawableTransitionOptions crossFade() {
  return crossFade(new DrawableCrossFadeFactory.Builder());
}

/*
    3. 從 DrawableCrossFadeFactory.Builder 創建 DrawableCrossFadeFactory
       並傳入 crossFade(DrawableCrossFadeFactory)
*/
@NonNull
public DrawableTransitionOptions crossFade(@NonNull DrawableCrossFadeFactory.Builder builder) {
  return crossFade(builder.build());
}

/*
    4. 調用 TransitionOptions 中的
       transition(TransitionFactory)
*/
@NonNull
public DrawableTransitionOptions crossFade(
    @NonNull DrawableCrossFadeFactory drawableCrossFadeFactory) {
  return transition(drawableCrossFadeFactory);
}
```




# Lru










<br><br><br>
