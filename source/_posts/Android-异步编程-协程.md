---
title: Android 异步编程-协程
date: 2020-04-04
typora-root-url: ../../source
tags:
- Android
- Kotlin
- Kotlin Coroutine
---

在前两篇博文中介绍了如何通过回调或者是 Rxjava 库来完成异步编程，回调是一种最原始的实现异步的方式，而 RxJava 作为第三方库虽然补足了回调实现的一些缺陷，但毕竟不是从语言层面解决问题，因此接口的设计多多少少还是受到了一些限制，例如只能通过链式 API 的表现形式解决地狱回调的问题，从而需要学习各式各样的 API 函数导致了学习成本的提高。那么有没有从语言层面解决回调缺陷问题的异步方案呢？答案是肯定的，那就是利用 kotlin 语言中的协程解决。关于协程的一些实现原理可以参看 {%post_link Kotlin-协程之-suspend-function %}，本文的重点将放在如何利用协程替代回调设计出更加易于使用的 API 接口上。

# 异步接口设计

同 RxJava 一样，同步的接口转换成通过协程的异步接口也有一些通用的原则：

* 同步函数改写成 suspend 函数：

  ```kotlin
  // 没有返回值的同步接口
  fun foo() {
    ...
  }
  
  // 没有返回值的异步接口
  suspend fun fooAsync() = withContext(Dispatchers.Default) { foo() }
  
  // 有返回值的同步接口
  fun getFoo(): String {
    ...
  }
  
  // 有返回值的异步接口
  suspend fun getFooAsync(): String = withContext(Dispatchers.Default) { getFoo() }
  ```

  在上面代码中，我们通过 withContext 函数，轻松的将耗时的同步函数切换到了默认的线程池中进行执行，从而不会对调用线程造成阻塞。

* Callback 类型异步函数改写成 suspend 函数：

  ``` kotlin
  suspend fun getFooAsync() = suspendCoroutine<String> { continuation ->
      getFoo { continuation.resume(it) }
  }
  
  fun getFoo(callback: (String) -> Unit) {
      ...
  }
  ```

* 注册值变化监听器、反注册值变化监听器的组合可以改写成返回值为 Flow<T> 的函数：

  ``` kotlin
  // 同步接口
  fun addOnFooChangeListener(listener: (String) -> Unit) {
    ...
  }
  
  fun removeFooChangeListener(listener: (String) -> Unit) {
    ...
  }
  
  // 异步接口
  fun observeFoo(): Flow<String> = callbackFlow {
      val observe = { next: String -> sendBlocking(next) }
      addOnFooChangeListener(observe)
      awaitClose { removeFooChangeListener(observe) }
  }
  ```

# 常见使用场景

得益于 kotlin 在语言层面增加了 suspend 关键字，异步接口的设计相较于 RxJava 少了许多概念，利用协程编写的异步代码风格几乎与同步代码一致，下面将简单列举一些常见的使用场景：

* 异步逻辑与同步逻辑组合

  ```kotlin
  showLoadingDialog()
  doSomethingAsync()
  dismissLoadingDialog()
  ```

* 异步逻辑与异步逻辑组合

  ```kotlin
  val original = loadBitmapAsync()
  val blured = blurBitmapAsync(original)
  ```

* 异常处理

  ```kotlin
  try{
    doSomethingAsync()
  }catch(e: BusinessException){
    // 处理异常
  }
  ```

* Flow

  Flow API 的设计与 RxJava 中的 Observable 大同小异，使用方式可以参看 [官方文档](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow/index.html) 。本文的重点不在于介绍响应式设计，便不赘述相关的内容，感兴趣的同学可以去专门去了解响应式函数编程的相关内容。

# 执行环境

在协程实现中，函数会绑定一个执行环境称为：`CoroutineContext`，`CoroutineContext` 由 `CoroutineContext.Element` 组成，标准库中包含了几个默认的元素，如果有需求的话也可以自己扩展新的元素类型。通过在调用 `launch`、`async`、`withContext` 函数时将 CoroutineContext 作为参数传入，即可完成执行环境的切换。例如：

```kotlin
object FooCoroutineScope : CoroutineScope {
    override val coroutineContext: CoroutineContext
        get() = CoroutineName("FooCoroutineScope")

    suspend fun foo() = async {
        println("ContextName: ${coroutineContext[CoroutineName]?.name}")
        // 输出：FooCoroutineScope
    }.await()
}

suspend fun foo() = coroutineScope {
    // this 引用可以获取到当前协程的执行环境
    println("ContextName: ${coroutineContext[CoroutineName]?.name}")
    // 输出：null

    withContext(CoroutineName("withContext")) {
        println("ContextName: ${coroutineContext[CoroutineName]?.name}")
        // 输出：withContext
        
        FooCoroutineScope.foo()
    }
}
```

## 元素（CoroutineContext.Element）

### CoroutineName

类似线程名一样，CoroutineName 标识了协程执行环境的名字

### ContinuationInterceptor

在协程中理应没有线程的概念，在服务器领域中并且有完善的异步 io 的支持下，其实是不用关心函数的执行线程的，可以完全托管给默认分发器（Dispatcher）管理。但是在 Android 环境下，有 UI 线程的存在并且没有异步 io 的支持，函数的执行线程还是必须由代码编写者自行进行控制，标准库中提供了三种类型的分发器方便用户使用：

1. Dispatchers.Main：在 UI 线程中执行
2. Dispatchers.Default：执行计算密集型任务使用
3. Dispatchers.IO：执行 io 密集型任务使用

