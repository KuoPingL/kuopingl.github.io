---
layout: post
title:  "Android - LiveData"
date:   2022-09-29 13:53:07 +0800
categories: [android jetpack, livedata, beginner]
---

### Background
If you are reading this article, I am assuming you have a basic idea on how **ViewModel** works and is trying to get hold on how **LiveData** works.

In this article, we will be looking into the structure of **LiveData** and the mechanism that is being used to pass data back and forth between View and **ViewModel**, as well as between Model and **ViewModel**.

### What is a LiveData?
A **LiveData<T>** is an abstract class that holds a `volatile Object` and a version number.
```java
public abstract class LiveData<T> {
  private volatile Object mData;
  private int mVersion; // default -1
}
```

**LiveData** also acts as a notifier, where it will notify all the `active` observers that it holds :
```java
private SafeIterableMap<Observer<? super T>, ObserverWrapper> mObservers =
            new SafeIterableMap<>();
```

>A **SafeIterableMap** is a linkedList, which pretends to be a map and supports modifications during iterations. Also, it is **Not** thread safe.

Through **Observer Design Pattern**, **LiveData** can easily notify the view when data changes occur.

And since **LiveData** is stored inside a **ViewModel**, the **ViewModel** can use it to receive users' input and acts upon it, such as fetching data and displaying it on the screen.


### How to use a LiveData ?
Usually, we would declare **LiveData** alongside with **MutableLiveData** on the same data inside a **ViewModel**.

>A **MutableLiveData** is a **LiveData** with data that can be alternated.
><br>By declaring both can encapsulate the data from changing by the users in expected way.


```java
class MyViewModel: ViewModel() {
  val didRequestFetchData: LiveData<Boolean>
  get = _didRequestFetchData

  val _didRequestFetchData = MutableLiveData<Boolean>(false)

  fun makeARequest() {
    _didRequestFetchData.value = true;
  }
}
```

Then, in Activity, we need to observe the changes on this data :

```java

private val _viewModel: MainViewModel by viewModels() // implementation "androidx.activity:activity-ktx:$activity_version"

override fun onCreate(savedInstanceState: Bundle?) {
  super.onCreate(savedInstanceState)
  setContentView(R.layout.activity_main)

  _viewModel.didRequestFetchData.observe(this) {
      if(it) {
        // do something if new value is true
      } else {
        // do something if new value is false
      }
  }

  findViewById<AppCompatButton>(R.id.btn).setOnClickListener {
    _viewModel.makeARequest()
  }
}
```

### How do we change the value in MutableLiveData ?
There are 3 ways to do so, 2 of them are shown above.
1. set a initial value
2. `setValue`

The third one is used when you wish to set a new value, from threads other than the **MainThread**
3. `postValue`

Let's take a closer look at what they do.

>**Setting a new Value will update the version**
In order for a **LiveData** to know if the value is a new value, it uses a version number to keep track of.

For instance, if we set a initial value :
```java
public LiveData(T value) {
    mData = value;
    mVersion = START_VERSION + 1;
}
```

then it would increment the version number by 1.

Similarly, set and post value will do the same thing
```java
@MainThread
protected void setValue(T value) {
    assertMainThread("setValue");
    mVersion++;
    mData = value;
    dispatchingValue(null);
}

protected void postValue(T value) {
    boolean postTask;
    synchronized (mDataLock) {
        postTask = mPendingData == NOT_SET;
        mPendingData = value;
    }
    if (!postTask) {
        return;
    }
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}
```
But of course the two functions require a bit more work.

>Examining `setValue`

The `setValue` will first check the function is called on **MainThread** before proceeding.

After setting the value, it will call `dispatchingValue`, which takes in a **ObserverWrapper** and sent notification to them all. If the input was `null`, then all **ObserverWrapper** will be notified :

```java
void dispatchingValue(@Nullable ObserverWrapper initiator) {
    if (mDispatchingValue) {
        mDispatchInvalidated = true;
        return;
    }
    mDispatchingValue = true;
    do {
        mDispatchInvalidated = false;
        if (initiator != null) {
            considerNotify(initiator);
            initiator = null;
        } else {
            for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                    mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                considerNotify(iterator.next().getValue());
                if (mDispatchInvalidated) {
                    break;
                }
            }
        }
    } while (mDispatchInvalidated);
    mDispatchingValue = false;
}
```
>Examining `postValue`

Since `postValue` is available for all multiple threads, it uses synchronized to make sure only a single thread can have access to the block of code :
```java
synchronized (mDataLock) {
    postTask = mPendingData == NOT_SET;
    mPendingData = value;
}
```

Here the `mPendingData` is `volatile` object and it is used to keep track if `mPostValueRunnable` has finished running or if the data has been posted yet.

