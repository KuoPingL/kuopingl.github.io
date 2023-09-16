---
layout: post
title:  "Android - CoordinatorLayout Behavior"
date:   2022-11-14 15:27:40 +08:00
use_math: true
categories: [android, layout]
---
# CoordinatorLayout.Behavior

這篇我們將要看看 **CoordinatorLayout.Behavior** 是什麼，以及如何實作。

**Behavior** 主要是用來定義 **CoordinatorLayout** 的 *child view* **對使用者行為所做出的反應**。

><br>Interaction behavior plugin for child views of CoordinatorLayout.<br>
A Behavior implements one or more interactions that a user can take on a child view. These interactions may include **drags**, **swipes**, **flings**, or any **other gestures**.
><br>

我們可以把源碼分為以下目的： <b>
- 創建
- 生命週期
- 互動
- UI
- 牽連
- 其他
</b>


## 創建

Behavior 可以用程式來創建，也可由 XML 來定義。

```java
public static abstract class Behavior<V extends View> {
  public Behavior()
  public Behavior(Context context, AttributeSet attrs)
}
```

如果想要使用 xml 就需要這樣寫：

```xml
app:layout_behavior="@string/bottom_sheet_behavior"
```

這裡的預設為 **BottomSheetBehavior** 。

```xml
<string name="bottom_sheet_behavior"
    translatable="false">
    com.google.android.material.bottomsheet.BottomSheetBehavior
</string>
```

## 生命週期

Behavior 的生命週期是由
- **onAttachedToLayoutParams**
  **onDetachedFromLayoutParams**
- **onMeasureChild**
  **onLayoutChild**
- **onRestoreInstanceState**
  **onSaveInstanceState**

```java
public void onAttachedToLayoutParams(@NonNull CoordinatorLayout.LayoutParams params)

public void onDetachedFromLayoutParams()

public boolean onMeasureChild(
    @NonNull CoordinatorLayout parent,
    @NonNull V child,
    int parentWidthMeasureSpec, int widthUsed,
    int parentHeightMeasureSpec, int heightUsed) {
    return false;
}

public boolean onLayoutChild(
    @NonNull CoordinatorLayout parent,
    @NonNull V child,
    int layoutDirection) {
    return false;
}

public void onRestoreInstanceState(@NonNull CoordinatorLayout parent, @NonNull V child,
        @NonNull Parcelable state) {
    // no-op
}

@Nullable
public Parcelable onSaveInstanceState(@NonNull CoordinatorLayout parent, @NonNull V child) {
    return BaseSavedState.EMPTY_STATE;
}
```

## 互動

Behavior 也會定義 View 與 使用者 和 ScollView 的行為：
- **onInterceptTouchEvent**
  **onTouchEvent**
- **onStartNestedScroll**

比較有趣的是 **blocksInteractionBelow**。 這個方法的預設行為是當 Behavior 的 `getScrimOpacity` $\gt$ **0. f** 時就會在 **CoordinatorLayout** 中的 `performIntercept` 阻斷下面 View 的反應。

