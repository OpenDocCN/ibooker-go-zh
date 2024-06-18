# 第五章：函数

到目前为止，你的程序在`main`函数中被限制在几行代码中。现在是时候扩展一下了。在本章中，你将学习如何在 Go 语言中编写函数，并看到你可以用它们做的所有有趣的事情。

# 声明和调用函数

Go 函数的基础对于任何已经在其他具有一级函数的语言（如 C、Python、Ruby 或 JavaScript）中编程过的人来说都是熟悉的。（Go 也有方法，我会在第七章中讨论。）就像控制结构一样，Go 在函数特性上添加了自己的风格。有些是改进，而其他一些则是应该避免的实验。我会在本章中都覆盖到。

你已经看到了函数的声明和使用。每个 Go 程序都从一个`main`函数开始，并且你一直在调用`fmt.Println`函数来打印到屏幕上。由于`main`函数不接受参数或返回值，让我们看看当一个函数有参数时会是什么样子：

```go
func div(num int, denom int) int {
    if denom == 0 {
        return 0
    }
    return num / denom
}
```

让我们看看这段代码样本中的所有新内容。函数声明由四部分组成：关键字`func`、函数名称、输入参数和返回类型。输入参数在括号内列出，用逗号分隔，参数名在前，类型在后。Go 是一种类型化语言，因此必须指定参数的类型。返回类型写在输入参数的右括号和函数体的左大括号之间。

就像其他语言一样，Go 语言有一个`return`关键字用于从函数中返回值。如果一个函数返回一个值，你*必须*提供一个`return`。如果一个函数不返回任何东西，则不需要在函数末尾写上`return`语句。只有在你在函数的最后一行之前退出函数时，才需要`return`关键字。

`main`函数没有输入参数或返回值。当一个函数没有输入参数时，使用空括号（`()`）。当一个函数不返回任何东西时，在输入参数的右括号和函数体的左大括号之间什么也不写：

```go
func main() {
    result := div(5, 2)
    fmt.Println(result)
}
```

