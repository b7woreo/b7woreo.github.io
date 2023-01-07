---
title: 使用 jmh 做基准测试（benchmarks）
date: 2019-03-30
tags:
- Java
---
jmh 是 JVM 平台上的基准测试（benchmarks）工具，方便在做性能优化时获取量化性能指标。本文主要将介绍在基于 gradle 构建的 kotlin 项目中如何引入 jmh 并使用。默认读者知道已经如何通过 gradle 创建一个 kotlin 工程，将跳过基本的使用 gradle 创建 kotlin 工程的步骤，直接从修改 build.gradle 脚本开始演示。  

# 添加基准测试源码目录

一般情况下，我们希望编写的基准测试代码与正式工程代码是源码分离的，基准测试代码依赖于正式工程代码，但是在正式打包正式工程代码的时候不会打包基准测试的代码。于是，我们需要添加一个新的源码目录专门用于存放基准测试代码，并使其依赖于正式工程代码。这可以通过修改 sourceSets 解决：  

``` gradle
sourceSets {
    jmh {
        compileClasspath += sourceSets.main.runtimeClasspath
        runtimeClasspath += sourceSets.main.runtimeClasspath
    }
}
```

修改完成后，便可以在 src 目录下新建 jmh/kotlin 目录，将基准测试代码放入其中。

# 添加依赖

使用到的 jmh 核心依赖有两个： 

* jmh-core  
* jmh-generator-annprocess  

其中一个需要用到注解处理器，在 kotlin 工程中，需要使用 kapt 替代默认的 annotationProcessor，所以需要先引入 kapt gradle plugin：  

``` gradle
apply plugin: 'kotlin-kapt'
```

之后在 dependencies 中添加对应的依赖：  

``` gradle
dependencies {
    ...
    jmhImplementation 'org.openjdk.jmh:jmh-core:1.21'
    kaptJmh 'org.openjdk.jmh:jmh-generator-annprocess:1.21'
}
```

注意到，因为我们只有在做基准测试时才会需要到这两个依赖，所以使用的是 jmhImplementation 和 kaptJmh，避免在打包主工程代码时 jmh 依赖也被打包进去。

# 编写基准测试代码

这里以计算1到10000的和平均耗时为例编写基准测试代码：

``` kotlin
@BenchmarkMode(Mode.AverageTime)
open class HelloBenchmark {

    @Benchmark
    fun sumOf10000() = (1..10000).sum()
}
```
需要注意的是，用于基准测试的类需要是可继承的，所以在 kotlin 中需要给类加上 open 修饰符。

# 运行基准测试

官方给出了两种运行基准测试的方式：  

0. 通过 Java API
0. 通过命令行

下面将分别演示这两种运行基准测试的使用方法。  

## 通过 Java API 运行

通过 Java API 运行基准测试首先需要给程序添加一个执行入口，也就是添加 main 函数。然后在 main 函数中调用对应的 api 函数即可。例如，我们在 com.sample.HelloBenchmark 文件中添加 main 函数：  

``` kotlin
fun main() {
    val opt = OptionsBuilder()
            .include(HelloBenchmark::class.simpleName)
            .forks(1)
            .build()
    Runner(opt).run()
}
```

然后为了自动化编译和运行操作，可以通过在 gradle 中添加 task 实现：

``` gradle
task jmh(type: JavaExec) {
    classpath = sourceSets.jmh.runtimeClasspath
    main = 'com.example.HelloBenchmarkKt'
}
```

最后，只需要执行 `gradle jmh` 命令即可运行基准测试。

## 通过命令行运行

官方给出的命令行调用示例为：

``` bash
java -jar target/benchmarks.jar 
```

实际上就是执行 target/benchmarks.jar 中预定义好的 main 函数，该函数定义在 org.openjdk.jmh.Main 中，所以其实就和上文一样通过添加一个 task 来执行该 main 函数即可：

``` gradle
task jmh(type: JavaExec) {
    classpath = sourceSets.jmh.runtimeClasspath
    main = 'org.openjdk.jmh.Main'
    args = [...]
}
```

与 Java API 不同点在于，基准测试的运行参数设置方式不一样。通过 Java API 运行的话是通过调用 OptionsBuilder 类的方法设置，而通过命令行调用的话需要把运行参数附加在命令后（在 jmh task 的 args 中添加对应参数）。  
最后也是通过执行 `gradle jmh` 命令即可运行基准测试。

# 参考
[OpenJDK: jmh](https://openjdk.java.net/projects/code-tools/jmh/)