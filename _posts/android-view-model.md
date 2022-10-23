---
layout: post
title:  "Android - ViewModel : an intro"
date:   2022-08-08 16:15:07 +0800
categories: [android jetpack, viewmodel, beginner]
---

### Background
**ViewModel** plays an essential role in **MVVM** architecture.

**MVVM** is composed of 3 roles :
- **V**iew <br>this is responsible for user interactions
- **M**odel <br>this is responsible for storing and fetching datas, locally and remotely.
- **V**iew**M**odel <br>this is responsible for the communication between User and Data.

![](/images/common/itsMagic.jpeg)

So the process goes like this :
- When user interacts with the **V**, **V** will send a request to **VM**.
- Then, **VM** will pass or make another request to **M**.
- **M** will fetch data as requested, and pass the data back to **VM**.
- And finally, **V** will be notified by **VM** when the data is ready.

For this process to occur, we usually use **callbacks** to pass data from **M** and **VM**.
<br>And use **listener** or **Observer Design Pattern** to pass data from **VM** to **V**.

>**What makes ViewModel so important ?**

We should already know about that fact that Activity or Fragment loses all data whenever configuration changes occur.

> Prior to **ViewModel**, these changes usually need to be store in other forms, most likely, inside a **Bundle**.

The saving and restoring needs to be done inside the `onSaveInstanceState` and `onRestoreInstanceState` methods, respectively.

Though this wouldn't be much of a problem for small data, but it will become more tedious and error prone when more data need to be saved and restored.

Thanks to the introduction of **ViewModel**, we can simply set the properties inside it and no need to manually perform save and restore when configuration change occur.

"**How does it work?**", you asked
<br>Well, we are about to find out.

Let's start by knowing how to use a **ViewModel** properly.

### Creating a ViewModel - the wrong way
In order to create a **ViewModel**, we need to implement the *abstract class* :

```Java
public abstract class ViewModel {}
```
Here's a simple implementation :

```Java
class MainViewModel: ViewModel() {}
```
>Next, we will add a function that allows the Activity to control the text of a button

```Java
class MainViewModel: ViewModel() {

    val buttonText: LiveData<String>
    get() = _buttonText

    private val _buttonText = MutableLiveData<String>()

    fun incrementButtonCount(text: String) {
        var v= text.toInt()
        _buttonText.postValue((++v).toString())
    }
}
```

Why using **LiveData** ? You will find out soon.

>Now that we have **ViewModel**, we will call it in the Activity.
><br> *Note how MainViewModel is being created.*

```Java

private val _viewModel: MainViewModel = MainViewModel()

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)

    _viewModel.buttonText.observe(this) {
        findViewById<AppCompatButton>(R.id.btn).text = it
    }

    findViewById<AppCompatButton>(R.id.btn).setOnClickListener {
        _viewModel.incrementButtonCount(findViewById<AppCompatButton>(R.id.btn).text.toString())
    }
}
```

>And here is what will happen :

The number will increment by pressing the button.

<center><img src="/images/posts/jekyll/2022-09-26-android-view-model-wrong-way-1.png" style="width:50%"/></center>

<br>But, once the screen rotates, the number will return to its initial state :

<center><img src="/images/posts/jekyll/2022-09-26-android-view-model-wrong-way-2.png" style="width:80%"/></center>

>**Hey, where is the data will save automatically feature ? You liar.**
><br> Wait, hear me out.

<br>Obviously this is due to the recreation of the **ViewModel** everytime Activity got recreated.

> **So, how do we solve it?**

We will need a **ViewModelProvider** :

```Java
private val _viewModel: MainViewModel by lazy {
    ViewModelProvider(this).get(MainViewModel::class.java)
}
```

Now if we try again, the number will stay up to date no matter how you flip the phone.
<br>
<center><img src="/images/posts/jekyll/2022-09-26-android-view-model-right-way-1.png" style="width:80%"/></center>
<br>

So how does a **ViewModelProvider** help ?

Let's take a look at what is a **ViewModelProvider** and how it helps to solve the problem.

### ViewModelProvider
> A **ViewModelProvider** is a class that uses the following classes to provide the **ViewModel** we need
> - **ViewModelStore**
> - **ViewModelProvider.Factory**
> - **CreationExtras**

```Java
public open class ViewModelProvider
  @JvmOverloads constructor (
    private val store: ViewModelStore,
    private val factory: Factory,
    private val defaultCreationExtras: CreationExtras = CreationExtras.Empty,
  ) {  }
```
Let's take a closer look at what each parameter is and what they do.

#### ViewModelStore
>A **ViewModelStore** is a class that help store **ViewModel**s in a **HashMap**.
><br> So this is the place that keeps our **ViewModel** alive.

```java
private final HashMap<String, ViewModel> mMap = new HashMap<>();
```

As for the Key that is being used to store or get the **ViewModel** can be found in **ViewModelProvider**'s `get` method :

