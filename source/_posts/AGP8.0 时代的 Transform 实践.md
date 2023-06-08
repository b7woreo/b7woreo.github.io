---
title: 'AGP8.0 时代的 Transform 实践'
date: 2023-06-03
tags:
- Android
- gradle
- AGP
---

众所周知, `Transform` 接口在 AGP8.0 中已经被移除了, 而以往的大量工程实践都需要使用该接口结合 ASM 对字节码进行操作, 例如: 代码插桩, 代码替换等.

接下来将探索在 AGP8.0 中有哪些方案可以替代 `Transform` 达到相同的效果.

# 一. Instrumentation

[Instrumentation](https://developer.android.com/reference/tools/gradle-api/8.0/com/android/build/api/variant/Instrumentation) 作为 Google 首推的 `Transform` 替代方案, 只需要实现 [AsmClassVisitorFactory](https://developer.android.com/reference/tools/gradle-api/8.0/com/android/build/api/instrumentation/AsmClassVisitorFactory) 接口即实现对字节码处理.

**原 `Transform` 处理流程:**

[![](https://mermaid.ink/img/pako:eNp1kL0KAkEMhF9lSaWgiJZXWFmJNmpnLMJtTg9us8duFhHx3V1PxT_shswMX5IzlN4yFFA1_lgeKKhZrFBQammTbucURmVDMXLcoWggiZUPbtzbbp5617-FlYNjW5Py-F9p8lXySX8RD7AZDqfmhXtHd9YH8Iv_2Z28b9BZdy4KDMDlFtU2n39GMQZBD-wYocjSckWpUQSUS45SUr8-SQmFhsQDSK3NsFlN-0AOioqamKd5BfVheX9p99nLFXXigNI?type=png)](https://mermaid.live/edit#pako:eNp1kL0KAkEMhF9lSaWgiJZXWFmJNmpnLMJtTg9us8duFhHx3V1PxT_shswMX5IzlN4yFFA1_lgeKKhZrFBQammTbucURmVDMXLcoWggiZUPbtzbbp5617-FlYNjW5Py-F9p8lXySX8RD7AZDqfmhXtHd9YH8Iv_2Z28b9BZdy4KDMDlFtU2n39GMQZBD-wYocjSckWpUQSUS45SUr8-SQmFhsQDSK3NsFlN-0AOioqamKd5BfVheX9p99nLFXXigNI)

**现 `Instrumentation` 处理流程:**

[![](https://mermaid.ink/img/pako:eNplkDELwjAQhf9KuEnBInbM4KKT6KLg0jgczUUDTVKSCyKl_91qBStux_uO9-5eB3XQBBJME-71DSOL_VF55a1vM1c7jMu6wZQoXZTniD6ZEN1qVm1e6tkmyyFe5hNW_rOQ-d_sEyGKYi2-xtOQX1ROM95otIUFOIoOrR6-6JQXQgHfyJECOYyaDOaGFSjfD6uYOZwevgbJMdMCcquRaWvxGtGBNNikQSX9Ov0wNvMuqH8CB0xreg?type=png)](https://mermaid.live/edit#pako:eNplkDELwjAQhf9KuEnBInbM4KKT6KLg0jgczUUDTVKSCyKl_91qBStux_uO9-5eB3XQBBJME-71DSOL_VF55a1vM1c7jMu6wZQoXZTniD6ZEN1qVm1e6tkmyyFe5hNW_rOQ-d_sEyGKYi2-xtOQX1ROM95otIUFOIoOrR6-6JQXQgHfyJECOYyaDOaGFSjfD6uYOZwevgbJMdMCcquRaWvxGtGBNNikQSX9Ov0wNvMuqH8CB0xreg)

相较于 `Transform` 处理流程, `Instrumentation` 流程免去对中间产物的读写, 并且一定是以 Jar/classes 文件为单位的增量处理, 使得全量编译和增量编译速度都有较大的提升.

## 插桩实践

下面以实现一个的 `TraceAsmClassVisitorFactory` 为例, 实践如何使用 `Instrumentation` 相关接口.

`TraceAsmClassVisitorFactory` 将在所有方法入口和出口插桩 `Trace#beginSection` 和 `Trace#endSection` 代码.

### TraceClassVisitor

```kotlin
class TraceClassVisitor(
    api: Int,
    next: ClassVisitor
) : ClassVisitor(api, next) {

    private var className: String? = null
    
    // 开始处理 class 文件时, 记录当前 class 的名字
    override fun visit(
        version: Int,
        access: Int,
        name: String?,
        signature: String?,
        superName: String?,
        interfaces: Array<out String>?
    ) {
        className = name
        super.visit(version, access, name, signature, superName, interfaces)
    }

    // 遍历 method 时, 使用 MethodVisitor 加入插桩逻辑
    override fun visitMethod(
        access: Int,
        name: String?,
        descriptor: String?,
        signature: String?,
        exceptions: Array<out String>?
    ): MethodVisitor {
        val mv = super.visitMethod(access, name, descriptor, signature, exceptions)
        return TraceMethodVisitor(api, mv, access, name, descriptor)
    }

    private inner class TraceMethodVisitor(
        api: Int,
        methodVisitor: MethodVisitor?,
        access: Int,
        name: String?,
        descriptor: String?
    ) : AdviceAdapter(
        api,
        methodVisitor,
        access,
        name,
        descriptor
    ) {
        
        // method 入口插入 `Trace#beginSection` 代码
        override fun onMethodEnter() {
            mv.visitLdcInsn("$className#$name")
            mv.visitMethodInsn(
                Opcodes.INVOKESTATIC,
                "android/os/Trace",
                "beginSection",
                "(Ljava/lang/String;)V",
                false
            )
        }

        // method 出口插入 `Trace#endSection` 代码
        override fun onMethodExit(opcode: Int) {
            mv.visitMethodInsn(
                INVOKESTATIC,
                "android/os/Trace",
                "endSection",
                "()V",
                false
            )
        }

    }
}
```

### TraceAsmClassVisitorFactory

```kotlin
abstract class TraceAsmClassVisitorFactory: AsmClassVisitorFactory<InstrumentationParameters.None> {
    
    // 对所有 class 文件都进行插桩处理
    override fun isInstrumentable(classData: ClassData): Boolean {
        return true
    }
    
    // 遍历 class 时, 使用 ClassVisitor 加入插桩逻辑
    override fun createClassVisitor(
        classContext: ClassContext,
        nextClassVisitor: ClassVisitor
    ): ClassVisitor {
        val apiVersion = instrumentationContext.apiVersion.get()
        return TraceClassVisitor(apiVersion, nextClassVisitor)
    }

}
```

### TraceTransformPlugin

```kotlin
abstract class TraceTransformPlugin : Plugin<Project> {

    override fun apply(project: Project) {
        project.plugins.withId("com.android.application") {
            project.extensions.configure(ApplicationAndroidComponentsExtension::class.java) { extension ->
                extension.onVariantsAsmClassVisitorFactory { variant ->
                    // 注册用于插桩的 AsmClassVisitorFactory
                    variant.instrumentation.transformClassesWith(
                        TraceAsmClassVisitorFactory::class.java,
                        InstrumentationScope.ALL
                    ) {}

                    //  由于对 method 进行了插桩, 需要重新计算被插桩函数的栈帧
                    variant.instrumentation.setAsmFramesComputationMode(FramesComputationMode.COMPUTE_FRAMES_FOR_INSTRUMENTED_METHODS)
                }
            }
        }
    }
    
}

```

## 优劣势

*   **优势:** 使用简单, 只需关注 `ClassVisitor` 逻辑, 无需处理 class 文件的读取与写入.
*   **劣势:** class 处理过程是固定的, 只能实现相对简单的代码插桩与替换, 无法满足个性化需求.

# 二. Transform Task

通过 `Instrumentation` 确实能很方便的实现插桩与替换逻辑, 但是理想很丰满，现实很骨感, 很多时候并不能满足需求, 例如:

*   **先分析再操作:** 在实现 hook 框架时, 通常需要先对工程中的所有 class 进行一次遍历分析后, 再对相关的 class 文件进行字节码操作.

*   **输出文件产物:** 上文中的 `TraceClassVisitor` 在调用 `Trace#beginSection` 时传入了完整的类名与方法名, 这往往会导致生成的 trace 文件过大. 工程实践中一般会 生成 `methodId` 与 `类名#方法名` 的映射, 在 `Trace#beginSection` 调用时仅仅传入 `methodId`, 这时就需要在插桩结束后保存当前的 `methodId` 与 `类名#方法名` 的映射文件.

当对 class 的处理过程有自定义需求时, 就需要通过 `Transform Task` 来实现. `Transform Task` 通过自己实现一个 `Gradle Task`, 然后使用 `ApplicationAndroidComponentsExtension` 提供的接口将其注册到 class 的处理流程中.

**class 处理流程:**

[![](https://mermaid.ink/img/pako:eNptksFqwzAMhl_F-NRB-wI5DMZ2GttlK-wQhyBsZfEa28GWGaP03WcnS-fQXIwlfZJ-yT5z6RTyineD-5Y9eGIvb8IKG1z0Ems5QAgYGmEVjpgOKzWG-gt89mlL6A0qDYTtH1qkrMI5ZclzkcZI2WpyLwimnfvtavJgQ-e8eZzLfGjqH4Jp7masVFHAz0l4iV4DLUE47erjYrNjsjOxTMgOh3v2L6AUM4W2Rlwv41qhdN6qva2Wl5GFbLWY6PUUG_vcpNb2hMz75ntuUj5old77LCxjglOPBgWv0lVhB3EgwYW9JBQiufcfK3lFPuKex1Gltk8aPj0YXnUwhORNYsj51_kPTV_p8gtJG-Gw?type=png)](https://mermaid.live/edit#pako:eNptksFqwzAMhl_F-NRB-wI5DMZ2GttlK-wQhyBsZfEa28GWGaP03WcnS-fQXIwlfZJ-yT5z6RTyineD-5Y9eGIvb8IKG1z0Ems5QAgYGmEVjpgOKzWG-gt89mlL6A0qDYTtH1qkrMI5ZclzkcZI2WpyLwimnfvtavJgQ-e8eZzLfGjqH4Jp7masVFHAz0l4iV4DLUE47erjYrNjsjOxTMgOh3v2L6AUM4W2Rlwv41qhdN6qva2Wl5GFbLWY6PUUG_vcpNb2hMz75ntuUj5old77LCxjglOPBgWv0lVhB3EgwYW9JBQiufcfK3lFPuKex1Gltk8aPj0YXnUwhORNYsj51_kPTV_p8gtJG-Gw)

`Transform Task` 会在 `Instrumentation` 执行完成后再执行, 并且输出的产物是一个包含所有 class 的 jar 文件.

## 插桩实践

下面以实现一个的 `TraceTransformTask` 为例, 调整上文的中的 `TraceClassVisitor`插桩逻辑, 在调用 `Trace#beginSection` 时传入 `methodId` 参数, 并在插桩完成后输出 `methodId` 与 `类名#方法名` 的映射文件.

### TraceClassVisitor

```kotlin
class TraceClassVisitor(
    api: Int,
    next: ClassVisitor,
    private  val mapping: (String) -> Long,
) : ClassVisitor(api, next) {

    private var className: String? = null

    override fun visit(
        version: Int,
        access: Int,
        name: String?,
        signature: String?,
        superName: String?,
        interfaces: Array<out String>?
    ) {
        className = name
        super.visit(version, access, name, signature, superName, interfaces)
    }

    override fun visitMethod(
        access: Int,
        name: String?,
        descriptor: String?,
        signature: String?,
        exceptions: Array<out String>?
    ): MethodVisitor {
        val mv = super.visitMethod(access, name, descriptor, signature, exceptions)
        return TraceMethodVisitor(api, mv, access, name, descriptor)
    }

    private inner class TraceMethodVisitor(
        api: Int,
        methodVisitor: MethodVisitor?,
        access: Int,
        name: String?,
        descriptor: String?
    ) : AdviceAdapter(
        api,
        methodVisitor,
        access,
        name,
        descriptor
    ) {

        override fun onMethodEnter() {
            // 使用 methodId 替代 className#methodName
            val methodId = mapping("$className#$name")
            mv.visitLdcInsn("$methodId")
            mv.visitMethodInsn(
                Opcodes.INVOKESTATIC,
                "android/os/Trace",
                "beginSection",
                "(Ljava/lang/String;)V",
                false
            )
        }

        override fun onMethodExit(opcode: Int) {
            mv.visitMethodInsn(
                INVOKESTATIC,
                "android/os/Trace",
                "endSection",
                "()V",
                false
            )
        }

    }
}
```

### TraceTransformTask

```kotlin
abstract class TransformTask : DefaultTask() {
    // 所有输入的 jar 文件
    @get:InputFiles
    abstract val allJars: ListProperty<RegularFile>
    
    // 所有输入的 classes 目录
    @get:InputFiles
    abstract val allDirectories: ListProperty<Directory>
    
    // 输出的 jar 文件
    @get:OutputFile
    abstract val outputJar: RegularFileProperty
    
    // 输出的 mapping 文件
    @get:OutputFile
    abstract val mappingFile: RegularFileProperty

    @TaskAction
    fun transform() {
        var currentMethodId = 0L
        val mapping = mutableMapOf<String, Long>()
        val factory: (ClassWriter) -> ClassVisitor = { next ->
            TraceClassVisitor(Opcodes.ASM9, next) { method ->
                mapping.getOrPut(method) { currentMethodId++ }
            }
        }
        
        JarOutputStream(
            outputJar.get().asFile
                .outputStream()
                .buffered()
        ).use { output ->
            // 遍历所有的 jar 文件输入, 并进行插桩
            allJars.get().forEach { jar ->
                transformJar(output, jar.asFile, factory)
            }
            
            // 遍历所有的 classes 目录输入, 并进行插桩
            allDirectories.get().forEach { dir ->
                transformClasses(output, dir.asFile, factory)
            }
        }
        
        // 插桩后, 输出 mapping 信息
        mappingFile.get().asFile
            .writeText(mapping.toString())
    }

    private fun transformJar(
        output: JarOutputStream,
        jar: File,
        factory: (ClassWriter) -> ClassVisitor
    ) {
        JarInputStream(
            jar.inputStream()
                .buffered()
        ).use { input ->
            while (true) {
                val entry = input.nextEntry ?: break
                if (entry.isDirectory) continue
                if (!entry.name.endsWith(".class")) continue

                transform(entry.name, input, output, factory)
            }
        }
    }

    private fun transformClasses(
        output: JarOutputStream,
        rootDir: File,
        factory: (ClassWriter) -> ClassVisitor
    ) {
        rootDir.walk().forEach { child ->
            if (child.isDirectory) return@forEach
            if (!child.name.endsWith(".class")) return@forEach
            val name = child.toRelativeString(rootDir)

            child.inputStream()
                .buffered()
                .use { input ->
                    transform(name, input, output, factory)
                }
        }
    }
    
    // 创建 ClassReader, ClassVisitor, ClassWriter 进行插桩
    private fun transform(
        name: String,
        input: InputStream,
        output: JarOutputStream,
        factory: (ClassWriter) -> ClassVisitor
    ) {
        val entry = ZipEntry(name)
        output.putNextEntry(entry)

        val cr = ClassReader(input)
        val cw = ClassWriter(ClassWriter.COMPUTE_MAXS)
        cr.accept(factory(cw), ClassReader.EXPAND_FRAMES)

        output.write(cw.toByteArray())
        output.closeEntry()
    }

}
```

### TraceTransformPlugin

```kotlin
abstract class TraceTransformPlugin : Plugin<Project> {

    override fun apply(project: Project) {
        project.plugins.withId("com.android.application") {
            project.extensions.configure(ApplicationAndroidComponentsExtension::class.java) { extension ->
                extension.onVariants {
                    // 配置 mapping 文件输出路径
                    val transformTask = project.tasks.register(
                        "transform${it.name.capitalized()}",
                        TransformTask::class.java
                    ) { task ->
                        task.mappingFile.set(project.buildDir.resolve("outputs/transform/mapping.txt"))
                    }
                    
                    // 将 TransformTask 注册到 class 处理流程中
                    it.artifacts
                        .forScope(ScopedArtifacts.Scope.ALL)
                        .use(transformTask)
                        .toTransform(
                            ScopedArtifact.CLASSES,
                            TransformTask::allJars,
                            TransformTask::allDirectories,
                            TransformTask::outputJar
                        )
                }
            }
        }
    }

}
```

## 优劣势

*   **优势:**
    *   处理 class 文件的方式更加灵活, 可以按需实现对应的处理流程, 例如可以先对全工程进行分析再进行字节码操作.
    *   可以与其它的 `Gradle Task` 配合, 使用其它 `Gradle Task` 的输出作为输入或将其输出作为其它 `Gradle Task` 的输入.
*   **劣势:**
    *   相较于 `Instrumentation` 需要处理许多额外的逻辑, 例如对 class 文件的读写.
    *   实现一个高性能的 `Task` 需要付出许多额外的努力, 例如实现 [Incremental tasks](https://docs.gradle.org/current/userguide/custom_tasks.html#incremental_tasks) 等.

# 参考资料

*   [Transform API is removed](https://developer.android.com/build/releases/gradle-plugin-api-updates#transform-removed)
