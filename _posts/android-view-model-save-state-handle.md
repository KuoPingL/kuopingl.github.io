---
layout: post
title:  "Android - ViewModel : How ViewModel Saves and Restores Data "
date:   2022-08-08 16:15:07 +0800
categories: [android jetpack, view model, beginner]
---

### Background
>Here is the official diagram showing the lifecycle of **ViewModel**

<figure>
<center>
<a href="https://developer.android.com/topic/libraries/architecture/viewmodel#lifecycle">
<img src="/images/common/viewmodel-lifecycle.png" style="width:80%"/>
</a>
</center>
</figure>

<br>

From the diagram, we can **see** that a **ViewModel** has a longer lifecycle compared to **Activity** or **Fragment**. Thus **ViewModel** can help us save datas on the page and restores it when needed, such as when `rotation` occur.

However, ever wonder how did that happen ?

In this article, we will look into the lifecycle of **ViewModel** and how it helps us in saving and restoring data.

### Lifecycle of a ViewModel
The lifecycle of a **ViewModel** consisted of 2 stages :
- Create
- Destroy

<b><u>What about in between?</b></u>

Well, after **ViewModel** is created, it would simply stay inside **ViewModelStore** until destroyed.

#### Creation of ViewModel
The creation of a **ViewModel** is typically happen during the `onCreate` method of **Activity** or `onViewCreated` method of **Fragment** by calling :

```java
ViewModelProvider(this).get(MyViewModel::class.java)
```

>**Note** : The reason for **ViewModelProvider** to take `this`, is not because they are **LifecycleOwner** but because they are **ViewModelStoreOwner**.
><br><br>This means, **ViewModel**'s lifecycle is independent from the ones in **Activity** or **Fragment**.

**ViewModelProvider** will create the target **ViewModel** through **ViewModelProvider.Factory** or fetch it from **ViewModelStore**, where the created **ViewModel** is cached.

By caching it, we are certain that before **ViewModelStore** got cleared, we will always get the same **ViewModel** instance. Thus, they should contain the same data whenever `onCreate` or `onViewCreated` is called.

The **ViewModel** will be stored as long as **ViewModelStore** is not destroyed or cleared.

#### Destruction of ViewModel
To destroy a **ViewModel**, we need to call `clear` method in **ViewModelStore**

```Java
public final void clear() {
    for (ViewModel vm : mMap.values()) {
        vm.clear();
    }
    mMap.clear();
}
```

Let's take a look when does **Activity** or **Fragment** call this function.

>**In Activity**
><br>`clear` method will be triggered when `Lifecycle.Event.ON_DESTROY` is observed by **LifecycleEventObserver**.
><br><br>The observer is added inside the constructor of **ComponentActivity**

```Java
// ComponentActivity.java
getLifecycle().addObserver(new LifecycleEventObserver() {
    @Override
    public void onStateChanged(@NonNull LifecycleOwner source,
            @NonNull Lifecycle.Event event) {
        if (event == Lifecycle.Event.ON_DESTROY) {
            // Clear out the available context
            mContextAwareHelper.clearAvailableContext();
            // And clear the ViewModelStore
            if (!isChangingConfigurations()) {
                getViewModelStore().clear();
            }
        }
    }
});
```

>**As for Fragment**
><br>It is more complicated
<figure>
<center>
<img src = "/images/posts/jekyll/2022-09-29-android-view-model-save-restore-fragment-clear.svg" style="width:80%"/>
</center>
</figure>

<br>

We can find `viewModelStore.clear()` is triggered in `clearNonConfigState(Fragment)` inside **FragmentManagerViewModel**:

```Java
void clearNonConfigState(@NonNull Fragment f) {
    // ...
    // Clear and remove the Fragment's ViewModelStore
    ViewModelStore viewModelStore = mViewModelStores.get(f.mWho);
    if (viewModelStore != null) {
        viewModelStore.clear();
        mViewModelStores.remove(f.mWho);
    }
}
```

This is triggered by **FragmentStateManager** in `destroy`:
```Java
void destroy() {
    if (FragmentManager.isLoggingEnabled(Log.DEBUG)) {
        Log.d(TAG, "movefrom CREATED: " + mFragment);
    }
    boolean beingRemoved = mFragment.mRemoving && !mFragment.isInBackStack();
    boolean shouldDestroy = beingRemoved
            || mFragmentStore.getNonConfig().shouldDestroy(mFragment);
    if (shouldDestroy) {
        // ...
        if (beingRemoved || shouldClear) {
            mFragmentStore.getNonConfig().clearNonConfigState(mFragment);
        }
        // ...
        mFragmentStore.makeInactive(this);
    } else {
        // ...
        mFragment.mState = Fragment.ATTACHED;
    }
}
```
And `destroy` is triggered by **FragmentManager** whenever the fragment go from `Fragment.CREATED` to `Fragment.ATTACHED` then `fragmentStateManager.destroy();` will be triggered.

So from what we can tell, **ViewModelStore** will be cleared in the following situations :
- When the state of **Activity** becomes `ON_DESTROY` and it is **not** caused by configuration changes.

- **Fragment**

TODO: CLEARIFY when will `FragmentManager.destroy` be triggered.

### Becoming the Bridge
In order for **ViewModel** to become the bridge between View and Model, **LiveData** is needed.

A **LiveData<T>** is an abstract class that holds a `volatile Object` and a version number.




Here, I am assuming that you know to create a **ViewModel**, we must first create a **ViewModelProvider** and through the `get` method, create a **ViewModel** from **ViewModelProvider.Factory**.

```java
ViewModelProvider(this).get(MyViewModel::class.java)
```

During the creation of **ViewModel**, it will first check if **ViewModelStore** has an instance of it :

```java
val viewModel = store[key]
if (modelClass.isInstance(viewModel)) {
    (factory as? OnRequeryFactory)?.onRequery(viewModel)
    return viewModel as T
} else {
    @Suppress("ControlFlowWithEmptyBody")
    if (viewModel != null) {
        // TODO: log a warning.
    }
}
```

If nothing was found, then a new **ViewModel** will be created and stored in the **HashMap** of a **ViewModelStore**.

```java
// ViewModelProvider.kt
try {
  factory.create(modelClass, extras)
} catch (e: AbstractMethodError) {
  factory.create(modelClass)
}.also { store.put(key, it) }
```

If we ever need the **ViewModel** again, **ViewModelProvider** can simple extract it from the **ViewModelStore**.

We can also call the `clear` method in **ViewModelStore** to clear all saved **ViewModel**
```java
// ViewModelStore.java
public final void clear() {
    for (ViewModel vm : mMap.values()) {
        vm.clear();
    }
    mMap.clear();
}
```

Now that we know how a **ViewModel** can be `created`, `restored` and `destroyed`, we can now examine how **ViewModel** plays a part during both **Activity** and **Fragment** lifecycle.

### ViewModel in Activity
In **Activity** we usually creates the **ViewModel** in `onCreate` method :
```java
class MyActivity: AppCompatActivity() {

  private val _viewModel: MainViewModel by lazy {
    ViewModelProvider(this).get(MainViewModel::class.java)
  }

  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)

    _viewModel.data.observe(this) {
      if (it != null) {
        // update UI
      }
    }
  }
}
```

As long as the `clear` method of **ViewModelStore** was not triggered, we can be sure that **ViewModelProvider** will return the same instance of **ViewModel** as before.

#### Update UI through LiveData
Once `_viewModel.data.observe(this)` is reached, it will trigger
