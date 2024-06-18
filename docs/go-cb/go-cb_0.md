# 第一章：错误处理技巧

# 1.0 引言

亚历山大·蒲柏在他的*论批评的散文*中写道：“出错是人性的”。由于软件是由人类编写的（目前是这样），软件也会出错。就像人类错误一样，重要的是我们如何优雅地从这些错误中恢复。这就是错误处理的关键所在——当程序陷入我们未预料或未考虑其正常流程的情况时，我们如何恢复。

作为程序员，我们经常把错误处理当作繁琐的工作和事后思考。这本身通常就是一个错误。事实上，就像我们应该对待测试一样，错误处理应该是首要考虑的事情，从错误中恢复应该是良好软件设计的一部分。在 Go 语言中，错误处理被认真对待，尽管有点不传统。Go 语言在标准库中有`errors`包，提供了许多处理错误的函数，但大部分错误处理都内置在语言中或者是 Go 语言编程习惯的一部分。在本章中，我们将讨论 Go 语言中错误处理的一些基本思想。

## 错误不是异常

在许多编程语言中，如 Python 和 Java，通过异常来处理错误。异常是表示错误的对象，无论何时出错，都可以抛出异常。调用函数通常有`try`和`catch`（或者在 Python 中是`try`和`except`等），用于处理任何出现的问题。

Go 在这方面略有不同（或者完全不同，这取决于你的看法）。Go 没有异常处理。而是使用错误。`error`是一种内置类型，表示意外情况。不会抛出异常，而是创建一个错误并将其返回给调用函数。也许你会想，这不是说*奥德赛*不是荷马写的，而是另一个名叫荷马的希腊人，他也生活在 2800 年前吗？

嗯，并不完全是这样。实际上，异常根本不会被函数返回，事实上，你根本不知道会不会返回任何异常，或者是什么样的异常（有时候你知道，但那是另一回事）。你必须使用`try`和`catch`来捕捉异常。只有在出现问题时才会抛出异常（这就是为什么称其为异常），你可以用相同的`try`和`catch`包裹多条语句。另一方面，错误是有意返回给调用函数的，以便逐个检查并进行处理。

结果，那些对异常处理更熟悉的人发现在 Go 语言中处理错误尤其乏味。与在一系列语句中捕获异常不同，你必须每次检查返回的错误并单独处理它。当然，你也可以选择完全忽略这些错误，虽然按惯例，你应该认真对待每次出现的错误。唯一的例外可能是当你根本不关心返回的结果时。

# 1.1 处理错误

## 问题

你希望在你的代码中处理一个意外情况。

## 解决方案

如果你正在编写一个函数，除了返回值（如果有的话），还需返回一个`error`。如果你正在调用一个函数，检查返回的错误，如果不是`nil`，则相应地处理错误。

## 讨论

处理错误的两种方式是如果你正在编写一个函数或者如果你正在调用一个函数。

### 编写一个函数

Go 使用内置的`error`接口类型来表示错误，这实际上是一个接口。在编写函数时的经验法则是，如果函数可能失败，你需要返回一个错误，以及函数的其他返回值。这是可能的，因为 Go 允许多返回值。按照约定，错误应该是最后一个返回值，例如下面的这个允许你猜测一个数字的函数。

```go
func guess(number uint) (answer bool, err error) {
    if number > 99 {
        err = errors.New("Number is larger than 100")
    }
    // check if guess is correct
    return
}
```

函数的输入应该小于 100，并且该函数应返回一个指示猜测是否正确的 true 或 false。显然，如果输入数字大于 100，你希望引发一个错误，这就是我们通过使用`errors.New`函数来创建新错误的方式。

```go
err = errors.New("Number is larger than 100")
```

不过创建新错误的方式并不仅限于此，另一种流行的创建新错误的方式是在流行的`fmt`包中使用`Errorf`函数。

```go
err = fmt.Errorf("Number is larger than 100")
```

