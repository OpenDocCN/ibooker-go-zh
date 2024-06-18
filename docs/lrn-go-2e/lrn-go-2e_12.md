# 第十二章《Go 语言并发》

*并发*是计算机科学术语，用于将单个进程分解为独立组件，并指定这些组件如何安全地共享数据。大多数语言通过操作系统级线程提供并发，通过尝试获取锁来共享数据。但 Go 语言不同。它的主要并发模型，可以说是 Go 语言最著名的特性，基于通信顺序进程（CSP）。这种并发风格在 1978 年由发明快速排序算法的 Tony Hoare 在一篇[paper](https://oreil.ly/x1IVG)中描述。使用 CSP 实现的模式与标准模式一样强大，但更易于理解。

在这一章中，您将快速回顾 Go 语言并发的核心特性：goroutines（协程）、channels（通道）和`select`关键字。然后，您将查看一些常见的 Go 并发模式，并了解在何种情况下使用底层技术是更好的方法。

# 何时使用并发

让我们先谨慎一点。确保您的程序从并发中获益。当新手 Go 开发人员开始尝试并发时，他们往往会经历一系列阶段：

1.  这太*神奇*了；我要把一切都放进 goroutines 中！

1.  我的程序并没有更快。我正在向我的 channels 添加缓冲区。

1.  我的 channels 在阻塞，我遇到了死锁。我将使用带有*非常*大缓冲区的 buffered channels。

1.  我的 channels 仍然在阻塞。我将使用互斥锁。

1.  算了吧，我放弃并发了。

人们被并发吸引，因为他们认为并发程序运行更快。不幸的是，这并不总是如此。更多的并发并不自动意味着事情更快，它可能会使代码更难理解。关键在于理解*并发不等于并行*。并发是一个工具，用来更好地解构您试图解决的问题。

并发代码是否并行（同时运行）取决于硬件和算法是否允许。1967 年，计算机科学先驱之一 Gene Amdahl 推导出了 Amdahl 定律。这是一个用于确定并行处理可以提升性能多少的公式，考虑了有多少工作必须顺序执行。如果您想深入了解 Amdahl 定律的细节，您可以在[*并发的艺术*](https://oreil.ly/HaZQ8)一书中了解更多，作者是 Clay Breshears（O’Reilly）。对于我们的目的，您只需理解更多的并发并不意味着更快。

广义上讲，所有程序都遵循相同的三步过程：获取数据，转换数据，然后输出结果。您是否应在程序中使用并发取决于数据在程序步骤之间的流动方式。有时候两个步骤可以并发进行，因为一个步骤的数据不需要等待另一个步骤才能继续进行，而其他时候两个步骤必须按顺序进行，因为一个步骤依赖于另一个步骤的输出。当您想要结合多个可以独立操作的操作的数据时，请使用并发。

另一个重要的注意事项是，并发不值得使用，如果运行并发的进程所花费的时间不多。并发并非免费；许多常见的内存算法非常快速，通过并发传递值的开销超过了通过并行运行并发代码可能获得的任何潜在时间节省。这就是为什么并发操作通常用于 I/O 的原因；读取或写入磁盘或网络比除了最复杂的内存过程以外的所有过程慢数千倍。如果您不确定并发是否有帮助，请首先将代码串行化，然后编写基准测试来比较并发实现的性能。（参见“使用基准测试”了解如何对您的代码进行基准测试。）

让我们来考虑一个例子。假设您正在编写一个调用其他三个网络服务的 Web 服务。您的程序将数据发送到其中两个服务，然后将这两个调用的结果发送给第三个服务，并返回结果。整个过程必须在 50 毫秒内完成，否则应返回错误。这是并发的一个很好的使用案例，因为代码中有需要执行 I/O 的部分，可以独立运行而无需相互交互，还有一个部分用于合并结果，并且代码需要运行的时间有限。在本章末尾，您将看到如何实现此代码。

# Goroutines

Go 语言并发模型的核心概念是**goroutine**。要理解 goroutine，让我们先定义几个术语。首先是*进程*。进程是计算机操作系统正在运行的程序的实例。操作系统会为进程分配一些资源，如内存，并确保其他进程无法访问这些资源。一个进程由一个或多个*线程*组成。线程是操作系统分配运行时间的执行单位。同一进程内的线程共享资源。根据处理器核心数量，CPU 可以同时执行一个或多个线程的指令。操作系统的一个任务是调度线程在 CPU 上运行，以确保每个进程（及其内部的每个线程）都有运行的机会。

将 Goroutine 看作是由 Go 运行时管理的轻量级线程。当 Go 程序启动时，Go 运行时会创建一些线程，并启动一个单独的 Goroutine 来运行您的程序。您的程序创建的所有 Goroutine（包括初始的）都会由 Go 运行时调度器自动分配到这些线程中，就像操作系统在 CPU 核心之间调度线程一样。这看起来可能是额外的工作，因为底层操作系统已经包含了管理线程和进程的调度器，但它有几个好处：

+   Goroutine 的创建速度比线程快，因为您不需要创建一个操作系统级别的资源。

+   Goroutine 的初始栈大小比线程栈大小小，并且可以根据需要增长。这使得 Goroutine 更加内存高效。

+   在 Goroutine 之间切换比在线程之间切换更快，因为它完全在进程内部进行，避免了（相对而言）缓慢的操作系统调用。

+   Goroutine 调度器能够优化其决策，因为它是 Go 进程的一部分。调度器与网络轮询器一起工作，当 Goroutine 因 I/O 阻塞时检测到可以取消调度。它还与垃圾回收器集成，确保工作在分配给 Go 进程的所有操作系统线程之间得到适当平衡。

这些优势使得 Go 程序能够同时启动数百、数千甚至数万个 Goroutine。如果在具有本地线程的语言中尝试启动数千个线程，您的程序将变得非常缓慢。

###### 提示

如果你对了解调度器如何工作感兴趣，请观看 Kavya Joshi 在 GopherCon 2018 上的演讲 [“调度器传奇”](https://oreil.ly/879mk)。

在函数调用前加上 `go` 关键字可以启动一个 Goroutine。与任何其他函数一样，您可以传递参数来初始化其状态。但是，函数返回的任何值都会被忽略。

任何函数都可以作为 Goroutine 启动。这与 JavaScript 不同，JavaScript 中只有在函数使用 `async` 关键字声明时，该函数才会异步运行。然而，在 Go 中，习惯上使用一个包装业务逻辑的闭包来启动 Goroutine。闭包负责并发的簿记工作。以下示例代码演示了这个概念：

```go
func process(val int) int {
    // do something with val
}

func processConcurrently(inVals []int) []int {
    // create the channels
    in := make(chan int, 5)
    out := make(chan int, 5)
    // launch processing goroutines
    for i := 0; i < 5; i++ {
        go func() {
            for val := range in {
                out <- process(val)
            }
        }()
    }
    // load the data into the in channel in another goroutine
    // read the data from the out channel
    // return the data
}
```

在这段代码中，`processConcurrently` 函数创建了一个闭包，从通道中读取值，并将它们传递给 `process` 函数中的业务逻辑。`process` 函数完全不知道它是在一个 goroutine 中运行。闭包然后将 `process` 的结果写回到另一个通道中。（我将在下一节简要概述通道。）这种责任分离使得你的程序模块化和可测试，并将并发性从你的 API 中分离出去。选择使用类似线程的模型来进行并发意味着 Go 程序避免了 Bob Nystrom 在他著名博客文章 ["你的函数是什么颜色？"](https://oreil.ly/0I_Op) 中描述的“函数着色”问题。

你可以在 [The Go Playground](https://oreil.ly/mw5NU) 上或 [第十二章存储库](https://oreil.ly/uSQBs) 中的 *sample_code/goroutine* 目录中找到完整的示例。

# 通道

Goroutine 使用 *通道* 进行通信。与切片和映射类似，通道是使用 `make` 函数创建的内置类型：

```go
ch := make(chan int)
```

像映射一样，通道也是引用类型。当你将一个通道传递给一个函数时，你实际上是传递了一个指向该通道的指针。与映射和切片一样，通道的零值是 `nil`。

## 读取、写入和缓冲

使用 `<-` 操作符与通道交互。通过将 `<-` 操作符放置在通道变量的左侧来从通道中读取，通过将其放置在右侧来向通道中写入：

```go
a := <-ch // reads a value from ch and assigns it to a
ch <- b   // write the value in b to ch
```

每个写入到通道的值只能被读取一次。如果有多个 goroutine 从同一个通道读取，那么写入到通道的值只会被其中一个读取。

单个 goroutine 很少会同时从同一个通道读取和写入。当将通道分配给变量或字段，或者将其传递给函数时，请在 `chan` 关键字之前使用箭头（`ch <-chan int`）来指示该 goroutine 只从通道中 *读取*。在 `chan` 关键字之后使用箭头（`ch chan<- int`）来指示该 goroutine 只 *写入* 通道。这样做可以让 Go 编译器确保通道只能被函数读取或写入。

默认情况下，通道是 *无缓冲* 的。每次向打开的无缓冲通道写入时，写入 goroutine 都会暂停，直到另一个 goroutine 从同一通道读取。同样地，从打开的无缓冲通道读取时，读取 goroutine 也会暂停，直到另一个 goroutine 向同一通道写入。这意味着你不能在没有至少两个并发运行的 goroutine 的情况下写入或读取无缓冲通道。

Go 语言还拥有 *缓冲* 通道。这些通道可以缓冲有限数量的写入操作而不会阻塞。如果在从通道读取之前缓冲区已满，那么对通道的后续写入会使写入 goroutine 暂停，直到有读取操作发生。与写入到满缓冲区的通道会阻塞一样，从空缓冲区读取的通道也会阻塞。

创建缓冲通道时，可以在创建通道时指定缓冲区的容量：

```go
ch := make(chan int, 10)
```

内置函数`len`和`cap`返回有关缓冲通道的信息。使用`len`可以查找当前缓冲区中有多少个值，使用`cap`可以查找缓冲区的最大容量。缓冲区的容量不能更改。

###### 注意

将无缓冲通道传递给`len`和`cap`都将返回 0。这是有道理的，因为按定义，无缓冲通道没有缓冲区来存储值。

大多数情况下，应该使用无缓冲通道。在“了解何时使用缓冲和非缓冲通道”，我将讨论使用缓冲通道的情况。

## 使用`for-range`和通道

你也可以通过使用`for-range`循环从通道中读取：

```go
for v := range ch {
    fmt.Println(v)
}
```

不像其他`for-range`循环，这里只声明了一个变量用于通道的值，即`v`。如果通道打开并且通道上有值可用，则将其赋给`v`并执行循环体。如果通道上没有可用值，则 goroutine 暂停，直到通道有值可用或通道关闭。循环会继续，直到通道关闭，或达到`break`或`return`语句。

## 关闭通道

当你完成向通道写入数据后，使用内置的`close`函数关闭它：

```go
close(ch)
```

一旦通道关闭，任何尝试向其写入或再次关闭它的操作都会引发恐慌。有趣的是，尝试从关闭的通道读取始终成功。如果通道是带缓冲的，并且还有未读取的值，它们将按顺序返回。如果通道是无缓冲的或缓冲通道没有更多值，则通道的类型的零值将返回。

这引出了一个问题，这个问题在你使用映射时可能会很熟悉：当你的代码从通道中读取时，如何区分是因为通道关闭而返回的零值，还是因为写入的零值？由于 Go 语言试图保持一致性，这里有一个熟悉的答案——使用逗号-ok 惯用法来检测通道是否已关闭：

```go
v, ok := <-ch
```

如果`ok`设置为`true`，则通道是打开的。如果设置为`false`，则通道已关闭。

###### 提示

每当从可能已关闭的通道读取时，请使用逗号-ok 惯用法确保通道仍然打开。

关闭通道的责任属于写入通道的 goroutine。请注意，只有当有 goroutine 在等待通道关闭时（例如使用`for-range`循环从通道读取数据时），才需要关闭通道。由于通道只是另一个变量，Go 的运行时可以检测到不再被引用的通道并对其进行垃圾回收。

通道是 Go 并发模型的两大特色之一。它们引导你将代码视为一系列阶段，并清晰地表达数据依赖关系，这使得推理并发变得更容易。其他语言依赖于全局共享状态来在线程之间通信。这种可变的共享状态使得理解数据如何在程序中流动变得困难，进而难以确定两个线程是否真的是独立的。

## 理解通道的行为方式

通道有多种状态，在读取、写入或关闭时有不同的行为。使用 表 12-1 来理清它们。

表 12-1\. 通道的行为方式

|  | 无缓冲，开放状态 | 无缓冲，关闭状态 | 带缓冲，开放状态 | 带缓冲，关闭状态 | 空 |
| --- | --- | --- | --- | --- | --- |
| 读取 | 等待直到有东西被写入 | 返回零值（使用 comma ok 来查看是否关闭） | 如果缓冲区为空则等待 | 返回缓冲区中的剩余值；如果缓冲区为空则返回零值（使用 comma ok 来查看是否关闭） | 永久挂起 |
| 写入 | 等待直到有东西被读取 | **PANIC** | 如果缓冲区满则等待 | **PANIC** | 永久挂起 |
| 关闭 | 可行 | **PANIC** | 可行，剩余值仍在 | **PANIC** | **PANIC** |

你必须避免导致 Go 程序引发 panic 的情况。如前所述，标准模式是在写入 goroutine 写入完毕后关闭通道。当多个 goroutine 向同一通道写入时，情况会变得更复杂，因为在同一通道上调用两次 `close` 会导致 panic。此外，如果一个 goroutine 中关闭了通道，另一个 goroutine 中写入该通道也会触发 panic。解决这个问题的方法是使用 `sync.WaitGroup`，你将在 “使用 WaitGroups” 中看到一个示例。

`nil` 通道在某些情况下也可能存在风险，但在其他情况下也是有用的。你将在 “关闭 select 中的 case” 中了解更多相关内容。

# `select`

`select` 语句是 Go 并发模型的另一大特色。它是 Go 中的并发控制结构，优雅地解决了一个常见问题：如果可以执行两个并发操作，那么应该先执行哪一个？你不能偏袒某个操作，否则有些情况将永远不会被处理。这被称为 *饥饿* 问题。

`select` 关键字允许一个 goroutine 从多个通道中读取或写入其中一个。它看起来与空的 `switch` 语句非常相似：

```go
select {
case v := <-ch:
    fmt.Println(v)
case v := <-ch2:
    fmt.Println(v)
case ch3 <- x:
    fmt.Println("wrote", x)
case <-ch4:
    fmt.Println("got value on ch4, but ignored it")
}
```

`select` 中的每个 `case` 都是对通道的读取或写入。如果某个 `case` 可以读取或写入，它将与 `case` 的主体一起执行。与 `switch` 类似，`select` 中的每个 `case` 都创建了自己的代码块。

如果多个情况都有可以读取或写入的通道会发生什么？`select`算法很简单：它从可以继续的任何情况中随机选择；顺序不重要。这与`switch`语句非常不同，后者总是选择第一个解析为`true`的`case`。它还清楚地解决了饥饿问题，因为没有一个`case`优先于另一个，并且所有情况同时被检查。

`select`随机选择的另一个优势是它防止了最常见的死锁原因之一：以不一致的顺序获取锁。如果有两个 goroutine 都访问相同的两个通道，它们必须在两个 goroutine 中以相同的顺序访问，否则它们将*死锁*。这意味着两者都无法继续，因为它们正在等待对方。如果您的 Go 应用程序中的每个 goroutine 都死锁，Go 运行时将终止您的程序（参见示例 12-1）。

##### 示例 12-1\. 死锁的 goroutine

```go
func main() {
    ch1 := make(chan int)
    ch2 := make(chan int)
    go func() {
        inGoroutine := 1
        ch1 <- inGoroutine
        fromMain := <-ch2
        fmt.Println("goroutine:", inGoroutine, fromMain)
    }()
    inMain := 2
    ch2 <- inMain
    fromGoroutine := <-ch1
    fmt.Println("main:", inMain, fromGoroutine)
}
```

如果您在[Go Playground](https://oreil.ly/eP3D1)或[第十二章库的 sample_code/deadlock 目录](https://oreil.ly/uSQBs)中运行此程序，您将看到以下错误：

```go
fatal error: all goroutines are asleep - deadlock!
```

请记住，`main`在启动时由 Go 运行时在一个 goroutine 中运行。显式启动的 goroutine 在`ch1`被读取之前无法继续，主 goroutine 在`ch2`被读取之前也无法继续。

如果主 goroutine 中的通道读取和通道写入被包裹在`select`中，则避免了死锁（参见示例 12-2）。

##### 示例 12-2\. 使用`select`避免死锁

```go
func main() {
    ch1 := make(chan int)
    ch2 := make(chan int)
    go func() {
        inGoroutine := 1
        ch1 <- inGoroutine
        fromMain := <-ch2
        fmt.Println("goroutine:", inGoroutine, fromMain)
    }()
    inMain := 2
    var fromGoroutine int
    select {
    case ch2 <- inMain:
    case fromGoroutine = <-ch1:
    }
    fmt.Println("main:", inMain, fromGoroutine)
}
```

如果您在[Go Playground](https://oreil.ly/Djtpj)或[第十二章库的 sample_code/select 目录](https://oreil.ly/uSQBs)中运行此程序，您将获得如下输出：

```go
main: 2 1
```

因为`select`检查其所有情况是否能继续，所以避免了死锁。显式启动的 goroutine 将值 1 写入了`ch1`，因此主 goroutine 中对`ch1`的读取到`fromGoroutine`是成功的。

尽管此程序不会死锁，但仍未能完成正确操作。在启动的 goroutine 中，`fmt.Println`语句永远不会执行，因为该 goroutine 已暂停，等待从`ch2`读取值。当主 goroutine 退出时，程序也退出并终止所有剩余的 goroutine，这在技术上解决了暂停的问题。然而，您应确保所有 goroutine 都能正确退出，以免*泄漏*它们。我在“始终清理您的 goroutine”中详细讨论了这个问题。

###### 注意

使此程序正常运行需要一些您将在本章后面学到的技术。您可以在[Go Playground](https://oreil.ly/G1bi7)上找到一个有效的解决方案。

由于`select`负责在多个通道之间进行通信，因此通常嵌入在`for`循环中：

```go
for {
    select {
    case <-done:
        return
    case v := <-ch:
        fmt.Println(v)
    }
}
```

这种组合通常被称为 `for-select` 循环。在使用 `for-select` 循环时，你必须包含一种退出循环的方法。你会在 “使用上下文终止 Goroutines” 中看到一种方法。

就像 `switch` 语句一样，`select` 语句也可以有一个 `default` 子句。和 `switch` 类似，当没有可以读取或写入的通道时，会选择 `default`。如果你想在通道上实现非阻塞读取或写入，使用带有 `default` 的 `select`。下面的代码在 `ch` 中没有可读取的值时不会等待，而是立即执行 `default` 的主体：

```go
select {
case v := <-ch:
    fmt.Println("read from ch:", v)
default:
    fmt.Println("no value written to ch")
}
```

你将会在 “实现反压力” 中看到 `default` 的用法。

###### 注意

在 `for-select` 循环内部有一个 `default` 案例几乎总是不正确的。当循环中任何一个案例没有读取或写入时，它会在每次循环时触发。这会使得你的 `for` 循环持续运行，消耗大量的 CPU。

# 并发实践和模式

现在你已经了解了 Go 提供的并发基本工具，让我们来看看一些并发的最佳实践和模式。

## 保持你的 API 不受并发的影响

并发是一个实现细节，良好的 API 设计应尽可能隐藏实现细节。这样可以在不改变代码调用方式的情况下改变代码的工作方式。

实际上，这意味着你不应该在 API 的类型、函数和方法中暴露通道或互斥锁（我将在 “何时使用互斥锁而不是通道” 中详细讨论互斥锁）。如果你暴露一个通道，你将把通道管理的责任放在 API 用户身上。用户则需要担心诸如通道是缓冲的、已关闭或 `nil` 的问题。他们也可能通过以意外的顺序访问通道或互斥锁来触发死锁。

###### 注意

这并不意味着你不应该将通道作为函数参数或结构体字段。它意味着它们不应该被导出。

这条规则有一些例外。如果你的 API 是一个带有并发辅助函数的库，通道将成为其 API 的一部分。

## Goroutines、for 循环和变量变化

大多数情况下，用于启动 Goroutine 的闭包没有参数。相反，它捕获了在声明它的环境中的值。在 Go 1.22 之前，有一个常见的情况是这种方式不起作用：尝试捕获 `for` 循环的索引或值。正如 “for-range 值是副本” 和 “使用 go.mod” 中提到的，Go 1.22 引入了一个破坏性变化，改变了 `for` 循环的行为，使其在每次迭代时为索引和值创建新变量，而不是重复使用单个变量。

以下代码演示了这个变更值得的原因。你可以在 GitHub 上的 [`goroutine_for_loop` 仓库](https://oreil.ly/8KkS9) 找到它，这个仓库属于《学习 Go 第二版》组织。

如果你在 Go 1.21 或更早版本运行以下代码（或在 Go 1.22 或更新版本中，`go.mod` 文件的 `go` 指令设置为 1.21 或更早版本），你会看到一个微妙的 bug：

```go
func main() {
    a := []int{2, 4, 6, 8, 10}
    ch := make(chan int, len(a))
    for _, v := range a {
        go func() {
            ch <- v * 2
        }()
    }
    for i := 0; i < len(a); i++ {
        fmt.Println(<-ch)
    }
}
```

对于 `a` 中的每个值，都会启动一个 goroutine。看起来每个 goroutine 接收到的值都不同，但实际运行代码显示的情况却不同：

```go
20
20
20
20
20
```

在早期的 Go 版本中，每个 goroutine 都向 `ch` 写入 `20` 的原因是，每个 goroutine 的闭包捕获了相同的变量。`for` 循环中的索引和数值变量在每次迭代中都被重用。变量 `v` 最后被赋的值是 `10`。当 goroutine 运行时，它们看到的就是这个值。

升级到 Go 1.22 或更高版本，并将 *go.mod* 中的 `go` 指令值改为 1.22 或更高版本，将会改变 `for` 循环的行为，使其在每次迭代时创建新的索引和数值变量。这样会得到预期的结果，每个 goroutine 接收到不同的值：

```go
20
8
4
12
16
```

如果你无法升级到 Go 1.22，可以通过两种方式解决这个问题。首先是在循环内部复制数值来遮蔽数值：

```go
for _, v := range a {
    v := v
    go func() {
        ch <- v * 2
    }()
}
```

如果你想避免遮蔽并使数据流更加清晰，也可以将该值作为参数传递给 goroutine：

```go
for _, v := range a {
    go func(val int) {
        ch <- val * 2
    }(v)
}
```

虽然 Go 1.22 可以避免 `for` 循环中索引和数值变量的问题，但对于闭包中捕获的其他变量仍需小心。每当闭包依赖可能会改变的变量值时，无论是否用作 goroutine，都必须将该值传递给闭包，或者确保为每个引用该变量的闭包创建一个独立的变量副本。

###### 小贴士

每当闭包使用一个可能会改变的变量时，使用参数将该变量的当前值传递给闭包。

## 始终清理你的 Goroutines

每当启动一个 goroutine 函数时，一定要确保它最终会退出。与变量不同，Go 运行时无法检测出 goroutine 是否会再次被使用。如果一个 goroutine 没有退出，那么分配给其栈上变量的所有内存都将保留，任何根据 goroutine 栈上变量分配的堆上内存也无法被垃圾回收。这被称为 *goroutine 泄漏*。

一个 goroutine 并不一定能保证退出。例如，假设你将 goroutine 用作生成器：

```go
func countTo(max int) <-chan int {
    ch := make(chan int)
    go func() {
        for i := 0; i < max; i++ {
            ch <- i
        }
        close(ch)
    }()
    return ch
}

func main() {
    for i := range countTo(10) {
        fmt.Println(i)
    }
}
```

###### 注意

这只是一个简短的示例；不要使用 goroutine 生成数字列表。这太简单了，违反了我们“何时使用并发”的指导原则之一。

在通常情况下，当你使用完所有值时，goroutine 会退出。但是，如果你提前退出循环，goroutine 就会永远阻塞，等待从通道中读取值：

```go
func main() {
    for i := range countTo(10) {
        if i > 5 {
            break
        }
        fmt.Println(i)
    }
}
```

## 使用上下文终止 Goroutines

要解决`countTo` goroutine 泄漏的问题，你需要一种方法告诉 goroutine 现在是停止处理的时候了。在 Go 中，你通过使用*上下文*来解决这个问题。下面是一个重写的`countTo`示例，演示了这种技术。你可以在[第十二章存储库](https://oreil.ly/uSQBs)的*sample_code/context_cancel*目录中找到这段代码。

```go
func countTo(ctx context.Context, max int) <-chan int {
    ch := make(chan int)
    go func() {
        defer close(ch)
        for i := 0; i < max; i++ {
            select {
            case <-ctx.Done():
                return
            case ch <- i:
            }
        }
    }()
    return ch
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    ch := countTo(ctx, 10)
    for i := range ch {
        if i > 5 {
            break
        }
        fmt.Println(i)
    }
}
```

`countTo`函数修改为除了`max`之外还接受一个`context.Context`参数。goroutine 中的`for`循环也已更改。现在是一个带有两个 case 的`for-select`循环。一个尝试向`ch`写入。另一个 case 检查上下文的`Done`方法返回的通道。如果它返回一个值，你退出`for-select`循环和 goroutine。现在，你有了一种方法可以在读取每个值时防止 goroutine 泄漏。

这就引出了一个问题，你如何让`Done`通道返回一个值？它通过*上下文取消*来触发。在`main`函数中，你使用`context`包中的`WithCancel`函数创建了一个上下文和一个取消函数。接下来，你使用`defer`在`main`函数退出时调用`cancel`。这关闭了`Done`返回的通道，并且由于关闭的通道始终返回一个值，它确保运行`countTo`的 goroutine 退出。

使用上下文来终止 goroutine 是一个非常常见的模式。它允许你基于调用堆栈中较早的某些东西来停止 goroutine。在“取消”，你将详细了解如何使用上下文告诉一个或多个 goroutine 现在是时候关闭了。

## 了解何时使用有缓冲和无缓冲通道

在 Go 并发中掌握最复杂的技术之一是决定何时使用缓冲通道。默认情况下，通道是无缓冲的，并且很容易理解：一个 goroutine 写入并等待另一个 goroutine 接管它的工作，就像接力赛中的接力棒一样。缓冲通道则复杂得多。你必须选择一个大小，因为缓冲通道永远不会有无限的缓冲区。正确使用缓冲通道意味着你必须处理缓冲区已满并且你的写入 goroutine 因等待读取 goroutine 而阻塞的情况。那么，什么是正确使用缓冲通道的方式呢？

缓冲通道的情况很微妙。简单地说：当你知道自己启动了多少个 goroutine 时，想要限制你将要启动的 goroutine 数量，或者想要限制排队的工作量时，缓冲通道是有用的。

缓冲通道在你想要从启动的一组 goroutine 中收集数据或者想要限制并发使用时非常有效。它们还有助于管理系统排队的工作量，防止你的服务落后并变得不堪重负。以下是几个示例，展示了它们的使用方式。

在第一个示例中，您正在处理通道上的前 10 个结果。为此，您启动 10 个 goroutine，每个 goroutine 将其结果写入缓冲通道：

```go
func processChannel(ch chan int) []int {
    const conc = 10
    results := make(chan int, conc)
    for i := 0; i < conc; i++ {
        go func() {
            v := <- ch
            results <- process(v)
        }()
    }
    var out []int
    for i := 0; i < conc; i++ {
        out = append(out, <-results)
    }
    return out
}
```

您确切地知道启动了多少个 goroutine，并且希望每个 goroutine 在完成其工作后立即退出。这意味着您可以为每个启动的 goroutine 创建一个带有一个空间的缓冲通道，并让每个 goroutine 向该 goroutine 写入数据而不阻塞。然后，您可以循环遍历缓冲通道，读取写入的值。当所有值都已读取时，您返回结果，知道没有泄漏任何 goroutine。

您可以在[第十二章存储库](https://oreil.ly/uSQBs)的*sample_code/buffered_channel_work*目录中找到此代码。

## 实现背压

另一种可以使用缓冲通道实现的技术是*背压*。这似乎违反直觉，但是当系统组件限制其愿意执行的工作量时，系统整体表现更佳。您可以使用缓冲通道和`select`语句来限制系统中同时请求的数量：

```go
type PressureGauge struct {
    ch chan struct{}
}

func New(limit int) *PressureGauge {
    return &PressureGauge{
        ch: make(chan struct{}, limit),
    }
}

func (pg *PressureGauge) Process(f func()) error {
    select {
    case pg.ch <- struct{}{}:
        f()
        <-pg.ch
        return nil
    default:
        return errors.New("no more capacity")
    }
}
```

在此代码中，您创建一个包含可以容纳多个“令牌”的缓冲通道和要运行的函数的结构体。每次 goroutine 想要使用函数时，它调用`Process`。这是同一个 goroutine 读取和写入同一个通道的少数例子之一。`select`尝试向通道写入令牌。如果可以，函数运行，然后从缓冲通道读取一个令牌。如果无法写入令牌，则运行`default` case，并返回错误。以下是一个快速示例，使用此代码与内置 HTTP 服务器（您将在“服务器”中学习更多关于与 HTTP 的工作）：

```go
func doThingThatShouldBeLimited() string {
    time.Sleep(2 * time.Second)
    return "done"
}

func main() {
    pg := New(10)
    http.HandleFunc("/request", func(w http.ResponseWriter, r *http.Request) {
        err := pg.Process(func() {
            w.Write([]byte(doThingThatShouldBeLimited()))
        })
        if err != nil {
            w.WriteHeader(http.StatusTooManyRequests)
            w.Write([]byte("Too many requests"))
        }
    })
    http.ListenAndServe(":8080", nil)
}
```

您可以在[第十二章存储库](https://oreil.ly/uSQBs)的*sample_code/backpressure*目录中找到此代码。

## 关闭`select`中的 case

当您需要从多个并发源合并数据时，`select`关键字非常有用。但是，您需要正确处理关闭的通道。如果`select`中的某个 case 正在读取一个关闭的通道，则它总是成功的，返回零值。每次选择该 case 时，您需要检查确保该值有效并跳过该 case。如果读取被分散，您的程序将浪费大量时间读取垃圾值。即使非关闭通道上有大量活动，您的程序仍会花费一部分时间从关闭的通道读取，因为`select`随机选择一个 case。

当发生这种情况时，你依赖于看起来像是错误的东西：读取一个`nil`通道。正如你之前看到的，从或写入`nil`通道会导致你的代码永远挂起。虽然如果由于 bug 触发是不好的，但你可以使用`nil`通道来禁用`select`中的`case`。当检测到通道已关闭时，将通道的变量设置为`nil`。相关的 case 将不再运行，因为从`nil`通道读取永远不会返回值。这里是一个`for-select`循环，从两个通道中读取直到两个通道都关闭：

```go
// in and in2 are channels
for count := 0; count < 2; {
    select {
    case v, ok := <-in:
        if !ok {
            in = nil // the case will never succeed again!
            count++
            continue
        }
        // process the v that was read from in
    case v, ok := <-in2:
        if !ok {
            in2 = nil // the case will never succeed again!
            count++
            continue
        }
        // process the v that was read from in2
    }
}
```

你可以在[Go Playground](https://oreil.ly/0nCDz)或*sample_code/close_case*目录中的[第十二章存储库](https://oreil.ly/uSQBs)中尝试这段代码。

## 超时代码

大多数交互式程序必须在一定时间内返回响应。在 Go 语言中，使用并发可以管理请求（或请求的一部分）运行的时间。其他语言在 promise 或 future 之上引入了额外的功能来添加这种功能，但 Go 语言的超时习惯表明了如何从现有部件构建复杂功能。让我们来看一下：

```go
func timeLimitT any T, limit time.Duration) (T, error) {
    out := make(chan T, 1)
    ctx, cancel := context.WithTimeout(context.Background(), limit)
    defer cancel()
    go func() {
        out <- worker()
    }()
    select {
    case result := <-out:
        return result, nil
    case <-ctx.Done():
        var zero T
        return zero, errors.New("work timed out")
    }
}
```

每当你需要在 Go 中限制操作花费的时间时，你会看到这种模式的变体。我在第十四章中讨论上下文，并详细介绍了如何使用超时在“带截止日期的上下文”中。现在，你只需要知道达到超时会取消上下文。上下文的`Done`方法返回一个通道，在上下文由于超时或调用上下文的取消方法而被取消时返回一个值。你可以通过使用`context`包中的`WithTimeout`函数创建一个定时上下文，并使用`time`包中的常量指定等待时间（我将在`time`中更多地讨论`time`包）。

一旦设置了上下文，就在一个 goroutine 中运行工作程序，然后使用`select`选择两种情况之间的一种。第一种情况在工作完成时从`out`通道读取值。第二种情况等待`Done`方法返回的通道返回一个值，就像在“使用上下文终止 Goroutines”中看到的那样。如果是这样，就返回超时错误。你可以写入一个大小为 1 的缓冲通道，以便即使`Done`首先触发，goroutine 中的通道写入也会完成。

你可以在[Go Playground](https://oreil.ly/mTgyA)或*sample_code/time_out*目录中的[第十二章存储库](https://oreil.ly/uSQBs)中尝试这段代码。

###### 注意

如果 `timeLimit` 在 goroutine 完成处理之前退出，那么 goroutine 会继续运行，最终将返回的值写入缓冲通道并退出。你只是不处理返回的结果。如果想在不再等待 goroutine 完成时停止其工作，可以使用上下文取消，我将在 “Cancellation” 中讨论。

## 使用 WaitGroups

有时一个 goroutine 需要等待多个 goroutine 完成它们的工作。如果你等待单个 goroutine，可以使用之前看到的上下文取消模式。但如果你正在等待多个 goroutine，则需要使用 `WaitGroup`，它位于标准库中的 `sync` 包中。这里是一个简单的例子，你可以在 [The Go Playground](https://oreil.ly/hg7IF) 运行，或者在 [第十二章仓库](https://oreil.ly/uSQBs) 的 *sample_code/waitgroup* 目录中运行：

```go
func main() {
    var wg sync.WaitGroup
    wg.Add(3)
    go func() {
        defer wg.Done()
        doThing1()
    }()
    go func() {
        defer wg.Done()
        doThing2()
    }()
    go func() {
        defer wg.Done()
        doThing3()
    }()
    wg.Wait()
}
```

`sync.WaitGroup` 不需要初始化，只需声明，因为它的零值是有用的。`sync.WaitGroup` 上有三个方法：`Add`，用于增加等待的 goroutine 计数器；`Done`，在 goroutine 完成时减少计数器，并且当它完成时调用；`Wait`，暂停其 goroutine 直到计数器为零。通常只调用一次 `Add`，并指定将要启动的 goroutine 数量。在 goroutine 内部调用 `Done`。为了确保即使 goroutine 恐慌也会调用 `Done`，可以使用 `defer`。

你会注意到，你并没有显式地传递 `sync.WaitGroup`。有两个原因。第一个是你必须确保每个使用 `sync.WaitGroup` 的地方都使用同一个实例。如果将 `sync.WaitGroup` 传递给 goroutine 函数并且不使用指针，则该函数有一个副本，调用 `Done` 不会减少原始的 `sync.WaitGroup`。通过使用闭包捕获 `sync.WaitGroup`，你确保每个 goroutine 引用的是同一个实例。

第二个原因是设计。记住，你应该将并发性从你的 API 中分离出去。正如你之前在通道中看到的，通常的模式是使用一个闭包启动一个 goroutine，该闭包封装业务逻辑周围的并发问题，而函数则提供算法。

让我们看一个更现实的例子。正如我之前提到的，当你有多个 goroutine 向同一个通道写入时，你需要确保只关闭被写入的通道一次。`sync.WaitGroup` 就非常适合这种情况。让我们看看它在一个函数中的运作方式，该函数并发处理通道中的值，将结果收集到一个切片中，并返回该切片：

```go
func processAndGatherT, R any R, num int) []R {
    out := make(chan R, num)
    var wg sync.WaitGroup
    wg.Add(num)
    for i := 0; i < num; i++ {
        go func() {
            defer wg.Done()
            for v := range in {
                out <- processor(v)
            }
        }()
    }
    go func() {
        wg.Wait()
        close(out)
    }()
    var result []R
    for v := range out {
        result = append(result, v)
    }
    return result
}
```

在这个示例中，你启动了一个监控 goroutine，等待所有处理 goroutine 退出。当它们退出时，监控 goroutine 在输出通道上调用`close`。当`out`关闭并且缓冲区为空时，`for-range`通道循环退出。最后，函数返回处理后的值。你可以在[第十二章存储库](https://oreil.ly/uSQBs)的*sample_code/waitgroup_close_once*目录中尝试此代码。

虽然`WaitGroups`很方便，但在协调 goroutine 时不应该是你的首选。仅在所有工作 goroutine 退出后需要进行清理（例如关闭它们写入的通道）时才使用它们。

## 仅运行代码一次

正如我在“避免 init 函数（如果可能的话）”中所述，`init`应该保留用于有效不可变的包级别状态的初始化。然而，有时候你想要*延迟加载*，或者在程序启动后仅调用某些初始化代码一次。这通常是因为初始化相对缓慢，并且可能并非每次程序运行都需要。`sync`包包含一个称为`Once`的方便类型，可以实现这种功能。让我们快速看一下它是如何工作的。假设你有一些需要长时间初始化的代码：

```go
type SlowComplicatedParser interface {
    Parse(string) string
}

func initParser() SlowComplicatedParser {
    // do all sorts of setup and loading here
}
```

下面是如何使用`sync.Once`延迟初始化`SlowComplicatedParser`的方法：

```go
var parser SlowComplicatedParser
var once sync.Once

func Parse(dataToParse string) string {
    once.Do(func() {
        parser = initParser()
    })
    return parser.Parse(dataToParse)
}
```

有两个包级别的变量：`parser`，类型为`SlowComplicatedParser`，以及`once`，类型为`sync.Once`。与`sync.WaitGroup`类似，你无需配置`sync.Once`的实例。这是一个例子，*使零值变得有用*，这是 Go 语言中的一种常见模式。

与`sync.WaitGroup`类似，你必须确保不复制`sync.Once`的实例，因为每个副本都有自己的状态来指示它是否已经被使用过。通常在函数内声明`sync.Once`实例是错误的做法，因为每次函数调用都会创建一个新实例，并且不会记住先前的调用。

在这个示例中，你需要确保`parser`仅初始化一次，因此你将`parser`的值设置在传递给`once`的`Do`方法的闭包中。如果调用`Parse`超过一次，`once.Do`将不会再次执行闭包。

你可以在[The Go Playground](https://oreil.ly/v7qtq)或[第十二章存储库](https://oreil.ly/uSQBs)中的*sample_code/sync_once*目录中尝试此代码。

Go 1.21 添加了一些辅助函数，使得执行函数仅一次变得更加容易：`sync.OnceFunc`、`sync.OnceValue`和`sync.OnceValues`。这三个函数之间唯一的区别是传入函数的返回值数量（分别为零、一个或两个）。`sync.OnceValue`和`sync.OnceValues`函数是通用的，因此它们适应原始函数返回值的类型。

使用这些函数非常简单。你将原始函数传递给辅助函数，然后得到一个仅调用原始函数一次的函数。原始函数返回的值会被缓存。以下是如何使用`sync.OnceValue`重写上一个示例中的`Parse`函数：

```go
var initParserCached func() SlowComplicatedParser = sync.OnceValue(initParser)

func Parse(dataToParse string) string {
    parser := initParserCached()
    return parser.Parse(dataToParse)
}
```

当`initParserCached`变量在包级别被赋值为`sync.OnceValue`返回的函数时，说明`initParser`被传递给它了。第一次调用`initParserCached`时，也会调用`initParser`，并缓存它的返回值。每次后续调用`initParserCached`时，都会返回缓存的值。这意味着你可以去掉包级别的`parser`变量。

你可以在[Go Playground](https://oreil.ly/VrR-s)上尝试这段代码，或者在[第十二章的代码库](https://oreil.ly/uSQBs)的*sample_code/sync_value*目录中尝试。

## 将你的并发工具整合起来

让我们回到本章第一节的示例中。你有一个调用三个 Web 服务的函数。你向其中两个服务发送数据，然后将这两次调用的结果发送给第三个服务，并返回结果。整个过程必须在 50 毫秒内完成，否则将返回错误。

你将从你调用的函数开始：

```go
func GatherAndProcess(ctx context.Context, data Input) (COut, error) {
    ctx, cancel := context.WithTimeout(ctx, 50*time.Millisecond)
    defer cancel()

    ab := newABProcessor()
    ab.start(ctx, data)
    inputC, err := ab.wait(ctx)
    if err != nil {
        return COut{}, err
    }

    c := newCProcessor()
    c.start(ctx, inputC)
    out, err := c.wait(ctx)
    return out, err
}
```

首先要做的是设置一个超时为 50 毫秒的`context.Context`，就像你在“Time Out Code”中看到的一样。

创建完 context 后，使用`defer`确保调用 context 的`cancel`函数。正如我在“Cancellation”中会讨论的，你必须调用这个函数，否则资源会泄漏。

你将`A`和`B`作为两个并行调用的服务名称，所以你将创建一个新的`abProcessor`来调用它们。然后你通过调用`start`方法开始处理，并通过调用`wait`方法等待结果。

当`wait`返回时，你进行标准的错误检查。如果一切顺利，你调用第三个服务，称之为`C`。逻辑与之前相同。通过在`cProcessor`上调用`start`方法开始处理，然后通过在`cProcessor`上调用`wait`方法等待结果。然后返回`wait`方法调用的结果。

这看起来很像标准的顺序代码，没有并发。让我们看看`abProcessor`和`cProcessor`中是如何进行并发的：

```go
type abProcessor struct {
    outA chan aOut
    outB chan bOut
    errs chan error
}

func newABProcessor() *abProcessor {
    return &abProcessor{
        outA: make(chan aOut, 1),
        outB: make(chan bOut, 1),
        errs: make(chan error, 2),
    }
}
```

`abProcessor`有三个字段，全部都是通道。它们是`outA`、`outB`和`errs`。接下来你将看到如何使用这些通道。注意每个通道都是有缓冲的，这样写入它们的 goroutine 在写完后就可以退出，而不必等待读取。`errs`通道的缓冲大小为`2`，因为最多可能会有两个错误写入其中。

接下来是`start`方法的实现：

```go
func (p *abProcessor) start(ctx context.Context, data Input) {
    go func() {
        aOut, err := getResultA(ctx, data.A)
        if err != nil {
            p.errs <- err
            return
        }
        p.outA <- aOut
    }()
    go func() {
        bOut, err := getResultB(ctx, data.B)
        if err != nil {
            p.errs <- err
            return
        }
        p.outB <- bOut
    }()
}
```

`start` 方法启动了两个 goroutine。第一个调用 `getResultA` 来与 `A` 服务通信。如果调用返回错误，就向 `errs` 通道写入。否则，向 `outA` 通道写入。由于这些通道是有缓冲的，所以无论写入哪个通道，goroutine 都不会挂起。同时注意，你将上下文传递给 `getResultA`，这允许它在超时时取消处理。

第二个 goroutine 与第一个完全相同，只是调用 `getResultB` 并在成功时向 `outB` 通道写入。

让我们看看 `ABProcessor` 的 `wait` 方法是什么样的：

```go
func (p *abProcessor) wait(ctx context.Context) (cIn, error) {
    var cData cIn
    for count := 0; count < 2; count++ {
        select {
        case a := <-p.outA:
            cData.a = a
        case b := <-p.outB:
            cData.b = b
        case err := <-p.errs:
            return cIn{}, err
        case <-ctx.Done():
            return cIn{}, ctx.Err()
        }
    }
    return cData, nil
}
```

`abProcessor` 上的 `wait` 方法是你需要实现的最复杂方法。它填充了一个 `cIn` 类型的结构体，该结构体保存从调用 `A` 服务和 `B` 服务返回的数据。你将输出变量 `cData` 定义为 `cIn` 类型。然后有一个 `for` 循环，计数到两个，因为你需要从两个通道读取以成功完成。在循环内部，有一个 `select` 语句。如果从 `outA` 通道读取到一个值，就设置 `cData` 的 `a` 字段。如果从 `outB` 通道读取到一个值，就设置 `cData` 的 `b` 字段。如果从 `errs` 通道读取到一个值，就立即返回错误。最后，如果上下文超时，就从上下文的 `Err` 方法立即返回错误。

一旦从 `p.outA` 通道和 `p.outB` 通道都读取到一个值，你就退出循环并返回输入，用于 `cProcessor` 使用。

`cProcessor` 看起来像是 `abProcessor` 的简化版本：

```go
type cProcessor struct {
    outC chan COut
    errs chan error
}

func newCProcessor() *cProcessor {
    return &cProcessor{
        outC: make(chan COut, 1),
        errs: make(chan error, 1),
    }
}

func (p *cProcessor) start(ctx context.Context, inputC cIn) {
    go func() {
        cOut, err := getResultC(ctx, inputC)
        if err != nil {
            p.errs <- err
            return
        }
        p.outC <- cOut
    }()
}

func (p *cProcessor) wait(ctx context.Context) (COut, error) {
    select {
    case out := <-p.outC:
        return out, nil
    case err := <-p.errs:
        return COut{}, err
    case <-ctx.Done():
        return COut{}, ctx.Err()
    }
}
```

`cProcessor` 结构体有一个输出通道和一个错误通道。

在 `cProcessor` 的 `start` 方法看起来像是 `abProcessor` 的 `start` 方法。它启动一个 goroutine，调用 `getResultC` 处理输入数据，在错误时向 `errs` 通道写入，成功时向 `outC` 通道写入。

最后，`cProcessor` 上的 `wait` 方法是一个简单的 `select` 语句，检查是否有值可以从 `outC` 通道、`errs` 通道或上下文的 `Done` 通道读取。

通过使用 goroutine、通道和 `select` 语句来组织代码，你可以将各个步骤分离，允许独立部分以任何顺序运行和完成，并在依赖部分之间清晰地交换数据。此外，你确保程序的任何部分都不会挂起，并且正确处理既在此函数内设置的超时，也在调用历史中的较早函数内设置的超时。如果你不确信这是否是实现并发的更好方法，请尝试在另一种语言中实现它。你可能会惊讶地发现它有多么困难。

你可以在[第十二章代码库](https://oreil.ly/uSQBs)的 *sample_code/pipeline* 目录中找到这个并发流水线的代码。

# 何时使用互斥锁而不是通道

如果你曾经在其他编程语言中协调线程间数据访问，你可能使用过*互斥锁*。这是*互斥排除*的缩写，互斥锁的作用是限制某些代码的并发执行或对共享数据的访问。受保护的部分称为*临界区*。

Go 语言的创建者设计通道和`select`来管理并发有很好的原因。互斥锁的主要问题是它们会混淆程序中的数据流。当值通过一系列通道从 goroutine 传递到 goroutine 时，数据流是清晰的。对值的访问局限于一次只有一个 goroutine。当互斥锁用于保护一个值时，没有任何指示当前哪个 goroutine 拥有该值的方式，因为所有并发进程共享对值的访问。这使得理解处理顺序变得困难。在 Go 社区中有一句话来描述这种哲学：“通过通信共享内存，不要通过共享内存进行通信。”

有时候，使用互斥锁会更清晰一些，Go 标准库包含了这些情况的互斥锁实现。最常见的情况是当你的 goroutine 读取或写入共享值，但不处理该值。让我们以内存中的多人游戏积分板为例。首先看看如何使用通道来实现这一点。下面是一个函数，你可以启动它作为一个 goroutine 来管理积分板：

```go
func scoreboardManager(ctx context.Context, in <-chan func(map[string]int)) {
    scoreboard := map[string]int{}
    for {
        select {
        case <-ctx.Done():
            return
        case f := <-in:
            f(scoreboard)
        }
    }
}
```

此函数声明了一个映射，然后监听一个通道以便接收一个读取或修改映射的函数，同时监听上下文的 Done 通道以知道何时关闭。让我们创建一个带有写入值到映射的方法的类型：

```go
type ChannelScoreboardManager chan func(map[string]int)

func NewChannelScoreboardManager(ctx context.Context) ChannelScoreboardManager {
    ch := make(ChannelScoreboardManager)
    go scoreboardManager(ctx, ch)
    return ch
}

func (csm ChannelScoreboardManager) Update(name string, val int) {
    csm <- func(m map[string]int) {
        m[name] = val
    }
}
```

更新方法非常直接：只需传递一个将值放入映射中的函数即可。但是如何从积分板中读取？你需要返回一个值。这意味着创建一个在传入函数中被写入的通道：

```go
func (csm ChannelScoreboardManager) Read(name string) (int, bool) {
    type Result struct {
        out int
        ok  bool
    }
    resultCh := make(chan Result)
    csm <- func(m map[string]int) {
        out, ok := m[name]
        resultCh <- Result{out, ok}
    }
    result := <-resultCh
    return result.out, result.ok
}
```

虽然这段代码可以工作，但它很繁琐，并且一次只允许一个读取者。更好的方法是使用互斥锁。标准库中有两种互斥锁实现，都在`sync`包中。第一种是`Mutex`，有两个方法，`Lock`和`Unlock`。调用`Lock`会导致当前 goroutine 暂停，直到另一个 goroutine 当前在临界区内为止。当临界区清除时，锁被当前 goroutine *获取*，并执行临界区中的代码。在`Mutex`上调用`Unlock`方法标记临界区的结束。

第二种互斥锁实现称为`RWMutex`，允许同时拥有读锁和写锁。虽然每次只能有一个写锁进入临界区，但读锁是共享的；多个读者可以同时进入临界区。写锁由`Lock`和`Unlock`方法管理，而读锁由`RLock`和`RUnlock`方法管理。

每当你获取一个互斥锁时，一定要确保释放锁。使用`defer`语句在调用`Lock`或`RLock`之后立即调用`Unlock`：

```go
type MutexScoreboardManager struct {
    l          sync.RWMutex
    scoreboard map[string]int
}

func NewMutexScoreboardManager() *MutexScoreboardManager {
    return &MutexScoreboardManager{
        scoreboard: map[string]int{},
    }
}

func (msm *MutexScoreboardManager) Update(name string, val int) {
    msm.l.Lock()
    defer msm.l.Unlock()
    msm.scoreboard[name] = val
}

func (msm *MutexScoreboardManager) Read(name string) (int, bool) {
    msm.l.RLock()
    defer msm.l.RUnlock()
    val, ok := msm.scoreboard[name]
    return val, ok
}
```

在[第十二章存储库](https://oreil.ly/uSQBs)的*sample_code/mutex*目录中可以找到示例。

现在你已经看到了使用互斥锁的实现，再在使用之前慎重考虑你的选择。Katherine Cox-Buday 的优秀著作[*《Go 语言并发编程》*](https://oreil.ly/G7bpu)（O’Reilly）包含一个决策树，帮助你决定是使用通道还是互斥锁：

+   如果你在协调 goroutine 或跟踪值在一系列 goroutine 中的转换过程中，请使用通道。

+   如果你正在共享结构体中的字段访问，请使用互斥锁。

+   如果在使用通道时发现了关键性能问题（参见“使用基准测试”了解如何做到这一点），并且找不到其他解决方法来修复问题，请修改你的代码以使用互斥锁。

因为你的记分板是结构体中的一个字段，并且没有记分板的传递，使用互斥锁是合理的。只有在数据存储在内存中时，才是互斥锁的良好使用场景。当数据存储在外部服务（如 HTTP 服务器或数据库）中时，请勿使用互斥锁来保护系统访问。

互斥锁需要你进行更多的簿记。例如，你必须正确配对锁定和解锁，否则你的程序很可能会死锁。示例中在同一方法中同时获取和释放锁。另一个问题是 Go 中的互斥锁不是*可重入*的。如果一个 goroutine 尝试两次获取同一个锁，它将死锁，等待自己释放锁。这与像 Java 这样的语言不同，那里的锁是可重入的。

非可重入锁使得在调用自身递归的函数中获取锁变得棘手。你必须在递归函数调用之前释放锁。总体而言，在持有锁时调用函数时要小心，因为你不知道这些调用中会获取哪些锁。如果你的函数调用另一个尝试获取相同互斥锁的函数，则 goroutine 将死锁。

像`sync.WaitGroup`和`sync.Once`一样，互斥锁绝不能被复制。如果它们被传递给一个函数或作为结构体上的字段访问，必须通过指针。如果复制互斥锁，则其锁将不会被共享。

###### 警告

永远不要尝试从多个 goroutine 访问一个变量，除非你首先为该变量获取互斥锁。这可能导致难以追踪的奇怪错误。参见“使用数据竞争检测器查找并发问题”了解如何检测这些问题。

# 原子操作——你可能不需要这些操作。

除了互斥锁，Go 还提供了另一种方式来在多个线程中保持数据一致性。`sync/atomic`包提供了访问现代 CPU 内置的*原子变量*操作，包括添加、交换、加载、存储或比较并交换（CAS）适合单个寄存器的值。

如果您需要尽可能提高性能，并且是编写并发代码的专家，您会很高兴知道 Go 包含原子支持。对于其他人，请使用 goroutine 和互斥锁来管理您的并发需求。

# 关于并发学习更多信息

我在这里介绍了一些简单的并发模式，但实际上还有很多。事实上，您可以撰写一本关于如何在 Go 中正确实现各种并发模式的完整书籍，幸运的是，Katherine Cox-Buday 已经做到了。在讨论互斥锁或通道如何选择时，我已经提到了*Concurrency in Go*，但它是关于 Go 和并发的优秀资源。如果您想了解更多，请查看她的书籍。

# 练习

有效地使用并发是 Go 开发人员最重要的技能之一。通过这些练习来检验您是否掌握了它们。解决方案可以在[第十二章的存储库](https://oreil.ly/uSQBs)中的*exercise_solutions*目录中找到。

1.  创建一个函数，启动三个使用通道进行通信的 goroutine。前两个 goroutine 各自向通道写入 10 个数字。第三个 goroutine 从通道中读取所有数字并打印出来。函数在所有值打印完毕后应退出。确保没有 goroutine 泄漏。如有需要，您可以创建额外的 goroutine。

1.  创建一个函数，启动两个 goroutine。每个 goroutine 向自己的通道写入 10 个数字。使用`for-select`循环从两个通道读取数据，打印出数字及其写入该值的 goroutine。确保在所有值都被读取后函数退出，并且没有 goroutine 泄漏。

1.  编写一个函数，构建一个`map[int]float64`，其中键是从 0（包括）到 100,000（不包括）的数字，而值是这些数字的平方根（使用[`math.Sqrt`](https://oreil.ly/DPNYi)函数计算平方根）。使用`sync.OnceValue`生成一个函数，缓存此函数返回的`map`，并使用缓存值来查找从 0 到 100,000 每 1000 个数字的平方根。

# 总结

在本章中，您已经了解了并发性，并学习了为什么 Go 的方法比传统的并发机制更简单。在此过程中，您还学会了何时应该使用并发，以及一些并发规则和模式。在下一章中，您将快速浏览 Go 的标准库，这个库以现代计算的“一揽子”理念为核心。
