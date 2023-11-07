---
layout: post
title:  "Android - Understand Spinner"
date:   2023-11-07 10:09:58 +0800
categories: [spinner, android, beginner]
---

# 序章
昨天，我朋友剛好問我是否有做過這樣的 **Spinner** ：


<center>
    <a href="https://stackoverflow.com/questions/38654110/change-background-color-of-the-selected-item-in-android-spinner"><img src = "/images/posts/jekyll/android/spinner/spinner_with_selected_color.png" style="width:70%"/></a>
</center>

原本想要自己試試看做出來，誰知道最後還是請出 stackoverflow 大神指點迷津。

發現自己對 Spinner 真的很不熟，所以決定寫一篇文章來分析 Spinner 的行為了。

# Spinner 的架構

其實 Spinner 本身是一個 **ViewGroup** :

<center>
    <img src = "/images/posts/jekyll/android/spinner/spinner_hierarchy.png" style="width:70%"/>
</center>

它之所以可以顯示選單是因為 **Adapter** 的緣故。我們會先從 **Adapter** 開起，最後會將整個 Spinner 的顯示流程推導出來。

## Adapter

**Adpater** 是一個介面，但他的行為與 **RecyclerView.Adapter** 極為相似：

```java 
public interface Adapter {
    void registerDataSetObserver(DataSetObserver observer);
    void unregisterDataSetObserver(DataSetObserver observer);

    int getCount();
    Object getItem(int position);
    long getItemId(int position);
    
    /**
     * Indicates whether the item ids are stable across changes to the
     * underlying data.
     * 
     * @return True if the same id always refers to the same object.
     */
    boolean hasStableIds();
    View getView(int position, View convertView, ViewGroup parent);

    static final int IGNORE_ITEM_VIEW_TYPE = AdapterView.ITEM_VIEW_TYPE_IGNORE;
    int getItemViewType(int position);
    int getViewTypeCount();
    
    static final int NO_SELECTION = Integer.MIN_VALUE;
    boolean isEmpty();

    default @Nullable CharSequence[] getAutofillOptions() {
        return null;
    }
}
```

繼承 Adapter 的兩個介面分別是 **ListAdapter** 與 **SpinnerAdapter** ：

<center>
    <img src = "/images/posts/jekyll/android/spinner/adapter_interfaces.png" style="width:70%"/>
</center>

要注意的是 **Spinner** 新增了一個 `getDropDownView(int, View, VieGroup)` 的方法。這也是提供我們取得選單上選項的 View 的接口。

竟然我們想要暸解 Spinner 我們就專注在 SpinnerAdapter 的實作與運用即可。

## SpinnerAdapter

```java 
public interface SpinnerAdapter extends Adapter {
    public View getDropDownView(int position, View convertView, ViewGroup parent);
}
```

他共有 6 個實作者：
<center>
    <img src = "/images/posts/jekyll/android/spinner/spinneradapter_impl.png" style="width:70%"/>
</center>

|Impls|Description|
|:--|:--|
|**DropDownAdapter**|這是一個繼承 **ListAdapter** 與 **SpinnerAdapter** 的靜態類別。 <br><br>與其說是實作者，更應該稱它為 Wrapper。 因為他會把全部實作委推給 `SpinnerAdapter mAdapter` 與 `ListAdapter mListAdapter`。|
|**BaseAdapter**|這是一個繼承 **ListAdapter** 與 **SpinnerAdapter** 的抽象類別。而他還新增了通知監控者的功能 (`notifyDataSetChanged`, `notifyDataSetInvalidated` )。 |
|**ArrayAdapter**|他是一個繼承 **BaseAdapter** 並實作 **Filterable** 與 **ThemedSpinnerAdapter** 的類別。<br><br>我們可以在 **MaterialArrayAdapter** 與 **AlertController** 中找到他。|
|**SimpleAdapter**|他與 **ArrayAdapter** 繼承與實作相同的類別。 但 SimpleAdapter 缺乏對資料的更改功能，無法進行新增、插入、移除、排序等等的功能。|
|**CursorAdapter**|這是一個繼承 **BaseAdapter** 並實作 **Filterable** 的抽象類別。 <br><br>我們可以在 **AlertController** 找到他。 而他還有一個繼承者 (**ResourceCursorAdapter**) 並由 **SimpleCursorAdapter** 實作。<br><br>從名字就知道這個 Adapter 是配合 Database 或是 ContentProvider 來運作的。有機會會做個 demo 來展現他的作用。|
|**SuggestionAdapter**|這是實作 **ResourceCursorAdapter** 的類別。 我們可以在 **SearchView** 看到他的作用。|


以上的類別都可以使用在 **AbsSpinner** 內。 而 AbsSpinner 則會由 **Spinner**、 **Gallery** 與 **AppCompatSpinner** 實作。

他們實作 SpinnerAdapter 的方法分別是以下：

```java 
// DropDownAdapter
@Override
public View getDropDownView(int position, View convertView, ViewGroup parent) {
    // 直接耍廢，委託給 SpinnerAdapter mAdapter 執行
    // 當然這是為了更好的減少耦合性
    return (mAdapter == null) ? null
            : mAdapter.getDropDownView(position, convertView, parent);
}

// BaseAdapter 
public View getDropDownView(int position, View convertView, ViewGroup parent) {
    // 調用 Adapter 的 getView
    return getView(position, convertView, parent);
}


// ArrayAdapter
@Override
public View getDropDownView(int position, @Nullable View convertView,
        @NonNull ViewGroup parent) {
    
    // 通過 LayoutInflator 與參數來創建一個 TextView
    final LayoutInflater inflater = mDropDownInflater == null ? mInflater : mDropDownInflater;
    return createViewFromResource(inflater, position, convertView, parent, mDropDownResource);
}

// SimpleAdapter
@Override
public View getDropDownView(int position, View convertView, ViewGroup parent) {

    // 這裡的行為與 ArrayAdapter 一樣，但 SimpleAdapter 的 createViewFromResources 不僅僅可以創建出 TextView  
    // 還可以是 ImageView 或 擁有 Clickable 功能的 TextView
    final LayoutInflater inflater = mDropDownInflater == null ? mInflater : mDropDownInflater;
    return createViewFromResource(inflater, position, convertView, parent, mDropDownResource);
}

// CursorAdapter
@Override
public View getDropDownView(int position, View convertView, ViewGroup parent) {
    if (mDataValid) {
        mCursor.moveToPosition(position);
        View v;
        if (convertView == null) {
            
            // 這裡會由實作 CursorAdapter#newView(Context, Cursor, ViewGroup) 功能的類別創建想要的 View
            v = newDropDownView(mContext, mCursor, parent);
        } else {
            v = convertView;
        }

        // 另外， bindView 也需要被實作來正確展示資料
        bindView(v, mContext, mCursor);
        return v;
    } else {
        return null;
    }
}

// SuggestionAdapter
@Override
public View getDropDownView(int position, View convertView, ViewGroup parent) {
    try {

        // 也是直接耍廢，讓 CursorAdapter 執行
        return super.getDropDownView(position, convertView, parent);
    } catch (RuntimeException e) {
        Log.w(LOG_TAG, "Search suggestions cursor threw exception.", e);

        // 若有 exception 就創建 TextViw 並顯示出來
        // Put exception string in item title
        final View v = newDropDownView(mProviderContext, getCursor(), parent);
        if (v != null) {
            final ChildViewCache views = (ChildViewCache) v.getTag();
            final TextView tv = views.mText1;
            tv.setText(e.toString());
        }
        return v;
    }
}

```


在看 **Spinner** 之前，我們要先看 AbsSpinner 的父類別 **AdapterView<T>** 的結構與運作。

## AdapterView<T extends Adapter> 

**AdapterView** 是一個 **ViewGroup** 的抽象類別。

