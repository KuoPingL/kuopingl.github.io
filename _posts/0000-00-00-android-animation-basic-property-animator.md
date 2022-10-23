---
layout: post
title:  "Android - Animation Basic : Property Animator"
date:   2022-08-08 16:15:07 +0800
categories: [animation, android, basic]
---

## Background
From the previous post of this series [Android - Animation Basic : View Animation](),
we know that **Animation** object can only apply animation to canvas.

> However, after Android 3.0, **Animator** is ...

In this article, we will look at Property Animators that will actually act to the whole component instead of just the canvas.

There are 3 types of Property Animators in general:
- **Value**Animator
- **Object**Animator
- **ViewProperty**Animator

Just like Animation, an Animator can also be defined using XML.
Here is the AttributeSet of Animator :
```xml
<declare-styleable name="Animator">
    <!-- Defines the interpolator used to smooth the animation movement in time. -->
    <attr name="interpolator" />
    <!-- Amount of time (in milliseconds) for the animation to run. -->
    <attr name="duration" />
    <!-- Delay in milliseconds before the animation runs, once start time is reached. -->
    <attr name="startOffset"/>
    <!-- Defines how many times the animation should repeat. The default value is 0. -->
    <attr name="repeatCount"/>
    <!-- Defines the animation behavior when it reaches the end and the repeat count is
         greater than 0 or infinite. The default value is restart. -->
    <attr name="repeatMode"/>
    <!-- Value the animation starts from. -->
    <attr name="valueFrom" format="float|integer|color|dimension|string"/>
    <!-- Value the animation animates to. -->
    <attr name="valueTo" format="float|integer|color|dimension|string"/>
    <!-- The type of valueFrom and valueTo. -->
    <attr name="valueType">
        <!-- The given values are floats. This is the default value if valueType is
             unspecified. Note that if any value attribute has a color value
             (beginning with "#"), then this attribute is ignored and the color values are
             interpreted as integers. -->
        <enum name="floatType" value="0" />
        <!-- values are integers. -->
        <enum name="intType"   value="1" />
        <!-- values are paths defined as strings.
             This type is used for path morphing in AnimatedVectorDrawable. -->
        <enum name="pathType"   value="2" />
        <!-- values are colors, which are integers starting with "#". -->
        <enum name="colorType"   value="3" />
    </attr>
    <!-- Placeholder for a deleted attribute. This should be removed before M release. -->
    <attr name="removeBeforeMRelease" format="integer" />
</declare-styleable>
```



## ValueAnimator
**ValueAnimator** is an **Animator** class that can help us animate by providing valid values of certain type continuously based on the evaluation implemented by TypeEvaluator.

```Java
public class ValueAnimator extends Animator implements AnimationHandler.AnimationFrameCallback {}
```

The types that are available for evaluating includes:
- integer using IntEvaluator
- float using FloatEvaluator
- Argb (int) with ArgbEvaluator
- and any object with custom TypeEvaluator.

Let's use integer as the sample implementation of how **ValueAnimator** is used.

### ValueAnimator.ofInt
Here is the full code for ValueAnimator implementation, but we will walk through it briefly.

```Kotlin
val valueAnimator = ValueAnimator.ofInt(0, 400, 200, 300, 0).setDuration(500)
valueAnimator.addUpdateListener {
    val curValue = it.animatedValue as Int
    btnViewPropertyTarget.layout(x0 + curValue, y0 + curValue,
        curValue + x0 + btnViewPropertyTarget.width, curValue + y0 + btnViewPropertyTarget.height)
}
valueAnimator.start()
```

Let's look at the first line

```Kotlin
val valueAnimator = ValueAnimator.ofInt(0, 400, 200, 300, 0).setDuration(500)
```

Again, we can also define this using XML :
```xml
```

This line is responsible for creating a ValueAnimator with integer type values.
And here is what happen behind the scene.

```Java
// ValueAnimator.java
public static ValueAnimator ofInt(int... values) {
    ValueAnimator anim = new ValueAnimator();
    anim.setIntValues(values);
    return anim;
}
```

Those integer values will be stored in a **PropertyValuesHolder** ,

