---
title: 深入探索 fitsSystemWindows
date: 2018-03-25
typora-root-url: ../../source
tags:
- Android
---
相信大家对 `fitsSystemWindows` 一定不陌生，在使用 Google 提供的 design support 库中的 CoordinatorLayout 布局时经常用到，但是又具体说不清这个东西到底有什么用！本文现在就来一探究竟，`fitsSystemWindows` 到底是干什么的。
# fitsSystemWindows 默认实现
在探究 fitsSystemWindows 之前，我们首先需要了解 window insets。简单来说，就是你期望你的 window 能占满整个屏幕并绘制画面，但是呢系统要显示状态栏，于是状态栏这个时候就会绘制在你的应用画面之上，为了不让你应用中的重要信息被状态栏遮挡，这个时候就需要 View 本身根据系统占用的绘制空间做一些处理。
``` java
// View 的 dispatchApplyWindowInsets 实现
public WindowInsets dispatchApplyWindowInsets(WindowInsets insets) {
    try {
        mPrivateFlags3 |= PFLAG3_APPLYING_INSETS;
        if (mListenerInfo != null && mListenerInfo.mOnApplyWindowInsetsListener != null) {
            return mListenerInfo.mOnApplyWindowInsetsListener.onApplyWindowInsets(this, insets);
        } else {
            return onApplyWindowInsets(insets);
        }
    } finally {
        mPrivateFlags3 &= ~PFLAG3_APPLYING_INSETS;
    }
}

// ViewGroup 的 dispatchApplyWindowInsets 实现
@Override
public WindowInsets dispatchApplyWindowInsets(WindowInsets insets) {
    insets = super.dispatchApplyWindowInsets(insets);
    if (!insets.isConsumed()) {
        final int count = getChildCount();
        for (int i = 0; i < count; i++) {
            insets = getChildAt(i).dispatchApplyWindowInsets(insets);
            if (insets.isConsumed()) {
                break;
            }
        }
    }
    return insets;
}
```
在系统占用了部分空间时，会通过 `WindowInsets` 告知应用，并且类似 Touch Event 的传递过程一般，会遍历 View 树寻找能处理该事件的 View，当 WindowInsets 被消耗后就不再向下传递。有两个办法可以自定义 WindowInsets 的处理流程，一个是通过设置 `OnApplyWindowInsetsListener`，还有一个是重写 `onApplyWindowInsets` 方法，其中设置 Listener 的处理优先级最高。View 默认提供了 `onApplyWindowInsets` 的实现，如下所示：
``` java
public WindowInsets onApplyWindowInsets(WindowInsets insets) {
    if ((mPrivateFlags3 & PFLAG3_FITTING_SYSTEM_WINDOWS) == 0) {
        // We weren't called from within a direct call to fitSystemWindows,
        // call into it as a fallback in case we're in a class that overrides it
        // and has logic to perform.
        if (fitSystemWindows(insets.getSystemWindowInsets())) {
            return insets.consumeSystemWindowInsets();
        }
    } else {
        // We were called from within a direct call to fitSystemWindows.
        if (fitSystemWindowsInt(insets.getSystemWindowInsets())) {
            return insets.consumeSystemWindowInsets();
        }
    }
    return insets;
}
```
可以看到根据标志，分别使用了不同的实现，但是其实 `fitSystemWindows` 的实现还是将请求转发给了 `fitSystemWindowsInt`:
``` java
// fitSystemWindows 重新添加了 PFLAG3_FITTING_SYSTEM_WINDOWS 后重新调用 dispatchApplyWindowInsets，最终将请求在 onApplyWindowInsets 中转发给了 fitSystemWindowsInt
protected boolean fitSystemWindows(Rect insets) {
    if ((mPrivateFlags3 & PFLAG3_APPLYING_INSETS) == 0) {
        if (insets == null) {
            // Null insets by definition have already been consumed.
            // This call cannot apply insets since there are none to apply,
            // so return false.
            return false;
        }
        // If we're not in the process of dispatching the newer apply insets call,
        // that means we're not in the compatibility path. Dispatch into the newer
        // apply insets path and take things from there.
        try {
            mPrivateFlags3 |= PFLAG3_FITTING_SYSTEM_WINDOWS;
            return dispatchApplyWindowInsets(new WindowInsets(insets)).isConsumed();
        } finally {
            mPrivateFlags3 &= ~PFLAG3_FITTING_SYSTEM_WINDOWS;
        }
    } else {
        // We're being called from the newer apply insets path.
        // Perform the standard fallback behavior.
        return fitSystemWindowsInt(insets);
    }
}
```
`onApplyWindowInsets` 所以核心实现是通过 `fitSystemWindowsInt` 进行处理。
``` java
private boolean fitSystemWindowsInt(Rect insets) {
    if ((mViewFlags & FITS_SYSTEM_WINDOWS) == FITS_SYSTEM_WINDOWS) {
        mUserPaddingStart = UNDEFINED_PADDING;
        mUserPaddingEnd = UNDEFINED_PADDING;
        Rect localInsets = sThreadLocal.get();
        if (localInsets == null) {
            localInsets = new Rect();
            sThreadLocal.set(localInsets);
        }
        boolean res = computeFitSystemWindows(insets, localInsets);
        mUserPaddingLeftInitial = localInsets.left;
        mUserPaddingRightInitial = localInsets.right;
        internalSetPadding(localInsets.left, localInsets.top,
                localInsets.right, localInsets.bottom);
        return res;
    }
    return false;
}
```
`fitSystemWindowsInt` 处理的方式很简单，就是判断本文的主角 `fitsSystemWindows` 是否设置为 `ture`，是的话就会在给 View 添加一个和 WindowInsets 值相同的 padding，并且返回值为 true，这样在 `onApplyWindowInsets` 就会将 WindowInsets 消耗掉。效果如下：