```java 
public abstract class AdapterView<T extends Adapter> extends ViewGroup
```

在建構子中，我們可以發現 AdapterView 只會做兩件事：
```java 
public AdapterView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
    super(context, attrs, defStyleAttr, defStyleRes);

    // 1. 確保 Accessibility 被開啟與
    // If not explicitly specified this view is important for accessibility.
    if (getImportantForAccessibility() == IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
        setImportantForAccessibility(IMPORTANT_FOR_ACCESSIBILITY_YES);
    }

    // 2. 在沒有 Adapter 之前，會確保不會被 Focus
    mDesiredFocusableState = getFocusable();
    if (mDesiredFocusableState == FOCUSABLE_AUTO) {
        // Starts off without an adapter, so NOT_FOCUSABLE by default.
        super.setFocusable(NOT_FOCUSABLE);
    }
}
```

除了建構子之外，內部有以下幾個重要的參數。 其中除了 **SelectionNotifier** 外都可以通過 setter/getter 設定與讀取：

|參數|功能|
|:--|:--|
|`View mEmptyView`|在沒有資料時顯示的 View|
|`OnItemSelectedListener mOnItemSelectedListener`<br>`OnItemClickListener mOnItemClickListener`<br>`OnItemLongClickListener mOnItemLongClickListener`|不同方式點擊物件時的監聽者|
|`SelectionNotifier mSelectionNotifier`|這是一個用來通知 selection event 的 **Runnable**|
|`SelectionNotifier mPendingSelectionNotifier`|這是正在等待被通知的 selection notifier|


以下是 **OnItemClickListner**、**OnItemLongClickListener** 與 **OnItemSelectedListener** 的介面：
```java 
public interface OnItemClickListener {
    void onItemClick(AdapterView<?> parent, View view, int position, long id);
}

public interface OnItemLongClickListener {
    boolean onItemLongClick(AdapterView<?> parent, View view, int position, long id);
}

public interface OnItemSelectedListener {
    // id - The row id of the item that is selected
    void onItemSelected(AdapterView<?> parent, View view, int position, long id);
    // Callback method to be invoked when the selection disappears from this view.
    // The selection can disappear for instance when touch is activated or when the adapter becomes empty.
    void onNothingSelected(AdapterView<?> parent);
}
```

雖說這是一個 ViewGroup 但他並不支援 `addView` 與 `removeView` 的功能。

由於 AdapterView 只是一個抽象類別，所以他的作用只是為了進行行為上的基礎設定。 而這些設定主要是與選取行為相關：

<details>
<summary  markdown='span'><code>handleDataChanged()</code> 被調用時會尋找之前被挑選的物件位置。 若出現選取或取消行為便會調用 <code>checkSelectionChanged()</code> 進行通知</summary>

```java 
void handleDataChanged() {
    final int count = mItemCount;
    boolean found = false;

    if (count > 0) {

        int newPos;

        // Find the row we are supposed to sync to
        if (mNeedSync) {
            // Update this first, since setNextSelectedPositionInt inspects
            // it
            mNeedSync = false;

            // See if we can find a position in the new data with the same
            // id as the old selection
            newPos = findSyncPosition();
            if (newPos >= 0) {
                // Verify that new selection is selectable
                int selectablePos = lookForSelectablePosition(newPos, true);
                if (selectablePos == newPos) {
                    // Same row id is selected
                    setNextSelectedPositionInt(newPos);
                    found = true;
                }
            }
        }
        if (!found) {
            // Try to use the same position if we can't find matching data
            newPos = getSelectedItemPosition();

            // Pin position to the available range
            if (newPos >= count) {
                newPos = count - 1;
            }
            if (newPos < 0) {
                newPos = 0;
            }

            // Make sure we select something selectable -- first look down
            int selectablePos = lookForSelectablePosition(newPos, true);
            if (selectablePos < 0) {
                // Looking down didn't work -- try looking up
                selectablePos = lookForSelectablePosition(newPos, false);
            }
            if (selectablePos >= 0) {
                setNextSelectedPositionInt(selectablePos);
                checkSelectionChanged();
                found = true;
            }
        }
    }
    if (!found) {
        // Nothing is selected
        mSelectedPosition = INVALID_POSITION;
        mSelectedRowId = INVALID_ROW_ID;
        mNextSelectedPosition = INVALID_POSITION;
        mNextSelectedRowId = INVALID_ROW_ID;
        mNeedSync = false;
        checkSelectionChanged();
    }

    notifySubtreeAccessibilityStateChangedIfNeeded();
}
```

</details>

<details>
<summary> <code>rememberSyncState()</code> - remember enough information to restore the screen state when the data has changed. </summary>

```java
void rememberSyncState() {
    if (getChildCount() > 0) {
        mNeedSync = true;
        mSyncHeight = mLayoutHeight;
        if (mSelectedPosition >= 0) {
            // Sync the selection state
            View v = getChildAt(mSelectedPosition - mFirstPosition);
            mSyncRowId = mNextSelectedRowId;
            mSyncPosition = mNextSelectedPosition;
            if (v != null) {
                mSpecificTop = v.getTop();
            }
            mSyncMode = SYNC_SELECTED_POSITION;
        } else {
            // Sync the based on the offset of the first view
            View v = getChildAt(0);
            T adapter = getAdapter();
            if (mFirstPosition >= 0 && mFirstPosition < adapter.getCount()) {
                mSyncRowId = adapter.getItemId(mFirstPosition);
            } else {
                mSyncRowId = NO_ID;
            }
            mSyncPosition = mFirstPosition;
            if (v != null) {
                mSpecificTop = v.getTop();
            }
            mSyncMode = SYNC_FIRST_POSITION;
        }
    }
}

```

</details>

<details>
<summary> 在 <code>rememberSyncState()</code> 中， 若有資料就會將 <code>mNeedSync</code> 設為 true。 而 <code>findSyncPosition()</code> 會在這個情況下被 <code>handleData()</code> 調用並用來尋找之前所選的選項。雖然說不是什麼特別的方法，但其中的演算法還是可以學習的。基本上就是 Bubble Search，但卻會隨機從任一位置開始往右尋找。 找不到再往左尋找。主要目的就是用機率的方式來加快尋找速度。 </summary>

```java
int findSyncPosition() {
    int count = mItemCount;

    if (count == 0) {
        return INVALID_POSITION;
    }

    long idToMatch = mSyncRowId;
    int seed = mSyncPosition;

    // If there isn't a selection don't hunt for it
    if (idToMatch == INVALID_ROW_ID) {
        return INVALID_POSITION;
    }

    // Pin seed to reasonable values
    // 確保 0 <= seed <= count - 1
    seed = Math.max(0, seed);
    seed = Math.min(count - 1, seed);

    long endTime = SystemClock.uptimeMillis() + SYNC_MAX_DURATION_MILLIS;

    long rowId;

    // first position scanned so far
    int first = seed;

    // last position scanned so far
    int last = seed;

    // True if we should move down on the next iteration
    boolean next = false;

    // True when we have looked at the first item in the data
    boolean hitFirst;

    // True when we have looked at the last item in the data
    boolean hitLast;

    // Get the item ID locally (instead of getItemIdAtPosition), so
    // we need the adapter
    T adapter = getAdapter();
    if (adapter == null) {
        return INVALID_POSITION;
    }

    while (SystemClock.uptimeMillis() <= endTime) {
        rowId = adapter.getItemId(seed);
        if (rowId == idToMatch) {
            // Found it!
            return seed;
        }

        hitLast = last == count - 1;
        hitFirst = first == 0;

        if (hitLast && hitFirst) {
            // Looked at everything
            break;
        }

        if (hitFirst || (next && !hitLast)) {
            // Either we hit the top, or we are trying to move down
            last++;
            seed = last;
            // Try going up next time
            next = false;
        } else if (hitLast || (!next && !hitFirst)) {
            // Either we hit the bottom, or we are trying to move up
            first--;
            seed = first;
            // Try going down next time
            next = true;
        }

    }

    return INVALID_POSITION;
}
```


