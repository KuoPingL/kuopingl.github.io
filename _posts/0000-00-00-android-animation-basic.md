---
layout: post
title:  "Android - Animation Basic : Animation"
date:   2022-08-08 16:15:07 +0800
categories: [animation, android, basic]
---

## Background
In Android, animations can be categorized into two categories:
- **View** Animation (Tween Animation, Frame Animation)
- **Property** Animation (ValueAnimator, ObjectAnimator)

> A **View Animation**, just like its name, it is an animation that can <u>only be
applied to views only</u>.

> While **Property Animation** can be applied to any properties in any class.

In this article, I will focus on how to define a View Animation and how it works behind the scene.

## About Animation

When we are applying animations to a View, we can perform the following simple transformations :
- alpha
- scale
- translate
- rotate

and of course, we can also create a new animation through different combinations of these 4 basic animations.

But first, let's take a look at the base class responsible for view animation, **Animation**.

### Abstract Class Animation
```Java
public abstract class Animation implements Cloneable
```
Since an **Animation** is an abstract class, we can't really use it directly. Instead, we need to use classes that implements it, including :

```Java
public class AlphaAnimation extends Animation
public class ScaleAnimation extends Animation
public class TranslateAnimation extends Animation
public class RotateAnimation extends Animation
public class AnimationSetAnimation extends Animation
```
If you wish, you can also create a custom Animation yourself. Just to give you an idea on how easy it is to implement an Animation, here is the code for **RotateAnimation**.

<details>
<summary><b>RotateAnimation</b> Source Code</summary>

```Java
public class RotateAnimation extends Animation {
    private float mFromDegrees;
    private float mToDegrees;

    private int mPivotXType = ABSOLUTE;
    private int mPivotYType = ABSOLUTE;
    private float mPivotXValue = 0.0f;
    private float mPivotYValue = 0.0f;

    private float mPivotX;
    private float mPivotY;

    public RotateAnimation(Context context, AttributeSet attrs) {
        super(context, attrs);

        TypedArray a = context.obtainStyledAttributes(attrs,
                com.android.internal.R.styleable.RotateAnimation);

        mFromDegrees = a.getFloat(
                com.android.internal.R.styleable.RotateAnimation_fromDegrees, 0.0f);
        mToDegrees = a.getFloat(com.android.internal.R.styleable.RotateAnimation_toDegrees, 0.0f);

        Description d = Description.parseValue(a.peekValue(
            com.android.internal.R.styleable.RotateAnimation_pivotX));
        mPivotXType = d.type;
        mPivotXValue = d.value;

        d = Description.parseValue(a.peekValue(
            com.android.internal.R.styleable.RotateAnimation_pivotY));
        mPivotYType = d.type;
        mPivotYValue = d.value;

        a.recycle();

        initializePivotPoint();
    }

    public RotateAnimation(float fromDegrees, float toDegrees) {
        mFromDegrees = fromDegrees;
        mToDegrees = toDegrees;
        mPivotX = 0.0f;
        mPivotY = 0.0f;
    }

    public RotateAnimation(float fromDegrees, float toDegrees, float pivotX, float pivotY) {
        mFromDegrees = fromDegrees;
        mToDegrees = toDegrees;

        mPivotXType = ABSOLUTE;
        mPivotYType = ABSOLUTE;
        mPivotXValue = pivotX;
        mPivotYValue = pivotY;
        initializePivotPoint();
    }

    public RotateAnimation(float fromDegrees, float toDegrees, int pivotXType, float pivotXValue,
            int pivotYType, float pivotYValue) {
        mFromDegrees = fromDegrees;
        mToDegrees = toDegrees;

        mPivotXValue = pivotXValue;
        mPivotXType = pivotXType;
        mPivotYValue = pivotYValue;
        mPivotYType = pivotYType;
        initializePivotPoint();
    }

    /**
     * Called at the end of constructor methods to initialize, if possible, values for
     * the pivot point. This is only possible for ABSOLUTE pivot values.
     */
    private void initializePivotPoint() {
        if (mPivotXType == ABSOLUTE) {
            mPivotX = mPivotXValue;
        }
        if (mPivotYType == ABSOLUTE) {
            mPivotY = mPivotYValue;
        }
    }

    // This is the main method
    @Override
    protected void applyTransformation(float interpolatedTime, Transformation t) {
        float degrees = mFromDegrees + ((mToDegrees - mFromDegrees) * interpolatedTime);
        float scale = getScaleFactor();

        if (mPivotX == 0.0f && mPivotY == 0.0f) {
            t.getMatrix().setRotate(degrees);
        } else {
            t.getMatrix().setRotate(degrees, mPivotX * scale, mPivotY * scale);
        }
    }

    @Override
    public void initialize(int width, int height, int parentWidth, int parentHeight) {
        super.initialize(width, height, parentWidth, parentHeight);
        mPivotX = resolveSize(mPivotXType, mPivotXValue, width, parentWidth);
        mPivotY = resolveSize(mPivotYType, mPivotYValue, height, parentHeight);
    }
}
```