在这个例子中两者之间的主要区别微不足道，但`fmt.Errorf`允许你格式化字符串，就像`fmt`包中的`Printf`、`Sprintf`、`Sscanf`等函数一样。不过`fmt.Errorf`还有更多功能，因为它实际上允许你在另一个错误周围包装一个错误，这是我们将在第 3.4 节讨论的内容。

### 调用一个函数

另一个经验法则，与 Go 中的错误处理哲学一起出现的是“不要忽略错误”。在 Go 中处理错误的标准方式非常简单——你处理它就像处理任何其他返回值一样。下面的例子摘自第一章，第 1.5 节。

```go
str := "123456789"
num, err := strconv.ParseInt(str, 10, 64)
if err != nil {
    panic(err)
}
```

函数`strconv.ParseInt`返回两个值，第一个是转换后的整数，第二个是错误。你应该检查错误，如果错误为 nil，说明一切正常，可以继续程序流程。如果不是 nil，那么你应该处理它。在上面的例子中，我用错误调用了`panic`，我们稍后会在本章中详细讨论这个问题，但你可以根据需要处理它，包括忽略它。当然，你也可以像这样有意选择忽略它们。

```go
num, _ := strconv.ParseInt(str, 10, 64)
```

在这里，我们将返回的错误赋值给下划线 `_`，这意味着我们正在忽略它。无论哪种情况，都很明显你是在有意忽略返回的错误，Go 无法阻止你，但是代码审查或者你的 IDE 可以快速发现这些问题，尤其是在团队中。

### 为什么 Go 会以这种方式处理错误？

你可能会想知道为什么 Go 会这样做，而不像许多其他语言那样使用异常。异常似乎更容易处理，因为你可以把语句组合在一起。Go 强制你在每个函数调用时处理错误的方式可能让人觉得很繁琐。

然而，由于这种方式，异常也很容易被忽略，除非你有`try`和`catch`。如果你用`try`和`catch`包装一堆语句，很容易忽略处理特定的错误，并且如果把太多语句放在一起，可能会导致混淆。

这种做法的另一个好处是返回的错误是一个值，你可以像处理其他任何值一样使用它。虽然你也可以处理异常，但它们是`try`和`catch`循环中的构造，不在你的正常流程中。这看起来可能是一个细枝末节，但它很重要，因为这使得错误处理成为编写代码的一个必要部分，而不是可选的。

# 1.2 简化重复的错误处理

## 问题

你想减少重复错误处理代码的行数。

## 解决方案

使用辅助函数来减少重复错误处理代码的行数。

## 讨论

Go 语言中关于错误处理最常见的抱怨之一，特别是对新手而言，就是不得不进行重复检查感到很繁琐。让我们以打开一个 JSON 文件并将其解析到一个结构体的代码片段为例。

```go
func unmarshal() (person Person) {
	r, err := http.Get("https://swapi.dev/api/people/1")
	if err != nil {
		// handle error
	}
	defer r.Body.Close()

	data, err := io.ReadAll(r.Body)
	if err != nil {
		// handle error
	}

	err = json.Unmarshal(data, &person)
	if err != nil {
		// handle error
	}
	return
}
```

你可以看到，有三组错误处理，一组是当我们调用`http.Get`来获取 API 响应并将其存入`http.Response`时，另一组是当我们调用`io.ReadAll`从`http.Response`中获取 JSON 文本时，最后一组是将 JSON 文本解析到`Person`结构体中。这些调用都有可能失败，所以我们需要处理由这些失败导致的错误。

然而，这些错误处理例程非常相似且实际上是重复的。我们如何解决这个问题？有几种方法，但最直接的可能是使用辅助函数。

