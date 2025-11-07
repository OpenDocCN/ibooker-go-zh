## 第七章\. 并发模式

*在本章中*

+   控制程序的生命周期

+   管理可重用资源池

+   创建一个可以处理工作的 goroutine 池

在第六章（kindle_split_014.html#ch06）中，你学习了并发是什么以及通道是如何工作的，并回顾了展示并发操作的代码。在本章中，你将通过回顾更多代码来扩展这些知识。我们将回顾三个实现不同并发模式的包，你可以在自己的项目中使用这些模式。每个包都提供了关于并发和通道的使用以及它们如何使并发程序更容易编写和推理的实用视角。

### 7.1\. Runner

`runner` 包的目的是展示如何使用通道来监控程序运行的时间，并在程序运行时间过长时终止程序。当开发计划作为后台任务进程运行的程序时，这种模式很有用。这可能是一个作为 cron 作业运行的程序，或者在基于工作者的云环境（如 Iron.io）中运行。 

让我们看看 `runner` 包中的 runner.go 代码文件。

##### 列表 7.1\. `runner`/runner.go

```
 01 // Example provided with help from Gabriel Aszalos.
 02 // Package runner manages the running and lifetime of a process.
 03 package runner
 04
 05 import (
 06     "errors"
 07     "os"
 08     "os/signal"
 09     "time"
 10 )
 11
 12 // Runner runs a set of tasks within a given timeout and can be
 13 // shut down on an operating system interrupt.
 14 type Runner struct {
 15     // interrupt channel reports a signal from the
 16     // operating system.
 17     interrupt chan os.Signal
 18
 19     // complete channel reports that processing is done.
 20     complete chan error
 21
 22     // timeout reports that time has run out.
 23     timeout <-chan time.Time
 24
 25     // tasks holds a set of functions that are executed
 26     // synchronously in index order.
 27     tasks []func(int)
 28 }
 29
 30 // ErrTimeout is returned when a value is received on the timeout.
 31 var ErrTimeout = errors.New("received timeout")
 32
 33 // ErrInterrupt is returned when an event from the OS is received.
 34 var ErrInterrupt = errors.New("received interrupt")
 35
 36 // New returns a new ready-to-use Runner.
 37 func New(d time.Duration) *Runner {
 38     return &Runner{
 39         interrupt: make(chan os.Signal, 1),
 40         complete:  make(chan error),
 41         timeout:   time.After(d),
 42     }
 43 }
 44
 45 // Add attaches tasks to the Runner. A task is a function that
 46 // takes an int ID.
 47 func (r *Runner) Add(tasks ...func(int)) {
 48     r.tasks = append(r.tasks, tasks...)
 49 }
 50
 51 // Start runs all tasks and monitors channel events.
 52 func (r *Runner) Start() error {
 53     // We want to receive all interrupt based signals.

 54     signal.Notify(r.interrupt, os.Interrupt)
 55
 56     // Run the different tasks on a different goroutine.
 57     go func() {
 58         r.complete <- r.run()
 59     }()
 60
 61     select {
 62     // Signaled when processing is done.
 63     case err := <-r.complete:
 64         return err
 65
 66     // Signaled when we run out of time.
 67     case <-r.timeout:
 68         return ErrTimeout
 69     }
 70 }
 71
 72 // run executes each registered task.
 73 func (r *Runner) run() error {
 74     for id, task := range r.tasks {
 75         // Check for an interrupt signal from the OS.
 76         if r.gotInterrupt() {
 77             return ErrInterrupt
 78         }
 79
 80         // Execute the registered task.
 81         task(id)
 82     }
 83
 84     return nil
 85 }
 86
 87 // gotInterrupt verifies if the interrupt signal has been issued.
 88 func (r *Runner) gotInterrupt() bool {
 89     select {
 90     // Signaled when an interrupt event is sent.
 91     case <-r.interrupt:
 92         // Stop receiving any further signals.
 93         signal.Stop(r.interrupt)
 95         return true
 96
 97     // Continue running as normal.
 98     default:
 99         return false
100     }
101 }
```

列表 7.1 中的程序展示了面向任务的程序在计划中无人看管运行时的并发模式。它设计了三个可能的终止点：

+   程序可以在规定的时间内完成其工作并正常终止。

+   程序未能按时完成并自行终止。

+   接收到操作系统中断事件，程序尝试立即干净地关闭。

让我们分析代码，看看每个点是如何实现的。

##### 列表 7.2\. `runner`/runner.go: 行 12–28

```
12 // Runner runs a set of tasks within a given timeout and can be
13 // shut down on an operating system interrupt.
14 type Runner struct {
15     // interrupt channel reports a signal from the
16     // operating system.
17     interrupt chan os.Signal
18
19     // complete channel reports that processing is done.
20     complete chan error
21
22     // timeout reports that time has run out.
23     timeout <-chan time.Time
24
25     // tasks holds a set of functions that are executed
26     // synchronously in index order.
27     tasks []func(int)
28 }
```

列表 7.2 从第 14 行开始声明名为 `Runner` 的结构体。此类型声明了三个用于管理程序生命周期的通道和一个表示要按顺序运行的不同任务的函数切片。

第 17 行的 `interrupt` 通道发送和接收 `os.Signal` 类型的接口值，并用于接收来自宿主操作系统的中断事件。

##### 列表 7.3\. golang.org/pkg/os/#Signal

```
// A Signal represents an operating system signal. The usual underlying
// implementation is operating system-dependent: on Unix it is
// syscall.Signal.
type Signal interface {
    String() string
    Signal() // to distinguish from other Stringers
}
```

`os.Signal` 接口的声明在 列表 7.3 中展示。此接口抽象了来自不同操作系统的捕获和报告事件的特定实现。

第二个字段名为 `complete`，是一个发送和接收 `error` 类型的接口值的通道。

##### 列表 7.4\. `runner`/runner.go: 行 19–20