```java
public boolean onInterceptTouchEvent(
    @NonNull CoordinatorLayout parent,
    @NonNull V child,
    @NonNull MotionEvent ev) {
    return false;
}

public boolean onTouchEvent(
    @NonNull CoordinatorLayout parent,
    @NonNull V child,
    @NonNull MotionEvent ev) {
    return false;
}

public boolean blocksInteractionBelow(@NonNull CoordinatorLayout parent, @NonNull V child) {
    return getScrimOpacity(parent, child) > 0.f;
}

public boolean onStartNestedScroll(
    @NonNull CoordinatorLayout coordinatorLayout,
    @NonNull V child,
    @NonNull View directTargetChild,
    @NonNull View target,
    @ScrollAxis int axes,
    @NestedScrollType int type) {

    if (type == ViewCompat.TYPE_TOUCH) {
        return onStartNestedScroll(coordinatorLayout, child, directTargetChild,
                target, axes);
    }
    return false;
}

public void onNestedScrollAccepted(
    @NonNull CoordinatorLayout coordinatorLayout,
    @NonNull V child,
    @NonNull View directTargetChild,
    @NonNull View target,
    @ScrollAxis int axes,
    @NestedScrollType int type) {

    if (type == ViewCompat.TYPE_TOUCH) {
        onNestedScrollAccepted(coordinatorLayout, child, directTargetChild,
                target, axes);
    }
}

public void onStopNestedScroll(
    @NonNull CoordinatorLayout coordinatorLayout,
    @NonNull V child,
    @NonNull View target,
    @NestedScrollType int type) {

    if (type == ViewCompat.TYPE_TOUCH) {
        onStopNestedScroll(coordinatorLayout, child, target);
    }
}

public void onNestedScroll(
    @NonNull CoordinatorLayout coordinatorLayout,
    @NonNull V child,
    @NonNull View target,
    int dxConsumed, int dyConsumed,
    int dxUnconsumed, int dyUnconsumed,
    @NestedScrollType int type,
    @NonNull int[] consumed) {
    // In the case that this nested scrolling v3 version is not implemented, we call the v2
    // version in case the v2 version is. We Also consume all of the unconsumed scroll
    // distances.
    consumed[0] += dxUnconsumed;
    consumed[1] += dyUnconsumed;
    onNestedScroll(coordinatorLayout, child, target, dxConsumed, dyConsumed,
            dxUnconsumed, dyUnconsumed, type);
}

public void onNestedPreScroll(
    @NonNull CoordinatorLayout coordinatorLayout,
    @NonNull V child,
    @NonNull View target, int dx, int dy,
    @NonNull int[] consumed,
    @NestedScrollType int type) {
    if (type == ViewCompat.TYPE_TOUCH) {
        onNestedPreScroll(coordinatorLayout, child, target, dx, dy, consumed);
    }
}

public boolean onNestedFling(
    @NonNull CoordinatorLayout coordinatorLayout,
    @NonNull V child,
    @NonNull View target,
    float velocityX, float velocityY,
    boolean consumed) {
    return false;
}

public boolean onNestedPreFling(
    @NonNull CoordinatorLayout coordinatorLayout,
    @NonNull V child,
    @NonNull View target,
    float velocityX, float velocityY) {
    return false;
}
```

## UI

```java

/*
* scrimColor 除了設顏色外，還可以讓使用者知道此 View 下面的
* View 皆為 inactive 。
*
* 而且通過 Opacity， CoordinatorLayout 便可知道下方的 View 是否為互動模式。
*/
@ColorInt
public int getScrimColor(@NonNull CoordinatorLayout parent, @NonNull V child) {
    return Color.BLACK;
}

@FloatRange(from = 0, to = 1)
public float getScrimOpacity(@NonNull CoordinatorLayout parent, @NonNull V child) {
    return 0.f;
}

@NonNull
public WindowInsetsCompat onApplyWindowInsets(
    @NonNull CoordinatorLayout coordinatorLayout,
    @NonNull V child,
    @NonNull WindowInsetsCompat insets) {
    return insets;
}

/*
* Called when a child of the view associated
* with this behavior wants a particular rectangle
* to be positioned onto the screen.
*
* 如同 ViewParent#requestChildRectangleOnScreen
*/
public boolean onRequestChildRectangleOnScreen(@NonNull CoordinatorLayout coordinatorLayout,
        @NonNull V child, @NonNull Rect rectangle, boolean immediate) {
    return false;
}
```

## 牽引

```java
public boolean layoutDependsOn(
    @NonNull CoordinatorLayout parent,
    @NonNull V child,
    @NonNull View dependency) {
    return false;
}

public boolean onDependentViewChanged(
    @NonNull CoordinatorLayout parent,
    @NonNull V child,
    @NonNull View dependency) {
    return false;
}

public void onDependentViewRemoved(
    @NonNull CoordinatorLayout parent,
    @NonNull V child,
    @NonNull View dependency) {
}

public boolean getInsetDodgeRect(@NonNull CoordinatorLayout parent, @NonNull V child,
        @NonNull Rect rect) {
    return false;
}
```

當我們想要將 Behavior 的 View 與某個 View 進行互相牽引，我們需要覆寫 `layoutDependsOn`。

`layoutDependsOn` 會在 layout request 時被調用至少一次。

如果 `layoutDependsOn` 回傳為 **true**，那 parent **CoordinatorLayout** 便會在 `child` 被佈局後進行此 View 的佈局。 **不管 child 與 View 的階層關係**。

