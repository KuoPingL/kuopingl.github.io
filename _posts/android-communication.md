---
layout: post
title:  "Android - Thread and Process Communication"
date:   2022-08-08 16:15:07 +0800
categories: [android, ipc, thread, beginner]
---

### Background
In any OS, there are always time where you need to perform communication within an app or across two apps. Have you ever wonder how Android does that ?

>Communications in OS can be categoried into 2 categories :
1. Communication can either be performed **Between Threads**
2. Or it can be performed **Between Processes (application)**

This article will give you are full details on mechanisms used in Android to perform communications.
<br>So sit tight and enjoy.

### About Computer System
Before we jump into the world of **Thread** and **Process**, we need to understand the structure of a computer system.

A computer system can be divided into 3 layers [[1][1]]:
- Application
- Operation System
- Computer Hardware

Users can interact with the application through various inputs (hardware), which will pass the signal to the OS and trigger actions in the application.

The application will then pass a signal back to the OS, which will display the output via the screen (hardware).

Next, let's take a closer look at what is an OS.

#### What constructs an OS ?
>An OS is usually defined as the **one program that runs at all time in computer** aka **kernel** [[1][1]]

However, in mobile OS, such as iOS and Android, kernel is not the only thing that make up the OS, but also **middleware**.
>A middleware is **a set of software frameworks that provide additional services to application developers**
><br><br>
>an example would be the **Application Framework** (Java API Framework) in Android

<figure>
<center>
<a href = "https://developer.android.com/guide/platform">
<img src = "/images/common/android-platform-architecture.png" style="width:80%" title="Android Platform architecture"/>
</a>
</center>
<br>
<figcaption><b>Figure 1 :</b> Android Platform Architecture</figcaption>
</figure>











[1]:https://www.os-book.com/OS10/ "Operating System Concepts, 10th ed."



<br><br><br><br>
