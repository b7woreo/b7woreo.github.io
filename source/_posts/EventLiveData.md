---
title: EventLiveData
date: 2018-08-12
tags:
- Android
---
Google 现在大力推广 arch 架构，其中提供了 LiveData 和 ViewModel 的通用抽象来连接 Model 层与 View 层的代码。LiveData 具有感知生命周期的能力，使得编写 ViewModel 的代码时候可以忽视生命周期变化带来的影响，但是因为 LiveData 在 observer 每次从不活跃状态进入活跃状态时都会确保 observer 收到了最近一次的值，这个特性使得 LiveData 适合表达 View 的最终状态，但不适合表达 View 变化过程中间的一次性状态。例如要显示当前的用户余额，用 LiveData 来表达很方便，但是如果想要表达用户余额发生变化时，触发一个过渡动画，这就比较麻烦了。

于是，就希望有这么一个类同时拥有 LiveData 和 Event 的特质：

0. 和 LiveData 拥有相同的接口，拥有生命周期的感知能力，只在 View 活跃状态传递值，通过 ViewModel 对外统一暴露 LiveData 作为接口，隐藏实现。
0. 和 Event 一样是一次性的，会被消耗的，observer 不会收到已经被消耗过的值。

为了实现以上功能，首先就先看看 LiveData 内部的原理。

# LiveData 原理

在 LiveData 中主要通过调用以下两个函数来观察值的变化：

``` java
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    owner.getLifecycle().addObserver(wrapper);
}

@MainThread
public void observeForever(@NonNull Observer<T> observer) {
    AlwaysActiveObserver wrapper = new AlwaysActiveObserver(observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && existing instanceof LiveData.LifecycleBoundObserver) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    wrapper.activeStateChanged(true);
}
```

其中的原理就是实现了 ObserverWrapper 用于控制 observer 的状态，在每次活跃状态变化或者接受到新的值时控制 observer 是否应该接受到新的值。
给 LiveData 设置一个新值，通过 setValue 函数实现( postValue 最终也是要调用 setValue 函数，只不过要通过 handle 切换到主线程执行 )：

``` java
@MainThread
protected void setValue(T value) {
    assertMainThread("setValue");
    mVersion++;
    mData = value;
    dispatchingValue(null);
}
```

其中的重点是，内部维护一个 `mVersion` 变量记录值的版本号，在每次调用 setValue 时都会跟新该值，而在 ObserverWrapper 中维护一个 `mLastVersion` 变量记录了该 observer 最近一次接受的值的版本号。在 dispatchingValue 函数中会比较 `mVersion` 和 `mLastVersion`，如果 `mLastVersion` 大于等于 `mVersion` 就说明该 observer 已经接收过最新的值了，就不会再此向该 observer 传递值。

# 实现 EventLiveData

通过继承 LiveData 可以即可满足 LiveData 的特质，现在我们需要解决的是如何使得值像 Event 一样。但是想要方便的值的事件化就没有这么容易了。因为实际上 observer 会不会接收到一个值主要由两个因素控制：

0. observer 是不是处于激活状态（由 lifecycle owner 控制）
0. 上一次接收到的值是否是最新的（mLastVersion 是一个私有变量）

上述两个因素阻止我们通过优雅的方式实现，不过在 `LiveData#observe` 方法中通过 Wrapper 给 observer 加上生命周期控制的能力给了我一个启示，其实我们也可以给 observer 加一个 wrapper，通过 wrapper observer 去订阅 LiveData 的更新，在 wrapper 内部控制是否需要把值传递给实际的 observer。并且因为 setValue 是同步方法，在每次调用 setValue 时，先发送需要传递的值，再发送一个信号值，在 wrapper 中每次收到这个信号值都会被拦截，这样就能实现值是一次性的特性了。

不过这样的实现有两个缺点：

0. 最合适的信号值是 `null`，使得 EventLiveData 不能用 null 当作一个事件传递
0. 而且因为每次事件发送完之后值一定为 null，那么 `getValue` 函数就变得不可用

