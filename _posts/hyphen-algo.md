---
layout: post
title: Algorithm - Hyphen Algorithm
categories: [Algorithm]
use_math: true
keywords: algorithm, hyphen
---

# 背景
之所以會談到這個 **Hyphen 演算法** 是因為在我探索 Android 的 **TextView** 時，得知原來 **Layout** 在調用 `drawText` 時：

```java
drawText(Canvas canvas, int firstLine, int lastLine)
```

會調用

```java
paint.setStartHyphenEdit(getStartHyphenEdit(lineNum));
paint.setEndHyphenEdit(getEndHyphenEdit(lineNum));
```

雖然 **StartHyphenEdit** 與 **EndHyphenEdit** 都是 **No_Edit** 但為什麽需要這兩個參數呢？

為了找到答案就得看看 **[Canvas.cpp](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/libs/hwui/hwui/Canvas.cpp;bpv=1;bpt=0)** 的 `Canvas::drawText`。 在畫出字串前， Canvas 會通過 **MinikinUtils** 的 `doLayout` 來取得字串的 Bounds。

```CPP
minikin::Layout MinikinUtils::doLayout(
  const Paint* paint,
  minikin::Bidi bidiFlags,
  const Typeface* typeface,
  const uint16_t* buf, size_t bufSize,
  size_t start, size_t count,
  size_t contextStart, size_t contextCount,
  minikin::MeasuredText* mt)
```

這方法會創建 **Layout** 並調用 `doLayout` 來進行字串的排列。 為了要節省運算時間， `doLayout` 中會調用 `doLayoutRunCached`。

```cpp
float Layout::doLayoutRunCached(const U16StringPiece& textBuf, const Range& range, bool isRtl,
                                const MinikinPaint& paint, size_t dstStart,
                                StartHyphenEdit startHyphen, EndHyphenEdit endHyphen,
                                Layout* layout, float* advances) {
    if (!range.isValid()) {
        return 0.0f;  // ICU failed to retrieve the bidi run?
    }
    float advance = 0;
    for (const auto[context, piece] : LayoutSplitter(textBuf, range, isRtl)) {
        // Hyphenation only applies to the start/end of run.
        const StartHyphenEdit pieceStartHyphen =
                (piece.getStart() == range.getStart()) ? startHyphen : StartHyphenEdit::NO_EDIT;
        const EndHyphenEdit pieceEndHyphen =
                (piece.getEnd() == range.getEnd()) ? endHyphen : EndHyphenEdit::NO_EDIT;
        float* advancesForRun =
                advances ? advances + (piece.getStart() - range.getStart()) : nullptr;
        advance += doLayoutWord(textBuf.data() + context.getStart(),
                                piece.getStart() - context.getStart(), piece.getLength(),
                                context.getLength(), isRtl, paint, piece.getStart() - dstStart,
                                pieceStartHyphen, pieceEndHyphen, layout, advancesForRun);
    }
    return advance;
}
```

`doLayoutRunCached` 會通過 **LayoutSplitter** 將字串按照對應的 Range 來取得合適長度的 text ：

```cpp
// LayoutSplitter split the input text into recycle-able pieces.
//
// LayoutSplitter basically splits the text before and after space characters.
//
// Here is an example of how the LayoutSplitter split the text into layout pieces.
// Input:
//   Text          : T h i s _ i s _ a n _ e x a m p l e _ t e x t .
//   Range         :            |-------------------|
//
// Output:
//   Context Range :          |---|-|---|-|-------------|
//   Piece Range   :            |-|-|---|-|---------|
//
// Input:
//   Text          : T h i s _ i s _ a n _ e x a m p l e _ t e x t .
//   Range         :                          |-------|
//
// Output:
//   Context Range :                      |-------------|
//   Piece Range   :                          |-------|
class LayoutSplitter {
public:
    LayoutSplitter(const U16StringPiece& textBuf, const Range& range, bool isRtl)
            : mTextBuf(textBuf), mRange(range), mIsRtl(isRtl) {}
```

而在 for-loop 創建 **LayoutSplitter** 的同時便會通過以下方法取得 **Range** 型別的 `context` 與 `piece`：

```cpp
// Range mContextRange;
// Range mPieceRange;
std::pair<Range, Range> operator*() const {
    return std::make_pair(mContextRange, mPieceRange);
}
```

在取得 **StartHyphenEdit** 與 **EndHyphenEdit** 後會調用 `doLayoutWord` ：

```cpp
float Layout::doLayoutWord(const uint16_t* buf, size_t start, size_t count, size_t bufSize,
                           bool isRtl, const MinikinPaint& paint, size_t bufStart,
                           StartHyphenEdit startHyphen, EndHyphenEdit endHyphen, Layout* layout,
                           float* advances) {
    float wordSpacing = count == 1 && isWordSpace(buf[start]) ? paint.wordSpacing : 0;
    float totalAdvance = 0;

    const U16StringPiece textBuf(buf, bufSize);
    const Range range(start, start + count);
    LayoutAppendFunctor f(layout, advances, &totalAdvance, bufStart, wordSpacing);
    LayoutCache::getInstance().getOrCreate(textBuf, range, paint, isRtl, startHyphen, endHyphen, f);

    if (wordSpacing != 0) {
        totalAdvance += wordSpacing;
        if (advances) {
            advances[0] += wordSpacing;
        }
    }
    return totalAdvance;
}
```

此時，


```cpp
// Do not use LayoutCache inside the callback function, otherwise dead-lock may happen.
template <typename F>
void getOrCreate(const U16StringPiece& text, const Range& range, const MinikinPaint& paint,
                 bool dir, StartHyphenEdit startHyphen, EndHyphenEdit endHyphen, F& f) {
    LayoutCacheKey key(text, range, paint, dir, startHyphen, endHyphen);
    if (paint.skipCache() || range.getLength() >= LENGTH_LIMIT_CACHE) {
        f(LayoutPiece(text, range, dir, paint, startHyphen, endHyphen), paint);
        return;
    }
    {
        std::lock_guard<std::mutex> lock(mMutex);
        LayoutPiece* layout = mCache.get(key);
        if (layout != nullptr) {
            f(*layout, paint);
            return;
        }
    }
    // Doing text layout takes long time, so releases the mutex during doing layout.
    // Don't care even if we do the same layout in other thred.
    key.copyText();
    std::unique_ptr<LayoutPiece> layout =
            std::make_unique<LayoutPiece>(text, range, dir, paint, startHyphen, endHyphen);
    f(*layout, paint);
    {
        std::lock_guard<std::mutex> lock(mMutex);
        mCache.put(key, layout.release());
    }
}
```





# 參考資料
1. [wiki - Syllabification](https://en.wikipedia.org/wiki/Syllabification)
2.









<br><br><br><br><br><br><br>
