# 第九章：错误

错误处理是开发者从其他语言转向 Go 时面临的最大挑战之一。对于习惯于异常的人来说，Go 的方法感觉过时。但 Go 方法的背后是坚实的软件工程原则。在本章中，您将学习如何在 Go 中处理错误。您还将了解`panic`和`recover`，Go 用于处理应该停止执行的错误的系统。

# 如何处理错误：基础知识

如同在第五章中简要介绍的那样，Go 通过在函数的最后一个返回值返回`error`类型的值来处理错误。虽然这完全是按约定来的，但这一约定如此之强，以至于不应该违反。当函数正常执行时，错误参数返回`nil`。如果出现问题，则返回一个错误值。调用函数通过将其与`nil`比较来检查错误返回值，处理错误，或返回自己的错误。一个带有错误处理的简单函数看起来像这样：

```go
func calcRemainderAndMod(numerator, denominator int) (int, int, error) {
    if denominator == 0 {
        return 0, 0, errors.New("denominator is 0")
    }
    return numerator / denominator, numerator % denominator, nil
}
```

使用`errors`包中的`New`函数通过字符串创建新的错误。错误消息不应大写，也不应以标点符号或换行符结尾。在大多数情况下，当返回非`nil`错误时，应将其他返回值设置为它们的零值。在我讨论哨兵错误时，您将看到此规则的一个例外。

与具有异常的语言不同，Go 没有特殊的结构来检测是否返回了错误。当函数返回错误时，使用`if`语句来检查错误变量是否为非`nil`：

```go
func main() {
    numerator := 20
    denominator := 3
    remainder, mod, err := calcRemainderAndMod(numerator, denominator)
    if err != nil {
        fmt.Println(err)
        os.Exit(1)
    }
    fmt.Println(remainder, mod)
}
```