</details>

<br>The main functions that need to be implemented are :
- `applyTransformation`
  - responsible for doing the actual transformation by manipulating the **Matrix** of the View
- `initialize`
  - responsible for initialization of properties

### Animation XML
Other than creating Animation through code, we can also define them using XML.

To give you an idea on what tags or properties we can define on an Animator, here is the attrs for Animation in XML

> This gives you a better understanding on what **properties** or **tags** can be defined
when defining an Animation in XML and in code.

<details>
<summary><b>XML</b></summary>

```XML
<!-- frameworks/base/core/res/res/values/attrs.xml -->
<declare-styleable name="Animation">
        <!-- Defines the interpolator used to smooth the animation movement in time. -->
        <attr name="interpolator" />
        <!-- When set to true, the value of fillBefore is taken into account. -->
        <attr name="fillEnabled" format="boolean" />
        <!-- When set to true or when fillEnabled is not set to true, the animation transformation
             is applied before the animation has started. The default value is true. -->
        <attr name="fillBefore" format="boolean" />
        <!-- When set to true, the animation transformation is applied after the animation is
             over. The default value is false. If fillEnabled is not set to true and the
             animation is not set on a View, fillAfter is assumed to be true.-->
        <attr name="fillAfter" format="boolean" />
        <!-- Amount of time (in milliseconds) for the animation to run. -->
        <attr name="duration" />
        <!-- Delay in milliseconds before the animation runs, once start time is reached. -->
        <attr name="startOffset" format="integer" />
        <!-- Defines how many times the animation should repeat. The default value is 0. -->
        <attr name="repeatCount" format="integer">
            <enum name="infinite" value="-1" />
        </attr>
        <!-- Defines the animation behavior when it reaches the end and the repeat count is
             greater than 0 or infinite. The default value is restart. -->
        <attr name="repeatMode">
            <!-- The animation starts again from the beginning. -->
            <enum name="restart" value="1" />
            <!-- The animation plays backward. -->
            <enum name="reverse" value="2" />
        </attr>
        <!-- Allows for an adjustment of the Z ordering of the content being
             animated for the duration of the animation.  The default value is normal. -->
        <attr name="zAdjustment">
            <!-- The content being animated be kept in its current Z order. -->
            <enum name="normal" value="0" />
            <!-- The content being animated is forced on top of all other
                 content for the duration of the animation. -->
            <enum name="top" value="1" />
            <!-- The content being animated is forced under all other
                 content for the duration of the animation. -->
            <enum name="bottom" value="-1" />
        </attr>
        <!-- Special background behind animation.  Only for use with window
             animations.  Can only be a color, and only black.  If 0, the
             default, there is no background. -->
        <attr name="background" />
        <!-- Special option for window animations: if this window is on top
             of a wallpaper, don't animate the wallpaper with it. -->
        <attr name="detachWallpaper" format="boolean" />
        <!-- Special option for window animations: show the wallpaper behind when running this
             animation. -->
        <attr name="showWallpaper" format="boolean" />
        <!-- Special option for window animations: whether window should have rounded corners.
             @see ScreenDecorationsUtils#getWindowCornerRadius(Resources) -->
        <attr name="hasRoundedCorners" format="boolean" />
    </declare-styleable>
```

