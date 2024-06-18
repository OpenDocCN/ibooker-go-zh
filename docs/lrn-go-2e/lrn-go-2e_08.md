# 第八章：泛型

“不要重复你自己”是常见的软件工程建议。重用数据结构或函数比重新创建它更好，因为很难保持重复代码之间的代码变更同步。在像 Go 这样的强类型语言中，必须在编译时知道每个函数参数和每个结构字段的类型。这种严格性使编译器能够帮助验证您的代码是否正确，但有时当您希望重用函数中的逻辑或结构中的字段时，您可能需要不同类型的支持。通过类型参数（俗称泛型），Go 提供了这种功能。在本章中，您将了解为什么人们需要泛型，Go 的泛型实现可以做什么，泛型不能做什么以及如何正确使用它们。

# 泛型减少重复代码并增加类型安全性。

Go 是一种静态类型语言，这意味着在编译代码时会检查变量和参数的类型。内置类型（映射、切片、通道）和函数（如`len`、`cap`或`make`）能够接受和返回不同具体类型的值，但是直到 Go 1.18 之前，用户定义的 Go 类型和函数不能。

如果您习惯于动态类型语言，其中类型直到运行代码时才会被评估，您可能不理解泛型的重要性，也可能对它们的含义有些不清楚。将它们视为“类型参数”会有所帮助。到目前为止，在本书中，您已经看到了接受参数的函数，这些参数的值在调用函数时指定。在“多返回值”中，函数`divAndRemainder`有两个`int`参数并返回两个`int`值：

```go
func divAndRemainder(num, denom int) (int, int, error) {
    if denom == 0 {
        return 0, 0, errors.New("cannot divide by zero")
    }
    return num / denom, num % denom, nil
}
```

类似地，通过在声明结构体时为字段指定类型来创建结构体。在这里，`Node`有一个类型为`int`的字段和另一个类型为`*Node`的字段：

```go
type Node struct {
    val  int
    next *Node
}
```

然而，在某些情况下，编写函数或结构，留下参数或字段的具体*类型*未指定，也是有用的。

泛型类型的优势很容易理解。在“为 nil 实例编写您的方法”中，您看到了一个`int`的二叉树。如果您想要一个字符串或 float64 的二叉树并且需要类型安全，您有几个选择。第一种可能性是为每种类型编写自定义树，但是这样大量重复的代码冗长且容易出错。

没有泛型，避免重复代码的唯一方法将是修改您的树实现，使其使用接口来指定如何排序值。该接口将如下所示：

```go
type Orderable interface {
    // Order returns:
    // a value < 0 when the Orderable is less than the supplied value,
    // a value > 0 when the Orderable is greater than the supplied value,
    // and 0 when the two values are equal.
    Order(any) int
}
```

现在您有了`Orderable`，可以修改您的`Tree`实现以支持它：

```go
type Tree struct {
    val         Orderable
    left, right *Tree
}

func (t *Tree) Insert(val Orderable) *Tree {
    if t == nil {
        return &Tree{val: val}
    }

    switch comp := val.Order(t.val); {
    case comp < 0:
        t.left = t.left.Insert(val)
    case comp > 0:
        t.right = t.right.Insert(val)
    }
    return t
}
```

有了`OrderableInt`类型，您可以插入`int`值：

```go
type OrderableInt int

func (oi OrderableInt) Order(val any) int {
    return int(oi - val.(OrderableInt))
}

func main() {
    var it *Tree
    it = it.Insert(OrderableInt(5))
    it = it.Insert(OrderableInt(3))
    // etc...
}
```

虽然此代码可以正确工作，但它不允许编译器验证插入到数据结构中的值是否都相同。如果您还有一个`OrderableString`类型：

```go
type OrderableString string

func (os OrderableString) Order(val any) int {
    return strings.Compare(string(os), val.(string))
}
```

以下代码编译通过：

```go
var it *Tree
it = it.Insert(OrderableInt(5))
it = it.Insert(OrderableString("nope"))
```

函数 `Order` 使用 `any` 来表示传入的值。这实际上绕过了 Go 的主要优势之一：检查编译时类型安全性。当你编译试图将 `OrderableString` 插入已包含 `OrderableInt` 的 `Tree` 的代码时，编译器接受该代码。然而，程序在运行时会出现恐慌：

```go
panic: interface conversion: interface {} is main.OrderableInt, not string
```

