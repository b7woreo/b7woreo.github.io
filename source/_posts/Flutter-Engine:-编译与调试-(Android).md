---
title: 'Flutter Engine: 编译与调试 (Android)'
date: 2023-05-13
tags:
- Android
- flutter
---

> - 操作系统: macOS 13.3.1
> - flutter sdk: 3.7.12

# 准备工具

下载和编译源码需要用到 `gclient`/`ninja`, 它们包含在 [depot_tools](https://commondatastorage.googleapis.com/chrome-infra-docs/flat/depot_tools/docs/html/depot_tools_tutorial.html#_setting_up) 工具集中.

1. 通过 git 下载拷贝 depot_tools 仓库
    ```
    git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
    ```
2. 将 depot_tools 路径添加到 PATH 环境变量中, 便于后续直接使用
    ```
    export PATH=/path/to/depot_tools:$PATH
    ```

# 下载源码

## 同步代码

1. 创建 engine 目录, 并在 engine 目录中添加 `.gclient` 文件:
    ```
    solutions = [
      {
        "managed": False,
        "name": "src/flutter",
        "url": "https://github.com/flutter/engine.git",
        "custom_deps": {},
        "deps_file": "DEPS",
        "safesync_url": "",
      },
    ]
    ```
2. 执行 `gclinet sync` 命令, 等完同步完成.

## 切换分支

一般情况下是要阅读和编译与 flutter sdk 关联的引擎代码.

1. 执行 `cat path/to/flutter/bin/internal/engine.version` 命令查看引擎 `git-hash`.
2. 进入 `src/flutter` 目录中, 执行 `git checkout <git-hash>`.
3. 执行 `gclient sync -D` 命令, 等待同步完成.


# 编译源码

同步完的代码会保存在 `src/flutter` 目录中, 接下来的构建操作都会在 `src` 目录下进行. flutter 支持在多个平台上运行, 针对不同的平台会生成不同的产物.

## 生成构建文件

在开始构建任务前, 需要通过 `gn` 命令生成构建前所需的文件. 以构建 android-arm 引擎为例, 需要准备 __android-arm__ 和 __host__ 的构建文件.

- android-arm: `./flutter/tools/gn --target-os android --android-cpu arm --runtime-mode debug --unoptimized --no-goma`
- host: `./flutter/tools/gn --runtime-mode debug --unoptimized --no-goma`

__命令参数__:
- `target-os`: 指定目标平台, 支持 android,ios,mac,linux,fuchsia,wasm,win.
- `android-cpu`: 目标平台为 android 时, 指定目标 abi. 支持 arm,x64,x86,arm64.
- `runtime-mode`: 指定运行模式, 支持 debug,profile,release,jit_release.
- `unoptimized`: 关闭优化, 提升构建速度, 在编译 release 产物时不要使用.
- `no-goma`: `goma`是 Google 内部使用的分布式构建工具, 默认会启用该工具, 需要通过 `--no-goma` 禁用.

## 执行构建操作

使用 `gn` 命令会在 `out` 目录下生成对应的构建文件, 接着使用 `ninja` 命令开始指定平台的构建任务.

- android-arm: `ninja -C out/android_debug_unopt`
- host: `ninja -C out/host_debug_unopt`

# 调试源码

## 代码跳转补全

推荐使用 vscode 进行代码阅读与编辑.

### c++

1. 安装插件: [clangd](https://marketplace.visualstudio.com/items?itemName=llvm-vs-code-extensions.vscode-clangd).
2. 把 `compile_commands.json` 添加到 `src/flutter` 目录下: 
    ```bash
    ln -s ../out/android_debug_unopt/compile_commands.json .
    ```
3. 用 vscode 打开 `src/flutter` 目录.

### Java

1. 安装插件: [Extension Pack for Java](https://marketplace.visualstudio.com/items?itemName=vscjava.vscode-java-pack).
2. android 相关的代码在 `shell/platform/android` 目录中, 在该目录下添加 `.vscode/settings.json` 文件配置依赖路径:
    ```json
    {
      "java.project.sourcePaths": [
        "."
      ],
      "java.project.referencedLibraries": [
        "path/to/engine/src/third_party/android_embedding_dependencies/lib/*.jar",
        "path/to/engine/src/third_party/android_tools/sdk/platforms/android-33/android.jar",
      ],
    }
    ```
3. 用 vscode 打开 `shell/platform/android` 目录.

## 使用本地引擎

使用 `flutter` 命令时, 通过 `--target-platform` 和 `--local-engine` 参数可以指定使用本地编译的引擎, 以编译 apk 为例:

```bash
flutter build apk \
--debug \
--target-platform=android-arm \
--local-engine-src-path=path/to/engine/src \
--local-engine=android_debug_unopt
```

## 断点调试代码

1. 安装插件: [CodeLLDB](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb).
2. 在设备上运行待调试的应用.
3. 在设备中运行 `lldb-server`:
   ```bash
   adb push path/to/lldb-server /data/local/tmp
   adb shell run-as <package-id> sh -c "cp -F /data/local/tmp/lldb-server /data/data/<package-id>/lldb-server"
   adb shell run-as <package-id> sh -c "/data/data/<package-id>/lldb-server platform --server --listen unix-abstract:///data/data/<package-id>/debug.socket"
   ```
   __参数__: 
   - `lldb-server`: 该工具在 `path/to/ndk/toolchains/llvm/prebuilt/<host>/lib64/clang/<clang-version>/lib/linux/<abi>` 目录中可以找到.
   - `package-id`: 待调试的应用包名.
4. 在 vscode 的 `launcher.json` 新增 debug 配置:
   ```json
   {
      "version": "0.2.0",
      "configurations": [
          {
              "name": "flutter_lldb",
              "type": "lldb",
              "request": "attach",
              "pid": <package-pid>,
              "initCommands": [
                  "platform select remote-android",
                  "platform connect unix-abstract-connect:///data/data/<package-id>/debug.socket"
              ],
              "postRunCommands": [
                  "target symbols add path/to/src/out/<local-engine>/libflutter.so"
              ]
          }
      ]
    }
   ```
   __参数__:
   - `package-pid`: 待调试应用的进程号, 可以通过 `adb shell pidof <package-id>` 获取.
   - `package-id`: 待调试的应用包名.
   - `local-engine`: 同 `flutter build` 中的参数.
5. 在 vscode 选择 `flutter_lldb` 配置并运行.

# 参考资料

- [Setting up the Engine development environment](https://github.com/flutter/flutter/wiki/Setting-up-the-Engine-development-environment#using-vscode-as-an-ide-for-the-android-embedding)
- [Compiling the engine](https://github.com/flutter/flutter/wiki/Compiling-the-engine)
- [Flutter's modes](https://github.com/flutter/flutter/wiki/Flutter%27s-modes)
- [Using a locally-built engine with the `flutter` tool](https://github.com/flutter/flutter/wiki/The-flutter-tool#using-a-locally-built-engine-with-the-flutter-tool)
- [Remote Debugging](https://lldb.llvm.org/use/remote.html)