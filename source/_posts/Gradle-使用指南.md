---
title: Gradle 使用指南
date: 2018-03-24
typora-root-url: ../../source
tags:
- Gradle
---
# 依赖版本管理
## 使用 config.gradle 配置依赖
当一个 project 中包含多个 module 的时候为了方便，可以在 **project** 根目录新建一个文件叫 **config.gradle** 专门用来管理依赖版本。示例如下： 
``` groovy
ext {
    versionCode = 1
    versionName = "1.0.0"
}

ext {
    buildToolsVersion = "27.0.3"
    compileSdkVersion = 27
    minSdkVersion = 14
    targetSdkVersion = 27
}

def kotlin_version = "1.2.30"

ext.plugin = [
        "android"      : "com.android.tools.build:gradle:3.0.1",
        "kotlin"       : "org.jetbrains.kotlin:kotlin-gradle-plugin:${kotlin_version}",
        "bintray"      : "com.jfrog.bintray.gradle:gradle-bintray-plugin:1.7.3",
        "android-maven": "com.github.dcendents:android-maven-gradle-plugin:1.5",
]

ext.lib = [
        "kotlin-stdlib-jre7": "org.jetbrains.kotlin:kotlin-stdlib-jre7:${kotlin_version}",
        "recyclerview-v7"   : "com.android.support:recyclerview-v7:27.0.2",
        "test-runner"       : "com.android.support.test:runner:0.5"
]
```
## 修改 project 目录下的 build.gradle
在 **project** 的 **build.gradle** 中引用：
``` groovy
buildscript {
    apply from: './config.gradle'

    ...
}

...
```
然后 **project** 中的依赖统一这样配置：
``` groovy
buildscript {
    ...

    dependencies {
        classpath rootProject.plugin["android"]
        classpath rootProject.plugin["kotlin"]
        classpath rootProject.plugin["bintray"]
        classpath rootProject.plugin["android-maven"]
    }
}
...
```
## 修改 module 目录下的 build.gradle 
在 **module** 中的依赖统一这样配置：
``` groovy
...

android {
    compileSdkVersion rootProject.compileSdkVersion
    buildToolsVersion rootProject.buildToolsVersion

    defaultConfig {
        minSdkVersion rootProject.minSdkVersion
        targetSdkVersion rootProject.targetSdkVersion
        versionCode rootProject.versionCode
        versionName rootProject.versionName

        ...
    }

    ...
}

dependencies {
    implementation rootProject.lib["kotlin-stdlib-jre7"]
    implementation rootProject.lib["recyclerview-v7"]
    androidTestImplementation rootProject.lib["test-runner"]
}
```
## 把配置文件托管在服务器上
我上面写的是把配置文件放在本地的，有需求的话可以将 **config.gradle** 文件以静态文件的方式在服务器上发布，即可在多个不同的 **project** 使用同一份配置。例如将配置文件上传到 Github 后，在该文件的浏览页面获取 url：

![](/images/20180324182300.jpg)

把 **project** 的 **build.config** 中引用语句改为：`apply from: 'your/url'` 即可。

# Gradle 下载缓慢
一般情况下不直接使用本地 Gradle 而是使用Gradle wrapper，这样的好处是在修改 Gradle 版本时，只需要修改项目根目录下的gradle.properties文件中的distributionUrl即可。当我指定Gradle的版本为4.0时，gradle.properties内容如下所示：
```
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-4.0-all.zip
```
然而由于网络的原因，导致直接通过
```
https\://services.gradle.org/distributions/gradle-4.0-all.zip
```
下载 **gradle-4.0-all.zip** 十分缓慢。其实我们可以利用百度云等离线下载工具帮助完成下载，下面以下载 **gradle-4.0-all.zip** 举例:
## 第一步
使用下载工具下载 distributionUrl 链接内容。
## 第二步
使用 Terminal 执行任意 gradlew 命令或用 AndroidStudio 打开项目触发 gradle 的下载，此时在 .gradle\wrapper 文件夹中会出现对应版本号的文件夹，在这个文件夹里会有一个哈希值命名的文件夹。例如：
```
.gradle\wrapper\dists\gradle-4.0-all\ac27o8rbd0ic8ih41or9l32mv
```
在出现这些文件夹之后终止 gradle 的下载。
## 第三步
清空哈希值命名的文件夹中的所有内容，把事先下载好的 gradle-4.0-all.zip 放入其中。
## 第四步
再次执行gradlew命令或用AndroidStudio打开项目，就会发现跳过了下载过程。