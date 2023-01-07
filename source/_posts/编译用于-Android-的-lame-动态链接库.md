---
title: 编译用于 Android 的 lame 动态链接库
date: 2019-06-30
tags:
- Android
---
本文介绍主要如何编译出用于 Android 的 lame 动态链接库，但适用于所有的使用了 autoconf 的项目。  

# 构建过程
在介绍具体的操作步骤之前，首先简单了解一下 c 源代码是如何通过编译器变成机器可执行文件的：

1. **预处理**：预处理过程主要处理源代码文件中的以“#”开始的预编译指令。该过程生成的文件一般以 **.i** 作为后缀。

2. **编译**：编译过程就是把预处理完的文件进行一系列词法分析、语法分析、语义分析及优化后生产相应的汇编代码文件。该过程生成的文件一般以 **.s** 作为后缀。

3. **汇编**：汇编过程将汇编代码转变成机器可以执行的指令。该过程生成的文件一般以 **.o** 作为后缀。

4. **链接**：链接过程将输出可执行文件或共享目标文件。共享目标文件即是本文所需要生成的最终文件形式，一般以 **.so** 作为后缀。

# 工具链
介绍完了基础的构建过程，对 c 源代码如何变成可执行文件应该有了大概的认识。在构建过程中，每一步都有相应的工具帮助我们完成转换，这些工具的集合被称为工具链。同样的，我们需要编译出用于 Android 系统的上的动态链接库，就需要使用到官方给出的特定的工具链用于完成转换。可以到 [NDK Downloads](https://developer.android.com/ndk/downloads) 中进行下载。需要注意的是，从 r18 版本开始官方移除了 gcc 的支持，可能会对一些项目的编译造成影响，不熟悉的最好使用旧版本的 ndk 完成编译，本文将使用 r16b 版本完成此次编译。  
完成下载后，通过执行以下命令可以自动生成用于编译指定环境的工具链：

``` bash
path/to/ndk/tools/make_standalone_toolchain.py \
    --arch arm64 --api 21 --install-dir ./toolchain
```
顺利执行后，将在当前目录下生成了一个新目录 toolchain，里面是用于编译可在 arm64-v8a，minSdk 版本为 21 的机器上运行的完整工具链。

# 源码下载
工具准备好了，接下来就开始准备原材料。到 [lame](http://lame.sourceforge.net/download.php) 官网即可下载到最新的源代码。下载完后解压备用即可。

# 配置和编译
进入 lame 源码目录，执行：
``` bash
./configure -h
```
将会打印出所有配置可选项，根据自身的需要进行配置即可。为了简化多次进行编译的流程，一般会编写 shell 脚本将配置和执行命令打包，例如适用于我的需求的完整编译脚本如下：
``` bash
#!/bin/bash

BASE_PATH=$(cd `dirname $0`; pwd)

export PATH=$PATH:`pwd`/toolchain/bin

TARGET_HOST=arm-linux-androideabi
export CC=$TARGET_HOST-gcc

export CFLAGS="-fPIE -fPIC -Os -D**ANDROID_API**=21"
export LDFLAGS="-pie"

./configure \
--host=$TARGET_HOST \
--enable-static=no \
--disable-frontend \
--prefix=$BASE_PATH/out/

make clean
make install
```
该脚本禁止了可执行程序和静态链接库生成，因为这些都是我不需要的。将脚本保存在 lame 根目录，toolchain 也拷贝到 lame 根目录之后，执行脚本即可在 out/lib 目录中看到 libmp3lame.so 共享目标文件以及在 out/inlucde/lame 目录中看到 lame.h 头文件。

需要注意的是，CFLAGS 一定要加上 `D**ANDROID_API**` 参数，值与生成工具链时指定的 minSdk 相同，不然可能会出现运行时异常：

```java
java.lang.UnsatisfiedLinkError: dlopen failed: could not load library "libmp3lame.so" needed by "libmp3encoder-lib.so"; caused by cannot locate symbol "stderr" referenced by "libmp3lame.so"...
```

这是工具链本身的存在问题，官方给出的解决方案，具体可以详见：[Changelog r14](https://github.com/android/ndk/wiki/Changelog-r14)。

# 参考

[Standalone Toolchains](https://developer.android.com/ndk/guides/standalone_toolchain.html#building_open_source_projects_using_standalone_toolchains)