</details>

<details>
<summary> <code>checkSelectionChanged()</code> 會通過隱藏的 <code>selectionChanged()</code> 進行立即通知 ( <code>dispatchOnItemSelected()</code> ) 或延後通知 ( <code>post(mSelectionNotifier)</code> ) </summary>

```java 
void checkSelectionChanged() {
    if ((mSelectedPosition != mOldSelectedPosition) || (mSelectedRowId != mOldSelectedRowId)) {
        selectionChanged();
        mOldSelectedPosition = mSelectedPosition;
        mOldSelectedRowId = mSelectedRowId;
    }

    // If we have a pending selection notification -- and we won't if we
    // just fired one in selectionChanged() -- run it now.
    if (mPendingSelectionNotifier != null) {
        mPendingSelectionNotifier.run();
    }
}

@UnsupportedAppUsage
void selectionChanged() {
    // We're about to post or run the selection notifier, so we don't need
    // a pending notifier.
    mPendingSelectionNotifier = null;

    if (mOnItemSelectedListener != null
            || AccessibilityManager.getInstance(mContext).isEnabled()) {
        if (mInLayout || mBlockLayoutRequests) {
            // If we are in a layout traversal, defer notification
            // by posting. This ensures that the view tree is
            // in a consistent state and is able to accommodate
            // new layout or invalidate requests.
            if (mSelectionNotifier == null) {
                mSelectionNotifier = new SelectionNotifier();
            } else {
                removeCallbacks(mSelectionNotifier);
            }
            post(mSelectionNotifier);
        } else {
            dispatchOnItemSelected();
        }
    }
    // Always notify AutoFillManager - it will return right away if autofill is disabled.
    final AutofillManager afm = mContext.getSystemService(AutofillManager.class);
    if (afm != null) {
        afm.notifyValueChanged(this);
    }
}

private class SelectionNotifier implements Runnable {
    public void run() {
        mPendingSelectionNotifier = null;

        if (mDataChanged && getViewRootImpl() != null
                && getViewRootImpl().isLayoutRequested()) {
            // Data has changed between when this SelectionNotifier was
            // posted and now. Postpone the notification until the next
            // layout is complete and we run checkSelectionChanged().
            if (getAdapter() != null) {
                mPendingSelectionNotifier = this;
            }
        } else {
            dispatchOnItemSelected();
        }
    }
}

// dispatchOnItemSelected 會通知 OnItemSelectedListener
// 若 selection == INVALID_POSITION == -1 就會通知 onNothingSelected.
// selection 其實就是 mNextSelectedPosition.
// mNextSelectedPosition 會在 onInvalidated() 或在 handleData 沒找到資料或沒資料時設為 INVALID_POSITION
private void dispatchOnItemSelected() {
    fireOnSelected();
    performAccessibilityActionsOnSelected();
}

private void fireOnSelected() {
    if (mOnItemSelectedListener == null) {
        return;
    }
    final int selection = getSelectedItemPosition();
    if (selection >= 0) {
        View v = getSelectedView();
        mOnItemSelectedListener.onItemSelected(this, v, selection,
                getAdapter().getItemId(selection));
    } else {
        mOnItemSelectedListener.onNothingSelected(this);
    }
}
```

</details>

現在我們大致上暸解 **AdapterView** 做了什麼。 接下來我們就要看看 **AbsSpinner** 了。

### AbsSpinner相較於 AdapterView， AbsSpinner 

AbsSpinner 是一個繼承 AdapterView 的抽象類別。一般來說，我們不會特地實作這個類別。

```java
public abstract class AbsSpinner extends AdapterView<SpinnerAdapter>
```

相較於 AdapterView， AbsSpinner 除了指定 `T:Adapter` 為 SpinnerAdapter， 還增加了 UI 相關的參數以及回收機制。

|參數|作用|
|`mHeightMeasureSpec`<br>`mWidthMeasureSpec`|在 `onMeasure` 中紀錄當下的 spec，僅此。|
|`mSelectionLeftPadding`<br>`mSelectionTopPadding`<br>`mSelectionRightPadding`<br>`mSelectionBottomPadding`|Paddings。 會在 `onMeasure` 中設定 `mSpinnerPadding` 的值。|
|`Rect mSpinnerPadding`|他的值都是 paddings，而這些值會在 `onMeasure` 中算出 `preferredHeight` 與 `preferredWidth`。|
|`RecycleBin mRecycler`|這是一個 AbsSpinner 內部的類別。 其主要目的是存取 View。|
|`DataSetObserver mDataSetObserver`|這個 DataSetObserver 會註冊在 Adapter 上。|
|`SpinnerAdapter mAdapter`|這個就是核心的 Adapter 了。 AbsSpinner 會通過他在 `onMeasure` 時 **創建** 新的 View。而 `mDataSetObserver` 也會通過他來監聽 `onInvalidated()` 與 `onChanged()` |

看 AbsSpinner 時，我們可以在建構子上看到他的預設：
```java
private void initAbsSpinner() {
    setFocusable(true); // 可被 Focus
    setWillNotDraw(false); // 這個 View 的 onDraw 會被調用，但 AbsSpinner 並沒由實作。
}
```

通過 `setWillNotDraw(Boolean)` View 會在 `setFlags(int flags, int mask)` 裡面定義是否要跳過 Draw :

```java
if ((changed & DRAW_MASK) != 0) {
    if ((mViewFlags & WILL_NOT_DRAW) != 0) {
        if (mBackground != null
                || mDefaultFocusHighlight != null
                || (mForegroundInfo != null && mForegroundInfo.mDrawable != null)) {

            // 若有背景、可被 Highlight 或有前景時，就會被強制需要畫
            mPrivateFlags &= ~PFLAG_SKIP_DRAW;
        } else {
            // 否則就跳過
            mPrivateFlags |= PFLAG_SKIP_DRAW;
        }
    } else {
        mPrivateFlags &= ~PFLAG_SKIP_DRAW;
    }
    requestLayout();
    invalidate(true);
}
```

通過以上的設定， `mPrivateFlags` 若設定為 `PFLAG_SKIP_DRAW` 變會在 `draw(@NonNull Canvas canvas, ViewGroup parent, long drawingTime)` 中調用 `dispatchDraw(Canvas)` 而非 `draw(Canvas)`。

- `dispatchDraw(Canvas)` 預設沒有行為。 他的主要功能是讓 View 可以控制他的 Children 如何被畫。而他的調用時間點是在 View 被畫出之後。 

- `draw(Canvas)` 會先調用 `onDraw(Canvas)` 來讓 View 決定要在 `dispatchDraw(Canvas)` 被調用前畫些什麼。他的預設也是沒有行為。



接下來，我們看看 `onMeasure` 吧。

