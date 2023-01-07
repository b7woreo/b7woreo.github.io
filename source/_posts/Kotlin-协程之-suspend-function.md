---
title: Kotlin 协程之 suspend function
date: 2018-12-02
tags:
- Kotlin
- Kotlin Coroutine
---
Kotlin 的协程设计可谓是非常的简洁，除了增加了 suspend 关键字，其余的所有功能都在标准库中实现。而 suspend 关键字正是整个协程实现的关键，本文将探索 suspend 关键字到底做了什么。

# 编写 suspend function  

``` kotlin
suspend fun foo() {
    println("start foo suspend function")
    delay(1000)
    println("end foo suspend function")
}
```

上面就是一个简单的 suspend function 示例，首先打印了一句 "start foo suspend function" 表明进入函数，然后调用 delay 方法暂停函数的执行，等待暂停完成后恢复函数的执行，打印了一句 "end foo suspend function" 表明函数执行结束将退出。

# 反编译 suspend function  

## Idea Kotlin Plugin  

一般 Kotlin 的代码编写都是在 Idea 或者 Android Studio 上完成的，官方提供了插件可以用于查看编写好的 Kotlin 代码的 bytecode 并可以通过 bytecode 还原成 Java 代码。这方便我们探究 Kotlin 编译器对 Kotlin 代码的操作。  
使用方法： 

0. 在 Idea 中打开想反编译的 Kotlin 源码文件
0. 点击 Idea 菜单栏的 Tools 菜单
0. 选中 Kotlin  菜单项
0. 点击 Show Kotlin Bytecode 选项（此时会弹出 Kotlin Bytecode 窗口）
0. 点击 Kotlin Bytecode 窗口上方的 Decompile 按钮

## 反编译后的 suspend function  

``` kotlin
@Nullable
public static final Object test(@NotNull Continuation var0) {
    Object $continuation;
    label28: {
        if (var0 instanceof <undefinedtype>) {
        $continuation = (<undefinedtype>)var0;
        if ((((<undefinedtype>)$continuation).label & Integer.MIN_VALUE) != 0) {
            ((<undefinedtype>)$continuation).label -= Integer.MIN_VALUE;
            break label28;
        }
        }

        $continuation = new ContinuationImpl(var0) {
        // $FF: synthetic field
        Object result;
        int label;

        @Nullable
        public final Object invokeSuspend(@NotNull Object result) {
            this.result = result;
            this.label |= Integer.MIN_VALUE;
            return MainKt.test(this);
        }
        };
    }

    Object var2 = ((<undefinedtype>)$continuation).result;
    Object var4 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
    String var1;
    switch(((<undefinedtype>)$continuation).label) {
    case 0:
        if (var2 instanceof Failure) {
        throw ((Failure)var2).exception;
        }

        var1 = "start test suspend function";
        System.out.println(var1);
        ((<undefinedtype>)$continuation).label = 1;
        if (DelayKt.delay(1000L, (Continuation)$continuation) == var4) {
        return var4;
        }
        break;
    case 1:
        if (var2 instanceof Failure) {
        throw ((Failure)var2).exception;
        }
        break;
    default:
        throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
    }

    var1 = "end test suspend function";
    System.out.println(var1);
    return Unit.INSTANCE;
}
```

将上文提到的 foo 函数反编译后将得到如上所示的 Java 代码。可见，每个 suspend function 最终会被编译成一个无状态的静态函数并且该函数比我们声明的函数的参数列表中多出了一个参数：Continuation，Continuation 中携带着该函数的运行状态。

# suspend function 运行步骤  

