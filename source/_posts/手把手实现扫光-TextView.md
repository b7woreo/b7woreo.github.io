---
title: 手把手实现扫光 TextView
date: 2018-06-24
typora-root-url: ../../source
tags:
- Android
---
最近发现很多 App 在 TextView 上使用了扫光特效，以吸引用户眼球。刚好最近也遇到了同样的需求，本文记录如何从零开始写一个带扫光特效的 TextView。

# 扫光原理
扫光的效果其实就是在要显示的内容上改变部分内容的颜色。

![](/images/20180624214906.png)

图中的就是在黑色的文字中，有部分文字的颜色改成了渐变透明的白色，辅以不断移动被改变的颜色的位置，便形成了扫光的动画。

# 实现
## 图层的叠加
扫光特效的核心就是改变部分区域内文字的颜色，而文字外的地方就不需要变化。所以想简单通过叠加两个 View ，一个显示文字，一个显示前景来实现是不可行的。需要通过在 Canvas 的操作中设置 _PorterDuff.Mode_ 来实现图层叠加。

[PorterDuff.Mode 文档](https://developer.android.com/reference/android/graphics/PorterDuff.Mode)

在官方的文档中给出了各种图层叠加模式下的图解，通过文档可知，_SRC_IN_模式正是我们需要的方案。为了修改图层叠加模式，需要用到 Paint，于是在 TextView 的构造函数中初始化一个 Paint 并设置图层叠加模式，专门用于绘制变色的文字。
``` java
    paint = new Paint(Paint.ANTI_ALIAS_FLAG);
    paint.setXfermode(new PorterDuffXfermode(Mode.SRC_IN));
```
需要注意的是，由于我们叠加的视图是带有透明色的，在叠加带有透明色的图层时需要开启离屏缓冲，不然会出现奇怪的问题。所以这里我们还需要在构造函数中添加如下代码开启硬件加速的离屏缓冲层
``` java
    setLayerType(LAYER_TYPE_HARDWARE, null);
```
## 变色的文字
一般情况下，变色的文字的颜色是渐变的，这里可以通过给 Paint 设置各种 Shader 来实现。例如上图中效果是利用 _LinearGradient_ 实现线性渐变色的文字，代码如下：
``` java
    LinearGradient gradient = new LinearGradient(
        0, 0,
        gradientWidth, 0,
        new int[]{0xffffffff, 0x72ffffff, 0xffffff},
        null,
        TileMode.CLAMP
    );
    paint.setShader(gradient);
```
## 绘制
通过查看 TextView 的源码可以得知，在 `onDraw` 方法中 TextView 完成了文字的绘制，所以我们也只需要重写 `onDraw` 方法，并且在 super 执行完成之后叠加上我们的自己图层即可。
``` java
  @Override
  protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);

    int count = canvas.save();
    try {
      rect.set(0, 0, gradientWidth, canvas.getHeight());
      canvas.drawRect(rect, paint);
    } finally {
      canvas.restoreToCount(count);
    }
  }
```
其中 rect 用于限制绘制叠加图层的范围，并不是整个 Canvas 都需要进行绘制。
## 动起来
最简单的动画实现方案便是通过动画时间，计算扫光色块的位置，通过偏移 Canvas 在不同的位置绘制变色效果即可。
``` java
  @Override
  protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);

    int count = canvas.save();
    try {
      canvas.translate(calculateTranslate(), 0);
      rect.set(0, 0, gradientWidth, canvas.getHeight());
      canvas.drawRect(rect, paint);
    } finally {
      canvas.restoreToCount(count);
    }

    invalidate();
  }

  private int calculateTranslate() {
    long dTime = AnimationUtils.currentAnimationTimeMillis() - startTime;
    long animatorTime = dTime % (ANIMATOR_DURATION + ANIMATOR_IDLE);
    float progress = Math.min(1f, animatorTime / (float) ANIMATOR_DURATION);
    return (int) (progress * (getWidth() + gradientWidth) - gradientWidth);
  }
```
由于 View 只有在失效的时候才会刷新，所以需要在每一次 draw 的同时标记其失效，使得再下一帧获得绘制新视图的机会。

# 改进
在上一节中已经完整实现了扫光的效果，不过光实现效果往往还不够，我们还要改进代码，使得代码易于使用与复用。这里先总结一下上面的代码出现的问题：
* Shader 是写死在代码里面，不可变
* 动画的时间、插值算法都是固定的，不可变
* 通过在 onDraw 中调用 invalidate 使得在扫光暂停时也需要不停的刷新视图，浪费计算资源和电量

## 完善动画
其实系统中 Animator 相关类已经给动画相关的实现一个很好的抽象，包括插值器等。我们应该尽可能的复用已有的代码，特别是基础类库中有的代码。所以内部动画的实现可以通过 ObjectAnimator 来实现。

``` java
  @Override
  protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    super.onLayout(changed, left, top, right, bottom);
    launchAnimator();
  }

  public void setStartDelay(long startDelay) {
    this.startDelay = startDelay;
    launchAnimator();
  }

  public void setDuration(long duration) {
    this.duration = duration;
    launchAnimator();
  }

  public void setInterpolator(Interpolator interpolator) {
    this.interpolator = interpolator;
    launchAnimator();
  }

  private void launchAnimator() {
  if (animator != null) {
    animator.cancel();
  }

  final Property<LightSweepTextView, Integer> property = new Property<LightSweepTextView, Integer>(
      Integer.class, "canvasTranslate") {
    @Override
    public void set(LightSweepTextView object, Integer value) {
      if (value == canvasTranslate) {
        return;
      }

      canvasTranslate = value;
      invalidate();
    }

    @Override
    public Integer get(LightSweepTextView object) {
      return canvasTranslate;
    }
  };

  final AnimatorListener listener = new AnimatorListenerAdapter() {
    @Override
    public void onAnimationEnd(Animator animation) {
      post(() -> animator.start());
    }
  };

  animator = ObjectAnimator.ofInt(this, property, -shaderWidth, getWidth());
  animator.setInterpolator(interpolator);
  animator.setDuration(duration);
  animator.setStartDelay(startDelay);
  animator.addListener(listener);
  animator.start();
}
```
动画主要由动画持续时间、动画开始延迟、动画的插值器描述完整的动画过程，通过自定义 Property，使得动画不断的修改 canvasTranslate 属性来移动扫光的色块位置，并且在设置 canvasTranslate 属性时需要显示调用 invalidate 函数刷新视图，为了避免不必要的视图刷新，在赋值前先进行判断，如果值没有改变，就放弃刷新。在 onLayout 时启动动画，并且在每次修改与动画相关的属性时，重新使用新的属性值启动动画。由于 Animator 的 repeat mode 在重复播放动画时会忽略开始延时，所以通过添加 AnimatorListener 在动画结束时手动重新开始动画，以达到无限循环的效果。需要注意的是，开始动画的调用需要 post 到队列中进行，因为如果 startDelay 设置为 0，在 onAnimationEnd 中直接调用 animator.start() 不会有任何效果。

## 修改 onDraw
``` java
  @Override
  protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);

    int count = canvas.save();
    try {
      canvas.translate(canvasTranslate, 0);
      rect.set(0, 0, shaderWidth, canvas.getHeight());
      canvas.drawRect(rect, paint);
    } finally {
      canvas.restoreToCount(count);
    }
  }
```
因为通过 ObjectAnimator 实现动画，所以新增一个 canvasTranslate 字段用于描述绘制扫光色块时的 canvas 的偏移量。

## 通过外部设置 Shader
``` java
  public void setShader(Shader shader, int shaderWidth) {
    paint.setShader(shader);
    this.shaderWidth = shaderWidth;
    launchAnimator();
  }
```
添加一个 setter 用于设置 shadow 样式。

## 代码
下面放出完整的代码实现
``` java
public class LightSweepTextView extends AppCompatTextView {

  private int shaderWidth;
  private Rect rect;
  private Paint paint;

  private Interpolator interpolator = new LinearInterpolator();
  private long duration = 2000;
  private long startDelay = 2500;

  private ObjectAnimator animator;
  private int canvasTranslate = Integer.MIN_VALUE;

  public LightSweepTextView(Context context) {
    super(context);
    initialize();
  }

  public LightSweepTextView(Context context, AttributeSet attrs) {
    super(context, attrs);
    initialize();
  }

  public LightSweepTextView(Context context, AttributeSet attrs, int defStyleAttr) {
    super(context, attrs, defStyleAttr);
    initialize();
  }

  private void initialize() {
    setLayerType(LAYER_TYPE_HARDWARE, null);

    rect = new Rect();
    paint = new Paint(Paint.ANTI_ALIAS_FLAG);
    paint.setXfermode(new PorterDuffXfermode(Mode.SRC_IN));
  }

  private void launchAnimator() {
    if (animator != null) {
      animator.cancel();
    }

    final Property<LightSweepTextView, Integer> property = new Property<LightSweepTextView, Integer>(
        Integer.class, "canvasTranslate") {
      @Override
      public void set(LightSweepTextView object, Integer value) {
        if (value == canvasTranslate) {
          return;
        }

        canvasTranslate = value;
        invalidate();
      }

      @Override
      public Integer get(LightSweepTextView object) {
        return canvasTranslate;
      }
    };

    final AnimatorListener listener = new AnimatorListenerAdapter() {
      @Override
      public void onAnimationEnd(Animator animation) {
        post(() -> animator.start());
      }
    };

    animator = ObjectAnimator.ofInt(this, property, -shaderWidth, getWidth());
    animator.setInterpolator(interpolator);
    animator.setDuration(duration);
    animator.setStartDelay(startDelay);
    animator.addListener(listener);
    animator.start();
  }

  public void setShader(Shader shader, int shaderWidth) {
    paint.setShader(shader);
    this.shaderWidth = shaderWidth;
    launchAnimator();
  }

  public void setInterpolator(Interpolator interpolator) {
    this.interpolator = interpolator;
    launchAnimator();
  }

  public void setStartDelay(long startDelay) {
    this.startDelay = startDelay;
    launchAnimator();
  }

  public void setDuration(long duration) {
    this.duration = duration;
    launchAnimator();
  }

  @Override
  protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    super.onLayout(changed, left, top, right, bottom);
    launchAnimator();
  }

  @Override
  protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);

    int count = canvas.save();
    try {
      canvas.translate(canvasTranslate, 0);
      rect.set(0, 0, shaderWidth, canvas.getHeight());
      canvas.drawRect(rect, paint);
    } finally {
      canvas.restoreToCount(count);
    }
  }
}
```