![](/images/20180325224033.jpg)

![](/images/20180325223906.jpg)

也就是说，默认情况下第一个设置 fitsSystemWindows 为 true 的 View 会在内部添加一个 padding 并将 WindowInsets 消耗，其它的 View 设置的 fitsSystemWindows 标记将会被忽略。

# 获取系统栏高度参数
在这里顺便提一下，有时候可能会需要获取系统状态栏的高度，在 Google 上一搜索，很多都是答案都是通过硬编码实现：在 xx 版本下是 xx-dp，在  xx 版本下是 xx-dp。通过浏览上面的 View 源代码，可以发现一个最通用的实现办法，即通过给根 View 设置 `OnApplyWindowInsetsListener` 实现：
``` kotlin
val rootView = findViewById(android.R.id.content)
rootView.setOnApplyWindowInsetsListener { v, insets ->
    // 通过 insets 获取系统界面的高度值等
    v.onApplyWindowInsets(insets)
}
```

# CoordinateLayout 的 fitsSystemWindows 实现
看了上文，可能会觉得不对呀，在使用 CoordinateLayout 时就算给子 View 设置 fitsSystemWindows 也会有效果的，而且必须父 View 也必须设置 fitsSystemWindows 为 true 才行。是的，这是因为 CoordinateLayout 内部修改了对 WindowInsets 的处理。
在 CoordinateLayout 的 fitsSystemWindows 值为 false 时的实现和上文是一致的，便不再多述。现在主要探究在 fitsSystemWindows 为 true 时行为是如何发生改变的。
``` java
private void setupForInsets() {
    if (Build.VERSION.SDK_INT < 21) {
        return;
    }

    if (ViewCompat.getFitsSystemWindows(this)) {
        if (mApplyWindowInsetsListener == null) {
            mApplyWindowInsetsListener =
                    new android.support.v4.view.OnApplyWindowInsetsListener() {
                        @Override
                        public WindowInsetsCompat onApplyWindowInsets(View v,
                                WindowInsetsCompat insets) {
                            return setWindowInsets(insets);
                        }
                    };
        }
        // First apply the insets listener
        ViewCompat.setOnApplyWindowInsetsListener(this, mApplyWindowInsetsListener);

        // Now set the sys ui flags to enable us to lay out in the window insets
        setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_STABLE
                | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN);
    } else {
        ViewCompat.setOnApplyWindowInsetsListener(this, null);
    }
}
```
`setupForInsets` 函数便是一切改变的开始，在 CoordinateLayout 的构造函数和调用 `setFitsSystemWindows` 时都会触发。可以看到 CoordinateLayout 在 API 大于 21 且 fitsSystemWindows 值为 true 时会通过给自身设置一个 OnApplyWindowInsetsListener 来改变行为，而没有选择通过重写 `onApplyWindowInsets` 方法实现。
接下来浏览 `setWindowInsets` 函数源码：
``` java
final WindowInsetsCompat setWindowInsets(WindowInsetsCompat insets) {
    if (!objectEquals(mLastInsets, insets)) {
        mLastInsets = insets;
        mDrawStatusBarBackground = insets != null && insets.getSystemWindowInsetTop() > 0;
        setWillNotDraw(!mDrawStatusBarBackground && getBackground() == null);

        // Now dispatch to the Behaviors
        insets = dispatchApplyWindowInsetsToBehaviors(insets);
        requestLayout();
    }
    return insets;
}
```
首先判断了 insets 和之前设置的值是否一致，在不一致时才会触发更改，是一种优化手段。`setWillNotDraw` 同理也是优化手段，不是本文的重点。在这个函数当中需要关注的有：变量 `mDrawStatusBarBackground` 和函数 `dispatchApplyWindowInsetsToBehaviors`。`mDrawStatusBarBackground` 变量决定了是否需要绘制系统状态栏的背景，这个下文会再详细说明，这里先看 `dispatchApplyWindowInsetsToBehaviors` 函数:
``` java
private WindowInsetsCompat dispatchApplyWindowInsetsToBehaviors(WindowInsetsCompat insets) {
    if (insets.isConsumed()) {
        return insets;
    }

    for (int i = 0, z = getChildCount(); i < z; i++) {
        final View child = getChildAt(i);
        if (ViewCompat.getFitsSystemWindows(child)) {
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            final Behavior b = lp.getBehavior();

            if (b != null) {
                // If the view has a behavior, let it try first
                insets = b.onApplyWindowInsets(this, child, insets);
                if (insets.isConsumed()) {
                    // If it consumed the insets, break
                    break;
                }
            }
        }
    }

    return insets;
}
```
该函数的核心就是遍历子 View，寻找 fitsSystemWindows 值为 true 且设置了 Behavior 的子 View 将 WindowInsets 消耗掉。看起来平淡无奇，但实际上暗藏玄机。只看上面的代码，会认为假如 CoordinateLayout 的 fitsSystemWindows 值为 true，子View 的 fitsSystemWindows 值为 false 的话效果与 FrameLayout 的 fitsSystemWindows 值为 false，子View 的 fitsSystemWindows 值也为 false 一致。因为如果 CoordinateLayout 子View 的 Behavior 没有设置的话，是不能消耗 WindowInsets 的，那么传递到 CoordinateLayout 的 WindowInsets 相当于没有任何的作为。但是实际效果确是下面这样的：