```
19     // complete channel reports that processing is done.
20     complete chan error
```

这个通道被称为 `complete`，因为它被运行任务的 goroutine 用于表示通道已完成。如果发生错误，它将通过通过通道发送的 `error` 接口值返回。如果没有发生错误，则发送 `nil` 作为错误接口值。

第三个字段名为 `timeout`，接收 `time.Time` 类型的值。

##### 列表 7.5\. `runner`/runner.go: 行 22–23

```
22     // timeout reports that time has run out.
23     timeout <-chan time.Time
```

这个通道用于管理进程完成所有任务所需的时间量。如果这个通道上收到了 `time.Time` 值，程序将尝试干净地关闭自己。

最后一个字段被命名为 `tasks`，它是一个函数值的切片。

##### 列表 7.6\. `runner`/runner.go: 行 25–27

```
25     // tasks holds a set of functions that are executed
26     // synchronously in index order.
27     tasks []func(int)
```

这些函数值代表了一系列依次运行的函数。这些函数的执行在一个单独但独立的 goroutine 上从 `main` 进行。

在声明了 `Runner` 类型之后，接下来我们有两个 `error` 接口变量。

##### 列表 7.7\. `runner`/runner.go: 行 30–34

```
30 // ErrTimeout is returned when a value is received on the timeout.
31 var ErrTimeout = errors.New("received timeout")
32
33 // ErrInterrupt is returned when an event from the OS is received.
34 var ErrInterrupt = errors.New("received interrupt")
```

第一个 `error` 接口变量被命名为 `ErrTimeout`。当接收到超时事件时，`Start` 方法返回这个错误值。第二个 `error` 接口变量被命名为 `ErrInterrupt`。当接收到操作系统事件时，`Start` 方法返回这个错误值。

现在，我们可以看看用户如何创建 `Runner` 类型的值。

##### 列表 7.8\. `runner`/runner.go: 行 36–43

```
36 // New returns a new ready-to-use Runner.
37 func New(d time.Duration) *Runner {
38     return &Runner{
39         interrupt: make(chan os.Signal, 1),
40         complete:  make(chan error),

41         timeout:   time.After(d),
42     }
43 }
```

列表 7.8 展示了一个名为 `New` 的工厂函数，它接受一个 `time.Duration` 类型的值，并返回一个 `Runner` 类型的指针。该函数创建一个 `Runner` 类型的值，并初始化每个通道字段。`tasks` 字段没有显式初始化，因为这个字段的零值是一个 `nil` 切片。每个通道字段都有一个独特的初始化，所以让我们更详细地探索每一个。

`interrupt` 通道被初始化为一个带有 `1` 个缓冲的缓冲通道。这保证了至少会从运行时接收到一个 `os.Signal` 值。运行时将以非阻塞的方式发送这个事件。如果一个 goroutine 没有准备好接收这个值，这个值就会被丢弃。例如，如果用户反复按 Ctrl+C，程序只有在通道中有缓冲并且所有其他事件都被丢弃时才会接收到这个事件。

`complete` 通道被初始化为一个无缓冲通道。当运行任务的 goroutine 完成时，它会通过这个通道发送一个 `error` 值或 `nil error` 值。然后它等待 `main` 接收它。一旦 `main` 接收到 `error` 值，goroutine 就可以安全地终止。

最后一个通道，`timeout`，使用 `time` 包中的 `After` 函数初始化。`After` 函数返回一个 `time.Time` 类型的通道。运行时将在指定持续时间过后通过这个通道发送一个 `time.Time` 值。

现在你已经看到了如何创建和初始化 `Runner` 值，我们可以看看与 `Runner` 类型相关的方法。第一个方法，`Add`，用于捕获要执行的函数任务。

##### 列表 7.9\. `runner`/runner.go: 行 45–49

```
45 // Add attaches tasks to the Runner. A task is a function that
46 // takes an int ID.
47 func (r *Runner) Add(tasks ...func(int)) {
48     r.tasks = append(r.tasks, tasks...)
49 }
```

列表 7.9 显示了`Add`方法，该方法使用一个名为`tasks`的单个可变参数声明。*可变参数*可以接受任何数量的传入值。在这种情况下，值必须是一个接受单个整数值并返回空值的函数。一旦进入代码，`tasks`参数就变成了这些函数值的切片。

现在我们来看看`run`方法。

##### 列表 7.10\. `runner`/runner.go: 第 72-85 行

```
72 // run executes each registered task.
73 func (r *Runner) run() error {
74     for id, task := range r.tasks {

75         // Check for an interrupt signal from the OS.
76         if r.gotInterrupt() {
77             return ErrInterrupt
78         }
79
80         // Execute the registered task.
81         task(id)
82     }
83
84     return nil
85 }
```

在列表 7.10 的第 73 行上的`run`方法遍历`tasks`切片，并按顺序执行每个函数。在执行任何函数之前（第 81 行），在第 76 行调用`gotInterrupt`方法，以查看是否有来自操作系统的任何事件要接收。

##### 列表 7.11\. `runner`/runner.go: 第 87-101 行

```
 87 // gotInterrupt verifies if the interrupt signal has been issued.
 88 func (r *Runner) gotInterrupt() bool {
 89     select {
 90     // Signaled when an interrupt event is sent.
 91     case <-r.interrupt:
 92         // Stop receiving any further signals.
 93         signal.Stop(r.interrupt)
 95         return true
 96
 97     // Continue running as normal.
 98     default:
 99         return false
100     }
101 }
```

列表 7.11 中的`gotInterrupt`方法展示了带有`default`情况的`select`语句的经典用法。在第 91 行，代码尝试在`interrupt`通道上接收。通常如果没有要接收的内容，它会阻塞，但我们有第 98 行的`default`情况。`default`情况将尝试在`interrupt`通道上接收的操作转换为非阻塞调用。如果有中断要接收，则接收并处理它。如果没有要接收的内容，则执行`default`情况。

