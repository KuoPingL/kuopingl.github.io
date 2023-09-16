---
layout: post
title:  "Android - Keyboard"
date:   2022-08-08 16:15:07 +0800
categories: [keyboard, android, intermediate]
---

### Background
Keyboard is one of the thing that taunt developers from time to time.

Some of the most common questions regarding keyboard are :
- How do we detect if keyboard is visible ?
- How to open and close keyboard programmatically ?
- How to close keyboard once `enter` is pressed when working with **EditText**?

In this article, we will be discussing some of the solutions available and learn to implement them in our code.

### TextView
**EditText** is the main component in the application that will trigger the keyboard.

By examining the process of showing the keyboard, we can have a better understanding on how we can detect the visibility of the keyboard.

Since **EditText** is based on **TextView**, we will start by looking into **TextView** to find our answer.

In **TextView**, we can find 3 functions that will show the keyboard once triggered :

<details>
<summary><b>onKeyUp</b></summary>

```Java
case KeyEvent.KEYCODE_DPAD_CENTER:
    if (event.hasNoModifiers()) {
        /*
         * If there is a click listener, just call through to
         * super, which will invoke it.
         *
         * If there isn't a click listener, try to show the soft
         * input method.  (It will also
         * call performClick(), but that won't do anything in
         * this case.)
         */
        if (!hasOnClickListeners()) {
            if (mMovement != null && mText instanceof Editable
                    && mLayout != null && onCheckIsTextEditor()) {
                InputMethodManager imm = getInputMethodManager();
                viewClicked(imm);
                if (imm != null && getShowSoftInputOnFocus()) {
                    imm.showSoftInput(this, 0);
                }
            }
        }
    }
    return super.onKeyUp(keyCode, event);
```

</details>
&nbsp;&nbsp;&nbsp; This function will make sure
<details>
<summary><b>onTouchEvent</b></summary>

```Java
if (DEBUG_CURSOR) {
    logCursor("onTouchEvent", "%d: %s (%f,%f)",
            event.getSequenceNumber(),
            MotionEvent.actionToString(event.getActionMasked()),
            event.getX(), event.getY());
}
final int action = event.getActionMasked();
if (mEditor != null) {
    if (!isFromPrimePointer(event, false)) {
        return true;
    }

    mEditor.onTouchEvent(event);

    if (mEditor.mInsertionPointCursorController != null
            && mEditor.mInsertionPointCursorController.isCursorBeingModified()) {
        return true;
    }
    if (mEditor.mSelectionModifierCursorController != null
            && mEditor.mSelectionModifierCursorController.isDragAcceleratorActive()) {
        return true;
    }
}

final boolean superResult = super.onTouchEvent(event);
if (DEBUG_CURSOR) {
    logCursor("onTouchEvent", "superResult=%s", superResult);
}

/*
 * Don't handle the release after a long press, because it will move the selection away from
 * whatever the menu action was trying to affect. If the long press should have triggered an
 * insertion action mode, we can now actually show it.
 */
if (mEditor != null && mEditor.mDiscardNextActionUp && action == MotionEvent.ACTION_UP) {
    mEditor.mDiscardNextActionUp = false;
    if (DEBUG_CURSOR) {
        logCursor("onTouchEvent", "release after long press detected");
    }
    if (mEditor.mIsInsertionActionModeStartPending) {
        mEditor.startInsertionActionMode();
        mEditor.mIsInsertionActionModeStartPending = false;
    }
    return superResult;
}

final boolean touchIsFinished = (action == MotionEvent.ACTION_UP)
        && (mEditor == null || !mEditor.mIgnoreActionUpEvent) && isFocused();

if ((mMovement != null || onCheckIsTextEditor()) && isEnabled()
        && mText instanceof Spannable && mLayout != null) {
    boolean handled = false;

    if (mMovement != null) {
        handled |= mMovement.onTouchEvent(this, mSpannable, event);
    }

    final boolean textIsSelectable = isTextSelectable();
    if (touchIsFinished && mLinksClickable && mAutoLinkMask != 0 && textIsSelectable) {
        // The LinkMovementMethod which should handle taps on links has not been installed
        // on non editable text that support text selection.
        // We reproduce its behavior here to open links for these.
        ClickableSpan[] links = mSpannable.getSpans(getSelectionStart(),
            getSelectionEnd(), ClickableSpan.class);

        if (links.length > 0) {
            links[0].onClick(this);
            handled = true;
        }
    }

    if (touchIsFinished && (isTextEditable() || textIsSelectable)) {
        // Show the IME, except when selecting in read-only text.
        final InputMethodManager imm = getInputMethodManager();
        viewClicked(imm);
        if (isTextEditable() && mEditor.mShowSoftInputOnFocus && imm != null) {
            imm.showSoftInput(this, 0);
        }

        // The above condition ensures that the mEditor is not null
        mEditor.onTouchUpEvent(event);

        handled = true;
    }

    if (handled) {
        return true;
    }
}
```

