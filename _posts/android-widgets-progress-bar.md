---
layout: post
title:  "Android - Understanding Progress Bar"
date:   2022-08-08 16:15:07 +0800
categories: [android, widget, study]
---

## Background
This article is about how Progress Bar works, perhaps by understanding how it was created and how it works, we can have a better idea how to create a better custom view.

## Progress Bar : An Introduction
```java
public class ProgressBar extends View {}
```
> A **ProgressBar** is a **View** that shows a repeated rotation of a drawable.

### Dive into Constructors
> Here, we take a look at what the constructors do.

<details>
<summary>Constructor Code</summary>

```java
public ProgressBar(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
    super(context, attrs, defStyleAttr, defStyleRes);

    mUiThreadId = Thread.currentThread().getId();
    initProgressBar();

    final TypedArray a = context.obtainStyledAttributes(
            attrs, R.styleable.ProgressBar, defStyleAttr, defStyleRes);
    saveAttributeDataForStyleable(context, R.styleable.ProgressBar,
            attrs, a, defStyleAttr, defStyleRes);

    mNoInvalidate = true;

    final Drawable progressDrawable = a.getDrawable(R.styleable.ProgressBar_progressDrawable);
    if (progressDrawable != null) {
        // Calling setProgressDrawable can set mMaxHeight, so make sure the
        // corresponding XML attribute for mMaxHeight is read after calling
        // this method.
        if (needsTileify(progressDrawable)) {
            setProgressDrawableTiled(progressDrawable);
        } else {
            setProgressDrawable(progressDrawable);
        }
    }

    mDuration = a.getInt(R.styleable.ProgressBar_indeterminateDuration, mDuration);

    mMinWidth = a.getDimensionPixelSize(R.styleable.ProgressBar_minWidth, mMinWidth);
    mMaxWidth = a.getDimensionPixelSize(R.styleable.ProgressBar_maxWidth, mMaxWidth);
    mMinHeight = a.getDimensionPixelSize(R.styleable.ProgressBar_minHeight, mMinHeight);
    mMaxHeight = a.getDimensionPixelSize(R.styleable.ProgressBar_maxHeight, mMaxHeight);

    mBehavior = a.getInt(R.styleable.ProgressBar_indeterminateBehavior, mBehavior);

    final int resID = a.getResourceId(
            com.android.internal.R.styleable.ProgressBar_interpolator,
            android.R.anim.linear_interpolator); // default to linear interpolator
    if (resID > 0) {
        setInterpolator(context, resID);
    }

    setMin(a.getInt(R.styleable.ProgressBar_min, mMin));
    setMax(a.getInt(R.styleable.ProgressBar_max, mMax));

    setProgress(a.getInt(R.styleable.ProgressBar_progress, mProgress));

    setSecondaryProgress(a.getInt(
            R.styleable.ProgressBar_secondaryProgress, mSecondaryProgress));

    final Drawable indeterminateDrawable = a.getDrawable(
            R.styleable.ProgressBar_indeterminateDrawable);
    if (indeterminateDrawable != null) {
        if (needsTileify(indeterminateDrawable)) {
            setIndeterminateDrawableTiled(indeterminateDrawable);
        } else {
            setIndeterminateDrawable(indeterminateDrawable);
        }
    }

    mOnlyIndeterminate = a.getBoolean(
            R.styleable.ProgressBar_indeterminateOnly, mOnlyIndeterminate);

    mNoInvalidate = false;

    setIndeterminate(mOnlyIndeterminate || a.getBoolean(
            R.styleable.ProgressBar_indeterminate, mIndeterminate));

    mMirrorForRtl = a.getBoolean(R.styleable.ProgressBar_mirrorForRtl, mMirrorForRtl);

    if (a.hasValue(R.styleable.ProgressBar_progressTintMode)) {
        if (mProgressTintInfo == null) {
            mProgressTintInfo = new ProgressTintInfo();
        }
        mProgressTintInfo.mProgressBlendMode = Drawable.parseBlendMode(a.getInt(
                R.styleable.ProgressBar_progressTintMode, -1), null);
        mProgressTintInfo.mHasProgressTintMode = true;
    }

    if (a.hasValue(R.styleable.ProgressBar_progressTint)) {
        if (mProgressTintInfo == null) {
            mProgressTintInfo = new ProgressTintInfo();
        }
        mProgressTintInfo.mProgressTintList = a.getColorStateList(
                R.styleable.ProgressBar_progressTint);
        mProgressTintInfo.mHasProgressTint = true;
    }

    if (a.hasValue(R.styleable.ProgressBar_progressBackgroundTintMode)) {
        if (mProgressTintInfo == null) {
            mProgressTintInfo = new ProgressTintInfo();
        }
        mProgressTintInfo.mProgressBackgroundBlendMode = Drawable.parseBlendMode(a.getInt(
                R.styleable.ProgressBar_progressBackgroundTintMode, -1), null);
        mProgressTintInfo.mHasProgressBackgroundTintMode = true;
    }

    if (a.hasValue(R.styleable.ProgressBar_progressBackgroundTint)) {
        if (mProgressTintInfo == null) {
            mProgressTintInfo = new ProgressTintInfo();
        }
        mProgressTintInfo.mProgressBackgroundTintList = a.getColorStateList(
                R.styleable.ProgressBar_progressBackgroundTint);
        mProgressTintInfo.mHasProgressBackgroundTint = true;
    }

    if (a.hasValue(R.styleable.ProgressBar_secondaryProgressTintMode)) {
        if (mProgressTintInfo == null) {
            mProgressTintInfo = new ProgressTintInfo();
        }
        mProgressTintInfo.mSecondaryProgressBlendMode = Drawable.parseBlendMode(
                a.getInt(R.styleable.ProgressBar_secondaryProgressTintMode, -1), null);
        mProgressTintInfo.mHasSecondaryProgressTintMode = true;
    }

    if (a.hasValue(R.styleable.ProgressBar_secondaryProgressTint)) {
        if (mProgressTintInfo == null) {
            mProgressTintInfo = new ProgressTintInfo();
        }
        mProgressTintInfo.mSecondaryProgressTintList = a.getColorStateList(
                R.styleable.ProgressBar_secondaryProgressTint);
        mProgressTintInfo.mHasSecondaryProgressTint = true;
    }

    if (a.hasValue(R.styleable.ProgressBar_indeterminateTintMode)) {
        if (mProgressTintInfo == null) {
            mProgressTintInfo = new ProgressTintInfo();
        }
        mProgressTintInfo.mIndeterminateBlendMode = Drawable.parseBlendMode(a.getInt(
                R.styleable.ProgressBar_indeterminateTintMode, -1), null);
        mProgressTintInfo.mHasIndeterminateTintMode = true;
    }

    if (a.hasValue(R.styleable.ProgressBar_indeterminateTint)) {
        if (mProgressTintInfo == null) {
            mProgressTintInfo = new ProgressTintInfo();
        }
        mProgressTintInfo.mIndeterminateTintList = a.getColorStateList(
                R.styleable.ProgressBar_indeterminateTint);
        mProgressTintInfo.mHasIndeterminateTint = true;
    }

    a.recycle();

    applyProgressTints();
    applyIndeterminateTint();

    // If not explicitly specified this view is important for accessibility.
    if (getImportantForAccessibility() == View.IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
        setImportantForAccessibility(View.IMPORTANT_FOR_ACCESSIBILITY_YES);
    }
}
```
</details>