当接收到中断事件时，代码通过在第 93 行调用`Stop`方法请求停止接收任何未来的事件。然后函数返回`true`。如果没有接收到中断事件，方法在第 99 行返回`false`。本质上，`gotInterrupt`方法允许 goroutine 检查中断事件，并在没有发出事件的情况下继续处理工作。

包中的最后一个方法称为`Start`。

##### 列表 7.12\. `runner`/runner.go: 第 51-70 行

```
51 // Start runs all tasks and monitors channel events.
52 func (r *Runner) Start() error {

53     // We want to receive all interrupt based signals.
54     signal.Notify(r.interrupt, os.Interrupt)
55
56     // Run the different tasks on a different goroutine.
57     go func() {
58         r.complete <- r.run()
59     }()
60
61     select {
62     // Signaled when processing is done.
63     case err := <-r.complete:
64         return err
65
66     // Signaled when we run out of time.
67     case <-r.timeout:
68         return ErrTimeout
69     }
70 }
```

`Start`方法实现了程序的主要工作流程。在列表 7.12 的第 52 行，`Start`设置了`gotInterrupt`方法接收来自操作系统的中断事件的能力。在第 56 到 59 行，声明并创建了一个匿名函数作为 goroutine。这是执行程序分配的任务集的 goroutine。在第 58 行，在这个 goroutine 内部，调用了`run`方法，并将返回的`error`接口值发送到`complete`通道。一旦接收到`error`接口值，goroutine 将返回该值给调用者。

创建 goroutine 后，`Start`进入一个`select`语句，阻塞等待两个事件之一发生。如果在`complete`通道上收到`error`接口值，则 goroutine 要么在分配的时间内完成了工作，要么收到了操作系统中断事件。无论如何，都会返回接收到的`error`接口值，方法终止。如果在`timeout`通道上收到`time.Time`值，则 goroutine 没有在分配的时间内完成工作。在这种情况下，程序返回`ErrTimeout`变量。

现在你已经看到了`runner`包的代码，并了解了它是如何工作的，让我们回顾`main.go`源代码文件中的测试程序。

##### 列表 7.13\. `runner`/main/main.go

```
01 // This sample program demonstrates how to use a channel to
02 // monitor the amount of time the program is running and terminate
03 // the program if it runs too long.
03 package main
04
05 import (
06     "log"
07     "time"
08
09     "github.com/goinaction/code/chapter7/patterns/runner"
10 )

11
12 // timeout is the number of second the program has to finish.
13 const timeout = 3 * time.Second
14
15 // main is the entry point for the program.
16 func main() {
17     log.Println("Starting work.")
18
19     // Create a new timer value for this run.
20     r := runner.New(timeout)
21
22     // Add the tasks to be run.
23     r.Add(createTask(), createTask(), createTask())
24
25     // Run the tasks and handle the result.
26     if err := r.Start(); err != nil {
27         switch err {
28         case runner.ErrTimeout:
29             log.Println("Terminating due to timeout.")
30             os.Exit(1)
31         case runner.ErrInterrupt:
32             log.Println("Terminating due to interrupt.")
33             os.Exit(2)
34         }
35     }
36
37     log.Println("Process ended.")
38 }
39
40 // createTask returns an example task that sleeps for the specified
41 // number of seconds based on the id.
42 func createTask() func(int) {
43     return func(id int) {
44         log.Printf("Processor - Task #%d.", id)
45         time.Sleep(time.Duration(id) * time.Second)
46     }
47 }
```

`main`函数可以在列表 7.13 的第 16 行找到。在第 20 行，将超时值传递给`New`函数，并返回一个类型为`Runner`的指针。然后，在第 23 行多次将`createTask`函数添加到`Runner`中。在第 42 行声明的`createTask`函数是一个假装执行指定时间工作的函数。一旦添加了函数，就在第 26 行调用`Start`方法，`main`函数等待`Start`返回。

当`Start`返回时，会检查返回的`error`接口值。如果发生了错误，代码使用`error`接口变量来识别方法是否由于超时事件或中断事件而终止。如果没有错误，则任务按时完成。在超时事件中，程序以错误代码 1 终止。在中断事件中，程序以错误代码 2 终止。在其他所有情况下，程序以错误代码 0 正常终止。

### 7.2\. 池化

`pool`包的目的是展示如何使用缓冲通道来收集一组资源，这些资源可以被任何数量的 goroutine 共享和单独使用。当你有一组静态资源需要共享时，这种模式很有用，例如数据库连接或内存缓冲区。当一个 goroutine 需要从池中获取这些资源之一时，它可以获取资源，使用它，然后将其返回到池中。

让我们看看`pool`包中的`pool.go`代码文件。

##### 列表 7.14\. `pool`/pool.go

