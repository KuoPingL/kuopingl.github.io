---
layout: post
title:  "Android Architecture (零) : An Introduction"
date:   2022-08-08 16:15:07 +0800
categories: [android, architecture, advance]
---

This post is for anyone who are interested in the architecture of Android.

### Background
Some of you might have seen these 2 architecture diagrams :
> 1. Android Platform Architecture


<figure>
<center>
  <a href = "https://developer.android.com/guide/platform">
  <img src="/images/common/android-platform-architecture.png" alt="Android Platform Architecture" style = "width:70%">
  <figcaption>Fig.1 - Android Platform Architecture</figcaption>
  </a>
  </center>
</figure>

>and
> 2. Android Architecture

<figure>
<center>
  <a href = "https://source.android.com/docs/core/architecture">
  <img src="/images/common/android-architecture.png" alt="Android Platform Architecture" style="width:70%">
  <figcaption>Fig.2 - Android System Architecture</figcaption>
  </a>
  </center>
</figure>

<br>
These two diagrams are basically describing the same thing, except for the application layer in platform architecture.


> The **first diagram** is what you will see if you are working on **Android Applications**.
<br><br>
The **second diagram** is when you are working on **AOSP (Android Open Source Project)**.

In this article, I want to focus on the second diagram.
