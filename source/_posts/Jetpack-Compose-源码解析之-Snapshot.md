---
layout: post
title: Jetpack Compose 源码解析之 Snapshot
typora-root-url: ../../source
date: 2021-10-10 16:51:34
tags: 
  - Android
  - AndroidX 
  - Jetpack
  - Jetpack Compose
---

# 引言

Snapshot 是 Jetpack Compose Runtime 中的一个子模块，主要用于解决以下两个问题：

0. 利用多版本并发控制算法，保证并发环境下视图状态的原子性、一致性、隔离性。
0. 提供对状态读写的监听能力。

## 用例演示

```kotlin
fun main() {
    // 创建可跟踪读写的可变状态，初始化值：0
    var state: Int by mutableStateOf(0)

    // 创建一个可写的快照环境
    val snapshot = Snapshot.takeMutableSnapshot(
        { state -> /* 状态被读取 */ },
        { state -> /* 状态被写入 */ }
    )

    // 进入快照环境
    snapshot.enter {
        // 修改状态的值
        state = 1
        assertEquals(1, state)
    }

    // 在快照提交前，快照环境外的状态值不会受到影响
    assertEquals(0, state)

    // 以原子的方式提交快照中所有对状态的修改，出现并发修改时提交可能会失败
    val result = snapshot.apply()
    assertEquals(true, result.succeeded)
    assertEquals(1, state)
}
```

# 原理分析

本章节主要介绍 Snapshot 是如何实现多版本并发控制算法的，为了简化概念突出重点，介绍以核心流程为主，和实际的源码实现存在出入。下面先介绍在算法中涉及的三个重要概念：

* Snapshot：快照对象，提供了原子性和隔离性的保证。
* StateObject：状态对象，提供了对状态的读写操作。
* StateRecord：状态记录，用于记录不同快照版本中的状态值。

## 多版本状态维护

多版本并发控制，顾名思义就是要利用版本号进行并发控制，其核心在于在全局环境中会维护着一个单调递增的版本号称作 `snapshotId`，每个 Snapshot 对象和 StateRecord 对象都会有一个与之关联的 snapshotId。

每个 StateObject 对象都会与一个 StateRecord 链表关联。每当一个 StateObject 被创建时都会被赋予一个初始的值，该值被记录在一个 StateRecord 中成为链表的头，该 StateRecord 关联的 snapshotId 和创建 StateObject 时 Snapshot 对象关联的 snapshotId 相同。

当 StateObject 被赋予新的值时，会用一个 StateRecord 用于记录新值，并加入到与 StateObject 关联的链表中。该 StateRecord 关联的 snapshotId 和赋值时的 Snapshot 对象关联的 snapshotId 相同。由于一般最新写入的 StateRecord 最可能是读取时需要的 StateRecord，所以一般把新写入的 StateRecord 放在链表的头作为性能优化的方法。

例如对于一个支持读写整形数的 StateObject，在 snapshotId = 1 的环境中创建了一个初始值为 1 的 StateObject 对象，在 snapshotId = 2 的环境中修改了 StateObject 对象的值为 2，此时 StateObject 对象的状态如下所示：

![](/images/202110071730.png)

## 多版本快照维护

无论何时，在创建、修改、读取状态时都需要有一个与之关联的 Snapshot 对象。Snapshot 提供了三种基础操作：

* 创建快照：创建一个新的快照对象，但是并没有使用新的快照替换当前的快照。
* 进入快照：进入该快照的环境，会使得对对象的创建、读取、写入操作基于该快照进行。
* 提交快照：将该快照中对状态的修改提交到其父快照中，使得父快照对其修改可见。

默认状态下存在一个全局快照（GlobalSnapshot），当没有显式的进入任何快照时，与 StateObject 关联的就是全局快照。每个快照对象内部主要维护了两个值：

* snapshotId：Snapshot 关联的快照版本号。
* invlid：快照版本黑名单，在 StateObject 读取选取 StateRecord 时用到。

