---
layout: post
title:  "Android - ViewModel : SavedStateViewModel"
date:   2022-08-08 16:15:07 +0800
categories: [android jetpack, viewmodel, beginner]
---

### Background
From the previous [post]() on **ViewModel**, we get to see how a **ViewModel** works and how we can use **ViewModelProvider**, **ViewModelProvider.Factory** and **CreationExtras** to generate the **ViewModel** we need.

In this article, I want to focus on **SavedStateViewModelFactory** and **AbstractSavedStateViewModelFactory**, which I didn't get a chance to talk about.

### SavedStateViewModelFactory vs AbstractSavedStateViewModelFactory
First, let's take a look at there differences.
#### Declaration

```java
// SavedStateViewModelFactory.kt
class SavedStateViewModelFactory : ViewModelProvider.OnRequeryFactory, ViewModelProvider.Factory{}

// AbstractSavedStateViewModelFactory.java
public abstract class AbstractSavedStateViewModelFactory extends ViewModelProvider.OnRequeryFactory
        implements ViewModelProvider.Factory {}
```

From the class, we see the main difference between is that **AbstractSavedStateViewModelFactory** is an abstract class, while **SavedStateViewModelFactory** is a class.

#### Constructor

```java
class SavedStateViewModelFactory : ViewModelProvider.OnRequeryFactory, ViewModelProvider.Factory {
  constructor() {
      factory = ViewModelProvider.AndroidViewModelFactory()
  }

  constructor(
      application: Application?,
      owner: SavedStateRegistryOwner
  ) : this(application, owner, null)

  @SuppressLint("LambdaLast")
  constructor(application: Application?, owner: SavedStateRegistryOwner, defaultArgs: Bundle?) {
      savedStateRegistry = owner.savedStateRegistry
      lifecycle = owner.lifecycle
      this.defaultArgs = defaultArgs
      this.application = application
      factory = if (application != null) getInstance(application)
          else ViewModelProvider.AndroidViewModelFactory()
  }

  /*
  // ViewModelProvider.java
  @JvmStatic
  public fun getInstance(application: Application): AndroidViewModelFactory {
      if (sInstance == null) {
          sInstance = AndroidViewModelFactory(application)
      }
      return sInstance!!
  }

  */
}
```


```java
@SuppressLint("LambdaLast")
public AbstractSavedStateViewModelFactory(@NonNull SavedStateRegistryOwner owner,
        @Nullable Bundle defaultArgs) {
    mSavedStateRegistry = owner.getSavedStateRegistry();
    mLifecycle = owner.getLifecycle();
    mDefaultArgs = defaultArgs;
}
```

Based on their constructors, their main differences are as follows:

|Parameters|**SavedStateViewModelFactory**|**AbstractSavedStateViewModelFactory**|
|:--|:--:|:--:|
|**Application** | optional  | x |
|**SavedStateRegistryOwner**| optional  |  required |
|**Bundle**   | optional  | optional  |

If we look closer, we can tell that
- **SavedStateViewModelFactory** will use **AndroidViewModelFactory** as the default factory
- **AbstractSavedStateViewModelFactory** does not have any factory.

Next, we will take a look at how they implements **ViewModelProvider.Factory**.

#### Implementation of ViewModelProvider.Factory
>First, let's take a look at how **SavedStateViewModelFactory** implements **Factory**

##### SavedStateViewModelFactory
<details>
<summary><b>create(Class<T>)</b></summary>

```Java
override fun <T : ViewModel> create(modelClass: Class<T>): T {
    val canonicalName = modelClass.canonicalName
        ?: throw IllegalArgumentException("Local and anonymous classes can not be ViewModels")
    return create(canonicalName, modelClass)
}

```

</details>

- This method will get the `canonicalName` of modelClass as the key and pass it to `create(String, Class)`

<details>
<summary><b>create(String, Class)</b></summary>

```Java
fun <T : ViewModel> create(key: String, modelClass: Class<T>): T {
    // empty constructor was called.
    if (lifecycle == null) {
        throw UnsupportedOperationException(
            "SavedStateViewModelFactory constructed with empty constructor supports only " +
                "calls to create(modelClass: Class<T>, extras: CreationExtras)."
        )
    }
    val isAndroidViewModel = AndroidViewModel::class.java.isAssignableFrom(modelClass)
    val constructor: Constructor<T>? = if (isAndroidViewModel && application != null) {
        findMatchingConstructor(modelClass, ANDROID_VIEWMODEL_SIGNATURE)
    } else {
        findMatchingConstructor(modelClass, VIEWMODEL_SIGNATURE)
    }
    // doesn't need SavedStateHandle
    if (constructor == null) {
        // If you are using a stateful constructor and no application is available, we
        // use an instance factory instead.
        return if (application != null) factory.create(modelClass)
            else instance.create(modelClass)
    }
    val controller = LegacySavedStateHandleController.create(
        savedStateRegistry, lifecycle, key, defaultArgs
    )
    val viewModel: T = if (isAndroidViewModel && application != null) {
        newInstance(modelClass, constructor, application!!, controller.handle)
    } else {
        newInstance(modelClass, constructor, controller.handle)
    }
    viewModel.setTagIfAbsent(
        AbstractSavedStateViewModelFactory.TAG_SAVED_STATE_HANDLE_CONTROLLER, controller
    )
    return viewModel
}
```