Let's try to walkthrough what happen during this function.

1. Initially, `mPendingData` is set to `NOT_SET`.

2. As new value passed in to the function
   - the local `postTask` will become `true`

   - `mPendingData` will set to `newValue`


3. Since `postTask == true`, **ArchTaskExecutor** will be executed, calling `postToMainThread(mPostValueRunnable)`

   - this function will run create a **Handler** on the **Looper** of MainThread and `post` the **Runnable** with it.

   - this will make sure the **Runnable** is executed on the MainThread
   - If you wish to learn more on Handler and Looper, check out this [post]().


4. Once the `mPostValueRunnable` is executed, it will grab the new value, set `mPendingData` back to `NOT_SET` and call `setValue` to set the data.

```java
protected void postValue(T value) {
    boolean postTask;
    synchronized (mDataLock) {
        postTask = mPendingData == NOT_SET;
        mPendingData = value;
    }
    if (!postTask) {
        return;
    }
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}

private final Runnable mPostValueRunnable = new Runnable() {
    @SuppressWarnings("unchecked")
    @Override
    public void run() {
        Object newValue;
        synchronized (mDataLock) {
            newValue = mPendingData;
            mPendingData = NOT_SET;
        }
        setValue((T) newValue);
    }
};
```

>**Question Time**
><br>Why do we not need to call `setValue` or `postValue` when setting the initial value ?
><br>How would the listeners know the value we have ?

To answer these questions, let's take a look at what happen when we add observers.

### What happen when observer is added ?
If you can recall how **LiveData** can be used, you should remember that to observe the changes on **LiveData**, you must called `observe` method on it.

```Java
_viewModel.didRequestFetchData.observe(this) {
    if(it) {
      // do something if new value is true
    } else {
      // do something if new value is false
    }
}
```

To observe a **LiveData**, you must have 2 things :
- a **lifecycleOwner**, and
- an observer that implemented **Observer** interface.
  ```Java
  public interface Observer<T> {
    void onChanged(T t);
}
  ```

>**Question Time**
><br>Why do we need **LifecycleOwner** when observing the data ?

The answer can be found in the `observe` method :

```Java
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    assertMainThread("observe");
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    owner.getLifecycle().addObserver(wrapper);
}
```

When `observe` is called, it will do the followings :
1. make sure the function is triggered on `MainThread`

2. check the state of the lifecycleOwner is not `DESTROYED`

3. create a **LifecycleBoundObserver**, a subclass of **ObserverWrapper**, using both **Observer** and **LifecycleOwner**
   - this class is used to response upon the changes in the **LifecycleOwner**'s state.
   - we will take a closer look on it later.


4. make sure the **ObserverWrapper** is added the first time and it is not attached to a **LifecycleOwner**.


5. attach the wrapper as an observer to **LifecycleOwner**

>Thus, **LifecycleOwner** is needed for **LifecycleBoundObserver** to know when is the **LifecycleOwner** got `DESTROYED`.

#### LifecycleBoundObserver
>**ObserverWrapper** is the parent class of **LifecycleBoundObserver**

It is a simple wrapper that holds a **Observer** and a boolean showing whether it is active or not.

The main method in **ObserverWrapper** is `activeStateChanged`, which is triggered by **LifecycleBoundObserver** when it sees the state of the **LifecycleOwner** has changed.
```Java
void activeStateChanged(boolean newActive) {
    if (newActive == mActive) {
        return;
    }
    // immediately set active state, so we'd never dispatch anything to inactive
    // owner
    mActive = newActive;
    changeActiveCounter(mActive ? 1 : -1);
    if (mActive) {
        dispatchingValue(this);
    }
}
```

Once the `activeStateChanged` is triggered, it will update the `int mActiveCount` to keep track how many observers are currently active.

Then, it will call `dispatchingValue` to notify the **Observer** that's wrapped inside.

>**LifecycleBoundObserver** is a subclass of **ObserverWrapper**
><br><br>In additional to the **Observer** that is wrapped in **ObserverWrapper**, **LifecycleBoundObserver** also holds a **LifecycleOwner**.

The main purpose of **LifecycleBoundObserver** is to observe the changes on **LifecycleOwner** state through `onStateChanged`.

```Java
@Override
public void onStateChanged(@NonNull LifecycleOwner source,
        @NonNull Lifecycle.Event event) {
    Lifecycle.State currentState = mOwner.getLifecycle().getCurrentState();
    if (currentState == DESTROYED) {
        removeObserver(mObserver);
        return;
    }
    Lifecycle.State prevState = null;
    while (prevState != currentState) {
        prevState = currentState;
        activeStateChanged(shouldBeActive());
        currentState = mOwner.getLifecycle().getCurrentState();
    }
}
```