下图演示了从最初的 GlobalSnapshot 开始执行了一系列的创建快照、提交快照操作后 snapshotId 和 invlid 的变化情况：

![](/images/202110101516.png)

### 快照创建

每当有快照被创建时，会有以下操作：

* 在创建子快照后晋级父快照的 snapshotId 至比子快照 snapshotId 大，这样做保证了子快照看不到后续在父快照中对 StateObject 的修改。
* 父快照把 （父快照旧版本, 父快照新版本) 区间加入到 invlid 集合中，这样做保证了父快照看不到子快照及兄弟快照对 StateObject 的修改。
* 子快照把 (父快照旧版本, 子快照版本) 区间加入到 invlid 集合中，这样做保证了子快照只能看到父快照以前对 StateObject 的修改。

### 快照提交

每当有快照被提交时，会有以下操作：

* 如果父快照 snapshotId 比子快照 snapshotId 小，那么父快照晋级 snapshotId 至比子快照 snapshotId 大，晋级时父快照把 （父快照旧版本, 父快照新版本) 区间加入到 invlid 集合中。
* 父快照从 invlid 集合中移除已提交的子快照 snapshotId（包含使用过的）。

这样做都是为了保证父快照后续能看到子快照已提交的对 StateObject 的修改。

## 状态读取

上文提到了，每个 StateObject 都会有一个与其关联的 StateRecord 链表，该链表中保存了多个与不同 snapshotId 关联的 StateRecord。当对 StateObject 进行读取操作时，实际上就是根据当前的 Snapshot 选取符合的 StateRecord。

例如对于如下所示的一个 StateObject，其在 snapshotId = [1, 3, 5] 三个快照版本中有对 StateObject 进行写入操作。

![](/images/202110101636.png)

当在 `Snapshot(snapshotId = 4, invlid = [3])` 的快照中对该 StateObject 进行读取时，读取到的 StateObject 的值为 1。因为：

* `StateRecord(snapshotId = 5, value = 5)` 记录中的 snapshotId 比 Snapshot 中的 snapshotId 大。
* `StateRecord(snapshotId = 3, value = 3)` 记录中的 snapshotId 在 Snapshot 的 invlid 集合中。
* `StateRecord(snapshotId = 1, value = 1)` 记录中的 snapshotId 比 Snapshot 中的 snapshotId 小，且记录中的 snapshotId 不在 Snapshot 的 invlid 集合中。

总的来说就是要读取 `StateRecord.snapshotId` 小于 `Snapshot.snapshotId` 且不在 `Snapshot.invlid` 集合中最大的 `StateRecord.value`。


# 源码解读

在知晓基本原理后，接下来将对源码进行解读。首先要了解的，Snapshot 模块由两个部分组成：

0. Snapshot 核心：实现创建、提交快照功能，负责维护快照内状态的值，提供快照内状态值读写的跟踪能力。
0. 可跟踪读写的状态：该部分是可扩展的，目前预置状态有：`mutableStateOf`、`mutableStateListOf`、`mutableStateMapOf`。

下文将以 `mutableStateOf` 为例，在源码中增加必要的注解信息解读源码的逻辑。下图展示以 `mutableStateOf` 为例的整个 Snapshot 的架构全貌：

![](/images/202110051620.png)

## 创建快照

### Snapshot#take[Mutable]Snapshot

```kotlin
/**
 * 用于创建只读的快照，主要步骤：
 * 1. 获取当前上下文中的快照。
 * 2. 通过当前上下文中的快照创建嵌套的只读快照。
 */
fun takeSnapshot(
    readObserver: ((Any) -> Unit)? = null
): Snapshot = currentSnapshot().takeNestedSnapshot(readObserver)

/**
 * 创建可写的快照，步骤与创建只读快照相似，只是多了一步检查：如果当前的上下文中的快照不是可写类型的话就会抛出异常。
 */
fun takeMutableSnapshot(
    readObserver: ((Any) -> Unit)? = null,
    writeObserver: ((Any) -> Unit)? = null
): MutableSnapshot =
    (currentSnapshot() as? MutableSnapshot)?.takeNestedMutableSnapshot(
        readObserver,
        writeObserver
    ) ?: error("Cannot create a mutable snapshot of an read-only snapshot")

/**
 * 获取当前上下文中的快照。
 * 1. 如果当前有与当前线程关联的 Snapshot，则返回。 当调用 Snapshot#enter 时会使得当前线程与 Snapshot 产生关联。
 * 2. 否则使用全局的 GlobalSnapshot。
 */
internal fun currentSnapshot(): Snapshot =
    threadSnapshot.get() ?: currentGlobalSnapshot.get()
```