```go
func helperUnmarshal() (person Person) {
	r, err := http.Get("https://swapi.dev/api/people/1")
	check(err, "Calling SW people API")
	defer r.Body.Close()

	data, err := io.ReadAll(r.Body)
	check(err, "Read JSON from response")

	err = json.Unmarshal(data, &person)
	check(err, "Unmarshalling")
	return
}

func check(err error, msg string) {
	if err != nil {
		log.Println("Error encountered:", msg)
        // do common error handling stuff
	}
}
```

这里的辅助函数是 `check` 函数，它接受一个错误和一个字符串。 除了记录字符串外，你还可以放置所有可能想做的通用错误处理工作。 你还可以接受一个在遇到错误时执行的函数而不是一个字符串。

当然，这只是一种可能的辅助函数类型，让我们看看另一种。 这次我们将使用标准库中另一个包中发现的模式。 在 `text/template` 包中，你可以找到一个名为 `template.Must` 的辅助函数，它包装了返回 `(*Template, error)` 的函数。 如果该函数返回非 nil 错误，则 `Must` 会 panic。 我们可以类似地创建一个类似的东西来包装其他函数调用。

```go
func must(param interface{}, err error) interface{} {
	if err != nil {
		// handle errors
	}
	return param
}
```

因为它接受任何单个参数（使用 `interface{}`）并返回单个值（也使用 `interface{}`），所以我们可以将其用于返回单个值和错误的任何函数。 例如，我们可以将我们之前的 `unmarshal` 函数转换为类似于这样的函数。

```go
func mustUnmarshal() (person Person) {
	r := must(http.Get("https://swapi.dev/api/people/1")).(*http.Response)
	defer r.Body.Close()
	data := must(io.ReadAll(r.Body)).([]byte)
	must(nil, json.Unmarshal(data, &person))
	return
}
```

代码更加简洁，但同时也使代码更难读取，因此应谨慎使用这种辅助函数。

# 1.3 创建自定义错误

## 问题

你想要创建我们自己的定制错误以便传达遇到的错误的更多信息。

## 解决方案

创建一个新的基于字符串的错误，或者通过创建一个具有返回字符串的 `Error` 方法的结构体来实现 `error` 接口。

## 讨论

### 使用基于字符串的错误

实现一个自定义错误的最简单方法是创建一个基于字符串的新错误。 你可以使用 `errors.New` 函数，它只是创建一个带有简单字符串的错误，或者使用 `fmt.Errorf` 允许你使用格式化。

`errors.New` 函数非常简单直接。

```go
err := errors.New("Syntax error in the code")
```

`fmt.Errorf` 函数，就像 `fmt` 函数中的许多函数一样，允许在字符串内进行格式化。

```go
err := fmt.Errorf("Syntax error in the code at line %d", line)
```

### 实现错误接口

`builtin` 包包含所有内置类型、接口和函数的定义。 其中一个接口是 `error` 接口。

```go
type error interface {
    Error() string
}
```

正如你所看到的，任何具有返回字符串的 `Error` 方法的结构体都是错误。 因此，如果你想要定义自己的自定义错误以返回自定义错误消息，只需实现自己的结构体并添加一个 `Error` 方法。 例如，假设你正在编写一个用于通信的程序，并且想要创建自己的错误来表示通信期间的错误。

```go
type CommsError struct{}

func (m CommsError) Error() string {
	return "An error happened during data transfer"
}
```

当然，你通常不会只想覆盖 `Error`，你可以向自定义错误添加字段和其他方法以携带更多信息。

假设你想要提供关于错误出现在代码行的位置信息。 你可以创建一个自定义错误来包含这些信息。

```go
type SyntaxError struct {
	Line int
	Col  int
}

func (err *SyntaxError) Error() string {
	return fmt.Sprintf("Error at line %d, column %d", err.Line, err.Col)
}
```

当你遇到这样的错误时，你可以使用逗号，ok 惯用法进行类型转换（因为如果你类型转换它并且它不是那种类型，它将 panic），并提取额外的数据进行处理。