```
 01 // Example provided with help from Fatih Arslan and Gabriel Aszalos.
 02 // Package pool manages a user defined set of resources.
 03 package pool
 04
 05 import (
 06     "errors"
 07     "log"
 08     "io"
 09     "sync"
 10 )
 11
 12 // Pool manages a set of resources that can be shared safely by
 13 // multiple goroutines. The resource being managed must implement
 14 // the io.Closer interface.
 15 type Pool struct {
 16     m         sync.Mutex
 17     resources chan io.Closer
 18     factory   func() (io.Closer, error)
 19     closed    bool
 20 }
 21
 22 // ErrPoolClosed is returned when an Acquire returns on a
 23 // closed pool.
 24 var ErrPoolClosed = errors.New("Pool has been closed.")
 25
 26 // New creates a pool that manages resources. A pool requires a
 27 // function that can allocate a new resource and the size of
 28 // the pool.
 29 func New(fn func() (io.Closer, error), size uint) (*Pool, error) {
 30     if size <= 0 {
 31         return nil, errors.New("Size value too small.")
 32     }
 33
 34     return &Pool{
 35         factory:   fn,
 36         resources: make(chan io.Closer, size),
 37     }, nil
 38 }
 39
 40 // Acquire retrieves a resource from the pool.
 41 func (p *Pool) Acquire() (io.Closer, error) {

 42     select {
 43     // Check for a free resource.
 44     case r, ok := <-p.resources:
 45         log.Println("Acquire:", "Shared Resource")
 46         if !ok {
 47             return nil, ErrPoolClosed
 48         }
 49         return r, nil
 50
 51     // Provide a new resource since there are none available.
 52     default:
 53         log.Println("Acquire:", "New Resource")
 54         return p.factory()
 55     }
 56 }
 57
 58 // Release places a new resource onto the pool.
 59 func (p *Pool) Release(r io.Closer) {
 60     // Secure this operation with the Close operation.
 61     p.m.Lock()
 62     defer p.m.Unlock()
 63
 64     // If the pool is closed, discard the resource.
 65     if p.closed {
 66         r.Close()
 67         return
 68     }
 69
 70     select {
 71     // Attempt to place the new resource on the queue.
 72     case p.resources <- r:
 73         log.Println("Release:", "In Queue")
 74
 75     // If the queue is already at capacity we close the resource.
 76     default:
 77         log.Println("Release:", "Closing")
 78         r.Close()
 79     }
 80 }
 81
 82 // Close will shutdown the pool and close all existing resources.
 83 func (p *Pool) Close() {
 84     // Secure this operation with the Release operation.
 85     p.m.Lock()
 86     defer p.m.Unlock()
 87
 88     // If the pool is already closed, don't do anything.
 89     if p.closed {
 90         return
 91     }
 92
 93     // Set the pool as closed.
 94     p.closed = true
 95
 96     // Close the channel before we drain the channel of its

 97     // resources. If we don't do this, we will have a deadlock.
 98     close(p.resources)
 99
100     // Close the resources
101     for r := range p.resources {
102         r.Close()
103     }
104 }
```

列表 7.14 中的`pool`包的代码声明了一个名为`Pool`的结构体，允许调用者创建所需数量的不同池。只要类型实现了`io.Closer`接口，每个池都可以管理任何类型的资源。让我们看看`Pool`结构体的声明。

##### 列表 7.15\. `pool`/pool.go: 行 12–20

```
12 // Pool manages a set of resources that can be shared safely by
13 // multiple goroutines. The resource being managed must implement
14 // the io.Closer interface.
15 type Pool struct {
16     m         sync.Mutex
17     resources chan io.Closer
18     factory   func() (io.Closer, error)
19     closed    bool
20 }
```

`Pool` 结构体声明了四个字段，每个字段都帮助以 goroutine 安全的方式管理池。在第 16 行，结构体开始于一个类型为 `sync.Mutex` 的字段。这个互斥锁用于确保对 `Pool` 值的所有操作在多 goroutine 访问时是安全的。第二个字段名为 `resources`，声明为一个接口类型 `io.Closer` 的通道。这个通道将被创建为一个带缓冲的通道，并将包含共享的资源。由于使用了接口类型，池可以管理任何实现了 `io.Closer` 接口的资源类型。

`factory` 字段是一个函数类型。任何不接受参数并返回 `io.Closer` 和 `error` 接口值的函数都可以分配给此字段。此函数的目的是在池需要时创建一个新的资源。这种功能是 `pool` 包范围之外的实施细节，需要由使用此包的用户实现和提供。

第 19 行的最后一个字段是 `closed` 字段。此字段是一个标志，表示 `Pool` 正在关闭或已经关闭。现在你已经看到了 `Pool` 结构体的声明，让我们看看第 24 行声明的 `error` 接口变量。

##### 列表 7.16\. `pool`/pool.go: 行 22–24

```
22 // ErrPoolClosed is returned when an Acquire returns on a
23 // closed pool.
24 var ErrPoolClosed = errors.New("Pool has been closed.")
```

在 Go 中创建 `error` 接口变量是一种常见做法。这允许调用者从包中的任何函数或方法中识别特定的返回错误值。在 列表 7.16 中的 `error` 接口变量已被声明，用于报告当用户调用 `Acquire` 方法且 `Pool` 已关闭时的情况。由于 `Acquire` 方法可以返回多个不同的错误，当 `Pool` 关闭时返回此错误变量允许调用者识别这个特定的错误而不是其他错误。

在声明了 `Pool` 类型和 `error` 接口变量之后，我们可以开始查看在 `pool` 包中声明的函数和方法。让我们从池的工厂函数 `New` 开始。

##### 列表 7.17\. `pool`/pool.go: 行 26–38

```
26 // New creates a pool that manages resources. A pool requires a
27 // function that can allocate a new resource and the size of
28 // the pool.
29 func New(fn func() (io.Closer, error), size uint) (*Pool, error) {
30     if size <= 0 {
31         return nil, errors.New("Size value too small.")
32     }
33
34     return &Pool{
35         factory:   fn,
36         resources: make(chan io.Closer, size),
37     }, nil
38 }
```

在 列表 7.17 中的 `New` 函数接受两个参数并返回两个值。第一个参数 `fn` 被声明为一个不接受参数并返回 `io.Closer` 和 `error` 接口值的函数类型。函数参数代表一个工厂函数，用于创建池管理的资源值。第二个参数 `size` 代表创建用于存储资源的带缓冲通道的大小。

在第 30 行，检查`size`的值以确保它不小于或等于 0。如果是，则代码返回`nil`作为返回的`pool`指针值，并即时创建一个错误接口值。由于这是此函数返回的唯一错误，因此不需要为此错误创建和使用错误接口变量。如果大小值有效，则创建并初始化一个新的`Pool`值。在第 35 行，将函数参数赋值，在第 36 行使用大小值创建缓冲通道。所有创建和初始化都可以在`return`语句的作用域内完成。因此，创建并返回指向新`Pool`值的指针和`nil`的错误接口值作为参数。