### GlobalSnapshot#takeNested[Mutable]Snapshot

```kotlin
override fun takeNestedSnapshot(readObserver: ((Any) -> Unit)?): Snapshot =
    /**
     * 通过 takeNewSnapshot 完成嵌套快照创建，保证嵌套快照创建后将当前 id 加入到 openSnapshots 名单中，
     * 并且晋级 GlobalSnapshot 的 id，保证 GlobalSnapshot 后续对 StateObject 的修改对嵌套 Snapshot 不可见。
     */
    takeNewSnapshot { invalid ->
        /**
         * GlobalSnapshot 时创建的嵌套只读快照对象类型是 ReadonlySnapshot。
         */
        ReadonlySnapshot(
            id = sync { nextSnapshotId++ },
            invalid = invalid,
            readObserver = readObserver
        )
    }

/**
 * 和 GlobalSnapshot#takeNestedSnapshot 类似，只是创建的嵌套可写快照对象类型是 MutableSnapshot。
 */
override fun takeNestedMutableSnapshot(
    readObserver: ((Any) -> Unit)?,
    writeObserver: ((Any) -> Unit)?
): MutableSnapshot = // 省略...

private fun <T : Snapshot> takeNewSnapshot(block: (invalid: SnapshotIdSet) -> T): T =
    /**
     * 通过 advanceGlobalSnapshot 方法晋级全局的 GlobalSnapshot 对象。
     */
    advanceGlobalSnapshot { invalid ->
        /**
         * 创建出新的嵌套快照，并将新快照的 id 加入 openSnapshots 集合中。
         */
        val result = block(invalid)
        sync {
            openSnapshots = openSnapshots.set(result.id)
        }
        result
    }

private fun <T> advanceGlobalSnapshot(block: (invalid: SnapshotIdSet) -> T): T {
    val previousGlobalSnapshot = currentGlobalSnapshot.get()
    val result = sync {
        /**
         * 晋级全局的 GlobalSnapshot 对象。
         */
        takeNewGlobalSnapshot(previousGlobalSnapshot, block)
    }

    /**
     * 通知所有通过 Snapshot#registerApplyObserver 注册的观察者，全局快照的数据有变更。
     */
    val modified = previousGlobalSnapshot.modified
    if (modified != null) {
        val observers: List<(Set<Any>, Snapshot) -> Unit> = sync { applyObservers.toMutableList() }
        observers.fastForEach { observer ->
            observer(modified, previousGlobalSnapshot)
        }
    }

    return result
}

private fun <T> takeNewGlobalSnapshot(
    previousGlobalSnapshot: Snapshot,
    block: (invalid: SnapshotIdSet) -> T
): T {
    /**
     * 执行 takeNewSnapshot 中传入的 block：创建出新的嵌套快照，并将新快照的 id 加入 openSnapshots 集合中。
     * 由于嵌套快照需要能够访问到 previousGlobalSnapshot 中对 StateObject 的修改，所以需要将 previousGlobalSnapshot.id 排除在 invalid 名单外。
     */
    val result = block(openSnapshots.clear(previousGlobalSnapshot.id))

    /**
     * 在通过 block 完成嵌套快照的创建后，需要晋级当前 GlobalSnapshot 的 id 至比嵌套快照 id 大的值，
     * 避免嵌套快照能访问到 GlobalSnapshot 中后续对 StateObject 的修改。
     */
    sync {
        val globalId = nextSnapshotId++
        openSnapshots = openSnapshots.clear(previousGlobalSnapshot.id)
        currentGlobalSnapshot.set(
            GlobalSnapshot(
                id = globalId,
                invalid = openSnapshots
            )
        )
        openSnapshots = openSnapshots.set(globalId)
    }

    return result
}
```