```go
if err != nil {
	err, ok := err.(*SyntaxError)
	if ok {
		// do something with the error
	} else {
		// do something else
	}
}
```

# 1.4 使用其他错误包裹错误

## 问题

您希望在返回错误之前为接收到的错误提供额外的信息和上下文。

## 解决方案

在返回前将收到的错误用自定义错误再次包裹。

## 讨论

有时，您会收到一个错误，但不只是返回它，而是在返回错误之前为其提供额外的上下文。例如，如果您遇到网络连接错误，您可能想知道发生错误的代码位置以及发生错误时正在做什么。

当然，您可以简单地提取信息，使用附加信息创建新的自定义错误并返回。或者，您也可以将错误包装在另一个错误中返回，通过调用堆栈传递错误并向其添加额外的信息和上下文。

包装错误有几种方法。最简单的方法就是再次使用`fmt.Errorf`，并将错误作为参数之一提供。

```go
err1 := errors.New("Oops something happened.")
err2 := fmt.Errorf("An error was encountered - %w", err1)
```

`%w`占位符允许我们将错误放置在格式化字符串中。在上面的例子中，`err2`包裹了`err1`。但是我们如何从`err2`中提取`err1`？`errors`包中有一个名为`Unwrap`的函数可以做到这一点。

```go
err := errors.Unwrap(err2)
```

这样就可以得到`err1`。

另一种包装错误的方法是创建一个自定义的错误结构体，像这样。

```go
type ConnectionError struct {
	Host string
	Port int
	Err  error
}

func (err *ConnectionError) Error() string {
	return fmt.Sprintf("Error connecting to %s at port %d", err.Host, err.Port)
}
```

记住，要使其成为错误，结构体应该有一个`Error`方法。为了允许结构体被展开，我们需要为其实现一个`Unwrap`函数。

```go
func (err *ConnectionError) Unwrap() error {
	return err.Err
}
```

# 1.5 检查错误

## 问题

您想要检查特定的错误或特定类型的错误。

## 解决方案

使用`errors.Is`和`errors.As`函数。`errors.Is`函数将错误与一个值进行比较，而`errors.As`函数则检查错误是否属于特定类型。

## 讨论

### 使用`errors.Is`

`errors.Is`函数本质上是一个相等性检查。假设在您的代码库中定义了一组自定义错误，例如当与 API 连接时发生错误时会出现`ApiErr`。

```go
var ApiErr error = errors.New("Error trying to get data from API")
```

在代码的其他位置，您有一个函数返回此错误。

```go
func connectAPI() error {
	// some other stuff happening here
	return ApiErr
}
```

您可以使用`errors.Is`来检查返回的错误是否真的是`ApiErr`。

```go
err := connectAPI()
if err != nil {
	if errors.Is(err, ApiErr) {
		// handle the API error
	}
}
```

您还可以验证`ApiErr`是否在包裹错误的链中的某个位置。让我们以一个返回`ConnectionError`并包装`ApiErr`的`connect`函数为例。

```go
func connect() error {
	return &ConnectionError{
		Host: "localhost",
		Port: 8080,
		Err:  ApiErr,
	}
}
```

该代码仍然有效，因为`ConnectionError`包裹了`ApiErr`。

```go
err := connect()
if err != nil {
	if errors.Is(err, ApiErr) {
		// handle the API error
	}
}
```

### 使用`errors.As`

`errors.As`函数允许我们检查特定类型的错误。让我们继续使用上面的例子，但这次我们要检查错误是否为`ConnectionError`类型。

```go
err := connect()
if err != nil {
	var connErr *ConnectionError
	if errors.As(err, &connErr) {
		log.Errorf("Cannot connect to host %s at port %d", connErr.Host, connErr.Port)
	}
}
```

我们可以使用`errors.As`来检查返回的错误是否真的是`ConnectionError`，通过传递返回的错误和类型为`*ConnectionError`的变量`connErr`。如果是，`errors.As`将把返回的错误赋给`connErr`，同时您可以处理错误。