```Java
public void setIntValues(int... values) {
    if (values == null || values.length == 0) {
        return;
    }
    if (mValues == null || mValues.length == 0) {
        setValues(PropertyValuesHolder.ofInt("", values));
    } else {
        PropertyValuesHolder valuesHolder = mValues[0];
        valuesHolder.setIntValues(values);
    }
    // New property/values/target should cause re-initialization prior to starting
    mInitialized = false;
}
```

A **PropertyValuesHolder** is a class that holds the following variables :
- **Property** : an abstract class that acts like a Map with String as key and Class as value.
- **Method** : a final Executable class that is used to store the Method. In our case, the Getter and Setter for certain property with a specific property name.
- **Class** : here, a class is the class of the property value type. (For setIntValues, this will be int.class).
- **Keyframes** : The set of keyframes (time/value pairs) that define this animation. This is where values were kept.
- **TypeEvaluator** : A SAM interface that is used to return a value after evaluating using startValue, endValue and friction.

```java
public interface TypeEvaluator<T> {
    public T evaluate(float fraction, T startValue, T endValue);
}
```

After creating a ValueAnimator, we can start the "animation" by calling `start()`
```Kotlin
valueAnimator.start()
```

Although I called it "animation", but what it does is to provide us with values between those given integers. So nothing will happen for now, but let's take a look at what `start()` does:

The `start()` function will do the followings :
1. calculate the `mSeekFraction`
2. notify **start Listeners**
3. called `addAnimationCallback` to register **ValueAnimator** as **AnimationFrameCallback** (<u>this is a very important step</u>)
4. called `setCurrentFraction` with `mSeekFraction`

```Java
@Override
public void start() {
    start(playBackward: false);
}

private void start(boolean playBackwards) {
    // Looper.myLooper() is the Looper from the Thread this method is called
    if (Looper.myLooper() == null) {
        throw new AndroidRuntimeException("Animators may only be run on Looper threads");
    }
    mReversing = playBackwards;
    mSelfPulse = !mSuppressSelfPulseRequested;
    // Special case: reversing from seek-to-0 should act as if not seeked at all.
    if (playBackwards && mSeekFraction != -1 && mSeekFraction != 0) {
        if (mRepeatCount == INFINITE) {
            // Calculate the fraction of the current iteration.
            float fraction = (float) (mSeekFraction - Math.floor(mSeekFraction));
            mSeekFraction = 1 - fraction;
        } else {
            mSeekFraction = 1 + mRepeatCount - mSeekFraction;
        }
    }
    mStarted = true;
    mPaused = false;
    mRunning = false;
    mAnimationEndRequested = false;
    // Resets mLastFrameTime when start() is called, so that if the animation was running,
    // calling start() would put the animation in the
    // started-but-not-yet-reached-the-first-frame phase.
    mLastFrameTime = -1;
    mFirstFrameTime = -1;
    mStartTime = -1;
    addAnimationCallback(0);

    if (mStartDelay == 0 || mSeekFraction >= 0 || mReversing) {
        // If there's no start delay, init the animation and notify start listeners right away
        // to be consistent with the previous behavior. Otherwise, postpone this until the first
        // frame after the start delay.
        startAnimation();
        if (mSeekFraction == -1) {
            // No seek, start at play time 0. Note that the reason we are not using fraction 0
            // is because for animations with 0 duration, we want to be consistent with pre-N
            // behavior: skip to the final value immediately.
            setCurrentPlayTime(0);
        } else {
            setCurrentFraction(mSeekFraction);
        }
    }
}
```


Next, `setCurrentFraction` will fetch values from the **TypeEvaluator** via the following process :
1. prepare the **PropertyValuesHolder**s that will be used for the animation
2. called `clampFraction` to make sure the friction calculated in valid, whether it is < 0 or in the case when it is not repeated infinitely, if *fraction* is greater than `mRepeatCount + 1`
3. after ensuring that *fraction* is valid, it will call `animateValue` to extract value from the **TypeEvaluator** and finally will call the **AnimatorUpdateListener** with the current value.