### [Nested]MutableSnapshot#takeNested[Mutable]Snapshot

```kotlin
override fun takeNestedSnapshot(readObserver: ((Any) -> Unit)?): Snapshot {
    validateNotDisposed()
    validateNotApplied()
    val previousId = id

    /**
     * 在 advance 传入的 block 被执行后，会晋级当前快照的 id，保证创建出的嵌套快照无法看到当前快照后续对 StateObject 的修改。
     */
    return advance {
        sync {
            val readonlyId = nextSnapshotId++
            openSnapshots = openSnapshots.set(readonlyId)

            /**
             * 计算嵌套快照的 invalid 名单，理论上嵌套快照只能看到当前快照的状态，
             * 所以需要将区间 [previousId+1, readonlyId) 的 id 都加入到 invalid 名单中。
             */
            var currentInvalid = invalid
            for (invalidId in previousId + 1 until readonlyId) {
                currentInvalid = currentInvalid.set(invalidId)
            }

            /**
             * 当前快照是 [Nested]MutableSnapshot 时，创建的嵌套只读快照类型是 NestedReadonlySnapshot。
             */
            NestedReadonlySnapshot(
                readonlyId,
                currentInvalid,
                readObserver,
                this
            )
        }
    }
}

/**
 * 和 MutableSnapshot#takeNestedSnapshot 类似，只是创建的嵌套可写快照类型是 NestedMutableSnapshot。
 */
open fun takeNestedMutableSnapshot(
    readObserver: ((Any) -> Unit)? = null,
    writeObserver: ((Any) -> Unit)? = null
): MutableSnapshot = // 省略...

internal inline fun <T> advance(block: () -> T): T {
    /**
     * 记录旧的 id 值，在 abandon 或 close 时需要用到。
     */
    recordPrevious(id)

    /**
     * 调用 block 方法完成嵌套快照的创建。
     */
    return block().also {
        val previousId = id

        /**
         * 增加快照的 id 值，保证后续在当前快照中对 StateObject 的修改不会对它的嵌套快照可见。
         */
        sync {
            id = nextSnapshotId++
            openSnapshots = openSnapshots.set(id)
        }

        /**
         * 更新 invalid 名单，保证当前快照不会看到 [previousId+1, id) 的任何快照对 StateObject 的修改，其中包括了它的嵌套快照。
         */
        var currentInvalid = invalid
        for (invalidId in previousId + 1 until id) {
            currentInvalid = currentInvalid.set(invalidId)
        }
        invalid = currentInvalid
    }
}
```

### [Nested]ReadonlySnapshot#takeNestedSnapshot

```kotlin
override fun takeNestedSnapshot(readObserver: ((Any) -> Unit)?) =
		/**
		 * 当快照是 [Nested]ReadonlySnapshot 时，创建的嵌套只读快照类型是 NestedReadonlySnapshot，
		 * 不同于 GlobalSnapshot 或 [Nested]MutableSnapshot 的是在创建嵌套快照时不需要晋级当前快照的 id，
		 * 嵌套快照的 id 值也与当前快照的 id 值相同。
		 */
    NestedReadonlySnapshot(id, invalid, readObserver, parent)
```

## 进入快照

### Snapshot#enter

```kotlin
inline fun <T> enter(block: () -> T): T {
    /**
     * 切换当前线程的 Snapshot
     */
    val previous = makeCurrent()
    try {
        /**
         * 此时 currentSnapshot() == this，执行 block 代码块。
         */
        return block()
    } finally {
        /**
         * 恢复当前线程的 Snapshot。
         */
        restoreCurrent(previous)
    }
}

internal open fun makeCurrent(): Snapshot? {
    val previous = threadSnapshot.get()
    threadSnapshot.set(this)
    return previous
}

internal open fun restoreCurrent(snapshot: Snapshot?) {
    threadSnapshot.set(snapshot)
}
```

