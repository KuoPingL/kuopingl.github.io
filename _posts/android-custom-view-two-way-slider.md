---
layout: post
title:  "Android Tutorial - Creating a Custom View : Two Way Slider v_1"
date:   2022-08-08 16:15:07 +0800
categories: [custom-view, tutorial, android, intermediate]
---

## Background
> Have you ever face a situation where you just wish to crank two values in a single slider ?
> Well, that's what happen during the process of creating a sample app to show Animation demos.

> <span style="color:rgb(255,100,255);"><b>NOTE</b></span><br> that being said, we could simply use [Material Slider](https://material.io/components/sliders#anatomy), which also support range selection.
> <br>But who needs a wheel when we can reinvent it, right ? XDD

If you are not familiar with Animation in Android, feel free to look at this [series]().

You will find that animations are nothing but changing a value from one to another, which usually involved with 2 or more values.

In my case, I just need to create a slider that can give 2 values (from and to).

Here is what we will create :

![](/images/posts/jekyll/2022-09-21-android-custom-view-two-way-slider-v1-demo.gif)

Enough said, let's hit the keyboard.

## Understand the Structure
In order to create a two-way slider, we need to understand how it looks and how it works.

<img class="centerH" src="/images/posts/jekyll/2022-09-21-android-custom-view-two-way-slider-v1-idle-structure.png"/>
<p class="centerH">Two Way Slider Structure in idle State</p>

> The Slider is consisted of the followings :
1. The bar that sliders slide on
2. The thumbs
3. The line connecting the sliders (Selected Range)
4. (optional) Values on each ends
5. (optional) Bubble to show current value
6. (optional) Ticks on the bar

Users can interact with the slider by sliding the thumbs and the thumb should automatically shift to its destination or step.

While sliding the thumb, a label will pop on top of the moving thumb :

<img class="centerH" src="/images/posts/jekyll/2022-09-21-android-custom-view-two-way-slider-v1-moving-structure.png"/>
<p class="centerH">Two Way Slider Structure in moving State</p>

Now that we got the structure clear, let's decide how do we make it.

> Since the slider needs to be touchable, so it must either be `ViewGroup` or a `View`.
> <br>But because it is not a complicated structure with multiple components, I have decided to go with `View`.

```Java
public class TwoWaySlider extends View {
      // need to add public to be accessible, otherwise, this is considered protected, since it is in a library
      public TwoWaySlider(@NonNull Context context) {
          this(context, null);
      }

      public TwoWaySlider(@NonNull Context context, @Nullable AttributeSet attrs) {
          this(context, attrs, 0);
      }
      public TwoWaySlider(@NonNull Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
          super(context, attrs, defStyleAttr);
      }
}
```
> Remember to add public modifier at the constructor, such that classes outside of the library can also use it.

Next, we need to decide what components does it contain.

### Properties
> Here we will define all the properties needed for the components mentioned above.

#### Bar & Selected Range
> The Bar is the main component that other drawings will be based on.
> <br> Therefore, other than a `Paint`, it also required a `RectF` to store its location and dimension.

<details>
<summary><b>Bar</b></summary>

```Java
private final Paint _barPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
private final RectF _barRect = new RectF();

public void setBarColor(int color) {
    _barPaint.setColor(color);
}

@RequiresApi(api = Build.VERSION_CODES.Q)
public void setBarColor(Long color) {
    _barPaint.setColor(color);
}

public void setBarStrokeWidth(float strokeWidth) {
    if (strokeWidth == _barPaint.getStrokeWidth()) return;
    _barPaint.setStrokeWidth(strokeWidth);
    postInvalidate();
}
```

</details>

> Since a selected range is a simplified version of a bar, I will also include it here
<details>
<summary><b>Selected Range</b></summary>

```Java
private final Paint _selectedRangePaint = new Paint(Paint.ANTI_ALIAS_FLAG);

public void setSelectedRangeColor(int color) {
    _selectedRangePaint.setColor(color);
    postInvalidate();
}

public void setSelectedRangeStrokeWidth(float width) {
    if (width < 0) return;
    _selectedRangePaint.setStrokeWidth(width);
    postInvalidate();
}

public void setSelectedRangePaint(Paint paint) {
    _selectedRangePaint.set(paint);
    postInvalidate();
}

Paint getSelectedRangePaint() {
    return _selectedRangePaint;
}
```

</details>

#### Thumb
> The Thumb is similar to Bar, but only a shorter version and is slidable.
> <br>But having multiple `Paint` and `RectF` laying around might make the code messy.
> <br> So instead, a `Thumb` class is created just to store info about a `Thumb`, such as its rect, paint and current state (`idle` or `moving`).
> <br> As for the step the `Thumb` is currently on, it will be calculated by other function.

<details>
<summary><b>Thumb</b></summary>

```Java
public class Thumb {

    public final static int IDLE = 0x00000000;
    public final static int MOVING = 0x00000001;

    public int state = STABLE;
    public RectF rect;
    public Paint paint;

    public Thumb(RectF rectF, Paint paint) {
        this.rect = rectF;
        this.paint = paint;
    }

    public boolean isMoving() {
        return state == MOVING;
    }

    public void setState(int state) {
        if (state == STABLE || state == MOVING) {
            this.state = state;
        }
    }

    public void setColor(int color) {
        paint.setColor(color);
    }

    public int getColor() {
        return paint.getColor();
    }

    public void setPaintStyle(Paint.Style style) {
        paint.setStyle(style);
    }

    public Paint getPaint() {
        return paint;
    }

    public void setRectF(RectF rect) {
        this.rect.set(rect);
    }

    public void setRectF(float left, float top, float right, float bottom) {
        this.rect.set(left, top, right, bottom);
    }
}
```

</details>

<details>
<summary><b>Thumb Properties</b></summary>

```Java
private final Thumb _thumbFrom = new Thumb(new RectF(), new Paint(Paint.ANTI_ALIAS_FLAG));
private final Thumb _thumbTo = new Thumb(new RectF(), new Paint(Paint.ANTI_ALIAS_FLAG));

private float _thumbSize;
private float _thumbFromCenter, _thumbToCenter;

// current step
private int _thumbFromStep = Integer.MIN_VALUE;
private int _thumbToStep = Integer.MIN_VALUE;

// used to decide which thumb to be drawn first
private Thumb _lastThumbMoved = _thumbFrom;

void setFromThumbColor(@ColorInt int color) {
    _thumbFrom.setColor(color);
    postInvalidate();
}

void setToThumbColor(@ColorInt int color) {
    _thumbTo.setColor(color);
    postInvalidate();
}

/**
 * @param size in dp
 */
public void setThumbSize(float size) {
    _thumbSize = size;
    postInvalidate();
}

/**
 *
 * @return Thumb Size in db
 */
public float getThumbSize() {
    return _thumbSize;
}
```

</details>

#### Ticks
> The Ticks are nothing but drawings on the canvas, so the only thing we need are `Paint` and a flag for user to decide whether to show ticks or not.

<details>
<summary><b>Ticks</b></summary>

```Java
private boolean _shouldShowTick = true;
private final Paint _tickPaint = new Paint(Paint.ANTI_ALIAS_FLAG);

public void showTick() {
    _shouldShowTick = true;
    postInvalidate();
}
public void hideTick() {
    _shouldShowTick = false;
    postInvalidate();
}
```

</details>
<br>
However, there are situations when we won't allow ticks to be shown, even the user asks for it, for instance :

- when the space to draw all ticks are not enough
- or when the bar is way to thin to show the ticks

These situations will be taken into account during the `onDraw` method.

#### Labels
> There are two types of labels we need to take care of.
> 1. The label showing the values on each end
> 2. The value label or bubble that appear when thumb is being slide.

In order to draw text on canvas, we need to use `TextPaint`, a subclass of `Paint`. But if we also need to know the size of the text, a `Rect` is also needed.

<details>
<summary><b>End Values</b></summary>

```Java
private MathContext _alignmentPrecision = new MathContext(2, RoundingMode.HALF_DOWN);
private int _startValue;
private int _endValue;
private final TextPaint _textPaint = new TextPaint();
private final Rect _textRect = new Rect();
private boolean _shouldShowBottomValue = true;

public void setStartValue(int value) {
    _startValue = value;
    _thumbFromStep = 0;
    calculateAlignmentPrecision();
    postInvalidate();
}

public void setEndValue(int value) {
    _endValue = value;
    _thumbToStep = getNumberOfSteps();
    calculateAlignmentPrecision();
    postInvalidate();
}

public void setTextTypeface(Typeface tf) {
    _textPaint.setTypeface(tf);
    postInvalidate();
}

public void setTextPaint(TextPaint tp) {
    _textPaint.set(tp);
    postInvalidate();
}

private void calculateAlignmentPrecision() {
    int steps = getNumberOfSteps();
    int newPrecision = 0;
    while(steps > 0) {
        newPrecision ++;
        steps /= 10;
    }
    if (_alignmentPrecision.getPrecision() != newPrecision) {
        _alignmentPrecision = new MathContext(newPrecision, RoundingMode.HALF_DOWN);
    }
}

public void hideBottomValue() {
    _shouldShowBottomValue = false;
    requestLayout();
}

public void showBottomValue() {
    _shouldShowBottomValue = true;
    requestLayout();
}
```

</details>


> Since changing the starting and ending values will affect the length of a single step, which will need a new precision when defining each step.
> <br>Therefore, every time they got altered, `calculateAlignmentPrecision` will be called.
> > `calculateAlignmentPrecision` will recalculate the precision needed. The more steps needed, the more accurate precision is required.


<details>
<summary><b>Value Label</b></summary>

```Java
private final PointF[] _arrowPointers = new PointF[]{
        new PointF(), new PointF(), new PointF()
};
private final Path _valueLabelPath = new Path();
private final Paint _labelPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
private boolean _shouldShowValueLabel = true;

public void hideBubbleLabel() {
    _shouldShowValueLabel = false;
    requestLayout();
}

public void showBubbleLabel() {
    _shouldShowValueLabel = true;
    requestLayout();
}
```

</details>

> If we take a closer look at the Value Label, we can see that it can be separated into two shapes : **an arrow** and **a rounded corner rectangle**.

So we will need an `PointF` array to hold the points for the arrow, a `Path` to hold the path of the label and a `Paint` to draw the path.


#### Others
> Other than components mentioned before, we will also need to do calculations or validations upon receiving new values.

Here are a list of functions that do the validations and calculations :

<details>
<summary><b>Precision for values display</b></summary>

```Java
private int _accuracy = 2;
private boolean _showFullPrecision = false;
private MathContext _roundMode = new MathContext(_accuracy, RoundingMode.HALF_DOWN);

/**
 *
 * @param precision is the number of decimal points the slider can show.
 *                  The upper limit is 5, just to keep the ui reasonable.
 */
public void setPrecision(int precision) {
    if (_accuracy == precision) {
        return;
    }

    if (precision > 6) {
        precision = MAX_PRECISION;
    }

    _accuracy = precision;
    _roundMode = new MathContext(_accuracy, RoundingMode.HALF_DOWN);
}

public void shouldShowFullPrecision(boolean shouldShowAllPrecision) {
    _showFullPrecision = shouldShowAllPrecision;
}
```

</details>

> **Precision for values**
> <br> User can choose the precision or number of decimals they wish to show the values to be.
> <br><br> If the values calculated has less decimal than requested, user can call `shouldShowFullPrecision` to make sure enough decimals are presented on the screen.


<details>
<summary><b>Setting the Value for the Thumbs</b></summary>

```Java
/**
 * Setting the value before setting the desired
 * total steps will resulted incorrect values being calculated
 * @param value the actual from value
 */
 public void setFromValue(float value) {
     value = validateValueBeforeSetting(value);
     float step = ((float)(value - _startValue)/(float)(_endValue - _startValue) * (float) _numberOfSteps);

     if (step - (int)step > 0.5) {
         _thumbFromStep = (int) (step + 1);
     } else {
         _thumbFromStep = (int) step;
     }
     postInvalidate();
 }

/**
 * Setting the value before setting the desired
 * total steps will resulted incorrect values being calculated
 * @param value the actual to value
 */
 public void setToValue(float value) {
     value = validateValueBeforeSetting(value);
     float step = ((float)(value - _startValue)/(float)(_endValue - _startValue) * (float) _numberOfSteps);

     if (step - (int)step > 0.5) {
         _thumbToStep = (int) (step + 1);
     } else {
         _thumbToStep = (int) step;
     }
     postInvalidate();
 }

private float validateValueBeforeSetting(float value) {
    if (_startValue < _endValue) {
        if (value < _startValue) {
            value = _startValue;
        } else if (value > _endValue) {
            value = _endValue;
        }

    } else if (_startValue > _endValue) {
        if (value > _startValue) {
            value = _startValue;
        } else if (value < _endValue) {
            value = _endValue;
        }
    }
    return value;
}
```

</details>

> **Setting Thumb Value**
> <br> When setting the values, we need to make sure the values are within the range of `_startValue` and `_endValue`.
> <br><br> After that, we will convert these values into the number of steps the thumb should be in. This is done using ratio.

```Java
_thumbFromStep =
        (int)((float)(value - _startValue)
        / (float)(_endValue - _startValue)
        * (float) _numberOfSteps);
```


<details>
<summary><b>Setting Number of Steps</b></summary>

```Java
private int _numberOfSteps = DEFAULT_NUMBER_OF_STEPS;
/**
 * @param value : To let slider auto calculate the number of steps, set it to {@link DEFAULT_NUMBER_OF_STEPS}.
 *              By default, the slider will return a 2 decimal place value.
 */
public void setNumberOfSteps(int value) throw IOException {
    if (value <= 0 && value != DEFAULT_NUMBER_OF_STEPS) throw new IOException(String.format("Number of step %d cannot be less than zero", value));
    _numberOfSteps = value;
    _thumbToStep = _numberOfSteps ;
    calculateAlignmentPrecision();
    postInvalidate();
}

private int getNumberOfSteps() {
    if (_numberOfSteps == DEFAULT_NUMBER_OF_STEPS)  {
        return Math.abs(_startValue - _endValue) * 100;
    } else {
        return _numberOfSteps;
    }
}
```

</details>

> **Setting Total Number of Steps**
> <br> By default, we will set it to **-1**, which will produces 100 steps between any adjacent integer.
> <br><br>If users decided to define their own number of steps, then we will make sure it has to be greater than 1, unless it is -1, otherwise it will return.


#### Initializing
> Upon creating the `TwoWaySlider`, we can start by initializing some of the parameters we will be using.
>> The initialization process is used to define some of the default values, including the color, the stroke width, the start and end values, as well as the positioning of thumbs.

<details>
<summary><b>Initializing</b></summary>

```Java
void init(Context context) {
    _thumbFrom.setColor(Color.WHITE);
    _thumbFrom.setPaintStyle(Paint.Style.FILL);

    _thumbTo.setColor(Color.BLACK);
    _thumbTo.setPaintStyle(Paint.Style.FILL);

    _barPaint.setColor(Color.LTGRAY);
    _barPaint.setStrokeCap(Paint.Cap.ROUND);
    _barPaint.setStyle(Paint.Style.STROKE);
    _barPaint.setStrokeWidth(getPxFromDp(DEFAULT_BAR_STROKE_WIDTH));

    _selectedRangePaint.setColor(Color.GRAY);
    _selectedRangePaint.setStyle(Paint.Style.STROKE);
    _selectedRangePaint.setStrokeWidth(getPxFromDp(DEFAULT_BAR_STROKE_WIDTH));

    _thumbSize = getPxFromDp(DEFAULT_SLIDER_SIZE);

    _startValue = 0;
    _endValue = 1;

    _textPaint.setTypeface(Typeface.DEFAULT);
    _textPaint.setTextSize(getPxFromSp(DEFAULT_TEXT_SIZE));

    _thumbFromStep = 0;
    _thumbToStep = getNumberOfSteps();

    _labelPaint.setStrokeJoin(Paint.Join.BEVEL);
    _labelPaint.setStyle(Paint.Style.STROKE);
    _labelPaint.setStrokeWidth(getPxFromDp(2));
    _labelPaint.setPathEffect(new CornerPathEffect(5));

    _tickPaint.setStyle(Paint.Style.FILL);
    _tickPaint.setColor(Color.WHITE);
}
```

</details>
<br>
With these basic components all set, let's try to display it.

### Displaying the View
In order to display it properly, we need to override certain methods, namely :
- `onMeasure`
  - responsible for calculating the size of the view.
- `onLayout`
  - responsible for deciding where the view lie.
- `onDraw`
  - responsible for displaying the view on canvas.

> But since we are working with a `View`, we won't be dealing with `onLayout`.

#### onMeasure
```Java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {}
```
> In `onMeasure`, there are several of things we can do, such as:
  -  extract the **MeasureSpec** modes using `MeasureSpec.getMode` to see which modes, `UNSPECIFIED`, `EXACTLY` and `AT_MOST` ,the width and height are in.
>> **EXACTLY** is easy to understand, this is when the view has a specific width or height it wants. Such as when its LayoutParams is set to be `MATCH_PARENT` or specific value.
>> <br><br>**AT_MOST** is also very straight forward, this is when LayoutParams is set to be `WRAP_CONTENT`.
>> <br><br>**UNSPECIFIED** can be a bit tricky. This usually happen when the view in added in a ScrollView, where it can be stretched in any sides it wish.

Based on the `MeasureSpec` mode, we can decide the size of the view that we can draw on, in the `onDraw` method.

However, in our case, we are not interested in the `MeasureSpec`, instead, we only care about the final calculated width and height.

In this case, we can do this :

>- get the measured width and height by calling `super.onMeasure`


<details>
<summary> <b>super.onMeasure</b> </summary>

```Java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}

public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);

    switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
}

protected int getSuggestedMinimumWidth() {
    return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}

protected int getSuggestedMinimumHeight() {
    return (mBackground == null) ? mMinHeight : max(mMinHeight, mBackground.getMinimumHeight());
}
```

</details>

> By calling the `super.onMeasure` method, it will calculate a suggested width and height that suits the current `ViewGroup`.


<br>Therefore, we can make our `onMeasure` to be :
```Java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);

    int actualWidth = getMeasuredWidth();
    int actualHeight = getMeasuredHeight();

    /*
    or we can do this, if we want to ignore the minimum width or height suggested by the background
    int measuredWidth = MeasureSpec.getSize(widthMeasureSpec);
    int measuredHeight = MeasureSpec.getSize(heightMeasureSpec);
    */

    // do final adjustments
}
```

Despite the fact that measured width and height have been calculated, we should also make sure that all components can be fit into the size.

So let's determine the minimum width and height required.

![](/images/posts/jekyll/2022-09-21-android-custom-view-two-way-slider-v1-minimum-size.png)

> **Minimum Width**
><br>The minimum width of the slider should at least contain enough space to put 2 thumbs side by side with text on each ends.
>> Note that I have ignored the possibility that the values at the bottom might overlap each other as well as the possible trimming on the values shown in the value label.
>><br><br> These will be left for the users to decide how to implement this library to best suit their needs.

```Java
minWidth = (int) (thumbSize * 2 + getPaddingLeft() + getPaddingRight());
```

>**Minimum Height**
><br> The minimum height will simply be the stroke width of the bar or the size of the thumb, depending on which one is larger.
><br><br> If user wish to show the bottom labels or the bubble, then the minimum height will add other values to it.

```Java
minHeight = _barPaint.getStrokeWidth() > thumbSize ?
        (int) (_barPaint.getStrokeWidth() + getPaddingTop() + getPaddingBottom()) :
        (int) (thumbSize + getPaddingTop() + getPaddingBottom());

if (_shouldShowValueLabel) {
    minHeight += bubbleToBarDistance + bubbleHeight;
}

if (_shouldShowBottomValue) {
    minHeight += textMaxHeight + getPxFromDp(BAR_TO_TEXT_MARGIN) ;
}
```

<details>
<summary>Constants Used</summary>

```Java
// Bubble Related Constants
static final int BUBBLE_VERTICAL_PADDING = 5;
static final int BUBBLE_HORIZONTAL_PADDING = 10;
static final int BUBBLE_TO_BAR_DISTANCE = 5;

// Slider Related Constants
static final int DEFAULT_SLIDER_SIZE = 20;

// Bar Related Constants
static final int DEFAULT_BAR_STROKE_WIDTH = 8; // Bar Height

// Extra Space to draw the text at the bottom
static final int EXTRA_SPACE = 5;
```

</details>

<br>

After determining the minimum width and height, we can call `setMeasuredDimension` to save the size and width that will be used in `onDraw`.

```Java
setMeasuredDimension(resultWidth, resultHeight);
```

<br>
Here is the complete code in `onMeasure`
<details>
<summary><b>onMeasure Implementation</b></summary>

```Java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);

    int measuredWidth = getMeasuredWidth(); //MeasureSpec.getSize(widthMeasureSpec);
    int widthMode = MeasureSpec.getMode(widthMeasureSpec);
    int measuredHeight = getMeasuredHeight(); //MeasureSpec.getSize(heightMeasureSpec);
    int heightMode = MeasureSpec.getMode(heightMeasureSpec);

    int resultWidth = measuredWidth;
    int resultHeight = measuredHeight;

    // start value text bounds
    String startValueString = String.valueOf(_startValue);
    _textPaint.getTextBounds(startValueString, 0, startValueString.length(), _textRect);
    int startTextWidth = _textRect.width();
    int startTextHeight = _textRect.height();

    // end value text bounds
    String endValueString = String.valueOf(_endValue);
    _textPaint.getTextBounds(endValueString, 0, endValueString.length(), _textRect);
    int endTextWidth = _textRect.width() + 1;

    // longest value text bounds (default 2 decimal place + '.' + largest start value or end value
    zeros.delete(0, zeros.length());
    for (int i = 0; i < _accuracy; i++) {
        zeros.append("0");
    }
    @SuppressLint("DefaultLocale")
    String valueWithDecimalString = String.format("%d.%s", endTextWidth > startTextWidth ? _endValue : _startValue, zeros);
    _textPaint.getTextBounds(valueWithDecimalString, 0, valueWithDecimalString.length(), _textRect);
    float textMaxHeight = _textPaint.getFontMetrics().bottom - _textPaint.getFontMetrics().top;
    float textMaxWidth = _textRect.width();

    // Bubble Height and Width
    float bubbleHeight = textMaxHeight + getPxFromDp(BUBBLE_VERTICAL_PADDING) * 2;
    float bubbleWidth = textMaxWidth + getPxFromDp(BUBBLE_HORIZONTAL_PADDING) * 2;

    // Bubble to Bar Distance
    float bubbleToBarDistance = getPxFromDp(BUBBLE_TO_BAR_DISTANCE);

    float thumbSize = _thumbSize;

    if (_barPaint.getStrokeWidth() > thumbSize) {
        thumbSize = _barPaint.getStrokeWidth();
    }

    int minHeight, minWidth;

    minHeight = _barPaint.getStrokeWidth() > thumbSize ?
            (int) (_barPaint.getStrokeWidth() + getPaddingTop() + getPaddingBottom()) :
            (int) (thumbSize + getPaddingTop() + getPaddingBottom());

    if (_shouldShowValueLabel) {
        minHeight += bubbleToBarDistance + bubbleHeight;
    }

    if (_shouldShowBottomValue) {
        minHeight += textMaxHeight + getPxFromDp(BAR_TO_TEXT_MARGIN) ;
    }

    // The text at the bottom
    minWidth = (int) (thumbSize * 2 + getPaddingLeft() + getPaddingRight());

    resultWidth = Math.max(measuredWidth, minWidth);
    resultHeight = Math.min(measuredHeight, minHeight);
    setMinimumWidth(minWidth);
    setMinimumHeight(minHeight);

    setMeasuredDimension(resultWidth, resultHeight);
}
```
</details>
<br>

Next, though we won't be overriding it, but let's talk about `onLayout` for a bit.

#### onLayout
```Java
@Override
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    super.onLayout(changed, left, top, right, bottom);
}
```

The `onLayout` method will tell you if the layout has changed and what are the bounds of the layout.

> `onLayout` is called when `layout` method is triggered, which is called by **ViewRootImpl**'s method `performLayout`.

```Java
// a method in ViewRootImpl
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,int desiredWindowHeight) {}

// a method in View
public void layout(int l, int t, int r, int b){}
```
This is the place where **ViewGroup** gets to decide where children can be located by calling :

```Java
child.layout(desiredX,desiredY,child.getMeasuredWidth(),child.getMeasuredHeight());
```

Since we are only dealing with a View, we don't really need to customize `onLayout` at all, so let's just leave it be.

Now the layout is ready, we will need to draw it out.

#### onDraw
Here is where we decide how and where to draw our components.

Let's start by considering the boundary of the components, what should we do in the following situations ?
- layout height > measured height
- layout width > measured width

Usually, we would draw the view at the center.
<br>
And to do so, there are two approaches :
1. Use `canvas.translate` to shift the canvas coordinate to a new reference.
2. or simply find the bounds and draw inside that bounds.

Both approaches are feasible, but depends on the property of the components drawn.

>For instance, if all components here are static, and no interactions are required. Then it would be easier to use `canvas.translate` to get rid of the top and right paddings once and for all.
><br><br> But, if you have components that are dynamic or interactive, then it might be better to simply calculate the actual bounds instead.
<br>
Because usually interactive components required extra calculations to figure out its final destination.

In our case, we used the second approach.

Let's go through the `onDraw` process step by step.

>**Preparing basic variables**
><br>First, we will start by calculating the paddings on both the view and the bar.

```Java
@Override
protected void onDraw(Canvas canvas) {
    // Draw Background
    super.onDraw(canvas);
    // After API 17, you can call getPaddingStart() and getPaddingEnd() instead, the API used in this project is 16.
    float startPadding = (float) getPaddingLeft();
    float endPadding = (float) getPaddingRight();
    if (ViewCompat.getLayoutDirection(this) == ViewCompat.LAYOUT_DIRECTION_RTL) {
        startPadding = (float) getPaddingRight();
        endPadding = (float) getPaddingLeft();
    }

    float barEndPadding = _thumbSize /2;

    // incase the strokeWidth is larger
    if (_barPaint.getStrokeWidth() > _thumbSize) {
        barEndPadding = _barPaint.getStrokeWidth() / 2;
    }
}
```

>**Drawing the labels at both ends**
><br>Next, we will start by drawing the labels at both ends.
><br><br>Drawing a text takes two steps :
> 1. Find the width and height of the Text to be drawn using `textPaint.getTextBounds`
> 2. Find the coordinate of the **baseline** (instead of finding the coordinate of the top-left corner).

```Java
// 1. Use getTextBounds to find the width of the Text.
_textPaint.getTextBounds(
    String.valueOf(_startValue),
    0, String.valueOf(_startValue).length(),
    _textRect);

float startValueWidth = _textRect.width();

_textPaint.getTextBounds(
    String.valueOf(_endValue),
    0, String.valueOf(_endValue).length(),
    _textRect);

float endValueWidth = _textRect.width();

// 2. Locate the baseline of the text and draw the text
canvas.drawText(
    String.valueOf(_startValue),
    (barEndPadding < startValueWidth / 2 ?
        getPxFromDp(EXTRA_SPACE):
        barEndPadding - 0.5f * startValueWidth) + startPadding,
    (float) (getMeasuredHeight() - getPaddingBottom()),
    _textPaint);

canvas.drawText(
    String.valueOf(_endValue),
    (float) getMeasuredWidth() - endPadding - (barEndPadding < endValueWidth ?
         endValueWidth + getPxFromDp(EXTRA_SPACE):
         barEndPadding * 2 - (barEndPadding * 2 - endValueWidth) / 2),
(float) (getMeasuredHeight() - getPaddingBottom()),_textPaint);
```
> The second step might looks complicated, let me try to walk you though the drawing of the **_startValue**.
> <br><br> a `canvas.drawText` method required 4 parameters :
> 1. The `x` value of x-axis, this is the value where the text will start to draw.
> 2. The `y` value of y-axis, this is where the **baseline** is located.
> 3. The text to be drawn.
> 4. The `TextPaint` that is responsible for drawing the text.

Here is how we find the `x` value :

> If the width of **_startValue** is less than the **barEndPadding** (`thumbSize / 2`), then the text won't be centered to the thumb, and it will be placed as close to the end as possible (**startPadding**).
> >The reason why the end padding is `thumbSize / 2`, is because when we make the bar line rounded cap and the stroke width the same as the thumb size, then it will create a rounded edge at the endings with the same radius as the thumb.
>
>  Else, the text will be centered with the starting point of the line, which is located at `startPadding + (barEndPadding - 0.5 * endValueWidth)`

As to find `y` value, I think you can figure it out.

Also, the code above will be wrapped inside a if statement :
```Java
float textHeight = 0;
if (_shouldShowBottomValue) {

    // INSERT HERE .... code for drawing text

    textHeight = _textPaint.getFontMetrics().bottom - _textPaint.getFontMetrics().top;
    // or we can use _textRect.height() to get in integer
}
```

With the text drawn, we can now draw the bar.

##### Bar
<img class="centerH" src="/images/posts/jekyll/2022-09-21-android-custom-view-two-way-slider-v1-idle-structure.png"/>

<br>

> Based on the design, the bar is placed above the end points text that we drawn above.

Instead of making a dynamic value to define the spacing or margin between the text and the bar, we used a constant :

```Java
static final int BAR_TO_TEXT_MARGIN = 3;
```

> However, if we want to get the location of the bar on the y-axis, it's easier to find the bar bottom and subtract our way to the center of the bar.

The bottom of the bar can be found :
```Java
float barBottom = (getBottom() - getTop() - getMeasuredHeight()) / 2f + (float) getMeasuredHeight();
```

As for the center of the bar, we need to take into account the stroke width :
```Java
float barCenterY = barBottom - getPaddingBottom() - _barPaint.getStrokeWidth() / 2f;
```

And if the bottom text is to be drawn, we also need to take the text height and margin into account :
```Java
if (_shouldShowBottomValue) {
  barCenterY -= (textHeight + getPxFromDp(BAR_TO_TEXT_MARGIN));
}
```

> With the `y` location found, we need to figure out the `x` location of the bar, as well as the `length` of the bar.

Obviously, the `x` location is equals to the `startPadding + barEndPadding`.

As for the length, we can calculate it from the very end and inwards :
```java
// the formula is expanded just to make it clear to understand
float barLength = getMeasuredWidth() - barEndPadding - endPadding -  startPadding - barEndPadding;
```

We will also save the Rect of the bar for calculating position of the thumb later.
```Java
_barRect.set( barEndPadding + startPadding,
                barCenterY,
                barEndPadding + startPadding + barLength,
                barCenterY);
```

So at the end here is the code for drawing the bar :
<details>
<summary><b>Code for drawing the Bar </b></summary>

```Java
float barBottom = (getBottom() - getTop() - getMeasuredHeight()) / 2f + (float) getMeasuredHeight();
float barCenterY = barBottom - getPaddingBottom() - _barPaint.getStrokeWidth() / 2f;

if (_shouldShowBottomValue) {
barCenterY -= (textHeight + getPxFromDp(BAR_TO_TEXT_MARGIN));
}

float barLength = getMeasuredWidth() - startPadding - endPadding - barEndPadding * 2;
_barRect.set(
    barEndPadding + startPadding,
    barCenterY,
    barEndPadding + startPadding + barLength,
    barCenterY
    );
canvas.drawLine(
    _barRect.left, _barRect.top,
    _barRect.right, _barRect.bottom,
    _barPaint);
```

</details>

We are almost done with the drawings, next we will draw the thumb.

##### Display the Thumbs & Selected Range
> In order to draw a thumb, we need to think of the following questions:
> 1. Where exactly the thumb should be?
> 2. What value does it represent?
> 3. Can we easily find the proper position when given a value or vice versa ?
> 4. Would I be able to get the same value if configuration has changed ? (the slider could be different length)

Let's walk it through on how to solve all these questions.

The first thing that came in mind when dealing with these questions was :
> I can simply solve them using **percentage**

Is that also how you think ?

Well, you are on the right track, however, a **percentage** usually means we need to divide the bar into a constant of **100** sections, which <u>wouldn't work</u>.

Because, a slider usually returns value **continuously**.
<br>Meaning, the bar can be divided into as many sections as needed or allowed.

<u>Does this mean we are flashing the **percentage** idea is down the drain ?</u>
<br>Not exactly.

We will simply use the ratio between the location of the thumb and the length of the bar and multiply by the total number of steps to find the step that the thumb is on :

```
The Step the Thumb Represents =
        (Location of the Thumb along the Bar) / (Length of the Bar) * (Total Number of Steps)
```

```Java
int calculateThumbStep(float thumbCenter) {

    double steps = (double) (thumbCenter - _barRect.left) / (_barRect.width()) * getNumberOfSteps();
    double diff = steps - (int) Math.floor(steps);
    if (diff >= 0.5) {
        steps++;
    }

    return (int) steps;
}
```

And to find the location of the thumb along the bar based on the step it is on, we can do something similar :
```
The Thumb Location = (Step of the Thumb) / (Total Number of Steps) * (Length of the Bar)
```

This seems reasonable, however, we need to be caution when dealing with float calculations due to its nature of being [inconsistent](https://learn.microsoft.com/en-us/cpp/build/why-floating-point-numbers-may-lose-precision?view=msvc-170).
> PS : I didn't use `BigDecimal` when calculating the step because the result will returns an `int`, so it doesn't need that kind of accuracy.

So we will convert all values into `BigDecimal` instead.
<br>Also, don't forget to make sure the thumb must stay within the bar.
```Java
float calculateThumbXPosition(int thumbPosition, float barWidth, float startPadding) {

    float barEndPadding = _thumbSize / 2f;
    if (_barPaint.getStrokeWidth() > _thumbSize) {
        barEndPadding = _barPaint.getStrokeWidth() / 2f;
    }

    // due to the canvas translation, thumbCenterX can ignore startPadding
    float thumbCenterX =  BigDecimal.valueOf((double) thumbPosition)
            .divide(BigDecimal.valueOf((double) getNumberOfSteps()), _alignmentPrecision)
            .multiply(BigDecimal.valueOf((double) barWidth))
            .floatValue() + barEndPadding + startPadding;

    // Make sure the thumb stays inside the bar
    if (thumbCenterX < barEndPadding + startPadding) {
        thumbCenterX = barEndPadding + startPadding;
    } else if (thumbCenterX > barEndPadding + barWidth + startPadding) {
        thumbCenterX =  barEndPadding + barWidth + startPadding;
    }

    return thumbCenterX;
}
```

> Now that the thumb location can be calculated, we can now draw the thumbs.
> <br>But before diving in, we need to keep in mind the thumb is movable.
> <br> Meaning, when the thumb is being moved by the user, we should let it go as it pleased, just to make sure it stays within the bar.
> <br><br> But once user stop moving it, it has to snap to the closest value.

```Java
// Make sure the thumb will snap to the closest value
if (!_thumbFrom.isMoving()) {
    _thumbFromCenter = calculateThumbXPosition(
            _thumbFromStep, barLength,
            startPadding);
}

float thumbOriginX = _thumbFromCenter - _thumbSize / 2;

// Remember the RectF for thumbs
_thumbFrom.setRectF(thumbOriginX, barCenterY - _thumbSize / 2, thumbOriginX + _thumbSize, barCenterY + _thumbSize / 2);

// Same thing goes with the To Thumb
if (!_thumbTo.isMoving()) {
    _thumbToCenter = calculateThumbXPosition(
            _thumbToStep, barLength,
            startPadding);
}
thumbOriginX = _thumbToCenter - _thumbSize / 2;
_thumbTo.setRectF(thumbOriginX, barCenterY - _thumbSize / 2, thumbOriginX + _thumbSize, barCenterY + _thumbSize / 2);
```

> With the thumbs located, we can first draw the line connecting them, such that it won't be on top of the thumb.

```Java
float x0 = _thumbFromCenter;
float x1 = _thumbToCenter;

if (x0 > x1) {
    x0 = _thumbToCenter;
    x1 = _thumbFromCenter;
}
canvas.drawLine(x0, barCenterY, x1, barCenterY, _selectedRangePaint);
```

> Similarly, we want the thumb that is moving to be on the top :
```Java
if (_thumbFrom.isMoving() || _lastThumbMoved == _thumbFrom) {
    canvas.drawOval(_thumbTo.rect, _thumbTo.paint);
    canvas.drawOval(_thumbFrom.rect, _thumbFrom.paint);
} else {
    canvas.drawOval(_thumbFrom.rect, _thumbFrom.paint);
    canvas.drawOval(_thumbTo.rect, _thumbTo.paint);
}
```
Thank God you've made it this far, we are nearly done with the `onDraw` method.

##### Showing the Value Label
> So far, you must be wondering why we need to keep the rect of **_thumbFrom** and **_thumbTo**, well this is why.
> <br><br> In order to draw the Value Label, we have to know the exact location of the thumb.
> <br> Not only because it is located on top of the thumb, but also because we have to adjust the the rectangular shape portion when approaching the endings.

To draw the label, we need the followings :
1. the Canvas
2. the Thumb RectF
3. the value to be wrapped by the label
4. the color of the label
5. the start and end padding of the View (boundary)

We will define a method to take all these in :
```Java
void drawValueLabelForThumb(Canvas canvas, RectF thumbRect, String value, int color, float startPadding, float endPadding) {}
```
<details>
<summary><b>Here is the full code </b></summary>

```Java
void drawValueLabelForThumb(Canvas canvas, RectF thumbRect, String value, int color, float startPadding, float endPadding) {

    if (_showFullPrecision) {
        String[] strings = value.split("[.]");

        while (strings[1].length() < _accuracy) {
            strings[1] = strings[1].concat("0");
        }

        value = strings[0] + "." + strings[1];
    }

    _textPaint.getTextBounds(value, 0, value.length(), _textRect);
    _labelPaint.setColor(color);

    float thumbSize = _thumbSize;
    float labelToBarDistance = getPxFromDp(BUBBLE_TO_BAR_DISTANCE);
    _arrowPointers[0].set(thumbRect.centerX() - thumbSize /2f, thumbRect.top - labelToBarDistance);
    _arrowPointers[1].set(thumbRect.centerX(), thumbRect.top);
    _arrowPointers[2].set(thumbRect.centerX() + thumbSize /2f, thumbRect.top - labelToBarDistance);

    _valueLabelPath.reset();

    // calculate the proper width
    float valueLabelLeft = _textRect.width()/2f + getPxFromDp(BUBBLE_HORIZONTAL_PADDING) + startPadding > thumbRect.centerX() ?
            startPadding + _labelPaint.getStrokeWidth():
            thumbRect.centerX() - _textRect.width()/2f - getPxFromDp(BUBBLE_HORIZONTAL_PADDING);

    if (valueLabelLeft == _labelPaint.getStrokeWidth() + startPadding) {
        // far left
        _arrowPointers[0].set(valueLabelLeft, _arrowPointers[0].y);
    }

    float valueLabelBottom = thumbRect.top - labelToBarDistance;
    float valueLabelTop = valueLabelBottom - getPxFromDp(BUBBLE_VERTICAL_PADDING) * 2 - _textRect.height();

    /*
        Here is the structure of the value label
        (value label left) | -(BUBBLE_HORIZONTAL_PADDING) - (text) - (BUBBLE_HORIZONTAL_PADDING) -| (value Label Right)

        If it approach the far right, then :
        valueLabelRight = getMeasuredWidth - endPadding - paintLabel.strokeWidth
     */
    float valueLabelRight = valueLabelLeft + _textRect.width() + getPxFromDp(BUBBLE_HORIZONTAL_PADDING) * 2 > getMeasuredWidth() - endPadding ?
            getMeasuredWidth() - _labelPaint.getStrokeWidth() - endPadding:
            valueLabelLeft + _textRect.width() + getPxFromDp(BUBBLE_HORIZONTAL_PADDING) * 2;

    if (valueLabelRight == getMeasuredWidth() - _labelPaint.getStrokeWidth() - endPadding) {
        // this means the thumb is at the far right
        valueLabelLeft = valueLabelRight - (_textRect.width() + getPxFromDp(BUBBLE_HORIZONTAL_PADDING) * 2);
        _arrowPointers[2].set(valueLabelRight, _arrowPointers[2].y);
    }

    _valueLabelPath.moveTo(_arrowPointers[0].x, _arrowPointers[0].y);
    _valueLabelPath.lineTo(_arrowPointers[1].x, _arrowPointers[1].y);
    _valueLabelPath.lineTo(_arrowPointers[2].x, _arrowPointers[2].y);
    _valueLabelPath.lineTo(valueLabelRight, _arrowPointers[2].y);
    _valueLabelPath.lineTo(valueLabelRight, valueLabelTop);
    _valueLabelPath.lineTo(valueLabelLeft, valueLabelTop);
    _valueLabelPath.lineTo(valueLabelLeft, _arrowPointers[0].y);
    _valueLabelPath.close();

    canvas.drawPath(_valueLabelPath, _labelPaint);
    canvas.drawText(value, valueLabelLeft + getPxFromDp(BUBBLE_HORIZONTAL_PADDING), valueLabelBottom - getPxFromDp(BUBBLE_VERTICAL_PADDING), _textPaint);
}
```

</details>

<br>
Let's take a look at what `drawValueLabelForThumb` does.

>**Getting the Value Width**
><br> If full precision is to be shown, then 0 will be appended at the end when needed.

```Java
if (_showFullPrecision) {
    String[] strings = value.split("[.]");

    while (strings[1].length() < _accuracy) {
        strings[1] = strings[1].concat("0");
    }

    value = strings[0] + "." + strings[1];
}

_textPaint.getTextBounds(value, 0, value.length(), _textRect);
```

> **Define the Arrow**
> <br>We will define the points that are needed to draw the arrow as follows :

```java
float thumbSize = _thumbSize;
// BUBBLE_TO_BAR_DISTANCE = 10;
float labelToBarDistance = getPxFromDp(BUBBLE_TO_BAR_DISTANCE);
_arrowPointers[0].set(thumbRect.centerX() - thumbSize /2f, thumbRect.top - labelToBarDistance);
_arrowPointers[1].set(thumbRect.centerX(), thumbRect.top);
_arrowPointers[2].set(thumbRect.centerX() + thumbSize /2f, thumbRect.top - labelToBarDistance);
```

Basically, the center point is right above the thumb, whereas the two adjacent points are aligned to the two sides of the thumb with a constant distance above.

Finally, we just need to define the rectangular part of the label.

> **Defining the Rectangular Shape**
> <br><br>It will be defined similar to `Rect`, we just find its **left**, **right**, **top** and **bottom**

To find the left of the label, we need to check if the label will be drawn out of the view when it is on the far left of the bar. We can use the following to do the checking :

```Java
_textRect.width()/2f + getPxFromDp(BUBBLE_HORIZONTAL_PADDING) + startPadding > thumbRect.centerX();
```

If the left hand side is greater than the center, this means the rectangular shape will not be centered to the thumb. In this case, we need to shift the rectangle to the right.

<u>So how far to the right?</u>
<br>
 We simply make it equals to the `startPadding`, and remember to also take the strokeWidth into consideration :
 ```Java
startPadding + _labelPaint.getStrokeWidth():
 ```
And if the *right hand side is greater*, then we will simply do a calculation :
```Java
thumbRect.centerX() - _textRect.width()/2f - getPxFromDp(BUBBLE_HORIZONTAL_PADDING)
```
If we put them together, we'll get :
```Java
float valueLabelLeft = _textRect.width()/2f + getPxFromDp(BUBBLE_HORIZONTAL_PADDING) + startPadding > thumbRect.centerX() ?
        startPadding + _labelPaint.getStrokeWidth():
        thumbRect.centerX() - _textRect.width()/2f - getPxFromDp(BUBBLE_HORIZONTAL_PADDING);
```

> If the label is sticking to the left border, then we also need to make sure the left most point of the arrow is the same value as `valueLabelLeft`.

```Java
if (valueLabelLeft == _labelPaint.getStrokeWidth() + startPadding) {
    // far left
    _arrowPointers[0].set(valueLabelLeft, _arrowPointers[0].y);
}
```

> The top and bottom of the label are easy to calculate since they won't be affected by the the border :
> ```Java
> float valueLabelBottom = thumbRect.top - labelToBarDistance;
float valueLabelTop = valueLabelBottom - getPxFromDp(BUBBLE_VERTICAL_PADDING) * 2 - _textRect.height();
```

Now all is left is to find the **right** border of the label.
<br> Since we already found `valueLabelLeft`, then finding `valueLabelRight` should be a piece of cake.

Unfortunately, that's being too optimistics.

We also need to consider when the label reaches the `endPadding`.

```Java
valueLabelLeft + _textRect.width() + getPxFromDp(BUBBLE_HORIZONTAL_PADDING) * 2 > getMeasuredWidth() - endPadding
```

If it reaches the end padding, we will simply make the right border to be :
```Java
getMeasuredWidth() - endPadding - _labelPaint.getStrokeWidth()
```

Otherwise, it should be equals to :
```Java
valueLabelLeft + _textRect.width() + getPxFromDp(BUBBLE_HORIZONTAL_PADDING) * 2
```
Oh, and don't forget when the label reaches the far end, the corresponding point on the arrow should also shift along it.

> Putting them together, we will get :

```Java
float valueLabelRight = valueLabelLeft + _textRect.width() + getPxFromDp(BUBBLE_HORIZONTAL_PADDING) * 2 > getMeasuredWidth() - endPadding ?
        getMeasuredWidth() - _labelPaint.getStrokeWidth() - endPadding:
        valueLabelLeft + _textRect.width() + getPxFromDp(BUBBLE_HORIZONTAL_PADDING) * 2;
if (valueLabelRight == getMeasuredWidth() - _labelPaint.getStrokeWidth() - endPadding) {
    // this means the thumb is at the far right
    valueLabelLeft = valueLabelRight - (_textRect.width() + getPxFromDp(BUBBLE_HORIZONTAL_PADDING) * 2);
    _arrowPointers[2].set(valueLabelRight, _arrowPointers[2].y);
}
```
> One final step is to draw it out :

```Java
_valueLabelPath.moveTo(_arrowPointers[0].x, _arrowPointers[0].y);
_valueLabelPath.lineTo(_arrowPointers[1].x, _arrowPointers[1].y);
_valueLabelPath.lineTo(_arrowPointers[2].x, _arrowPointers[2].y);
_valueLabelPath.lineTo(valueLabelRight, _arrowPointers[2].y);
_valueLabelPath.lineTo(valueLabelRight, valueLabelTop);
_valueLabelPath.lineTo(valueLabelLeft, valueLabelTop);
_valueLabelPath.lineTo(valueLabelLeft, _arrowPointers[0].y);
_valueLabelPath.close();

canvas.drawPath(_valueLabelPath, _labelPaint);
canvas.drawText(value, valueLabelLeft + getPxFromDp(BUBBLE_HORIZONTAL_PADDING), valueLabelBottom - getPxFromDp(BUBBLE_VERTICAL_PADDING), _textPaint);
```
> **Calling of `drawValueLabelForThumb`**
><br> This function is called inside the `onDraw` method, right after drawing the thumbs. Whenever the thumb is about to move or is currently moving.

```Java
if (_thumbTo.isMoving()) {
    // draw the indicator
    drawValueLabelForThumb(canvas, _thumbTo.rect, String.valueOf(getToValue()), _thumbTo.getColor(), startPadding, endPadding);
}

if (_thumbFrom.isMoving()) {
    // draw the indicator
    drawValueLabelForThumb(canvas, _thumbFrom.rect, String.valueOf(getFromValue()), _thumbFrom.getColor(), startPadding, endPadding);
}
```
##### Showing Ticks
> Ticks are labels indicating the step on the slider.
> <br>It's quite simple, so no explanation will be provided.

```Java
if (_shouldShowTick && getNumberOfSteps() != DEFAULT_NUMBER_OF_STEPS && getNumberOfSteps() > 2 && barLength / getNumberOfSteps() > getPxFromDp(3)) {
    for (int i = 0; i <= getNumberOfSteps(); i++) {
        float x = getPositionForStep(i, barLength, startPadding);
        float r = ((_barRect.bottom - _barRect.top) - getPxFromDp(2)) / 6f;

        if (r * 2 < barLength / getNumberOfSteps()) {
            r = (barLength / getNumberOfSteps()) / 6f;
        }

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            canvas.drawOval(x - r, barCenterY - r, x + r, barCenterY + r, _tickPaint);
        } else {
            canvas.drawCircle(x, barCenterY, r, _tickPaint);
        }
    }
}
```

YAY , we are done ... on `onDraw`.
<br> Now comes the easy parts, `onTouchEvent`.

#### Enable Interactions
> First thing first, while user interacts with the slider, we would want to get a feedback on what is going on.
> <br> To do so, we will need a **Listener**, an interface that the slider can notify when values are being changed.


##### Interface
The interface does not need to be fancy or anything, as long as it can provide us with the values needed.

So in our case, we can simply make a **SAM** (**S**ingle **A**bstract **M**ethod), an interface with a single method.

```java
interface Listener {
    void onSliderMoved(float from, float to);
}
``````

Next, let's take a look what happen when user interacts.

##### onTouchEvent
> The 4 events that we are interested in are :
> 1. MotionEvent.**ACTION_DOWN**
> 2. MotionEvent.**ACTION_MOVE**
> 3. MotionEvent.**ACTION_UP**
> 4. MotionEvent.**ACTION_CANCEL**

###### Action Down
> Here is where we check if the touch event happens inside the thumbs.
> <br>If it is, then we need to set up the flags, so the canvas can be drawn properly.

```Java
case MotionEvent.ACTION_DOWN:

      if (_lastThumbMoved.rect.contains(x,y)) {
          _lastThumbMoved.setState(Thumb.MOVING);
          postInvalidate();
          return true;
      } else {
          if (_thumbTo.rect.contains(x, y)) {
              _thumbTo.setState(Thumb.MOVING);
              _lastThumbMoved = _thumbTo;
              postInvalidate();
              return true;
          } else if (_thumbFrom.rect.contains(x, y)) {
              _thumbFrom.setState(Thumb.MOVING);
              _lastThumbMoved = _thumbFrom;
              postInvalidate();
              return true;
          }
      }
```

Returning `true` means we will deal with this interaction manually.

###### Action Move
> Here is where we calculate the new value and feedback to the listener.
><br><br><b>Note</b>
><br>We also need to make sure the thumb should never move out of the bar.

 ```Java
 case MotionEvent.ACTION_MOVE:
     if (_thumbFrom.isMoving() || _thumbTo.isMoving()) {

         if (_thumbFrom.isMoving()) {
             _thumbFromCenter = x;
             if (_thumbFromCenter < _barRect.left) {
                 _thumbFromCenter = _barRect.left;
             } else if (_thumbFromCenter > _barRect.right) {
                 _thumbFromCenter = _barRect.right;
             }
             // save thumb location
             _thumbFromStep = calculateThumbStep(_thumbFromCenter);
         }

         if (_thumbTo.isMoving()) {
             _thumbToCenter = x;
             if (_thumbToCenter < _barRect.left) {
                 _thumbToCenter = _barRect.left;
             } else if (_thumbToCenter > _barRect.right) {
                 _thumbToCenter = _barRect.right;
             }
             // save thumb location
             _thumbToStep = calculateThumbStep(_thumbToCenter);
         }

         if (_l != null) {
             _l.onSliderMoved(getFromValue(), getToValue());
         }

         // performClick is mainly used for people who need accessibility.
         if (!_thumbTo.isMoving() && !_thumbFrom.isMoving()) {
             performClick();
         }


         postInvalidate();
         return true;
     }
 ```

 ###### Action Up / Action Cancel
 > Last but not least, we need to make sure whenever user finishes with interacting with the slider, we need to reset all flags.

 ```java
 case MotionEvent.ACTION_UP:
 case MotionEvent.ACTION_CANCEL:
     _thumbTo.setState(Thumb.IDLE);
     _thumbFrom.setState(Thumb.IDLE);
     postInvalidate();
     return true;
 ```

That is it, we have now created a working two way slider.

But wait, ever wonder what will happen when the app restarted or when configuration changes occur, what will be shown on the slider?

Will it be the same value as before ? Or will the thumbs be in the default states ?

Let's find out and see how we fix it.

### Configuration change
> Here is what happen when configuration changes occur :

![](/images/posts/jekyll/2022-09-21-android-custom-view-two-way-slider-v1-config-change.gif)

As you can see, the thumb will reset back to whatever value we had set in `onCreate` method.

```Java
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    _binding = ActivityAnimationDemoBinding.inflate(layoutInflater)
    setContentView(_binding.root)
    _binding.slider.setStartValue(0);
    _binding.slider.setEndValue(10);
    _binding.slider.setNumberOfSteps(10);
    _binding.slider.setPrecision(4);
    _binding.slider.setFromValue(5f);
//        _binding.slider.hideBottomValue();
//        _binding.slider.hideBubbleLabel();
    _binding.slider.setBarStrokeWidth(_binding.slider.thumbSize + TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, 10f, resources.displayMetrics))
    _binding.slider.setSelectedRangeStrokeWidth(_binding.slider.thumbSize)
}
```

To solve this, we need to override `onSaveIsntanceState` and `onRestoreInstanceState` methods to save and restore the current values:

```Java
@Nullable
@Override
protected Parcelable onSaveInstanceState() {
    Bundle b = new Bundle();
    b.putParcelable("superState", super.onSaveInstanceState());
    b.putInt(THUMB_TO_STEP, _thumbToStep);
    b.putInt(THUMB_FROM_STEP, _thumbFromStep);
    return b;
}