</details>




<br> Since all the Animation mentioned below are inherited from **Animation**,
therefore, they can also define attributes shown in **Animation** XML.

Next, we should see how different Animation can be defined in XML and programmatically.

### AlphaAnimation
> Alpha animation is used to change the view's transparancy from *fromAlpha*
  to *toAlpha*.

```XML
<declare-styleable name="AlphaAnimation">
    <attr name="fromAlpha" format="float" />
    <attr name="toAlpha" format="float" />
</declare-styleable>
```
> The alpha value ranges from **0.0f to 1.0f**

Alpha animation can be created using **AlphaAnimation** or **<alpha>** tag
in anim xml.

In XML :

```xml
<?xml version="1.0" encoding="utf-8"?>
<alpha xmlns:android="http://schemas.android.com/apk/res/android"
    android:fromAlpha="0" android:toAlpha="100" android:duration="5000"/>
```

In Kotlin :

```Kotlin
/*
    public AlphaAnimation(float fromAlpha, float toAlpha) {
        mFromAlpha = fromAlpha;
        mToAlpha = toAlpha;
    }
*/
val animation = AlphaAnimation(0f, 100f).apply{ duration = 5000 }
```

In Java :

```java
Animation animation = new AlphaAnimation(0f, 100f);
animation.setDuration(5000);
```

### Scale
> Scale is used to animate by changing the view's size with scale value.

```XML
<declare-styleable name="ScaleAnimation">
    <attr name="fromXScale" format="float|fraction|dimension" />
    <attr name="toXScale" format="float|fraction|dimension" />
    <attr name="fromYScale" format="float|fraction|dimension" />
    <attr name="toYScale" format="float|fraction|dimension" />
    <attr name="pivotX" />
    <attr name="pivotY" />
</declare-styleable>
```

> scale value can be float (0.25f), fraction (1/4) or dimension (10dp)
>> pivots defines the point where every points on the view will use it as reference.



Similarly, a scale animation can also be created in **<scale>** XML tag and **ScaleAnimation** class :
```xml
<scale xmlns:android="http://schemas.android.com/apk/res/android"
    android:pivotX="50%p" android:pivotY="50%" android:fromXScale="1" android:toXScale="10" android:fromYScale="1" android:toYScale="0.1" android:duration="1000"/>
<!-- when we want to use the pivot with repect to parent, we can simply add a `p` at the end of the value -->
```
In Kotlin

```kotlin
/*
public TranslateAnimation(int fromXType, float fromXValue, int toXType, float toXValue,
        int fromYType, float fromYValue, int toYType, float toYValue)
*/

TranslateAnimation(
    Animation.RELATIVE_TO_SELF, 0f,
    Animation.RELATIVE_TO_SELF, 0f,
    Animation.RELATIVE_TO_SELF, -1.0f,
    Animation.RELATIVE_TO_SELF, 1.0f).apply {
    duration = 1000
}
```



### Translate
> Translate is used to move the canvas from Point A to Point B based on the changes (delta) in X and Y axis.
```xml
<declare-styleable name="TranslateAnimation">
    <attr name="fromXDelta" format="float|fraction" />
    <attr name="toXDelta" format="float|fraction" />
    <attr name="fromYDelta" format="float|fraction" />
    <attr name="toYDelta" format="float|fraction" />
</declare-styleable>
```

In XML :
```xml
<!-- This will make the target view's canvas to go  -->
<translate xmlns:android="http://schemas.android.com/apk/res/android"
    android:fromXDelta="0%p" android:toXDelta="100%p" android:duration="1000"/>
```

