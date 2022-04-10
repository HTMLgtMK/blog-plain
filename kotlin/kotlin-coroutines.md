## 协程 Coroutine

进程是系统资源调度和分派的基本单位，是操作系统层面的并发。\
线程是CPU资源调度和分派的基本单位，是进程内部的并发。\
协程是线程的子任务，是线程内部的并发。

协程形式上与函数类似，但可以在执行过程中挂起，线程转而执行其他任务。\
协程具有的优势：
1. 协程间切换不需要切换线程，没有线程切换的开销。
2. 协程间共享状态和变量可能不需要锁。因为同一个线程内的协程是交替执行的。但是如果代码中存在临界区（critical section），则需要利用 Runtime 锁来保证数据一致性。

适用 kotlin 协程需要引入写协程库 ``kotlinx.coroutines.*``, 其中包括 launch, async 等原语。

### 协程构建器

1. ``launch``   构建一个非阻塞的协程
2. ``runBlocking`` 构建一个阻塞的协程
3. ``async``    并发的协程
4. ``flow`` 流构建器

示例:
```kotlin
fun main() = runBlocking {  // 阻塞的协程等待内部协程结束
    launch {  // 非阻塞的协程
        delay(1000L)
        println("run on inner")
    }
    println("run on outter")
}
```

### 阻塞
1. `delay` 挂起函数；仅挂起协程，不阻塞线程。只能在协程中使用。
2. `runBlocking` 协程构建器，会阻塞线程直到协程结束。
3. `join`, 由 launch 返回的 job，可以用 join 等待。
4. `await`，由 async 返回的 defered，可以 await 等待结果。
5. `coroutineScope()` 协程作用域构建器，将挂起线程。

### 取消

```kotlin
var job = launch {}
job.cancel() // 取消运行
job.join()  // 等待结束
job.cancelAndJoin()  // 取消并等待
```

在协程被取消时将抛出 CancellationException, 可用 try-catch-finally 包裹。

**不可取消**：
with(NonCancellable) {}

**超时**:
withTimeout() {}  // 将抛出 TimeoutCancellationException


### 组合函数挂起
并发 `async`, 将启动一个协程。与 launch 不同点：（返回值 ）
* launch: Job, 不附带任何结果值。 
* async: Defered, 一个轻量级非阻塞future。

defered 可以使用 await 获取返回值。

### 惰性启动
``async(start = CoroutineStart.Lazy) {}``

```kotlin
var one = async(CoroutineStart.Lazy) {}
one.start()  // 启动并发
one.await()  // 串行等待（如未启动，则串行执行）
```

### 结构化并发
```kotlin
suspend fun concurrentSum(): Int = coroutineScope {
    var one = async { doOne() }
    var two = async { doTwo() }
    one.await() + two.await()
}
```

若concurrentSum()内部出现错误，且抛出错一个异常，则所在的整个作用域中所启动的协程都将被取消。

### kotlin 协程作用域 Scope
开启 kotlin 协程需要在 ``CoroutineScope`` 上使用 `launch`, `async` 这些 `coroutine builder`.

```kotlin
public interface CoroutineScope {
    public val coroutineContext: CoroutineContext
}

public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
```

协程作用域构建器：`coroutineScope`, 它会创建一个协程作用域并且在所有已启动子协程执行完毕之前不会结束。
```shell
fun main() = runBlocking {
    launch {
        delay(1000L)
        println("run on inner")
    }

    coroutineScope {
        launch {
            delay(1000L)
            println("run on inner scope task 1")
        }
        launch {
            delay(1000L)
            println("run on inner scope task 2")
        }
    }

    println("run on outter")
}
```

``runBlocking`` 与 ``coroutineScope`` 比较：
- 相同：它们都会等待其协程体以及所有子协程结束。
- 不同：`runBlocking` 将阻塞当前线程进行等待，`coroutineScope` 则是挂起，释放底层线程。

特殊的协程作用域：
- `GlobalScope`: 作用域存在于应用的整个生命周期，因此 `GlobalScope` 中所创建的 coroutine 必须又用户自己维护状态。
```kotlin
@DelicateCoroutinesApi
public object GlobalScope : CoroutineScope {
    override val coroutineContext: CoroutineContext
        get() = EmptyCoroutineContext
}
```

- `MainScope`：作用域与具有生命周期的组件绑定，该函数定义了一个使用 `SupervisorJob` 和 `Dispatchers.Main` 为 Scope context 的实现。
```kotlin
@Suppress("FunctionName")
public fun MainScope(): CoroutineScope = ContextScope(SupervisorJob() + Dispatchers.Main)
```