![](/images/20180326002906.jpg)

TextView 并没有绘制在系统状态栏背景中，而是自动绘制到了系统状态栏的下方，这是为什么呢！
其实，这是因为 CoordinateLayout 绘制了系统状态栏的背景，于是改变了一些行为，就是与上文中的变量 `mDrawStatusBarBackground` 有关。

``` java
@Override
public void onDraw(Canvas c) {
    super.onDraw(c);
    if (mDrawStatusBarBackground && mStatusBarBackground != null) {
        final int inset = mLastInsets != null ? mLastInsets.getSystemWindowInsetTop() : 0;
        if (inset > 0) {
            mStatusBarBackground.setBounds(0, 0, getWidth(), inset);
            mStatusBarBackground.draw(c);
        }
    }
}
```
在 onDraw 函数中，在 `mDrawStatusBarBackground` 为 true 并且 `mStatusBarBackground` 不等于 null 的情况下绘制了背景。但是按照 View 的绘制流程，子 View 的绘制是要在 onDraw 之后进行的，那么子 View 就会覆盖掉 CoordinateLayout 的成果，但是从上面的效果中看到，并不是这样的。那么，这意味着子 View 的位置发生了改变。能让子 View 位置发生改变，就很有可能是在 measure 和 layout 阶段做了手脚。
``` java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        ...

        int childWidthMeasureSpec = widthMeasureSpec;
        int childHeightMeasureSpec = heightMeasureSpec;
        if (applyInsets && !ViewCompat.getFitsSystemWindows(child)) {
            // We're set to handle insets but this child isn't, so we will measure the
            // child as if there are no insets
            final int horizInsets = mLastInsets.getSystemWindowInsetLeft()
                    + mLastInsets.getSystemWindowInsetRight();
            final int vertInsets = mLastInsets.getSystemWindowInsetTop()
                    + mLastInsets.getSystemWindowInsetBottom();

            childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(
                    widthSize - horizInsets, widthMode);
            childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                    heightSize - vertInsets, heightMode);
        }

        ...
}
```
在 `onMeasure` 中，假如子 View fitsSystemWindows 值为 false 的话，会修改父 View 的区域。
``` java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    final int layoutDirection = ViewCompat.getLayoutDirection(this);
    final int childCount = mDependencySortedChildren.size();
    for (int i = 0; i < childCount; i++) {
        final View child = mDependencySortedChildren.get(i);
        if (child.getVisibility() == GONE) {
            // If the child is GONE, skip...
            continue;
        }

        final LayoutParams lp = (LayoutParams) child.getLayoutParams();
        final Behavior behavior = lp.getBehavior();

        if (behavior == null || !behavior.onLayoutChild(this, child, layoutDirection)) {
            onLayoutChild(child, layoutDirection);
        }
    }
}

public void onLayoutChild(View child, int layoutDirection) {
    final LayoutParams lp = (LayoutParams) child.getLayoutParams();
    if (lp.checkAnchorChanged()) {
        throw new IllegalStateException("An anchor may not be changed after CoordinatorLayout"
                + " measurement begins before layout is complete.");
    }
    if (lp.mAnchorView != null) {
        layoutChildWithAnchor(child, lp.mAnchorView, layoutDirection);
    } else if (lp.keyline >= 0) {
        layoutChildWithKeyline(child, lp.keyline, layoutDirection);
    } else {
        layoutChild(child, layoutDirection);
    }
}

private void layoutChild(View child, int layoutDirection) {
    final LayoutParams lp = (LayoutParams) child.getLayoutParams();
    final Rect parent = acquireTempRect();
    parent.set(getPaddingLeft() + lp.leftMargin,
            getPaddingTop() + lp.topMargin,
            getWidth() - getPaddingRight() - lp.rightMargin,
            getHeight() - getPaddingBottom() - lp.bottomMargin);

    if (mLastInsets != null && ViewCompat.getFitsSystemWindows(this)
            && !ViewCompat.getFitsSystemWindows(child)) {
        // If we're set to handle insets but this child isn't, then it has been measured as
        // if there are no insets. We need to lay it out to match.
        parent.left += mLastInsets.getSystemWindowInsetLeft();
        parent.top += mLastInsets.getSystemWindowInsetTop();
        parent.right -= mLastInsets.getSystemWindowInsetRight();
        parent.bottom -= mLastInsets.getSystemWindowInsetBottom();
    }

    final Rect out = acquireTempRect();
    GravityCompat.apply(resolveGravity(lp.gravity), child.getMeasuredWidth(),
            child.getMeasuredHeight(), parent, out, layoutDirection);
    child.layout(out.left, out.top, out.right, out.bottom);

    releaseTempRect(parent);
    releaseTempRect(out);
}
```
在 `layoutChild` 函数中同理，最终使得子 View 的位置发生偏移。

