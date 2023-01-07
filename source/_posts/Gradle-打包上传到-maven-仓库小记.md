---
title: Gradle 打包上传到 maven 仓库小记
date: 2019-02-18
tags:
- Gradle
---

> 本文已过时，不再推荐使用 maven 插件发布产物，推荐使用 maven-publish 插件。
> # 参考资料：
> [Maven Publish Plugin](https://docs.gradle.org/current/userguide/publishing_maven.html)
> [使用 Maven Publish 插件](https://developer.android.com/studio/build/maven-publish-plugin?hl=zh-cn)

在 Android 开发中，说到 maven 仓库大家一定都不陌生，每当我们需要使用第三方的库的时候，最方便的添加依赖的方法如下：

> implementation "io.reactivex.rxjava2:rxjava:2.x.y"  

只需在 dependences 代码块中添加一行代码，gradle 脚本便会自动从配置的 maven 仓库（Android 默认配置为 JCenter）中下载相应的依赖。  
本文将介绍如何打包上传我们自己写的库代码，也能像上面一样方便的进行引用。

# 上传至本地仓库
首先，在 build.gradle 中我们需要添加 maven 插件，maven 插件会为我们添加一个 install 任务，该任务可以自动生成 pom 配置并打包上传至本地 maven 仓库（默认路径：%HOME/.m2）。  
依赖描述主要由三部分组成：groupId、artifactId、version。用上文提到的 RxJava 举例的话：groupId 是 “io.reactivex.rxjava2”，artifactId 是 “rxjava”，version 是 "2.x.y"。  
于是为了能正确引用到依赖，我们需要正确的配置上面三个参数，下面将通过一个 Java 库来演示，在 build.gradle 文件中添加如下代码：

``` groovy
apply plugin: 'maven'

install {
    repositories.mavenInstaller{
        pom.groupId = 'com.chrnie'
        pom.artifactId = 'java-library'
        pom.version = '1.0.0-SNAPSHOT'
    }
}
```

然后运行命令 `gradle install` 命令，即可在 build/pom 目录下看到插件生成的 pom 描述文件：pom-default.xml，在 build/libs 目录下看到将被打包上传的 jar 文件，并且还在本地 maven 仓库中创建对应的文件。于是，在其它的本地项目中，可以通过添加本地 maven 仓库的配置引用本地 maven 中的库文件。示例代码片段如下：

``` groovy
repositories {
    mavenLocal() 
    ...
}

implementation 'com.chrnie:java-library:1.0.0-SNAPSHOT'
```

需要注意的是，pom 的 groupId、artifactId、version 如果没有配置，那么将依次使用 project.group、project.name、project.version 作为默认值。  
除了上面提到的三个值之外，还有其它许多的参数可以进行配置，例如很多开源项目还会配置 license、developer、scm 等信息，示例模板如下：

``` groovy
pom {
    groupId = ''
    artifactId = ''
    version = ''

    project {
        name ''
        description ''
        url ''
        licenses {
            license {
                name ''
                url ''
                distribution ''
            }
        }
        developers {
            developer {
                id ''
                name ''
                email ''
            }
        }
        scm {
            connection ''
            url ''
            developerConnection ''
        }
    }
}
```

关于 pom 具体可以进行哪些配置，可以参考文档：[MavenPom](https://docs.gradle.org/current/javadoc/org/gradle/api/artifacts/maven/MavenPom.html)。  
由于 maven 插件的 install 任务与 Android 插件不兼容，如果在使用 Android 插件后想使用 install 命令，需要使用第三方的插件 [android-maven-gradle-plugin](https://github.com/dcendents/android-maven-gradle-plugin) 代替 maven 插件，具体配置与 maven 插件相同，仅仅是为了与 Android 插件兼容，同时 pom 配置中默认的 packaging 值由 jar 变为 aar。

# 打包更多的文件
默认情况下，插件只会打包工程生成的直接产物，例如 Java 工程生成的 jar 文件，Android 工程生成的 aar 文件。为了方便开发，IDE 还可以自动解析依赖对应的 *-source.jar 和 *-javadoc.jar 文件以自动匹配源码和文档。例如可以在 Java 工程中添加如下配置自动生成对应的源码和文档打包文件：

``` groovy
task sourcesJar(type: Jar, dependsOn: classes) {
    classifier = "sources"
    from sourceSets.main.allSource
}

task javadocJar(type: Jar, dependsOn: javadoc) {
    classifier = "javadoc"
    from javadoc.destinationDir
}
```

和通过 artifacts 代码块配置需要额外打包的文件项：

``` groovy
artifacts {
    archives sourcesJar
    archives javadocJar
}
```

这样就可以自动生成带有相应源码和文档的打包文件了，运行 `gradle install` 命令后可以在 build/libs 目录下查看所有的将要被打包的文件。

# 上传至远程仓库
maven gradle 插件除了提供 install 任务外，还提供了 uploadArchives 任务用于打包上传至远程的仓库。配置方式与 install 任务类似，示例如下：

``` groovy
uploadArchives {
    repositories {
        mavenDeployer {
            beforeDeployment { MavenDeployment deployment -> signing.signPom(deployment) }

            repository(url: "url/to/remote/release/maven") {
                authentication(userName: 'user_name', password: 'password')
            }
            snapshotRepository(url: "url/to/remote/snapshot/maven") {
                authentication(userName: 'user_name', password: 'password')
            }

            pom{
                ...
            }
        }
    }
}
```

与 install 任务区别主要在可以通过 repository 代码块配置远程 release 版本仓库的信息和通过 snapshotRepository 代码块配置远程 snapshot 版本仓库的信息。值得注意的是，如果想要上传 release 版本至 mavenCenter 仓库，需要对打包的文件进行签名，可以添加 signing 配置并在 mavenDeployer.beforeDeployment 中调用 siging 对打包文件进行签名：

``` groovy
signing {
    required { /* isRelease */ }
    sign configurations.archives
}
```

# 上传至 JCenter
除了 mavenCenter 外，还可以选择将包上传至 JCenter 仓库。目前 Android 创建的项目模板中 JCenter 已经替代 mavenCenter 成为了默认的选择。上传至 JCenter 需要使用 bintray 官方提供的 gradle 插件：[gradle-bintray-plugin](https://github.com/bintray/gradle-bintray-plugin)，然后就是在 build.gradle 中添加如下配置：

``` groovy
bintray {   
    user = 'bintray_user'
    key = 'bintray_api_key'
    pkg {
        repo = ''
        name = ''
        licenses = ['']
        vcsUrl = ''
    }
}
```

示例中提及到的参数都是必填项，并且 bintray 插件的任务依赖于 install 任务，所以还需要按照上文介绍的配置好 install 任务参数。需要注意的是，bintray 插件在上传时会使用 project.group、project.version 值和 pom 配置中的 groupId、version 值做校验，所以建议不在 pom 代码块中配置 groupId、version 参数，而是统一配置 project.group、project.version 参数。而且 version 不能带有类似 SNAPSHOT 字样，只能是纯数字和点组成，否则会上传失败。
因为一般情况下，一个多 module 工程所有的子 module 的 group 和 version 值都是统一的，这里有个小技巧，可以在 rootProject 中配置一次即可：

``` groovy
subprojects {
    version = 'project_version'
    group = 'project_group'
}
```

在完成配置后，即可通过调用 `gradle bintrayUpload` 执行打包上传的任务。

# 参考资料
[Maven Plugin](https://docs.gradle.org/current/userguide/maven_plugin.html)