```Kotlin
/*
public TranslateAnimation(
  int fromXType, float fromXValue,
  int toXType, float toXValue,
  int fromYType, float fromYValue,
  int toYType, float toYValue) {
*/
private val _animationTranslate = TranslateAnimation(
    Animation.RELATIVE_TO_PARENT, 0f,
    Animation.RELATIVE_TO_PARENT, 1.0f,
    Animation.RELATIVE_TO_SELF, 0f,
    Animation.RELATIVE_TO_SELF, 0f).apply {
    duration = 1000
}
```


### Rotate
> Rotate is used to rotate the canvas from degree A to degree B with respect to the pivot point.
```xml
<declare-styleable name="RotateAnimation">
    <attr name="fromDegrees" />
    <attr name="toDegrees" />
    <attr name="pivotX" />
    <attr name="pivotY" />
</declare-styleable>
```

```xml
<rotate xmlns:android="http://schemas.android.com/apk/res/android"
    android:fromDegrees="0" android:toDegrees="360"
    android:pivotY="50%" android:pivotX="50%"
    android:duration="1000"/>
```

```kotlin
private val _animationRotate = RotateAnimation(
    0f, 360f,
    Animation.RELATIVE_TO_SELF, 0.5f,
    Animation.RELATIVE_TO_SELF, 0.5f
).apply {
    duration = 1000
}
```


### AnimationSet
Now that you know how a single animation is created, it's time to spice things up by combining animations.

Animations can be combined using AnimationSet.
```Java
// android.view.animation
public class AnimationSet extends Animation {
  public AnimationSet(boolean shareInterpolator) {
      setFlag(PROPERTY_SHARE_INTERPOLATOR_MASK, shareInterpolator);
      init();
  }

  public AnimationSet(Context context, AttributeSet attrs) {
      // ... init from xml
  }
}
```

Again, we can create **AnimationSet** either programmatically
```kotlin
val animationSet = AnimationSet(true).apply {
    addAnimation(_animationAlpha)
    addAnimation(_animationTranslate)
    addAnimation(_animationScale)
    addAnimation(_animationRotate)
}
```
or in XML :

```xml
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <alpha
        android:fromAlpha="0" android:toAlpha="100"
        android:duration="1000" android:interpolator="@android:interpolator/decelerate_quad"/>
    <rotate
        android:fromDegrees="0" android:toDegrees="360"
        android:pivotY="0%" android:pivotX="50%"
        android:duration="1000"/>
    <scale
        android:pivotX="50%p" android:pivotY="50%"
        android:fromXScale="1" android:toXScale="10"
        android:fromYScale="1" android:toYScale="0.1"
        android:duration="1000"
        android:interpolator="@android:interpolator/accelerate_cubic"/>
    <translate
        android:fromXDelta="0%p" android:toXDelta="100%p" android:duration="1000"/>s
</set>
```

By default, all animations in **AnimationSet** shares the same interpolator, and by default, `AccelerateDecelerateInterpolator` is used :
```Java
protected void ensureInterpolator() {
    if (mInterpolator == null) {
        mInterpolator = new AccelerateDecelerateInterpolator();
    }
}
```

## Animation Types
Depending on whre animation is applied to,

### Tween Animation
> Creates an animation by performing a series of transformations on a single image with an Animation


### Frame Animation
> Creates an animation by showing a sequence of images in order with an AnimationDrawable


## Understand the process of Animation
From above, we know how to define any animation, but it's also important to understand how it works.

If we want to start any animation, we can do either called `startAnimation` or `setAnimation` :
```Kotlin
view.startAnimation(animation)
// or
view.setAnimation(animation)
```

If we were using XML, then we need to parse it using AnimationUtils :
```Kotlin
view.setAnimation(AnimationUtils.loadAnimation(context, xmlId))
```

