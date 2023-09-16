---
layout: post
title:  "Android - IPC / Binder"
date:   2022-08-08 16:15:07 +0800
categories: [android, binder, intermediate]
---

https://www.youtube.com/watch?v=14OffK9SR9c&list=PLIUuxxIJtMjWR77V4_QbKZY0gZb1kzoJ8

https://hackmd.io/TtT34BAwRP2sZ3vALt4Drg?view
https://hackmd.io/@AlienHackMd/S1R5Rkjbj

https://android.googlesource.com/kernel/msm/+/android-msm-anthias-3.10-lollipop-mr1-wear-release/drivers/staging/android/binder.c

https://paul.pub/android-binder-driver/

### Background
When developing application, there are times when we wish to interact with other applications.

In order to do so, we need to use **Binder** to achieve **Inter-Process Communication** or IPC.

>IPC ( **Inter-process communication** ) refers to the mechanisms an operating system provides to allow the processes to manage shared data [ [wiki][wiki_ipc] ].

In this article, we will take a look at what exactly is a **Binder** is and how does it achieve **IPC**.

### What is a Binder ?
For Android, a Binder is a **driver**.

>**A driver is a program that is responsible for interacting with hardware or services.**

But every driver will have a corresponding **device**.

### IPC Mechanisms


In Linux,




### Other IPC Mechanisms


### Binder Creation (AOSP)
In order to create a **Binder**, we need to call **ProcessState** :
```cpp
sp<ProcessState> ProcessState::self()
{
    return init(kDefaultDriver, false /*requireDefault*/);
}
```

<details>
<summary><b>init(const char *driver, bool requireDefault)</b></summary>

```cpp
sp<ProcessState> ProcessState::init(const char *driver, bool requireDefault)
{

    if (driver == nullptr) {
        std::lock_guard<std::mutex> l(gProcessMutex);
        if (gProcess) {
            verifyNotForked(gProcess->mForked);
        }
        return gProcess;
    }

    [[clang::no_destroy]] static std::once_flag gProcessOnce;
    // call_once(once_flag& __once, _Callable&& __f, _Args&&... __args)
    std::call_once(gProcessOnce, [&](){

        // int access(const char *pathname, int mode);
        // check user's permissions for a file
        // 若使用 R_OK 或 W_OK mode，當有此 file 就會回傳 0
        if (access(driver, R_OK) == -1) {
            ALOGE("Binder driver %s is unavailable. Using /dev/binder instead.", driver);
            driver = "/dev/binder";
        }

        // we must install these before instantiating the gProcess object,
        // otherwise this would race with creating it, and there could be the
        // possibility of an invalid gProcess object forked by another thread
        // before these are installed
        // pthread_atfork(3) - register fork handlers that are to
        //                      be executed when fork(2) is called by this thread
        // pthread_atfork(before, parent_after, child_after)
        // --------
        //  參數說明
        // --------
        // - prepare : specifies a handler that is executed before fork(2) processing starts.
        // - parent : specifies a handler that is executed in the parent process after fork(2) processing completes.
        // - child : specifies a handler that is executed in the child process after fork(2) processing completes.
        //
        int ret = pthread_atfork(ProcessState::onFork, ProcessState::parentPostFork,
                                 ProcessState::childPostFork);
        LOG_ALWAYS_FATAL_IF(ret != 0, "pthread_atfork error %s", strerror(ret));

        std::lock_guard<std::mutex> l(gProcessMutex);
        gProcess = sp<ProcessState>::make(driver);
    });

    if (requireDefault) {
        // Detect if we are trying to initialize with a different driver, and
        // consider that an error. ProcessState will only be initialized once above.
        LOG_ALWAYS_FATAL_IF(gProcess->getDriverName() != driver,
                            "ProcessState was already initialized with %s,"
                            " can't initialize with %s.",
                            gProcess->getDriverName().c_str(), driver);
    }

    verifyNotForked(gProcess->mForked);
    return gProcess;
}
```

</details>

>In the `init` function it will do the followings : <br>
> 1. If `driver` is null, check if `static sp<ProcessState> gProcess` hasn't been fork yet.
> 2.





[wiki_ipc]: https://en.wikipedia.org/wiki/Inter-process_communication (Inter-process communication - Wikipedia)