</details>

>This method will do the followings :
  1. Make sure this class owns a **Lifecycle**.
  2. Uses `isAssignableFrom` to see if the modelClass is a **AndroidViewModel**.
  3. call `findMatchingConstructor` and use the Class and Parameters types to find the corresponding constructor.
  4. `return` If `isAndroidViewModel == true`, `constructor == null` and `application != null`, then create it using **AndroidViewModelFactory**.
  5. `return` If `isAndroidViewModel == false`,`constructor == null` and `application == null`, then create it using **NewInstanceFactory**.
  6. If `constructor != null`, a **LegacySavedStateHandleController** will be created.
  7. Then, a ViewModel will be created using `newInstance(Class<T>, Constructor<T>,
    vararg params: Any)`

```java
internal fun <T : ViewModel?> newInstance(
    modelClass: Class<T>,
    constructor: Constructor<T>,
    vararg params: Any
): T {
    return try {
        constructor.newInstance(*params)
    } catch (e: IllegalAccessException) {
        throw RuntimeException("Failed to access $modelClass", e)
    } catch (e: InstantiationException) {
        throw RuntimeException("A $modelClass cannot be instantiated.", e)
    } catch (e: InvocationTargetException) {
        throw RuntimeException(
            "An exception happened in constructor of $modelClass", e.cause
        )
    }
}
```

We will take some time to understand what `findMatchingConstructor` and **LegacySavedStateHandleController** does, in the next section.

<details>
<summary><b>create(Class, CreationExtras)</b></summary>

```Java
override fun <T : ViewModel> create(modelClass: Class<T>, extras: CreationExtras): T {
    val key = extras[ViewModelProvider.NewInstanceFactory.VIEW_MODEL_KEY]
        ?: throw IllegalStateException(
            "VIEW_MODEL_KEY must always be provided by ViewModelProvider"
        )

    return if (extras[SAVED_STATE_REGISTRY_OWNER_KEY] != null &&
        extras[VIEW_MODEL_STORE_OWNER_KEY] != null) {
        val application = extras[ViewModelProvider.AndroidViewModelFactory.APPLICATION_KEY]
        val isAndroidViewModel = AndroidViewModel::class.java.isAssignableFrom(modelClass)
        val constructor: Constructor<T>? = if (isAndroidViewModel && application != null) {
            findMatchingConstructor(modelClass, ANDROID_VIEWMODEL_SIGNATURE)
        } else {
            findMatchingConstructor(modelClass, VIEWMODEL_SIGNATURE)
        }
        // doesn't need SavedStateHandle
        if (constructor == null) {
            return factory.create(modelClass, extras)
        }
        val viewModel = if (isAndroidViewModel && application != null) {
            newInstance(modelClass, constructor, application, extras.createSavedStateHandle())
        } else {
            newInstance(modelClass, constructor, extras.createSavedStateHandle())
        }
        viewModel
    } else {
        val viewModel = if (lifecycle != null) {
            create(key, modelClass)
        } else {
            throw IllegalStateException("SAVED_STATE_REGISTRY_OWNER_KEY and" +
                "VIEW_MODEL_STORE_OWNER_KEY must be provided in the creation extras to" +
                "successfully create a ViewModel.")
        }
        viewModel
    }
}
```

</details>

> This function is pretty much the same as `create(String, Class)`.
> <br><br>
> but instead of creating the **LegacySavedStateHandleController**, the controller was fetched from **CreationExtras** by `extras.createSavedStateHandle()`


###### findMatchingConstructor
> I think this is a very interesting function, you will be able to learn how it uses reflection to find the constructor of interest.
<details>
<summary><b>findMatchingConstructor(Class<T>, List&lt;Class&lt; * >>)</b></summary>

```Java

internal fun <T> findMatchingConstructor(
    modelClass: Class<T>,
    signature: List<Class<*>>
): Constructor<T>? {
    for (constructor in modelClass.constructors) {
        val parameterTypes = constructor.parameterTypes.toList()
        if (signature == parameterTypes) {
            @Suppress("UNCHECKED_CAST")
            return constructor as Constructor<T>
        }
        if (signature.size == parameterTypes.size && parameterTypes.containsAll(signature)) {
            throw UnsupportedOperationException(
                "Class ${modelClass.simpleName} must have parameters in the proper " +
                    "order: $signature"
            )
        }
    }
    return null
}
```