### Job

在介绍 RxJava 的时候提到过它的生命周期控制设计得并不是那么友好，没有将 API 融入到链式 API 的设计当中，导致需要依赖其它的第三方库才能更好的完善使用体验。但在协程中就不一样了，Job 就是对一个任务的抽象，每当调用`launch`、`async` 函数时，就会新建一个 `CoroutineScope.coroutineContext`  中的 Job 的子任务，当调用父任务的 `cancel` 函数时，即可将其本身及所有的子任务一起取消，可以非常方便的完成调用生命周期的控制，详细的生命周期控制示例参照 [实现登录流程](#实现登录流程) 章节示例代码。

### CoroutineExceptionHandler

类似于 `Thread.setDefaultUncaughtExceptionHandler` ，可以给协程设置默认的异常处理器，当从协程中抛出的异常没有被处理时，将进入 CoroutineExceptionHandler 中处理。

# 实现登录流程

接下来将用协程重新设计我们的登录流程相关的接口并重新实现该业务流程。

## 接口设计

```kotlin
interface LoginService {

    suspend fun loginAsync(username: String, password: String)

}

interface SsoService {

    suspend fun getTokenAsync(username: String, password: String): Token

}

interface ImService{

    suspend fun login(token: Token)

}

interface UserInfoService{

    suspend fun getUserInfoAsync(token: Token): UserInfo

}
```

## 生命周期控制

在编写接口实现前，我们先介绍一下在协程中如何高效的进行生命周期控制。考虑一般的场景下，应用的生命周期应该如下所示：

![](/images/202004052118.png)

Service 的生命周期和进程的生命周期一致，而 Activity 的生命周期只是进程生命周期的一个子集。结合登录的业务场景，在登录页面用户点击登录按钮，登录的请求转发到了 LoginService 接口，然后在请求服务器的过程中如果 Activity 的生命周期结束了，但是进程的生命周期没有结束，其实整个登录流程也还是可以继续进行，只是在登录完成的时候，不需要再回调到 activity 中执行 UI 相关的逻辑罢了，等到用户再次回到应用时，用户看到的就是已经完成了登录。

这个生命周期的控制在协程中可以很轻易的通过定义 `CoroutineScope` 实现，对于 Activity 和 Service 我们分别定义 CoroutineActivity 和 CoroutineService 如下所示：

```kotlin
abstract class CoroutineActivity : Activity(), CoroutineScope {

    private lateinit var job: Job

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        job = SupervisorJob()
    }

    override fun onDestroy() {
        super.onDestroy()
        job.cancel()
    }

    override val coroutineContext: CoroutineContext
        get() = Dispatchers.Main + job + CoroutineName(this::class.simpleName!!)

}

abstract class CoroutineService : CoroutineScope{

    override val coroutineContext: CoroutineContext
        get() = Dispatchers.Default + CoroutineName(this::class.simpleName!!)

}
```

在 `CoroutineActivity` 中在 `onCreate` 的时候创建与之关联的 Job，在 `onDestroy` 的时候调用 `Job.cancel` 取消，由于通过该 `CoroutineActivity.launch`  和 `CoroutineActivity.async` 创建的所有协程都是继承自 Job 的，所以在 `onDestroy` 时所有的进行中的协程任务都将被结束。`CoroutineService` 因为不需要取消，所以就不需要创建与之关联的 Job。

在上述 Scope 中，除了完成了不同抽象层次的生命周期控制，还定义了 `CoroutineName` 便于 Debug 时获取相关信息和将 Activity 相关的协程任务指定在 UI 线程中执行，Service 相关的协程任务指定在默认线程池中执行，模板化的完成了线程切换任务，让业务逻辑更加关注与业务本身。

## LoginService 实现

```kotlin
class LoginServiceImpl(
    val ssoService: SsoService,
    val imService: ImService,
    val userInfoService: UserInfoService
): CoroutineService(), LoginService{

    override suspend fun loginAsync(username: String, password: String) = async {
        try {
            val token = ssoService.getTokenAsync(username, password)
            joinAll(
                async { imService.login(token) },
                async {
                    userInfoService.getUserInfoAsync(token)
                    saveUserInfoAsync()
                }
            )
        } catch (t: ServiceException) {
            handleLoginException(t)
            throw t
        }
    }.await()

    private suspend fun saveUserInfoAsync() {

    }

    private suspend fun handleLoginException(t: Throwable){
    }
}
```

## 异步接口的使用

```kotlin
launch {
  try {
    loginService.loginAsync(username, password)
    // hint user login success
  }catch (e: ServiceException){
    // hint user login failure 
  }
}

```

通过示例代码可以看到，利用协程设计的 API 几乎与同步 API 一模一样，除了多了 suspend 的关键字。而代码实现也非常的简洁，业务逻辑中只专注于业务逻辑的体现，不像 RxJava 需要专门学习各种操作符的含义也可基本理解整个业务流程，这非常有利于快速在团队中推广普及。

# 结论

**优点**：

* 编写、阅读异步代码逻辑和同步代码相似，相比起 RxJava 学习成本低。
* 基于 cas 实现了一套用于协程环境中的同步容器用于替代 Java 标准库中的基于锁实现的同步容器，拥有更加优异的并发性能 。
* 函数支持 inline 关键字，协程库实现时会尽最大可能减少高阶函数调用的开销。 

**缺点**：

* 需要有 kotlin 基础。