在能够创建和初始化`Pool`值之后，接下来让我们看看`Acquire`方法。此方法允许调用者从池中获取资源。

##### 列表 7.18\. `pool`/pool.go: 第 40–56 行

```
40 // Acquire retrieves a resource from the pool.
41 func (p *Pool) Acquire() (io.Closer, error) {
42     select {
43     // Check for a free resource.
44     case r, ok := <-p.resources:
45         log.Println("Acquire:", "Shared Resource")
46         if !ok {
47             return nil, ErrPoolClosed
48         }
49         return r, nil
50
51     // Provide a new resource since there are none available.
52     default:
53         log.Println("Acquire:", "New Resource")
54         return p.factory()
55     }
56 }
```

列表 7.18 包含`Acquire`方法的代码。此方法如果池中有可用资源，则返回资源，或者为调用创建一个新的。此实现是通过使用`select / case`语句来检查缓冲通道中是否有资源来完成的。如果有，则接收并返回给调用者。这可以在第 44 行和第 49 行看到。如果没有资源在缓冲通道中接收，则执行`default`情况。在这种情况下，在第 54 行执行用户的工厂函数，创建一个新的资源并返回。

在获取资源并且不再需要后，必须将其释放回池中。这就是`Release`方法发挥作用的地方。但要理解`Release`方法背后的代码机制，我们首先需要查看`Close`方法。

##### 列表 7.19\. `pool`/pool.go: 第 82–104 行

```
 82 // Close will shutdown the pool and close all existing resources.
 83 func (p *Pool) Close() {
 84     // Secure this operation with the Release operation.
 85     p.m.Lock()
 86     defer p.m.Unlock()
 87
 88     // If the pool is already closed, don't do anything.
 89     if p.closed {
 90         return
 91     }
 92
 93     // Set the pool as closed.
 94     p.closed = true
 95
 96     // Close the channel before we drain the channel of its
 97     // resources. If we don't do this, we will have a deadlock.
 98     close(p.resources)
 99

100     // Close the resources
101     for r := range p.resources {
102         r.Close()
103     }
104 }
```

一旦程序完成对池的使用，它应该调用`Close`方法。`Close`方法的代码在列表 7.19 中显示。该方法在第 98 行和第 101 行关闭并清空缓冲通道，关闭任何存在的资源，直到通道为空。此方法中的所有代码必须一次只由一个 goroutine 执行。实际上，当此代码正在执行时，goroutine 还必须防止在`Release`方法中执行代码。你很快就会明白为什么这很重要。

在第 85 行和第 86 行，互斥锁被锁定，并在函数返回时计划解锁。在第 89 行，检查`closed`标志以查看池是否已经关闭。如果是，方法立即返回，从而释放锁。如果是第一次调用该方法，则将标志设置为`true`，并关闭并清空`resources`通道。

现在我们可以查看`Release`方法，并了解它是如何与`Close`方法协调工作的。

##### 列表 7.20\. `pool`/pool.go: 第 58–80 行

```
58 // Release places a new resource onto the pool.
59 func (p *Pool) Release(r io.Closer) {
60     // Secure this operation with the Close operation.
61     p.m.Lock()
62     defer p.m.Unlock()
63
64     // If the pool is closed, discard the resource.
65     if p.closed {
66         r.Close()
67         return
68     }
69
70     select {
71     // Attempt to place the new resource on the queue.
72     case p.resources <- r:
73         log.Println("Release:", "In Queue")
74
75     // If the queue is already at capacity we close the resource.
76     default:
77         log.Println("Release:", "Closing")
78         r.Close()
79     }
80 }
```

`Release` 方法的实现可以在 代码列表 7.20 中找到。该方法从第 61 和 62 行开始锁定和解锁互斥锁。这与 `Close` 方法中的互斥锁相同。这就是如何通过不同的 goroutine 防止两种方法同时运行。互斥锁的使用有两个目的。首先，它保护第 65 行的 `closed` 标志的读取不会与 `Close` 方法中此标志的写入同时发生。其次，我们不希望尝试向已关闭的通道发送数据，因为这会导致恐慌。当 `closed` 字段为 `false` 时，我们知道 `resources` 通道已被关闭。

在第 66 行，当池关闭时，直接调用资源上的 `Close` 方法。这是因为没有方法可以将资源释放回池中。此时，池已被关闭和刷新。对 `closed` 标志的读写必须同步，否则 goroutine 可能会被误导，认为池是打开的，并尝试在通道上执行无效操作。

现在您已经看到了池代码并了解了它的工作原理，让我们回顾一下 main.go 源代码文件中的测试程序。

##### 代码列表 7.21\. `pool`/`main`/main.go

