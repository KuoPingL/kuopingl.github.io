---
layout: post
title:  "Android - About Threads "
date:   2022-08-08 16:15:07 +0800
categories: [android jetpack, thread, beginner]
---
### Thread


### Handler & Looper

### Executor

#### ExecutorService
##### AbstractExecutorService
###### ThreadPoolExecutor


### TaskExecutor
> This class is still fairly new, so Google warned us :
> <br>Not to use this from outside, we don't know what the API will look like yet.

>A task executor that can divide tasks into logical groups.
It holds a collection a executors for each group of task.

```java
@RestrictTo(RestrictTo.Scope.LIBRARY_GROUP_PREFIX)
public abstract class TaskExecutor {
    public abstract void executeOnDiskIO(@NonNull Runnable runnable);
    public abstract void postToMainThread(@NonNull Runnable runnable);
    public void executeOnMainThread(@NonNull Runnable runnable) {
        if (isMainThread()) {
            runnable.run();
        } else {
            postToMainThread(runnable);
        }
    }
    public abstract boolean isMainThread();
}
```

#### ArchTaskExecutor
>**ArchTaskExecutor** is a static class that serves as a central point to execute common task.

It can provides us with **Executor** on `MainThread` and `IOThread`.

Also, it also serves as a proxy by delegating all the **TaskExecutor** implementation to a delegate, which is **DefaultTaskExecutor** by default.

```java
@RestrictTo(RestrictTo.Scope.LIBRARY_GROUP_PREFIX)
public class ArchTaskExecutor extends TaskExecutor {
    private static volatile ArchTaskExecutor sInstance;

    @NonNull
    private TaskExecutor mDelegate;

    @NonNull
    private TaskExecutor mDefaultTaskExecutor;

    @NonNull
    private static final Executor sMainThreadExecutor = new Executor() {
        @Override
        public void execute(Runnable command) {
            getInstance().postToMainThread(command);
        }
    };

    @NonNull
    private static final Executor sIOThreadExecutor = new Executor() {
        @Override
        public void execute(Runnable command) {
            getInstance().executeOnDiskIO(command);
        }
    };

    private ArchTaskExecutor() {
        mDefaultTaskExecutor = new DefaultTaskExecutor();
        mDelegate = mDefaultTaskExecutor;
    }

    @NonNull
    public static ArchTaskExecutor getInstance() {
        if (sInstance != null) {
            return sInstance;
        }
        synchronized (ArchTaskExecutor.class) {
            if (sInstance == null) {
                sInstance = new ArchTaskExecutor();
            }
        }
        return sInstance;
    }

    public void setDelegate(@Nullable TaskExecutor taskExecutor) {
        mDelegate = taskExecutor == null ? mDefaultTaskExecutor : taskExecutor;
    }

    @Override
    public void executeOnDiskIO(Runnable runnable) {
        mDelegate.executeOnDiskIO(runnable);
    }

    @Override
    public void postToMainThread(Runnable runnable) {
        mDelegate.postToMainThread(runnable);
    }

    @NonNull
    public static Executor getMainThreadExecutor() {
        return sMainThreadExecutor;
    }

    @NonNull
    public static Executor getIOThreadExecutor() {
        return sIOThreadExecutor;
    }

    @Override
    public boolean isMainThread() {
        return mDelegate.isMainThread();
    }
}
```

#### DefaultTaskExecutor