@Override
protected void onRestoreInstanceState(Parcelable state) {

    if (state instanceof Bundle) {
        Bundle b = (Bundle) state;
        _thumbToStep = b.getInt(THUMB_TO_STEP);
        _thumbFromStep = b.getInt(THUMB_FROM_STEP);
        state = b.getParcelable("superState");
    }

    super.onRestoreInstanceState(state);
}

static final String THUMB_TO_STEP = "Thumb To Step";
static final String THUMB_FROM_STEP = "Thumb From Step";
```

#### XML
> So far, all we did is programmatically create and customize the slider, wouldn't it be easier if user can customize it using XML ?

From what we've done so far, we can gether up all the properties that can be customized by users :

```Java
boolean _shouldShowBottomValue;
boolean _shouldShowValueLabel;
boolean _shouldShowTick;

@ColorInt int thumbToColor, thumbFromColor;
@ColorInt int selectedRangeColor, barColor;
@ColorInt int textColor;

font textFont;

int thumbSize;
int selectedRangeStrokeWidth, barStrokeWidth;
int numberOfSteps;

float initialFromValue, initialToValue;

int precision;

Listener l;
```

Now, we just need to convert these into `attrs.xml`.
> I've added some more.

```XML
<?xml version="1.0" encoding="utf-8"?>
<resources>