## 创建状态

### mutableStateOf

```kotlin
/**
 * createSnapshotMutableState 会创建出 SnapshotMutableStateImpl 或其子类。
 */
fun <T> mutableStateOf(
    value: T,
    policy: SnapshotMutationPolicy<T> = structuralEqualityPolicy()
): MutableState<T> = createSnapshotMutableState(value, policy)

internal open class SnapshotMutableStateImpl<T>(
    value: T,
    override val policy: SnapshotMutationPolicy<T>
) : StateObject, SnapshotMutableState<T> {

    /**
     * 通过构造函数中传入的初始值，创建了第一个 StateRecord，与该对象关联的快照 id 是当前上下文中的快照 id。
     */
    private var next: StateStateRecord<T> = StateStateRecord(value)
    
    /**
     * 省略部分代码...
     */
}
```

## 修改状态

```kotlin
internal open class SnapshotMutableStateImpl<T>(
    value: T,
    override val policy: SnapshotMutationPolicy<T>
) : StateObject, SnapshotMutableState<T> {

    override var value: T
        get() = // 省略...
				
        /**
         * 当对 MutableState 进行赋值时，最终会调用到这里。
         * 1. 通过 StateRecord#withCurrent 拿到与当前 Snapshot 匹配的 StateRecord。
         * 2. 判断写入的新值是否与当前值相同，如果相同则忽略。
         * 3. 如果写入的新值与当前值不相同，则调用 StateRecord#overwritable 修改当前值。
         */
        set(value) = next.withCurrent {
            if (!policy.equivalent(it.value, value)) {
                next.overwritable(this, it) { this.value = value }
            }
        }
    
    /**
     * 省略其它代码...
     */
}

/**
 * 通过 current 获取与当前 Snapshot 匹配的 StateRecord，执行 block 时作为参数传入。
 */
inline fun <T : StateRecord, R> T.withCurrent(block: (r: T) -> R): R =
    block(current(this, Snapshot.current))

/**
 * current 与 readable(T, Snapshot) 的区别是：readable(StateObject, Snapshot) 会记录 StateObject 的读取行为。
 */
internal fun <T : StateRecord> current(r: T, snapshot: Snapshot) =
    readable(r, snapshot.id, snapshot.invalid) ?: readError()
```

### StateRecord#overwritable