```
01 // This sample program demonstrates how to use the pool package
02 // to share a simulated set of database connections.
03 package main
04
05 import (
06     "log"
07     "io"
08     "math/rand"
09     "sync"
10     "sync/atomic"
11     "time"
12
13     "github.com/goinaction/code/chapter7/patterns/pool"
14 )
15
16 const (
17     maxGoroutines   = 25 // the number of routines to use.
18     pooledResources = 2  // number of resources in the pool
19 )
20
21 // dbConnection simulates a resource to share.
22 type dbConnection struct {
23     ID int32
24 }
25
26 // Close implements the io.Closer interface so dbConnection
27 // can be managed by the pool. Close performs any resource
28 // release management.
29 func (dbConn *dbConnection) Close() error {
30     log.Println("Close: Connection", dbConn.ID)
31     return nil
32 }
33
34 // idCounter provides support for giving each connection a unique id.
35 var idCounter int32
36
37 // createConnection is a factory method that will be called by
38 // the pool when a new connection is needed.
39 func createConnection() (io.Closer, error) {

40     id := atomic.AddInt32(&idCounter, 1)
41     log.Println("Create: New Connection", id)
42
43     return &dbConnection{id}, nil
44 }
45
46 // main is the entry point for all Go programs.
47 func main() {
48     var wg sync.WaitGroup
49     wg.Add(maxGoroutines)
50
51     // Create the pool to manage our connections.
52     p, err := pool.New(createConnection, pooledResources)
53     if err != nil {
54         log.Println(err)
55     }
56
57     // Perform queries using connections from the pool.
58     for query := 0; query < maxGoroutines; query++ {
59         // Each goroutine needs its own copy of the query
60         // value else they will all be sharing the same query
61         // variable.
62         go func(q int) {
63             performQueries(q, p)
64             wg.Done()
65         }(query)
66     }
67
68     // Wait for the goroutines to finish.
69     wg.Wait()
70
71     // Close the pool.
72     log.Println("Shutdown Program.")
73     p.Close()
74 }
75
76 // performQueries tests the resource pool of connections.
77 func performQueries(query int, p *pool.Pool) {
78     // Acquire a connection from the pool.
79     conn, err := p.Acquire()
80     if err != nil {
81         log.Println(err)
82         return
83     }
84
85     // Release the connection back to the pool.
86     defer p.Release(conn)
87
88     // Wait to simulate a query response.
89     time.Sleep(time.Duration(rand.Intn(1000)) * time.Millisecond)
90     log.Printf("QID[%d] CID[%d]\n", query, conn.(*dbConnection).ID)
91 }
```

在 main.go 中显示的代码，如 代码列表 7.21，使用 `pool` 包来管理模拟的数据库连接池。代码首先声明了两个常量 `maxGoroutines` 和 `pooledResources`，以设置程序将要使用的 goroutine 和资源数量。接着声明了资源和实现了 `io.Closer` 接口。

##### 代码列表 7.22\. `pool`/`main`/main.go: 行 21–32

```
21 // dbConnection simulates a resource to share.
22 type dbConnection struct {
23     ID int32
24 }
25
26 // Close implements the io.Closer interface so dbConnection
27 // can be managed by the pool. Close performs any resource
28 // release management.
29 func (dbConn *dbConnection) Close() error {
30     log.Println("Close: Connection", dbConn.ID)
31     return nil
32 }
```

代码列表 7.22 展示了 `dbConnection` 结构体的声明及其实现 `io.Closer` 接口。`dbConnection` 类型模拟了一个管理数据库连接的结构体，目前有一个字段 `ID`，包含每个连接的唯一 ID。`Close` 方法只是报告连接正在关闭并显示其 ID。

接下来，我们有一个工厂函数，用于创建 `dbConnection` 的值。

##### 代码列表 7.23\. `pool`/`main`/main.go: 行 34–44

```
34 // idCounter provides support for giving each connection a unique id.
35 var idCounter int32
36
37 // createConnection is a factory method that will be called by
38 // the pool when a new connection is needed.
39 func createConnection() (io.Closer, error) {
40     id := atomic.AddInt32(&idCounter, 1)
41     log.Println("Create: New Connection", id)
42
43     return &dbConnection{id}, nil
44 }
```

代码列表 7.23 展示了 `createConnection` 函数的实现。该函数为连接生成一个新的唯一 ID，显示正在创建连接，并返回一个具有此唯一 ID 的 `dbConnection` 类型的指针值。唯一 ID 的生成是通过 `atomic.AddInt32` 函数完成的。它用于安全地递增包级别变量 `idCounter` 的值。现在我们有了资源和工厂函数，我们可以使用 `pool` 包来使用它。

接下来，让我们看看 `main` 函数内部的代码。

##### 代码列表 7.24\. `pool`/`main`/main.go: 行 48–55

```
48     var wg sync.WaitGroup
49     wg.Add(maxGoroutines)
50
51     // Create the pool to manage our connections.
52     p, err := pool.New(createConnection, pooledResources)
53     if err != nil {
54         log.Println(err)
55     }
```

`main` 函数开始于第 48 行声明一个 `WaitGroup` 并将 `WaitGroup` 的值设置为将要创建的 goroutine 数量。使用 `pool` 包的 `New` 函数创建一个新的 `Pool`。传入工厂函数和管理资源数量。这返回一个指向 `Pool` 值的指针，并检查任何可能的错误。现在我们有了 `Pool`，我们可以创建可以共享池管理的资源的 goroutine。

##### 列表 7.25\. `pool`/`main`/main.go: 行 57–66

```
57     // Perform queries using connections from the pool.
58     for query := 0; query < maxGoroutines; query++ {
59         // Each goroutine needs its own copy of the query
60         // value else they will all be sharing the same query
61         // variable.
62         go func(q int) {
63             performQueries(q, p)
64             wg.Done()
65         }(query)
66     }
```

在 列表 7.25 中使用 `for` 循环创建将使用池的 goroutine。每个 goroutine 调用一次 `performQueries` 函数然后退出。`performQueries` 函数提供了一个唯一的 ID 用于日志记录，以及指向 `Pool` 的指针。一旦所有 goroutine 都创建完成，`main` 函数等待 goroutine 完成。

##### 列表 7.26\. `pool`/`main`/main.go: 行 68–73

```
68     // Wait for the goroutines to finish.
69     wg.Wait()
70
71     // Close the pool.
72     log.Println("Shutdown Program.")
73     p.Close()
```

在 列表 7.26 中，`main` 函数等待 `WaitGroup`。一旦所有 goroutine 报告已完成，关闭 `Pool` 并终止程序。接下来，让我们看看 `performQueries` 函数，它使用了池的 `Acquire` 和 `Release` 方法。

##### 列表 7.27\. `pool`/`main`/main.go: 行 76–91

```
76 // performQueries tests the resource pool of connections.
77 func performQueries(query int, p *pool.Pool) {
78     // Acquire a connection from the pool.
79     conn, err := p.Acquire()
80     if err != nil {
81         log.Println(err)
82         return
83     }
84
85     // Release the connection back to the pool.
86     defer p.Release(conn)
87
88     // Wait to simulate a query response.
89     time.Sleep(time.Duration(rand.Intn(1000)) * time.Millisecond)
90     log.Printf("QID[%d] CID[%d]\n", query, conn.(*dbConnection).ID)
91 }
```

