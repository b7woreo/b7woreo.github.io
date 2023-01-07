---
title: aosp 源码学习环境搭建
date: 2021-06-08
typora-root-url: ../../source
tags: 
- Android
- aosp
---

对于 Android 应用开发工程师来说想要技术上更上一层楼，那么阅读和学习 aosp 代码可以说是一个必经之路，因为通过学习 aosp 代码能更好的理解 Android 系统整个运作机制，从而指导我们更好的设计应用技术方案。

虽然现在谷歌搭建好了在线的源码阅读平台：[Android Code Search](https://cs.android.com/?hl=zh-cn)，但是使用体验不如人意。而且纸上得来终觉浅，只有实际修改代码运行才会有更深的理解，这里我将分享我的 aosp 学习环境搭建的过程供大家参考。

# 环境准备

最好是有一台运行 Linux 系统的电脑，Linux 的发行版本不做限制依据个人喜好自行选择。本文的所有操作均是在 Linux 环境下完成，在其他操作系统上可能会不奏效。

即使宿主电脑已经是运行 Linux 系统了依然不建议直接使用宿主电脑执行编译操作，因为你很有可能会在编译过程中遇到一些莫名其妙的问题。而根据[官方文档](https://source.android.com/setup/build/initializing)的说法，谷歌会保证 aosp 是能够在 Ubuntu 14.04 正常完成编译的：

> The Android build is routinely tested in house on Ubuntu LTS (14.04) and Debian testing. Most other distributions should have the required build tools available.

所以强烈推荐大家使用 docker 部署一个专用的编译环境，官方的 docker 镜像部署及使用指南可以参考：[tools/docker/README.md](https://android.googlesource.com/platform/build/+/master/tools/docker)。不过官方的 Dockerfile 不一定能正常构建，因为 [tools/docker/Dockerfile](https://android.googlesource.com/platform/build/+/master/tools/docker/Dockerfile) 会在构建镜像时会下载 repo 工具而且会对其 sha256 进行校验。repo 版本是一直在更新的，而 Dockerfile 没有更新，所以可能会出现校验失败的问题，这个就需要自行更新哈希值或移除校验步骤。所以官方给的 Dockerfile 只是一个参考，大家可以根据自身的需要进行修改，增加或移除一些不必要的构建过程。

这里我给大家贴出我修改后的 Dockerfile ：

``` dockerfile
FROM ubuntu:18.04

ARG userid
ARG groupid
ARG username

RUN apt-get update

# Install required packages, see: https://source.android.com/setup/build/initializing#installing-required-packages-ubuntu-1804
RUN apt-get install -y git-core gnupg flex bison build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig

# Instal addition required packages
RUN apt-get install -y openjdk-8-jdk rsync python python3

# Install repo
RUN curl -o /usr/local/bin/repo https://mirrors.tuna.tsinghua.edu.cn/git/git-repo \
 && chmod a+x /usr/local/bin/repo

# Install my packages
RUN apt-get install -y zsh

# Create group & user
RUN groupadd -g $groupid $username \
 && useradd -m -u $userid -g $groupid $username \
 && echo $username >/root/username

# Setup env
ENV HOME=/home/$username
ENV USER=$username
ENV DISPLAY=:0

ENTRYPOINT chroot --userspec=$(cat /root/username):$(cat /root/username) / /bin/zsh -i
```

由于 Ubuntu14.04 安装 jdk8 不方便，最终我还是选择了 Ubuntu18.04 这个发行版本。相较于与官方的 Dockerfile 的不同就是使用了清华的镜像并且安装了 zsh 替代 bash。除此之外比较重要的一点是设置了 `DISPLAY` 环境变量，这样就可以在容器内直接运行 GUI 程序，例如：Android Emulator。

通过 `docker build --build-arg userid=$(id -u) --build-arg groupid=$(id -g) --build-arg username=$(id -un) -t android-build-trusty .` 命令即可完成镜像的编译，编译完成后通过如下命令启动容器环境：

``` bash
docker run -it --rm \
  --network=host \
  --privileged \
  -v $(echo ~):$(echo ~) \
  android-build-trusty
```

这里给大家简单说明一下各个参数的作用：

* `-it --rm`：容器环境是一次性的，推出后会自动销毁，再此进入又将是全新的。
* `--network=host`：共享宿主机器的网络环境，主要用途就是宿主机器可以直接访问容器内启动的模拟器。
* `--privileged`：容器可以访问宿主机器的所有硬件设备，例如当在容器内使用 x86 镜像启动模拟器时需要访问 /dev/kvm 设备。
* `-v $(echo ~):$(echo ~)`：将宿主机器的 `$HOME` 目录挂载到容器的 `$HOME` 目录，一是可以方便的访问各种资源，二是一般情况下 zsh 的配置保存在 `~/.zshrc` 中，这样可以保证在容器内 zsh 的使用体验是和在宿主机器中一致的。

# 源码下载

## repo

aosp 项目是由一系列的 git 仓库组成的，在下载 aosp 源码前首先需要下载一个名叫：[repo](https://source.android.com/setup/develop?hl=zh-cn#repo) 的软件，该软件主要用途是为了简化对 aosp 代码的管理操作流程。由于众多因素的影响，推荐大家使用清华的镜像下载，相关的使用文档可以参考：[Git Repo 镜像使用帮助](https://mirrors.tuna.tsinghua.edu.cn/help/git-repo/)。可以把 repo 的安装放在 Docker 镜像的构建流程中一并完成。

## aosp

### 配置代理

装好 repo 后便可以开始 aosp 源码下载，这里同样推荐大家使用清华的源镜像进行下载，只需要在 git 配置中设置把 https://android.googlesource.com 链接替换为 ：https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/ 即可无感知的使用镜像：

``` bash
git config --global url.https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/.insteadof https://android.googlesource.com
```

### 下载源码

在 [android/platform/manifest](https://android.googlesource.com/platform/manifest/+refs) 中挑选好想要下载的源码版本的分支，例如：android-10.0.0_r1，然后执行以下命令完成工作目录初始化：

``` bash
mkdir WORKING_DIRECTORY
cd WORKING_DIRECTORY
repo init -u https://android.googlesource.com -b android-10.0.0_r1
```

接着只需要在工作目录中执行 `repo sync` 即可开始源码下载，在下载过程中可能会出现一些异常中断的情况，只要多运行几次 `repo sync` 命令最终都能成功完成代码同步。

# 编译运行

在 docker  环境中的源码根目录下执行以下命令即可开始编译：

``` bash
. build/envsetup.sh
lunch aosp_x86_64-eng
m
```

事例中编译的系统版本变种是：aosp_x86_64-eng，因为 x86_64 在模拟器中有较高的运行效率，方便我们直接在模拟器中测试编译好的系统。成功完成编译后执行以下命令即可通过模拟器运行编译好的系统：

``` bash
. build/envsetup.sh
lunch aosp_x86_64-eng
emulator -writable-system
```

在通过 `emulator` 命令启动虚拟机时增加了 `-writable-system` 参数，这个参数的作用会在下文再详细解释。

# 代码修改

## 目录结构

在介绍如何编辑代码前首先需要了解一下 aosp 项目的整个目录结构，理论上只要知道了目录的构成情况后就可以在任何的编辑器中索引、编辑代码了（需要为对应的编辑器写好项目结构的配置文件）。

对于 Android 应用开发人员而言，一般我们需要阅读学习的是 framework 层的代码。这一层级的代码主要由 java 实现，包括了 AndroidSdk 中提供的接口和系统服务的实现代码，主要包括以下几个目录的代码：

* `frameworks/base/core`：AndroidSdk API
* `frameworks/base/graphics`：AndroidSdk 图形绘制相关 API 
* `frameworks/base/services`：Android 系统服务
* `libcore/ojluni`：Java 标准 API

## 使用 Android Studio 编辑

因为 aosp 的目录结构比较特殊，Android Studio 没办法识别对于的模块及代码，需要在成功完成一次全量编译后，在源码根目录下运行：`mmm development/tools/idegen && development/tools/idegen/idegen.sh` 命令使得在源码根目录下生成 android.ipr 文件，用 Android Studio 直接打开该文件才能让 Android Studio 知道如何对代码进行索引以支持跳转功能，参考文档见：[/tools/idegen/README](https://android.googlesource.com/platform/development/+/refs/heads/master/tools/idegen/README)。

如果以上步骤操作完成后在 Android Studio 中还是不能正常进行代码跳转，需要逐个确认以下配置是否正确：

1. 在新版本的 Android Studio 中打开 android.ipr 时会提示该工程结构已过时，需要进行转换。点击 `Convert` 按钮后 `android.iml` 文件内容会发生变化：

![](/images/20210530122008.png)

其中 `sourceFolder` 全部被标记成了 `kotlin-test` 或者 `kotlin-source`，我们需要将其全部替换为 `java-test` 或者 `java-source`。

2. 确保移除了所有的外部依赖，形如下面两张图所示：

![](/images/20210530122652.png)

![](/images/20210530122652.png)

3. 在 **Edit Custom Properties** 中添加了 `idea.max.intellisense.filesize=100000` 配置，不如无法正确的索引 R 文件。

准备好编辑环境后，就可以开始尝试修改代码了。首先可以尝试增加一行日志，接着直接在源码根目录执行 `m` 命令即可开始增量编译。增量编译的时间相较于全量编译会有很大的缩短，不过相较于应用编译的时间还是比较长的。当编译完成后再次通过 `emulator` 打开模拟器运行的就是已经被修改过的代码了。

## 修改 API 接口

如果有修改公开的 API 接口，那么需要运行：`make update-api` 命令以更新 API 接口文档，否则编译会出错。

# 断点调试

在开发过程中不可避免的会使用到断点调试功能，这个功能在阅读学习代码的过程中也同样重要，通过断点堆栈可以轻松方便的了解方法调用的整个链路情况。在断点调试系统服务代码时需要使用到 AndroidStudio。首先需要将项目的 Sdk 改成 Android API（如下图所示），否则 AndroidStudio 可能无法正常识别到设备。

![](/images/20210606173123.png)

![](/images/20210606173918.png)

接着点击 **Attach Debugger to Android Process** 并在弹窗中选择 **system_process** 进程：

![](/images/20210606174138.png)

最后在合适的地方加上断点并执行程序运行到该位置即可，例如在 `ActivityTaskManagerService#startActivity` 添加断点，在 Activity 的启动过程中就会出发中断，可以在 AndroidStudio 中完成单步调试：

![](/images/20210607074543.png)