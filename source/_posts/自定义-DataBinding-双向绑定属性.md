---
title: 自定义 DataBinding 双向绑定属性
date: 2016-12-02
tags:
- Android
---
# DataBinding
DataBinding 是 Google 推出的用于完成 Java 层数据到 xml 中的绑定工具，便于完成 MVVM 架构。
[Data Binding Library](https://developer.android.com/topic/libraries/data-binding/index.html#layout_details)官方给出的向导很详细的讲述了 DataBinding 如何使用的，但是却没有提到如何自定义双向绑定属性，本文接下来将通过自定义一个View，使其能支持双向绑定的 *display* 属性来讲解自定义双向绑定属性的过程。

# 关键点
0. [InverseBindingAdapter](https://developer.android.com/reference/android/databinding/InverseBindingAdapter.html)
0. [InverseBindingMethod](https://developer.android.com/reference/android/databinding/InverseBindingAdapter.html)
0. [InverseBindingMethods](https://developer.android.com/reference/android/databinding/InverseBindingMethods.html)
0. [InverseBindingListener](https://developer.android.com/reference/android/databinding/InverseBindingListener.html)

# 第零步
在写代码之前，先分析完成一个支持双向绑定的属性需要什么。
0. 需要属性的 getter 和 setter 方法。
0. 需要属性发生变化时触发回调。


# 第一步
由于 View 默认没有办法设置 VisibilityChange 监听，所以需要简单的对 View 进行一些改造：
重写 `View#onVisibilityChanged` 方法并提供一个 `setVisibilityChangeListener` 方法。

```java
    private OnVisibilityChangeListener mListener;

    @Override
    protected void onVisibilityChanged(View changedView, int visibility) {
        super.onVisibilityChanged(changedView, visibility);
        if (mListener != null) {
            mListener.onChange();
        }
    }

    public void setVisibilityChangeListener(OnVisibilityChangeListener listener) {
        mListener = listener;
    }

    public interface OnVisibilityChangeListener {
        void onChange();
    }
```

# 第二步
使用 `BindingAdapter` 注解设置 display 属性的 setter 函数，注解参数 value 是在 xml 中对应的属性名，方法的第一个参数是需要设置属性的 view，第二个参数是需要设置的属性值。

``` java
    @BindingAdapter(value = "display")
    public static void setDisplay(CustomView view, boolean isDisplay) {
        view.setVisibility(isDisplay ? View.VISIBLE : View.INVISIBLE);
    }
```

# 第三步
用 `InverseBindingAdapter` 定义 getter 函数。

```java
    @InverseBindingAdapter(attribute = "display", event = "displayAttrChanged")
    public static boolean isDisplay(CustomView view) {
        return view.getVisibility() == View.VISIBLE;
    }
```

`attribute` 是xml中的属性名，`event` 是设置属性监听的属性名，类型是 InverseBindingListener。

# 第四步
使用 `BindingAdapter` 注解设置与第三步中 event 值相同的回调函数的 setter 函数。

```java
    @BindingAdapter(value = "displayAttrChanged")
    public static void setChangeListener(CustomView view, InverseBindingListener listener) {
        view.setVisibilityChangeListener(listener::onChange);
    }
```

# 第五步
在xml中使用 `@={}` 语法。

```xml
<CustomView
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    app:display="@={isDisplay}" />
```
于是就可以使用 isDisplay 控制 view 的显示和隐藏，并且当 view 可见状态改变时 isDisplay 值也会随之改变。