列表 7.27 中 `performQueries` 函数的实现展示了池的 `Acquire` 和 `Release` 方法的使用。函数开始时通过调用 `Acquire` 方法从池中检索一个 `dbConnection`。检查返回的 `error` 接口值，然后在第 86 行使用 `defer` 将 `dbConnection` 释放回池中。在第 89 和 90 行发生随机睡眠，以使用 `dbConnection` 模拟工作时间。

### 7.3\. 工作

`work` 包的目的是展示如何使用无缓冲通道创建一个 goroutine 池，该池将执行并控制并发完成的工作量。这种方法比使用任意静态大小的缓冲通道作为工作队列并将大量 goroutine 扔给它要好。无缓冲通道提供了一种保证，即两个 goroutine 之间已经交换了数据。使用无缓冲通道的方法允许用户知道池何时正在执行工作，当通道忙于处理更多工作而无法接受时，它会推回。永远不会丢失任何工作，也不会在工作队列中卡住，因为无法保证它将被处理。

让我们看看 `work` 包中的 work.go 代码文件。

##### 列表 7.28\. `work`/work.go

```
01 // Example provided with help from Jason Waldrip.
02 // Package work manages a pool of goroutines to perform work.
03 package work
04
05 import "sync"
06
07 // Worker must be implemented by types that want to use
08 // the work pool.
09 type Worker interface {
10     Task()

11 }
12
13 // Pool provides a pool of goroutines that can execute any Worker
14 // tasks that are submitted.
15 type Pool struct {
16     work chan Worker
17     wg   sync.WaitGroup
18 }
19
20 // New creates a new work pool.
21 func New(maxGoroutines int) *Pool {
22     p := Pool{
23         tasks: make(chan Worker),
24     }
25
26     p.wg.Add(maxGoroutines)
27     for i := 0; i < maxGoroutines; i++ {
28         go func() {
29             for w := range p.work {
30                 w.Task()
31             }
32             p.wg.Done()
33         }()
34     }
35
36     return &p
37 }
38
39 // Run submits work to the pool.
40 func (p *Pool) Run(w Worker) {
41     p.work <- w
42 }
43
44 // Shutdown waits for all the goroutines to shutdown.
45 func (p *Pool) Shutdown() {
46     close(p.tasks)
47     p.wg.Wait()
48 }
```

列表 7.28 中的 `work` 包开始于一个名为 `Worker` 的接口和一个名为 `Pool` 的结构的声明。

##### 列表 7.29\. `work`/work.go: 行 07–18

```
07 // Worker must be implemented by types that want to use
08 // the work pool.
09 type Worker interface {
10     Task()
11 }
12
13 // Pool provides a pool of goroutines that can execute any Worker
14 // tasks that are submitted.
15 type Pool struct {
16     work chan Worker
17     wg   sync.WaitGroup
18 }
```

在 列表 7.29 的第 09 行上的 `Worker` 接口声明了一个名为 `Task` 的单个方法。在第 15 行声明了一个名为 `Pool` 的结构体，这是实现 goroutines 池的类型，并将具有处理工作的方法。该类型声明了两个字段，一个名为 `work`，它是一个 `Worker` 接口类型的通道，另一个名为 `wg` 的 `sync.WaitGroup`。

接下来让我们看看 `work` 包的工厂函数。

##### 列表 7.30\. `work`/work.go: 行 20–37

```
20 // New creates a new work pool.
21 func New(maxGoroutines int) *Pool {
22     p := Pool{
23         work: make(chan Worker),
24     }
25
26     p.wg.Add(maxGoroutines)
27     for i := 0; i < maxGoroutines; i++ {
28         go func() {
29             for w := range p.work {
30                 w.Task()
31             }
32             p.wg.Done()
33         }()
34     }
35
36     return &p
37 }
```

列表 7.30 展示了用于创建配置了固定数量 goroutines 的工作池的 `New` 函数。goroutines 的数量作为参数传递给 `New` 函数。在第 22 行创建了一个类型为 `Pool` 的值，并将 `work` 字段初始化为一个无缓冲通道。

然后，在第 26 行初始化了 `WaitGroup`，在第 27 行至 34 行创建了相同数量的 goroutines。这些 goroutines 只接收类型为 `Worker` 的接口值，并调用这些值上的 `Task` 方法。

##### 列表 7.31\. `work`/work.go: 行 28–33

```
28         go func() {
29             for w := range w.work {
30                 w.Task()
31             }
32             p.wg.Done()
33         }()
```

`for range` 循环会阻塞，直到在 `work` 通道上有 `Worker` 接口值可以接收。当接收到值时，会调用 `Task` 方法。一旦 `work` 通道关闭，`for range` 循环结束，并调用 `WaitGroup` 上的 `Done` 方法。然后 goroutine 终止。

现在我们能够创建一个可以等待并执行工作的 goroutines 池，让我们看看如何将工作提交到池中。

##### 列表 7.32\. `work`/work.go: 行 39–42

```
39 // Run submits work to the pool.
40 func (p *Pool) Run(w Worker) {
41     w.work <- w
42 }
```

列表 7.32 展示了 `Run` 方法。此方法用于将工作提交到池中。它接受一个类型为 `Worker` 的接口值，并通过 `work` 通道发送该值。由于 `work` 通道是一个无缓冲通道，调用者必须等待池中的某个 goroutine 接收它。这正是我们想要的，因为调用者需要保证在 `Run` 调用返回后，提交的工作正在被处理。

在某个时刻，工作池需要关闭。这就是 `Shutdown` 方法发挥作用的地方。

##### 列表 7.33\. `work`/work.go: 行 44–48