```Java
@CallSuper
@UnsupportedAppUsage
void animateValue(float fraction) {
   fraction = mInterpolator.getInterpolation(fraction);
   mCurrentFraction = fraction;
   int numValues = mValues.length;
   for (int i = 0; i < numValues; ++i) {
       mValues[i].calculateValue(fraction);
   }
   if (mUpdateListeners != null) {
       int numListeners = mUpdateListeners.size();
       for (int i = 0; i < numListeners; ++i) {
           mUpdateListeners.get(i).onAnimationUpdate(this);
       }
   }
}
```

Now that you know the process of how ValueAnimator works, if we want to use the values from the TypeEvaluator, we need get ourself an **AnimatorUpdateListener**, so :
```kotlin
valueAnimator.addUpdateListener {
    val curValue = it.animatedValue as Int
    btnViewPropertyTarget.layout(x0 + curValue, y0 + curValue,
        curValue + x0 + btnViewPropertyTarget.width, curValue + y0 + btnViewPropertyTarget.height)
}
```

This will make the button shift from its original position to the curValue relative to its original position.

However, have you notice, that this seems to be a one time call only.
But that's not what we expect, the ValueAnimator should give an ongoing update on the value until it is due.

#### How does it work ?
So how did it do it ?

Remember the *important* step from beginning ?
```Java
private void start(boolean playBackwards) {
    // ...
    addAnimationCallback(0);
    // ...
}
```

`addAnimationCallback` will result in adding the current ValueAnimator to *mAnimationCallbacks*
```Java
private void addAnimationCallback(long delay) {
    if (!mSelfPulse) {
        return;
    }
    getAnimationHandler().addAnimationFrameCallback(this, delay);
}
```
which will then trigger the `addAnimationFrameCallback` in **AnimationHandler**.

```Java
// AnimationHandler.java
public void addAnimationFrameCallback(final AnimationFrameCallback callback, long delay) {
    if (mAnimationCallbacks.size() == 0) {
        getProvider().postFrameCallback(mFrameCallback);
    }
    if (!mAnimationCallbacks.contains(callback)) {
        mAnimationCallbacks.add(callback);
    }

    if (delay > 0) {
        mDelayedCallbackStartTime.put(callback, (SystemClock.uptimeMillis() + delay));
    }
}
```

Before any **AnimationFrameCallback** is being added, AnimationHandler will generate a **AnimationFrameCallbackProvider**, which is implemented as **MyFrameCallbackProvider**, and triggered `postFrameCallback`

```Java
@Override
public void postFrameCallback(Choreographer.FrameCallback callback) {
    mChoreographer.postFrameCallback(callback);
}
```

`postFrameCallback` will eventually triggered `postFrameCallbackDelayed` in **Choreographer**, which will posts a frame callback to run on the next frame after a specific delay.

```Java
public void postFrameCallbackDelayed(FrameCallback callback, long delayMillis) {
    if (callback == null) {
        throw new IllegalArgumentException("callback must not be null");
    }

    postCallbackDelayedInternal(CALLBACK_ANIMATION,
            callback, FRAME_CALLBACK_TOKEN, delayMillis);
}
```

And this **FrameCallback** is defined in **AnimationHandler**

```Java
private final Choreographer.FrameCallback mFrameCallback = new Choreographer.FrameCallback() {
    @Override
    public void doFrame(long frameTimeNanos) {
        doAnimationFrame(getProvider().getFrameTime());
        if (mAnimationCallbacks.size() > 0) {
            getProvider().postFrameCallback(this);
        }
    }
};
```

This callback will be triggered as long as `mAnimationCallbacks.size() > 0` and that's how it provide us with continue update on fraction value.

Also, as you can see that every time this callback is triggered, it will do two things :
1. `doAnimationFrame` : This function will trigger `doAnimationFrame` in `callback`, which is a **AnimationFrameCallback** or in our case **ValueAnimator**
    ```Java
    private void doAnimationFrame(long frameTime) {
        long currentTime = SystemClock.uptimeMillis();
          final int size = mAnimationCallbacks.size();
            for (int i = 0; i < size; i++) {
                final AnimationFrameCallback callback = mAnimationCallbacks.get(i);
                if (callback == null) {
                    continue;
                }
                if (isCallbackDue(callback, currentTime)) {
                    callback.doAnimationFrame(frameTime);
                      if (mCommitCallbacks.contains(callback)) {
                          getProvider().postCommitCallback(new Runnable() {
                              @Override
                                public void run() {
                                    commitAnimationFrame(callback, getProvider().getFrameTime());
                                }
                      });
                  }
            }
        }
        cleanUpList();
    }
    ```