若 child 有所變動，就會調用 `onDependentViewChanged` 。

另外，我們還可以通過 `getInsetDodgeRect` 來設定 View 需要躲避 CoordinatorLayout 中的 **Rect** 。

## 其他
```java
public static void setTag(
    @NonNull View child,
    @Nullable Object tag) {

    final LayoutParams lp = (LayoutParams) child.getLayoutParams();
    lp.mBehaviorTag = tag;
}

@Nullable
public static Object getTag(
    @NonNull View child) {

    final LayoutParams lp = (LayoutParams) child.getLayoutParams();
    return lp.mBehaviorTag;
}
```


# 實作
## ViewOffsetBehavior

**ViewOffsetBehavior** 只有覆寫了 **Behavior** 的 `onLayoutChild` ：

```java
@Override
public boolean onLayoutChild(
    @NonNull CoordinatorLayout parent, @NonNull V child, int layoutDirection) {
  // First let lay the child out
  layoutChild(parent, child, layoutDirection);

  if (viewOffsetHelper == null) {
    viewOffsetHelper = new ViewOffsetHelper(child);
  }
  viewOffsetHelper.onViewLayout();
  viewOffsetHelper.applyOffsets();

  if (tempTopBottomOffset != 0) {
    viewOffsetHelper.setTopAndBottomOffset(tempTopBottomOffset);
    tempTopBottomOffset = 0;
  }
  if (tempLeftRightOffset != 0) {
    viewOffsetHelper.setLeftAndRightOffset(tempLeftRightOffset);
    tempLeftRightOffset = 0;
  }

  return true;
}

protected void layoutChild(
    @NonNull CoordinatorLayout parent, @NonNull V child, int layoutDirection) {
  // Let the parent lay it out by default
  parent.onLayoutChild(child, layoutDirection);
}
```

從以上可看出，這個實作只是讓 View 可以由 **ViewOffsetHelper** 動態更新 **上下左右** 的 **offset**。

我們會在 **AppBarLayout** 中看到這類別的使用。不過在此之前，我們看看 **HeaderBehavior**，一個繼承 **ViewOffsetBehavior** 的抽象類別。

## HeaderBehavior

```java
abstract class HeaderBehavior<V extends View> extends ViewOffsetBehavior<V>
```

><br>
>The CoordinatorLayout.Behavior for a view that sits <i>vertically</i> <b><u>above</u> scrolling a view</b>.
><br><br>

這個抽象類別主要是覆寫了 `onInterceptTouchEvent` 與 `onTouchEvent` 。 因為源碼挺長的，所以我們將分開探討。

**onInterceptTouchEvent**

這方法中可以分為兩個步驟：
1. 取得當下 `activePointerId`、確保 `velocityTracker` 的存在 和 更新 `isBeingDragged`，並停止 `scroller` 的動畫。
2. 確保使用者在拖拉時手指沒有移開，算出 `yDiff` 並更新 `lastMotionY`，最後通過回傳 `true` 來調用 `onTouchEvent`。

```java
@Override
public boolean onInterceptTouchEvent(
    @NonNull CoordinatorLayout parent, @NonNull V child, @NonNull MotionEvent ev) {
  if (touchSlop < 0) {
    touchSlop = ViewConfiguration.get(parent.getContext()).getScaledTouchSlop();
  }

  // Shortcut since we're being dragged
  /* 第二步：
      確保 pointer 一致，並且手指未離開
      取得 y 位置
      計算出 yDiff 並更新 lastMotionY
      回傳 true

      接下來 onTouchEvent 就會被調用
  */
  if (ev.getActionMasked() == MotionEvent.ACTION_MOVE && isBeingDragged) {
    if (activePointerId == INVALID_POINTER) {
      // If we don't have a valid id, the touch down wasn't on content.
      return false;
    }
    int pointerIndex = ev.findPointerIndex(activePointerId);
    if (pointerIndex == -1) {
      return false;
    }

    int y = (int) ev.getY(pointerIndex);
    int yDiff = Math.abs(y - lastMotionY);
    if (yDiff > touchSlop) {
      lastMotionY = y;
      return true;
    }
  }

  /**
  * 第一步：
  *   取得 activePointerId、
  *   確保 velocityTracker 的存在，
  *   停止 OverScroller 的行為
  */
  if (ev.getActionMasked() == MotionEvent.ACTION_DOWN) {
    activePointerId = INVALID_POINTER;

    int x = (int) ev.getX();
    int y = (int) ev.getY();
    isBeingDragged = canDragView(child) && parent.isPointInChildBounds(child, x, y);
    if (isBeingDragged) {
      lastMotionY = y;
      activePointerId = ev.getPointerId(0);
      ensureVelocityTracker();

      // There is an animation in progress. Stop it and catch the view.
      if (scroller != null && !scroller.isFinished()) {
        scroller.abortAnimation();
        return true;
      }
    }
  }

  if (velocityTracker != null) {
    velocityTracker.addMovement(ev);
  }

  return false;
}
```