<details>
<summary>其實 <code>onMeasure</code> 的實作並不複雜，因為他只會顯示被選取的物件。並不包括 popup。</summary>

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int widthMode = MeasureSpec.getMode(widthMeasureSpec);
    int widthSize;
    int heightSize;

    // 1. 設定 mSpinnerPadding
    mSpinnerPadding.left = mPaddingLeft > mSelectionLeftPadding ? mPaddingLeft
            : mSelectionLeftPadding;
    mSpinnerPadding.top = mPaddingTop > mSelectionTopPadding ? mPaddingTop
            : mSelectionTopPadding;
    mSpinnerPadding.right = mPaddingRight > mSelectionRightPadding ? mPaddingRight
            : mSelectionRightPadding;
    mSpinnerPadding.bottom = mPaddingBottom > mSelectionBottomPadding ? mPaddingBottom
            : mSelectionBottomPadding;

    // 2. mDataChanged == true 表示 onRestoreInstanceState 被執行，並且之前有選取 item
    //    此時就要調用 handleDataChanged
    if (mDataChanged) {
        handleDataChanged();
    }

    int preferredHeight = 0;
    int preferredWidth = 0;
    boolean needsMeasuring = true;

    // 3. 若選取的位置在合理範圍中，就會通過 RecycleBin 與 SpinnerAdapter 的合力創建及存取 View
    int selectedPosition = getSelectedItemPosition();
    if (selectedPosition >= 0 && mAdapter != null && selectedPosition < mAdapter.getCount()) {
        // Try looking in the recycler. (Maybe we were measured once already)
        View view = mRecycler.get(selectedPosition);
        if (view == null) {
            // Make a new one
            view = mAdapter.getView(selectedPosition, null, this);

            if (view.getImportantForAccessibility() == IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
                view.setImportantForAccessibility(IMPORTANT_FOR_ACCESSIBILITY_YES);
            }
        }

        if (view != null) {
            // Put in recycler for re-measuring and/or layout
            mRecycler.put(selectedPosition, view);

            // 4. 當 view 的 layoutParams 為 null，這表示他是剛被創建的
            //    所以我們需要設定他的 LayoutParams。
            //    在此之前，我們需要先把 mBlockLayoutRequests 設為 true 來防止 requestLayout 的調用。
            //    mBlockLayoutRequests 還會在選取新的物件時被設為 true。
            if (view.getLayoutParams() == null) {
                mBlockLayoutRequests = true;

                // 5. LayoutParams 預設為 MATCH_PARENT , WRAP_CONTENT
                view.setLayoutParams(generateDefaultLayoutParams());
                mBlockLayoutRequests = false;
            }

            // 6. 這會由 ViewGroup 預設行為來取得 view 的計算後的大小
            measureChild(view, widthMeasureSpec, heightMeasureSpec);

            // 7. 通過 padding 與 width, height 的配合，我們可以算出此 view 的適當大小
            preferredHeight = getChildHeight(view) + mSpinnerPadding.top + mSpinnerPadding.bottom;
            preferredWidth = getChildWidth(view) + mSpinnerPadding.left + mSpinnerPadding.right;

            needsMeasuring = false;
        }
    }

    if (needsMeasuring) {
        // No views -- just use padding
        preferredHeight = mSpinnerPadding.top + mSpinnerPadding.bottom;
        if (widthMode == MeasureSpec.UNSPECIFIED) {
            preferredWidth = mSpinnerPadding.left + mSpinnerPadding.right;
        }
    }

    preferredHeight = Math.max(preferredHeight, getSuggestedMinimumHeight());
    preferredWidth = Math.max(preferredWidth, getSuggestedMinimumWidth());

    heightSize = resolveSizeAndState(preferredHeight, heightMeasureSpec, 0);
    widthSize = resolveSizeAndState(preferredWidth, widthMeasureSpec, 0);

    setMeasuredDimension(widthSize, heightSize);
    mHeightMeasureSpec = heightMeasureSpec;
    mWidthMeasureSpec = widthMeasureSpec;
}
```

</details>

目前我們看到 AbsSpinner 對 UI 的作用，但其實他也會在得到 Adapter 時進行一些設置喔。

<details>
<summary>通過 <code>setAdapter(SpinnerAdapter)</code> AbsSpinner 會重新註冊 DataSetObserver，並重設 selected position。 最後再進行 <code>requestLayout()</code> 來啟動 <code>measure</code> 與 <code>draw</code></summary>

```java
@Override
public void setAdapter(SpinnerAdapter adapter) {
    if (null != mAdapter) {
        mAdapter.unregisterDataSetObserver(mDataSetObserver);
        resetList();
    }

    mAdapter = adapter;

    mOldSelectedPosition = INVALID_POSITION;
    mOldSelectedRowId = INVALID_ROW_ID;

    if (mAdapter != null) {
        mOldItemCount = mItemCount;
        mItemCount = mAdapter.getCount();
        checkFocus();

        mDataSetObserver = new AdapterDataSetObserver();
        mAdapter.registerDataSetObserver(mDataSetObserver);

        int position = mItemCount > 0 ? 0 : INVALID_POSITION;

        setSelectedPositionInt(position);
        setNextSelectedPositionInt(position);

        if (mItemCount == 0) {
            // Nothing selected
            checkSelectionChanged();
        }

    } else {
        checkFocus();
        resetList();
        // Nothing selected
        checkSelectionChanged();
    }

    requestLayout();
}
```

</details>


現在我們已經知道 AbsSpinner 的作用了，是時候看看 Spinner 了。



## Spinner 
```java
public class Spinner extends AbsSpinner implements OnClickListener 
``` 

目前前面兩者都各有處理的事項，包括資料的存取與 UI 的更新。 但 Spinner 的功能最主要就是顯示選單上的選項。為此， Spinner 提供了兩種選單的顯示模式 (`MODE_DIALOG` 或 `MODE_DROPDOWN`)。 根據不同的顯示模式 Spinner 需要用到不同的類別與參數。

我們簡單地介紹一下 Spinner 中的主要參數、類別與功能吧。

|參數|功能
|:--|:--|
|`Context mPopupContext`|這是 dialog 或 popup 進行 inflation 所需要的內容 |
|`ForwardingListener mForwardingListener`|這是一個繼承 **View.OnTouchListener** 與 **View.OnAttachStateChangeListener** 的抽象類別。<br><br>他的主要功能是接收 Spinner 的 `onTouch(Event)` 並按情況決定是否要處理此 Event。 我之後會講解一下。 |
|`SpinnerPopup mPopup`|這是一個負責顯示選單的介面。通過這個介面，Spinner 實現了 **DialogPopup** 與 **DropdownPopup**。由於他的實作很值得去了解，所以會在下一個章節特別研究。 |



### ForwardingListener
```java
public abstract class ForwardingListener
        implements View.OnTouchListener, View.OnAttachStateChangeListener