</details>
<details>
<summary><b>performAccessibilityActionClick</b></summary>

```Java
// Show the IME, except when selecting in read-only text.
if ((mMovement != null || onCheckIsTextEditor()) && hasSpannableText() && mLayout != null
        && (isTextEditable() || isTextSelectable()) && isFocused()) {
    final InputMethodManager imm = getInputMethodManager();
    viewClicked(imm);
    if (!isTextSelectable() && mEditor.mShowSoftInputOnFocus && imm != null) {
        handled |= imm.showSoftInput(this, 0);
    }
}
```

</details>

&nbsp;&nbsp;&nbsp;&nbsp;This function will make sure **TextView** is
<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; - `focused` and
<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; - the text is either `editable` or `selectable`.

<br>By looking at the code, you will find that whenever a keyboard needs to be shown, the following code will appear :
```Java
final InputMethodManager imm = getContext().getSystemService(InputMethodManager.class);
imm.showSoftInput(this, 0);
```

Similarly, the following 3 functions will hide the keyboard :
<details>
<summary><b>onKeyUp</b></summary>

```Java
case KeyEvent.KEYCODE_ENTER:
case KeyEvent.KEYCODE_NUMPAD_ENTER:
if (event.hasNoModifiers()) {
    if (mEditor != null && mEditor.mInputContentType != null
            && mEditor.mInputContentType.onEditorActionListener != null
            && mEditor.mInputContentType.enterDown) {
        mEditor.mInputContentType.enterDown = false;
        if (mEditor.mInputContentType.onEditorActionListener.onEditorAction(
                this, EditorInfo.IME_NULL, event)) {
            return true;
        }
    }

    if ((event.getFlags() & KeyEvent.FLAG_EDITOR_ACTION) != 0
            || shouldAdvanceFocusOnEnter()) {
        /*
         * If there is a click listener, just call through to
         * super, which will invoke it.
         *
         * If there isn't a click listener, try to advance focus,
         * but still call through to super, which will reset the
         * pressed state and longpress state.  (It will also
         * call performClick(), but that won't do anything in
         * this case.)
         */
        if (!hasOnClickListeners()) {
            View v = focusSearch(FOCUS_DOWN);

            if (v != null) {
                if (!v.requestFocus(FOCUS_DOWN)) {
                    throw new IllegalStateException("focus search returned a view "
                            + "that wasn't able to take focus!");
                }

                /*
                 * Return true because we handled the key; super
                 * will return false because there was no click
                 * listener.
                 */
                super.onKeyUp(keyCode, event);
                return true;
            } else if ((event.getFlags()
                    & KeyEvent.FLAG_EDITOR_ACTION) != 0) {
                // No target for next focus, but make sure the IME
                // if this came from it.
                InputMethodManager imm = getInputMethodManager();
                if (imm != null && imm.isActive(this)) {
                    imm.hideSoftInputFromWindow(getWindowToken(), 0);
                }
            }
        }
    }
    return super.onKeyUp(keyCode, event);
}
break;
```

</details>
<details>
<summary><b>setEnabled</b></summary>

```Java
if (!enabled) {
    // Hide the soft input if the currently active TextView is disabled
    InputMethodManager imm = getInputMethodManager();
    if (imm != null && imm.isActive(this)) {
        imm.hideSoftInputFromWindow(getWindowToken(), 0);
    }
}
```