`onInterceptTouchEvent` 的重點在於 `isBeingDragged` 這個參數的變更。

`isBeingDragged` 只會當使用者在可被拖拉的 child 中 **按下** 時被設為 **true**。

><br>
>
> <b>HeaderBehavior</b> 的 `canDragView` 預設為 **false**。
><br>

而當 `isBeingDragged` 為 **true** 時， **onInterceptTouchEvent** 會更新 `activePointerId` 、 `velocityTracker` 並停止 `OverScroller scroller` 的動畫。

><br>
>
> `activePointerId` 是用來記錄按下的那個點，如此一來就知道使用者的手指是否有拿開。
>
>而 `velocityTracker` 可以讓我們將 `motionEvent` 放入並得到對應的 `velocityX` 與 `velocityY`。
>
> `scroller` 則是用來定義滾動時的行為。
><br><br/>

通過 `onInterceptTouchEvent`，如果 View 的拖拉已被確實，就會調用 `onTouchEvent` 進行接下來的行為。

**onTouchEvent**

```java
@Override
public boolean onTouchEvent(
    @NonNull CoordinatorLayout parent, @NonNull V child, @NonNull MotionEvent ev) {
  boolean consumeUp = false;

  switch (ev.getActionMasked()) {

    /* 移動時：
          1. 確保調用 onInterceptTouchEvent 時的手指沒有移開。
          2. 取得目前 y 與 lastMotionY 的差別 (dy) 並更新 lastMotionY
          3. 將 dy 傳入 scroll 方法中進行 setHeaderTopBottomOffset 的調用
    */
    case MotionEvent.ACTION_MOVE:
      final int activePointerIndex = ev.findPointerIndex(activePointerId);
      if (activePointerIndex == -1) {
        return false;
      }

      final int y = (int) ev.getY(activePointerIndex);
      int dy = lastMotionY - y;
      lastMotionY = y;
      // We're being dragged so scroll the ABL
      scroll(parent, child, dy, getMaxDragOffset(child), 0);
      break;

    /* 當只少還有一隻手指在畫面上
          1. 查看移開的手指是否 index 0 (原來的手指) 並進行更新
          2. 按照新的手指位置更新 lastMotionY
    */
    case MotionEvent.ACTION_POINTER_UP:
      int newIndex = ev.getActionIndex() == 0 ? 1 : 0;
      activePointerId = ev.getPointerId(newIndex);
      lastMotionY = (int) (ev.getY(newIndex) + 0.5f);
      break;

    /* 當最後一隻手指離開畫面時，
       如果 velocityTracker 存在，那就：
          1. 更新 consumeUp = true (告知行為完成)
          2. 將 motionEvent 放入 velocityTracker
          3. 設定 velocityTracker 的目前速度為 1000 pixel / second
          4. 從 velocityTracker 取得計算後的 yVelocity
          5. 將這個 velocity 放入 fling 方法來顯示正確行為 (這裡會使用 Runnable 之後可以看到)
    */
    case MotionEvent.ACTION_UP:
      if (velocityTracker != null) {
        consumeUp = true;
        velocityTracker.addMovement(ev);

        // 1000 表示 1000 pixel / second
        velocityTracker.computeCurrentVelocity(1000);
        float yvel = velocityTracker.getYVelocity(activePointerId);
        fling(parent, child, -getScrollRangeForDragFling(child), 0, yvel);
      }

      // $FALLTHROUGH
      // 進行回收
    case MotionEvent.ACTION_CANCEL:
      isBeingDragged = false;
      activePointerId = INVALID_POINTER;
      if (velocityTracker != null) {
        velocityTracker.recycle();
        velocityTracker = null;
      }
      break;
  }

  if (velocityTracker != null) {
    velocityTracker.addMovement(ev);
  }

  // 若 consumedUp 或 isBeingDragged 就表示此 View 會處理這些行為
  return isBeingDragged || consumeUp;
}
```