<br> Inside the constructor, the following things occurred :

#### 1. <u>Saving the current thread ID</u>

```java
mUiThreadId = Thread.currentThread().getId();
```
By saving the current thread ID, which should be the main thread, it will make sure `doRefreshProgress` is only called in the main thread.

```java
@UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
private synchronized void refreshProgress(int id, int progress, boolean fromUser,
        boolean animate) {
    if (mUiThreadId == Thread.currentThread().getId()) {
        doRefreshProgress(id, progress, fromUser, true, animate);
    } else {
        // create a runnable
        if (mRefreshProgressRunnable == null) {
            mRefreshProgressRunnable = new RefreshProgressRunnable();
        }

        final RefreshData rd = RefreshData.obtain(id, progress, fromUser, animate);
        mRefreshData.add(rd);
        if (mAttached && !mRefreshIsPosted) {
            post(mRefreshProgressRunnable);
            mRefreshIsPosted = true;
        }
    }
}
```

If the current thread is not the one stored, then a **Runnable** will be created.
<br>
This runnable is solely responsible for calling `doRefreshProgress` with the data stored inside the array of **RefreshData**, `mRefreshData`.
<br>
This runnable will be responsible for

<details>
<summary>RefreshProgressRunnable </summary>

```java
private class RefreshProgressRunnable implements Runnable {
    public void run() {
        synchronized (ProgressBar.this) {
            final int count = mRefreshData.size();
            for (int i = 0; i < count; i++) {
                final RefreshData rd = mRefreshData.get(i);
                doRefreshProgress(rd.id, rd.progress, rd.fromUser, true, rd.animate);
                rd.recycle();
            }
            mRefreshData.clear();
            mRefreshIsPosted = false;
        }
    }
}
```
</details>
<details>
<summary>RefreshData</summary>

```java
private static class RefreshData {
    private static final int POOL_MAX = 24;
    private static final SynchronizedPool<RefreshData> sPool =
            new SynchronizedPool<RefreshData>(POOL_MAX);

    public int id;
    public int progress;
    public boolean fromUser;
    public boolean animate;

    public static RefreshData obtain(int id, int progress, boolean fromUser, boolean animate) {
        RefreshData rd = sPool.acquire();
        if (rd == null) {
            rd = new RefreshData();
        }
        rd.id = id;
        rd.progress = progress;
        rd.fromUser = fromUser;
        rd.animate = animate;
        return rd;
    }

    public void recycle() {
        sPool.release(this);
    }
}
```
</details>

#### 2. Initialize Progress Bar variables
```java
private void initProgressBar() {
    mMin = 0;
    mMax = 100;
    mProgress = 0;
    mSecondaryProgress = 0;
    mIndeterminate = false;
    mOnlyIndeterminate = false;
    mDuration = 4000;
    mBehavior = AlphaAnimation.RESTART;
    mMinWidth = 24;
    mMaxWidth = 48;
    mMinHeight = 24;
    mMaxHeight = 48;
}
```

This is quite straight-forward, the variables that might be confusing are `mBehavior` and variables on `Indeterminate`.

> Behavior is the repeat mode of the animation.
<br><br>By stating it to be `AlphaAnimation.RESTART` means that the animation will restart from the beginning if the repeat count is either positive or `Animation.INFINTE`.


As for **Indeterminate**