</details>
<details>
<summary><b>onEditorAction</b></summary>

```Java
if (actionCode == EditorInfo.IME_ACTION_NEXT) {
    View v = focusSearch(FOCUS_FORWARD);
    if (v != null) {
        if (!v.requestFocus(FOCUS_FORWARD)) {
            throw new IllegalStateException("focus search returned a view "
                    + "that wasn't able to take focus!");
        }
    }
    return;

} else if (actionCode == EditorInfo.IME_ACTION_PREVIOUS) {
    View v = focusSearch(FOCUS_BACKWARD);
    if (v != null) {
        if (!v.requestFocus(FOCUS_BACKWARD)) {
            throw new IllegalStateException("focus search returned a view "
                    + "that wasn't able to take focus!");
        }
    }
    return;

} else if (actionCode == EditorInfo.IME_ACTION_DONE) {
    InputMethodManager imm = getInputMethodManager();
    if (imm != null && imm.isActive(this)) {
        imm.hideSoftInputFromWindow(getWindowToken(), 0);
    }
    return;
}
```

</details>

<br>In these functions, we can see that the following code is used when hiding the keyboard :
```Java
final InputMethodManager imm = getContext().getSystemService(InputMethodManager.class);
if (imm != null && imm.isActive(this)) {
    imm.hideSoftInputFromWindow(getWindowToken(), 0);
}
```

Now that we see how soft keyboard can be shown and hide through code, let's take a look at what really happen.

### InputMethodManager
#### Getting InputMethodManager
In this section, let's try to understand the following code :
```Java
getContext().getSystemService(InputMethodManager.class);
```

When we call `getSystemService`, it will try to find the service name then find the proper service by the name :

```Java
public final @Nullable <T> T getSystemService(@NonNull Class<T> serviceClass) {
    // Because subclasses may override getSystemService(String) we cannot
    // perform a lookup by class alone.  We must first map the class to its
    // service name then invoke the string-based method.
    String serviceName = getSystemServiceName(serviceClass);
    return serviceName != null ? (T)getSystemService(serviceName) : null;
}
```
Though `getSystemServiceName` and `getSystemService` is called within **Context**, it is actually implemented in **BridgeContext** :

<details>
<summary><b>getSystemServiceName</b></summary>

```Java
@Override
public String getSystemServiceName(Class<?> serviceClass) {
    return SystemServiceRegistry.getSystemServiceName(serviceClass);
}
```

<details>
<summary><b>-SystemServiceRegistry</b></summary>

>**SystemServiceRegistry** will look into `SYSTEM_SERVICE_NAMES`, a `Map<Class<?>, String>`, to find the name.

```java
public static String getSystemServiceName(Class<?> serviceClass) {
    if (serviceClass == null) {
        return null;
    }
    final String serviceName = SYSTEM_SERVICE_NAMES.get(serviceClass);
    if (sEnableServiceNotFoundWtf && serviceName == null) {
        // This should be a caller bug.
        Slog.wtf(TAG, "Unknown manager requested: " + serviceClass.getCanonicalName());
    }
    return serviceName;
}
```

</details>

<details>
<summary><b>--Preparing SYSTEM_SERVICE_NAMES</b></summary>

  >Registration is done in the `static` block.

```Java
registerService(Context.INPUT_SERVICE, InputManager.class,
      new StaticServiceFetcher<InputManager>() {
  @Override
  public InputManager createService() {
      return InputManager.getInstance();
}});
```

</details>

</details>

<details>
<summary><b>getSystemService</b></summary>

>After getting the service name, it will use `getSystemService` to get **InputManager**

```Java
@Override
public Object getSystemService(String service) {
    switch (service) {
        // ...
        case INPUT_METHOD_SERVICE:  // needed by SearchView and Compose
            return InputMethodManager.forContext(this);
        // ...
    }
```

</details>

<br>After getting **InputMethodManager**, next, we will examine `hideSoftInputFromWindow` and `showSoftInput`.

#### showSoftInput & hideSoftInputFromWindow
<details>
<summary><b>showSoftInput</b></summary>