不过这两个缺点是可以接受的。在 RxJava 中同样就不支持传递 null 值，有很多中办法弥补不能传递 null，例如使用一个全局唯一的实例用来表示 null event。`getValue` 函数变得不可用理论上影响更加小，因为其实根本没有必要去知道一个已经被消耗掉的事件是什么，事件应该只能被用于观察。

上面的思路实现的 EventLiveData 还有一个不足就是事件在 LiveData 没有任何活跃 observer 时也会被消耗，有时候我们希望事件至少被一个 observer 消耗（也叫做 粘滞事件），这其实不难，只需要在 setValue 时判断当前 LiveData 是否拥有活跃的 observer，如果没有的话就先暂时保存在一个变量中，待 LiveData 拥有活跃的 observer 时会调用 `onActive` 函数，这时再真正的去调用 setValue 函数即可。

下面放出整个实现的源码：

``` java
public class EventLiveData<T> extends LiveData<T> {

  private final Map<Observer<T>, Observer<T>> observers = new WeakHashMap<>();

  private final boolean sticky;
  private final AtomicReference<T> paddingValue = new AtomicReference<>(null);

  public EventLiveData() {
    this(false);
  }

  public EventLiveData(boolean sticky) {
    this.sticky = sticky;
  }

  @Nullable
  @Override
  public T getValue() {
    throw new RuntimeException("event can only be observed");
  }

  @Override
  public void setValue(T value) {
    assertMainThread();

    if (value == null) {
      throw new NullPointerException("event can not be null");
    }

    if (interceptValue(value)) {
      return;
    }

    super.setValue(value);
    super.setValue(null);
  }

  @Override
  public void postValue(T value) {
    if (value == null) {
      throw new NullPointerException("event can not be null");
    }

    super.postValue(value);
  }

  private boolean interceptValue(T value) {
    if (sticky && !hasActiveObservers()) {
      paddingValue.set(value);
      return true;
    }

    return false;
  }

  @Override
  protected void onActive() {
    super.onActive();
    T value = paddingValue.getAndSet(null);
    if (value != null) {
      setValue(value);
    }
  }

  @Override
  public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
    assertMainThread();

    Observer<T> wrapper = observers.get(observer);
    if (wrapper == null) {
      wrapper = new WrapperObserver<>(observer);
      observers.put(observer, wrapper);
    }
    super.observe(owner, wrapper);
  }

  @Override
  public void observeForever(@NonNull Observer<T> observer) {
    assertMainThread();

    Observer<T> wrapper = observers.get(observer);
    if (wrapper == null) {
      wrapper = new WrapperObserver<>(observer);
      observers.put(observer, wrapper);
    }
    super.observeForever(wrapper);
  }

  @Override
  public void removeObserver(@NonNull Observer<T> observer) {
    assertMainThread();

    Observer<T> wrapper = observers.get(observer);
    Observer<T> removed = wrapper == null ? observer : wrapper;
    super.removeObserver(removed);
  }

  private static class WrapperObserver<T> implements Observer<T> {

    private final Observer<T> observer;

    WrapperObserver(Observer<T> observer) {
      this.observer = observer;
    }

    @Override
    public void onChanged(@Nullable T value) {
      if (value == null) {
        return;
      }

      observer.onChanged(value);
    }
  }

  private static void assertMainThread() {
    if (Thread.currentThread() != Looper.getMainLooper().getThread()) {
      throw new RuntimeException("must call on main thread");
    }
  }
}
```

上面的代码还有一个需要注意的地方是 `removeObserver` 函数的实现，该函数的入参有三种可能：

0. 之前通过 observeForever 注册的 observer
0. 在生命周期进入 DESTORY 状态时是 wrapper observer
0. 用户传入的不无效的 observer

所以我们先通过查表，看传入的 observer 是否有对应的 wrapper observer，如果有那么应该被移除的是 wrapper observer，否则直接将入参的 observer 传递给父方法做处理。

至此，完整的 EventLiveData 已经实现了，完整代码包括单元测试在：[event-livedata](https://github.com/chrnie233/event-livedata)