```java
@MainThread
public open operator fun <T : ViewModel> get(modelClass: Class<T>): T {
    val canonicalName = modelClass.canonicalName
        ?: throw IllegalArgumentException("Local and anonymous classes can not be ViewModels")
    return get("$DEFAULT_KEY:$canonicalName", modelClass)
}
```

In network, a **Canonical Name** (CNAME) is a record in the DNS database that indicates the true host name of a computer associated with its aliases. It is essential when running multiple services from a single IP address.

It's similar here, but the naming convension will follow the **Java Language Specification**.

> an example : `com.project.name.classname`

Classes that holds a **ViewModelStore** must also implement the **ViewModelStoreOwner** interface.

##### ViewModelStoreOwner
>This is a SAM interface
```Java
@NonNull
ViewModelStore getViewModelStore();
```

Classes that implements it include :
- ComponentActivity
- Fragment

As a result, they can easily get a **ViewModelStore** from these classes to save **ViewModel**s they need.

However, to create a **ViewModel** we need a **ViewModelProvider.Factory** to do so.

#### ViewModelProvider.Factory
>A **ViewModelProvider.Factory** is the Factory **Interface** that is responsible for creating **ViewModel** using the following methods :

- `create(Class<T>): T`
- `create(Class<T>, CreationExtras): T`

So obviously, the creation of **ViewModel** implements the **Factory Design Pattern**.

There are a set of prebuilt **ViewModelProvider.Factory** available at our disposal, including :
- **NewInstanceFactory**
- **AndroidViewModelFactory**
- **SavedStateViewModelFactory**
- **AbstractSavedStateViewModelFactory**

Let's take a look at what these factories can create.

|Factory|ViewModel|
|:--|:--|
|**NewInstanceFactory**|This will create any **ViewModel** by calling `modelClass.newInstance()`|
|**AndroidViewModelFactory**|This will take in a **nullable** `Application`. <br> **null** &rarr; call `modelClass.newInstance()` <br>else &rarr; create a **AndroidViewModel**   |
|**SavedStateViewModelFactory**|This class is used by **ComponentActivity** and **Fragment**, which will create a simple **ViewModel** or **AndroidViewModel** based on the parameters in their constructor.|
|**AbstractSavedStateViewModelFactory**|This will take in a **SavedStateHandle** when creating a **ViewModel**|

>**NOTE** : I won't spend my time on discussing on **SavedStateHandle** and **AbstractSavedStateViewModelFactory** here, since it would take another post to do so.

>But, what if our **ViewModel** required customized constructor and not of them mentioned fit my requirements ?
<br><br>Let's say if I want to put Local and Remote Repository ?

In that case, we will have to create a subclass.

But first, let's take a quick look at what **CreationExtras** is.

#### CreationExtras
> **CreationExtras** is an abstract class for storing data.
> <br><br>
> What's interesting is that it uses generic as a type checker.

```Java
public abstract class CreationExtras internal constructor() {
    internal val map: MutableMap<Key<*>, Any?> = mutableMapOf()

    public interface Key<T>

    public abstract operator fun <T> get(key: Key<T>): T?

    /**
     * Empty [CreationExtras]
     */
    object Empty : CreationExtras() {
        override fun <T> get(key: Key<T>): T? = null
    }
}
```

Here's where the magic occur :
> A key that holds a type `T` can only get the same type (`T`) of object.

For instance, if I were to store and retrive a Repository object from the class, I will have to do the followings :
>**Prepare a CreationExtras.Key<`T`>**
><br>This will be used as the Key for retrieving type `T`

```Java
@JvmField
val REPOSITORY_KEY = object : CreationExtras.Key<Repo> {}
```

>**Create a MutableCreationExtras**
><br>It is not possible to create a **CreationExtras** because its constructor is set as internal

```Java
val repoBasedCE = MutableCreationExtras().apply {
    set(REPOSITORY_KEY, Repo())
}
```

>Final Step
><br>**Retrive repository from CreationExtras**

```java
override fun <T : ViewModel> create(modelClass: Class<T>, extras: CreationExtras): T {

    val repo = extras.get(REPOSITORY_KEY)

    repo?.let {
      return RepoBasedViewModel(repo)
    }

    return super.create(modelClass, extras)
}
```
Now that is done, let's create a customized ViewModelProvider.Factory.

#### Customizing ViewModelProvider.Factory - The Old Way
The following will show examples on how we can create our own ViewModelProvider.Factory if we have parameters that are neither empty nor a single `Application`.

>If we need to create **ViewModel**s that need both **localRepo** and **remoteRepo**
><br><br>Here is how I would have done it, in the old way.

