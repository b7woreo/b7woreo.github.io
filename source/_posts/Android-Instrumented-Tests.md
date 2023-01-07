---
title: Android Instrumented Tests
date: 2019-05-26
tags:
- Android
---
测试 Android 代码一共有两种方式：一种是运行在本地电脑上，称作 Android Junit；一种是运行在 Android 上，称作 Android Instrumented Test。它们的区别是 Junit 不能调用 Android api，只能用于测试 Android 无关的代码；而 Instrumented Test 则没有任何的限制，不过由于 Instrumented Test 需要运行在 Android 宿主上，所以在执行流程上比 Junit 更繁琐和耗时。本文讲介绍如何配置、编写、执行 Instrumented Test。

# 引入依赖  
要执行 Instrumented Test 首先需要引入必要的依赖库，Google 发布了 AndroidX 作为新的支持库，本文也将使用 AndroidX 作为演示，与旧的 support 库会有少许的不同。在 module 级别的 build.gralde 中添加如下代码：  
``` gradle
dependencies{
    androidTestImplementation 'androidx.test:runner:1.1.1'
    androidTestImplementation 'androidx.test.ext:junit:1.1.0'
}
```
`androidTestImplementation` 代表只有在执行 Instrumented Test 的时候才会打包该依赖，其它情况下不会引入测试用的依赖库。

# 编码测试代码
添加好了必要的依赖后，就可以在开始编写测试代码了。测试代码存放在 /src/androidTest/java 目录中。示例如下：
``` java
@RunWith(AndroidJUnit4.class)
public class InstrumentedTestDemo {

    @Test
    public void testHelloWorld() {
        assertEquals("Hello world", "Hello world");
    }

}
```
注意，需要用 `@RunWith` 注解指定 Runner。其它就和 Junit 没有什么差别了。要获取 Context 或者执行一些 Android 特有的测试操作具体可以参考官方文档给出的 api。

# 部署测试  
测试代码编写好后，接下来就可以把代码部署到测试机器上了。机器与电脑连接后，首先需要把我们的主应用安装到机器上，在终端中执行：`gradlew installDebug`。接着部署测试代码，在终端中执行：`gradlew installDebugAndroidTest`。这时候，我们可以确认一下测试是否已经成功部署，在终端中执行：`adb shell pm list instrumentation`，  这时在终端中会给出一个列表，本文示例代码 application id = com.chrnie.android.instrumented.test，于是会列表中会有如下一行信息代表测试代码部署成功：  
``` shell
instrumentation:com.chrnie.android.instrumented.test.test/androidx.test.runner.AndroidJUnitRunner (target=com.chrnie.android.instrumented.test)
```
其中 target 的值与我们的 application id 相同。`com.chrnie.android.instrumented.test.test` 代表 **test package**，`androidx.test.runner.AndroidJUnitRunner` 代表 **runner_class**，这两个参数在下面执行时需要用到。

# 执行测试
确认测试已经部署成功后，就可以运行我们的 Instrumented Test 了，在终端中执行：`adb shell am instrument -w <test_package_name>/<runner_class>`。  
**test_package_name** 和 **runner_class** 是部署测试章节中提到的两个变量。如此，就可以在终端中看到测试结果了。本示例运行的结果如下：
``` shell

com.chrnie.android.instrumented.test.InstrumentedTestDemo:.

Time: 0.027

OK (1 test)

```

# 利用 gradle 执行测试
上文已经介绍完了 Instrumented Test 的完整流程了，不过在部署和执行阶段需要输入多个命令很是繁琐。其实 Android Gradle Plugin 已经帮我们编写好了一个 task，只要执行就可以完成一条龙的部署测试、执行测试、卸载测试的流程了，该 task 的名字是：`connectedAndroidTest`。不过在运行该命令之前，还有一个很重要的事情需要完成，就是在 gradle 脚本中指定 runner class，默认的 runner class 是 android.test.InstrumentationTestRunner 和我们用到的不同。在添加了测试的 module 的 build.gradle 中添加如下代码指定我们需要的 runner class：`android.defaultConfig.testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"`，之后再在终端中执行: `gradlew connectedAndroidTest` 就可以享受到一条龙的服务了。

# 参考
[Test from the command line](https://developer.android.com/studio/test/command-line)