# 1.6 使用 panic 处理错误

## 问题

您希望报告导致程序停止并且无法继续的错误。

## 解决方案

使用内置的`panic`函数来停止程序。

## 讨论

有时您的程序会遇到使其无法继续的错误。在这种情况下，您将希望停止程序。Go 提供了一个名为`panic`的内置函数，它接受任何类型的单个参数，并停止当前 goroutine 的正常执行。

当函数调用`panic`时，函数的正常执行立即停止，并且在函数返回给其调用者之前执行任何延迟操作（任何以`defer`开头的内容）。

调用函数也会发生 panic 并停止正常执行，并且执行延迟操作。这会一直上升，直到整个程序以非零退出代码退出。

让我们更仔细地看一下这个问题。我们创建了一个函数的正常流程，其中`main`调用`A`，`A`调用`B`，`B`调用`C`。

```go
package main

import "fmt"

func A() {
	defer fmt.Println("defer on A")
	fmt.Println("A")
	B()
	fmt.Println("end of A")
}

func B() {
	defer fmt.Println("defer on B")
	fmt.Println("B")
	C()
	fmt.Println("end of B")
}

func C() {
	defer fmt.Println("defer on C")
	fmt.Println("C")
	fmt.Println("end of C")
}

func main() {
	defer fmt.Println("defer on main")
	fmt.Println("main")
	A()
	fmt.Println("end of main")
}
```

执行结果如下，这是预期的正常流程。

```go
% go run main.go
main
A
B
C
end of C
defer on C
end of B
defer on B
end of A
defer on A
end of main
defer on main
```

如果我们在`C`中两个`fmt.Println`语句之间调用`panic`，会发生什么？

```go
func C() {
	defer fmt.Println("defer on C")
	fmt.Println("C")
	panic("panic called in C")
	fmt.Println("end of C")
}
```

当我们在执行过程中`C`调用`panic`时，`C`立即停止并执行其范围内的延迟代码。之后它上升到调用者`B`，`B`也立即停止并执行其范围内的延迟代码，并返回给其调用者`A`。同样的情况发生在`A`身上，它上升到`main`函数，该函数执行其范围内的延迟代码。由于这是链的末端，它将打印出`panic`的参数。

如果您在终端上运行此代码，您将看到以下内容。

```go
% go run main.go
main
A
B
C
defer on C
defer on B
defer on A
defer on main
panic: panic called in C
```

正如您从结果中看到的那样，所有函数中的其余代码都不会被执行，但是所有延迟代码在 panic 退出时都会执行。

# 1.7 从 panic 中恢复

## 问题

您的一个 goroutine 出现错误并且无法继续，调用了 panic，但您不希望停止程序的其余部分。

## 解决方案

使用内置的`recover`函数来阻止`panic`。这仅在延迟函数中有效。

## 讨论

在前一个示例中，我们看到`panic`如何停止代码的正常执行，运行延迟代码并上升直到程序终止。有时我们希望阻止`panic`终止程序。在这种情况下，我们可以使用内置的`recover`函数来阻止`panic`并继续程序的执行。

但是为什么我们要这样做呢？可能有几个原因。您可能正在使用一个在遇到无法恢复的情况时引发 panic 的包。但这并不意味着您希望程序终止（即使该部分代码无法继续）。或者您可能只是想停止 goroutine 的执行，而不希望杀死主程序，例如，如果您的 Web 应用正在运行，您不希望发生 panic 的处理程序关闭整个服务器。

无论是哪种情况，只有在`defer`内部使用`recover`才有效。这是因为当函数调用`panic`时，除了延迟执行的代码外，其他所有代码都将停止工作。

让我们使用前一个示例中的示例。

