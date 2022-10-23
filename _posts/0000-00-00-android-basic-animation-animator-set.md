---
layout: post
title:  "Android - Animation Basic : AnimationSet and AnimatorSet"
date:   2022-08-08 16:15:07 +0800
categories: [animation, android, basic]
---

## Background
From the previous parts of the series,



## AnimationSet
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

### Behind the Scene

## AnimatorSet


### Behind the Scene


## Playing Sequentially vs Playing Together
If we were to use **AnimationSet**, by default, animations
