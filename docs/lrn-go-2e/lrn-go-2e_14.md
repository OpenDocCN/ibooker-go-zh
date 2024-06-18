# 第十四章。上下文

服务器需要一种处理个别请求元数据的方式。这些元数据可以大致分为两类：一是正确处理请求所需的元数据，二是控制何时停止处理请求的元数据。例如，一个 HTTP 服务器可能希望使用跟踪 ID 标识一系列通过一组微服务的请求。它还可能希望设置一个计时器，以便在其他微服务请求时间过长时结束请求。

许多语言使用*线程本地*变量来存储这类信息，将数据关联到特定的操作系统线程执行上。这在 Go 语言中行不通，因为 goroutine 没有可以用来查找值的唯一标识。更重要的是，线程本地变量感觉像是魔术；值放在一个地方，却在其他地方出现。

Go 语言通过一种称为*context*的构造解决请求元数据问题。让我们看看如何正确使用它。

# 什么是上下文？

不是为语言添加新功能，上下文仅仅是符合`context`包中定义的`Context`接口的实例。正如你所知，Go 语言鼓励通过函数参数显式传递数据。对于上下文而言也是如此，它只是作为你的函数的另一个参数。

就像 Go 语言约定的最后一个返回值是`error`一样，Go 还约定上下文作为函数的第一个参数显式传递到你的程序中。上下文参数的通常名称是`ctx`：

```go
func logic(ctx context.Context, info string) (string, error) {
    // do some interesting stuff here
    return "", nil
}
```

除了定义`Context`接口外，`context`包还包含几个工厂函数，用于创建和包装上下文。当你没有现成的上下文，比如在命令行程序的入口点时，可以使用函数`context.Background`创建一个空的初始上下文。这将返回一个`context.Context`类型的变量。（是的，这与通常从函数调用中返回具体类型的模式不同。）

空上下文是一个起点；每次向上下文添加元数据时，都可以通过`context`包中的工厂函数*包装*现有的上下文。

###### 注意

另一个函数`context.TODO`也创建一个空的`context.Context`。它用于开发过程中的临时使用。如果你不确定上下文将来自何处或将如何使用它，请使用`context.TODO`在代码中放置一个占位符。生产代码不应包含`context.TODO`。

在编写 HTTP 服务器时，你使用稍微不同的模式来通过中间件层传递并获取上下文，最终传递给顶级的`http.Handler`。不幸的是，上下文是在`net/http`包创建之后很长时间才添加到 Go API 中的。由于兼容性承诺，无法修改`http.Handler`接口以添加`context.Context`参数。

兼容性承诺确实允许在现有类型中添加新方法，这也是 Go 团队所做的。`http.Request` 上有两个与上下文相关的方法：

+   `Context` 返回与请求关联的 `context.Context`。

+   `WithContext` 接收一个 `context.Context` 并返回一个新的 `http.Request`，其中包含旧请求的状态和提供的 `context.Context` 的组合。

这是一般的模式：

```go
func Middleware(handler http.Handler) http.Handler {
    return http.HandlerFunc(func(rw http.ResponseWriter, req *http.Request) {
        ctx := req.Context()
        // wrap the context with stuff -- you'll see how soon!
        req = req.WithContext(ctx)
        handler.ServeHTTP(rw, req)
    })
}
```

在你的中间件中，第一件事情是通过使用 `Context` 方法从请求中提取现有的上下文。（如果你想要跳过，可以看看如何将值放入上下文中的“值”部分。）在将值放入上下文后，你可以使用 `WithContext` 方法基于旧请求和现在填充的上下文创建一个新请求。最后，调用 `handler` 并传递你的新请求和现有的 `http.ResponseWriter`。

当你实现处理程序时，使用 `Context` 方法从请求中提取上下文，并将上下文作为第一个参数调用你的业务逻辑，就像之前看到的那样：

```go
func handler(rw http.ResponseWriter, req *http.Request) {
    ctx := req.Context()
    err := req.ParseForm()
    if err != nil {
        rw.WriteHeader(http.StatusInternalServerError)
        rw.Write([]byte(err.Error()))
        return
    }
    data := req.FormValue("data")
    result, err := logic(ctx, data)
    if err != nil {
        rw.WriteHeader(http.StatusInternalServerError)
        rw.Write([]byte(err.Error()))
        return
    }
    rw.Write([]byte(result))
}
```

当你的应用程序从另一个 HTTP 服务进行 HTTP 调用时，请使用 `net/http` 包中的 `NewRequestWithContext` 函数来构造一个请求，其中包含现有的上下文信息：

