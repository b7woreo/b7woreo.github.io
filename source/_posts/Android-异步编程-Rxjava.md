---
title: Android 异步编程-RxJava
date: 2020-01-27
tags:
- Android
- RxJava
---

在 {% post_link Android-异步编程-回调 %} 中我们已经介绍了如何通过回调设计异步接口。相比起使用回调设计异步接口，RxJava 完成了一些异步通用逻辑的抽象，使用它解决异步问题能让我们更专注于业务逻辑本身。

# 设计异步接口

同步的接口转换成通过 RxJava 来实现的异步接口有一些通用的原则，并且 RxJava 库本身已经提供了一些函数用于帮助我们将原本同步设计的接口迁移至用 RxJava 实现的异步接口：

* 没有返回值的函数变成返回值为 Completable 的函数，例：

  ``` java
  // 同步接口
  void foo(){
    ...    
  }
  
  // 异步接口
  Completable fooAsync(){
    return Completable.fromAction(this::foo) 
      .subscribeOn(...) // 指定 foo() 的执行线程
  }
  ```

* 有返回值的函数变成返回值为 Single<T> 的函数，例：

  ``` java
  // 同步接口
  String getFoo() {
  	...
  }
  
  // 异步接口
  Single<String> getFooAsync() {
    return Single.fromCallable(this::getFoo)
      .subscribeOn(...)  // 指定 getFoo() 的执行线程
  }
  ```

* 注册值变化监听器、反注册值变化监听器的组合可以变成返回值为 Observable<T> 的函数，例：

  ``` java
  // 同步接口
  void addOnFooChangeListener(Listener listener) {
    ...
  }
  
  void removeOnFooChangeListener(Listener listener) {
    ...
  }
  
  // 异步接口
  Observable<String> observeFoo() {
    return Observable.create(emitter -> {
      // 值变化时调用 onNext 将最新值向下游传递
      Listener l = emitter::onNext;  
      // 当 Disposable.dispose() 被调用时时移除监听器
      emitter.setCancellable(() -> removeOnFooChangeListener(l)); 
      // 开始监听
      addOnFooChangeListener(l); 
    });
  }
  ```

上面的示例中有一个地方值得注意，在 `fooAsync()` 和 `getFooAsync()` 中我都通过调用 `subscribeOn()` 切换了线程，而在 `observeFoo()` 中我却没有这么做，是因为作为实现方我明确的知道调用 `foo()` 和 `getFoo()` 是耗时操作，需要确保运行在一个更为理想的环境中（如果是计算密集行操作可以使用 `Schedulers.computation()`，如果是 IO 密集型操作可以使用 `Schedulers.io()`）；而 `addOnFooChangeListener()` 并不耗时，切换执行线程并不是一个必要操作。

就如最开始所说的，实现方能非常明确的知道什么操作耗时什么操作不耗时，耗时的操作是什么原因引起的，能用最好的决策完成执行线程的调度。而接口的使用者也能非常方便的通过 `observeOn()` 函数切换自身想要的线程中执行。实现方和使用方完全解耦，不需要关心各自需要在什么样的线程中执行代码。

# 常用的操作符

常用的异步逻辑已经被 RxJava 封装到了操作符中，所以必须要对操作符有一个大概的了解才能比较好的理解用 RxJava 写出的代码，学习操作符的方式主要有以下两种：