**`suspend` 修饰符**
在将`launch`内部的代码提取为一个函数时可以适用 `suspend` 修饰符，这样函数内部可以使用 CoroutineScope 的扩展函数。
```kotlin
suspend fun <funName>()
```

如果提取出的函数包含一个在当前作用域中调用的协程构建器的话，该怎么办？
1. 将该函数作为 ``CoroutineScope`` 的扩展函数
```kotlin
fun CoroutineScope.launchDemo() {
    launch {

    }
}
```
2.显示的将 ``CoroutineScope`` 做为该类的一个字段, 或者直接实现 `CoroutineScope` 接口。
```kotlin
class ScopeDemo { // 或 : CoroutineScope by MainScope()
    private val scope = MainScope() + CoroutineName("MyScope")

    fun demoTest() {
        scope.launch { }
    }
}
```

实现一个 CoroutineScope:
```kotlin
class ScopeDemo2:CoroutineScope {
    private lateinit var job: Job
    override val coroutineContext: CoroutineContext
        get() = Dispatchers.Default + job

    fun create() {
        job = Job()
    }

    fun test() {
        launch {
            for (i in 1..100) {
                delay(1000L)
                println("current is $i")
            }
        }
    }

    fun terminate() {
        // 取消该 Scope 管理的 job,
        // 这样在该 Scope 内创建的子 Coroutine 都会被自动的取消
        job.cancel()
    }
}
```

### 协程上下文 CoroutineContext
调度器 CoroutineDispatcher

1. 特定的线程
   1. launch(){}  // 父协程上下文
   2. launch(newSingleThreadContext(""")){} // 运行在新线程
2. 不受限制的运行：launch(Dispatchers.undefined){}
3. 主线程： launch(Dispatchers.Default){}，与 GlobalScope.launch() 共享相同调度器.

android 使用 MainScope(), Dispatchers.Main 调度 UI。

**改变上下文**：
withContext(ctx){}

**命名协程**：
async(CoroutineName("<name>"")){}



### 序列、流
挂起函数仅返回异步的单个值，如何返回多个值呢？

#### 同步流 Sequence
```kotlin
fun simple(): Sequence<Int> { // 将阻塞主线程
    for (i in 1..3) {
        Thread.sleep(1000L)
       yeild(i)
    }
}
fun main() {
    simple().forEach { value-> println(value) }
}
```

#### 异步流 Flow
- flow() 流构建器
- asFlow() 构建器
- flowOf

```kotlin
fun simple(): Flow<Int> {  // 不阻塞主线程
    for (i in 1..3) {
        delay(1000L)
        emit(i)  // 发射下一个值
    }
}
fun main() {
    simple().collect {value -> println(value) }  // 收集，直到 collect 时 flow 才开始运行
}
```

- 过滤：map, filter. 可以挂起函数（与rx的区别）。
- 转换：transform, 可发射任意值任意次。
- 限长：take(n) 只取前 n 个值。
- 末端：toList(), toSet(), first(), single(), reduce(), fold().
- 流缓冲：buffer()
- 合并: conflate()
- 组合多个流: zip()
- 展平流: flatten(), flatMap(), flatMapConcat(), flatMapReduce(), flatMapLatest().
- 声明式处理：onCompletion() 完全收集时回调
- 声明式捕获：catch{} 仅捕获上游异常
- 可取消：cancellation()

### 通道 Channel
用于在流中传输值，可用于多个协程交换信息。
与 BlockingQueue 类似，区别：

| 出入 |  Channel  |  BlockingQueue |
| 扇入 |   send    |     put        |
| 扇出 |  receive  |    take        |

```kotlin
var channel = Channel<Int>(bufferSize)  // 带缓冲
launch { repeat(5) { channel.send(1) } } 
repeat(5) channel.receive()
```

关闭通道： channel.close()
迭代通道：for (x in channel)

producer-consumer 模型（将相互挂起）：

```kotlin
fun CoroutineScope.produceSqueres(): ReceiveChannel<Int> = produce {
   for (i in 1..5) {
       send(i * i)
   }
}

fun main() {
    runBlocking {
        var squares = produceSqueres()
       squares.consumeEach{println(it)}
    }
}
```


### 参考资料：
- [1] [协程设计文档](https://github.com/Kotlin-zh/KEEP/blob/master/proposals/coroutines.md)
- [2] [适用协程 UI 编程指南](https://github.com/hltj/kotlinx.coroutines-cn/blob/master/ui/coroutines-guide-ui.md)
- [3] [kotlin 语言中文站](https://www.kotlincn.net/docs/reference/coroutines/coroutines-guide.html)


