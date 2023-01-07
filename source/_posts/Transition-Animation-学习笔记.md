---
title: Transition Animation 学习笔记
date: 2017-02-23
tags:
- Android
- Animation
---
# Android中的动画
Android框架为开发者提供了三种类型动画机制：
* Property Animation - 属性动画
* View Animation - 补间动画
* Drawable Animation - 帧动画

# Transition Animation
在新版本中，引入了一种新的动画机制--Transition Animation（过渡动画）。该动画本质上是对于属性动画的一种扩展。该动画有以下特点：
* 作用在ViewGroup级别，对一个视图层次中的所有view造成效果。
* 预定义的动画效果针对视图的渐近渐出、位移、大小变化。
* 针对两个场景的切换而制造中间动画效果。

但是过渡动画也有着一些本身的限制：
* 对SurfaceView及TextureView支持不完美。
* 不支持ListView等继承自AdapterView的控件。（用RecyclerView替代ListView可解决问题）
* 当TextView设置有文字时，改变大小的过渡动画会导致文字显示异常。

# Scene
因为过渡动画是针对两个场景之间切换过程中所做的动画效果，所以在做动画之前，需要确定好开始和结束时的视图层次结构，称为Scene。一般情况下只需要提供动画结束后的Scene，Scene的创建很简单，代码如下：
``` java
// 确定Scene的根布局
ViewGroup sceneRoot = (ViewGroup) findViewById(R.id.scene_root);

// 根据scene.xml布局文件创建Scene
Scene scene = Scene.getSceneForLayout(sceneRoot, R.layout.scene, this);
```

# TransitionManager
定义好结束Scene之后，就可以用TransitionManager开始过渡动画了。
``` java
Fade fade = new Fade();
TransitionManager.go(scene, fade);
```
调用TransitionManager.go()后，框架会把上一次过渡动画的结束Scene当作此次过渡动画的开始Scene，如果没有上一次过渡动画，那么就根据当前的视图作为开始Scene。然后就会根据开始Scene和结束Scene中的视图差异进行动画（根据view id确定前后是否是同一个view）。如上代码会在有view添加进sceneRoot和有view从sceneRoot中移除时有渐近渐出效果。
系统已经预定义了几个常用的过渡动画效果：
* AutoTransition：根据渐出、移动、改变大小、渐进的顺序做动画。
* Fade：渐近渐出动画。
* ChangeBounds：改变大小，位移时过渡动画。

# 自定义Transition
自定义Transition只需要继承Transition类并重写其中的三个抽象方法就即可。
``` java
public class CustomTransition extends Transition {
    // 推荐命名格式：packageName:transitionName:attrName
    private static final String PROPNAME_BACKGROUND =
            "com.example.android.customtransition:CustomTransition:background";

    @Override
    public void captureStartValues(TransitionValues values) {
        captureValues(transitionValues);
    }

    @Override
    public void captureEndValues(TransitionValues values) {}

    private void captureValues(TransitionValues transitionValues) {
        View view = transitionValues.view;
        transitionValues.values.put(PROPNAME_BACKGROUND, view.getBackground());
    }

    @Override
    public Animator createAnimator(ViewGroup sceneRoot,
                                   TransitionValues startValues,
                                   TransitionValues endValues) {}
}
```
captureStartValues和captureEndValues方法分别用于记录开始动画时view状态和结束动画时的view状态，然后在createAnimator中根据开始状态和结束状态自定义动画效果。当view被移除时，endValues将为null，view被添加时，startValues为null。

# 在Fragment和Activity中应用过渡动画
上面的过渡动画应用的例子是基于一个ViewGroup的，在实际的开发中，一般需要将视图模块化，用Fragment和Activity封装不同功能的模块。框架本身也提供了对两个Fragment或Activity之间的切换使用过渡动画的功能，只不过API调用和在ViewGroup上使用不同。

先定义布局文件：
``` xml
<!--旧布局-->
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <ImageView
        android:id="@+id/image"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:src="..."
        android:transitionName="@string/share_element" />

    <Button
        android:id="@+id/btn"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="@string/show_big_image" />
</LinearLayout>


<!--新布局-->
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <ImageView
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:src="..."
        android:transitionName="@string/share_element" />
</FrameLayout>
```
在这里，使用了transitionName来确认前后布局文件中的相同元素（也称为共享元素）。
* Fragment之间切换的逻辑代码
``` java
View image = findViewById(R.id.image);
getSupportFragmentManager()
            .beginTransaction()
            .replace(...)
            .addSharedElement(image, getString(R.string.share_element))
            .commit();
```

* Activity之间切换的逻辑代码
``` java
View image = findViewById(R.id.image);
Intent intent = new Intent(...);
Bundle option = ActivityOptionsCompat
            .makeSceneTransitionAnimation(activity, image, getString(R.string.share_element))
            .toBundle();
startActivity(intent, option);
```

# 总结
过渡动画框架提供了一个更高层次的抽象，让开发者能专注于两个场景中的视图差异来定制动画效果，但实际上的动画效果还是依赖属性动画来描述的。

# 参考
[Animating Views Using Scenes and Transitions](https://developer.android.google.cn/training/transitions/index.html)