```go
package main

import "fmt"

func A() {
	defer fmt.Println("defer on A")
	fmt.Println("A")
	B()
	fmt.Println("end of A")
}

func B() {
	defer dontPanic()
	fmt.Println("B")
	C()
	fmt.Println("end of B")
}

func C() {
	defer fmt.Println("defer on C")
	fmt.Println("C")
	fmt.Println("end of C")
}

func main() {
	defer fmt.Println("defer on main")
	fmt.Println("main")
	A()
	fmt.Println("end of main")
}

func dontPanic() {
	err := recover()
	if err != nil {
		fmt.Println("panic called but everything's ok now:", err)
	} else {
		fmt.Println("defer on B")
	}
}
```

我们添加了一个名为`dontPanic`的新函数。在此函数中，我们调用内置函数`recover`。如果在程序冒泡`panic`时调用此函数，则会返回传递给`panic`的参数，并阻止`panic`继续。在正常情况下，`recover`只返回 nil，并且正常的延迟代码会运行。

执行如预期的正常流程。调用`recover`返回 nil，因此正常的延迟代码会运行。

```go
% go run main.go
main
A
B
C
end of C
defer on C
end of B
defer on B
end of A
defer on A
end of main
defer on main
```

现在让我们在`C`中添加一个`panic`。

```go
func C() {
	defer fmt.Println("defer on C")
	fmt.Println("C")
	panic("panic called in C")
	fmt.Println("end of C")
}
```

让我们再次运行它，看看会发生什么。

```go
% go run main.go
main
A
B
C
defer on C
panic called but everything's ok now: panic called in C
end of A
defer on A
end of main
defer on main
```

在`C`中调用`panic`时，`C`中的延迟代码会触发，而不会运行`C`中其余的代码，并冒泡到`B`。 `B`停止运行其余的代码并开始运行延迟代码，其中调用`dontPanic`。 `dontPanic`调用`recover`，返回传递给`C`中调用`panic`的参数，并运行恢复代码。

`B`的正常执行不会发生，但当`B`返回到`A`时一切正常，代码的正常执行流程继续进行。

# 1.8 处理中断

## 问题

程序从操作系统接收到中断信号（例如，如果用户按下`ctrl-c`），您希望进行清理并优雅地退出。

## 解决方案

使用 goroutine 使用`os/signal`包监控中断。将清理代码放在 goroutine 中。

## 讨论

信号是发送给运行中程序的消息，以触发程序内部的某些行为。信号是异步的，可以由操作系统或其他运行中的程序发送。当发送信号时，操作系统中断运行的程序以传递信号。如果进程具有信号处理程序，则将执行该处理程序；否则将执行默认处理程序。

在命令行上，某些键组合（例如`ctrl-c`）会触发信号（在本例中，`ctrl-c`发送`SIGINT`信号）到前台运行的程序。`SIGINT`信号或*信号中断*是一种中断运行中程序并导致其终止的信号。

您可以使用`os/signal`包在 Go 中捕获此类信号。

让我们看看如何做到这一点。

```go
ch := make(chan os.Signal)
signal.Notify(ch, os.Interrupt)

go func() {
	<-ch
	// clean up before graceful exit
	os.Exit(0)
}()
```

首先，我们创建一个通道`ch`来发送信号。然后我们使用`signal.Notify`函数将传入的信号转发到`ch`。`signal.Notify`的第一个参数是通道，第二个参数是可变参数，这意味着我们可以传入零个或多个参数。在这种情况下，我们传入我们想要捕获的各种信号。在上面的示例代码中，我们想要转发`os.Interrupt`，这是一个`syscall.SIGINT`或`ctrl-c`。如果不传入任何参数，则所有信号将被转发到通道。

设置好之后，我们会启动一个 goroutine，在那里我们通过从 `ch` 接收信号来等待信号的到来。这会导致 goroutine 阻塞，直到收到信号。一旦收到信号，我们继续执行该 goroutine，执行我们想要的任何清理例程，然后从程序中优雅地退出。