2. After calling `doAnimationFrame` on **ValueAnimator**, it will eventually trigger `animateValue` method in **ValueAnimator**, which will update the current value and triggered `onAnimationUpdate` on **AnimatorUpdateListener** or the AnimatorUpdateListener we created :

```Kotlin
valueAnimator.addUpdateListener {
   mBinding.textviewChangeColorTarget.setTextColor(it.animatedValue as Int)
}
```



Whew, now that's done. Let's take a look at how `fraction` is being formed.

#### How is fraction calculated ?

This is done by compare the difference between starting frame time and current frame time.



## ObjectAnimator
> Unlike ValueAnimator, an ObjectAnimator does not require to add listener where we need to calculate the proper values to apply to the property of the target of interest.
> Instead, it takes an Object target and a property name that can be retrieved from the target via JNI or reflection.

### Usage
Here are some examples on how ObjectAnimator can be used :

```kotlin
mBinding.apply {
    btnAlpha.setOnClickListener {
        ObjectAnimator.ofFloat(btnAlpha, "alpha", 1f, 0f ,1f).apply {
            duration = 2000
        }.start()
    }

    btnRotate.setOnClickListener {
        ObjectAnimator.ofFloat(btnRotate, "rotation", 0f, 180f ,0f).apply {
            duration = 2000
        }.start()
    }

    btnRotateX.setOnClickListener {
        ObjectAnimator.ofFloat(btnRotateX, "rotationX", 0f, 180f ,0f).apply {
            duration = 2000
        }.start()
    }

    btnRotateY.setOnClickListener {
        ObjectAnimator.ofFloat(btnRotateY, "rotationY", 0f, 180f ,0f).apply {
            duration = 2000
        }.start()
    }
}
```

<u>How do I know what are the property names, you ask ?</u>
We can simply look at the **View.java** source code, we can find tons of getters and setters with naming convension of set{propertyName} or get{propertyName}. Hence, anything with these a getter and a setter can be used in ObjectAnimator.

Of course, we can do the same using ValueAnimator, just that we have to add our own listners.

Similar to ValueAnimator, an ObjectAnimator also contains builder methods for basic types and object:
```Java
ObjectAnimator.ofArgb(...)
ObjectAnimator.ofFloat(...)
ObjectAnimator.ofMultiFloat(...)
ObjectAnimator.ofObject(...)
ObjectAnimator.ofPropertyValuesHolder(...)
```

### How does it work ?
An **ObjectAnimator** works similarly as ValueAnimator, which is by using PropertyValuesHolders to generate fraction. As you can see below, where the creation of an ObjectAnimator will create a PropertyValuesHolder and cache  for a propertyName.

```Java
private ObjectAnimator(Object target, String propertyName) {
    setTarget(target);
    setPropertyName(propertyName);
}
```

> The target will be saved as a **WeakReference** object.

```Java
@Override
public void setTarget(@Nullable Object target) {
    final Object oldTarget = getTarget();
    if (oldTarget != target) {
        if (isStarted()) {
            cancel();
        }
        mTarget = target == null ? null : new WeakReference<Object>(target);
        // New target should cause re-initialization prior to starting
        mInitialized = false;
    }
}
```

> propertyName on the other hand, will be saved in a **PropertyValuesHolder**, which will be cached, and it will be stored in a Map<String, PropertyValuesHolder>.

```Java
public void setPropertyName(@NonNull String propertyName) {
    // mValues could be null if this is being constructed piecemeal. Just record the
    // propertyName to be used later when setValues() is called if so.
    if (mValues != null) {
        PropertyValuesHolder valuesHolder = mValues[0];
        String oldName = valuesHolder.getPropertyName();
        valuesHolder.setPropertyName(propertyName);
        mValuesMap.remove(oldName);
        mValuesMap.put(propertyName, valuesHolder);
    }
    mPropertyName = propertyName;
    // New property/values/target should cause re-initialization prior to starting
    mInitialized = false;
}
```