您可以在[第九章库](https://oreil.ly/KCSeb)的*sample_code/error_basics*目录中尝试此代码。

`error`是一个内置接口，定义了一个方法：

```go
type error interface {
    Error() string
}
```

任何实现此接口的内容都被视为错误。您从函数返回`nil`以指示未发生错误的原因是`nil`是任何接口类型的零值。

Go 使用返回的错误而不是抛出异常有两个非常好的原因。首先，异常会通过代码添加至少一条新的代码路径。这些路径有时不太清晰，特别是在函数不包括异常可能性声明的语言中。这会导致代码以令人惊讶的方式崩溃，当异常没有被正确处理时，甚至更糟糕的是，代码并不会崩溃，但其数据没有正确初始化、修改或存储。

第二个原因更微妙，但展示了 Go 的特性如何协同工作。Go 编译器要求读取所有变量。使错误返回值迫使开发者要么检查和处理错误条件，要么通过使用下划线(`_`)明确表明他们正在忽略错误。

异常处理可能会生成更短的代码，但更少的行数并不一定使代码更易于理解或维护。正如您所见，Go 的惯用法更青睐清晰的代码，即使代码行数更多。

另一个需要注意的是 Go 中代码的流程。错误处理位于`if`语句的缩进内，而业务逻辑则不是。这为哪些代码处于“黄金路径”以及哪些代码是异常条件提供了快速的视觉线索。

###### 注意

第二种情况是重用的`err`变量。Go 编译器要求每个变量至少被读取一次。它不要求对变量的每次写入都进行读取。如果多次使用`err`变量，则只需读取一次即可使编译器满意。在“staticcheck”中，您将看到一种检测此问题的方法。

# 使用字符串表示简单错误

Go 标准库提供了两种从字符串创建错误的方式。第一种是`errors.New`函数。它接受一个`string`并返回一个`error`。当您调用返回的错误实例的`Error`方法时，此字符串将被返回。如果将错误传递给`fmt.Println`，它会自动调用`Error`方法：

```go
func doubleEven(i int) (int, error) {
    if i % 2 != 0 {
        return 0, errors.New("only even numbers are processed")
    }
    return i * 2, nil
}

func main() {
    result, err := doubleEven(1)
    if err != nil {
        fmt.Println(err) // prints "only even numbers are processed"
    }
    fmt.Println(result)
}
```

第二种方法是使用`fmt.Errorf`函数。此函数允许您通过使用`fmt.Printf`占位符在错误消息中包含运行时信息。与`errors.New`类似，调用返回的错误实例的`Error`方法时返回此字符串：

```go
func doubleEven(i int) (int, error) {
    if i % 2 != 0 {
        return 0, fmt.Errorf("%d isn't an even number", i)
    }
    return i * 2, nil
}
```

您可以在[第九章代码库](https://oreil.ly/KCSeb)的*sample_code/string_error*目录中找到此代码。

# 哨兵错误

有些错误意味着由于当前状态存在问题而无法继续处理。在他的博客文章[“不仅仅检查错误，优雅地处理它们”](https://oreil.ly/TiJnS)中，Dave Cheney，一个多年活跃于 Go 社区的开发者，创造了术语*哨兵错误*来描述这些错误：

> 名称源自计算机编程实践中使用特定值表示无法进一步处理。在 Go 中也是如此，我们使用特定值来表示错误。

哨兵错误是少数在包级别声明的变量之一。按照惯例，它们的名称以`Err`开头（`io.EOF`是显著的例外）。它们应被视为只读；虽然 Go 编译器无法强制执行这一点，但修改它们的值是编程错误。

哨兵错误通常用于指示无法启动或继续处理。例如，标准库包括一个用于处理 ZIP 文件的包，`archive/zip`。此包定义了几个哨兵错误，包括`ErrFormat`，当传入的数据不表示 ZIP 文件时返回。您可以在[The Go Playground](https://oreil.ly/DaW-s)或第九章代码库的*sample_code/sentinel_error*目录中尝试此代码：

```go
func main() {
    data := []byte("This is not a zip file")
    notAZipFile := bytes.NewReader(data)
    _, err := zip.NewReader(notAZipFile, int64(len(data)))
    if err == zip.ErrFormat {
        fmt.Println("Told you so")
    }
}
```

标准库中另一个哨兵错误的例子是`rsa.ErrMessageTooLong`，位于`crypto/rsa`包中。它指示由于消息过长而无法使用提供的公钥进行加密。哨兵错误`context.Canceled`在第十四章中有详细介绍。

在定义哨兵错误之前，请确保您确实需要一个。一旦定义了哨兵错误，它就成为您公共 API 的一部分，并且您已经承诺它在所有未来向后兼容的发布中都可用。更好的做法是重用标准库中的现有错误之一，或者定义一个错误类型，其中包含有关导致返回错误的条件的信息（您将在下一节中看到如何实现）。但是，如果您有一个错误条件，指示应用程序中已达到特定状态，进一步处理不可能，并且不需要使用其他信息来解释错误状态，则哨兵错误是正确的选择。

如何测试哨兵错误？如前面的代码示例所示，使用`==`来测试调用函数时是否返回了文档明确说明返回哨兵错误的情况。在“Is and As”中，我将讨论如何在其他情况下检查哨兵错误。

到目前为止，您看到的所有错误都是字符串。但是 Go 错误可以包含更多信息。让我们看看如何做到这一点。

# 错误是值

由于`error`是一个接口，您可以定义自己的错误类型，其中包含用于日志记录或错误处理的附加信息。例如，您可能希望在错误中包含状态码以指示应向用户报告的错误类型。这样可以避免字符串比较（其文本可能会更改）来确定错误原因。让我们看看这是如何工作的。首先，定义自己的枚举以表示状态码：

```go
type Status int

const (
    InvalidLogin Status = iota + 1
    NotFound
)
```

接下来，定义一个`StatusErr`来保存这个值：

```go
type StatusErr struct {
    Status    Status
    Message   string
}

func (se StatusErr) Error() string {
    return se.Message
}
```

现在，您可以使用`StatusErr`来提供有关出现问题的更多详细信息：

```go
func LoginAndGetData(uid, pwd, file string) ([]byte, error) {
    token, err := login(uid, pwd)
    if err != nil {
        return nil, StatusErr{
            Status:    InvalidLogin,
            Message: fmt.Sprintf("invalid credentials for user %s", uid),
        }
    }
    data, err := getData(token, file)
    if err != nil {
        return nil, StatusErr{
            Status:    NotFound,
            Message: fmt.Sprintf("file %s not found", file),
        }
    }
    return data, nil
}
```

您可以在[第九章存储库](https://oreil.ly/KCSeb)的*sample_code/custom_error*目录中找到此代码。

即使定义了自己的自定义错误类型，也要始终将`error`作为错误结果的返回类型。这允许您从函数中返回不同类型的错误，并允许调用者选择不依赖于特定的错误类型。

如果您使用自己的错误类型，请确保不要返回未初始化的实例。看看如果这样做会发生什么。在[The Go Playground](https://oreil.ly/MPaHx)或[第九章存储库](https://oreil.ly/KCSeb)的*sample_code/return_custom_error*目录中尝试以下代码：

```go
func GenerateErrorBroken(flag bool) error {
    var genErr StatusErr
    if flag {
        genErr = StatusErr{
            Status: NotFound,
        }
    }
    return genErr
}

func main() {
    err := GenerateErrorBroken(true)
    fmt.Println("GenerateErrorBroken(true) returns non-nil error:", err != nil)
    err = GenerateErrorBroken(false)
    fmt.Println("GenerateErrorBroken(false) returns non-nil error:", err != nil)
}
```

运行此程序将产生以下输出：

```go
true
true
```

这不是指针类型与值类型的问题；如果你声明`genErr`的类型是`*StatusErr`，你会看到相同的输出。`err`非空的原因是`error`是一个接口。正如我在“接口和 nil”中讨论的那样，要使接口被认为是`nil`，必须同时满足底层类型和底层值都是`nil`。不管`genErr`是不是指针，接口的底层类型部分都不是`nil`。

你可以用两种方法修复这个问题。最常见的方法是在函数成功完成时显式地返回`nil`作为错误值：

```go
func GenerateErrorOKReturnNil(flag bool) error {
    if flag {
        return StatusErr{
            Status: NotFound,
        }
    }
    return nil
}
```

这样做的好处是不需要你阅读代码来确保`return`语句上的错误变量被正确定义。

另一种方法是确保任何持有`error`的局部变量都是`error`类型：

```go
func GenerateErrorUseErrorVar(flag bool) error {
    var genErr error
    if flag {
        genErr = StatusErr{
            Status: NotFound,
        }
    }
    return genErr
}
```

###### 警告

当使用自定义错误时，永远不要定义一个变量为你自定义错误的类型。要么在没有错误发生时显式地返回`nil`，要么定义变量为`error`类型。

如在“谨慎使用类型断言和类型切换”中所述，不要使用类型断言或类型切换来访问自定义错误的字段和方法。相反，请使用`errors.As`，这在“Is 和 As”中有讨论。

# 包装错误

当错误通过你的代码传递回来时，你通常希望为其添加信息。这可以是接收到错误的函数的名称或它试图执行的操作。当你在保留错误的同时添加信息时，称为*包装*错误。当你有一系列被包装的错误时，称为*错误树*。

Go 标准库中的一个函数用于包装错误，你已经见过它了。`fmt.Errorf`函数有一个特殊的占位符`%w`。使用它可以创建一个错误，其格式化字符串包含另一个错误的格式化字符串，并且包含原始错误。惯例是在错误格式字符串的末尾写上`: %w`，并且将要被包装的错误作为`fmt.Errorf`的最后一个参数传递。

标准库还提供了一个解包错误的函数，即`errors`包中的`Unwrap`函数。你将一个错误传递给它，如果存在包装错误，它将返回被包装的错误。如果没有，它将返回`nil`。这里有一个快速示例程序，演示了使用`fmt.Errorf`包装和使用`errors.Unwrap`解包的过程。你可以在[The Go Playground](https://oreil.ly/HxdHz)上运行它，或者在[第九章代码库](https://oreil.ly/KCSeb)的*sample_code/wrap_error*目录中运行它：

```go
func fileChecker(name string) error {
    f, err := os.Open(name)
    if err != nil {
        return fmt.Errorf("in fileChecker: %w", err)
    }
    f.Close()
    return nil
}

func main() {
    err := fileChecker("not_here.txt")
    if err != nil {
        fmt.Println(err)
        if wrappedErr := errors.Unwrap(err); wrappedErr != nil {
            fmt.Println(wrappedErr)
        }
    }
}
```

当你运行这个程序时，你会看到以下输出：

```go
in fileChecker: open not_here.txt: no such file or directory
open not_here.txt: no such file or directory
```

###### 注意

通常不直接调用`errors.Unwrap`。相反，你可以使用`errors.Is`和`errors.As`来查找特定的包装错误。我将在下一节讨论这两个函数。

如果你想用你自定义的错误类型包装一个错误，你的错误类型需要实现 `Unwrap` 方法。该方法不带参数并返回一个 `error`。下面是对你之前定义的错误的更新，演示了如何实现这一点。你可以在 *sample_code/custom_wrapped_error* 目录中的 [第九章存储库](https://oreil.ly/KCSeb) 找到它：

```go
type StatusErr struct {
    Status  Status
    Message string
    Err     error
}

func (se StatusErr) Error() string {
    return se.Message
}

func (se StatusErr) Unwrap() error {
    return se.Err
}
```

现在你可以使用 `StatusErr` 包装底层错误：

```go
func LoginAndGetData(uid, pwd, file string) ([]byte, error) {
    token, err := login(uid,pwd)
    if err != nil {
        return nil, StatusErr {
            Status: InvalidLogin,
            Message: fmt.Sprintf("invalid credentials for user %s",uid),
            Err: err,
        }
    }
    data, err := getData(token, file)
    if err != nil {
        return nil, StatusErr {
            Status: NotFound,
            Message: fmt.Sprintf("file %s not found",file),
            Err: err,
        }
    }
    return data, nil
}
```

并非所有的错误都需要包装。一个库可能会返回一个错误，意味着处理无法继续，但错误消息包含了在程序其他部分不需要的实现细节。在这种情况下，创建一个全新的错误并返回是完全可以接受的。理解情况并确定需要返回什么。

###### 提示

如果你想创建一个包含另一个错误消息的新错误，但又不想包装它，请使用 `fmt.Errorf` 创建一个错误，但是使用 `%v` 动词而不是 `%w`：

```go
err := internalFunction()
if err != nil {
    return fmt.Errorf("internal failure: %v", err)
}
```

# 包装多个错误

有时一个函数会生成多个应返回的错误。例如，如果你编写一个函数来验证结构体中的字段，最好为每个无效字段返回一个错误。由于标准函数签名返回 `error` 而不是 `[]error`，你需要将多个错误合并为单个错误。这就是 `errors.Join` 函数的用途：

```go
type Person struct {
    FirstName string
    LastName  string
    Age       int
}

func ValidatePerson(p Person) error {
    var errs []error
    if len(p.FirstName) == 0 {
        errs = append(errs, errors.New("field FirstName cannot be empty"))
    }
    if len(p.LastName) == 0 {
        errs = append(errs, errors.New("field LastName cannot be empty"))
    }
    if p.Age < 0 {
        errs = append(errs, errors.New("field Age cannot be negative"))
    }
    if len(errs) > 0 {
        return errors.Join(errs...)
    }
    return nil
}
```

你可以在 *sample_code/join_error* 目录中的 [第九章存储库](https://oreil.ly/KCSeb) 找到这段代码。

合并多个错误的另一种方法是将多个 `%w` 动词传递给 `fmt.Errorf`：

```go
err1 := errors.New("first error")
err2 := errors.New("second error")
err3 := errors.New("third error")
err := fmt.Errorf("first: %w, second: %w, third: %w", err1, err2, err3)
```

你可以实现自己的支持多个包装错误的 `error` 类型。为此，实现 `Unwrap` 方法但让它返回 `[]error` 而不是 `error`：

```go
type MyError struct {
    Code   int
    Errors []error
}

type (m MyError) Error() string {
    return errors.Join(m.Errors...).Error()
}

func (m MyError) Unwrap() []error {
    return m.Errors
}
```

Go 不支持方法重载，因此你不能创建一个提供 `Unwrap` 的单一类型。还要注意，如果将 `errors.Unwrap` 函数传递给实现 `[]error` 变体的错误，它将返回 `nil`。这也是你不应直接调用 `errors.Unwrap` 函数的另一个原因。

如果你需要处理可能包装零个、一个或多个错误的错误，请使用此代码作为基础。你可以在 *sample_code/custom_wrapped_multi_error* 目录中的 [第九章存储库](https://oreil.ly/KCSeb) 找到它：

```go
var err error
err = funcThatReturnsAnError()
switch err := err.(type) {
case interface {Unwrap() error}:
    // handle single error
    innerErr := err.Unwrap()
    // process innerErr
case interface {Unwrap() []error}:
    //handle multiple wrapped errors
    innerErrs := err.Unwrap()
    for _, innerErr := range innerErrs {
        // process each innerErr
    }
default:
    // handle no wrapped error
}
```

由于标准库没有定义接口来表示具有 `Unwrap` 变体的错误，此代码使用匿名接口在类型切换中匹配方法并访问包装的错误。在编写自己的代码之前，请查看是否可以使用 `errors.Is` 和 `errors.As` 检查错误树。让我们看看它们是如何工作的。

# Is 和 As

包装错误是获取有关错误的附加信息的有用方式，但它也引入了问题。如果包装了一个 sentinel 错误，则无法使用 `==` 进行检查，也无法使用类型断言或类型切换来匹配包装的自定义错误。Go 通过 `errors` 包中的两个函数 `Is` 和 `As` 解决了这个问题。

要检查返回的错误或其包装的任何错误是否与特定的 sentinel 错误实例匹配，请使用 `errors.Is`。它接受两个参数：要检查的错误和要比较的实例。如果错误树中的任何错误与提供的 sentinel 错误匹配，则 `errors.Is` 函数返回 `true`。您将编写一个简短的程序来演示 `errors.Is` 的工作原理。您可以在[Go Playground](https://oreil.ly/5_6rI)或[第九章存储库](https://oreil.ly/KCSeb)的 *sample_code/is_error* 目录中运行它：

```go
func fileChecker(name string) error {
    f, err := os.Open(name)
    if err != nil {
        return fmt.Errorf("in fileChecker: %w", err)
    }
    f.Close()
    return nil
}

func main() {
    err := fileChecker("not_here.txt")
    if err != nil {
        if errors.Is(err, os.ErrNotExist) {
            fmt.Println("That file doesn't exist")
        }
    }
}
```

运行此程序会产生以下输出：

```go
That file doesn't exist
```

默认情况下，`errors.Is` 使用 `==` 比较每个包装错误与指定错误。如果对您定义的错误类型（例如，如果您的错误是非可比较类型），这种比较方式不起作用，请在您的错误上实现 `Is` 方法：

```go
type MyErr struct {
    Codes []int
}

func (me MyErr) Error() string {
    return fmt.Sprintf("codes: %v", me.Codes)
}

func (me MyErr) Is(target error) bool {
    if me2, ok := target.(MyErr); ok {
        return slices.Equal(me.Codes, me2.Codes)
    }
    return false
}
```

（`slices.Equal` 函数在“Slices”中已经提到。）

定义自己的 `Is` 方法的另一个用途是允许与不完全相同实例的错误进行比较。您可能希望对错误进行模式匹配，指定一个过滤器实例来匹配具有部分相同字段的错误。定义一个新的错误类型，`ResourceErr`：

```go
type ResourceErr struct {
    Resource     string
    Code         int
}

func (re ResourceErr) Error() string {
    return fmt.Sprintf("%s: %d", re.Resource, re.Code)
}
```

如果您希望两个 `ResourceErr` 实例在任何字段设置时匹配，可以通过编写自定义的 `Is` 方法来实现：

```go
func (re ResourceErr) Is(target error) bool {
    if other, ok := target.(ResourceErr); ok {
        ignoreResource := other.Resource == ""
        ignoreCode := other.Code == 0
        matchResource := other.Resource == re.Resource
        matchCode := other.Code == re.Code
        return matchResource && matchCode ||
            matchResource && ignoreCode ||
            ignoreResource && matchCode
    }
    return false
}
```

现在，您可以找到例如所有与数据库相关的错误，无论代码如何：

```go
if errors.Is(err, ResourceErr{Resource: "Database"}) {
    fmt.Println("The database is broken:", err)
    // process the codes
}
```

您可以在[Go Playground](https://oreil.ly/Mz_Op)或[第九章存储库](https://oreil.ly/KCSeb)的 *sample_code/custom_is_error_pattern_match* 目录中查看此代码。

`errors.As` 函数允许您检查返回的错误（或其包装的任何错误）是否与特定类型匹配。它接受两个参数。第一个参数是正在检查的错误，第二个参数是您要查找的类型的变量的指针。如果函数返回 `true`，则在错误树中找到了匹配的错误，并且该匹配的错误被赋给第二个参数。如果函数返回 `false`，则在错误树中找不到匹配项。尝试使用 `MyErr`：

```go
err := AFunctionThatReturnsAnError()
var myErr MyErr
if errors.As(err, &myErr) {
    fmt.Println(myErr.Codes)
}
```

请注意，您使用 `var` 声明特定类型的变量，并将此变量的指针传递给 `errors.As`。

您不必将错误类型的变量指针作为 `errors.As` 的第二个参数传递。您可以传递一个指向接口的指针，以找到符合接口的错误：

```go
err := AFunctionThatReturnsAnError()
var coder interface {
    CodeVals() []int
}
if errors.As(err, &coder) {
    fmt.Println(coder.CodeVals())
}
```

该示例使用了匿名接口，但任何接口类型都可以。你可以在[第九章存储库](https://oreil.ly/KCSeb)的*sample_code/custom_as_error*目录中找到`errors.As`的两个示例。

###### 警告

如果`errors.As`的第二个参数不是指向错误或接口的指针，则该方法会 panic。

正如你可以通过`Is`方法覆盖默认的`errors.Is`比较一样，你也可以通过在你的错误上实现一个`As`方法来覆盖默认的`errors.As`比较。实现`As`方法是复杂的，并且需要反射（我将在第十六章中讨论 Go 中的反射）。只有在不寻常的情况下才应该这样做，例如当你想匹配一种类型的错误并返回另一种类型时。

###### 提示

当你寻找特定*实例*或特定*值*时，请使用`errors.Is`。当你寻找特定*类型*时，请使用`errors.As`。

# 使用 defer 包装错误

有时你会发现自己用相同的消息包装多个错误：

```go
func DoSomeThings(val1 int, val2 string) (string, error) {
    val3, err := doThing1(val1)
    if err != nil {
        return "", fmt.Errorf("in DoSomeThings: %w", err)
    }
    val4, err := doThing2(val2)
    if err != nil {
        return "", fmt.Errorf("in DoSomeThings: %w", err)
    }
    result, err := doThing3(val3, val4)
    if err != nil {
        return "", fmt.Errorf("in DoSomeThings: %w", err)
    }
    return result, nil
}
```

你可以通过使用`defer`来简化这段代码：

```go
func DoSomeThings(val1 int, val2 string) (_ string, err error) {
    defer func() {
        if err != nil {
            err = fmt.Errorf("in DoSomeThings: %w", err)
        }
    }()
    val3, err := doThing1(val1)
    if err != nil {
        return "", err
    }
    val4, err := doThing2(val2)
    if err != nil {
        return "", err
    }
    return doThing3(val3, val4)
}
```

你必须给返回值命名，这样你才能在延迟函数中引用`err`。如果你给一个返回值命名，那么所有的返回值都必须命名，所以在这里你对未显式赋值的字符串返回值使用了下划线。

在`defer`闭包中，代码会检查是否返回了错误。如果是这样，它会将错误重新赋值为一个新的错误，此错误会用一个消息包装原始错误，指示哪个函数检测到了错误。

当每个错误都用相同的消息包装时，这种模式效果很好。如果你想要用更多的细节定制包装错误，可以在每个`fmt.Errorf`中同时放置具体和一般的消息。

# panic 和 recover

之前的章节中提到过`panic`，但没有详细介绍它们是什么。*Panic* 类似于 Java 或 Python 中的`Error`。这是 Go 运行时在无法确定接下来应该发生什么时生成的一种状态。这几乎总是由于编程错误引起的，例如尝试读取超出切片末尾或将负大小传递给`make`。如果 Go 运行时检测到自身的错误，例如垃圾收集器的不当行为，它也会`panic`。但我从未见过这种情况发生。如果发生`panic`，最后责备运行时。

一旦发生 panic，当前函数立即退出，并且任何附加到当前函数的 defer 开始运行。当这些 defer 完成时，调用函数的 defer 运行，依此类推，直到达到`main`。然后程序以消息和堆栈跟踪退出。

###### 注意

如果除了主 goroutine 之外的 goroutine 中发生了 panic（goroutine 在“Goroutines”中介绍），则 defer 链以启动该 goroutine 的函数结束。如果*任何*goroutine 发生了 panic 但未被 recover，程序将退出。

如果您的程序中有任何不可恢复的情况，您可以自行创建`panic`。内置函数`panic`接受一个参数，可以是任何类型。通常是一个字符串。以下是一个简单的会触发 panic 的程序；您可以在[Go Playground](https://oreil.ly/yCBib)上运行它，或者在[第九章存储库](https://oreil.ly/KCSeb)的*sample_code/panic*目录中运行它：

```go
func doPanic(msg string) {
    panic(msg)
}

func main() {
    doPanic(os.Args[0])
}
```

运行此代码会产生以下输出：

```go
panic: /tmpfs/play

goroutine 1 [running]:
main.doPanic(...)
    /tmp/sandbox567884271/prog.go:6
main.main()
    /tmp/sandbox567884271/prog.go:10 +0x5f
```

如您所见，`panic`会打印出其消息，然后是堆栈跟踪。

Go 提供了一种捕获`panic`的方法，以提供更优雅的关闭或防止关闭。内置的`recover`函数是从`defer`中调用的，用于检查是否发生了`panic`。如果有`panic`，则返回给`panic`赋的值。一旦发生`recover`，执行会正常继续。让我们看看另一个示例程序。在[Go Playground](https://oreil.ly/f5Ybe)上运行它，或者在[第九章存储库](https://oreil.ly/KCSeb)的*sample_code/panic_recover*目录中运行它：

```go
func div60(i int) {
    defer func() {
        if v := recover(); v != nil {
            fmt.Println(v)
        }
    }()
    fmt.Println(60 / i)
}

func main() {
    for _, val := range []int{1, 2, 0, 6} {
        div60(val)
    }
}
```

使用`recover`有一个特定的模式。您可以使用`defer`注册一个函数来处理可能的`panic`。在`if`语句中调用`recover`，并检查是否找到了非 nil 值。您必须在`defer`中调用`recover`，因为一旦发生`panic`，只有延迟函数会运行。

运行此代码会产生以下输出：

```go
60
30
runtime error: integer divide by zero
10
```

由于`recover`使用非 nil 值来检测是否发生了`panic`，聪明的读者可能会提出一个问题：*如果您调用`panic(nil)`并且有一个`recover`会发生什么？*在 Go 版本 1.21 之前编译的代码中，答案是“没有什么了不起的事情。”在这些版本中，`recover`会停止`panic`的传播，但没有消息或数据表明发生了什么。从 Go 1.21 开始，`panic(nil)`的调用与`panic(new(runtime.PanicNilError))`相同。

虽然`panic`和`recover`看起来很像其他语言中的异常处理，但它们不打算这样使用。保留`panic`用于致命情况，并使用`recover`作为优雅处理这些情况的一种方式。如果您的程序发生了 panic，请小心尝试在 panic 发生后继续执行。在计算机由于内存或磁盘空间等资源不足而触发`panic`时，最安全的做法是使用`recover`将情况记录到监控软件并使用`os.Exit(1)`关闭。如果程序错误导致了 panic，您可以尝试继续执行，但很可能会再次遇到相同的问题。在上述示例程序中，如果出现除以零，则检查并返回错误是惯用的做法。

你不依赖于`panic`和`recover`的原因是`recover`不清楚*什么*可能失败。它只是确保*如果*某事失败，你可以打印出一条消息并继续。惯用的 Go 代码偏爱明确列出可能的失败条件的代码，而不是处理任何事情但什么都不说的短代码。

在一个情况下推荐使用`recover`。如果你正在为第三方创建库，请不要让`panic`逃离公共 API 的边界。如果可能发生`panic`，公共函数应该使用`recover`将`panic`转换为错误，返回它，并让调用代码决定如何处理它们。

###### 注意

虽然内置在 Go 中的 HTTP 服务器在处理程序中恢复了 panic，但戴维·西蒙兹在[GitHub 评论](https://oreil.ly/BGOmg)中说，截至 2015 年，Go 团队认为这是一个错误。

# 从错误获取堆栈跟踪

新 Go 开发人员倾向于使用`panic`和`recover`的原因之一是他们希望在出现问题时获得堆栈跟踪。默认情况下，Go 不提供这个功能。如我所示，你可以使用错误包装手动构建调用堆栈，但一些第三方带有错误类型的库会自动生成这些堆栈（参见第十章了解如何将第三方代码整合到你的程序中）。Cockroachdb 提供了一个[第三方库](https://oreil.ly/-n1EX)，其中包含用于带堆栈跟踪的错误包装的函数。

默认情况下，不会打印出堆栈跟踪。如果想看到堆栈跟踪，请使用`fmt.Printf`和详细输出动词（`%+v`）。查阅[文档](https://oreil.ly/3-5Ql)了解更多信息。

###### 注意

当你的错误中有一个堆栈跟踪时，输出会包含程序编译时计算机上文件的完整路径。如果不想暴露路径，构建代码时请使用`-trimpath`标志。这会用包名替换完整路径。

# 练习

查看*sample_code/exercise*目录中[第九章存储库](https://oreil.ly/KCSeb)中的代码。你将在每个练习中修改这些代码。它的功能是正确的，但应改进其错误处理。

1.  创建一个哨兵错误来表示无效的 ID。在`main`中，使用`errors.Is`检查哨兵错误，并在发现时打印消息。

1.  定义一个自定义错误类型来表示空字段错误。此错误应包括空`Employee`字段的名称。在`main`中，使用`errors.As`检查此错误。打印一个包含字段名的消息。

1.  而不是返回发现的第一个错误，返回一个包含所有验证期间发现的错误的单个错误。更新`main`中的代码，正确报告多个错误。

# 结束

本章涵盖了 Go 语言中的错误，它们是什么，如何定义自己的错误，以及如何检查它们。您还学习了`panic`和`recover`。下一章将讨论包和模块，如何在程序中使用第三方代码，以及如何发布您自己的代码供他人使用。