In order to trigger this method, it needs to be added as an observer to **LifecycleOwner**, just like what happen in `observe` method.

```Java
owner.getLifecycle().addObserver(wrapper);
```
When the **Observer** is being added, `dispatchEvent` will be triggered on the **Observer** multiple times until it reaches the same state as **LifecycleOwner**.

```Java
// addObserver in LifecycleRegistry.Java
@Override
public void addObserver(@NonNull LifecycleObserver observer) {
    enforceMainThreadIfNeeded("addObserver");
    State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
    ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
    ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);

    if (previous != null) {
        return;
    }
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        // it is null we should be destroyed. Fallback quickly
        return;
    }

    boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
    State targetState = calculateTargetState(observer);
    mAddingObserverCounter++;
    while ((statefulObserver.mState.compareTo(targetState) < 0
            && mObserverMap.contains(observer))) {
        pushParentState(statefulObserver.mState);
        final Event event = Event.upFrom(statefulObserver.mState);
        if (event == null) {
            throw new IllegalStateException("no event up from " + statefulObserver.mState);
        }
        statefulObserver.dispatchEvent(lifecycleOwner, event);
        popParentState();
        // mState / subling may have been changed recalculate
        targetState = calculateTargetState(observer);
    }

    if (!isReentrance) {
        // we do sync only on the top level.
        sync();
    }
    mAddingObserverCounter--;
}
```

By triggering `dispatchEvent`, it will call the `onStateChanged` in **LifecycleBoundObserver**
```Java
void dispatchEvent(LifecycleOwner owner, Event event) {
    State newState = event.getTargetState();
    mState = min(mState, newState);
    mLifecycleObserver.onStateChanged(owner, event);
    mState = newState;
}
```
Upon calling `onStateChanged`, `activeStateChanged` in **ObserverWrapper** will be triggered, which will eventually `dispatchingValue`.

>Now you should understand why we don't need to call `setValue` or `postValue` if we set the initial value.
><br>**Summary**

<figure>
<center>
<img src = "/images/posts/jekyll/2022-09-29-android-livedata_onstatechanged.svg" style = "width:90%"/>
</center>
</figure>

### ObserveForever
>Now that we know how `observe` works, but what about `observeForever` ?

Instead of creating a **LifecycleBoundObserver** like `observe`, this method will create **AlwaysActiveObserver**.

```Java
@MainThread
public void observeForever(@NonNull Observer<? super T> observer) {
    assertMainThread("observeForever");
    AlwaysActiveObserver wrapper = new AlwaysActiveObserver(observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing instanceof LiveData.LifecycleBoundObserver) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    wrapper.activeStateChanged(true);
}
```
Since it is always active, it doesn't need to keep track of the lifecycle.

```Java
private class AlwaysActiveObserver extends ObserverWrapper {

    AlwaysActiveObserver(Observer<? super T> observer) {
        super(observer);
    }

    @Override
    boolean shouldBeActive() {
        return true;
    }
}
```
At the end of `observeForever`, `activeStateChanged(true)` will be triggered.

```Java
void activeStateChanged(boolean newActive) {
    if (newActive == mActive) {
        return;
    }
    // immediately set active state, so we'd never dispatch anything to inactive
    // owner
    mActive = newActive;
    changeActiveCounter(mActive ? 1 : -1);
    if (mActive) {
        dispatchingValue(this);
    }
}
```
This will make sure `dispatchingValue` is called whenever value update.


### Summary

**LiveData** is a class that can be observed by **Observer**s.

If you wish to alternate the data, you will need to use **MutableLiveData**.

```Java
class MyViewModel: ViewModel() {
  val didRequestFetchData: LiveData<Boolean>
  get = _didRequestFetchData

  val _didRequestFetchData = MutableLiveData<Boolean>(false)

  fun makeARequest() {
    _didRequestFetchData.value = true;
  }
}
```

To display the data on UI, an observer needs to be created in the **LifecycleOwner** instance, such as **Activity** and **Fragment** and implements the **Observer** interface.

```Java
_viewModel.livedata.observe(this) {
  // do update
}
```

You can use either `observe` or `observeForever` to observe **LiveData**.

>`observe` will use **LifecycleBoundObserver** to observe and response upon lifecycle state changes.
><br><br>Whereas, `observeForever` uses **AlwaysActiveObserver**, which would be triggered whenever the value is updated.


### What's Next ?
That's about it on the topic on **LiveData**.

If you wish to learn other things that are related to **LiveData**, you can start by reading **ViewModel**.

Here are some of the posts available :
**COMING SOON**