以下是 `fling` 的實作，裡面的 `layout` 指的是 `child` 。

```java
final boolean fling(
    CoordinatorLayout coordinatorLayout,
    @NonNull V layout,
    int minOffset,
    int maxOffset,
    float velocityY) {

  // 1. 移除 flingRunnable 的 callback
  if (flingRunnable != null) {
    layout.removeCallbacks(flingRunnable);
    flingRunnable = null;
  }

  // 2. 創建 scroller
  if (scroller == null) {
    scroller = new OverScroller(layout.getContext());
  }

  // 3. 這裡預設 overX 與 overY 為 0 並計算正確的 velocityX 與 velocityY。
  //    最後就會通過 OverScroller scrollerX 與 scrollerY 來進行 X, Y 方向的滑動
  scroller.fling(
      0, // startX
      getTopAndBottomOffset(), // curr, startY
      0, // velocityX
      Math.round(velocityY), // velocityY
      0, // minX, will not scroll past this point
      0, // maxX
      minOffset, // minY
      maxOffset); // maxY

  if (scroller.computeScrollOffset()) {
    // 4. 只有當動畫尚未完成，才可進去
    //    進去之後就會創建 FlingRunnable，
    //    並通過 postOnAnimation 將他放入 HandlerActionQueue 中

    flingRunnable = new FlingRunnable(coordinatorLayout, layout);
    ViewCompat.postOnAnimation(layout, flingRunnable);
    return true;
  } else {
    // 4. 若動畫已結束，那就調用 onFlingFinished
    //    來進行任何 Fling 過後要做的事。 預設為無為。
    onFlingFinished(coordinatorLayout, layout);
    return false;
  }
}
```

`onFlingFinished` 的運用範例可以在 **AppBarLayout** 中找到：

```java
@Override
void onFlingFinished(@NonNull CoordinatorLayout parent, @NonNull T layout) {
  // At the end of a manual fling,
  // check to see if we need to snap to the edge-child
  snapToChildIfNeeded(parent, layout);
  if (layout.isLiftOnScroll()) {
    layout.setLiftedState(layout.shouldLift(findFirstScrollingChild(parent)));
  }
}
```

不過運行部分會在另一個篇幅中詳細敘述。

## HeaderScrollingViewBehavior

```java
abstract class HeaderScrollingViewBehavior<V extends View> extends ViewOffsetBehavior<V>
```

><br>
>The CoordinatorLayout.Behavior for a view that sits <i>vertically</i> <b><u>below</u></b> ( 與 HeaderBehavior 相反 ) <b>scrolling a view</b>.
><br><br>

這個抽象類別蠻有趣的。 有別於覆寫 `onTouchEvent` 與 `onInterceptTouchEvent` 的 **HeaderBehavior** ， **HeaderScrollingViewBehavior** 覆寫了 `onMeasureChild` 與 `layoutChild` 。

**onMeasureChild**
此方法會算出 Layout