这个简单程序的代码在[第五章仓库](https://oreil.ly/EzU8N)的*sample_code/simple_div*目录中。

调用一个函数对于有经验的开发者来说应该是很熟悉的。在`:=`的右侧，你用值 5 和 2 调用`div`函数。在左侧，你将返回的值赋给变量`result`。

###### 提示

当你有两个或更多连续的相同类型的输入参数时，你可以像这样一次性指定类型：

```go
func div(num, denom int) int {
```

## 模拟命名和可选参数

在介绍 Go 的独特函数特性之前，我将提到 Go *没有*的两个功能：命名和可选输入参数。除了下一节将要讲解的一个例外，你必须为函数提供所有参数。如果你想模仿命名和可选参数，定义一个结构体，其中包含与所需参数匹配的字段，并将结构体传递给函数。示例 5-1 展示了演示此模式的代码片段。

##### 示例 5-1\. 使用结构体模拟命名参数

```go
type MyFuncOpts struct {
    FirstName string
    LastName  string
    Age       int
}

func MyFunc(opts MyFuncOpts) error {
    // do something here
}

func main() {
    MyFunc(MyFuncOpts{
        LastName: "Patel",
        Age:      50,
    })
    MyFunc(MyFuncOpts{
        FirstName: "Joe",
        LastName:  "Smith",
    })
}
```

本程序的代码位于 *sample_code/named_optional_parameters* 目录中的 [第五章存储库](https://oreil.ly/EzU8N)。

在实践中，没有命名和可选参数并不是限制。一个函数不应该有太多参数，并且当函数有很多输入时，命名和可选参数才有用。如果你发现自己处于这种情况中，你的函数可能太复杂了。

## 变参输入参数和切片

你一直在使用 `fmt.Println` 将结果打印到屏幕上，你可能已经注意到它允许任意数量的输入参数。它是如何做到的呢？与许多语言一样，Go 支持*变参参数*。变参参数*必须*是输入参数列表中的最后一个（或仅有的）参数。你用三个点（`...`）*在*类型前指示它。在函数内部创建的变量是指定类型的切片。你可以像使用任何其他切片一样使用它。我们通过编写一个程序来看看它们是如何工作的，该程序将一个基础数添加到可变数量的参数中，并将结果作为 `int` 类型的切片返回。你可以在 [The Go Playground](https://oreil.ly/nSad4) 或 [第五章存储库](https://oreil.ly/EzU8N) 的 *sample_code/variadic* 目录中运行此程序。首先，编写变参函数：

```go
func addTo(base int, vals ...int) []int {
    out := make([]int, 0, len(vals))
    for _, v := range vals {
        out = append(out, base+v)
    }
    return out
}
```

现在以几种方式调用它：

```go
func main() {
    fmt.Println(addTo(3))
    fmt.Println(addTo(3, 2))
    fmt.Println(addTo(3, 2, 4, 6, 8))
    a := []int{4, 3}
    fmt.Println(addTo(3, a...))
    fmt.Println(addTo(3, []int{1, 2, 3, 4, 5}...))
}
```

正如你所见，你可以为变参参数提供任意多的值，也可以完全不提供值。由于变参参数被转换为切片，因此你可以将切片作为输入。然而，你必须在变量或切片文字后面加上三个点（`...`）。如果不这样做，将导致编译时错误。

当你构建并运行此程序时，会得到以下输出：

```go
[]
[5]
[5 7 9 11]
[7 6]
[4 5 6 7 8]
```

## 多返回值

你会看到 Go 和其他语言之间的第一个区别是 Go 允许多返回值。让我们向前面的除法程序添加一个小功能。你将从函数中返回被除数和余数。下面是更新后的函数：

```go
func divAndRemainder(num, denom int) (int, int, error) {
    if denom == 0 {
        return 0, 0, errors.New("cannot divide by zero")
    }
    return num / denom, num % denom, nil
}
```

支持多返回值的几个更改。当一个 Go 函数返回多个值时，返回值的类型将以逗号分隔放在括号内。此外，如果一个函数返回多个值，你必须用逗号分隔它们全部返回。不要在返回值周围加括号；这会导致编译时错误。

还有一些您尚未看到的内容：创建并返回`error`。如果您想了解更多关于错误的信息，请跳转到第九章。目前，您只需知道可以使用 Go 的多返回值支持在函数出现问题时返回一个`error`。如果函数成功完成，将为错误的值返回`nil`。按照惯例，`error`始终是函数返回的最后一个（或唯一的）值。

调用更新后的函数如下：

```go
func main() {
    result, remainder, err := divAndRemainder(5, 2)
    if err != nil {
        fmt.Println(err)
        os.Exit(1)
    }
    fmt.Println(result, remainder)
}
```

此程序的代码位于[第五章存储库](https://oreil.ly/EzU8N)中的*sample_code/updated_div*目录中。

我在“var Versus :=”中讨论了一次性分配多个值的问题。在这里，您正在使用该功能将函数调用的结果分配给三个变量。在`:=`的右侧，您使用值 5 和 2 调用`divAndRemainder`函数。在左侧，您将返回的值分配给变量`result`、`remainder`和`err`。通过将`err`与`nil`进行比较来检查是否发生了错误。

## 多返回值是多个值

如果您熟悉 Python，可能会认为多返回值与 Python 函数返回的元组类似，如果将元组的值分配给多个变量，则可以选择进行解构。示例 5-2 展示了在 Python 解释器中运行的一些示例代码。

##### 示例 5-2\. Python 中的多返回值是解构的元组

```go
>>> def div_and_remainder(n,d):
...   if d == 0:
...     raise Exception("cannot divide by zero")
...   return n / d, n % d
>>> v = div_and_remainder(5,2)
>>> v
(2.5, 1)
>>> result, remainder = div_and_remainder(5,2)
>>> result
2.5
>>> remainder
1
```

在 Python 中，您可以将所有返回的值分配给单个变量或多个变量。但是，Go 不是这样工作的。您必须为函数返回的每个值分配一个变量。如果尝试将多个返回值分配给一个变量，则会收到编译时错误。

## 忽略返回值

如果调用函数时不想使用所有返回的值怎么办？正如“Unused Variables”中所述，Go 不允许未使用的变量。如果函数返回多个值，但您不需要读取一个或多个值，则将未使用的值分配给名称`_`。例如，如果您不打算读取`remainder`，则可以将赋值写为`result, _, err := divAndRemainder(5, 2)`。

令人惊讶的是，Go 确实允许您隐式地忽略*所有*函数的返回值。您可以写`divAndRemainder(5,2)`，返回的值会被丢弃。实际上，自最早的示例以来，您一直在这样做：`fmt.Println`返回两个值，但习惯上会忽略它们。在几乎所有其他情况下，您应该明确地使用下划线来表示您正在忽略返回值。

###### 提示

当您不需要读取函数返回的值时，请使用`_`。

## 命名返回值

除了允许从函数返回多个值外，Go 还允许为返回的值指定*名称*。再次重写`divAndRemainder`函数，这次使用命名返回值：

```go
func divAndRemainder(num, denom int) (result int, remainder int, err error) {
    if denom == 0 {
        err = errors.New("cannot divide by zero")
        return result, remainder, err
    }
    result, remainder = num/denom, num%denom
    return result, remainder, err
}
```

当你为返回值提供名称时，你所做的是预先声明在函数中用于保存返回值的变量。它们被写成括号内的逗号分隔列表。*即使只有单个返回值，你也必须用括号括起命名的返回值*。命名返回值在创建时被初始化为它们的零值。这意味着你可以在任何显式使用或分配之前返回它们。

###### 小贴士

如果你只想为*一些*返回值命名，可以使用`_`作为任何你希望匿名的返回值的名称。

有一件重要的事情需要注意：用于命名返回值的名称仅在函数内部有效；它不会强制外部使用任何名称。将返回值分配给不同名称的变量是完全合法的：

```go
func main() {
    x, y, z := divAndRemainder(5, 2)
    fmt.Println(x, y, z)
}
```

虽然命名返回值有时可以帮助澄清代码，但它们确实存在一些潜在的特例情况。首先是阴影问题。与任何其他变量一样，你可以遮蔽一个命名返回值。确保你正在将值分配给返回值而不是其阴影。

命名返回值的另一个问题是你不一定要返回它们。让我们看看`divAndRemainder`的另一个变体。你可以在[The Go Playground](https://oreil.ly/FzUkw)上运行它，或者在[第五章存储库](https://oreil.ly/EzU8N)的*sample_code/named_div*目录中的*main.go*中的`divAndRemainderConfusing`函数中运行它：

```go
func divAndRemainder(num, denom int) (result int, remainder int, err error) {
    // assign some values
    result, remainder = 20, 30
    if denom == 0 {
        return 0, 0, errors.New("cannot divide by zero")
    }
    return num / denom, num % denom, nil
}
```

注意，你分配了`result`和`remainder`的值，然后直接返回了不同的值。在运行这段代码之前，试着猜测当传递 5 和 2 到这个函数时会发生什么。结果可能会让你惊讶：

```go
2 1
```

即使从`return`语句返回的值从未分配给命名返回参数，它们也会被返回。这是因为 Go 编译器插入了将返回的任何内容分配给返回参数的代码。命名返回参数提供了一种声明*意图*使用变量来保存返回值的方法，但不*要求*你使用它们。

一些开发人员喜欢使用命名返回参数，因为它们提供了额外的文档。然而，我发现它们的价值有限。阴影使它们变得混乱，忽略它们也一样。命名返回参数在一个情况下是必不可少的。我将在本章稍后讨论`defer`时谈到这一点。

## 空白返回——永远不要使用这些！

如果你使用命名返回值，你需要注意 Go 语言中一个严重的设计缺陷：空白（有时称为*naked*）返回。如果你有命名的返回值，你可以只写`return`而不指定返回的值。这将返回分配给命名返回值的最后值。最后一次，用空白返回重写`divAndRemainder`函数：

```go
func divAndRemainder(num, denom int) (result int, remainder int, err error) {
    if denom == 0 {
        err = errors.New("cannot divide by zero")
        return
    }
    result, remainder = num/denom, num%denom
    return
}
```

使用空返回会对函数做一些额外的更改。当输入无效时，函数会立即返回。由于未给`result`和`remainder`赋值，它们的零值会被返回。如果您正在返回命名返回值的零值，请确保它们是有意义的。还要注意，您仍然必须在函数末尾放置一个`return`。即使函数使用空返回，该函数也会返回值。省略`return`将导致编译时错误。

起初，您可能会发现空返回很方便，因为它们允许您避免一些输入。然而，大多数经验丰富的 Go 开发人员认为空返回是一个坏主意，因为它们会使数据流变得更难理解。优秀的软件应该清晰易读；它的运行过程应该是显而易见的。当您使用空返回时，代码的读者需要通过程序回溯查找最后一次给返回参数赋值的值，才能看到实际返回的内容。

###### 警告

如果您的函数返回值，*永远不要*使用空返回。这会让弄清楚实际返回的值变得非常混乱。

# 函数是值

就像在许多其他语言中一样，Go 语言中的函数也是值。函数的类型由关键字`func`、参数类型和返回值类型组成。这种组合称为函数的*签名*。具有完全相同数量和类型的参数和返回值的任何函数都符合该类型的签名。

由于函数是值，因此可以声明一个函数变量：

```go
var myFuncVariable func(string) int
```

`myFuncVariable`可以被分配任何具有单一`string`类型参数和单一`int`类型返回值的函数。以下是一个更长的示例：

```go
func f1(a string) int {
    return len(a)
}

func f2(a string) int {
    total := 0
    for _, v := range a {
        total += int(v)
    }
    return total
}

func main() {
    var myFuncVariable func(string) int
    myFuncVariable = f1
    result := myFuncVariable("Hello")
    fmt.Println(result)

    myFuncVariable = f2
    result = myFuncVariable("Hello")
    fmt.Println(result)
}
```

您可以在[The Go Playground](https://oreil.ly/lPj9X)或[第五章代码库的 sample_code/func_value 目录](https://oreil.ly/EzU8N)上运行此程序。输出结果为：

```go
5
500
```

函数变量的默认零值为`nil`。尝试使用具有`nil`值的函数变量会导致恐慌（这在“panic 和 recover”中有涵盖）。

将函数作为值可以让您做一些聪明的事情，例如使用函数作为映射中的值来构建一个简单的计算器。让我们看看这是如何工作的。以下代码可在[The Go Playground](https://oreil.ly/L59VY)或[第五章代码库的 sample_code/calculator 目录](https://oreil.ly/EzU8N)中找到。首先，创建一组具有相同签名的函数：

```go
func add(i int, j int) int { return i + j }

func sub(i int, j int) int { return i - j }

func mul(i int, j int) int { return i * j }

func div(i int, j int) int { return i / j }
```

接下来，创建一个映射，将数学运算符与每个函数关联起来：

```go
var opMap = map[string]func(int, int) int{
    "+": add,
    "-": sub,
    "*": mul,
    "/": div,
}
```

最后，尝试使用几个表达式来测试计算器：

```go
func main() {
    expressions := [][]string{
        {"2", "+", "3"},
        {"2", "-", "3"},
        {"2", "*", "3"},
        {"2", "/", "3"},
        {"2", "%", "3"},
        {"two", "+", "three"},
        {"5"},
    }
    for _, expression := range expressions {
        if len(expression) != 3 {
            fmt.Println("invalid expression:", expression)
            continue
        }
        p1, err := strconv.Atoi(expression[0])
        if err != nil {
            fmt.Println(err)
            continue
        }
        op := expression[1]
        opFunc, ok := opMap[op]
        if !ok {
            fmt.Println("unsupported operator:", op)
            continue
        }
        p2, err := strconv.Atoi(expression[2])
        if err != nil {
            fmt.Println(err)
            continue
        }
        result := opFunc(p1, p2)
        fmt.Println(result)
    }
}
```

使用标准库中的`strconv.Atoi`函数将`string`转换为`int`时，第二个返回值是`error`。和以往一样，您需要检查函数返回的错误并正确处理错误条件。

你使用`op`作为`opMap`映射中的键，并将与该键关联的值分配给变量`opFunc`。`opFunc`的类型是`func(int, int) int`。如果映射中没有与提供的键关联的函数，你会打印错误消息并跳过循环的其余部分。然后，你使用之前解码的`p1`和`p2`变量调用分配给`opFunc`变量的函数。在变量中调用函数的方式与直接调用函数的方式完全相同。

当你运行这个程序时，你可以看到简单计算器的工作：

```go
5
-1
6
0
unsupported operator: %
strconv.Atoi: parsing "two": invalid syntax
invalid expression: [5]
```

###### 注意

不要编写脆弱的程序。这个示例的核心逻辑相对简短。在`for`循环内的 22 行代码中，有 6 行用于实现实际算法，另外 16 行用于错误检查和数据验证。你可能会被诱惑不验证输入数据或检查错误，但这样做会产生不稳定、难以维护的代码。错误处理是区分专业人员和业余人员的关键。

## 函数类型声明

就像你可以使用`type`关键字定义`struct`一样，你也可以用它来定义函数类型（我将在第七章详细介绍类型声明）：

```go
type opFuncType func(int,int) int
```

然后你可以将`opMap`的声明重写为这样：

```go
var opMap = map[string]opFuncType {
    // same as before
}
```

你不需要修改这些函数。任何具有两个`int`类型输入参数和单个`int`类型返回值的函数都会自动满足该类型，并可以被分配为映射中的值。

声明函数类型的优点是什么？一个用途是文档化。如果你要多次引用某个东西，给它起个名字是很有用的。在“函数类型是接口的桥梁”中，你会看到另一个用途。

## 匿名函数

你不仅可以将函数分配给变量，还可以在函数内部定义新函数并将其分配给变量。这里有一个演示程序，你可以在[Go Playground](https://oreil.ly/WPI6w)上运行它。该代码也可以在[第五章代码库](https://oreil.ly/EzU8N)的*sample_code/anon_func*目录中找到：

```go
func main() {
    f := func(j int) {
        fmt.Println("printing", j, "from inside of an anonymous function")
    }
    for i := 0; i < 5; i++ {
        f(i)
    }
}
```

内部函数是*匿名*的；它们没有名称。你可以使用关键字`func`紧跟输入参数、返回值和开括号来声明匿名函数。试图在`func`和输入参数之间放置函数名是编译时错误。

就像任何其他函数一样，通过使用括号来调用匿名函数。在这个例子中，你将`for`循环中的`i`变量传递到这里。它被分配给匿名函数的`j`输入参数。

该程序输出如下：

```go
printing 0 from inside of an anonymous function
printing 1 from inside of an anonymous function
printing 2 from inside of an anonymous function
printing 3 from inside of an anonymous function
printing 4 from inside of an anonymous function
```

不必将匿名函数分配给变量。你可以直接编写内联函数并立即调用它们。前面的程序可以重写为这样：

```go
func main() {
    for i := 0; i < 5; i++ {
        func(j int) {
            fmt.Println("printing", j, "from inside of an anonymous function")
        }(i)
    }
}
```

你可以在[The Go Playground](https://oreil.ly/EnkN6)上运行这个示例，或者在[第五章代码库](https://oreil.ly/EzU8N)的*sample_code/anon_func*目录中运行。

现在，这不是你通常会做的事情。如果你正在声明并立即执行一个匿名函数，那么你可能会摆脱匿名函数，直接调用代码。然而，在不将匿名函数赋给变量的情况下声明它们，在两种情况下是有用的：`defer`语句和启动 goroutines。稍后我会讲解`defer`语句。Goroutines 在第十二章中有详细介绍。

由于你可以在包范围内声明变量，你也可以声明分配给匿名函数的包范围变量：

```go
var (
    add = func(i, j int) int { return i + j }
    sub = func(i, j int) int { return i - j }
    mul = func(i, j int) int { return i * j }
    div = func(i, j int) int { return i / j }
)

func main() {
    x := add(2, 3)
    fmt.Println(x)
}
```

与普通函数定义不同，你可以给包级别的匿名函数分配一个新值：

```go
func main() {
    x := add(2, 3)
    fmt.Println(x)
    changeAdd()
    y := add(2, 3)
    fmt.Println(y)
}

func changeAdd() {
    add = func(i, j int) int { return i + j + j }
}
```

运行这段代码将得到以下输出：

```go
5
8
```

你可以在[The Go Playground](https://oreil.ly/nK6Z9)上尝试这个示例。这段代码也在[第五章代码库](https://oreil.ly/EzU8N)的*sample_code/package_level_anon*目录中。

在使用包级别的匿名函数之前，请务必确定你确实需要这种能力。包级别的状态应该是不可变的，以便更容易理解数据流。如果一个函数的含义在程序运行时发生变化，不仅数据流变得难以理解，处理数据也变得困难。

# 闭包

在函数内部声明的函数是特殊的；它们是*closures*。这是一个计算机科学术语，意味着在函数内部声明的函数能够访问和修改在外部函数中声明的变量。让我们看一个快速的示例，看看这是如何工作的。你可以在[The Go Playground](https://oreil.ly/6aILJ)上找到这段代码。这段代码也在[第五章代码库](https://oreil.ly/EzU8N)的*sample_code/closure*目录中：

```go
func main() {
    a := 20
    f := func() {
        fmt.Println(a)
        a = 30
    }
    f()
    fmt.Println(a)
}
```

运行这个程序将得到以下输出：

```go
20
30
```

分配给`f`的匿名函数可以读取和写入`a`，即使`a`没有传递给函数。

就像任何内部作用域一样，你可以在闭包内部遮蔽一个变量。如果你改变代码为：

```go
func main() {
    a := 20
    f := func() {
        fmt.Println(a)
        a := 30
        fmt.Println(a)
    }
    f()
    fmt.Println(a)
}
```

这将会打印出以下内容：

```go
20
30
20
```

在闭包中使用`:=`而不是`=`会创建一个新的`a`，当闭包退出时它将不复存在。在处理内部函数时，务必使用正确的赋值操作符，特别是当左手边有多个变量时。你可以在[The Go Playground](https://oreil.ly/JXn8P)上尝试这段代码。这段代码也在[第五章代码库](https://oreil.ly/EzU8N)的*sample_code/closure_shadow*目录中。

这种内部函数和闭包的东西起初可能看起来并不那么有用。在一个更大的函数内部创建微型函数有什么好处呢？为什么 Go 语言具有这个特性？

闭包允许您限制函数的作用域。如果一个函数只会从另一个函数调用，但会被多次调用，可以使用内部函数来“隐藏”调用函数。这样可以减少包级别的声明数量，从而更容易找到未使用的名称。

如果在函数内部多次重复使用某个逻辑片段，可以使用闭包来消除这种重复。我编写了一个简单的 Lisp 解释器，其中有一个`Scan`函数，用于处理输入字符串以查找 Lisp 程序的各个部分。它依赖于两个闭包，`buildCurToken` 和 `update`，以使代码更简洁、更易于理解。您可以在 [GitHub](https://oreil.ly/qanW3) 上查看它。

当闭包被传递给其他函数或从函数中返回时，它们确实变得非常有趣。它们允许您获取函数内部的变量并在函数外部使用这些值。

## 将函数作为参数传递

由于函数是值，您可以使用其参数和返回类型指定函数的类型，因此可以将函数作为参数传递给其他函数。如果您不习惯将函数视为数据来处理，您可能需要一些时间来考虑创建引用局部变量的闭包并将其传递给另一个函数的后果。这是一个非常有用的模式，在标准库中多次出现。

其中一个例子是对切片进行排序。标准库中的`sort`包含一个名为`sort.Slice`的函数。它接受任何切片以及一个用于对传入切片排序的函数。我们来看看如何通过使用不同的字段对结构体切片进行排序。

###### 注意

`sort.Slice` 函数在 Go 语言引入泛型之前就存在了，因此它会执行一些内部魔法来使其与任何类型的切片一起工作。我将在 第十六章 中更详细地讨论这些魔法。

让我们看看如何使用闭包以不同方式对相同数据进行排序。您可以在 [The Go Playground](https://oreil.ly/deOHW) 上运行此代码。该代码也位于 [第五章代码库](https://oreil.ly/EzU8N) 的 *sample_code/sort_sample* 目录中。首先，定义一个简单的类型，一个该类型值的切片，并打印出该切片：

```go
type Person struct {
    FirstName string
    LastName  string
    Age       int
}

people := []Person{
    {"Pat", "Patterson", 37},
    {"Tracy", "Bobdaughter", 23},
    {"Fred", "Fredson", 18},
}
fmt.Println(people)
```

接下来，按姓氏对切片进行排序并打印出结果：

```go
// sort by last name
sort.Slice(people, func(i, j int) bool {
    return people[i].LastName < people[j].LastName
})
fmt.Println(people)
```

传递给 `sort.Slice` 的闭包有两个参数，`i` 和 `j`，但在闭包内部使用了 `people`，因此可以按 `LastName` 字段对其进行排序。在计算机科学术语中，`people` 被 *闭包捕获*。接下来，您可以按 `Age` 字段进行相同的操作：

```go
// sort by age
sort.Slice(people, func(i, j int) bool {
    return people[i].Age < people[j].Age
})
fmt.Println(people)
```

运行这段代码将得到以下输出：

```go
[{Pat Patterson 37} {Tracy Bobdaughter 23} {Fred Fredson 18}]
[{Tracy Bobdaughter 23} {Fred Fredson 18} {Pat Patterson 37}]
[{Fred Fredson 18} {Tracy Bobdaughter 23} {Pat Patterson 37}]
```

调用`sort.Slice`会改变`people`切片。我在 “Go Is Call by Value” 中简要讨论了这一点，并在下一章节中详细介绍。

###### 提示

将函数作为参数传递给其他函数对于在同一类型数据上执行不同操作非常有用。

## 从函数中返回函数

除了使用闭包将一些函数状态传递给另一个函数外，你还可以从函数中返回一个闭包。让我们通过编写一个返回乘法器函数的函数来演示这一点。你可以在[The Go Playground](https://oreil.ly/8tpbN)上运行这个程序。代码也在[第五章仓库](https://oreil.ly/EzU8N)的*sample_code/makeMult*目录中。这是一个返回闭包的函数：

```go
func makeMult(base int) func(int) int {
    return func(factor int) int {
        return base * factor
    }
}
```

下面是函数的使用方法：

```go
func main() {
    twoBase := makeMult(2)
    threeBase := makeMult(3)
    for i := 0; i < 3; i++ {
        fmt.Println(twoBase(i), threeBase(i))
    }
}
```

运行这个程序将得到以下输出：

```go
0 0
2 3
4 6
```

现在你已经看到闭包的实际应用，你可能想知道 Go 开发者经常使用它们多少。事实证明，它们非常有用。你看到它们如何用于对切片进行排序。闭包还用于使用`sort.Search`高效地搜索排序的切片。至于返回闭包，当你为 Web 服务器构建中间件时，你会在“中间件”中看到这种模式。Go 还使用闭包实现资源清理，通过`defer`关键字。

###### 注意

如果你花时间与使用像 Haskell 这样的函数式编程语言的程序员在一起，你可能会听到*高阶函数*这个术语。这是一种说法，即一个函数的输入参数或返回值是一个函数。作为 Go 开发者，你与他们一样酷！

# defer

程序经常创建临时资源，如文件或网络连接，需要进行清理。无论一个函数有多少个退出点，或者函数是否成功完成，都必须进行清理。在 Go 中，清理代码使用`defer`关键字附加到函数中。

让我们看看如何使用`defer`来释放资源。通过编写一个简单版本的`cat`，Unix 的用于打印文件内容的实用程序。你不能在 The Go Playground 上传入参数，但是你可以在[第五章仓库](https://oreil.ly/EzU8N)的*sample_code/simple_cat*目录中找到这个示例的代码：

```go
func main() {
    if len(os.Args) < 2 {
        log.Fatal("no file specified")
    }
    f, err := os.Open(os.Args[1])
    if err != nil {
        log.Fatal(err)
    }
    defer f.Close()
    data := make([]byte, 2048)
    for {
        count, err := f.Read(data)
        os.Stdout.Write(data[:count])
        if err != nil {
            if err != io.EOF {
                log.Fatal(err)
            }
            break
        }
    }
}
```

这个例子介绍了我在后续章节中更详细介绍的一些新功能。随时阅读以获取更多信息。

首先，通过检查`os.Args`（在`os`包中的切片）的长度来确保命令行中指定了文件名。`os.Args`的第一个值是程序的名称。剩余的值是传递给程序的参数。检查`os.Args`的长度至少为 2 来确定是否提供了程序的参数。如果没有，使用`log`包中的`Fatal`函数打印一条消息并退出程序。接下来，使用`os`包中的`Open`函数获取一个只读文件句柄。`Open`函数返回的第二个值是一个错误。如果打开文件时出现问题，打印错误消息并退出程序。正如前面提到的，我会在第九章讨论错误。

一旦确定存在有效的文件句柄，你需要在使用后关闭它，无论如何退出函数。为了确保清理代码运行，你使用`defer`关键字，后跟函数或方法调用。在这种情况下，你使用文件变量的`Close`方法。（我在第七章中详细介绍了 Go 中的`at`方法。）通常情况下，函数调用会立即运行，但`defer`延迟调用直到周围函数退出。

通过将字节片段传递给文件变量的`Read`方法，你可以从文件句柄读取。我将在“io and Friends”中详细讨论如何使用这个方法，但`Read`会返回已读入片段的字节数和一个错误。如果发生错误，你需要检查是否是文件结束标记。如果已到文件末尾，使用`break`退出`for`循环。对于所有其他错误，使用`log.Fatal`报告并立即退出。我将在“Go Is Call by Value”中稍作介绍切片和函数参数，并在下一章节中详细讨论这种模式时，详细讨论指针。

从*simple_cat*目录内构建并运行程序会产生以下结果：

```go
$ go build
$ ./simple_cat simple_cat.go
package main

import (
    "io"
    "log"
    "os"
)
...

```

你应该了解关于`defer`的更多事项。首先，你可以在`defer`中使用函数、方法或闭包。（当我提到带有`defer`的函数时，心理上扩展为“函数、方法或闭包”。）

你可以在 Go 函数中`defer`多个函数。它们按照后进先出（LIFO）的顺序运行；最后注册的`defer`首先运行。

在返回语句之后，`defer`函数内的代码会运行。正如我提到的，你可以向`defer`提供带有输入参数的函数。输入参数会立即评估，并且它们的值会存储，直到函数运行。

这是一个快速示例，演示了`defer`的这两个特性：

```go
func deferExample() int {
    a := 10
    defer func(val int) {
        fmt.Println("first:", val)
    }(a)
    a = 20
    defer func(val int) {
        fmt.Println("second:", val)
    }(a)
    a = 30
    fmt.Println("exiting:", a)
    return a
}
```

运行此代码会产生以下输出：

```go
exiting: 30
second: 20
first: 10
```

你可以在[Go Playground](https://oreil.ly/SgAcq)上运行此代码。它还可以在[第五章代码库](https://oreil.ly/EzU8N)的*sample_code/defer_example*目录中找到。

###### 注意

你可以向`defer`提供返回值的函数，但没有办法读取这些值：

```go
func example() {
    defer func() int {
        return 2 // there's no way to read this value
    }()
}
```

也许你在想是否有一种方法让延迟函数检查或修改其周围函数的返回值。确实有一种方法，这也是使用命名返回值的最佳理由。它允许你的代码根据错误采取行动。当我在第九章讨论错误时，我将讨论一种使用`defer`向从函数返回的错误添加上下文信息的模式。让我们看一种使用命名返回值和`defer`处理数据库事务清理的方法：

```go
func DoSomeInserts(ctx context.Context, db *sql.DB, value1, value2 string)
                  (err error) {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer func() {
        if err == nil {
            err = tx.Commit()
        }
        if err != nil {
            tx.Rollback()
        }
    }()
    _, err = tx.ExecContext(ctx, "INSERT INTO FOO (val) values $1", value1)
    if err != nil {
        return err
    }
    // use tx to do more database inserts here
    return nil
}
```

我不会在本书中涵盖 Go 的数据库支持，但标准库在`database/sql`包中提供了广泛的数据库支持。在示例函数中，您创建一个事务来执行一系列数据库插入操作。如果其中任何一个失败，您希望回滚（不修改数据库）。如果它们全部成功，您希望提交（保存数据库更改）。您使用`defer`和闭包来检查是否已给`err`赋值。如果没有，您运行`tx.Commit`，这也可能返回一个错误。如果返回了错误，变量`err`将被修改。如果任何数据库交互返回错误，您会调用`tx.Rollback`。

###### 注意

新的 Go 开发者往往会忘记在指定`defer`函数时的大括号后面加上括号。如果省略它们，这是一个编译时错误，最终会形成习惯。记住提供括号可以让您指定在运行函数时传递的值。

Go 中的常见模式是，分配资源的函数还会返回一个清理资源的闭包。在[第五章的示例代码/simple_cat_cancel 目录](https://oreil.ly/EzU8N)中，有一个重写的简单`cat`程序。首先，您编写一个打开文件并返回一个闭包的辅助函数：

```go
func getFile(name string) (*os.File, func(), error) {
    file, err := os.Open(name)
    if err != nil {
        return nil, nil, err
    }
    return file, func() {
        file.Close()
    }, nil
}
```

辅助函数返回一个文件、一个函数和一个错误。`*`表示在 Go 中，文件引用是一个指针。我将在下一章中详细讨论这一点。

现在在`main`中，您使用`getFile`函数：

```go
f, closer, err := getFile(os.Args[1])
if err != nil {
    log.Fatal(err)
}
defer closer()
```

因为 Go 不允许未使用的变量，从函数返回`closer`意味着如果函数未被调用，则程序将无法编译。这提醒用户使用`defer`。正如前面所述，当您`defer`时，在`closer`后面加上括号。

###### 注意

如果您习惯于使用语言中的块在函数内部控制资源清理（例如 Java、JavaScript 和 Python 中的`try/catch/finally`块或 Ruby 中的`begin/rescue/ensure`块），使用`defer`可能会感觉奇怪。

这些资源清理块的缺点是它们会在函数中创建另一个缩进级别，这使得代码更难阅读。嵌套代码难以跟踪不仅仅是我的观点。在一项发表于[2017 年的*Empirical Software Engineering*论文](https://oreil.ly/VcYrR)中，Vard Antinyan 等人发现，“在提出的十一种代码特征中，只有两种明显影响复杂性增长：嵌套深度和缺乏结构。”

关于使程序更易于阅读和理解的研究并不新鲜。您可以找到许多几十年前的论文，包括 Richard Miara 等人在 1983 年的[一篇论文](https://oreil.ly/s0xcq)，尝试找出适合使用的正确缩进量（根据他们的结果，是两到四个空格）。

# Go 是按值调用的

你可能听人说 Go 是一个*按值传递*的语言，想知道这是什么意思。这意味着当你把一个变量作为参数传递给函数时，Go 总是会复制变量的值。我们来看看。你可以在[Go Playground](https://oreil.ly/yo_rY)上运行这段代码。它还在[第五章代码库](https://oreil.ly/EzU8N)的*sample_code/pass_value_type*目录中。首先，你定义一个简单的结构体：

```go
type person struct {
    age  int
    name string
}
```

接下来，你将编写一个函数，接收一个`int`、一个`string`和一个`person`，并修改它们的值：

```go
func modifyFails(i int, s string, p person) {
    i = i * 2
    s = "Goodbye"
    p.name = "Bob"
}
```

然后，你从`main`中调用这个函数，看看修改是否生效：

```go
func main() {
    p := person{}
    i := 2
    s := "Hello"
    modifyFails(i, s, p)
    fmt.Println(i, s, p)
}
```

如函数名称所示，运行此代码将显示函数不会更改传递给它的参数的值：

```go
2 Hello {0 }
```

我包含了`person`结构体，以表明这不仅适用于基本类型。如果你有 Java、JavaScript、Python 或 Ruby 的编程经验，可能会觉得结构体的行为很奇怪。毕竟，这些语言在将对象作为参数传递给函数时允许修改对象的字段。这种差异的原因是我在讨论指针时会解释的。

对于映射和切片的行为有所不同。让我们看看在函数内部尝试修改它们时会发生什么。你可以在[Go Playground](https://oreil.ly/kKL4R)上运行这段代码。它还在[第五章代码库](https://oreil.ly/EzU8N)的*sample_code/pass_map_slice*目录中。你将编写一个修改映射参数的函数和一个修改切片参数的函数：

```go
func modMap(m map[int]string) {
    m[2] = "hello"
    m[3] = "goodbye"
    delete(m, 1)
}

func modSlice(s []int) {
    for k, v := range s {
        s[k] = v * 2
    }
    s = append(s, 10)
}
```

然后，你从`main`中调用这些函数：

```go
func main() {
    m := map[int]string{
        1: "first",
        2: "second",
    }
    modMap(m)
    fmt.Println(m)

    s := []int{1, 2, 3}
    modSlice(s)
    fmt.Println(s)
}
```

当你运行这段代码时，会看到一些有趣的事情：

```go
map[2:hello 3:goodbye]
[2 4 6]
```

对于映射来说，解释发生的事情很容易：对映射参数的任何更改都会反映在传递到函数中的变量中。对于切片来说，情况就更复杂了。你可以修改切片中的任何元素，但不能扩展切片的长度。这对直接传递给函数的映射和切片以及结构体中的映射和切片字段都适用。

该程序引出了一个问题：为什么映射和切片的行为与其他类型不同？这是因为映射和切片都是用指针实现的。我将在下一章节详细介绍。

###### 小贴士

Go 中的每种类型都是值类型。只是有时候这个值是一个指针。

按值传递是 Go 对常量支持有限的一个原因。由于变量是按值传递的，你可以确保调用函数不会修改传递进去的变量的值（除非变量是切片或映射）。总的来说，这是件好事。当函数不修改其输入参数，而是返回新计算的值时，它使得理解程序中数据流动的过程变得更容易。

虽然这种方法很容易理解，但有时你需要将可变的东西传递给一个函数。那么你该怎么办？这时你就需要一个指针。

# 练习

这些练习测试你对 Go 语言中函数的理解以及如何使用它们的知识。解决方案可在[第五章仓库](https://oreil.ly/EzU8N)的*exercise_solutions*目录中找到。

1.  这个简单的计算器程序没有处理一个错误情况：除以零。更改数学操作的函数签名，以返回一个`int`和一个`error`。在`div`函数中，如果除数是`0`，则返回`errors.New("division by zero")`作为错误。在所有其他情况下，返回`nil`。调整`main`函数以检查此错误。

1.  编写一个名为`fileLen`的函数，该函数具有`string`类型的输入参数，并返回一个`int`和一个`error`。该函数接受一个文件名，并返回文件的字节数。如果读取文件时出错，返回错误。使用`defer`确保文件正确关闭。

1.  编写一个名为`prefixer`的函数，该函数具有`string`类型的输入参数，并返回一个函数，该函数具有`string`类型的输入参数并返回一个`string`类型的返回值。返回的函数应该将其输入以`prefixer`传入的字符串作为前缀。使用以下`main`函数来测试`prefixer`：

    ```go
    func main() {
        helloPrefix := prefixer("Hello")
        fmt.Println(helloPrefix("Bob")) // should print Hello Bob
        fmt.Println(helloPrefix("Maria")) // should print Hello Maria
    }
    ```

# 结语

在本章中，你已经学习了 Go 语言中的函数，它们与其他语言中的函数相似之处以及它们的独特特性。在下一章中，你将学习指针，了解它们并不像许多新的 Go 开发者期望的那样可怕，并学习如何利用它们编写高效的程序。