0. 当我们在 suspend function A 中调用 suspend function B 时，会将 suspend function A 的 Continuation 当做参数传入调用的 suspend function B 中。每个 suspend function 都会由编译器生成一个特定的 Continuation 子类，所以一开始进入方法时，首先会检查传入的 Continuation 类型与当前 suspend function 所生成的 Continuation 的类型是否一致。如果 Continuation 类型不一致的话说明是被别的 suspend function 调用进入该方法，需要生成一个新的 Continuation 实例记录当前函数的运行状态，新的 Continuation 是 ContinuationImpl 的子类型。如果 Continuation 类型一致的话，说明该函数是从上一个中断点恢复继续执行的，则不需要创建新的 Continuation 实例。  
0. 从 Continuation 中的 result 字段中，可以得到上一次 suspend function 中断后运行的结果。例如在我们的示例中调用了 delay 函数，该函数的返回值类型是 Unit 类型，那么在顺利执行 delay 函数并恢复 foo 函数执行时，Continuation.result 中的结果应该是 Unit.INSTANCE。但是如果 delay 被取消了，那么 delay 会抛出一个 CancellationException，异常会被封装在 Failure 类型中。  
0. Continuation 中的 label 字段描述了当前函数运行到了哪个阶段。通过 label 跳转到对应的暂停点恢复函数的执行，在我们编写 Kotlin 代码时，每一次对 suspend function 的调用都会由编译器自动生成一个新的中断恢复点，具体实现方法就是在 switch 中增加一个 case 代码块。  
0. 进入到 label 对应的代码块后，首先会检查 result 的类型，在上面有提到过，调用的 suspend function 抛出异常时，会被封装在 Failure 类型中，如果 result 是 Failure 类型的，那么就需要抛出封装在内的异常，否则函数继续执行，直到遇到对 suspend function 的调用。在调用 suspend function 前，首先会将 Continuation.label 设置为当函数恢复执行后需要进入的对应代码块的 label 值（一般就是当前 label 值 +1），在调用 suspend function 时，如果调用的 suspend function 直接返回的结果是 COROUTINE_SUSPENDED，那么函数也将返回，等待恢复。  
0. 最终函数会返回对应的执行结果，例如示例中函数签名返回值类型为 Unit，那么最后返回了 Unit.INSTANCE。  

# ContinuationImpl 实现原理  

ContinuationImpl 实现了 Continuation 接口，Continuation 接口代码如下：  

``` kotlin
/**
 * Interface representing a continuation after a suspension point that returns value of type `T`.
 */
@SinceKotlin("1.3")
public interface Continuation<in T> {
    /**
     * Context of the coroutine that corresponds to this continuation.
     */
    public val context: CoroutineContext

    /**
     * Resumes the execution of the corresponding coroutine passing successful or failed [result] as the
     * return value of the last suspension point.
     */
    public fun resumeWith(result: Result<T>)
}
```

从接口签名可以看出，Continuation 实际上就是一个带有回调执行环境的回调接口，每当一个 suspend function 执行完成之后，就会通过调用 resumeWith 方法告知调用方执行结果。那么ContinuationImpl 是如何实现 resumeWith 函数的呢？  
ContinuationImpl 继承自 BaseContinuationImpl，resumeWith 方法就在该类中被实现。具体实现如下：   

``` kotlin
    public final override fun resumeWith(result: Result<Any?>) {
        // Invoke "resume" debug probe only once, even if previous frames are "resumed" in the loop below, too
        probeCoroutineResumed(this)
        // This loop unrolls recursion in current.resumeWith(param) to make saner and shorter stack traces on resume
        var current = this
        var param = result
        while (true) {
            with(current) {
                val completion = completion!! // fail fast when trying to resume continuation without completion
                val outcome: Result<Any?> =
                    try {
                        val outcome = invokeSuspend(param)
                        if (outcome === COROUTINE_SUSPENDED) return
                        Result.success(outcome)
                    } catch (exception: Throwable) {
                        Result.failure(exception)
                    }
                releaseIntercepted() // this state machine instance is terminating
                if (completion is BaseContinuationImpl) {
                    // unrolling recursion via loop
                    current = completion
                    param = outcome
                } else {
                    // top-level completion reached -- invoke and return
                    completion.resumeWith(outcome)
                    return
                }
            }
        }
    }
```

该实现的核心内容在 while 循环体内。首先调用 invokeSuspend 方法，该方法由编译器实现，实现的具体内容就是调用对应的 suspend function 由编译器生成的静态方法，返回结果表明了函数的执行状态。如果调用了 suspend function 导致函数暂停了，那么返回值为 COROUTINE_SUSPEND，将直接退出 resumeWith 方法，等待 suspend function 执行结束后的下一次 resumeWith 调用。如果该函数已经执行完成，那么执行结果将被封装在 Result 中，通过 completion.resumeWith 告知调用方。  
这里有一个优化，当调用方的 Continuation 的实现也是 BaseContinuationImpl 的子类时，不通过调用 resumeWith 方法的形式通知调用方结果，而是直接更新循环的参数（current 和 param）再执行一次循环体中的内容，这样可以减少一次函数调用栈的分配，防止递归深度过大而出现爆栈问题。

# 参考资料
[KotlinConf 2017 - Deep Dive into Coroutines on JVM by Roman Elizarov](https://www.youtube.com/watch?v=YrrUCSi72E8)