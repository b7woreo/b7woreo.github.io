---
title: Kotlin 中另类的单例写法
date: 2019-08-02
tags:
- Kotlin
---
Java 的单例就不用多说了，懒汉式、饿汉式、双重检查锁等等很多种写法。而在 Kotlin 中呢，最开始学习到的一定是通过 object 关键字来编写单例。示例：  
``` kotlin
object Singleton {
 
    fun foo(): String = “Singleton"
}
```
但是这种写法有一个缺点，不支持有参构造函数。而在 Android 环境中，大部分情况都是需要配合 Context 才能完成一系列操作的，Context 几乎是大部分方法都需要用到的参数，所以通过构造函数获取并在对象中持有该实例是最好不过的了。于是乎，想要编写有参构造函数的单例就不得不放弃 Kotlin 的语法糖，将 Java 中的单例模板翻译成用 Kotlin 样子。就有了这样的代码：
``` kotlin
class Singleton(context: Context) {

    companion object {

        @Volatile
        var _instance: Singleton? = null

        fun instance(context: Context): Singleton {
            return _instance ?: synchronized(this) {
                _instance ?: Singleton(context).also { _instance = it }
            }
        }
    }

    fun foo(): String = "Singleton"
}
```
这种粗暴的翻译写法令我陷入了思考，有没有更 Kotlin style 的写法。最终有了如下代码：
``` kotlin
inline fun <reified T> singleton(crossinline initializer: (Context) -> T): (Context) -> T {
    var instance: T? = null
    return { context ->
        instance ?: synchronized(T::class) {
            instance ?: initializer(context).also { instance = it }
        }
    }
}

class Singleton(context: Context) {

    companion object {

        val instance = singleton { Singleton(it) }
    }

    fun foo(): String = "Singleton"
}
```
这个示例利用 Kotlin 的闭包，将原来暴露在 companion object 中的实例变量封装在了的函数域内，除了 singleton 方法返回的高阶函数外其它任何地方的代码都无法访问，提升了闭合性。并利用 Kotlin 对函数的调用的语法糖，模糊了调用变量形式的函数与 fun 形式的函数的区别，看起来就和调用一个普通的函数一样。