```
44 // Shutdown waits for all the goroutines to shutdown.
45 func (p *Pool) Shutdown() {
46     close(p.work)
47     p.wg.Wait()
48 }
```

在 列表 7.33 中的 `Shutdown` 方法做了两件事。首先，它关闭了 `work` 通道，这导致池中的所有 goroutines 关闭并调用 `WaitGroup` 上的 `Done` 方法。然后 `Shutdown` 方法调用 `WaitGroup` 上的 `Wait` 方法，这导致 `Shutdown` 方法等待所有 goroutines 报告它们已经终止。

现在您已经看到了工作包的代码，并了解了它是如何工作的，让我们回顾一下 `main.go` 源代码文件中的测试程序。

##### 列表 7.34\. `work`/`main`/main.go

```
01 // This sample program demonstrates how to use the work package
02 // to use a pool of goroutines to get work done.
03 package main
04
05 import (
06     "log"
07     "sync"
08     "time"
09
10     "github.com/goinaction/code/chapter7/patterns/work"
11 )
12

13 // names provides a set of names to display.
14 var names = []string{
15     "steve",
16     "bob",
17     "mary",
18     "therese",
19     "jason",
20 }
21
22 // namePrinter provides special support for printing names.
23 type namePrinter struct {
24     name string
25 }
26
27 // Task implements the Worker interface.
28 func (m *namePrinter) Task() {
29     log.Println(m.name)
30     time.Sleep(time.Second)
31 }
32
33 // main is the entry point for all Go programs.
34 func main() {
35     // Create a work pool with 2 goroutines.
36     p := work.New(2)
37
38     var wg sync.WaitGroup
39     wg.Add(100 * len(names))
40
41     for i := 0; i < 100; i++ {
42         // Iterate over the slice of names.
43         for _, name := range names {
44             // Create a namePrinter and provide the
45             // specific name.
46             np := namePrinter{
47                 name: name,
48             }
49
50             go func() {
51                 // Submit the task to be worked on. When RunTask
52                 // returns we know it is being handled.
53                 p.Run(&np)
54                 wg.Done()
55             }()
56         }
57     }
58
59     wg.Wait()
60
61     // Shutdown the work pool and wait for all existing work
62     // to be completed.
63     p.Shutdown()
64 }
```

列表 7.34 显示了使用`work`包执行名称显示的测试程序。代码从第 14 行开始，声明了一个名为`names`的包级变量，它被声明为一个字符串切片。该切片还初始化了五个名称。然后声明了一个名为`namePrinter`的类型。

##### 列表 7.35. `work`/`main`/main.go: 第 22–31 行

```
22 // namePrinter provides special support for printing names.
23 type namePrinter struct {
24     name string
25 }
26
27 // Task implements the Worker interface.
28 func (m *namePrinter) Task() {
29     log.Println(m.name)
30     time.Sleep(time.Second)
31 }
```

在列表 7.35 的第 23 行，声明了`namePrinter`类型，并跟随`Worker`接口的实现。工作的目的是在屏幕上显示名称。该类型包含一个字段`name`，它将包含要显示的名称。`Worker`接口的实现使用`log.Println`函数来显示名称，然后等待一秒后返回。第二次等待只是为了减慢测试程序的速度，以便你可以看到并发操作的效果。

通过实现`Worker`接口，我们可以查看`main`函数内部的代码。

##### 列表 7.36. `work`/`main`/main.go: 第 33–64 行

```
33 // main is the entry point for all Go programs.
34 func main() {
35     // Create a work pool with 2 goroutines.
36     p := work.New(2)
37
38     var wg sync.WaitGroup
39     wg.Add(100 * len(names))
40
41     for i := 0; i < 100; i++ {
42         // Iterate over the slice of names.
43         for _, name := range names {
44             // Create a namePrinter and provide the
45             // specific name.
46             np := namePrinter{
47                 name: name,
48             }
49
50             go func() {
51                 // Submit the task to be worked on. When RunTask
52                 // returns we know it is being handled.
53                 p.Run(&np)

54                 wg.Done()
55             }()
56         }
57     }
58
59     wg.Wait()
60
61     // Shutdown the work pool and wait for all existing work
62     // to be completed.
63     p.Shutdown()
64 }
```

在列表 7.36 的第 36 行，从`work`包中调用了`New`函数来创建工作池。调用中传入了数字 2，表示池中只应包含两个 goroutine。在第 38 和 39 行，声明并初始化了一个`WaitGroup`，用于每个将被创建的 goroutine。在这种情况下，将为`names`切片中的每个名称创建 100 个 goroutine。这是为了创建大量的 goroutine，它们将竞争向池提交工作。

在第 41 和 43 行声明了内部和外部`for`循环来创建所有 goroutine。在内循环的每次迭代中，创建一个类型为`namePrinter`的值，并提供一个要打印的名称。然后，在第 50 行，声明并创建了一个匿名函数，作为一个 goroutine。该 goroutine 调用工作池的`Run`方法来提交`namePrinter`值到池中。一旦工作池中的 goroutine 接收到了值，`Run`的调用就会返回。这反过来导致 goroutine 减少`WaitGroup`计数并终止。

一旦创建了所有 goroutine，`main`函数就会在`WaitGroup`上调用`Wait`。该函数将等待所有创建的 goroutine 提交它们的工作。一旦`Wait`返回，通过调用`Shutdown`方法关闭工作池。此方法不会返回，直到所有工作都完成。在我们的例子中，此时会有两个未完成的工作项。

### 7.4. 摘要

+   你可以使用通道来控制程序的生存期。

+   可以使用带有默认情况的`select`语句来尝试在通道上进行非阻塞的发送或接收。

+   缓冲通道可以用来管理一个可重用的资源池。

+   通道的协调和同步由运行时处理。

+   使用无缓冲通道创建一个 goroutine 池来执行工作。

+   任何可以使用未缓冲通道在两个 goroutine 之间交换数据的时候，你都有可以信赖的保证。
