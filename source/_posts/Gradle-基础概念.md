---
layout: post
title: Gradle 基础概念
typora-root-url: ../../source
date: 2021-04-18 16:54:53
tags:
- Gradle
---


# 是什么？

>**Gradle**是一个基于[Apache Ant](https://zh.wikipedia.org/wiki/Apache_Ant)和[Apache Maven](https://zh.wikipedia.org/wiki/Apache_Maven)概念的项目[自动化建构](https://zh.wikipedia.org/wiki/自動化建構)工具。Gradle 构建脚本使用的是 [Groovy](https://zh.wikipedia.org/wiki/Groovy) 或 [Kotlin](https://zh.wikipedia.org/wiki/Kotlin) 的[特定领域语言](https://zh.wikipedia.org/wiki/特定领域语言)来编写的[[2\]](https://zh.wikipedia.org/wiki/Gradle#cite_note-2)，而不是传统的[XML](https://zh.wikipedia.org/wiki/XML)。[[3\]](https://zh.wikipedia.org/wiki/Gradle#cite_note-3)
>
>当前官方支持的语言为[Java](https://zh.wikipedia.org/wiki/Java)、[Groovy](https://zh.wikipedia.org/wiki/Groovy)、[Scala](https://zh.wikipedia.org/wiki/Scala)、[C++](https://zh.wikipedia.org/wiki/C%2B%2B)、[Swift](https://zh.wikipedia.org/wiki/Swift_(程式語言))、[JavaScript](https://zh.wikipedia.org/wiki/JavaScript)等以及[Spring框架](https://zh.wikipedia.org/wiki/Spring框架)[[4\]](https://zh.wikipedia.org/wiki/Gradle#cite_note-4)。

上文引用了维基百科中对 Gradle 的介绍，简单来说 Gradle 是使用 Grovvy 或 Kotlin 编写构建脚本的自动化的构建工具，与其对标的工具有：Cmake、Webpack 等。Gradle 虽然是运行在 JVM 平台上的，但是其不仅仅支持 JVM 平台语言的自动化构建能力还支持了包括 C++、Swift 等非 JVM 平台语言的构建能力，而且该工具提供了非常强大的扩展能力你甚至于可以自己编写插件以支持任何产物的构建。

作为自动化构建工具，其最主要要解决的问题就是将源代码、资源和依赖项打包成可以运行在指定平台上的产物，例如：将 Java 代码打包成 Jar 在 JVM 上运行或打包成 Apk 在 Android 系统中运行。下图简单展示了对于一个 Java 工程来说自动化构建工具所要完成的任务，可以看到对于自动化构建工具而言，其最主要任务就是**管理依赖、输入源文件、输出产物文件**，而 Gradle 也是围绕着这三件事情来建立抽象。

![自动化构建工具](/images/202103281231.svg)

# 关键概念

在了解了自动化构建是做什么的、解决了什么问题之后，下面介绍一些 Gradle 中的关键概念，这些关键概念定义了统一的抽象概念，帮助不同类型语言的生态完成协作（例如：可以基于 Java 的自动化构建能力开发 Kotlin 的自动化构建功能）、降低定制化构建流程的门槛。

## Project

Project 是用于组织项目结构的最小单元，一般一个 Project 由一系列的源文件通过完成一个或多个 Plugin 定义的自动化构建任务最终输出一些系列的产物。每个 Gradle 工程至少会有一个 Project 位于工程的根目录，也称作 RootProject，在 RootProject 目录下可以创建一个 setting.gradle 文件用于定义该工程的 Project 结构，每个 Project 目录可以创建一个 build.gradle 文件用于描述 Project 的信息。

下面以 gradle-demo 工程为例介绍一下常见的项目结构，gradle-demo 的目录结构及 setting.gradle 文件内容如下所示。

![demo-工程](/images/20210328232814.png)

``` groovy
// file: setting.gradle

include(":sample")
include(":sample:app")
include(":sample:lib")
```

* gradle 、gradlew、gradlew.bat 是使用 gradle wrapper 所需要的文件，当使用 gradlew 替换 gradle 执行任务时使用的就是 gradle wrapper。gradlew wrapper 相较于直接使用本地 gradle 的好处是可以将构建时使用的 gradle 版本同源文件一起使用版本控制系统一起管理，这是为了满足规范的 CI\CD 流程中要求从任意版本控制系统节点中迁出的工程在任何时间构建都必须得到一致的产物的需求。其中 `gradle/wrapper/gradle-wrapper.properties` 文件中就定义了 gradlew 所需要使用的 gradle 版本。

* 在 setting.gradle 中引入了 gradle-demo 的子工程 :sample、:sample:app、:sample:lib，而 :sample:app 和 :sample:lib 同时又是 :sample 的子工程。形如 `:sample:app` 这样的表达叫做 Project 路径， RootProject 的路径是 `:`，Project 路径是 Project 的唯一标识符。

Project 的目录默认与 Project 路径一一对应，例如：`:sample:app` 的目录是 `sample/app`，但是也可以在 setting.gradle 中进行修改。例如下面的示例代码修改了 `:app` 和 `:lib` 两个 Project 的目录，此时虽然 gradle-demo 整个工程的目录结构没有改动但是 Project 结构却发生了变化。

``` groovy
include(":app")
project(":app").projectDir = file("sample/app")

include(":lib")
project(":lib").projectDir = file("sample/lib")
```

在 IDE 中识别到的 Project 结构如下所示，可以看到 sample 不再作为一个 Project 存在于 gradle-demo 工程中。

![gradle-demo Project 结构](/images/20210329171420065.png)

## Plugin

在 Project 刚新建完成时，该 Project 本身是不具备任何的构建能力的。可以在 Project 的 build.gradle 文件中通过编写 Task 等逻辑来自定义构建流程或者是通过引入 Plugin 来复用预定义的 Task 等逻辑，Plugin 就相当于一个已发布的、可重复利用的 build.gradle 文件 。

### 引入插件

下面以引入 `com.android.application` 插件为例示例插件引入的方式：

```groovy
/*
 * buildscript 代码块一般定义在 RootProject 的 build.gradle 文件中，
 * 因为在一次构建中不允许引入不同版本的依赖项。
 */
buildscript {
  
  	/*
  	 * 定义下载插件时需要用到的 Maven 仓库
  	 */
    repositories {
        google()
    }
  	
  	/*
  	 * 引入插件运行时所需要的 Jar 依赖项
  	 */ 
    dependencies {
        classpath 'com.android.tools.build:gradle:4.1.0'
    }
}

/*
 * 引入插件
 */
apply plugin: 'com.android.application'
```

### 编写插件

Gradle 插件其实就是一个编译打包好的 Jar 文件，在其中会有一个或多个路径为：`META-INF/gradle-plugins/{plugin_id}.properties` 的文件，文件的内容形如：

```
implementation-class={plugin_class_name}
```

* **plugin_id** ：在 build.gradle 中 apply plugin 时使用，例如：com.android.application。

* **plugin_class_name**：与插件 id 对应的插件实现类全类名，该类需要继承自 `Plugin<Project>` ，例如：

  ```java
  public class CustomPlugin implements Plugin<Project> {
  		
    	/*
    	 * 入参 project 是与 build.gradle 文件关联的 Project 实例，
       * 在 build.gradle 文件中 this 也是指向该 Project 实例。
    	 */
      @Override
      public void apply(Project project) {
      }
  }
  ```

## Extension

在介绍 Plugin 时知道了其实它就是一个可重复利用的 build.gradle 文件，但是在真实的环境中不可能所有的项目都是完全相同的构建环境，而 Extension 就是用于描述构建环境的 DSL（*Domain Specific Language*，领域特定语言），构建环境参数的不同可能会有不同的构建流程或影响相同流程中的 Task 的输出。作为 Android 应用的开发者最常见的 Extension 就是 `android` 代码块了：

``` groovy
android {
  /*
   * 指定编译时使用的 SDK 版本
   */
  compileSdkVersion 30
  	
  /*
   * 新增一个名为 dev 的 ProductFlavor 并对其进行配置
   */
  productFlavors {
    dev {
      // 代码块内可以配置 ProductFlavor 的相关参数
    }
  }
}
```

每个 Extension 背后都是由一个实体类定义的，例如 `com.android.application` 插件中的 `android` 扩展由 `com.android.build.gradle.AppExtension` 完成定义，将上面 groovy 语言的描述转换成 Java 的描述就变成了下面这样：

``` java
BaseExtension android = (BaseExtension) project.getExtensions().findByName("android");
android.compileSdkVersion(30);
android.productFlavors(productFlavors -> {
  productFlavors.create(
    "dev",
    productFlavor -> {
      //configure dev productFlavor
    });
});
```

## Configuration

Configuration 其实大家一定都不陌生，平时开发需求时经常需要引入第三方的依赖库，形如：`implementation "com.squareup.okhttp3:okhttp:3.14.2"`，其实 implementation 就是一个 Configuration，当前的 Project 的 Configuraiton 集合可以通过：`project.getConfigurations()` 获取。

Configuration 在源码中的类型定义：`public interface Configuration extends FileCollection, HasConfigurableAttributes<Configuration>`，可以看到它继承自了 FileCollection 接口，实际上它的主要职责就是作为一个文件的容器帮助在 Project 之间管理依赖文件的输入与输出。上面引入第三方库那代码的意思就是把 `com.squareup.okhttp3:okhttp:3.14.2` 这个依赖库文件加入到 `implementation` 这个文件集合中。

Configuration 有两个很重要的属性：isCanBeResolved 和 isCanBeConsumed ，这两个属性的值决定了该 Configuration 的用途：

| Configuration 用途        | isCanBeResolved | isCanBeConsumed | 描述                                                         |
| ------------------------- | --------------- | --------------- | ------------------------------------------------------------ |
| Bucket of dependencies    | false           | false           | 添加依赖时使用的容器，例如：implementation、api。            |
| Resolve for certain usage | true            | false           | 编译或打包时用于获取依赖项文件，例如：compileClasspath、runtimeClasspath。 |
| Exposed to consumers      | false           | true            | 向其它项目提供依赖，例如：compileElements、runtimeElements。 |
| Legacy, don’t use         | true            | true            | 用于兼容旧版本使用的，已经废弃。例如：compile。              |

只有 `isCanBeResolved` 为 true 类型的 Configuration 可以调用 FileCollection 接口的相关方法获取到实际的依赖文件（依赖文件可能通过网络下载或者由其它 Project 编译产生），而 `isCanBeConsumed` 为 true 类型的 Configuration 会在 Project 间分享产物时用到，在下面的 [如何在 Project 之间提供产物][#如何在 Project 之间提供产物] 章节会做详细的介绍。

## Task

Task 就是一个对 **输入的文件、输入的参数** 执行一定的操作最终可能 **输出文件** 的逻辑单元。Gradle 提供了丰富的 API 帮助 Task 间构造依赖关系（在 [怎么解决 Task 依赖][#怎么解决 Task 依赖] 章节会有详细的介绍），在执行 Task 时会先生成一张有向无环的依赖关系图，最终按照赖关系图的顺序完成所有任务的执行。可以通过如下代码获取到任务执行的依赖关系图：

``` java
project.getGradle().getTaskGraph().whenReady(taskExecutionGraph -> {
  // 通过 taskExecutionGraph 获取依赖信息或完成其它操作
});
```

### 自定义 Task

Task 由一组有序的 Action 定义而成，当 Task 被执行时就会按顺序依次执行对应的代码逻辑。一般自定义的 Task 类都继承自 `org.gradle.api.DefaultTask`，然后根据业务需要添加相应的 Action，例如：

``` kotlin
abstract class HelloWorld : DefaultTask() {

    init {
        // 后执行
        doLast {
            logger.lifecycle("world")
        }
        
        // 先执行
        doFirst {
            logger.lifecycle("hello")
        }
    }
}
```

``` bash
# 任务执行结果
> Task :hello
hello
world
```

除了通过调用 `doFirst` 、`doFirst` 方法添加 Action 外，还可以通过注解的方式添加：

``` kotlin
abstract class HelloWorld : DefaultTask() {

    @TaskAction
    fun action() {
        logger.lifecycle("hello world")
    }
}
```

```bash
# 任务执行结果
> Task :hello
hello world
```

### 增量构建

每一个 Task 都会有一系列的输入和输出，以编译 Java 代码使用到的 JavaCompile 任务为例，其输入有源代码和编译的目标 JDK 版本，其输出为 Class 文件。

![taskInputsOutputs](/images/202104181607.png)

Gradle 为了能加速增量构建的速度，会在每次运行 Task 前先判断该 Task 的输入和输出有没有变化，如果没有变化的话就会跳过该次 Task 的执行过程，在控制台中会以 `UP-TO-DATE` 标识被跳过的执行的 Task：

``` bash
> Task :compileJava UP-TO-DATE
```

为了能让 Gradle 系统知道 Task 中有哪些输入项和输出项，`org.gradle.api.Task` 接口提供了 `TaskInputs getInputs()` 和 `TaskOutputs getOutputs()` 方法可以用于配置输入和输出参数：

``` kotlin
abstract class CustomTask : DefaultTask() {

    var targetVersion: Int = 0

    lateinit var classpath: FileCollection

    lateinit var outputDir: File

    init {
        inputs.property("target_version", targetVersion);
        inputs.files(classpath)
        outputs.dir(outputDir)
    }
}
```

为了减少冗余代码编写，也可以用注解的方式达成同样的效果：

``` kotlin
abstract class CustomTask : DefaultTask() {

    @get:Input
    var targetVersion: Int = 0

    @get:InputFiles
    lateinit var classpath: FileCollection

    @get:OutputDirectory
    lateinit var outputDir: File
    
}
```

关于增加构建的详细信息可以查看官方文档：[Up-to-date checks (AKA Incremental Build)](https://docs.gradle.org/current/userguide/more_about_tasks.html#sec:up_to_date_checks)

# 常见的问题

## 怎么解决 Task 依赖

Gradle 为 Task 提供了多种依赖管理的方式，其最主要可以分为两类：**显式依赖** 和 **隐式依赖**。

### 显式依赖

Task 接口自身提供了 `Task dependsOn(Object... paths);` 用于直接声明 Task 间的依赖关系，例如：

``` java
tasks.create("a")

tasks.create("b") {
	// 任务 “b” 依赖于任务 “a”
  dependsOn("a")
}
```

### 隐式依赖

Task 比较常见的逻辑是对 **输入文件** 进行处理后生成 **输出文件**，例如 `JavaCompile` 任务是对 Java 源文件进行编译操作后生成 class 文件，而 Jar 任务是将 class 文件打包在一起输出 Jar 文件。所以除了显式的声明 Task 之间的依赖关系外，还可以通过输入和输出文件将 Task 关联起来形成依赖。例如：

``` kotlin
/*
 * 在 outputFile 文件中写入 “Hello world!” 字符串
 */
abstract class ProducerTask : DefaultTask() {

    @OutputFile
    val outputFile: RegularFileProperty = project.objects.fileProperty()

    @TaskAction
    fun produce() {
        outputFile.get().asFile.writeText("Hello world!")
    }

}
```

``` kotlin
/*
 * 读取 inputFile 的内容并打印在控制台中
 */
abstract class ConsumerTask : DefaultTask() {

    @InputFiles
    val inputFile: RegularFileProperty = project.objects.fileProperty()

    @TaskAction
    fun consume() {
        println(inputFile.get().asFile.readText())
    }
}
```

``` groovy
def producer = tasks.create("producer", ProducerTask.class) {
    outputFile = layout.buildDirectory.file("out.txt")
}

tasks.create("consumer", ConsumerTask.class) {
    inputFile = producer.outputFile
}
```

``` bash
# 任务执行结果
> Task :consumer
Hello world!
```

## 如何在 Project 之间分享产物

在项目中我们经常会使用类似 `implementation project(":library")` 的方式引入依赖，以这种方式引入依赖时实际上是将当前 Project 中名为 `implementation` 的 Configuration 与 library 项目中的某个 Confiugration 关联了起来。在介绍它们是如何关联之前先来看一下在引入 `java-library` 插件后 Project 中 Configuration 之间的依赖关系图：

<img src="/images/202103311435.png" alt="java-library COnfiguration 依赖关系图"  />

上图列出了 `java-library` 中所有的 Configuration 及与其关联的 Attribute，例如 `compileClasspath` 拥有如下 Attribute：

* org.gradle.category = library
* org.gradle.usage = java-api
* org.gradle.libraryelements = jar
* org.gradle.dependency.bundling = external

当 `compileClasspath `开始解析时会遍历解析它继承的所有 Configuration，例如`implementation`。而当 `implementation` 解析时由于其引入了 Project 类型的依赖，那么就会寻找 library 项目中 Attribute 匹配并且  `isCanBeConsumed` 属性为 true 的 Configuration 作为其继承的 Configuration，从依赖图中可以看到该 Configuration 是 `apiElements`。我们在仔细看一下 `apiElements` 的继承关系，会发现它只继承了 `api `但是并没有继承 `implementation`，这也就是为什么在 library 中通过 implementation 引入的依赖无法在依赖了 library 的项目中直接使用的原因。

## 如何兼容现有的生态

大家都知道一个 `com.android.application` 的 Project 可以引入 `java-library` 类型的 Project 作为其依赖项，但是反过来就不行。这是因为 Android Gradle Plugin 对 Java 生态进行了兼容处理，反过来 Java 却没有针对 Android 的生态进行兼容处理。 

同 Configuration 一样依赖也会有 Attribute，对于 java-library 的 Project 而言其产物会有一个 Attribute 为 `artifactType = jar`，而对于 com.application.library 的 Project 而言其产物会有一个 Attribute 为 `artifactType = aar`，而 Android 类型的 Project 编译时所需要的类型为 `artifactType = processed-jar`。在 Android Gradle Plugin 中会注册一些 Transform 用来处理这些产物在不同类型间进行变换，代码片段如下：

``` java
final DependencyHandler dependencies = project.getDependencies();

/*
 * 将 artifactType=jar 的产物转换成 artifactType=processed-jar 的产物
 */
dependencies.registerTransform(
  JetifyTransform.class,
  spec -> {
    spec.getParameters().getProjectName().set(project.getName());
    spec.getParameters().getBlackListOption().set(jetifierBlackList);
    spec.getFrom().attribute(ARTIFACT_FORMAT, JAR.getType());
    spec.getTo().attribute(ARTIFACT_FORMAT, PROCESSED_JAR.getType());
  });

/*
 * 将 artifactType=aar 的产物转换成 artifactType=processed-aar 的产物
 */
dependencies.registerTransform(
  IdentityTransform.class,
  spec -> {
    spec.getParameters().getProjectName().set(project.getName());
    spec.getFrom().attribute(ARTIFACT_FORMAT, AAR.getType());
    spec.getTo().attribute(ARTIFACT_FORMAT, TYPE_PROCESSED_AAR);
  });

/*
 * 将 artifactType=processed-aar 的产物转换成 artifactType=processed-jar、android-res 等产物
 */
for (ArtifactType transformTarget : AarTransform.getTransformTargets()) {
  dependencies.registerTransform(
    AarTransform.class,
    spec -> {
      spec.getParameters().getProjectName().set(project.getName());
      spec.getParameters().getTargetType().set(transformTarget);
      spec.getParameters().getSharedLibSupport().set(sharedLibSupport);
      spec.getParameters()
        .getAutoNamespaceDependencies()
        .set(autoNamespaceDependencies);
      spec.getFrom().attribute(ARTIFACT_FORMAT, EXPLODED_AAR.getType());
      spec.getTo().attribute(ARTIFACT_FORMAT, transformTarget.getType());
    });
}

/*
 * 将 artifactType=processed-jar 的产物转换成 artifactType=android-classes 的产物
 */
for (String classesOrResources :new String[] {ArtifactType.CLASSES.getType(), ArtifactType.JAVA_RES.getType()}) {
  dependencies.registerTransform(
    IdentityTransform.class,
    spec -> {
      spec.getFrom().attribute(ARTIFACT_FORMAT, PROCESSED_JAR.getType());
      spec.getTo().attribute(ARTIFACT_FORMAT, classesOrResources);
    });
}
```

在设置 JavaCompile 任务时会将 artifactType=android-classes 的产物文件赋值给 classpath 参数：

``` kotlin
fun JavaCompile.configureProperties(scope: VariantScope) {
    val compileOptions = scope.globalScope.extension.compileOptions

    this.options.bootstrapClasspath = scope.bootClasspath
    this.classpath = scope.getJavaClasspath(COMPILE_CLASSPATH, CLASSES) // 使用 artifactType=android-classes 的产物文件

    this.sourceCompatibility = compileOptions.sourceCompatibility.toString()
    this.targetCompatibility = compileOptions.targetCompatibility.toString()
    this.options.encoding = compileOptions.encoding
}
```

scope.getJavaClasspath 的主要逻辑就是从 compileClasspath Configuration 中获取到指定 artifactType 的文件，简化后的核心逻辑：

``` kotlin
fun getArtifactFileCollection(project: Project, artifactType: String): FileCollection {
    val compilerClasspath = project.configurations.getByName("compileClasspath")
    return compilerClasspath
        .incoming
        .artifactView { config ->
            config.attributes { container ->
                container.attribute(Attribute.of("artifactType", String::class.java), artifactType)
            }
        }
        .artifacts
        .artifactFiles
}
```

# 参考文档

[Gradle User Manual](https://docs.gradle.org/current/userguide/userguide.html)