```

這個 Listener 會通過 `onTouch(View, Event)` 來決定是否要將此 Event 傳遞下去：

<center>
    <img src = "/images/posts/jekyll/android/spinner/spinner_forwardinglistener_ontouch_action.png" style = "width:70%"/>
</center>

從流程便可以知道其實就知道 forward 與否取決於 Popup 是否有顯示或將 event 處理掉。


### Spinner 建構子
我們要看一下 Spinner 變數的預設，這就需要從建構子中看了：

```java
public Spinner(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes, int mode,
        Theme popupTheme) {

    // 1. defStyleAttr 預設為 com.android.internal.R.attr.spinnerStyle
    super(context, attrs, defStyleAttr, defStyleRes);

    final TypedArray a = context.obtainStyledAttributes(
            attrs, R.styleable.Spinner, defStyleAttr, defStyleRes);
    
    saveAttributeDataForStyleable(context, R.styleable.Spinner,
            attrs, a, defStyleAttr, defStyleRes);

    // 2. 創建 mPopupContext
    if (popupTheme != null) {
        mPopupContext = new ContextThemeWrapper(context, popupTheme);
    } else {
        final int popupThemeResId = a.getResourceId(R.styleable.Spinner_popupTheme, 0);
        if (popupThemeResId != 0) {
            mPopupContext = new ContextThemeWrapper(context, popupThemeResId);
        } else {
            mPopupContext = context;
        }
    }

    // 3. 預設 mode = MODE_THEME，所以預設 Spinner 會以 Dialog 方式顯示 popup
    if (mode == MODE_THEME) {
        mode = a.getInt(R.styleable.Spinner_spinnerMode, MODE_DIALOG);
    }

    // 4. 創建對應的 mPopup
    switch (mode) {
        case MODE_DIALOG: {
            mPopup = new DialogPopup();
            mPopup.setPromptText(a.getString(R.styleable.Spinner_prompt));
            break;
        }

        case MODE_DROPDOWN: {

            // 5. 這裡會創建 DropdownPopup
            //      其中的 mDropDownWidth 是 WRAP_CONTENT
            //      
            final DropdownPopup popup = new DropdownPopup(
                    mPopupContext, attrs, defStyleAttr, defStyleRes);
            final TypedArray pa = mPopupContext.obtainStyledAttributes(
                    attrs, R.styleable.Spinner, defStyleAttr, defStyleRes);
            mDropDownWidth = pa.getLayoutDimension(R.styleable.Spinner_dropDownWidth,
                    ViewGroup.LayoutParams.WRAP_CONTENT);
            if (pa.hasValueOrEmpty(R.styleable.Spinner_dropDownSelector)) {
                popup.setListSelector(pa.getDrawable(
                        R.styleable.Spinner_dropDownSelector));
            }
            popup.setBackgroundDrawable(pa.getDrawable(R.styleable.Spinner_popupBackground));
            popup.setPromptText(a.getString(R.styleable.Spinner_prompt));
            pa.recycle();

            mPopup = popup;
            mForwardingListener = new ForwardingListener(this) {
                @Override
                public ShowableListMenu getPopup() {
                    return popup;
                }

                @Override
                public boolean onForwardingStarted() {
                    if (!mPopup.isShowing()) {
                        mPopup.show(getTextDirection(), getTextAlignment());
                    }
                    return true;
                }
            };
            break;
        }
    }

    mGravity = a.getInt(R.styleable.Spinner_gravity, Gravity.CENTER);
    mDisableChildrenWhenDisabled = a.getBoolean(
            R.styleable.Spinner_disableChildrenWhenDisabled, false);

    a.recycle();

    // Base constructor can call setAdapter before we initialize mPopup.
    // Finish setting things up if this happened.
    if (mTempAdapter != null) {
        setAdapter(mTempAdapter);
        mTempAdapter = null;
    }
}
```





## SpinnerPopup

就如之前所說， **SpinnerPopup** 是一個負責顯示 popup 的介面。 想要知道如何實作就需要看看 **DropdownPopup** 與 **DialogPopup** 了。

```java
private interface SpinnerPopup {
    public void setAdapter(ListAdapter adapter);
    public void show(int textDirection, int textAlignment);
    public void dismiss();

    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
    public boolean isShowing();

    public void setPromptText(CharSequence hintText);
    public CharSequence getHintText();

    public void setBackgroundDrawable(Drawable bg);
    public void setVerticalOffset(int px);
    public void setHorizontalOffset(int px);
    public Drawable getBackground();
    public int getVerticalOffset();
    public int getHorizontalOffset();
}
```

### DialogPopup

```java
private class DialogPopup implements SpinnerPopup, DialogInterface.OnClickListener
```

DialogPopup 在調用 show 時會直接顯示一個 **AlertDialog** 。 也因為如此，他在以下方法只需要回傳預設行為或值即可：
```java
public void setBackgroundDrawable(Drawable bg);
public void setVerticalOffset(int px);
public void setHorizontalOffset(int px);
public Drawable getBackground();
public int getVerticalOffset();
public int getHorizontalOffset();
```

重點在 `show()` 與 `onClick(DialogInterface, int)`。 通過 `show`，一個擁有 ListView 的 AlertDialog 便可被展現出來：
```java
public void show(int textDirection, int textAlignment) {
    if (mListAdapter == null) {
        return;
    }
    AlertDialog.Builder builder = new AlertDialog.Builder(getPopupContext());
    if (mPrompt != null) {
        builder.setTitle(mPrompt);
    }
    // 預設為 SingleChoiceItems，並將 ListAdapter 傳給 AlertController
    mPopup = builder.setSingleChoiceItems(mListAdapter,
            getSelectedItemPosition(), this).create();

    // 有了 ListAdapter，AlertController 便會創建一個 RecycleListView: ListView
    // 其實他就是一個 ListView 並沒看到實作任何回收機制。
    final ListView listView = mPopup.getListView();

    // 進行 Text 顯示的設定
    listView.setTextDirection(textDirection);
    listView.setTextAlignment(textAlignment);
    mPopup.show();
}

public void onClick(DialogInterface dialog, int which) {
    // 保存被挑選者。 這是由 AbsSpinner 實現的。
    setSelection(which);
    
    // 這裡會傳回給 AdapterView 來處理。
    // 若 AdapterView 中有 OnItemClickListener 那就會啟動音效並調用他的  
    // OnItemClickListener (AdapterView<?> parent, View view, int position, long id)
    if (mOnItemClickListener != null) {
        performItemClick(null, which, mListAdapter.getItemId(which));
    }
    dismiss();
}
```

### DropdownPopup
```java
private class DropdownPopup extends ListPopupWindow implements SpinnerPopup
```

簡簡單單的一行其實含金量很多。 DropdownPopup 其實是由兩個類別組成的：
|類別|功能|
|:--|:--|
|**PopupWindow**| 他的主要功能就是客製化並顯示 popup。 而客製化則包括以下：<br>-動畫<br>-anchor 點<br>-背景<br>-ContentView<br>-Elevate<br>等等...|
|**ListPopupWindow**|這裡包含三個重要參數： **DropDownListView**, |
|**DropdownPopup**||






# Spinner 的顯示流程

## Spinner 的創建
當畫面被創建時， Android 會通過 xml 知道 Spinner 需要被創建，然後就會經過以下流程：

<center>
    <img src = "/images/posts/jekyll/android/spinner/uml_createview_spinner.png" />
</center>

當我們在創建時，我們可以通過 **AdapterCompatSpinner** 或 **Spinner** 的 `setAdapter(SpinnerAdapter)` 將 Adapter 傳入。

傳入的時候， 這個 Adapter 會在不同 Spinner 類別進行不同的行為。
- **AbsSpinner** 中，他會對這個 Adapter 進行 `registerDataSetObserver`
- **Spinner** 中，他先傳給 **AbsSpinner**。 之後會創建一個 **Spinner.DropDownAdapter** :
    ```java
    private static class DropDownAdapter implements ListAdapter, SpinnerAdapter {
        private SpinnerAdapter mAdapter;
        private ListAdapter mListAdapter;
        // ... rest of the code
    }
    ```
    <br>
    在創建之後，他會先被傳入 **ListPopupWindow** 進行 `registerDataObserver`，再設為 **DropdownPopup** 的 `mAdapter`。 
    
    ```java
    private class DropdownPopup extends ListPopupWindow implements SpinnerPopup {
        private CharSequence mHintText;
        private ListAdapter mAdapter;
    }
    ```
- **AppCompatSpinner** 中，他會先傳給 **Spinner**。 之後會創建一個 **AppCompatSpinner.DropDownAdapter** ：
    ```java
    private static class DropDownAdapter implements ListAdapter, SpinnerAdapter {
        private SpinnerAdapter mAdapter;
        private ListAdapter mListAdapter;
    }

    @VisibleForTesting
    class DropdownPopup extends ListPopupWindow implements SpinnerPopup {
        private CharSequence mHintText;
        ListAdapter mAdapter;
        private final Rect mVisibleRect = new Rect();
        private int mOriginalHorizontalOffset;
    }
    ```

    基本上與 Spinner 一樣。 

從他們的行為我們可以看出， AppCompatSpinner 與 Spinner 雖然共用一個 Adapter，但他們各自會有自己的 **DropdownPopup**。 

## Spinner 的顯示

我將 `onMeaure` 、 `onLayout` 與 `onDraw` 的流程分開來看。

首先就是 `onMeasure`：

<center>
    <img src = "/images/posts/jekyll/android/spinner/uml_spinner_measure.png" />
</center>

從流程你可以看見他與 itemView 的 height 相比，更注重 itemView 的 width。 而 Spinner 也會從 **SpinnerAdapter** 取得 itemView 來進行測量 。 

值得注意的是，在 Measure 流程中， **Spinner** 會在 `makeView()` 中通過我們傳入的 Adapter，調用他的 `mAdapter.getView(selectedPosition, null, this);` 來取得第一個 itemView：

```kotlin 
object : ArrayAdapter<String?>(this, android.R.layout.simple_spinner_item, resources.getStringArray(R.array.planets_array)) {

