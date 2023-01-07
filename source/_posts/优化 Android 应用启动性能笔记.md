---
title: 优化 Android 应用启动性能笔记
date: 2019-05-02
tags:
- Android
---
# 启动流程

## 完整流程
完整的应用启动流程分为两个阶段执行：  
1. 系统执行阶段
    1. 加载应用
    2. 显示应用 StartingWindow
    3. 创建应用进程
2. 应用执行阶段
    1. 创建 Application 对象
    2. 启动主线程
    3. 创建 Activity 对象
    4. 填充视图
    5. 测量、布局视图
    6. 渲染首帧

## 不同场景流程
并不是每次启动时都需要执行完整的启动流程，在不同的场景下启动流程会有不同程度的精简。  

### Cold
该场景下，应用完全被杀死，进程不存在。**启动应用时需要执行完整的启动流程，没有流程被精简**。

### Warm
该场景下又细分出两种场景：  

#### 场景1
该场景下，应用被系统从内存中移除，应用启动时，在 Activity 创建过程中获取到的 savedInstanceState 参数不为空。**应用启动时和 Cold 场景一致，需要执行完整的启动流程**。

#### 场景2 
该场景下，用户退出了所有的 Activity，但是进程还保留在内存中。**应用启动时，流程 1.1、1.3、2.1、2.2 将被跳过**。

### Hot  
该场景下，应用进入后台状态，进程、Activity都依旧保留在内存中。**应用启动时，只会执行流程 1.2、2.6**。

# 性能指标获取

## 首帧渲染耗时

### 指标
首帧渲染耗时包括三个指标：

0. ThisTime：渲染首帧的 Activity 创建到完成第一帧绘制的耗时
0. TotalTime：进程创建耗时 + Application 创建耗时 + 无界面 Activity 耗时 + ThisTime
0. WaitTime：AMS 调度耗时 + 发起跳转的 Activity pause 耗时 + TotalTime

TotalTime 中提到的“无界面 Activity 耗时”指的是没有调用 setContentView 方法，用于完成中转的 Activity。例如一个 SplashActivity 在 onCreate 方法中选择应用启动时需要跳转到的 Activity，执行 startActivity 方法后调用 finish 方法结束自己，这段时间会被计入 TotalTime 中。一般情况下应用开发者主要关注 TotalTime 指标。

### 获取方式
一共有两种方式可以获取到上述指标。

#### Logcat
当应用启动后，ActivityManager 会在 Logcat 中输出一条日志，格式形如：
```
 ActivityManager: Displayed {application id}/{activity}: {this time} (total {total time})
```

#### adb shell
通过执行 adb 命令：
```
adb [-d|-e|-s <serialNumber>] shell am start -S -W
{application id}/{activity}
-c android.intent.category.LAUNCHER
-a android.intent.action.MAIN
```
即可让应用执行完整的应用启动流程，并会输出如下结果：
```
Starting: Intent
Status: ok
Activity:  {application id}/{activity}
ThisTime:  {this time}
TotalTime: {total time}
WaitTime:  {wait time}
Complete
```

## 完整绘制耗时
现在应用都会在启动时通过异步的方式加载数据，在数据加载完整前会显示一个 loading 界面，在数据加载完成后才会显示完整的页面，此时用户也才真正的可以开始于应用进行交互。为了测量应用启动到应用完成完整的页面绘制的耗时，需要手动在完成数据加载和渲染后调用如下方法：`Activity.reportFullyDrawn()`。此时，ActivityManager 会在 Logcat 中输出如下日志：

```
ActivityManager: Fully drawn {application id}/{activity}: {full drawn time}
```

## 函数运行耗时
为了找出应用启动时的可优化点，需要测量函数运行的耗时。测量函数运行耗时有两种方法可供选择。

### Android Profiler
该方法通过三个 adb 命令完成信息采集。

0. 开始统计：`adb shell am start -n {application id}/{activity} --start-profiler {trace file}`
0. 结束统计：`adb shell am profile stop`
0. 获取输出文件：`adb pull {trace file}`

在获取到输出文件后，打开 Android Studio 中的 Android Profiler，通过“Load from file”选项打开新的 Session 便可以分析函数运行耗时信息。
由于该方式使用的是采样的方式获取函数运行信息，如果函数的执行时间过短可能导致不能采集到该函数执行相关信息。为了保证一些函数的信息一定能被能被获取到，可以手动在代码当中插入如下代码：

``` java
Debug.startMethodTracing() // 开始跟踪函数
Debug.stopMethodTracing() // 结束跟踪函数
```

### systrace

该方法通过 systrace 软件完成信息采集，在使用之前需要安装只要的依赖软件，详情参考官方文档：[systrace](https://developer.android.com/studio/command-line/systrace)。
通过执行如下命令开始获取统计数据：

```
systrace -a {application id} [category]
```
其中的 category 参数在不同的设备上支持情况不一样，可以通过 `systrace -l` 命令获取当前连接的设备支持的 category 列表，当需要停止时在命令行中按下“回车键”即可。

结束统计后，该软件会输出一个 html 文件，在浏览器中直接打开该文件即可获取到函数运行耗时信息。不过默认情况下获取到的信息量很少，想要测量更多的函数运行耗时数据需要手动在代码中插入如下代码：

``` java
Trace.beginSection() // 开始跟踪函数
Trace.endSection() // 结束跟踪函数
```

systrace 的插桩对应用性能影响最小，一般情况下优先选择 systrace 作为函数耗时分析工具。

# 参考
[App startup time](https://developer.android.com/topic/performance/vitals/launch-time)