# AppBarLayout 的 fitsSystemWindows 实现
AppBarLayout 作为经常配合 CoordinateLayout 一起使用的视图组件，也拥有和普通 View 不一样的针对 fitsSystemWindows 的实现。在该组件的构造函数中就设置了 OnApplyWindowInsetsListener，无论 fitsSystemWindows 的值是否为 true。
``` java
public AppBarLayout(Context context, AttributeSet attrs) {
...

ViewCompat.setOnApplyWindowInsetsListener(
    this,
    new android.support.v4.view.OnApplyWindowInsetsListener() {
        @Override
        public WindowInsetsCompat onApplyWindowInsets(View v, WindowInsetsCompat insets) {
        return onWindowInsetChanged(insets);
        }
    });
}
```
该函数通过调用 `onWindowInsetChanged` 处理 WindowInsets。
``` java
  WindowInsetsCompat onWindowInsetChanged(final WindowInsetsCompat insets) {
    WindowInsetsCompat newInsets = null;

    if (ViewCompat.getFitsSystemWindows(this)) {
      // If we're set to fit system windows, keep the insets
      newInsets = insets;
    }

    // If our insets have changed, keep them and invalidate the scroll ranges...
    if (!ObjectsCompat.equals(lastInsets, newInsets)) {
      lastInsets = newInsets;
      invalidateScrollRanges();
    }

    return insets;
  }

  private void invalidateScrollRanges() {
    // Invalidate the scroll ranges
    totalScrollRange = INVALID_SCROLL_RANGE;
    downPreScrollRange = INVALID_SCROLL_RANGE;
    downScrollRange = INVALID_SCROLL_RANGE;
  }
```
AppBarLayout 对于 fitsSystemWindows 的处理比较简单，就是在 fitsSystemWindows 为 true 的时候计算滑动范围会考虑系统状态栏的高度，需要注意的是 AppBarLayout 无论如何都不会消耗 WindowInsets。
具体效果就通过图片来说明，下图是没有进行任何滑动的样子，可以看到 AppBarLayout 中的 TextView 完全的显示出来了：