    // 這個方法主要是影響 width 的計算
    override fun getView(position: Int, convertView: View?, parent: ViewGroup): View {
        return super.getView(position, convertView, parent)
    }
}
```

這個 itemView 主要是用來取得 baseline。

進入 onMeasure 後，Spinner 會通過多次的 `measureContentWidth(SpinnerAdapter, Drawable)` 調用跟 Adapter 取得 selectedItemPosition 至 end 的全部 itemView 的 **最大寬度** 來設定 Spinner 的顯示寬度。

但這還沒完，之後 **AppCompatSpinner** 還會在 `onMeasure` 中通過 `compatMeasureContentWidth` 進行相同行為。

從兩者的 `onMeasure` 可以看出其實行為是一樣的：
```java
// Spinner 
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    if (mPopup != null && MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.AT_MOST) {
        final int measuredWidth = getMeasuredWidth();
        setMeasuredDimension(
            Math.min(Math.max(measuredWidth,
                              measureContentWidth(getAdapter(), getBackground())),
                     MeasureSpec.getSize(widthMeasureSpec)),
           getMeasuredHeight());
    }
}

// AppCompatSpinner
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);

    if (mPopup != null && MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.AT_MOST) {
        final int measuredWidth = getMeasuredWidth();
        setMeasuredDimension(
            Math.min(Math.max(measuredWidth,
                              compatMeasureContentWidth(getAdapter(), getBackground())),
                     MeasureSpec.getSize(widthMeasureSpec)),
            getMeasuredHeight());
    }
}
```

<details>
<summary>相同的，我們來比較 Spinner 的 `measureContentWidth` 與 AppCompatSpinner 的 `compatMeasureContentWidth` 也會發現是一模一樣。</summary>

```java


// Spinner 
int measureContentWidth(SpinnerAdapter adapter, Drawable background) {
    if (adapter == null) {
        return 0;
    }

    int width = 0;
    View itemView = null;
    int itemType = 0;
    final int widthMeasureSpec =
        MeasureSpec.makeSafeMeasureSpec(getMeasuredWidth(), MeasureSpec.UNSPECIFIED);
    final int heightMeasureSpec =
        MeasureSpec.makeSafeMeasureSpec(getMeasuredHeight(), MeasureSpec.UNSPECIFIED);

    // Make sure the number of items we'll measure is capped. If it's a huge data set
    // with wildly varying sizes, oh well.
    int start = Math.max(0, getSelectedItemPosition());
    final int end = Math.min(adapter.getCount(), start + MAX_ITEMS_MEASURED);
    final int count = end - start;
    start = Math.max(0, start - (MAX_ITEMS_MEASURED - count));
    for (int i = start; i < end; i++) {
        final int positionType = adapter.getItemViewType(i);
        if (positionType != itemType) {
            itemType = positionType;
            itemView = null;
        }
        itemView = adapter.getView(i, itemView, this);
        if (itemView.getLayoutParams() == null) {
            itemView.setLayoutParams(new ViewGroup.LayoutParams(
                    ViewGroup.LayoutParams.WRAP_CONTENT,
                    ViewGroup.LayoutParams.WRAP_CONTENT));
        }
        itemView.measure(widthMeasureSpec, heightMeasureSpec);
        width = Math.max(width, itemView.getMeasuredWidth());
    }

    // Add background padding to measured width
    if (background != null) {
        background.getPadding(mTempRect);
        width += mTempRect.left + mTempRect.right;
    }

    return width;
}


// AppCompatSpinner
int compatMeasureContentWidth(SpinnerAdapter adapter, Drawable background) {
    if (adapter == null) {
        return 0;
    }

    int width = 0;
    View itemView = null;
    int itemType = 0;
    final int widthMeasureSpec =
            MeasureSpec.makeMeasureSpec(getMeasuredWidth(), MeasureSpec.UNSPECIFIED);
    final int heightMeasureSpec =
            MeasureSpec.makeMeasureSpec(getMeasuredHeight(), MeasureSpec.UNSPECIFIED);

    // Make sure the number of items we'll measure is capped. If it's a huge data set
    // with wildly varying sizes, oh well.
    int start = Math.max(0, getSelectedItemPosition());
    final int end = Math.min(adapter.getCount(), start + MAX_ITEMS_MEASURED);
    final int count = end - start;
    start = Math.max(0, start - (MAX_ITEMS_MEASURED - count));
    for (int i = start; i < end; i++) {
        final int positionType = adapter.getItemViewType(i);
        if (positionType != itemType) {
            itemType = positionType;
            itemView = null;
        }
        itemView = adapter.getView(i, itemView, this);
        if (itemView.getLayoutParams() == null) {
            itemView.setLayoutParams(new LayoutParams(
                    LayoutParams.WRAP_CONTENT,
                    LayoutParams.WRAP_CONTENT));
        }
        itemView.measure(widthMeasureSpec, heightMeasureSpec);
        width = Math.max(width, itemView.getMeasuredWidth());
    }

    // Add background padding to measured width
    if (background != null) {
        background.getPadding(mTempRect);
        width += mTempRect.left + mTempRect.right;
    }

    return width;
}
```

</details>

<details>
<summary>當然， **AbsSpinner** 的 `onMeasure` 也是有類似行為。 但他會處理 padding，並且會確保資料變更已完畢才計算寬與高度。</summary>

```java
// AbsSpinner
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int widthMode = MeasureSpec.getMode(widthMeasureSpec);
    int widthSize;
    int heightSize;

    mSpinnerPadding.left = mPaddingLeft > mSelectionLeftPadding ? mPaddingLeft
            : mSelectionLeftPadding;
    mSpinnerPadding.top = mPaddingTop > mSelectionTopPadding ? mPaddingTop
            : mSelectionTopPadding;
    mSpinnerPadding.right = mPaddingRight > mSelectionRightPadding ? mPaddingRight
            : mSelectionRightPadding;
    mSpinnerPadding.bottom = mPaddingBottom > mSelectionBottomPadding ? mPaddingBottom
            : mSelectionBottomPadding;

    if (mDataChanged) {
        handleDataChanged();
    }

    int preferredHeight = 0;
    int preferredWidth = 0;
    boolean needsMeasuring = true;

    int selectedPosition = getSelectedItemPosition();
    if (selectedPosition >= 0 && mAdapter != null && selectedPosition < mAdapter.getCount()) {
        // Try looking in the recycler. (Maybe we were measured once already)
        View view = mRecycler.get(selectedPosition);
        if (view == null) {
            // Make a new one
            view = mAdapter.getView(selectedPosition, null, this);

            if (view.getImportantForAccessibility() == IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
                view.setImportantForAccessibility(IMPORTANT_FOR_ACCESSIBILITY_YES);
            }
        }

        if (view != null) {
            // Put in recycler for re-measuring and/or layout
            mRecycler.put(selectedPosition, view);

            if (view.getLayoutParams() == null) {
                mBlockLayoutRequests = true;
                view.setLayoutParams(generateDefaultLayoutParams());
                mBlockLayoutRequests = false;
            }
            measureChild(view, widthMeasureSpec, heightMeasureSpec);

            preferredHeight = getChildHeight(view) + mSpinnerPadding.top + mSpinnerPadding.bottom;
            preferredWidth = getChildWidth(view) + mSpinnerPadding.left + mSpinnerPadding.right;

            needsMeasuring = false;
        }
    }

    if (needsMeasuring) {
        // No views -- just use padding
        preferredHeight = mSpinnerPadding.top + mSpinnerPadding.bottom;
        if (widthMode == MeasureSpec.UNSPECIFIED) {
            preferredWidth = mSpinnerPadding.left + mSpinnerPadding.right;
        }
    }

    preferredHeight = Math.max(preferredHeight, getSuggestedMinimumHeight());
    preferredWidth = Math.max(preferredWidth, getSuggestedMinimumWidth());

    heightSize = resolveSizeAndState(preferredHeight, heightMeasureSpec, 0);
    widthSize = resolveSizeAndState(preferredWidth, widthMeasureSpec, 0);

    setMeasuredDimension(widthSize, heightSize);
    mHeightMeasureSpec = heightMeasureSpec;
    mWidthMeasureSpec = widthMeasureSpec;
}
```

</details>

<br>

再來就是進行 `onLayout`：
<center>
    <img src = "/images/posts/jekyll/android/spinner/uml_spinner_layout.png" />
</center>

`onLayout` 這裡 Spinner 只會顯示被選中的 View。 也是因為如此， `onMeasure` 中會比較介意 itemView 的寬度。

由於通過以上流程， childView 的所在位置都已經確定了。 所以無需特別進行 `onDraw`。

在 `onLayout` 流程中， Spinner 會在 `layout` 中通過 `makeView` 從 `Adapter.getView` 取得 `mSelectedPosition` 的 itemView。 當然， AppCompatSpinner 也會再次重複這個行為。

除了找出目前被挑選的 itemView 外，在這個流程中還會多次調用 `handleDataChanged()` 與 `checkSelectionChanged()` 來確定 selection 或 資料是否有更新。 一旦資料更新， Spinner 會通過 **OnItemSelectedListener** 與 **SelectionNotifier** 進行通知。

因為在不知道誰被選取時，預設都會是 0 ，所以我們第一次收到的就是 item position == 0。


## Spinner 點擊行為
現在 Spinner 已被顯示，接下來就看看他被點擊後的行為。

<center>
    <img src = "/images/posts/jekyll/android/spinner/uml_spinner_show.png" />
</center>

我們可以看見當我們點選 Spinner 後， Spinner 會通過 `SpinnerPopup mPopup` ，也就是在創建時創建的 **DropdownPopup** ，算出最大寬度並設定 content width。

在設定好後， **ListPopupWindow** 會負責計算出 popup 的 高度。在計算高度時， ListPopupWindow 會創建並設定 **DropdownListView**。而高度的計算有以下考量：
- DropdownListView 的 list content
- hint view 
- background

算好高度後，ListPopupWindow 還會查看是否有 Input Method， 並重新設定 **AppCompatPopupWindow** 的 hight 與 width spec。

最後就會通過 `showAsDropDown` 將 AppCommpatPopupWindow 顯示出來：

```java 
PopupWindowCompat.showAsDropDown(mPopup, getAnchorView(), mDropDownHorizontalOffset,    
    mDropDownVerticalOffset, mDropDownGravity);