您可以在[第八章存储库](https://oreil.ly/E0Ay8)的 *sample_code/non_generic_tree* 目录中尝试此代码。

有了泛型，可以为多个类型实现数据结构并在编译时检测不兼容的数据。稍后您将看到如何正确使用它们。

没有泛型的数据结构虽然不方便，但真正的限制在于编写函数。Go 标准库中的几个实现决策是因为泛型最初不是该语言的一部分。例如，而不是编写多个处理不同数值类型的函数，Go 使用 `float64` 参数实现了 `math.Max`、`math.Min` 和 `math.Mod` 函数，这些参数的范围足以准确表示几乎所有其他数值类型（除了值大于 2⁵³ – 1 或小于 –2⁵³ – 1 的 `int`、`int64` 或 `uint`）。

没有泛型，一些其他事情是不可能的。您无法创建一个由接口指定的变量的新实例，也无法指定同一接口类型的两个参数也是相同的具体类型。没有泛型，您无法编写一个处理任何类型切片的函数，而不用反射并放弃一些性能以及编译时类型安全性（这就是 `sort.Slice` 的工作原理）。这意味着在泛型引入 Go 之前，操作切片的函数（如 `map`、`reduce` 和 `filter`）将被重复为每种切片类型实现。虽然简单的算法足够简单，但许多（如果不是大多数）软件工程师发现简单复制代码很烦人，只是因为编译器不足够智能而无法自动完成。

# 引入 Go 中的泛型

自从 Go 首次发布以来，人们一直呼吁将泛型添加到该语言中。Go 的开发领导 Russ Cox 在 2009 年写了一篇[博客文章](https://oreil.ly/U4huA)，解释了为什么最初没有包含泛型。Go 强调快速编译器、可读性代码和良好执行时间，但他们所知的所有泛型实现都无法同时满足这三个要求。经过十年的研究，Go 团队提出了一种可行的方法，详见[类型参数提案](https://oreil.ly/31ay7)。

通过查看一个栈来了解泛型在 Go 中的工作原理。如果你没有计算机科学背景，栈是一种数据类型，其中的值按照 LIFO 顺序添加和删除。这就像一堆等待被清洗的盘子；最先放置的在底部，只有通过处理后来添加的才能得到它们。让我们看看如何使用泛型来制作一个栈：

```go
type Stack[T any] struct {
    vals []T
}

func (s *Stack[T]) Push(val T) {
    s.vals = append(s.vals, val)
}

func (s *Stack[T]) Pop() (T, bool) {
    if len(s.vals) == 0 {
        var zero T
        return zero, false
    }
    top := s.vals[len(s.vals)-1]
    s.vals = s.vals[:len(s.vals)-1]
    return top, true
}
```

有几件事需要注意。首先，在类型声明后面你有`[T any]`。类型参数信息放在括号内，有两部分。第一部分是类型参数的名称。你可以为类型参数选择任何名称，但使用大写字母是惯例。第二部分是*类型约束*，它使用 Go 接口指定哪些类型是有效的。如果任何类型都可以使用，可以使用宇宙块标识符`any`来指定，你首次在“空接口不表达任何内容”中看到它。在`Stack`声明内部，你声明`vals`的类型为`[]T`。

接下来，看一下方法声明。就像你在你的`vals`声明中使用了`T`一样，在这里也是一样。你还需要在接收器部分使用`Stack[T]`而不是`Stack`来引用类型。

最后，泛型使得零值处理有些有趣。在`Pop`中，你不能简单地返回`nil`，因为对于值类型（如`int`）来说，这不是有效的值。获取泛型的零值的最简单方法是使用`var`声明一个变量并返回它，因为根据定义，`var`始终将其变量初始化为零值，如果未分配其他值。

使用泛型类型类似于使用非泛型类型：

```go
func main() {
    var intStack Stack[int]
    intStack.Push(10)
    intStack.Push(20)
    intStack.Push(30)
    v, ok := intStack.Pop()
    fmt.Println(v, ok)
}
```

唯一的区别是，在声明变量时，你包括要与你的`Stack`一起使用的类型——在本例中是`int`。如果尝试将字符串推送到栈上，编译器将捕获它。添加以下行：

```go
intStack.Push("nope")
```

会产生编译错误：

```go
cannot use "nope" (untyped string constant) as int value
  in argument to intStack.Push
```

你可以在[Go Playground](https://oreil.ly/9vzHB)上或[第八章代码库](https://oreil.ly/E0Ay8)中的*sample_code/stack*目录中尝试这个泛型栈。

向你的栈添加另一个方法来告诉你栈是否包含一个值：

```go
func (s Stack[T]) Contains(val T) bool {
    for _, v := range s.vals {
        if v == val {
            return true
        }
    }
    return false
}
```

不幸的是，这不会编译。它会产生一个错误：

```go
invalid operation: v == val (type parameter T is not comparable with ==)
```

就像`interface{}`什么也不说一样，`any`也一样。你只能存储`any`类型的值并检索它们。要使用`==`，你需要不同的类型。由于几乎所有 Go 类型都可以使用`==`和`!=`进行比较，所以在宇宙块中定义了一个新的内置接口称为`comparable`。如果将`Stack`的定义更改为使用`comparable`：

```go
type Stack[T comparable] struct {
    vals []T
}
```

然后可以使用你的新方法：

```go
func main() {
    var s Stack[int]
    s.Push(10)
    s.Push(20)
    s.Push(30)
    fmt.Println(s.Contains(10))
    fmt.Println(s.Contains(5))
}
```

这将输出以下内容：

```go
true
false
```

你可以在[Go Playground](https://oreil.ly/Qc4J3)上或[第八章代码库](https://oreil.ly/E0Ay8)中的*sample_code/comparable_stack*目录中尝试这个更新的栈。

稍后，您将看到如何制作一个通用二叉树。首先，我将介绍一些额外的概念：*通用函数*，通用函数如何与接口配合工作，以及*类型术语*。

# 通用函数抽象算法

正如我所示，您也可以编写通用函数。我之前提到没有泛型使得编写适用于所有类型的映射、归约和过滤实现变得困难。泛型使这变得容易。以下是来自类型参数提案的实现：

```go
// Map turns a []T1 to a []T2 using a mapping function.
// This function has two type parameters, T1 and T2.
// This works with slices of any type.
func MapT1, T2 any T2) []T2 {
    r := make([]T2, len(s))
    for i, v := range s {
        r[i] = f(v)
    }
    return r
}

// Reduce reduces a []T1 to a single value using a reduction function.
func ReduceT1, T2 any T2) T2 {
    r := initializer
    for _, v := range s {
        r = f(r, v)
    }
    return r
}

// Filter filters values from a slice using a filter function.
// It returns a new slice with only the elements of s
// for which f returned true.
func FilterT any bool) []T {
    var r []T
    for _, v := range s {
        if f(v) {
            r = append(r, v)
        }
    }
    return r
}
```

函数在变量参数之前将其类型参数放在函数名称后。`Map`和`Reduce`有两个类型参数，都是`any`类型，而`Filter`只有一个。当您运行代码时：

```go
words := []string{"One", "Potato", "Two", "Potato"}
filtered := Filter(words, func(s string) bool {
    return s != "Potato"
})
fmt.Println(filtered)
lengths := Map(filtered, func(s string) int {
    return len(s)
})
fmt.Println(lengths)
sum := Reduce(lengths, 0, func(acc int, val int) int {
    return acc + val
})
fmt.Println(sum)
```

您将得到输出：

```go
[One Two]
[3 3]
6
```

在[The Go Playground](https://oreil.ly/Ahf2b)或[第八章存储库](https://oreil.ly/E0Ay8)中的*sample_code/map_filter_reduce*目录上自行尝试。

# 泛型和接口

您可以使用任何接口作为类型约束，不仅仅是`any`和`comparable`。例如，假设您想制作一个类型，该类型持有任何两个相同类型的值，只要该类型实现了`fmt.Stringer`。泛型使得在编译时强制执行这一点成为可能：

```go
type Pair[T fmt.Stringer] struct {
    Val1 T
    Val2 T
}
```

您还可以创建具有类型参数的接口。例如，这是一个带有方法的接口，该方法与指定类型的值进行比较并返回`float64`。它还嵌入了`fmt.Stringer`：

```go
type Differ[T any] interface {
    fmt.Stringer
    Diff(T) float64
}
```

你将使用这两种类型来创建比较函数。该函数接受两个`Pair`实例，这些实例具有`Differ`类型的字段，并返回较接近数值的`Pair`：

```go
func FindCloser[T Differ[T]](pair1, pair2 Pair[T]) Pair[T] {
    d1 := pair1.Val1.Diff(pair1.Val2)
    d2 := pair2.Val1.Diff(pair2.Val2)
    if d1 < d2 {
        return pair1
    }
    return pair2
}
```

注意，`FindCloser`接受具有符合`Differ`接口的字段的`Pair`实例。`Pair`要求其字段都是相同类型，并且该类型满足`fmt.Stringer`接口；这使得该函数更具选择性。如果`Pair`实例中的字段不满足`Differ`，编译器将阻止您将其与`FindCloser`一起使用。

现在定义满足`Differ`接口的一对类型：

```go
type Point2D struct {
    X, Y int
}

func (p2 Point2D) String() string {
    return fmt.Sprintf("{%d,%d}", p2.X, p2.Y)
}

func (p2 Point2D) Diff(from Point2D) float64 {
    x := p2.X - from.X
    y := p2.Y - from.Y
    return math.Sqrt(float64(x*x) + float64(y*y))
}

type Point3D struct {
    X, Y, Z int
}

func (p3 Point3D) String() string {
    return fmt.Sprintf("{%d,%d,%d}", p3.X, p3.Y, p3.Z)
}

func (p3 Point3D) Diff(from Point3D) float64 {
    x := p3.X - from.X
    y := p3.Y - from.Y
    z := p3.Z - from.Z
    return math.Sqrt(float64(x*x) + float64(y*y) + float64(z*z))
}
```

这是使用此代码的样子：

```go
func main() {
    pair2Da := Pair[Point2D]{Point2D{1, 1}, Point2D{5, 5}}
    pair2Db := Pair[Point2D]{Point2D{10, 10}, Point2D{15, 5}}
    closer := FindCloser(pair2Da, pair2Db)
    fmt.Println(closer)

    pair3Da := Pair[Point3D]{Point3D{1, 1, 10}, Point3D{5, 5, 0}}
    pair3Db := Pair[Point3D]{Point3D{10, 10, 10}, Point3D{11, 5, 0}}
    closer2 := FindCloser(pair3Da, pair3Db)
    fmt.Println(closer2)
}
```

在[The Go Playground](https://oreil.ly/rnKj9)或[第八章存储库](https://oreil.ly/E0Ay8)中的*sample_code/generic_interface*目录上自行运行它。

# 使用类型术语指定运算符

泛型还需要表示运算符。`divAndRemainder`函数在`int`上运行良好，但在其他整数类型上使用需要类型转换，而`uint`允许您表示`int`无法处理的值。如果要编写`divAndRemainder`的通用版本，您需要一种指定可以使用`/`和`%`的方式。Go 泛型通过*类型元素*实现这一点，该元素由一个或多个*类型术语*组成在接口内：

```go
type Integer interface {
    int | int8 | int16 | int32 | int64 |
        uint | uint8 | uint16 | uint32 | uint64 | uintptr
}
```

在 “嵌入和接口” 中，您学习了如何嵌入接口以指示包含接口的方法集包括嵌入接口的方法。类型元素指定可以分配给类型参数的类型和支持的操作符。它们列出了由 `|` 分隔的具体类型。允许的操作符是所有列出类型都有效的那些操作符。模数 (`%`) 操作符仅适用于整数，因此我们列出了所有整数类型（可以省略 `byte` 和 `rune`，因为它们分别是 `uint8` 和 `int32` 的类型别名）。

请注意，具有类型元素的接口仅作为类型约束有效。将它们用作变量、字段、返回值或参数的类型会导致编译时错误。

现在您可以编写您自己的 `divAndRemainder` 的通用版本，并将其与内置的 `uint` 类型（或 `Integer` 中列出的任何其他类型）一起使用。

```go
func divAndRemainderT Integer (T, T, error) {
    if denom == 0 {
        return 0, 0, errors.New("cannot divide by zero")
    }
    return num / denom, num % denom, nil
}

func main() {
    var a uint = 18_446_744_073_709_551_615
    var b uint = 9_223_372_036_854_775_808
    fmt.Println(divAndRemainder(a, b))
}
```

默认情况下，类型术语精确匹配。如果您尝试使用 `divAndRemainder` 与其底层类型为 `Integer` 中列出的类型之一的用户定义类型，将会收到错误。这段代码：

```go
type MyInt int
var myA MyInt = 10
var myB MyInt = 20
fmt.Println(divAndRemainder(myA, myB))
```

会产生以下错误：

```go
MyInt does not satisfy Integer (possibly missing ~ for int in Integer)
```

错误文本提供了解决此问题的提示。如果您希望一个类型术语对任何具有该类型术语作为其底层类型的类型有效，可以在类型术语前加上 `~`。这将修改 `Integer` 的定义如下：

```go
type Integer interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
        ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr
}
```

您可以在 [The Go Playground](https://oreil.ly/OLd32) 上或在 [第八章存储库](https://oreil.ly/E0Ay8) 的 *sample_code/type_terms* 目录中查看 `divAndRemainder` 函数的通用版本。

添加类型术语使您可以定义一种类型，使您可以编写通用比较函数：

```go
type Ordered interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
        ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
        ~float32 | ~float64 |
        ~string
}
```

`Ordered` 接口列出了所有支持 `==`, `!=`, `<`, `>`, `<=`, 和 `>=` 操作符的类型。有一种方法可以指定一个变量表示一个可排序类型是如此有用，因此 Go 1.21 添加了 [`cmp` 包](https://oreil.ly/nXRdO)，该包定义了这个 `Ordered` 接口。该包还定义了两个比较函数。`Compare` 函数返回 -1、0 或 1，具体取决于其第一个参数是否小于、等于或大于其第二个参数，而 `Less` 函数在其第一个参数小于其第二个参数时返回 `true`。

在用于类型参数的接口中，同时具有类型元素和方法元素是合法的。例如，您可以指定一个类型必须具有 `int` 的底层类型和一个 `String() string` 方法：

```go
type PrintableInt interface {
    ~int
    String() string
}
```

请注意，Go 语言允许您声明一个类型参数接口，但实际上无法实例化。如果在`PrintableInt`中使用`int`而不是`~int`，则没有任何有效的类型可以满足它，因为`int`没有方法。这可能看起来很糟糕，但编译器仍会帮助您。如果您声明了一个带有不可能类型参数的类型或函数，任何尝试使用它都会导致编译器错误。假设您声明了这些类型：

```go
type ImpossiblePrintableInt interface {
    int
    String() string
}

type ImpossibleStruct[T ImpossiblePrintableInt] struct {
    val T
}

type MyInt int

func (mi MyInt) String() string {
    return fmt.Sprint(mi)
}
```

即使您无法实例化`ImpossibleStruct`，编译器对所有这些声明也没有任何问题。但是，一旦尝试使用`ImpossibleStruct`，编译器就会抱怨。此代码：

```go
s := ImpossibleStruct[int]{10}
s2 := ImpossibleStruct[MyInt]{10}
```

会产生以下编译时错误：

```go
int does not implement ImpossiblePrintableInt (missing String method)
MyInt does not implement ImpossiblePrintableInt (possibly missing ~ for
int in constraint ImpossiblePrintableInt)
```

在[Go Playground](https://oreil.ly/eFx6t)或[第八章存储库](https://oreil.ly/E0Ay8)的*sample_code/impossible*目录中尝试这个。

除了内置的基本类型之外，类型项还可以是切片、映射、数组、通道、结构体，甚至函数。当您想要确保类型参数具有特定的基础类型和一个或多个方法时，它们非常有用。

# 类型推断和泛型

就像在使用`:=`运算符时 Go 支持类型推断一样，在调用泛型函数时也支持类型推断以简化调用。您可以在之前对`Map`、`Filter`和`Reduce`的调用中看到这一点。在某些情况下，类型推断是不可能的（例如，当类型参数仅用作返回值时）。发生这种情况时，必须指定所有类型参数。下面是一个稍微愚蠢的代码片段，演示了类型推断不起作用的情况：

```go
type Integer interface {
    int | int8 | int16 | int32 | int64 | uint | uint8 | uint16 | uint32 | uint64
}

func ConvertT1, T2 Integer T2 {
    return T2(in)
}

func main() {
    var a int = 10
    b := Convertint, int64 // can't infer the return type
    fmt.Println(b)
}
```

在[Go Playground](https://oreil.ly/tWsu3)或[第八章存储库](https://oreil.ly/E0Ay8)的*sample_code/type_inference*目录中尝试它。

# 类型元素限制常量

类型元素还指定了可以分配给泛型类型变量的常量。像运算符一样，这些常量需要对类型元素中的所有类型项有效。在`Ordered`中没有可以分配给每个列出的类型的常量，因此您不能将常量分配给该泛型类型的变量。如果使用`Integer`接口，则以下代码不会编译，因为您不能将值 1,000 分配给 8 位整数：

```go
// INVALID!
func PlusOneThousandT Integer T {
    return in + 1_000
}
```

然而，这是有效的：

```go
// VALID
func PlusOneHundredT Integer T {
    return in + 100
}
```

# 将泛型函数与泛型数据结构结合起来

让我们返回二叉树示例，并看看如何将您学到的所有内容结合起来，使得单个树适用于任何具体类型。

关键是要意识到，您的树需要的是一个单一的泛型函数，它比较两个值并告诉您它们的顺序：

```go
type OrderableFunc [T any] func(t1, t2 T) int
```

现在您有了`OrderableFunc`，可以稍微修改树的实现。首先，您将其拆分为两种类型，`Tree`和`Node`：

```go
type Tree[T any] struct {
    f    OrderableFunc[T]
    root *Node[T]
}

type Node[T any] struct {
    val         T
    left, right *Node[T]
}
```

使用构造函数构建一个新的`Tree`：

```go
func NewTreeT any *Tree[T] {
    return &Tree[T]{
        f: f,
    }
}
```

`Tree`的方法非常简单，因为它们只是调用`Node`来完成所有真正的工作。

```go
func (t *Tree[T]) Add(v T) {
    t.root = t.root.Add(t.f, v)
}

func (t *Tree[T]) Contains(v T) bool {
    return t.root.Contains(t.f, v)
}
```

`Node`上的`Add`和`Contains`方法与之前看到的非常相似。唯一的区别在于，你正在使用的函数用于排序元素的方式是通过参数传递的：

```go
func (n *Node[T]) Add(f OrderableFunc[T], v T) *Node[T] {
    if n == nil {
        return &Node[T]{val: v}
    }
    switch r := f(v, n.val); {
    case r <= -1:
        n.left = n.left.Add(f, v)
    case r >= 1:
        n.right = n.right.Add(f, v)
    }
    return n
}

func (n *Node[T]) Contains(f OrderableFunc[T], v T) bool {
    if n == nil {
        return false
    }
    switch r := f(v, n.val); {
    case r <= -1:
        return n.left.Contains(f, v)
    case r >= 1:
        return n.right.Contains(f, v)
    }
    return true
}
```

现在你需要一个函数来匹配`OrderedFunc`的定义。幸运的是，你已经见过一个：`cmp`包中的`Compare`。当你在`Tree`中使用它时，看起来像这样：

```go
t1 := NewTree(cmp.Compare[int])
t1.Add(10)
t1.Add(30)
t1.Add(15)
fmt.Println(t1.Contains(15))
fmt.Println(t1.Contains(40))
```

对于结构体，你有两个选项。你可以写一个函数：

```go
type Person struct {
    Name string
    Age int
}

func OrderPeople(p1, p2 Person) int {
    out := cmp.Compare(p1.Name, p2.Name)
    if out == 0 {
        out = cmp.Compare(p1.Age, p2.Age)
    }
    return out
}
```

然后，在创建树时，你可以将该函数传递进去：

```go
t2 := NewTree(OrderPeople)
t2.Add(Person{"Bob", 30})
t2.Add(Person{"Maria", 35})
t2.Add(Person{"Bob", 50})
fmt.Println(t2.Contains(Person{"Bob", 30}))
fmt.Println(t2.Contains(Person{"Fred", 25}))
```

你也可以不使用函数，而是为`NewTree`提供一个方法。正如我在“方法也是函数”中所说的，你可以使用方法表达式将方法视为函数。我们在这里就这样做。首先，编写这个方法：

```go
func (p Person) Order(other Person) int {
    out := cmp.Compare(p.Name, other.Name)
    if out == 0 {
        out = cmp.Compare(p.Age, other.Age)
    }
    return out
}
```

然后使用它：

```go
t3 := NewTree(Person.Order)
t3.Add(Person{"Bob", 30})
t3.Add(Person{"Maria", 35})
t3.Add(Person{"Bob", 50})
fmt.Println(t3.Contains(Person{"Bob", 30}))
fmt.Println(t3.Contains(Person{"Fred", 25}))
```

你可以在[The Go Playground](https://oreil.ly/-tus2)找到此树的代码，或者在[第八章仓库](https://oreil.ly/E0Ay8)的*sample_code/generic_tree*目录中找到。

# 更多关于可比较性的内容

正如你在“接口可比较”中看到的那样，接口是 Go 语言中可比较的类型之一。这意味着在使用`==`和`!=`操作符时需要小心，如果接口的底层类型不可比较，你的代码会在运行时抛出异常。

当使用泛型与`comparable`接口时，这个坑仍然存在。假设你已经定义了一个接口和一些实现：

```go
type Thinger interface {
    Thing()
}

type ThingerInt int

func (t ThingerInt) Thing() {
    fmt.Println("ThingInt:", t)
}

type ThingerSlice []int

func (t ThingerSlice) Thing() {
    fmt.Println("ThingSlice:", t)
}
```

你还定义了一个通用函数，只接受`comparable`的值：

```go
func ComparerT comparable {
    if t1 == t2 {
        fmt.Println("equal!")
    }
}
```

用`int`或`ThingerInt`类型的变量调用此函数是合法的。

```go
var a int = 10
var b int = 10
Comparer(a, b) // prints true

var a2 ThingerInt = 20
var b2 ThingerInt = 20
Comparer(a2, b2) // prints true
```

编译器不允许你使用`ThingerSlice`（或`[]int`）类型的变量调用此函数：

```go
var a3 ThingerSlice = []int{1, 2, 3}
var b3 ThingerSlice = []int{1, 2, 3}
Comparer(a3, b3) // compile fails: "ThingerSlice does not satisfy comparable"
```

然而，用`Thinger`类型的变量调用它是完全合法的。如果使用`ThingerInt`，代码编译并按预期工作：

```go
var a4 Thinger = a2
var b4 Thinger = b2
Comparer(a4, b4) // prints true
```

但是你也可以将`ThingerSlice`分配给`Thinger`类型的变量。这就是问题所在：

```go
a4 = a3
b4 = b3
Comparer(a4, b4) // compiles, panics at runtime
```

编译器不会阻止你构建此代码，但如果运行它，你的程序会因为`panic: runtime error: comparing uncomparable type main.ThingerSlice`而崩溃（详见“panic 和 recover”获取更多信息）。你可以在[The Go Playground](https://oreil.ly/NVIA4)或者[第八章仓库](https://oreil.ly/E0Ay8)的*sample_code/more_comparable*目录中自己尝试这段代码。

想要了解有关可比较类型与泛型交互以及为何做出此设计决策的更多技术细节，请阅读 Go 团队成员 Robert Griesemer 的博文[“All Your Comparable Types”](https://oreil.ly/AsWs-)。

# 遗漏的事情

Go 保持一种小而集中的语言风格，而 Go 的泛型实现并未包含许多其他语言泛型实现中存在的功能。本节描述了 Go 泛型初始实现中缺少的一些功能。

虽然你可以构建一个适用于用户定义和内置类型的单一树，像 Python、Ruby 和 C++ 这样的语言通过不同的方式解决了这个问题。它们包括*运算符重载*，允许用户定义的类型指定操作符的实现。Go 将不会添加这个特性。这意味着你不能使用`range`来迭代用户定义的容器类型，也不能使用`[]`来对它们进行索引。

不使用运算符重载有很多理由。首先，Go 有惊人数量的运算符。其次，Go 也没有函数或方法重载，你需要一种方式来为不同的类型指定不同的操作符功能。此外，运算符重载可能导致代码难以理解，因为开发人员会为符号创造聪明的含义（在 C++ 中，`<<`对某些类型意味着“左移位”，对其他类型意味着“将右侧的值写入左侧的值”）。这些是 Go 试图避免的可读性问题。

另一个有用的特性被初始 Go 泛型实现忽略，即方法上的额外类型参数。回顾`Map/Reduce/Filter`函数，你可能会认为它们作为方法会很有用，就像这样：

```go
type functionalSlice[T any] []T

// THIS DOES NOT WORK
func (fs functionalSlice[T]) MapE any E) functionalSlice[E] {
    out := make(functionalSlice[E], len(fs))
    for i, v := range fs {
        out[i] = f(v)
    }
    return out
}

// THIS DOES NOT WORK
func (fs functionalSlice[T]) ReduceE any E) E {
    out := start
    for _, v := range fs {
        out = f(out, v)
    }
    return out
}
```

你可以像这样使用它：

```go
var numStrings = functionalSlice[string]{"1", "2", "3"}
sum := numStrings.Map(func(s string) int {
    v, _ := strconv.Atoi(s)
    return v
}).Reduce(0, func(acc int, cur int) int {
    return acc + cur
})
```

不幸的是，对于函数式编程的爱好者来说，这并不适用。与其将方法调用链在一起，你需要嵌套函数调用或者采用更可读的方法，逐个调用函数并将中间值赋给变量。类型参数提案详细说明了排除参数化方法的原因。

Go 也没有可变类型参数。如 “可变输入参数和切片” 中所讨论的那样，要实现一个接受不同数量参数的函数，你需要指定最后一个参数类型以 `...` 开头。例如，没有办法为这些可变参数指定某种类型模式，比如交替的`string`和`int`。所有可变变量必须匹配一个声明的单一类型，可以是泛型或非泛型的。

Go 泛型中省略的其他特性更为奥秘。包括以下内容：

特化

一个函数或方法可以在泛型版本之外重载为一个或多个特定于类型的版本。由于 Go 没有重载，这个特性不在考虑之列。

柯里化

允许你根据另一个泛型函数或类型部分实例化一个函数或类型，指定一些类型参数。

元编程

允许你指定在编译时运行的代码，以生成在运行时运行的代码。

# Go 的惯用方式和泛型

明显地，向 Go 中添加泛型改变了某些使用 Go 习惯的建议。使用 `float64` 来表示任何数值类型将不再适用。在数据结构或函数参数中，应该使用 `any` 而不是 `interface{}` 表示未指定类型。可以使用单个函数处理不同的切片类型。但不必立即将所有旧代码转换为使用类型参数。随着新的设计模式的发明和完善，你的旧代码仍将正常运行。

目前来看，判断泛型对性能的长期影响为时尚早。截至目前，编译时间没有影响。Go 1.18 编译器比以前的版本慢，但 Go 1.20 的编译器解决了这个问题。

也有一些关于泛型运行时影响的研究。Vicent Marti 写了一篇[详细博文](https://oreil.ly/YK4HT)，探讨了泛型导致代码变慢的案例和解释其实现细节。相反，Eli Bendersky 写了一篇[博文](https://oreil.ly/2Mqms)，展示了泛型使排序算法更快的情况。

特别是，不要为了提高性能而将具有接口参数的函数更改为具有泛型类型参数的函数。例如，将这个简单的函数：

```go
type Ager interface {
    age() int
}

func doubleAge(a Ager) int {
    return a.age() * 2
}
```

转换为：

```go
func doubleAgeGenericT Ager int {
    return a.age() * 2
}
```

在 Go 1.20 中，函数调用慢了约 30%。（对于非平凡函数，性能差异不显著。）你可以使用[第八章仓库](https://oreil.ly/E0Ay8)中 *sample_code/perf* 目录下的代码运行基准测试。

对于其他语言中有泛型经验的开发人员，这可能会令人感到惊讶。例如，在 C++ 中，编译器使用泛型在抽象数据类型上执行正常的运行时操作（确定使用的具体类型），将其转换为编译时操作，为每个具体类型生成唯一的函数。这使得生成的二进制文件更大但更快。正如 Vicent 在他的博客文章中解释的那样，当前的 Go 编译器仅为不同的基础类型生成唯一的函数。此外，所有指针类型共享单个生成的函数。为了区分传递给共享生成函数的不同类型，编译器添加额外的运行时查找。这导致性能下降。

同样地，随着 Go 中泛型实现的成熟，预计运行时性能会改善。与往常一样，目标是编写可维护的程序，其速度足以满足您的需求。使用“使用基准测试”中讨论的基准测试和性能分析工具来衡量和改进您的代码。

# 向标准库添加泛型

Go 1.18 版本中泛型的初始发布非常保守。它在 universe 块中添加了新的接口`any`和`comparable`，但标准库中没有发生任何 API 更改来支持泛型。已进行了风格上的更改；标准库中几乎所有使用`interface{}`的地方都被替换为`any`。

随着 Go 社区对泛型的适应越来越舒适，我们开始看到更多变化。从 Go 1.21 开始，标准库包括使用泛型来实现切片、映射和并发常见算法的函数。在第三章中，我介绍了`slices`和`maps`包中的`Equal`和`EqualFunc`函数。这些包中的其他函数简化了切片和映射的操作。`slices`包中的`Insert`、`Delete`和`DeleteFunc`函数允许开发人员避免编写一些非常棘手的切片处理代码。`maps.Clone`函数利用 Go 运行时提供了创建映射的浅复制的更快方法。在“仅运行代码一次”中，您将了解到`sync.OnceValue`和`sync.OnceValues`，它们使用泛型构建仅调用一次并返回一个或两个值的函数。建议使用这些包中的函数而不是编写自己的实现。标准库的未来版本可能会包含利用泛型的其他新函数和类型。

# 未来解锁的功能

泛型可能成为其他未来功能的基础。一个可能性是*总和类型*。正如类型元素用于指定可以替换为类型参数的类型一样，它们也可以用于变量参数中的接口。这将启用一些有趣的功能。今天，Go 在 JSON 中存在一个常见情况的问题：一个字段可以是单个值或值列表。即使有了泛型，处理这种情况的唯一方法仍然是使用类型为`any`的字段。添加总和类型将允许您创建一个接口，指定一个字段可以是字符串、字符串切片，以及其他内容。然后，类型开关可以完全枚举每个有效类型，提高类型安全性。这种指定一组有界类型的能力允许许多现代语言（包括 Rust 和 Swift）使用总和类型来表示枚举。鉴于 Go 当前枚举功能的弱点，这可能是一个吸引人的解决方案，但需要时间来评估和探索这些想法。

# 练习

现在你已经看到了泛型的工作原理，将其应用于解决以下问题。解决方案位于[第八章代码库](https://oreil.ly/E0Ay8)的*exercise_solutions*目录中。

1.  编写一个通用函数，用于将传入的任何整数或浮点数的值加倍。定义所需的通用接口。

1.  定义一个名为 `Printable` 的泛型接口，该接口匹配实现了 `fmt.Stringer` 并且底层类型为 `int` 或 `float64` 的类型。定义满足此接口的类型。编写一个函数，接受一个 `Printable` 并将其值使用 `fmt.Println` 打印到屏幕上。

1.  编写一个泛型的单链表数据类型。每个元素可以存储一个可比较的值，并且有一个指向列表中下一个元素的指针。要实现的方法如下：

    ```go
    // adds a new element to the end of the linked list
    Add(T)
    // adds an element at the specified position in the linked list
    Insert(T, int)
    // returns the position of the supplied value, -1 if it's not present
    Index (T) int
    ```

# 结束语

在本章中，你已经了解了泛型以及如何使用它们来简化你的代码。对于 Go 语言而言，泛型仍处于早期阶段。看到它们如何帮助语言发展，同时仍保持 Go 语言特有的精神，这将是令人兴奋的。

在接下来的章节中，你将学习如何正确地使用 Go 语言中最具争议的特性之一：errors。