![](/images/20180326115217.jpg)

在 fitsSystemWindows 值为 false 时，通过滑动屏幕可以让 TextView 完全退出，不在屏幕中显示：

![](/images/20180326115234.jpg)

在 fitsSystemWindows 值为 true 时，通过滑动屏幕，TextView 无论如何都会保留和系统状态相同高度的最小高度：

![](/images/20180326115249.jpg)

# CollapsingToolbarLayout 的 fitsSystemWindows 实现
CollapsingToolbarLayout 作为经常配合 AppBarLayout 使用的视图控件，也拥有独特的针对 fitsSystemWindows 的处理。和 AppBarLayout 很像，也是在构造函数中就设置了 OnApplyWindowInsetsListener：
``` java
public CollapsingToolbarLayout(Context context, AttributeSet attrs, int defStyleAttr) {
...

ViewCompat.setOnApplyWindowInsetsListener(
    this,
    new android.support.v4.view.OnApplyWindowInsetsListener() {
        @Override
        public WindowInsetsCompat onApplyWindowInsets(View v, WindowInsetsCompat insets) {
        return onWindowInsetChanged(insets);
        }
    });
}
```
同样核心也是在 `onWindowInsetChanged` 实现：
``` java
WindowInsetsCompat onWindowInsetChanged(final WindowInsetsCompat insets) {
WindowInsetsCompat newInsets = null;

if (ViewCompat.getFitsSystemWindows(this)) {
    // If we're set to fit system windows, keep the insets
    newInsets = insets;
}

// If our insets have changed, keep them and invalidate the scroll ranges...
if (!ObjectsCompat.equals(lastInsets, newInsets)) {
    lastInsets = newInsets;
    requestLayout();
}

// Consume the insets. This is done so that child views with fitSystemWindows=true do not
// get the default padding functionality from View
return insets.consumeSystemWindowInsets();
}
```
实现很简单明了，在 fitSystemWindows 值为 true 时会记录当前的 WindowInsets 值，并请求重新进行布局，那么在 measure、layout 阶段肯定就针对 WindowInsets 做了处理，需要注意的是和 AppBarLayout 不同，无论如何 CollapsingToolbarLayout 都会消耗掉 WindowInsets。
``` java
  @Override
  protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    ensureToolbar();
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);

    final int mode = MeasureSpec.getMode(heightMeasureSpec);
    final int topInset = lastInsets != null ? lastInsets.getSystemWindowInsetTop() : 0;
    if (mode == MeasureSpec.UNSPECIFIED && topInset > 0) {
      // If we have a top inset and we're set to wrap_content height we need to make sure
      // we add the top inset to our height, therefore we re-measure
      heightMeasureSpec =
          MeasureSpec.makeMeasureSpec(getMeasuredHeight() + topInset, MeasureSpec.EXACTLY);
      super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    }
  }
```
在 measure 阶段，如果 CollapsingToolbarLayout 高度被设置为 wrap_content，那么会在正常测量得到的高度上再加上系统状态栏的高度以能容纳绘制系统状态栏背景的需求。

``` java
  @Override
  protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    super.onLayout(changed, left, top, right, bottom);

    if (lastInsets != null) {
      // Shift down any views which are not set to fit system windows
      final int insetTop = lastInsets.getSystemWindowInsetTop();
      for (int i = 0, z = getChildCount(); i < z; i++) {
        final View child = getChildAt(i);
        if (!ViewCompat.getFitsSystemWindows(child)) {
          if (child.getTop() < insetTop) {
            // If the child isn't set to fit system windows but is drawing within
            // the inset offset it down
            ViewCompat.offsetTopAndBottom(child, insetTop);
          }
        }
      }
    }
    
    ...
  }
```
在 layout 阶段，如果子 View 没有设置 fitSystemWindows 值为 true 的话就会对子 View 进行一个位移，使得该子 View 绘制在系统状态栏的下方，不会被系统状态栏遮挡，具体效果还是用图片来进行说明。
fitSystemWindows 值为 false 时效果：

![](/images/20180326143044.jpg)

fitSystemWindows 值为 true 时效果：

![](/images/20180326143032.jpg)

至此，便分析完了常见的用到 fitSystemWindows 的 View 的场景。
# 参考资料
[droidcon NYC 2017 - Becoming a master window fitter](https://www.youtube.com/watch?v=_mGDMVRO3iE)