```go
type ServiceCaller struct {
    client *http.Client
}

func (sc ServiceCaller) callAnotherService(ctx context.Context, data string)
                                          (string, error) {
    req, err := http.NewRequestWithContext(ctx, http.MethodGet,
                "http://example.com?data="+data, nil)
    if err != nil {
        return "", err
    }
    resp, err := sc.client.Do(req)
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()
    if resp.StatusCode != http.StatusOK {
        return "", fmt.Errorf("Unexpected status code %d",
                              resp.StatusCode)
    }
    // do the rest of the stuff to process the response
    id, err := processResponse(resp.Body)
    return id, err
}
```

你可以在[第十四章的仓库](https://oreil.ly/iT-az)中的 *sample_code/context_patterns* 目录中找到这些代码示例。

现在你知道如何获取和传递上下文了，让我们开始让它们有用起来。你将从传递值开始。

# 值

默认情况下，你应该优先通过显式参数传递数据。正如之前提到的，习惯上 Go 更偏向于显式而非隐式，包括显式数据传递。如果一个函数依赖于某些数据，它应该清楚地指出它需要什么数据以及数据来自何处。

然而，在某些情况下，你不能显式地传递数据。最常见的情况是 HTTP 请求处理程序及其关联的中间件。正如你所见，所有的 HTTP 请求处理程序都有两个参数，一个用于请求，一个用于响应。如果你想要在中间件中使一个值对处理程序可用，你需要将它存储在上下文中。可能的情况包括从 JWT（JSON Web Token）中提取用户或创建一个通过多层中间件传递到处理程序和业务逻辑的每个请求的 GUID。

有一个用于将值放入上下文的工厂方法，`context.WithValue`。它接收三个值：一个上下文，一个键来查找值以及值本身。键和值参数声明为 `any` 类型。`context.WithValue` 函数返回一个上下文，但它不是传入函数的同一个上下文。相反，它是一个包含键-值对并包裹传入的父 `context.Context` 的*子*上下文。

###### 注意

你会多次看到这种包装模式。上下文被视为一个不可变实例。每当向上下文添加信息时，都是通过将现有的父上下文包装为子上下文来实现的。这使你能够使用上下文将信息传递到代码的更深层。上下文永远不用于将信息从更深的层传递到更高的层。

`context.Context`上的`Value`方法检查一个值是否在上下文或其任何父上下文中。此方法接受一个键，并返回与该键关联的值。同样，键参数和值结果都声明为`any`类型。如果未找到提供的键的值，则返回`nil`。使用逗号-ok 惯用法将返回的值断言为正确的类型：

```go
ctx := context.Background()
if myVal, ok := ctx.Value(myKey).(int); !ok {
    fmt.Println("no value")
} else {
    fmt.Println("value:", myVal)
}
```

###### 注意

如果您熟悉数据结构，您可能会意识到在上下文链中搜索存储的值是*线性*搜索。当只有少量值时，这不会对性能造成严重影响，但如果在请求期间将几十个值存储在上下文中，性能会表现不佳。也就是说，如果您的程序正在创建具有几十个值的上下文链，那么您的程序可能需要进行一些重构。

上下文中存储的值可以是任何类型，但选择正确的键很重要。就像`map`的键一样，上下文值的键必须是可比较的。不要仅仅使用像`"id"`这样的`string`。如果你使用`string`或者另一个预定义或导出的类型作为键的类型，不同的包可能会创建相同的键，导致冲突。这会引起难以调试的问题，例如一个包向上下文写入数据，掩盖了另一个包写入的数据，或者从上下文读取由另一个包写入的数据。

有两种模式用于保证键是唯一且可比较的。第一种是基于`int`创建一个新的、未导出的键类型：

```go
type userKey int
```

在声明您的未导出键类型后，然后声明该类型的未导出常量：

```go
const (
    _ userKey = iota
    key
)
```

由于类型和键的类型常量都是未导出的，因此来自包外部的代码无法将数据放入上下文以引起冲突。如果您的包需要将多个值放入上下文，请为每个值定义相同类型的不同键，使用您在“iota 用于枚举——有时”中查看过的`iota`模式。由于您只关心常量的值作为区分多个键的一种方式，这是`iota`的一个完美用例。

接下来，构建一个 API 来将值放入上下文并从上下文中读取该值。仅在包外的代码应该能够读取和写入您的上下文值时，使这些函数公开。创建具有该值的上下文的函数的名称应以`ContextWith`开头。从上下文返回值的函数的名称应以`FromContext`结尾。以下是设置和从上下文中读取用户的函数实现：

```go
func ContextWithUser(ctx context.Context, user string) context.Context {
    return context.WithValue(ctx, key, user)
}

func UserFromContext(ctx context.Context) (string, bool) {
    user, ok := ctx.Value(key).(string)
    return user, ok
}
```

另一个选项是通过使用空结构体定义未导出的键类型：

```go
type userKey struct{}
```

然后更改用于管理上下文值访问的函数：

```go
func ContextWithUser(ctx context.Context, user string) context.Context {
    return context.WithValue(ctx, userKey{}, user)
}

func UserFromContext(ctx context.Context) (string, bool) {
    user, ok := ctx.Value(userKey{}).(string)
    return user, ok
}
```

如何知道使用哪种键样式？如果您有一组用于在上下文中存储不同值的相关键，则使用`int`和`iota`技术。如果只有一个单一的键，则任何一种都可以。重要的是，您希望使上下文键不可能发生冲突。

现在您已编写了用户管理代码，让我们看看如何使用它。您将编写中间件，从 cookie 中提取用户 ID：

```go
// a real implementation would be signed to make sure
// the user didn't spoof their identity
func extractUser(req *http.Request) (string, error) {
    userCookie, err := req.Cookie("identity")
    if err != nil {
        return "", err
    }
    return userCookie.Value, nil
}

func Middleware(h http.Handler) http.Handler {
    return http.HandlerFunc(func(rw http.ResponseWriter, req *http.Request) {
        user, err := extractUser(req)
        if err != nil {
            rw.WriteHeader(http.StatusUnauthorized)
            rw.Write([]byte("unauthorized"))
            return
        }
        ctx := req.Context()
        ctx = ContextWithUser(ctx, user)
        req = req.WithContext(ctx)
        h.ServeHTTP(rw, req)
    })
}
```

在中间件中，您首先获取用户值。接下来，使用`Context`方法从请求中提取上下文，并使用`ContextWithUser`函数创建一个包含用户的新上下文。当您包装上下文时，重用`ctx`变量名是惯用的。然后，您通过使用`WithContext`方法从旧请求和新上下文创建一个新请求。最后，您使用我们提供的`http.ResponseWriter`调用处理程序链中的下一个函数。

在大多数情况下，您希望在请求处理程序中从上下文中提取值，并将其显式传递给业务逻辑。Go 函数具有显式参数，不应将上下文用作通过 API 绕过的方式：

```go
func (c Controller) DoLogic(rw http.ResponseWriter, req *http.Request) {
    ctx := req.Context()
    user, ok := identity.UserFromContext(ctx)
    if !ok {
        rw.WriteHeader(http.StatusInternalServerError)
        return
    }
    data := req.URL.Query().Get("data")
    result, err := c.Logic.BusinessLogic(ctx, user, data)
    if err != nil {
        rw.WriteHeader(http.StatusInternalServerError)
        rw.Write([]byte(err.Error()))
        return
    }
    rw.Write([]byte(result))
}
```

通过在请求上使用`Context`方法获取上下文，使用`UserFromContext`函数从上下文中提取用户，然后调用业务逻辑，您的处理程序会得到上下文。这段代码展示了关注点分离的价值；如何加载用户对`Controller`来说是未知的。一个真实的用户管理系统可以在中间件中实现，并且可以在不更改任何控制器代码的情况下进行交换。

此示例的完整代码位于[第十四章代码库](https://oreil.ly/iT-az)的*sample_code/context_user*目录中。

在某些情况下，最好将值保留在上下文中。前面提到的跟踪 GUID 就是一个例子。此信息用于管理您的应用程序；它不是业务状态的一部分。通过将跟踪 GUID 明确地通过您的代码传递，可以添加额外的参数，并防止与不知道您元信息的第三方库集成。通过在上下文中留下跟踪 GUID，它将在不需要了解跟踪的业务逻辑中隐式传递，并在您的程序写日志消息或连接到另一个服务器时可用。

这是一个简单的上下文感知的 GUID 实现，用于跟踪服务之间的流动，并在日志中包含 GUID：

```go
package tracker

import (
    "context"
    "fmt"
    "net/http"

    "github.com/google/uuid"
)

type guidKey int

const key guidKey = 1

func contextWithGUID(ctx context.Context, guid string) context.Context {
    return context.WithValue(ctx, key, guid)
}

func guidFromContext(ctx context.Context) (string, bool) {
    g, ok := ctx.Value(key).(string)
    return g, ok
}

func Middleware(h http.Handler) http.Handler {
    return http.HandlerFunc(func(rw http.ResponseWriter, req *http.Request) {
        ctx := req.Context()
        if guid := req.Header.Get("X-GUID"); guid != "" {
            ctx = contextWithGUID(ctx, guid)
        } else {
            ctx = contextWithGUID(ctx, uuid.New().String())
        }
        req = req.WithContext(ctx)
        h.ServeHTTP(rw, req)
    })
}

type Logger struct{}

func (Logger) Log(ctx context.Context, message string) {
    if guid, ok := guidFromContext(ctx); ok {
        message = fmt.Sprintf("GUID: %s - %s", guid, message)
    }
    // do logging
    fmt.Println(message)
}

func Request(req *http.Request) *http.Request {
    ctx := req.Context()
    if guid, ok := guidFromContext(ctx); ok {
        req.Header.Add("X-GUID", guid)
    }
    return req
}
```

`Middleware` 函数从传入的请求中提取或生成一个新的 GUID。在这两种情况下，它都将 GUID 放入上下文中，创建一个带有更新后的上下文的新请求，并继续调用链。

接下来您可以看到如何使用这个 GUID。`Logger` 结构提供了一个通用的日志记录方法，接受上下文和字符串作为参数。如果上下文中有 GUID，则将其附加到日志消息的开头并输出。当此服务调用另一个服务时，会使用 `Request` 函数。它接受一个 `*http.Request`，如果上下文中存在 GUID，则添加一个带有 GUID 的头部，并返回更新后的 `*http.Request`。

一旦您获得了这个包，您可以使用我在 “隐式接口使依赖注入更容易” 中讨论的依赖注入技术来创建完全不知道任何跟踪信息的业务逻辑。首先，声明一个接口来表示您的日志记录器，一个函数类型来表示请求装饰器，以及一个依赖于它们的业务逻辑结构：

```go
type Logger interface {
    Log(context.Context, string)
}

type RequestDecorator func(*http.Request) *http.Request

type LogicImpl struct {
    RequestDecorator RequestDecorator
    Logger           Logger
    Remote           string
}
```

接下来，实现您的业务逻辑：

```go
func (l LogicImpl) Process(ctx context.Context, data string) (string, error) {
    l.Logger.Log(ctx, "starting Process with "+data)
    req, err := http.NewRequestWithContext(ctx,
        http.MethodGet, l.Remote+"/second?query="+data, nil)
    if err != nil {
        l.Logger.Log(ctx, "error building remote request:"+err.Error())
        return "", err
    }
    req = l.RequestDecorator(req)
    resp, err := http.DefaultClient.Do(req)
    // process the response...
}
```

GUID 通过日志记录器和请求装饰器传递，而业务逻辑不知道它，从而将程序逻辑所需的数据与程序管理所需的数据分离。唯一知道关联的地方是在 `main` 中将您的依赖项连接起来的代码。

```go
controller := Controller{
    Logic: LogicImpl{
        RequestDecorator: tracker.Request,
        Logger:           tracker.Logger{},
        Remote:           "http://localhost:4000",
    },
}
```

您可以在 [第十四章存储库](https://oreil.ly/iT-az) 的 *sample_code/context_guid* 目录中找到 GUID 追踪器的完整代码。

###### 小贴士

使用上下文通过标准 API 传递值。在处理业务逻辑时，将上下文中的值复制到显式参数中。系统维护信息可以直接从上下文中访问。

# 取消

虽然上下文值对于传递元数据并解决 Go 的 HTTP API 限制非常有用，但上下文还有第二个用途。上下文还允许您控制应用程序的响应性并协调并发的 goroutine。让我们看看如何实现。

我在 “使用上下文终止 Goroutines” 中简要讨论了这个问题。想象一下，您有一个请求启动了几个 goroutine，每个 goroutine 调用不同的 HTTP 服务。如果一个服务返回错误，导致您无法返回有效结果，则继续处理其他 goroutine 没有意义。在 Go 中，这被称为 *取消*，上下文提供了其实现的机制。

要创建一个可取消的上下文，使用`context.WithCancel`函数。它以`context.Context`作为参数，并返回一个`context.Context`和一个`context.CancelFunc`。就像`context.WithValue`一样，返回的`context.Context`是传入函数的上下文的子上下文。`context.CancelFunc`是一个不带参数的函数，用于*取消*上下文，告诉所有监听潜在取消事件的代码停止处理。

每当你创建一个带有关联取消函数的上下文时，无论处理是否以错误结束，*必须*调用该取消函数。如果不这样做，你的程序将泄露资源（内存和 goroutine），最终会变慢或崩溃。多次调用取消函数不会引发错误；第一次之后的每次调用都不会产生任何效果。

确保调用取消函数的最简单方法是使用`defer`在取消函数返回后立即调用它：

```go
ctx, cancelFunc := context.WithCancel(context.Background())
defer cancelFunc()
```

这带来一个问题，你如何检测取消？`context.Context`接口有一个叫做`Done`的方法。它返回一个类型为`struct{}`的通道。（选择这种返回类型的原因是空结构体不使用内存。）当调用`cancel`函数时，这个通道会关闭。记住，关闭的通道在尝试读取时会立即返回其零值。

###### 警告

如果在一个不可取消的上下文中调用`Done`，它将返回`nil`。正如在“在`select`语句中关闭一个`case`”中所述，从`nil`通道读取永远不会返回。如果这不是在`select`语句的一个`case`内完成，你的程序将挂起。

让我们看看这是如何工作的。假设你有一个程序从多个 HTTP 端点收集数据。如果其中任何一个失败，你希望结束所有的处理。上下文取消允许你这样做。

###### 注意

在这个例子中，你将利用一个名为[httpbin.org](http://httpbin.org)的优秀服务。你可以向它发送 HTTP 或 HTTPS 请求，以测试你的应用程序对各种情况的响应方式。你将使用它的两个端点：一个是延迟指定秒数后返回响应的端点，另一个会返回你发送的状态码之一。

首先，创建你的可取消上下文、一个用于从 goroutine 获取数据的通道，以及一个`sync.WaitGroup`以允许等待直到所有 goroutine 完成：

```go
ctx, cancelFunc := context.WithCancel(context.Background())
defer cancelFunc()
ch := make(chan string)
var wg sync.WaitGroup
wg.Add(2)
```

接下来，启动两个 goroutine，一个调用 URL，随机返回一个错误的状态，另一个在延迟后发送一个预设的 JSON 响应。首先是随机状态的 goroutine：

```go
    go func() {
        defer wg.Done()
        for {
            // return one of these status code at random
            resp, err := makeRequest(ctx,
                "http://httpbin.org/status/200,200,200,500")
            if err != nil {
                fmt.Println("error in status goroutine:", err)
                cancelFunc()
                return
            }
            if resp.StatusCode == http.StatusInternalServerError {
                fmt.Println("bad status, exiting")
                cancelFunc()
                return
            }
            select {
            case ch <- "success from status":
            case <-ctx.Done():
            }
            time.Sleep(1 * time.Second)
        }
    }()
```

`makeRequest`函数是一个辅助函数，用于使用提供的上下文和 URL 进行 HTTP 请求。如果返回 OK 状态，你将向通道写入一条消息并休眠一秒钟。当出现错误或者返回了一个错误的状态码时，你调用`cancelFunc`并退出 goroutine。

延迟 goroutine 类似：

```go
    go func() {
        defer wg.Done()
        for {
            // return after a 1 second delay
            resp, err := makeRequest(ctx, "http://httpbin.org/delay/1")
            if err != nil {
                fmt.Println("error in delay goroutine:", err)
                cancelFunc()
                return
            }
            select {
            case ch <- "success from delay: " + resp.Header.Get("date"):
            case <-ctx.Done():
            }
        }
    }()
```

最后，您使用 for/select 模式从 goroutine 写入的通道中读取数据，并等待取消的发生：

```go
loop:
    for {
        select {
        case s := <-ch:
            fmt.Println("in main:", s)
        case <-ctx.Done():
            fmt.Println("in main: cancelled!")
            break loop
        }
    }
    wg.Wait()
```

在你的`select`语句中，有两种情况。一种从消息通道读取，另一种等待完成通道关闭。当它关闭时，你退出循环并等待 goroutine 退出。你可以在[第十四章仓库](https://oreil.ly/iT-az)的*sample_code/cancel_http*目录中找到此程序。

这是您运行代码时发生的情况（结果是随机的，所以请继续运行几次以查看不同的结果）：

```go
in main: success from status
in main: success from delay: Thu, 16 Feb 2023 03:53:57 GMT
in main: success from status
in main: success from delay: Thu, 16 Feb 2023 03:53:58 GMT
bad status, exiting
in main: cancelled!
error in delay goroutine: Get "http://httpbin.org/delay/1": context canceled
```

有一些有趣的事情需要注意。首先，您多次调用`cancelFunc`。正如前面提到的，这是完全可以的，不会引起问题。接下来，请注意，在触发取消后，您从延迟 goroutine 获取了一个错误。这是因为 Go 标准库中内置的 HTTP 客户端尊重取消。您使用可取消的上下文创建了请求，当取消时，请求结束。这会触发 goroutine 中的错误路径，并确保它不会泄漏。

你可能想知道导致取消的错误以及如何报告它。名为`WithCancelCause`的`WithCancel`的替代版本返回一个接受错误作为参数的取消函数。`context`包中的`Cause`函数返回传递给取消函数首次调用的错误。

###### 注意

`Cause`是`context`包中的一个函数，而不是`context.Context`上的方法，因为在 Go 1.20 中将通过取消返回错误的功能添加到了`context`包中，这是在最初引入`context`之后很久的事情。如果在`context.Context`接口上添加了一个新方法，这将破坏任何实现它的第三方代码。另一个选项是定义一个包含此方法的新接口，但现有代码已经到处传递`context.Context`，并将其转换为具有`Cause`方法的新接口将需要类型断言或类型切换。添加一个函数是最简单的方法。随着时间的推移，有几种方式可以演化你的 API。你应该选择对用户影响最小的方式。

让我们重写程序以捕获错误。首先，您更改了上下文的创建：

```go
ctx, cancelFunc := context.WithCancelCause(context.Background())
defer cancelFunc(nil)
```

接下来，您将对两个 goroutine 进行轻微修改。状态 goroutine 中`for`循环的主体现在如下所示：

```go
resp, err := makeRequest(ctx, "http://httpbin.org/status/200,200,200,500")
if err != nil {
    cancelFunc(fmt.Errorf("in status goroutine: %w", err))
    return
}
if resp.StatusCode == http.StatusInternalServerError {
    cancelFunc(errors.New("bad status"))
    return
}
ch <- "success from status"
time.Sleep(1 * time.Second)
```

您已删除`fmt.Println`语句，并将错误传递给`cancelFunc`。延迟 goroutine 中`for`循环的主体现在如下所示：

```go
resp, err := makeRequest(ctx, "http://httpbin.org/delay/1")
if err != nil {
    fmt.Println("in delay goroutine:", err)
    cancelFunc(fmt.Errorf("in delay goroutine: %w", err))
    return
}
ch <- "success from delay: " + resp.Header.Get("date")
```

`fmt.Println`仍然存在，因此您可以显示仍然生成并传递给`cancelFunc`的错误。

最后，您使用`context.Cause`在初始取消和等待 goroutine 完成后打印错误：

```go
loop:
    for {
        select {
        case s := <-ch:
            fmt.Println("in main:", s)
        case <-ctx.Done():
            fmt.Println("in main: cancelled with error", context.Cause(ctx))
            break loop
        }
    }
    wg.Wait()
    fmt.Println("context cause:", context.Cause(ctx))
```

你可以在[第十四章存储库](https://oreil.ly/iT-az)的*sample_code/cancel_error_http*目录中找到此代码。

运行新程序会生成以下输出：

```go
in main: success from status
in main: success from delay: Thu, 16 Feb 2023 04:11:49 GMT
in main: cancelled with error bad status
in delay goroutine: Get "http://httpbin.org/delay/1": context canceled
context cause: bad status
```

当最初在`switch`语句中检测到取消时，你会看到来自状态 goroutine 的错误输出，以及在等待延迟 goroutine 完成后。请注意，延迟 goroutine 使用错误调用了`cancelFunc`，但该错误并未覆盖最初的取消错误。

当你的代码达到结束处理的逻辑状态时，手动取消非常有用。有时候你想取消是因为任务花费的时间太长。在这种情况下，你可以使用定时器。

# 带有截止时间的上下文。

服务器的最重要工作之一是管理请求。初学者程序员通常认为服务器应该尽可能多地接受请求，并在可能的情况下处理它们，直到为每个客户端返回结果。

问题在于这种方法不具备可扩展性。服务器是共享资源。像所有共享资源一样，每个用户都希望从中获取尽可能多的资源，并且并不十分关心其他用户的需求。共享资源的责任是自我管理，以便为所有用户提供公平的时间。

通常，服务器可以执行四项操作来管理其负载：

+   限制同时请求。

+   限制等待运行的排队请求数量。

+   限制请求可以运行的时间量。

+   限制请求可以使用的资源（如内存或磁盘空间）。

Go 提供了处理前三个问题的工具。在学习并发性时，你了解到如何处理前两个问题（参见第十二章）。通过限制 goroutine 的数量，服务器可以管理同时负载。通过缓冲通道处理等待队列的大小。

上下文提供了一种控制请求运行时间的方法。在构建应用程序时，你应该有一个性能范围的概念：在用户体验不佳之前，请求完成的最长时间。如果你知道请求可以运行的最长时间，可以使用上下文进行强制执行。

###### 注意

虽然`GOMEMLIMIT`提供了限制 Go 程序使用的内存量的软性方法，但如果你想强制约束单个请求使用的内存或磁盘空间，你必须编写管理此类限制的代码。本书讨论此主题超出了范围。

你可以使用两个函数之一创建有时间限制的上下文。第一个是`context.WithTimeout`。它接受两个参数：一个现有的上下文和指定持续时间的`time.Duration`，在此持续时间后上下文将自动取消。它返回一个上下文，在指定持续时间后自动触发取消，并返回一个可立即调用以取消上下文的取消函数。

第二个函数是`context.WithDeadline`。此函数接收一个现有的上下文和一个指定上下文何时自动取消的`time.Time`。像`context.WithTimeout`一样，它返回一个上下文，在指定时间后自动触发取消以及一个取消函数。

###### 提示

如果你将过去的时间传递给`context.WithDeadline`，上下文已经被创建取消。

就像从`context.WithCancel`和`context.WithCancelCause`返回的取消函数一样，你必须确保`context.WithTimeout`和`context.WithDeadline`返回的取消函数至少被调用一次。

如果你想知道上下文何时会自动取消，请使用`context.Context`的`Deadline`方法。它返回一个`time.Time`，指示时间和一个`bool`，指示是否设置了超时。这与读取映射或通道时使用的 comma ok 习语类似。

当你为请求的总持续时间设置时间限制时，你可能希望将这段时间细分。如果你从你的服务调用另一个服务，你可能希望限制允许网络调用运行的时间，为其余处理或其他网络调用预留一些时间。通过使用`context.WithTimeout`或`context.WithDeadline`创建包装父上下文的子上下文，你可以控制每个单独调用所花费的时间。

你在子上下文设置的任何超时都受父上下文设置的超时的限制；如果父上下文在 2 秒钟后超时，你可以声明子上下文在 3 秒钟后超时，但当父上下文在 2 秒钟后超时时，子上下文也将超时。

你可以通过一个简单的程序看到这一点：

```go
ctx := context.Background()
parent, cancel := context.WithTimeout(ctx, 2*time.Second)
defer cancel()
child, cancel2 := context.WithTimeout(parent, 3*time.Second)
defer cancel2()
start := time.Now()
<-child.Done()
end := time.Now()
fmt.Println(end.Sub(start).Truncate(time.Second))
```

在这个示例中，你在父上下文上指定了 2 秒钟的超时，在子上下文上指定了 3 秒钟的超时。然后，你通过等待从子`context.Context`的`Done`方法返回的通道来等待子上下文完成。我将在下一节更多地讨论`Done`方法。

你可以在[第十四章存储库](https://oreil.ly/iT-az)的*sample_code/nested_timers*目录中找到此代码，或在[Go Playground](https://oreil.ly/FS8h2)上运行此代码。你将看到以下结果：

```go
2s
```

由于带有计时器的上下文可以因超时或显式调用取消函数而取消，上下文 API 提供了一种告知取消原因的方法。`Err`方法返回`nil`，如果上下文仍处于活动状态，或者如果上下文已被取消，则返回两种哨兵错误之一：`context.Canceled`或`context.DeadlineExceeded`。当显式取消时返回第一个错误，当超时触发取消时返回第二个错误。

让我们看看它们的用法。你将对你的 httpbin 程序进行一次修改。这一次，你要为控制 goroutine 运行时间的上下文添加一个超时：

```go
ctx, cancelFuncParent := context.WithTimeout(context.Background(), 3*time.Second)
defer cancelFuncParent()
ctx, cancelFunc := context.WithCancelCause(ctx)
defer cancelFunc(nil)
```

###### 注意

如果您希望返回取消原因的错误选项，则需要将由`WithTimeout`或`WithDeadline`创建的上下文包装在由`WithCancelCause`创建的上下文中。您必须推迟两个取消函数，以防止资源泄漏。如果希望在上下文超时时返回自定义哨兵错误，请改用`context.WithTimeoutCause`或`context.WithDeadlineCause`函数。

现在，如果返回 500 状态代码或者在 3 秒内未获取 500 状态代码，您的程序将退出。您对程序的唯一其他更改是在取消发生时打印`Err`返回的值：

```go
fmt.Println("in main: cancelled with cause:", context.Cause(ctx),
    "err:", ctx.Err())
```

您可以在[第十四章存储库](https://oreil.ly/iT-az)的*sample_code/timeout_error_http*目录中找到该代码。

结果是随机的，因此运行程序多次以查看不同的结果。如果运行程序并达到超时，您将获得如下输出：

```go
in main: success from status
in main: success from delay: Sun, 19 Feb 2023 04:36:44 GMT
in main: success from status
in main: success from status
in main: success from delay: Sun, 19 Feb 2023 04:36:45 GMT
in main: cancelled with cause: context deadline exceeded
    err: context deadline exceeded
in delay goroutine: Get "http://httpbin.org/delay/1":
    context deadline exceeded
context cause: context deadline exceeded
```

注意，`context.Cause`返回的错误与`Err`方法返回的错误相同：`context.DeadlineExceeded`。

如果状态错误发生在 3 秒内，您将获得如下输出：

```go
in main: success from status
in main: success from status
in main: success from delay: Sun, 19 Feb 2023 04:37:14 GMT
in main: cancelled with cause: bad status err: context canceled
in delay goroutine: Get "http://httpbin.org/delay/1": context canceled
context cause: bad status
```

现在`context.Cause`返回的错误是`bad status`，但`Err`返回`context.Canceled`错误。

# 您自己代码中的上下文取消

大多数情况下，您无需担心自己的代码超时或取消；它根本不会运行足够长时间。每当调用另一个 HTTP 服务或数据库时，应传递上下文；这些库通过上下文正确处理取消。

您应考虑处理两种情况的取消。第一种情况是当您的函数使用`select`语句读取或写入通道时。如“取消”中所示，包括检查上下文上的`Done`方法返回的通道的`case`。这允许您的函数在上下文取消时退出，即使 goroutine 未正确处理取消。

第二种情况是当您编写的代码运行时间足够长，应该被上下文取消中断时。在这种情况下，定期使用`context.Cause`检查上下文的状态。`context.Cause`函数如果上下文已被取消，则返回错误。

这是支持您代码中上下文取消的模式：

```go
func longRunningComputation(ctx context.Context, data string) (string, error) {
    for {
        // do some processing
        // insert this if statement periodically
        // to check if the context has been cancelled
        if err := context.Cause(ctx); err != nil {
            // return a partial value if it makes sense,
            // or a default one if it doesn't
            return "", err
        }
        // do some more processing and loop again
    }
}
```

这是一个示例循环，通过使用低效的 Leibniz 算法计算π的函数。使用上下文取消允许您控制其运行时间：

```go
i := 0
for {
    if err := context.Cause(ctx); err != nil {
        fmt.Println("cancelled after", i, "iterations")
        return sum.Text('g', 100), err
    }
    var diff big.Float
    diff.SetInt64(4)
    diff.Quo(&diff, &d)
    if i%2 == 0 {
        sum.Add(&sum, &diff)
    } else {
        sum.Sub(&sum, &diff)
    }
    d.Add(&d, two)
    i++
}
```

您可以查看完整的示例程序，演示了*sample_code/own_cancellation*目录中的这种模式，在[第十四章存储库](https://oreil.ly/iT-az)中找到。

# 练习

现在您已经看到如何使用上下文，请尝试实现这些练习。所有答案都可以在[第十四章存储库](https://oreil.ly/iT-az)中找到。

1.  创建一个生成中间件的函数，它创建一个带有超时的上下文。该函数应该有一个参数，即请求允许运行的毫秒数。它应返回一个`func(http.Handler) http.Handler`。

1.  编写一个程序，将在生成的随机数介于 0（含）和 100,000,000（不含）之间的范围内随机生成，直到两件事中的一件发生为止：生成数字 1234 或经过 2 秒。打印出总和、迭代次数以及结束原因（超时或达到数字）。

1.  假设你有一个简单的日志记录函数，看起来像这样：

    ```go
    func Log(ctx context.Context, level Level, message string) {
        var inLevel Level
        // TODO get a logging level out of the context and assign it to inLevel
        if level == Debug && inLevel == Debug {
            fmt.Println(message)
        }
        if level == Info && (inLevel == Debug || inLevel == Info) {
            fmt.Println(message)
        }
    }
    ```

    定义一个名为`Level`的类型，其底层类型为`string`。定义该类型的两个常量，`Debug`和`Info`，分别设置为`"debug"`和`"info"`。

    在上下文中创建函数来存储日志级别并提取它。

    创建一个中间件函数，从名为`log_level`的查询参数中获取日志级别。`log_level` 的有效值为 `debug` 和 `info`。

    最后，在`Log`中填写`TODO`，从上下文中正确提取日志级别。如果未分配日志级别或不是有效值，则不应打印任何内容。

# 总结

在本章中，您学习了如何使用上下文管理请求元数据。现在，您可以设置超时时间，执行显式取消操作，通过上下文传递值，并知道应该在何时执行这些操作。在下一章中，您将了解 Go 的内置测试框架，并学习如何使用它来查找程序中的错误并诊断性能问题。