```Java
public boolean showSoftInput(View view, int flags) {
    // Re-dispatch if there is a context mismatch.
    final InputMethodManager fallbackImm = getFallbackInputMethodManagerIfNecessary(view);
    if (fallbackImm != null) {
        return fallbackImm.showSoftInput(view, flags);
    }

    return showSoftInput(view, flags, null);
}

public boolean showSoftInput(View view, int flags, ResultReceiver resultReceiver) {
    return showSoftInput(view, flags, resultReceiver, SoftInputShowHideReason.SHOW_SOFT_INPUT);
}

private boolean showSoftInput(View view, int flags, ResultReceiver resultReceiver,
        @SoftInputShowHideReason int reason) {
    ImeTracing.getInstance().triggerClientDump("InputMethodManager#showSoftInput", this,
            null /* icProto */);
    // Re-dispatch if there is a context mismatch.
    final InputMethodManager fallbackImm = getFallbackInputMethodManagerIfNecessary(view);
    if (fallbackImm != null) {
        return fallbackImm.showSoftInput(view, flags, resultReceiver);
    }

    checkFocus();
    synchronized (mH) {
        if (!hasServedByInputMethodLocked(view)) {
            Log.w(TAG, "Ignoring showSoftInput() as view=" + view + " is not served.");
            return false;
        }

        try {
            Log.d(TAG, "showSoftInput() view=" + view + " flags=" + flags + " reason="
                    + InputMethodDebug.softInputDisplayReasonToString(reason));
            return mService.showSoftInput(
                    mClient,
                    view.getWindowToken(),
                    flags,
                    resultReceiver,
                    reason);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
}
```

</details>

<details>
<summary><b>hideSoftInputFromWindow</b></summary>

```Java
public boolean hideSoftInputFromWindow(IBinder windowToken, int flags) {
    return hideSoftInputFromWindow(windowToken, flags, null);
}

public boolean hideSoftInputFromWindow(IBinder windowToken, int flags,
        ResultReceiver resultReceiver) {
    return hideSoftInputFromWindow(windowToken, flags, resultReceiver,
            SoftInputShowHideReason.HIDE_SOFT_INPUT);
}

private boolean hideSoftInputFromWindow(IBinder windowToken, int flags,
        ResultReceiver resultReceiver, @SoftInputShowHideReason int reason) {
    ImeTracing.getInstance().triggerClientDump("InputMethodManager#hideSoftInputFromWindow",
            this, null /* icProto */);
    checkFocus();
    synchronized (mH) {
        final View servedView = getServedViewLocked();
        if (servedView == null || servedView.getWindowToken() != windowToken) {
            return false;
        }

        try {
            return mService.hideSoftInput(mClient, windowToken, flags, resultReceiver, reason);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
}
```

</details>

>By calling `showSoftInput` or `hideSoftInputFromWindow`, it will trigger the same function in **IInputMethodManager**.aidl

<details>
<summary><b>IInputMethodManager.aidl</b></summary>