Once we called `start()` on an ObjectAnimator, this is what will happen :
```Java
@Override
public void start() {
    AnimationHandler.getInstance().autoCancelBasedOn(this);
    if (DBG) {
        Log.d(LOG_TAG, "Anim target, duration: " + getTarget() + ", " + getDuration());
        for (int i = 0; i < mValues.length; ++i) {
            PropertyValuesHolder pvh = mValues[i];
            Log.d(LOG_TAG, "   Values[" + i + "]: " +
                pvh.getPropertyName() + ", " + pvh.mKeyframes.getValue(0) + ", " +
                pvh.mKeyframes.getValue(1));
        }
    }
    super.start();
}
```
First, it will be cancel it's any previous ObjectAnimator that is still running.

This is done by creating an AnimationHandler and trigger `autoCancelBasedOn(ObjectAnimator)` and scan through **AnimationFrameCallback** available in AnimationHandler.

When AnimationHandler found a non-null AnimationFrameCallback, it will let `objectAnimator` decide whether it should be cancelled by triggering `mAnimationCallbacks.get(i)).cancel()`.

```Java
// AnimationHandler
void autoCancelBasedOn(ObjectAnimator objectAnimator) {
    for (int i = mAnimationCallbacks.size() - 1; i >= 0; i--) {
        AnimationFrameCallback cb = mAnimationCallbacks.get(i);
        if (cb == null) {
            continue;
        }
        if (objectAnimator.shouldAutoCancel(cb)) {
            ((Animator) mAnimationCallbacks.get(i)).cancel();
        }
    }
}
```

The requirements for a **AnimationFrameCallback** to be cancelled are as follows :
- it is an instance of ObjectAnimator
- it's `mAutoCancel` is *true*
- it has the same target and properties as the current ObjectAnimator

```Java
boolean shouldAutoCancel(AnimationHandler.AnimationFrameCallback anim) {
    if (anim == null) {
        return false;
    }

    if (anim instanceof ObjectAnimator) {
        ObjectAnimator objAnim = (ObjectAnimator) anim;
        if (objAnim.mAutoCancel && hasSameTargetAndProperties(objAnim)) {
            return true;
        }
    }
    return false;
}
```
Finally, it will call `super.start()`, which will trigger the `start(false)` method in `ValueAnimator`, and resulting in the same process as described in the first section.

**WAIT !!!** If that's the end of the story, how does it update the property ? Where is the listener ?

If you remember, at the end of `doAnimationFrame`, ValueAnimator will trigger `animateBaseOnTime`, which will trigger `animateValue`.

In the case of ObjectAnimator, this is what happen when `animateValue` is triggered :

```Java
@CallSuper
@Override
void animateValue(float fraction) {
    final Object target = getTarget();
    if (mTarget != null && target == null) {
        // We lost the target reference, cancel and clean up. Note: we allow null target if the
        /// target has never been set.
        cancel();
        return;
    }

    super.animateValue(fraction);
    int numValues = mValues.length;
    for (int i = 0; i < numValues; ++i) {
        mValues[i].setAnimatedValue(target);
    }
}
```

After calling `super.animateValue`, getting the new fraction value from interpolator, the new value will be given to PropertyValuesHolder via `mValues[i].setAnimatedValue(target)`.

```Java
void setAnimatedValue(Object target) {
    if (mProperty != null) {
        mProperty.set(target, getAnimatedValue());
    }
    if (mSetter != null) {
        try {
            mTmpValueArray[0] = getAnimatedValue();
            mSetter.invoke(target, mTmpValueArray);
        } catch (InvocationTargetException e) {
            Log.e("PropertyValuesHolder", e.toString());
        } catch (IllegalAccessException e) {
            Log.e("PropertyValuesHolder", e.toString());
        }
    }
}
```
This method will get the setter for the property and set a new value to it.

### Side Notes
If we were to use `{Animator}.of{Type}(obj, "{F(x) Name}", {values})`, please remember that if you only pass a single value as parameter, it will call `get{F(x) Name}` to get the initial value.

> For type integer and float, if you only pass a single value, it will assumed that initial value is 0.

## AnimatorSet
> Note: This is different from `AnimationSet`, which is used for `Animation`.


## Behind the Scene



## ViewPropertyAnimator