```java
// RepoViewModelFactory.kt
class RepoViewModelFactory(private val localRepo: LocalRepo, private val remoteRepo: RemoteRepo): ViewModelProvider.Factory {
    override fun <T : ViewModel> create(modelClass: Class<T>): T {

        return with(modelClass) {
            when {
                isAssignableFrom(BookListViewModel::class.java) -> {
                    BookListViewModel(localRepo, remoteRepo)
                }

                isAssignableFrom(VideoListViewModel::class.java) -> {
                    VideoListViewModel(localRepo, remoteRepo)
                }

                else -> {
                    throw  RuntimeException("${modelClass::class.java} does not fit in ${this.simpleName}")
                }
            }
        } as T
    }
}
```

Some people like to extend **ViewModelProvider.NewInstanceFactory** instead, but there's no difference if we need to override the `create` method.

However, on the other hand, it would be more straight forward on what this class does if we uses **NewInstanceFactory** vs plain **Factory**.

>Next, what if RepoBasedViewModels **need more than** just **localRepo** and **remoteRepo**
?

```Java
class BookKeepingViewModel(localRepo: LocalRepo, remoteRepo: RemoteRepo, private val token: String): RepoBasedViewModel(localRepo, remoteRepo)
```
If we want to keep using **RepoViewModelFactory**, then we will need to override `create(Class<T>, CreationExtras)`

```java
companion object {
    @JvmStatic
    val STRING_CREATION_EXTRA = object :CreationExtras.Key<String>{}
}

override fun <T : ViewModel> create(modelClass: Class<T>, extras: CreationExtras): T {
        return with(modelClass) {
            when {
                isAssignableFrom(BookKeepingViewModel::class.java) -> {

                    val s = extras.get(STRING_CREATION_EXTRA)

                    if (s.isNullOrEmpty()) {
                        throw  RuntimeException("${modelClass::class.java} missing String in CreationExtras")
                    } else {
                        BookKeepingViewModel(localRepo, remoteRepo, s)
                    }
                }

                else -> {
                    throw  RuntimeException("${modelClass::class.java} does not fit in ${this.simpleName}")
                }
            }
        } as T
    }
```



Now everything about **ViewModel** creation is clear, let's take a look at how to create it in the **Right** way.

### Creating a ViewModel - The Right Way
Recall from the example before, you've seen how we can generate a **ViewModel** by calling :
```java
private val _viewModel: MainViewModel by lazy {
    ViewModelProvider(this).get(MainViewModel::class.java)
}
```

This code will do the followings :
1. Create a **ViewModelProvider**

2. Check if **ViewModelStore** has an instance of it based on the key given, which by default if the canonicalName of the modelClass

3. If a **ViewModel** was found and the factory has implemented **OnRequeryFactory**, such as **SavedStateViewModelFactory** for **ComponentActivity** and **Fragment**. <br>Then it will take care of adding and removing listeners during the appropriate lifecycle state.<br>And the **ViewModel** will be returned.

4. If nothing was found, then it will pass it to the factory, **AndroidViewModelFactory** by default, to create the **ViewModel** using **CreationExtras**.

> So how can we create **ViewModel** when we are using a customized **ViewModelProvider.Factory** ?

Well, we can use the constructor ViewModelProvider(Context, Factory) instead:

```Java
private val _repoViewModel: RepoBasedViewModel by lazy {
    ViewModelProvider(this, RepoViewModelFactory(LocalRepo(), RemoteRepo())).get(BookKeepingViewModel::class.java)
}
// or if we need to add CreationExtras
private val _repoViewModel: RepoBasedViewModel by lazy {
    ViewModelProvider(this, RepoViewModelFactory(LocalRepo(), RemoteRepo())).get(BookKeepingViewModel::class.java)
}
```

This would be a valid way of creating **ViewModel** IF the parameters are not repositories, since it is best to have a single source of repository.

> How do we fix this then ?

We could create a service locator, a singleton class that takes in an **Application** context.

Then prepare the service locator once the application starts, which will create the Local and Remote Repository.

With a service locator, we can change it to :
```Java
private val _bookKeepingViewModel: BookKeepingViewModel by lazy {
    ViewModelProvider(this, RepoViewModelFactory(ServiceLocator.localRepo, ServiceLocator.remoteRepo), _bookKeepingCE).get(BookKeepingViewModel::class.java)
}
```

But this would be another story if we are working with **Jetpack Compose**, there will be another time.

### Life as ViewModel
Finally, we are coming back to the major question
>** How does a ViewModel work ?**

This question can be divided into two sub-questions.
- How does ViewModel communicates with the View ?
- What is the Lifecycle of a ViewModel ?

#### How does ViewModel communicates with the View ?
This is were **LiveData** comes in to play.

Simply put it, a **LiveData** allow **LifecycleOwner** to observe the value stored in a **MutableLiveData**.

Whenever the value in a **LiveData** changes, and **LifecycleOwner** remains active and not destroyed, it will notifies it the change.  

If you feel like to dig in deeper, please check out this [post]().

#### What is the Lifecycle of a ViewModel ?

```java
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