<declare-styleable name="TwoWaySlider">

    <attr name="showBottomValue" format="boolean"/>
    <attr name="showValueLabel" format="boolean"/>
    <attr name="showTick" format="boolean"/>

    <attr name="startValue" format="integer"/>
    <attr name="endValue" format="integer"/>
    <attr name="thumbFromValue" format="float"/>
    <attr name="thumbToValue" format="float"/>
    <attr name="precision" format="integer"/>
    <attr name="numberOfSteps" format="integer"/>

    <attr name="thumbToColor" format="reference|color"/>
    <attr name="thumbFromColor" format="reference|color"/>
    <attr name="thumbSize" format="dimension"/>

    <attr name="barColor" format="reference|color"/>
    <attr name="barStrokeWidth" format="dimension"/>

    <attr name="selectedRangeColor" format="reference|color"/>
    <attr name="selectedRangeStrokeWidth" format="dimension"/>

    <attr name="tickColor" format="reference|color"/>

    <attr name="fontName" format="string"/>
    <attr name="typeface" format="integer"/>
    <attr name="textColor" format="reference|color"/>
    <attr name="textSize" format="dimension"/>

    <attr name="listener" format="reference"/>

</declare-styleable>

</resources>
```

With these all set, we just need to retrieve these data in the `init` function.

```Java
void init(Context context, @Nullable AttributeSet attrs) {

    TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.TwoWaySlider);
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        saveAttributeDataForStyleable(context, R.styleable.TwoWaySlider, attrs, a, 0, 0);
    }

    try {

        _shouldShowBottomValue = a.getBoolean(R.styleable.TwoWaySlider_showBottomValue, true);
        _shouldShowValueLabel = a.getBoolean(R.styleable.TwoWaySlider_showValueLabel, true);
        _shouldShowTick = a.getBoolean(R.styleable.TwoWaySlider_showTick, true);

        setNumberOfSteps(a.getInt(R.styleable.TwoWaySlider_numberOfSteps, DEFAULT_NUMBER_OF_STEPS));

        setStartValue(a.getInt(R.styleable.TwoWaySlider_startValue, _startValue));
        setEndValue(a.getInt(R.styleable.TwoWaySlider_endValue, _endValue));

        setFromValue(a.getFloat(R.styleable.TwoWaySlider_thumbFromValue, (float) _startValue));
        setToValue(a.getFloat(R.styleable.TwoWaySlider_thumbToValue, (float) _endValue));

        setPrecision(a.getInt(R.styleable.TwoWaySlider_precision, _accuracy));

        if (a.hasValue(R.styleable.TwoWaySlider_thumbFromColor)) {
            ColorStateList cl = a.getColorStateList(R.styleable.TwoWaySlider_thumbFromColor);
            if (cl != null) {
                _thumbFrom.setColor(cl.getDefaultColor());
            }
        } else {
            _thumbFrom.setColor(Color.WHITE);
        }

        if (a.hasValue(R.styleable.TwoWaySlider_thumbToColor)) {
            ColorStateList cl = a.getColorStateList(R.styleable.TwoWaySlider_thumbToColor);
            if (cl != null) {
                _thumbTo.setColor(cl.getDefaultColor());
            }
        } else {
            _thumbTo.setColor(Color.BLACK);
        }

        _thumbSize = a.getDimension(R.styleable.TwoWaySlider_thumbSize, getPxFromDp(DEFAULT_THUMB_SIZE));

        if (a.hasValue(R.styleable.TwoWaySlider_barColor)) {
            ColorStateList cl = a.getColorStateList(R.styleable.TwoWaySlider_barColor);
            if (cl != null) {
                _barPaint.setColor(cl.getDefaultColor());
            }
        } else {
            _barPaint.setColor(Color.LTGRAY);
        }

        float barPaintStrokeWidth = a.getDimension(R.styleable.TwoWaySlider_barStrokeWidth, getPxFromDp(DEFAULT_BAR_STROKE_WIDTH));
        _barPaint.setStrokeWidth(barPaintStrokeWidth);

        if (a.hasValue(R.styleable.TwoWaySlider_selectedRangeColor)) {
            ColorStateList cl = a.getColorStateList(R.styleable.TwoWaySlider_selectedRangeColor);
            if (cl != null) {
                _selectedRangePaint.setColor(cl.getDefaultColor());
            }
        } else {
            _selectedRangePaint.setColor(Color.GRAY);
        }

        _selectedRangePaint.setStrokeWidth(a.getDimension(R.styleable.TwoWaySlider_selectedRangeStrokeWidth, getPxFromDp(DEFAULT_THUMB_SIZE)));

        if (a.hasValue(R.styleable.TwoWaySlider_tickColor)) {
            ColorStateList cl = a.getColorStateList(R.styleable.TwoWaySlider_tickColor);
            if (cl != null) {
                _tickPaint.setColor(cl.getDefaultColor());
            }
        }

        String fontName = a.getString(R.styleable.TwoWaySlider_fontName);

        if (fontName != null && !fontName.isEmpty()) {
            Typeface tf = Typeface.createFromAsset(context.getAssets(), "fonts/" + fontName);
            _textPaint.setTypeface(tf);
        } else if (a.hasValue(R.styleable.TwoWaySlider_typeface)) {
                int fontId = a.getResourceId(R.styleable.TwoWaySlider_typeface, -1);
                Typeface tf = ResourcesCompat.getFont(context, fontId);
                _textPaint.setTypeface(tf);
        } else {
            _textPaint.setTypeface(Typeface.DEFAULT);
        }

        if (a.hasValue(R.styleable.TwoWaySlider_textColor)) {
            ColorStateList cl = a.getColorStateList(R.styleable.TwoWaySlider_textColor);
            if (cl != null) {
                _textPaint.setColor(cl.getDefaultColor());
            }
        } else {
            _textPaint.setColor(Color.BLACK);
        }

        _textPaint.setTextSize(a.getDimension(R.styleable.TwoWaySlider_textSize, getPxFromSp(DEFAULT_TEXT_SIZE)));

        if (a.hasValue(R.styleable.TwoWaySlider_tickColor)) {
            ColorStateList cl = a.getColorStateList(R.styleable.TwoWaySlider_tickColor);
            if (cl != null) {
                _tickPaint.setColor(cl.getDefaultColor());
            }
        } else {
            _tickPaint.setColor(Color.WHITE);
        }

    } finally {
        a.recycle();
    }

    _thumbFrom.setPaintStyle(Paint.Style.FILL);
    _thumbTo.setPaintStyle(Paint.Style.FILL);

    _barPaint.setStrokeCap(Paint.Cap.ROUND);
    _barPaint.setStyle(Paint.Style.STROKE);
    _barPaint.setStrokeWidth(getPxFromDp(DEFAULT_BAR_STROKE_WIDTH));

    _selectedRangePaint.setStyle(Paint.Style.STROKE);

    _tickPaint.setStyle(Paint.Style.FILL);

    _labelPaint.setStrokeJoin(Paint.Join.BEVEL);
    _labelPaint.setStyle(Paint.Style.STROKE);
    _labelPaint.setStrokeWidth(getPxFromDp(2));
    _labelPaint.setPathEffect(new CornerPathEffect(5));
}
```

With these setup, user can now define properties in XML :
```xml
<com.jimmy.two_way_slider.TwoWaySlider
    android:id="@+id/slider"
    android:layout_width="wrap_content"
    android:layout_height="500dp"
    app:layout_constraintTop_toTopOf="parent"
    app:layout_constraintStart_toStartOf="parent"
    app:startValue="0"
    app:endValue="100"
    app:numberOfSteps="10"
    app:precision="2"
    app:thumbFromValue="18"
    app:thumbToValue="180"
    app:barColor="#f99"
    app:tickColor="#aff"
    app:barStrokeWidth="40dp"
    app:thumbFromColor="@color/purple_200"
    app:thumbToColor="#f33"
    app:selectedRangeColor="@color/design_default_color_primary"
/>
```

### Bonus : Publishing a Library
> Now that we have created our own very first custome view, perhaps it would be nice if we make it a library so other people can use it.




### Future
> By the time I finished this post, Google is now pushing for everyone to learn and use **Jetpack Compose**, a framework that is meant to get rid of the tedious code in XML and not only reduces the time to create but also reduces memory leakage from Views.

So I guess the next project is to convert this code into **Jetpack Compose**.

I will see you on the other side.

---

If you find this tutorial useful please share it with others.

Also, if you have any questions or comments regarding this topic, please do leave a message, I will get back to you ... whenever I can.

Thank you again.