</details>

> This class does the followings
> 1. loop through `modelClass.constructors`
> 2. compare the parameters, `constructor.parameterTypes.toList()`, with the given parameters.
> 3. if `signature == parameterTypes` it will `return` the constructor, because their order, size and types are the same.
> 4. if they are in the wrong order, then **UnsupportedOperationException** will be thrown.
> 5. if nothing found, `null` will be returned.


##### AbstractSavedStateViewModelFactory

<details>
<summary><b>create(Class<T>)</b></summary>

```Java
@NonNull
@Override
public final <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
    // ViewModelProvider calls correct create that support same modelClass with different keys
    // If a developer manually calls this method, there is no "key" in picture, so factory
    // simply uses classname internally as as key.
    String canonicalName = modelClass.getCanonicalName();
    if (canonicalName == null) {
        throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
    }
    if (mLifecycle == null) {
        throw new UnsupportedOperationException(
                "AbstractSavedStateViewModelFactory constructed "
                        + "with empty constructor supports only calls to "
                        +   "create(modelClass: Class<T>, extras: CreationExtras)."
        );
    }
    return create(canonicalName, modelClass);
}
```

</details>

> Similar to **SavedStateViewModelFactory**, the `create(Class<T>)` method will call `create(String, Class<T>)` method.
><br><br>
>This is because they both need to create **SavedStateHandleController**, which requires String key, **Lifecycle** and **SavedStateRegistry** parameters.
><br><br>
>The main difference is that **AbstractSavedStateViewModelFactory** will do the lifecycle checking in this function, instead in `create(String, Class<T>)`.

<details>
<summary><b>create(String, Class<T>)</b></summary>

```Java
@NonNull
private <T extends ViewModel> T create(@NonNull String key, @NonNull Class<T> modelClass) {
    SavedStateHandleController controller = LegacySavedStateHandleController
            .create(mSavedStateRegistry, mLifecycle, key, mDefaultArgs);
    T viewmodel = create(key, modelClass, controller.getHandle());
    viewmodel.setTagIfAbsent(TAG_SAVED_STATE_HANDLE_CONTROLLER, controller);
    return viewmodel;
}
```

</details>

>Comparing with **SavedStateViewModelFactory**, **AbstractSavedStateViewModelFactory** does not need to check if the ViewModel is **AndroidViewModel** or not, it simply calls `create(String, Class<T>, SavedStateHandle)` to create the ViewModel, and save the **SavedStateHandleController** in it.

<details>
<Summary><b>create(String, Class<T>, SavedStateHandle)</b></summary>

```Java
@NonNull
protected abstract <T extends ViewModel> T create(@NonNull String key,
        @NonNull Class<T> modelClass, @NonNull SavedStateHandle handle);
```

</details>

>This is the method that we have to implement if we are subclassing **AbstractSavedStateViewModelFactory**.


### Implementation of ViewModelProvider.OnRequeryFactory

**ViewModelProvider.OnRequeryFactory** is a SAM :
```java
@RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
public open class OnRequeryFactory {
    public open fun onRequery(viewModel: ViewModel) {}
}
```
The main purpose this method is to attach the **SavedStateHandleController** in the viewModel to the lifecycle if needed.

So both **SavedStateViewModelFactory** and **AbstractSavedStateViewModelFactory** share the same implementation.

```Java
@RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
override fun onRequery(viewModel: ViewModel) {
    // needed only for legacy path
    if (lifecycle != null) {
        LegacySavedStateHandleController.attachHandleIfNeeded(
            viewModel,
            savedStateRegistry,
            lifecycle
        )
    }
}
```

### Summary
Based on their implementations, it is clear that they are practically doing the same thing.

Except the following differences :

||**SavedStateViewModelFactory**|**AbstractSavedStateViewModelFactory**|
|:--|:--|:--|
| Extendable or Implementable  |  NO (final class)| YES (abstract class)  |
|Provides default Factory   | YES (**AndroidViewModelFactory** and **NewInstanceFactory**)  |  NO |

Now that we have seen their differences, let's take a look at some of the terms that we've come across.

### Extra
In this section, we will take a look at the classes and functions that we came across but didn't get the chance to talk about it, including :
- **SavedStateHandle**
- **SavedStateHandleController**
- **LegacySavedStateHandleController**


#### SavedStateHandle
>A handle to saved state passed down to **ViewModel**



### How to implement AbstractSavedStateViewModelFactory