```


也就是說，這個流程包含了以下重要的角色：
- **DropdownPopup** : 負責從 **SpinnerAdapter** 中取得 itemView 並計算出最大寬度
- **ListPopupWindow** : 負責畫面的建設，包括創造與設定 **DropdownListView**
- **DropdownListView** : 負責互動的行為
- **PopupWindow** : 負責最後將 ListPopupWindow 顯示在 Window 上

這些物件的關係如下：
- **ListPopupWindow** 會包含一個 PopupWindow 以及一個 DropdownListView。
- **Spinner** 則會有一個 DropdownPopup，一個 ListPopupWindow 的子類別。 而 DropdownPopup 會將互動行為委託 Adpaters 來執行。


統整一下，他們的關係如下：

<center>
    <img src = "/images/posts/jekyll/android/spinner/spinner_popupwindow_adapter.png" />
</center>


## Spinner 點選的行為

此時 Spinner 已被展開，這裡我們看看當我們點選 itemView 會發生什麼事吧。

<center>
    <img src = "/images/posts/jekyll/android/spinner/uml_spinner_select.png" />
</center>

這裡的流程很簡單，當我們 touch 時，他會通過 **PopupWindow.PopupDecorView** 來判斷是否在 DecorView 之內。 若不在之內，就會調用 `dismiss()` 關閉 PopupWindow。 若在之內，就會傳給 PopupDecorView 的 content，也就是 **PopupWindow.PopupBackgroundView**。 

但 PopupBackgroundView 只有覆寫了 `onCreateDrawableState(int)` ，其他都是 FrameLayout 的預設行為。所以最後會將 touch 拋給 **DropdownListView** 和 **AbsListView** 來處理。

如果只看這流程表，也許你會以為點選的行為會由 **DropdownPopup.OnItemClickListener** 傳遞回我們設定的 OnItemClickListener。 但其實這是錯誤的。

當我們手指按下時， AbsListView 會通過 `onInterceptTouchEvent` 進行攔截並更新 `mMotionPosition` ：
```java
int motionPosition = findMotionRow(y);

// ListView
@Override
int findMotionRow(int y) {
    int childCount = getChildCount();
    if (childCount > 0) {
        if (!mStackFromBottom) {
            for (int i = 0; i < childCount; i++) {
                View v = getChildAt(i);
                if (y <= v.getBottom()) {
                    return mFirstPosition + i;
                }
            }
        } else {
            for (int i = childCount - 1; i >= 0; i--) {
                View v = getChildAt(i);
                if (y >= v.getTop()) {
                    return mFirstPosition + i;
                }
            }
        }
    }
    return INVALID_POSITION;
}
```



之後，AbsListView 會在 `onTouchDown` 中創建一個 **CheckForTab** 的 **Runnable** 來處理 LongPress 的行為 :

```java
private final class CheckForTap implements Runnable {
    float x;
    float y;

    @Override
    public void run() {
        if (mTouchMode == TOUCH_MODE_DOWN) {
            mTouchMode = TOUCH_MODE_TAP;
            final View child = getChildAt(mMotionPosition - mFirstPosition);
            if (child != null && !child.hasExplicitFocusable()) {
                mLayoutMode = LAYOUT_NORMAL;

                if (!mDataChanged) {
                    final float[] point = mTmpPoint;
                    point[0] = x;
                    point[1] = y;
                    transformPointToViewLocal(point, child);
                    child.drawableHotspotChanged(point[0], point[1]);
                    child.setPressed(true);
                    setPressed(true);
                    layoutChildren();
                    positionSelector(mMotionPosition, child);
                    refreshDrawableState();

                    final int longPressTimeout = ViewConfiguration.getLongPressTimeout();
                    final boolean longClickable = isLongClickable();

                    if (mSelector != null) {
                        final Drawable d = mSelector.getCurrent();
                        if (d != null && d instanceof TransitionDrawable) {
                            if (longClickable) {
                                ((TransitionDrawable) d).startTransition(longPressTimeout);
                            } else {
                                ((TransitionDrawable) d).resetTransition();
                            }
                        }
                        mSelector.setHotspot(x, y);
                    }

                    if (longClickable) {
                        if (mPendingCheckForLongPress == null) {
                            mPendingCheckForLongPress = new CheckForLongPress();
                        }
                        mPendingCheckForLongPress.setCoords(x, y);
                        mPendingCheckForLongPress.rememberWindowAttachCount();
                        postDelayed(mPendingCheckForLongPress, longPressTimeout);
                    } else {
                        mTouchMode = TOUCH_MODE_DONE_WAITING;
                    }
                } else {
                    mTouchMode = TOUCH_MODE_DONE_WAITING;
                }
            }
        }
    }
}
```

同時，他也會在 `onTouchDown` 中更新 `mMotionPosition`。

當我們手指起來時，他會先調用 `AbsListView.onTouchUp`。 這方法中，他會創建一個 `mTouchModeReset` 的 Runnable 來負責點選的行為：

```java
mTouchModeReset = new Runnable() {
    @Override
    public void run() {
        mTouchModeReset = null;
        mTouchMode = TOUCH_MODE_REST;
        child.setPressed(false);
        setPressed(false);
        if (!mDataChanged && !mIsDetaching
                && isAttachedToWindow()) {
            performClick.run();
        }
    }
};
postDelayed(mTouchModeReset,
        ViewConfiguration.getPressedStateDuration());