```kotlin
internal inline fun <T : StateRecord, R> T.overwritable(
    state: StateObject,
    candidate: T,
    block: T.() -> R
): R {
    var snapshot: Snapshot = snapshotInitializer
    return sync {
        /**
         * 获取当前的 Snapshot 对象。
         */
        snapshot = Snapshot.current

        /**
         * 通过 overwritableRecord 获取用于保存新写入值的 StateRecord，执行 block 函数会进行最终的赋值操作。
         */
        this.overwritableRecord(state, snapshot, candidate).block()
    }.also {
        /**
         * 通知当前的 Snapshot 的写入观察者，StateObject 有写入行为。
         */
        notifyWrite(snapshot, state)
    }
}

internal fun <T : StateRecord> T.overwritableRecord(
    state: StateObject,
    snapshot: Snapshot,
    candidate: T
): T {
    /**
     * 当 Snapshot 是只读的时候，直接调用 recordModified 触发快速失败。
     */
    if (snapshot.readOnly) {
        snapshot.recordModified(state)
    }
    val id = snapshot.id

    /**
     * StateObject 当前的 StateRecord 关联的 snapshotId 是否与当前的 Snapshot 相同，
     * 如果相同则复用当前的 StateRecord。
     */
    if (candidate.snapshotId == id) return candidate

    /**
     * 当 StateRecord 关联的 snapshotId 与当前的 Snapshot 不相同则时，
     * 需要创建新的 StateRecord 并与当前的 Snapshot 进行关联：
     * 1. StateRecord#snapshotId == Snapshot#id
     * 2. 通过 Snapshot#recordModified 记录 StateObject 有变更行为。
     */
    val newData = newOverwritableRecord(state, snapshot)
    newData.snapshotId = id
    snapshot.recordModified(state)

    return newData
}

internal fun <T : StateRecord> T.newOverwritableRecord(state: StateObject, snapshot: Snapshot): T {
    /**
     * 1. 通过 used 寻找一个可复用的 StateRecord 对象，如果找不到的话就创建一个新的对象。
     * 2. 初始化 StateRecord：将 snapshotId 设置为最大值
     * 3. 修改 StateObject#firstStateRecord 链表的头。
     */
    @Suppress("UNCHECKED_CAST")
    return (used(state, snapshot.id, openSnapshots) as T?)?.apply {
        snapshotId = Int.MAX_VALUE
    } ?: create().apply {
        snapshotId = Int.MAX_VALUE
        this.next = state.firstStateRecord
        state.prependStateRecord(this as T)
    } as T
}

private fun used(state: StateObject, id: Int, invalid: SnapshotIdSet): StateRecord? {
    var current: StateRecord? = state.firstStateRecord
    var validRecord: StateRecord? = null
    val lowestOpen = invalid.lowest(id)

    /**
     * 寻找可以被复用的 StateRecord：可以被复用的 StateRecord 满足以下条件：
     * 1. currentId == INVALID_SNAPSHOT：当 Snapshot 被抛弃(abandon)时（调用 Snapshot#abandon 方法），与之关联的 StateRecord#snapshotId 会被置为
     * INVALID_SNAPSHOT。
     * 2. 找到两个与已经被提交（apply）的 Snapshot 关联的 StateRecord，snapshotId 较小的那个可以被复用。
     */
    while (current != null) {
        val currentId = current.snapshotId
        if (currentId == INVALID_SNAPSHOT) {
            return current
        }
        if (valid(current, lowestOpen, invalid)) {
            if (validRecord == null) {
                validRecord = current
            } else {
                return if (current.snapshotId < validRecord.snapshotId) current else validRecord
            }
        }
        current = current.next
    }
    return null
}
```

## 读取状态

```kotlin
internal open class SnapshotMutableStateImpl<T>(
    value: T,
    override val policy: SnapshotMutationPolicy<T>
) : StateObject, SnapshotMutableState<T> {

    override var value: T
        /**
         * 通过 StateRecord#readable 拿到与当前 Snapshot 匹配的 StateRecord，返回 StateRecord 中记录的 value 值。
         */
        get() = next.readable(this).value
				
        set(value) = // 省略...
    
    /**
     * 省略其它代码...
     */
}
```

### SateRecord#readable

```kotlin
fun <T : StateRecord> T.readable(state: StateObject): T =
    /**
     * 根据当前的 Snapshot 查找与之对应的 StateRecord。
     */
    readable(state, currentSnapshot())

fun <T : StateRecord> T.readable(state: StateObject, snapshot: Snapshot): T {
    /**
     * 通知 Snapshot 的读取观察者，StateObject 有读取行为。
     */
    snapshot.readObserver?.invoke(state)
    return readable(this, snapshot.id, snapshot.invalid) ?: readError()
}

private fun <T : StateRecord> readable(r: T, id: Int, invalid: SnapshotIdSet): T? {
    var current: StateRecord? = r
    var candidate: StateRecord? = null
    while (current != null) {
        /**
         * StateRecord 有效性检查：
         * 1. StateRecord#snapshotId != INVALID_SNAPSHOT。
         * 2. StateRecord#snapshotId 小于等于 id。
         * 3. invalid 集合中不存在 StateRecord#snapshotId。
         */
        if (valid(current, id, invalid)) {
            /**
             * 选择 StateRecord#snapshotId 最大的作为最终的返回值。
             */
            candidate = if (candidate == null) current
            else if (candidate.snapshotId < current.snapshotId) current else candidate
        }
        current = current.next
    }
    if (candidate != null) {
        @Suppress("UNCHECKED_CAST")
        return candidate as T
    }
    return null
}
```

