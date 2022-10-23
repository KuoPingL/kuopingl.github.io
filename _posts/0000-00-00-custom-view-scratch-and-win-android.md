---
layout: post
title:  "Scratch And Win - in Android"
date:   2022-08-08 16:15:07 +0800
categories: [android, custom, view, demo]
---
## Background
This is a demo that I came up with when I was learning about Cavas and Paint.

Simply puts it, I am just curious how to make a scratch and win in Android app.

Without further ado, let's get into it.

## What is a Canvas ?
For beginners, a canvas is the region where graphics are being drawn.
However, it is much more than that.

A Canvas is a wrapper of SKCanvas, a class from the 2D graphic library [SKIA](https://skia.org/).

> SkCanvas provides an interface for drawing, and how the drawing is clipped and transformed. SkCanvas contains a stack of SkMatrix and clip values.[ref](https://github.com/google/skia/blob/main/include/core/SkCanvas.h)

As stated above, a SKCanvas is responsible for providing info on how images should be
displayed using SKMatrix(es), but more on that later.

Along with SKPaint,






## References
1. [Getting Started with Android Canvas Drawing](https://medium.com/over-engineering/getting-started-with-drawing-on-the-android-canvas-621cf512f4c7)