### Behind the Scene
#### Calling startAnimation
```Java
public void startAnimation(Animation animation) {
    animation.setStartTime(Animation.START_ON_FIRST_FRAME);
    setAnimation(animation);
    invalidateParentCaches();
    invalidate(true);
}

public void setAnimation(Animation animation) {
    mCurrentAnimation = animation;

    if (animation != null) {
        // If the screen is off assume the animation start time is now instead of
        // the next frame we draw. Keeping the START_ON_FIRST_FRAME start time
        // would cause the animation to start when the screen turns back on
        if (mAttachInfo != null && mAttachInfo.mDisplayState == Display.STATE_OFF
                && animation.getStartTime() == Animation.START_ON_FIRST_FRAME) {
            animation.setStartTime(AnimationUtils.currentAnimationTimeMillis());
        }
        animation.reset();
    }
}
```
By calling `startAnimation`, it will do the followings :
1. set the start time of the animation to START_ON_FIRST_FRAME (-1)
2. pass the animation into `setAnimation`, and do the followings :
   - make sure animation is not null
   - set the start time of animation to current time if the following happens:
     1. The view has not yet detached from Window (`mAttachInfo != null`)
     2. The display is currently off
     3. The animation start time is START_ON_FIRST_FRAME
   - performs a reset on the animation, such that it will start over again once the display has turned back on.
3. invalidate parent's cache, forcing the parent view to rebuild display list.
    ```Java
    /**
    - Used to indicate that the parent of this view should clear its caches. This functionality
    - is used to force the parent to rebuild its display list (when hardware-accelerated),
    - which is necessary when various parent-managed properties of the view change, such as
    - alpha, translationX/Y, scrollX/Y, scaleX/Y, and rotation/X/Y. This method only
    - clears the parent caches and does not causes an invalidate event.
    -
    - @hide
    */
    @UnsupportedAppUsage
    protected void invalidateParentCaches() {
      if (mParent instanceof View) {
          ((View) mParent).mPrivateFlags |= PFLAG_INVALIDATED;
      }
    }
    ```
4. do a full invalidation by calling `invalidate(true)`.

So far, the code doesn't seem to be performing any animation, so when do animations occur ?

If we look at the comments in `View.java`, we can see the following statement :
```Java
/*
* You can attach an {@link Animation} object to a view using
* {@link #setAnimation(Animation)} or
* {@link #startAnimation(Animation)}. The animation can alter the scale,
* rotation, translation and alpha of a view over time. If the animation is
* attached to a view that has children, the animation will affect the entire
* subtree rooted by that node. When an animation is started, the framework will
* take care of redrawing the appropriate views until the animation completes.
*/
```

From this, we are certain that the animation occur during drawing.

#### Drawing
There are two functions responsible for drawing :
```Java
public void draw(Canvas canvas) {
  // Manually render this view (and all of its children) to the given Canvas.
}

boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {
  // This is called by ViewGroup.drawChild() to have each child view draw itself.
  // This is where the View specializes rendering behavior based on layer type,
  // and hardware acceleration.
}
```

Obviously, the one we are interested is the latter one.
This function is called when `draw(Canvas canvas)` is triggered :
![](/images/posts/jekyll/2022-09-10-android-animation-basic-01.svg)

```Java
boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {
  // ...
  final Animation a = getAnimation();
  if (a != null) {
      more = applyLegacyAnimation(parent, drawingTime, a, scalingRequired);
      concatMatrix = a.willChangeTransformationMatrix();
      if (concatMatrix) {
          mPrivateFlags3 |= PFLAG3_VIEW_IS_ANIMATING_TRANSFORM;
      }
      transformToApply = parent.getChildTransformation();
  } else {
      // ...
  }
}
```

Upon triggering, the function will check if an Animation is assigned.
If an animation is found, then it will pass the Animation object to `applyLegacyAnimation`.

```Java
private boolean applyLegacyAnimation(ViewGroup parent, long drawingTime,
        Animation a, boolean scalingRequired){}
```

This function will first check if Animation is initialized, if not, then animation will be initialized.

```Java
final boolean initialized = a.isInitialized();
if (!initialized) {
    a.initialize(mRight - mLeft, mBottom - mTop, parent.getWidth(), parent.getHeight());
    a.initializeInvalidateRegion(0, 0, mRight - mLeft, mBottom - mTop);
    if (mAttachInfo != null) a.setListenerHandler(mAttachInfo.mHandler);
    onAnimationStart();
}
```
