---
layout: post
title:  "Android Tutorial - How to Release a Library?"
date:   2022-08-08 16:15:07 +0800
categories: [android, tutorial, library, intermediate]
---

## Background
In this article, we will take some times to go through the process to release your library in the two most common repositories : **Maven** and **JitPack**.

At the end, we will also take a look at the library releasing process given by the Android [official](https://developer.android.com/studio/publish-library).

## JitPack

><br>If you have a **GitHub** repository, this is a **easiest** way to release a library.
><br/>

In order to release your library via JitPack, you need to provide access to your GitHub repository to [JitPack.io](https://jitpack.io/).

This can be done by following steps below :

1. Select "Tag" on your library GitHub page

<figure>
<center>
<img src = "/images/posts/jekyll/release_library/select_tag.png" style="width=80%"/>
</center>
</figure>

2. Add a tag for your release :

<figure>
<center>
<img src = "/images/posts/jekyll/release_library/add_tag.png" style = "width=80%"/>
</center>
</figure>

This tag can later be used when users use your library in their project :
```groovy
dependencies {
        implementation 'com.github.KuoPingL:TwoWaySlider:Tag'
}
```

And that's it.

When users want to use it, they need to add **maven { url "http://jitpack.io" }** at **settings.gradle** :

```groovy
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        maven { url "https://jitpack.io" }
    }
}
```

And add dependency at app/build.gradle :

```groovy
implementation 'com.github.KuoPingL:TwoWaySlider:v1.0.0'
```

## Maven Central

><br>**Maven Central** on the other hand is more complicated, but it is the **standard** way and the standard repository used in Android.
><br>In addition, you get to understand what is **JitPack** did for you behind the scene.
><br/>

According to the best practice [guide](https://maven.apache.org/repository-management.html), to publish our library we will need to pick a **repository manager**, which acts as dedicated proxy server for public Maven repositories.

We can find a list of the available repository manager on [this](https://maven.apache.org/repository-management.html#available-repository-managers) page.

In our case, [Sonatype](https://central.sonatype.org/publish/publish-guide/) is selected.

After selecting the repository manager, we will continue with the process :

1. To start using **Sonatype**, we need to create a ticket with Sonatype.
   But since Sonatype uses JIRA to manage requests, we need to **[create a JIRA account](https://issues.sonatype.org/secure/Signup!default.jspa)**.
2. After registering an account, we can [create a new project ticket](https://issues.sonatype.org/secure/CreateIssue.jspa?pid=10134&issuetype=21).
   <br><img src = "/images/posts/jekyll/release_library/sonatype_create_new_project_ticket.png"/>
3. Now ... wait for 2 business days 




**References**
1. [ Oh No - Another Publishing Android Artifacts to Maven Central Guide?](https://medium.com/nerd-for-tech/oh-no-another-publishing-android-artifacts-to-maven-central-guide-9d7f300ebd74)

## Android Official



## How to choose repository ?








<br><br><br><br><br><br><br><br><br><br>