```

當 `run()` 被調用時，他會進行 `performClick.run()`：

```java
private class PerformClick extends WindowRunnnable implements Runnable {
    int mClickMotionPosition;

    @Override
    public void run() {
        // The data has changed since we posted this action in the event queue,
        // bail out before bad things happen
        if (mDataChanged) return;

        final ListAdapter adapter = mAdapter;
        final int motionPosition = mClickMotionPosition;
        if (adapter != null && mItemCount > 0 &&
                motionPosition != INVALID_POSITION &&
                motionPosition < adapter.getCount() && sameWindow() &&
                adapter.isEnabled(motionPosition)) {
            final View view = getChildAt(motionPosition - mFirstPosition);
            // If there is no view, something bad happened (the view scrolled off the
            // screen, etc.) and we should cancel the click
            if (view != null) {
                performItemClick(view, motionPosition, adapter.getItemId(motionPosition));
            }
        }
    }
}
```

重點在 **PerformClick** 中的 `int mClickMotionPosition`。 這個參數會在 `onTouchUp` 時進行更新 :

```java
final AbsListView.PerformClick performClick = mPerformClick;
performClick.mClickMotionPosition = motionPosition;
performClick.rememberWindowAttachCount();
```

也因為如此，所以最後 PerformClick 被進行時，才可以將挑選的 index 傳出去。


以下便是如何將 selected position 傳遞出去的流程：

<center>
    <img src = "/images/posts/jekyll/android/spinner/uml_performitemclick_dismiss.png" />
</center>


最後 Spinner 會在 `layout` 時，由於 `mSelectedPosition` 更新了，所以便會通過 `AdapterView.selectionChanged` 通知 **SelectionNotifier** 調用 `dispatchOnItemSelected` 進而通知我們設定的 OnItemSelectedListener。

```kotlin 
override fun onItemSelected(parent: AdapterView<*>?, view: View?, position: Int, id: Long) {
    selectedPosition = position
    Toast.makeText(this, "Selected ${parent?.selectedItem}", Toast.LENGTH_SHORT).show()
}
```


# FAQ
## 如何在點擊選項時顯示不同的顏色？

如果我們看 [官方的資料](https://developer.android.com/reference/android/widget/Spinner#attr_android:dropDownSelector) 你會想說使用 `android:dropDownSelector` 應該是可行的。 

但你會錯得非常離譜。 當然，錯不在你，而是 Google。 這個屬性從 2012 年就存在了，但這個 bug 也從未離開過。

之所以會如此，我們就要暸解 [Spinner 顯示的流程](#spinner-的顯示流程)了。

我先給急著想要答案的各位一個解答吧。 我們所要做的就是在 `theme.xml` 中定義客製化 theme 並將他指定為 `android:dropDownListViewStyle`:

```groovy
<style name="Base.Theme.SpinnerDemo" parent="Theme.Material3.DayNight.NoActionBar">
    <item name="android:dropDownListViewStyle">@style/My.Theme.Spinner</item>
</style>

<style name="My.Theme.Spinner" parent= "android:Widget.ListView">
    <item name="android:divider">@null</item>
    <item name="android:listSelector">@drawable/spinner_selector</item>
</style>
```

接下來就是 [解釋](https://stackoverflow.com/a/77414896/18597115) 了。

首先，當我們在 xml 中設定 `android:dropDownSelector` 時，這個設定的確會通過 `mDropDownList.setSelector(Drawable)` 傳給 **DropdownPopup** 。 記住，是 **DropdownPopup**。

但是當我們按下 Spinner，他會通過以下流程創建 **DropDownListView** ：

<center>
    <img src = "/images/posts/jekyll/android/spinner/uml_show_dropdownlistview_init.png" style = "width:70%"/>
</center>

而 **DropDownListView** 才是會影響選項被點選時的反應。

從他的建構子可以看出他會帶入一個 defStyleAttr `R.attr.dropDownListViewStyle`：
```java
DropDownListView(@NonNull Context context, boolean hijackFocus) {
    super(context, null, R.attr.dropDownListViewStyle);
    mHijackFocus = hijackFocus;
    setCacheColorHint(0); // Transparent, since the background drawable could be anything.
}
```

後來在建構 **AbsListView** 時便會從這個 defStyleAttr 取得 `selector` :
```java
final Drawable selector = a.getDrawable(R.styleable.AbsListView_listSelector);
if (selector != null) {
    setSelector(selector);
}
```

而按照我此時所使用的主題， `Theme.Material3.DayNight.NoActionBar`，通過順籐摸瓜，我們便可找到他的父類別中的 `dropDownListViewStyle` 的定義了：

```groovy
<style name="Base.V7.Theme.AppCompat.Light" parent="Platform.AppCompat.Light">
    <!-- 其他屬性 -->
    <item name="dropDownListViewStyle">?android:attr/dropDownListViewStyle</item>
    <!-- 其他屬性 -->
</style>
```

`?android:attr/dropDownListViewStyle` 表示使用 Android build-in 的 attr：
```groovy
<style name="Theme.Holo.Light" parent="Theme.Light">
    <!-- ... -->
    <item name="dropDownListViewStyle">@style/Widget.Holo.ListView.DropvDown</item>
    <item name="listChoiceBackgroundIndicator">@drawable/list_selector_holo_light</item>
</style>
```

繼續追下去就會找到：
```groovy
<style name="Widget.Holo.ListView" parent="Widget.ListView">
    <item name="divider">?attr/listDivider</item>
    <item name="listSelector">?attr/listChoiceBackgroundIndicator</item>
</style>
```


## 如何在顯示 Spinner Dropdown 時標出上次的選項？

這個很簡單，我們只需要定義 **AdapterView.OnItemSelectedListener** 與 覆寫 Adapter 的 `getDropDownView(position: Int, convertView: View?, parent: ViewGroup)`。 

之所以使用這兩者即可是因為他的職責如下：
- `getDropDownView` : 定義 DropdownListView 中的 itemView 並回傳。 這是在顯示前最後客製化的時機。
- **OnItemSelectedListener** : 這會在 Spinner 重新 layout 時調用，並將當下 selected position 傳回。

範例：

```kotlin

object : ArrayAdapter<String?>(this, android.R.layout.simple_spinner_item, resources.getStringArray(R.array.planets_array)) {
    override fun getDropDownView(
        position: Int,
        convertView: View?,
        parent: ViewGroup
    ): View {

        val v =  super.getDropDownView(position, convertView, parent)

        // 進行標記
        if (position == selectedPosition) {
            v.setBackgroundColor(ResourcesCompat.getColor(resources, android.R.color.holo_blue_dark, theme))
        }

        return v
    }
}.also {
    it.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item)
    spinner.adapter = it
}

// OnItemSelectedListener
override fun onItemSelected(parent: AdapterView<*>?, view: View?, position: Int, id: Long) {
    selectedPosition = position
    Toast.makeText(this, "Selected ${parent?.selectedItem}", Toast.LENGTH_SHORT).show()
}

override fun onNothingSelected(parent: AdapterView<*>?) {
    Toast.makeText(this, "NothingSelected", Toast.LENGTH_SHORT).show()
}

```






<br><br><br><br><br><br>
