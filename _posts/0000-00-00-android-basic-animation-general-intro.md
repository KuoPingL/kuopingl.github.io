---
layout: post
title:  "Android - Animation Basic : General Introduction"
date:   2022-08-08 16:15:07 +0800
categories: [animation, android, basic]
---

## Background
If you were like me, banging your head against a brick of wall while going through the official documents trying to learn about animations, then you are at the right place.

Like any official documents, they will throw at you many terms that you kind of
understand, but you really don't.

Don't worry, I will do my best to help you understand them.

## About Animations
In Android, we can create animations using :
1. **[Drawable](https://developer.android.com/develop/ui/views/animations/drawable-animation)**
   - [Official Doc](https://developer.android.com/develop/ui/views/animations/drawable-animation)
   - we can either show bitmap frame by frame, aka **Frame Animation**
   - or after **Android 5.0** we can use SVG  

2. **[Animation](https://developer.android.com/reference/android/view/animation/package-summary)**
   - [Official Doc](https://developer.android.com/reference/android/view/animation/package-summary)
   - this is used to performs series of simple transformations, allowing us to create **Tween Animation**
   - however, this only affects the <u>*appearance of the view*</u>.
   - This is how animation is displayed prior **Android 3.0**.

3. **[Animator](https://developer.android.com/reference/android/animation/package-summary)**
   - [Official Doc]((https://developer.android.com/reference/android/animation/package-summary))
   - after Android 3.0, we can use this class to change View Property, which can change the location of the View component instead of just appearance.


4. **Frameworks**
   - we can also use **Transition** Framework to display transition between activities or layouts.

We are going to cover all of these one by one, you can find more info by clicking on the topic you are interested in.

Also, I will cover what happen behind the scene. By understanding what happen, it will make your debugging life easier.