```java
@Override
public boolean onMeasureChild(
    @NonNull CoordinatorLayout parent,
    @NonNull View child,
    int parentWidthMeasureSpec,
    int widthUsed,
    int parentHeightMeasureSpec,
    int heightUsed) {

  final int childLpHeight = child.getLayoutParams().height;

  // 如果有定義 MATCH_PARENT 或 WRAP_CONTENT 才會進行調整
  if (childLpHeight == ViewGroup.LayoutParams.MATCH_PARENT
      || childLpHeight == ViewGroup.LayoutParams.WRAP_CONTENT) {

    // If the menu's height is set to match_parent/wrap_content then measure it
    // with the maximum visible height

    /* 1. 取得所有牽引對象
    *
    *   CoordinatorLayout 會在 prepareChildren 方法
    *   將所有 child 與他的 dependency view 都放入 DAG 中。
    *
    *   因此，我們可以在此從 parent 中取得 child 的 dependencies
    *   final List<View> dependencies = mChildDag.getOutgoingEdges(child);
    */
    final List<View> dependencies = parent.getDependencies(child);

    /* 2. findFirstDependency 是一個抽象方法
    *     通過此方法，我們理應找到滑動時需要注意到的物件
    *     譬如： AppBarLayout 中 ScrollingViewBehavior
    *     便是回傳 (AppBarLayout) view;
    */
    final View header = findFirstDependency(dependencies);


    if (header != null) {
      // 3. 計算可用的高度
      int availableHeight = View.MeasureSpec.getSize(parentHeightMeasureSpec);

      // 4. 進行高度調整
      if (availableHeight > 0) {

        // 這裡會確保 header 是 Fit System Window
        // ( 預設行為是全螢幕 )
        // 所以可用高度也要將 System Window 的 insets 加入參考
        if (ViewCompat.getFitsSystemWindows(header)) {

          final WindowInsetsCompat parentInsets = parent.getLastWindowInsets();
          if (parentInsets != null) {
            availableHeight += parentInsets.getSystemWindowInsetTop()
                + parentInsets.getSystemWindowInsetBottom();
          }
        }
      } else {
        // If the measure spec doesn't specify a size, use the current height
        availableHeight = parent.getHeight();
      }

      // getScrollRange 回傳的是 view.getMeasuredHeight()
      int height = availableHeight + getScrollRange(header);

      int headerHeight = header.getMeasuredHeight();

      // shouldHeaderOverlapScrollingChild 預設為 false ()
      if (shouldHeaderOverlapScrollingChild()) {
        // 這會將 child 往上移
        child.setTranslationY(-headerHeight);
      } else {
        // 這會將 height 變回 availableHeight
        height -= headerHeight;
      }


      final int heightMeasureSpec =
          View.MeasureSpec.makeMeasureSpec(
              height,
              childLpHeight == ViewGroup.LayoutParams.MATCH_PARENT
                  ? View.MeasureSpec.EXACTLY
                  : View.MeasureSpec.AT_MOST);

      // Now measure the scrolling view with the correct height
      parent.onMeasureChild(
          child, parentWidthMeasureSpec, widthUsed, heightMeasureSpec, heightUsed);

      return true;
    }
  }
  return false;
}
```



## AppBarLayout.Behavior



## BottomSheetBehavior
><br> An interaction behavior plugin for a child view of CoordinatorLayout to make it work as a bottom sheet.
><br>

```java
public class BottomSheetBehavior<V extends View>
        extends CoordinatorLayout.Behavior<V>
```
<details>
<summary><b>Private 參數</b></summary>

```java
private boolean fitToContents = true;
private boolean updateImportantForAccessibilityOnSiblings = false;
private float maximumVelocity;

private int peekHeight;
private boolean peekHeightAuto;
private int peekHeightMin;
private int peekHeightGestureInsetBuffer;

private boolean shapeThemingEnabled;
private MaterialShapeDrawable materialShapeDrawable;
private ShapeAppearanceModel shapeAppearanceModelDefault;
private boolean isShapeExpanded;

private int maxWidth = NO_MAX_SIZE;
private int maxHeight = NO_MAX_SIZE;

private int gestureInsetBottom;
private boolean gestureInsetBottomIgnored;

private boolean paddingBottomSystemWindowInsets;
private boolean paddingLeftSystemWindowInsets;
private boolean paddingRightSystemWindowInsets;
private boolean paddingTopSystemWindowInsets;
private int insetBottom;
private int insetTop;

private SettleRunnable settleRunnable = null;

@Nullable private ValueAnimator interpolatorAnimator;
private static final int DEF_STYLE_RES = R.style.Widget_Design_BottomSheet_Modal;
```

</details>





```java

public abstract static class BottomSheetCallback {
  /**
    State 包含
      STATE_DRAGGING
      STATE_SETTLING
      STATE_EXPANDED
      STATE_COLLAPSED
      STATE_HIDDEN
      STATE_HALF_EXPANDED
  **/
  public abstract void onStateChanged(@NonNull View bottomSheet, @State int newState);
  public abstract void onSlide(@NonNull View bottomSheet, float slideOffset);
}

```