```c++
interface IInputMethodManager {
    void addClient(in IInputMethodClient client, in IInputContext inputContext,
            int untrustedDisplayId);
    // TODO: Use ParceledListSlice instead
    List<InputMethodInfo> getInputMethodList(int userId);
    List<InputMethodInfo> getAwareLockedInputMethodList(int userId, int directBootAwareness);
    // TODO: Use ParceledListSlice instead
    List<InputMethodInfo> getEnabledInputMethodList(int userId);
    List<InputMethodSubtype> getEnabledInputMethodSubtypeList(in String imiId,
            boolean allowsImplicitlySelectedSubtypes);
    InputMethodSubtype getLastInputMethodSubtype();
    boolean showSoftInput(in IInputMethodClient client, IBinder windowToken, int flags,
            in ResultReceiver resultReceiver, int reason);
    boolean hideSoftInput(in IInputMethodClient client, IBinder windowToken, int flags,
            in ResultReceiver resultReceiver, int reason);
    // If windowToken is null, this just does startInput().  Otherwise this reports that a window
    // has gained focus, and if 'attribute' is non-null then also does startInput.
    // @NonNull
    InputBindResult startInputOrWindowGainedFocus(
            /* @StartInputReason */ int startInputReason,
            in IInputMethodClient client, in IBinder windowToken,
            /* @StartInputFlags */ int startInputFlags,
            /* @android.view.WindowManager.LayoutParams.SoftInputModeFlags */ int softInputMode,
            int windowFlags, in EditorInfo attribute, in IInputContext inputContext,
            in IRemoteAccessibilityInputConnection remoteAccessibilityInputConnection,
            int unverifiedTargetSdkVersion, in ImeOnBackInvokedDispatcher imeDispatcher);
    void showInputMethodPickerFromClient(in IInputMethodClient client,
            int auxiliarySubtypeMode);
    void showInputMethodPickerFromSystem(in IInputMethodClient client,
            int auxiliarySubtypeMode, int displayId);
    void showInputMethodAndSubtypeEnablerFromClient(in IInputMethodClient client, String topId);
    boolean isInputMethodPickerShownForTest();
    InputMethodSubtype getCurrentInputMethodSubtype();
    void setAdditionalInputMethodSubtypes(String id, in InputMethodSubtype[] subtypes);
    // This is kept due to @UnsupportedAppUsage.
    // TODO(Bug 113914148): Consider removing this.
    int getInputMethodWindowVisibleHeight(in IInputMethodClient client);
    oneway void reportVirtualDisplayGeometryAsync(in IInputMethodClient parentClient,
            int childDisplayId, in float[] matrixValues);
    oneway void reportPerceptibleAsync(in IBinder windowToken, boolean perceptible);
    /** Remove the IME surface. Requires INTERNAL_SYSTEM_WINDOW permission. */
    void removeImeSurface();
    /** Remove the IME surface. Requires passing the currently focused window. */
    oneway void removeImeSurfaceFromWindowAsync(in IBinder windowToken);
    void startProtoDump(in byte[] protoDump, int source, String where);
    boolean isImeTraceEnabled();
    // Starts an ime trace.
    void startImeTrace();
    // Stops an ime trace.
    void stopImeTrace();
    /** Start Stylus handwriting session **/
    void startStylusHandwriting(in IInputMethodClient client);
}
```

</details>

>An **AIDL** is an interface written in **C++** that both client and service agreed upon in order to communicate with each other using **IPC**.
>
>After writing an AIDL, Android SDK tool will generate a **IBinder** interface `.java` file in `gen/` directory with the same name as `aidl`.
>
> Through





### InputMethodManager & InputMethodManagerService




### Soft Keyboard
#### Check Visibility
In this section, let's take a look at how we can detect the keyboard visibility.

##### ADB

```shell
adb shell dumpsys window InputMethod | grep "mHasSurface"
```
<u>**Understand the Code**</u><br>
Here we are using the tool <u>[dumpsys](https://developer.android.com/studio/command-line/dumpsys)</u> that runs in Android devices to fetch info on **InputMethod** from the **window** service.

And in order to communicate with Android, we need to do so via Android Debug Bridge [ADB](https://developer.android.com/studio/command-line/adb).

By running this code, if keyboard is visible, then we will get :
```
mHasSurface=true mShownFrame=[0.0,377.0][1080.0,1794.0] isReadyForDisplay()=true
```
Otherwise :
```
mHasSurface=false mShownFrame=[0.0,377.0][1080.0,1794.0] isReadyForDisplay()=false
```


However, this is useless when we are running the app.

##### ViewTreeObserver
(source)[https://www.jianshu.com/p/ba6b0e08ba63]

```Java
fun Activity.observeKeyboardChange(onChange: (isShowing: Boolean) -> Unit) {
    val rootView = this.window.decorView
    val r = Rect()
    var lastHeight = 0
    rootView.viewTreeObserver.addOnGlobalLayoutListener {
        rootView.getWindowVisibleDisplayFrame(r)
        val height = r.height()
        if (lastHeight == 0) {
            lastHeight = height
        } else {
            val diff = lastHeight - height
            if (diff > 200) {
                onChange(true)
                lastHeight = height
            } else if (diff < -200) {
                onChange(false)
                lastHeight = height
            }
        }
    }
}
```

##### InputMethodManager

##### Manifest + onConfigurationChange


##### API 21
