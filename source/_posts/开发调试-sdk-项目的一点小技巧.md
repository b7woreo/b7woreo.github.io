---
layout: post
title: 开发调试 sdk 项目的一点小技巧
tags:
  - Gradle
typora-root-url: ../../source
date: 2021-05-23 15:53:37
---


# 开发调试时遇到的麻烦

以一个常见的 sdk 项目结构为例，该项目中包含了三个模块：

1. common：通用的模块，不直接对外暴露。
2. library：对外暴露的 sdk 实现模块，会依赖 common 模块。

1. sample：用于演示 library 的用法，会依赖 library 模块。

一般 sdk 最终会完成打包并发布至 maven 仓库中方便其他业务开发使用，假定当前项目的 `Group ID = com.sdk` 、 `Version = 1.0.0` ，向 maven 仓库发布后会得到两个依赖项：

1. `com.sdk:common:1.0.0`：common 模块发布的产物。
2. `com.sdk:library:1.0.0`：library 模块发布的产物。

于是乎在 Gradle 项目中的结构就会是下面这个样子：

![](/images/20210521233449.png)

``` groovy
// file: settings.gradle

rootProject.name = "sdk"

include(':common')
include(':library')
include(':sample')
```

``` groovy
// file: build.gradle

ext {
    sdk_group = 'com.sdk'
    sdk_version = '0.0.1'
}
```

``` groovy
// file: common/build.gradle

apply plugin: 'java-library'
```

``` groovy
// file: library/build.gradle

apply plugin: 'java-library'

dependencies {
    implementation("$sdk_group:common:$sdk_version")
}
```

``` groovy
// file: sample/build.gradle

apply plugin: 'java'

dependencies {
    implementation("$sdk_group:library:$sdk_version")
}
```

乍一看上面的项目配置没有任何的问题，非常符合我们的需求。但是一旦想运行 sample 工程进行代码调试时就会发现 library 模块没有完成发布找不到依赖，当想对 library 模块进行发布时又会发现 common 模块没有完成发布找不到依赖。这是最先想到的办法就是把 sample、library 模块对其它模块的依赖先改成 `project` 的形式：

``` groovy
// file: library/build.gradle

apply plugin: 'java-library'

dependencies {
    //implementation("$sdk_group:common:$sdk_version")
    implementation project(':common')
}
```

```groovy
// file: sample/build.gradle

apply plugin: 'java'

dependencies {
    //implementation("$sdk_group:library:$sdk_version")
    implementation project(':library')
}
```

这样一改完确实能够正常运行了，**但繁琐的地方就在于完成需求开发上传代码合入主干发版本前一定要记得把依赖给改回来**，不然通过 maven 引用了这个模块的项目依赖就会解析失败，而且在执行发布时一定要按照依赖的顺序才能完成编译发布。

# 解决方案

上文描述了在开发 sdk 时常遇到的一个痛点问题，实际上 Gradle 早以为这种场景提供了解决方案——DependencySubstitutions。还是以上文中的项目为例，实际上只要在根目录的 `build.gradle` 中添加一段代码即可解决问题：

``` groovy
// file: build.gradle

ext {
    sdk_group = 'com.sdk'
    sdk_version = '0.0.1'
}

// START：新增代码
subprojects { p ->
    p.configurations.all { configuration ->
        rootProject.childProjects
                .findAll { _, sibling -> sibling != p }
                .each { _, sibling ->
                    configuration.resolutionStrategy.dependencySubstitution {
                        substitute module("${sdk_group}:${sibling.name}:${sdk_version}") with project(sibling.path)
                    }
                }
    }
}
// END：新增代码
```

上述代码的主要逻辑就是将所有`GroupId=com.sdk`并且`Version=1.0.0`的依赖项在解析依赖时替换为项目的中对应的`project`。至此当我们不管是修改代码后运行 sample 工程进行代码调试，还是在执行模块发布时都不需要关心依赖的其他本地模块是否已经完成发布了，甚至于在 IDE 中进行代码跳转时都可以直接跳转至源代码中而不是跳转至已发布的 jar 包里。 

# 参考文档

[Customizing resolution of a dependency directly](https://docs.gradle.org/current/userguide/resolution_rules.html)

[DependencySubstitutions](https://docs.gradle.org/current/dsl/org.gradle.api.artifacts.DependencySubstitutions.html)