## 提交快照

### MutableSnapshot#apply

```kotlin
open fun apply(): SnapshotApplyResult {
    val modified = modified

    /**
     * 在不持锁的情况下，先进行一轮快速合并。
     */
    val optimisticMerges = if (modified != null) optimisticMerges(
        currentGlobalSnapshot.get(),
        this,
        openSnapshots.clear(currentGlobalSnapshot.get().id)
    ) else null

    val (observers, globalModified) = sync {
        validateOpen(this)

        if (modified == null || modified.size == 0) {
            /**
             * 如果该 Snapshot 没有对 StateObject 进行修改，那么：
             * 1. 关闭该 Snapshot：将其从 openSnapshots 中名单中移除。
             * 2. 晋级 GlobalSnapshot，使其 invalid 名单与 openSnapshots 保持同步。
             */
            close()
            val previousGlobalSnapshot = currentGlobalSnapshot.get()
            takeNewGlobalSnapshot(previousGlobalSnapshot, emptyLambda)
            val globalModified = previousGlobalSnapshot.modified
            if (globalModified != null && globalModified.isNotEmpty())
                applyObservers.toMutableList() to globalModified
            else
                emptyList<(Set<Any>, Snapshot) -> Unit>() to null
        } else {
            val previousGlobalSnapshot = currentGlobalSnapshot.get()
            /**
             * 持锁状态下，将该 Snapshot 修改的 StateObject 和 GlobalSnapshot 进行合并。
             */
            val result = innerApply(
                nextSnapshotId,
                optimisticMerges,
                openSnapshots.clear(previousGlobalSnapshot.id)
            )

            /**
             * 合并失败的情况下，中断 apply 操作。
             */
            if (result != SnapshotApplyResult.Success) return result

            /**
             * 合并成功的情况下：
             * 1. 关闭当前 Snapshot：将其从 openSnapshots 中名单中移除。
             * 2. 晋级 GlobalSnapshot，使得 GlobalSnapshot 能访问到该 Snapshot 对 StateObject 的修改。
             */
            close()
            takeNewGlobalSnapshot(previousGlobalSnapshot, emptyLambda)
            val globalModified = previousGlobalSnapshot.modified
            this.modified = null
            previousGlobalSnapshot.modified = null

            applyObservers.toMutableList() to globalModified
        }
    }

    /**
     * 标记该 Snapshot 的状态为已提交。
     */
    applied = true

    /**
     * 通知因为本次提交引发的 GlobalSnapshot 修改一并所提交的 globalModified。
     */
    if (globalModified != null && globalModified.isNotEmpty()) {
        observers.fastForEach {
            it(globalModified, this)
        }
    }

    /**
     * 通知因为本次提交该 Snapshot 合并进入 GlobalSnapshot 的 modified。
     */
    if (modified != null && modified.isNotEmpty()) {
        observers.fastForEach {
            it(modified, this)
        }
    }

    return SnapshotApplyResult.Success
}
```

### NestedMutableSnapshot#apply

