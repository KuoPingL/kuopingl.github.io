---
layout: post
title:  "Android Tutorial - How to Create a Library?"
date:   2022-08-08 16:15:07 +0800
categories: [android, tutorial, library, intermediate]
---

### Background
Before we started, let's straighten up some of the concepts.

#### What is a Library ?

A library is a collection of **reusable** code that is meant to solve a common problem.

An example would be :

><br> <u>**[RxJava](https://github.com/ReactiveX/RxJava)**</u>
> <br>a Java VM implementation of Reactive Extensions: a library for composing asynchronous and event-based programs by using observable sequences.
><br><br/>

Another term that is similar to library is called **Framework**, which is also a set of reusable code written to solve common problems.

**Now tell me**
What are the differences between a **library** and a **framework**?

<details>
<summary><b>Answer</b></summary>

Personally, I've always consider a **Framework** as a *collection of libraries*, but *I was wrong*.

A **library** is like an accessory for your car, you can always

A **framework**, on the other hand, is like blueprints and materials that you might need to build a car.

Another way to put it is that a **framework** provides you with a specific resources that you can use to achieve a specific goal.

For instance, you cannot build a house using a **framework** specialized for car. Instead, you can get tools to convert a car into a house by importing a **library** for house building.

The technical difference lies in the term **inversion of control**.

<u><b>[What is IoC ?](https://en.wikipedia.org/wiki/Inversion_of_control)</b></u>

> "This inversion of control is sometimes
named the Hollywood principle, “Do not call us, we call You”"
><br><br>
> This term was introduced by Michael Mattsson in 1996 in his [thesis](https://www.researchgate.net/publication/2238535_Object-Oriented_Frameworks) on Object Oriented Framework.

An example of a framework would be :
> **Java API Framework** in Android Platform Architecture (aka **Application Framework**)
> <br><br>

<figure>
<center>
<a href = "https://developer.android.com/guide/platform">
<img src = "/images/common/android-platform-architecture.png" style = "width=50%"/>
<figcaption>Android Platform Architecture</figcaption>
</a>
</center>
</figure>

<br>
This is the framework Android application is build upon.

It provides you with the materials you'll need, such as Activity, Services, Content Provider and much more.

Not only that, it also controls how the app begins, how components interact with each other and the lifecycle of them.

These are some of the things that you are not in control of, thus Hollywood principle.


> <b>References :</b>
> - <b><u>[Framework vs Library: Full Comparison](https://www.interviewbit.com/blog/framework-vs-library/)</b></u>
> - <b><u>[The Difference Between a Framework and a Library](https://www.freecodecamp.org/news/the-difference-between-a-framework-and-a-library-bd133054023f/)</b></u>
> -<b><u>[Android Plateform Architecture](https://developer.android.com/guide/platform)</u></b>

</details>

#### What are the benefit in Modulization (creating libraries) ?
> The first thing comes in mind is **Cleaner** code and architecture.

By creating modules we can achieve
- the **Separation of Concerns**
- making the app more **Testible**, **Reusable**, **Scalable**
- easier to **Manage** or **Organized**
- we can even **Customize Delivery**
- and last but not least, creating modules makes our application module thinner, thus **reduces the overall build time**.

Here are some articles on how modulization helps application development :
- [A Guide to a Modern Android App](https://medium.com/quandoo/a-guide-to-a-modern-android-app-part-1-modularization-and-architecture-9f2aa91b3090) by [Marcello Galhardo](https://medium.com/@marcellogalhardo)
- [Modularization of Android Applications based on Clean Architecture](https://ahmad-efati.medium.com/modularization-of-android-applications-based-on-clean-architecture-18dc643e0562) by  [Ahmad Efati](https://ahmad-efati.medium.com/)
- <u>[Google](https://developer.android.com/topic/modularization)</u>
- <u>[How modularization can speed up your Android app’s built time](https://www.freecodecamp.org/news/how-modularisation-affects-build-time-of-an-android-application-43a984ce9968/)</u>

Even though modulization can help us

#### When should we perform Modulization ?
Modulization is not suitable for every cases.




### Creating a Module
> Just to be cleared.
  <br>A **module** is a single purposed **library**.
> <br>Whereas a **library** is a <u>collection of related functionality</u>.
> <br>[[source](https://stackoverflow.com/a/4101270/18597115)]


To create a module in AS, we just need to go to `File > New > New Module` : <br>
<figure>
<center>
<img src ="/images/posts/jekyll/2022-09-23-android-create-library-new-module.png" style="width:80%">

</center>
</figure>
<br>

Now you should see a dialog that shows a list of **Templates** that we can use on the left.
<br>

<figure>
<center>
<img src = "/images/posts/jekyll/2022-09-23-android-create-library-new-module-dialog.png" style="width:80%">
</center>
</figure>

<br>

The ones that are most likely to use are :
- **Phone & Tablet**
- **Android Library**
- **Android Native Library**
- **Java or Kotlin Library**
- **Kotlin Multiplatform Shared Module** aka **KMM**

#### Looking at the Template
Let's take a look at the purpose of these templates.

> **Phone & Tablet**

This is just the typical Application Template when we create an application.

> **Android Library**

This is a library similar to Application Template, but with some restrictions.

It is **mandatory** to include the following files :
- `/AndroidManifest.xml` (**mandatory**)
- `/classes.jar` (**mandatory**)
- `/res/` (**mandatory**)
- `/R.txt` (**mandatory**)

while the rest are left as **optional** :
- `/assets/` (optional)
- `/libs/*.jar` (optional)
- `/jni//*.so` (optional)
- `/proguard.txt` (optional)
- `/lint.jar` (optional)

> **Android Native Library**

This is used to add **C/C++** library in the app and uses **JNI** (**J**ava **N**ative **I**nteface) to interact with it.

> **Java or Kotlin Library**

This is similar with the Native Library, but instead of using C/C++, this library only contains **Java** or **Kotlin**.

#### Choosing a Template
So depends on your needs, you can choose any one you wish.

If you only wish the library to contain plain codes without any resources, then **Java or Kotlin Library** and **Android Native Library** might be the choice for you.

If you were like me, just trying to share a awesome custom view, then the one we are interested in is **Android Library**.

### Creating an Android Library
After creating the library, you should be able to see another package shown in the Project directory.

<figure>
<center>
<img src = "/images/posts/jekyll/2022-09-23-android-create-library-android-library.png" style="width:50%"/>
</center>
</figure>

<br>
The main difference between the library and the application module lies inside the Gradle.

Looking at the `build.gradle`, you will find the followings :
>**Android Library**
```java
plugins {
    id 'com.android.library'
}
```

>**Application Module**
```java
plugins {
    id 'com.android.application'
    id 'org.jetbrains.kotlin.android'
}
```

You can see Android Library uses `com.android.library` plugin instead of `com.android.application`.

Other than that, there's not much of difference.

After creating your library and writing your code, you can choose to import your library in serveral methods.

#### Importing a Library
You can import a library in 3 ways :
- Local Libary Module
- Local Binary
- Remote Binary

>**Local Library Module**

This is what we are currently facing.

In order to use the local library, we can simply add `implementation` in the `dependencies` :
```Java
dependencies {
    // Dependency on a local library module
    implementation project(':mylibrary')
}
```

>**Local Binaries**

This is when we wish to import a local `jar` file in the project.

You can go to `File > Project Structure > Dependencies` and follow the [instructions](https://developer.android.com/studio/projects/android-library#psd-add-aar-jar-dependency).

By the end, you should have this in the `dependencies` block :

```java
implementation files('my_src_folder/my_lib.aar')
// or if you are importing jar
implementation files('my_src_folder/my_lib.jar')
```

If you are not using Android Studio, then you can use the following instead :

```java
dependencies {
    implementation fileTree(dir: "my_src_folder", include: ["*.jar", "*.aar"])
}
```

> **Remote Binary**

This is when you need to use a library that's shared remotely.

Basically all you need to do is add :
```Java
implementation 'com.example.android:app-magic:12.3'

// which is the same as
implementation group:'com.example.android', name:'app-magic', version:'12.3'
```
Now that we know how to import libraries from various source, let's m

### Share Remotely

The two most popular repositories used are :

1. **[JitPack](https://docs.jitpack.io/intro/)**

2. **[Maven Central](https://maven.apache.org/repository/index.html)**
   an open source build system, developed by the Apache foundation and mostly used for java projects [[medium](https://proandroiddev.com/publishing-a-maven-artifact-1-3-glossary-bc0068a440e0)].

We will walkthrough on how to publish our library in both repositories.

#### Maven Central

><br>This is the "standard" way to publish, but it takes more steps.
><br/>

Before we publish our library, we need to install **Maven** by :
1. Download apache-maven from [here](https://maven.apache.org/download.cgi)
2. Install Maven by the steps over [here](https://maven.apache.org/install.html) and check if maven has been installed successfully.
   **Note:** in the step
   ```shell
   export PATH=/opt/apache-maven-3.8.6/bin:$PATH
   ```
   you need to use the proper path here you extracted `apache-maven-3.8.6` folder.

At the end, when you enter `mvn -v`, you should be able to see :

```shell
Apache Maven 3.8.6 (84538c9988a25aec085021c365c560670ad80f63)
Maven home: /Users/jimmy/Documents/apache-maven-3.8.6
Java version: 15.0.2,
vendor: AdoptOpenJDK,
runtime: /Library/Java/JavaVirtualMachines/adoptopenjdk-15-openj9.jdk/Contents/Home
```

If you get `command not found`, try closing your terminal and re-open it, allowing it read the updated PATH.


### Extra
Depending on the module you choose, you will be able to create libraries in different file types.

>If you choose **Java or Kotlin Library**

You can create a **Java Archive Resources** ( `jar` ) file.

This can be done using the following command :
```shell
jar cf jar-file input-file(s)
```

> If you choose **Android library**

You can create an **Android Archive Resource** ( `aar` )  file.

`.aar` can be created in [several ways](https://www.geeksforgeeks.org/different-ways-to-create-aar-file-in-android-studio/).

> If you choose **C/C++**

This will have to wait until I tried it out.

### References
1. [CodePath: Building your own Android library](https://github.com/codepath/android_guides/wiki/Building-your-own-Android-library)

2. [How to create a JAR file](https://www.tutorialspoint.com/How-to-create-a-JAR-file)