1. 阅读 [RxJava入门指南· ReactiveX文档中文翻译](https://mcxiaoke.gitbooks.io/rxdocs/content/topics/Getting-Started.html)，对所有的操作符有一个大概的认识。
2. 在代码中遇到有不知道的操作符，在 AndroidStudio 中利用 Quick Documentation 功能可以快速查看带有示意图的操作符文档。

下面将介绍一些常用的操作符以及给出一些示例。

## 线程池

在 RxJava 中，将函数执行的线程环境抽象为 `Scheduler` 接口，并且默认已经提供了一套默认的实现。在 Android 开发中常用的实现有三个：

* `AndroidSchedulers.mainThread()`：在主线程中执行
* `Schedulers.computation()`：内部维护了一个固定线程数的线程池，用于执行计算密集型任务
* `Schedulers.io()`：内部维护了一个可复用但没有上限的线程池，用于执行阻塞式 IO 任务

## 切换线程

* `subscribeOn`：设置上游订阅逻辑的执行线程，例：

  ``` java
  /*
   * 除了 create 操作符外，fromAction、fromCallable、fromRunnable 
   * 这些操作符入参的执行线程也受 subscribeOn 影响
   */
  Completable.create(emitter -> {
    System.out.println(Thread.currentThread().getName());
    emitter.onComplete();
  })
    .subscribeOn(Schedulers.io())
    .doOnSubscribe(d -> System.out.println(Thread.currentThread().getName()))
    .subscribeOn(Schedulers.computation())
    .doOnSubscribe(d -> System.out.println(Thread.currentThread().getName()))
    .subscribe();
  
  /*
   * 执行结果：
   * main
   * RxComputationThreadPool-1
   * RxCachedThreadScheduler-1
   */
  ```

* `unsubscribeOn`：设置上游取消订阅逻辑的执行线程，例：

  ``` java
  Disposable disposable = Completable.create(emitter ->
    emitter.setCancellable(() ->
      System.out.println(Thread.currentThread().getName())
    )
  )
    .unsubscribeOn(Schedulers.io())
    .doOnDispose(() -> System.out.println(Thread.currentThread().getName()))
    .unsubscribeOn(Schedulers.computation())
    .doOnDispose(() -> System.out.println(Thread.currentThread().getName()))
    .subscribe();
  
  disposable.dispose();
  
  /*
   * 执行结果：
   * main
   * RxComputationThreadPool-1
   * RxCachedThreadScheduler-1
   */
  ```

* `observeOn`：设置下游观察者的执行线程，例：

  ``` java
  /*
   * map、filter、doOnXXX 这类操作符入参的执行线程受 observeOn 影响
   */
  Completable.complete()
    .observeOn(Schedulers.io())
    .doOnComplete(()-> System.out.println(Thread.currentThread().getName()))
    .observeOn(Schedulers.computation())
    .doOnComplete(()-> System.out.println(Thread.currentThread().getName()))
    .subscribe(()-> System.out.println(Thread.currentThread().getName()));
  
  /*
   * 执行结果：
   * RxCachedThreadScheduler-1
   * RxComputationThreadPool-1
   * RxNewThreadScheduler-1
   */
  ```

## 异步逻辑与同步逻辑组合

* `map`：通过当前值计算生成新的值，例：

  ``` java
  // 将所有值翻倍
  Observable.fromArray(1, 2, 3)
          .map(n -> n * 2)
          .subscribe(doubleN ->{
             // do something 
          });
  ```

* ```filter```：过滤出符合条件的值，例：

  ``` java
  // 过滤出所有的奇数
  Observable.fromArray(1, 2, 3)
          .filter(n -> n % 2 != 0)
          .subscribe(odd -> {
  						// do something 
          });
  ```

* `doOn...`：在指定的生命周期执行相应的逻辑，例：

  ``` java
  // 在执行异步任务时弹出等待对话框
  doSomethingAsync()
          .doOnSubscribe(ignore-> showLoadingDialog())
          .doOnTerminate(()-> dismissLoadingDialog())
  ```

## 异步逻辑与异步逻辑组合

* `flatMap`：类似于 map，只不过计算生成新值是异步逻辑，例：

  ``` java
  // 加载图片后进行高斯模糊
  loadBitmapAsync()
          .flatMap(bitmap -> blurBitmapAsync(bitmap))
  ```

* `zip`：通过多个异步计算结果计算出新的值，例：

  ``` java
  // 同步逻辑
  Bitmap bitmap1 = loadBitmap1();
  Bitmap bitmap2 = loadBitmap2();
  Bitmap result = composeBitmap(bitmap1, bitmap2);
  
  // 异步逻辑
  Single.zip(
          loadBitmap1Async(),
          loadBitmap2Async(),
          Pair::new
  )
          .flatMap(bitmapPair -> composeBitmapAsync(bitmapPair.first, bitmapPair.second))
  ```


* `merge/concat`：聚合多个异步计算结果，例：

  ``` java
  // 先利用本地数据展示页面，等网络数据返回再做一次更新
  Single.concatEager(Arrays.asList(
          loadFromLocalAsync(),
          loadFromRemoteAsync()
  ))
  ```

## 异步逻辑异常处理

* `onErrorReturn\onErrorResumeNext`：可恢复异常执行异常恢复，例如：

  ``` java
  // 同步异常处理逻辑
  private int foo() {
      try {
          return doSomethingMaybeThrow();
      }catch (Throwable t){
          return 0;
      }
  }
  
  // 异步异常处理逻辑
  private Single<Integer> fooAsync(){
     return Single.fromCallable(()-> doSomethingMaybeThrow())
             .subscribeOn(...)
             .onErrorReturn(throwable -> 0);
  }
  ```

# 实现登陆流程

接下来将用 RxJava 重新设计我们的登陆流程相关的接口并重新实现该业务流程。

## 接口设计

``` java
public interface LoginService {

    Completable loginAsync(String username, String password);
}

public interface SsoService {

    Single<Token> getTokenAsync(String username, String password);
}

public interface ImService {

    Completable login(Token token);

}

public interface UserInfoService {

    Single<UserInfo> getUserInfoAsync(Token token);
}
```

## LoginService 实现

```java
class LoginServiceImpl implements LoginService {

    private final SsoService ssoService;
    private final ImService imService;
    private final UserInfoService userInfoService;

    LoginServiceImpl(
      SsoService ssoService,
      ImService imService,
      UserInfoService userInfoService) {
        this.ssoService = ssoService;
        this.imService = imService;
        this.userInfoService = userInfoService;
    }

    @Override
    public Completable loginAsync(String username, String password) {
        return ssoService.getTokenAsync(username, password)
                .flatMapCompletable(token -> Completable.mergeArray(
                        imService.login(token),
                        userInfoService.getUserInfoAsync(token)
                                .flatMapCompletable(this::saveUserInfoAsync)
                ))
                .onErrorResumeNext(this::handleLoginException);
    }

    private Completable saveUserInfoAsync(UserInfo userInfo) {
        ...
    }

    private Completable handleLoginException(Throwable t) {
        ...
        return Completable.error(new BusinessException());
    }

}
```

可以看到，相比起利用回调实现的异步代码，代码量减少了很多，很多异步细节已经被封装到了操作符中，所以就需要大家对 RxJava 的操作符比较熟悉才能比较好的理解代码的含义，充分利用上 Android Studio 的 Quick Documentation 功能，相信理解操作符含义不是一件难事。

## 异步接口的使用

```java
Disposable disposable = loginService.loginAsync(..., ...)
        .observeOn(...)  // 切换 subscribe 中代码执行线程
        .subscribe(() -> {
            hint user login success
        }, throwable -> {
            hint user login failure
        });

disposable.dispose(); // 中断异步执行逻辑
```

对于使用者来说，可以通过调用 `observeOn()`  指定执行线程，在调用 `subscribe()` 后会返回一个 Disposable 对象，该对象就同在回调章节提到的 Cancelable 的作用一样，调用该对象的 `dispose()` 方法即可用于中断异步流程。不过 RxJava 没有将异步的生命周期控制封装进入链式 API 中，如果有大量需要取消的异步流程管理起来非常的麻烦，这时候推荐引入第三方库的扩展 API 来进行管理，目前我个人认为扩展 API 设计得比较优秀的是：[uber/AutoDispose](https://github.com/uber/AutoDispose)，有兴趣的同学可以自行学习。

# 总结

**优点**：

* 链式的 API 设计，从源头上避免了地狱回调的产生。
* 封装了线程切换的 API，使得切换代码执行线程更加简单方便。
* 丰富的操作符封装了通用的异步概念，让核心逻辑清晰。
* 可以作为 Java Stream API 在 Android 平台上的替代品。

**缺点**：

* 相比起同步代码，需要熟悉 API 才能理解代码流程，有学习成本。
* 每个操作符背后都需要新建一系列的小对象，有一定的性能开销。
* 异步调用的生命周期控制没有封装进链式 API 中，需要通过第三方库来补足 。