```kotlin
override fun apply(): SnapshotApplyResult {
        if (parent.applied || parent.disposed) return SnapshotApplyResult.Failure(this)

        val modified = modified
        val id = id

        /**
         * 在无锁的情况下先对被修改 StateObject 进行一轮快速合并。
         */
        val optimisticMerges = if (modified != null)
            optimisticMerges(parent, this, parent.invalid)
        else null

        sync {
            validateOpen(this)
            if (modified == null || modified.size == 0) {
                /**
                 * 如果该 Snapshot 没有对 StateObject 进行修改，那么直接关闭即可：将该 id 从 openSnapshots 中移除。
                 */
                close()
            } else {
                /**
                 * 在持锁的情况下，将当前 Snapshot 对 StateObject 的修改合并进入父 Snapshot 中。
                 */
                val result = innerApply(parent.id, optimisticMerges, parent.invalid)

                /**
                 * 如果合并失败，那么中断 apply 操作。
                 */
                if (result != SnapshotApplyResult.Success) return result

                /**
                 * 如果合并成功，那么将当前 Snapshot 的修改记录添加只父 Snapshot 的修改记录中。
                 */
                (parent.modified ?: HashSet<StateObject>().also { parent.modified = it })
                    .addAll(modified)
            }

            /**
             * 晋级父 Snapshot 的 id 值，并且将该 Snapshot 的 id 从父 Snapshot 的 invalid 名单中移除，
             * 使其能看到当前 Snapshot 对 StateObject 的修改。
             */
            parent.advance()
            parent.invalid = parent.invalid.clear(id).andNot(previousIds)

            /**
             * 将当前 Snapshot 的 id 记录到父 Snapshot 的 previousIds 名单中，当父 Snapshot 被丢弃时需要用到。
             */
            parent.recordPrevious(id)
            parent.recordPreviousList(previousIds)
        }

        applied = true
        return SnapshotApplyResult.Success
    }
}
```

## 弃置快照

### MutableSnapshot#dispose

```kotlin
override fun dispose() {
    if (!disposed) {
        /**
         * 标记：disposed = true
         */
        super.dispose()

        /**
         * 当没有活跃的子 Snapshot 并且当前 Snapshot 没有提交的话，调用 abandon 方法释放资源。
         */
        nestedDeactivated(this)
    }
}

override fun nestedDeactivated(snapshot: Snapshot) {
    require(snapshots > 0)
    if (--snapshots == 0) {
        if (!applied) {
            abandon()
        }
    }
}

private fun abandon() {
    val modified = modified
    if (modified != null) {
        validateNotApplied()

        /**
         * 将当前 Snapshot 对 StateObject 修改时所使用的 StateRecord 标记为 INVALID_SNAPSHOT，
         * 当有新的 Snapshot 需要对该 StateObject 进行写入时，可以复用被标记为 INVALID_SNAPSHOT 的 StateRecord 对象。
         */
        this.modified = null
        val id = id
        for (state in modified) {
            var current: StateRecord? = state.firstStateRecord
            while (current != null) {
                if (current.snapshotId == id || current.snapshotId in previousIds) {
                    current.snapshotId = INVALID_SNAPSHOT
                }
                current = current.next
            }
        }
    }

    /**
     * 将当前 Snapshot 从 openSnapshots 名单中移除。
     */
    close()
}
```

### NestedMutableSnapshot#dispose

```kotlin
override fun dispose() {
    if (!disposed) {
        /**
         * 逻辑同 MutableSnapshot#dispose
         */
        super.dispose()

        /**
         * 通知父 Snapshot，当前 Snapshot 处于未活跃状态。
         */
        parent.nestedDeactivated(this)
    }
}
```

### ReadonlySnapshot#dispose

```kotlin
override fun dispose() {
    if (!disposed) {
        /**
         * 如果没有嵌套的 Snapshot 活跃了，关闭当前的 Snapshot
         */
        nestedDeactivated(this)

        /**
         * 标记：disposed = true
         */
        super.dispose()
    }
}

override fun nestedActivated(snapshot: Snapshot) {
    snapshots++
}

override fun nestedDeactivated(snapshot: Snapshot) {
    if (--snapshots == 0) {
        close()
    }
}
```

### NestedReadonlySnapshot#dispose

```kotlin
override fun dispose() {
    if (!disposed) {
        /**
         * 如果当前 Snapshot 不是和父 Snapshot 共享 id 的话，将它从 openSnapshots 名单中移除。
         */
        if (id != parent.id) {
            close()
        }

        /**
         * 通知父 Snapshot，当前 Snapshot 处于未活跃状态了。
         */
        parent.nestedDeactivated(this)

        /**
         * 标记：disposed = true
         */
        super.dispose()
    }
}
```