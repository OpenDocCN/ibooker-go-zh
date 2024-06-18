# 第七章\. 类型、方法和接口

正如你在前面的章节中看到的，Go 是一种静态类型语言，具有内置类型和用户定义类型。像大多数现代语言一样，Go 允许你为类型附加方法。它还具有类型抽象，允许你编写调用方法的代码，而不需显式指定实现方式。

然而，Go 对方法、接口和类型的处理方式与今天大多数其他常用语言非常不同。Go 设计旨在鼓励软件工程师提倡的最佳实践，避免继承，而鼓励组合。在本章中，你将了解类型、方法和接口，并看看如何使用它们构建可测试和可维护的程序。

# Go 中的类型

回到“结构体”，你看到了如何定义结构体类型：

```go
type Person struct {
    FirstName string
    LastName  string
    Age       int
}
```

这应该被理解为声明一个名为 `Person` 的用户定义类型，并具有随后的结构文字的 *基础类型*。除了结构文字，你还可以使用任何原始类型或复合类型文字来定义具体类型。以下是一些例子：

```go
type Score int
type Converter func(string)Score
type TeamScores map[string]Score
```

Go 允许你在任何代码块级别声明类型，从包级块到更低级别。然而，你只能在其作用域内访问该类型。唯一的例外是从其他包导出的类型。关于这些，我会在第十章详细讨论。

###### 注意

为了更容易讨论类型，我将定义几个术语。*抽象类型* 是指规定类型应该做什么而不是如何做到的类型。*具体类型* 指定了什么和如何。这意味着类型有指定的存储数据的方法，并提供了在类型上声明的任何方法的实现。虽然 Go 中的所有类型都是抽象或具体的，但某些语言允许混合类型，如 Java 中具有默认方法的抽象类或接口。

# 方法

像大多数现代语言一样，Go 支持用户定义类型的方法。

一种类型的方法在包级别块中定义：

```go
type Person struct {
    FirstName string
    LastName  string
    Age       int
}

func (p Person) String() string {
    return fmt.Sprintf("%s %s, age %d", p.FirstName, p.LastName, p.Age)
}
```

方法声明看起来像函数声明，但有一个额外的 *接收器* 规范。接收器出现在关键字 `func` 和方法名称之间。像所有其他变量声明一样，接收器名称出现在类型之前。按照惯例，接收器名称通常是类型名称的简短缩写，通常是其首字母。使用 `this` 或 `self` 是非惯用法的。

声明方法和函数之间有一个关键区别：方法只能在包级别块中定义，而函数可以在任何块中定义。

就像函数一样，方法名不能被重载。您可以为不同的类型使用相同的方法名，但不能在同一类型的两个不同方法上使用相同的方法名。尽管从具有方法重载的语言过来时，这种哲学感觉有限制，但不重用名称是 Go 的一部分，使得代码的作用更加明确。

我将在第十章中详细讨论包，但请注意，方法必须在与其关联类型相同的包中声明；Go 不允许您向不受您控制的类型添加方法。虽然您可以在同一包中类型声明的不同文件中定义方法，但最好将类型定义及其关联的方法放在一起，以便易于理解实现。

方法调用对于那些在其他语言中使用方法的人应该看起来很熟悉：

```go
p := Person{
    FirstName: "Fred",
    LastName:  "Fredson",
    Age:       52,
}
output := p.String()
```

## 指针接收者和值接收者

正如我在第六章中所述，Go 使用指针类型的参数来表示函数可能修改参数。方法接收者也适用相同的规则。它们可以是*指针接收者*（类型是指针）或*值接收者*（类型是值类型）。以下规则帮助您确定何时使用每种类型的接收者：

+   如果您的方法修改接收者，您*必须*使用指针接收者。

+   如果您的方法需要处理`nil`实例（参见“为 nil 实例编写方法”），那么它*必须*使用指针接收者。

+   如果您的方法不修改接收者，您*可以*使用值接收者。

如果一个方法没有修改接收者，是否使用值接收者取决于类型声明中的其他方法。当一个类型有*任何*指针接收者方法时，一个常见的做法是保持一致，即使对于*所有*不修改接收者的方法也使用指针接收者。

这里是一些简单的代码，演示指针和值接收者。它从一个类型开始，该类型有两个方法，一个使用值接收者，另一个使用指针接收者：

```go
type Counter struct {
    total       int
    lastUpdated time.Time
}

func (c *Counter) Increment() {
    c.total++
    c.lastUpdated = time.Now()
}

func (c Counter) String() string {
    return fmt.Sprintf("total: %d, last updated: %v", c.total, c.lastUpdated)
}
```

然后，您可以尝试使用以下代码来测试这些方法。您可以在[Go Playground](https://oreil.ly/aqY0i)上运行它，或者使用[第七章代码库中的*sample_code/pointer_value*目录](https://oreil.ly/qJQgV)中的代码：

```go
var c Counter
fmt.Println(c.String())
c.Increment()
fmt.Println(c.String())
```

您应该看到以下输出：

```go
total: 0, last updated: 0001-01-01 00:00:00 +0000 UTC
total: 1, last updated: 2009-11-10 23:00:00 +0000 UTC m=+0.000000001
```

您可能注意到的一件事是，即使`c`是值类型，您仍然可以调用指针接收者方法。当您在本地变量（值类型）上使用指针接收者时，Go 在调用方法时会自动取本地变量的地址。在这种情况下，`c.Increment()` 被转换为 `(&c).Increment()`。

如果在指针变量上调用值接收者，Go 在调用方法时会自动解引用指针。在代码中：

```go
c := &Counter{}
fmt.Println(c.String())
c.Increment()
fmt.Println(c.String())
```

调用 `c.String()` 会被悄悄地转换为 `(*c).String()`。

###### 警告

如果您使用空指针实例调用值接收器方法，您的代码将编译通过，但在运行时会引发恐慌（我在“恐慌和恢复”中讨论了恐慌）。

请注意，仍然适用将值传递给函数的规则。如果您将值类型传递给函数并在传递的值上调用指针接收器方法，则是在对*副本*调用该方法。您可以在[The Go Playground](https://oreil.ly/bGdDi)上尝试以下代码，或使用[第七章存储库](https://oreil.ly/qJQgV)中的*sample_code/update_wrong*目录中的代码：

```go
func doUpdateWrong(c Counter) {
    c.Increment()
    fmt.Println("in doUpdateWrong:", c.String())
}

func doUpdateRight(c *Counter) {
    c.Increment()
    fmt.Println("in doUpdateRight:", c.String())
}

func main() {
    var c Counter
    doUpdateWrong(c)
    fmt.Println("in main:", c.String())
    doUpdateRight(&c)
    fmt.Println("in main:", c.String())
}
```

运行此代码时，您将得到以下输出：

```go
in doUpdateWrong: total: 1, last updated: 2009-11-10 23:00:00 +0000 UTC
    m=+0.000000001
in main: total: 0, last updated: 0001-01-01 00:00:00 +0000 UTC
in doUpdateRight: total: 1, last updated: 2009-11-10 23:00:00 +0000 UTC
    m=+0.000000001
in main: total: 1, last updated: 2009-11-10 23:00:00 +0000 UTC m=+0.000000001
```

`doUpdateRight`中的参数类型是`*Counter`，它是一个指针实例。如您所见，您可以在其上同时调用`Increment`和`String`。对于指针实例，Go 认为指针和值接收器方法都在*方法集*中。对于值实例，只有值接收器方法在方法集中。这看起来现在可能是一个琐碎的细节，但我将在接下来讨论接口时再回到它。

###### 注意

对于新手 Go 程序员（实际上，即使对于不那么新的 Go 程序员），这可能会有些混淆，但是 Go 对指针类型到值类型以及反之间的自动转换只是一种语法糖。这与方法集概念是独立的。Alexey Gronskiy 在[博客文章](https://oreil.ly/i7P5_)中详细探讨了为什么指针实例的方法集包括指针和值接收器方法，但值实例的方法集只包括值接收器方法。

最后一条注意事项：除非需要满足接口的要求，否则不要为 Go 结构体编写 getter 和 setter 方法（我将在“接口快速入门课程”中开始介绍接口）。Go 鼓励您直接访问字段。保留方法用于业务逻辑。唯一的例外是当您需要将多个字段作为单个操作更新或更新不是简单的新值分配时。之前定义的`Increment`方法展示了这两个属性。

## 为`nil`实例编写您的方法

前面的部分涵盖了指针接收器，这可能会让您想知道在调用`nil`实例上的方法时会发生什么。在大多数语言中，这会产生某种错误。（Objective-C 允许您在`nil`实例上调用方法，但它总是什么都不做。）

Go 有些不同之处。它实际上尝试调用该方法。如前所述，如果它是一个值接收器方法，您将会得到一个恐慌，因为指针没有指向任何值。如果它是一个指针接收器方法，如果该方法编写得能够处理`nil`实例的可能性，它就能正常工作。

在某些情况下，期望一个`nil`接收器可以使代码更简单。这里有一个利用`nil`接收器值的二叉树实现：

```go
type IntTree struct {
    val         int
    left, right *IntTree
}

func (it *IntTree) Insert(val int) *IntTree {
    if it == nil {
        return &IntTree{val: val}
    }
    if val < it.val {
        it.left = it.left.Insert(val)
    } else if val > it.val {
        it.right = it.right.Insert(val)
    }
    return it
}

func (it *IntTree) Contains(val int) bool {
    switch {
    case it == nil:
        return false
    case val < it.val:
        return it.left.Contains(val)
    case val > it.val:
        return it.right.Contains(val)
    default:
        return true
    }
}
```

###### 注意

`Contains` 方法不修改`*IntTree`，但是它是用指针接收器声明的。这展示了之前提到的关于支持`nil`接收器的规则。具有值接收器的方法不能检查`nil`，并且如前所述，如果使用`nil`接收器调用，它会引发恐慌。

下面的代码使用了这棵树。你可以在[Go Playground](https://oreil.ly/-F2i-)上试一下，或者使用[第七章代码库](https://oreil.ly/qJQgV)中的*sample_code/tree*目录中的代码：

```go
func main() {
    var it *IntTree
    it = it.Insert(5)
    it = it.Insert(3)
    it = it.Insert(10)
    it = it.Insert(2)
    fmt.Println(it.Contains(2))  // true
    fmt.Println(it.Contains(12)) // false
}
```

Go 允许你在`nil`接收器上调用方法非常聪明，而且在像前面的树节点示例中这样的情况下非常有用。然而，大多数情况下它并不是很有用。指针接收器的工作原理与指针函数参数相似；传递给方法的是指针的副本。就像将`nil`参数传递给函数一样，如果更改指针的副本，则并未更改原始指针。这意味着你不能编写一个处理`nil`并使原始指针非`nil`的指针接收器方法。

如果你的方法有一个指针接收器，并且对于`nil`接收器不起作用，你必须决定你的方法应该如何处理`nil`接收器。一种选择是将其视为致命缺陷，就像尝试访问切片超出其长度的位置一样。在这种情况下，什么都不做，让代码发生恐慌。（同时确保编写良好的测试，如第 15 章中所讨论的。）如果`nil`接收器是可恢复的内容，检查`nil`并返回一个错误（我在第 9 章中讨论错误）。

## 方法也是函数

Go 中的方法与函数非常相似，你可以在任何需要函数类型的变量或参数的地方使用方法作为函数的替代品。

让我们从这个简单类型开始：

```go
type Adder struct {
    start int
}

func (a Adder) AddTo(val int) int {
    return a.start + val
}
```

你可以以通常的方式创建类型的实例并调用其方法：

```go
myAdder := Adder{start: 10}
fmt.Println(myAdder.AddTo(5)) // prints 15
```

你还可以将方法分配给变量或将其传递给`func(int)int`类型的参数。这称为*方法值*：

```go
f1 := myAdder.AddTo
fmt.Println(f1(10))           // prints 20
```

方法值有点像闭包，因为它可以访问创建它的实例字段中的值。

你还可以从类型本身创建一个函数。这称为*方法表达式*：

```go
f2 := Adder.AddTo
fmt.Println(f2(myAdder, 15))  // prints 25
```

在方法表达式中，第一个参数是方法的接收器；函数签名是`func(Adder, int) int`。

方法值和方法表达式并不是什么聪明的边界情况。当你查看“隐式接口使依赖注入更容易”时，你会看到如何使用它们之一。

## 函数与方法

由于你可以将方法用作函数，你可能会想知道何时应该声明函数，何时应该使用方法。

区分因素是您的函数是否依赖于其他数据。正如我多次提到的那样，包级别的状态应该是有效不可变的。每当您的逻辑依赖于在启动时配置或在程序运行时更改的值时，这些值应存储在一个结构体中，并且该逻辑应作为方法实现。如果您的逻辑仅依赖于输入参数，则应该是一个函数。

## 类型声明不等同于继承

除了基于内置 Go 类型和结构字面量声明类型之外，您还可以基于其他用户定义的类型声明用户定义的类型：

```go
type HighScore Score
type Employee Person
```

许多概念都可以被视为“面向对象的”，但一个概念尤为突出：*继承*。通过继承，声明了父类型的状态和方法可以在子类型上使用，并且子类型的值可以替代父类型的值¹。

声明一个基于另一种类型的类型看起来有点像继承，但实际上并不是。这两种类型具有相同的基础类型，但仅此而已。这些类型之间没有层次结构。在具有继承的语言中，子实例可以在任何父实例可以使用的地方使用。子实例还具有父实例的所有方法和数据结构。但在 Go 语言中并非如此。你不能将`HighScore`类型的实例赋给`Score`类型的变量，反之亦然，除非进行类型转换。也不能将它们中的任何一个分配给`int`类型的变量而不进行类型转换。此外，在`Score`上定义的任何方法在`HighScore`上也不存在：

```go
// assigning untyped constants is valid
var i int = 300
var s Score = 100
var hs HighScore = 200
hs = s                  // compilation error!
s = i                   // compilation error!
s = Score(i)            // ok
hs = HighScore(s)       // ok
```

具有内置类型作为其基础类型的用户定义类型可以分配与基础类型兼容的字面量和常量。它们也可以与这些类型的运算符一起使用：

```go
var s Score = 50
scoreWithBonus := s + 100 // type of scoreWithBonus is Score
```

###### 提示

在具有共享基础类型的类型之间进行类型转换会保持相同的基础存储，但关联不同的方法。

## 类型是可执行文档

尽管我们深知应该声明一个结构体类型来保存一组相关数据，但当您应该声明一个基于其他内置类型或基于另一个用户定义类型的用户定义类型时，这一点并不十分清晰。简短的答案是，类型是文档。它们通过为概念提供名称并描述预期数据的类型，使代码更加清晰。对于阅读您代码的人来说，当一个方法有一个`Percentage`类型的参数时，比有一个`int`类型的参数更加清晰，而且更难以使用无效值调用它。

当声明一个用户定义的类型基于另一个用户定义的类型时，相同的逻辑也适用。当您拥有相同的基础数据但需要执行不同操作集时，请创建两种类型。声明一种类型基于另一种类型可以避免一些重复，并清楚地表明这两种类型是相关的。

# `iota`用于枚举——有时

许多编程语言都有枚举的概念，允许你指定类型只能有限的几个值。Go 语言没有枚举类型，而是使用 `iota`，它允许你为一组常量赋予递增的值。

###### 注意

`iota` 的概念源自编程语言 APL（全称“A Programming Language”）。在 APL 中，要生成前三个正整数的列表，你可以写作 `ι3`，其中 `ι` 是小写希腊字母 iota。

APL 因其高度依赖自定义符号而闻名，以至于需要配备特殊键盘的计算机。例如，(~R∊R∘.×R)/R←1↓ιR 是一个 APL 程序，用于找出小于变量 R 值的所有质数。

Go 语言强调可读性，却从另一种以简洁著称的语言借鉴概念，这或许有些讽刺，但这也是为什么你应该学习多种编程语言：你可以从任何地方获得灵感。

使用 `iota` 时，最佳实践是首先定义一个基于 `int` 的类型，该类型将表示所有有效值：

```go
type MailCategory int
```

接下来，使用 `const` 块为你的类型定义一组值：

```go
const (
    Uncategorized MailCategory = iota
    Personal
    Spam
    Social
    Advertisements
)
```

在 `const` 块中，第一个常量的类型已经指定，并且其值设置为 `iota`。随后的每一行既没有指定类型也没有指定值。当 Go 编译器看到这种写法时，会将类型和赋值重复到块中的所有后续常量，这就是 `iota` 的作用。`iota` 的值递增，从 `0` 开始为第一个常量（`Uncategorized`），第二个常量为 `1`（`Personal`），依此类推。当创建新的 `const` 块时，`iota` 会被重置为 `0`。

`iota` 的值递增，每个常量在 `const` 块中，无论是否使用 `iota` 来定义常量的值。以下代码演示了在 `const` 块中间歇性使用 `iota` 时会发生什么：

```go
const (
    Field1 = 0
    Field2 = 1 + iota
    Field3 = 20
    Field4
    Field5 = iota
)

func main() {
    fmt.Println(Field1, Field2, Field3, Field4, Field5)
}
```

你可以在[The Go Playground](https://oreil.ly/jTXxD)上运行此代码，并查看（也许是意外的）结果：

```go
0 2 20 20 4
```

`Field2` 被赋值为 `2`，因为在 `const` 块的第二行中，`iota` 的值为 `1`。`Field4` 被赋值为 `20`，因为它没有显式指定类型或值，所以它获取了前一行的具有类型和赋值的值。最后，`Field5` 获取值 `4`，因为它是第五行，而 `iota` 从 `0` 开始计数。

这是我见过的关于 `iota` 的最佳建议：

> 不要将*iota*用于定义其值在其他地方明确定义的常量。例如，在实施规范的部分和规范指定分配给哪些常量的值时，应明确写出常量值。仅在“内部”目的中使用*iota*。也就是说，通过名称引用而不是通过值引用常量。这样，您可以在任何时刻/列表中的位置上最优地使用*iota*插入新常量，而不会造成破坏。
> 
> [丹尼·范·休曼](https://oreil.ly/3MKwn)

重要的是要理解，Go 中没有任何东西会阻止您（或其他任何人）创建您类型的其他值。此外，如果在字面常量列表中间插入一个新标识符，则所有后续标识符将重新编号。如果这些常量表示另一个系统或数据库中的值，则会以微妙的方式破坏您的应用程序。鉴于这两个限制条件，基于 `iota` 的枚举仅在您关心能够区分一组值并且不特别关心背后的值时才有意义。如果实际值很重要，请明确指定它。

###### 警告

因为您可以将字面表达式分配给常量，所以您会看到示例代码建议在此类情况下使用 `iota`：

```go
type BitField int

const (
    Field1 BitField = 1 << iota // assigned 1
    Field2                      // assigned 2
    Field3                      // assigned 4
    Field4                      // assigned 8
)
```

虽然这很聪明，但在使用此模式时要小心。如果这样做，请记录您所做的事情。如前所述，当您关心值时，使用 `iota` 与常量是脆弱的。您不希望未来的维护者在列表中间插入新常量并破坏您的代码。

注意，`iota` 从 0 开始编号。如果您使用自己的常量集表示不同的配置状态，则零值可能会有用。您在 `MailCategory` 类型中之前看到过这一点。当邮件首次到达时，它是未分类的，因此零值是合理的。如果您的常量没有合理的默认值，一个常见的模式是将常量块中的第一个 `iota` 值分配给 `_` 或指示该值无效的常量。这样可以轻松检测变量是否已正确初始化。

# 使用嵌入进行组合

软件工程建议“优先使用对象组合而不是类继承”可以追溯到至少 1994 年出版的 *设计模式* 一书（Erich Gamma、Richard Helm、Ralph Johnson 和 John Vlissides，Addison-Wesley），更为人所知的是四人帮书籍。虽然 Go 没有继承，但通过内置支持组合和提升来鼓励代码重用：

```go
type Employee struct {
    Name         string
    ID           string
}

func (e Employee) Description() string {
    return fmt.Sprintf("%s (%s)", e.Name, e.ID)
}

type Manager struct {
    Employee
    Reports []Employee
}

func (m Manager) FindNewEmployees() []Employee {
    // do business logic
}
```

请注意，`Manager` 包含一个类型为 `Employee` 的字段，但未为该字段分配名称。这使 `Employee` 成为一个*嵌入字段*。在嵌入字段上声明的任何字段或方法都会*提升*到包含的结构体中，并且可以直接在其上调用。这使得以下代码有效：

```go
m := Manager{
    Employee: Employee{
        Name: "Bob Bobson",
        ID:   "12345",
    },
    Reports: []Employee{},
}
fmt.Println(m.ID)            // prints 12345
fmt.Println(m.Description()) // prints Bob Bobson (12345)
```

###### 注意

你可以嵌入结构中的任何类型，不仅仅是另一个结构。这样可以将嵌入类型的方法提升到包含它的结构体中。

如果包含的结构体具有与嵌入字段相同名称的字段或方法，则需要使用嵌入字段的类型来引用被隐藏的字段或方法。如果你定义的类型如下：

```go
type Inner struct {
    X int
}

type Outer struct {
    Inner
    X int
}
```

只能通过显式指定`Inner`来访问`Inner`上的`X`：

```go
o := Outer{
    Inner: Inner{
        X: 10,
    },
    X: 20,
}
fmt.Println(o.X)       // prints 20
fmt.Println(o.Inner.X) // prints 10
```

# 嵌入不是继承

编程语言中内置的嵌入支持很少见（我不知道其他流行语言支持它）。许多熟悉继承（在许多语言中都有）的开发人员尝试将嵌入视为继承来理解。这条路不通。你不能将`Manager`类型的变量赋给`Employee`类型的变量。如果你想访问`Manager`中的`Employee`字段，必须显式地这样做。你可以在[Go Playground](https://oreil.ly/vBl7o)上运行以下代码或者使用[第七章仓库](https://oreil.ly/qJQgV)中的*sample_code/embedding*目录中的代码：

```go
var eFail Employee = m        // compilation error!
var eOK Employee = m.Employee // ok!
```

你将会得到错误：

```go
cannot use m (type Manager) as type Employee in assignment
```

此外，Go 没有具体类型的*动态分派*。嵌入字段上的方法并不知道它们被嵌入了。如果在嵌入字段上有一个方法调用另一个嵌入字段上的方法，并且包含结构体具有同名方法，则会调用嵌入字段上的方法，而不是包含结构体上的方法。此行为在以下代码中得到了演示，你可以在[Go Playground](https://oreil.ly/yN6bV)上运行它，或者使用[第七章仓库](https://oreil.ly/qJQgV)中的*sample_code/no_dispatch*目录中的代码：

```go
type Inner struct {
    A int
}

func (i Inner) IntPrinter(val int) string {
    return fmt.Sprintf("Inner: %d", val)
}

func (i Inner) Double() string {
    return i.IntPrinter(i.A * 2)
}

type Outer struct {
    Inner
    S string
}

func (o Outer) IntPrinter(val int) string {
    return fmt.Sprintf("Outer: %d", val)
}

func main() {
    o := Outer{
        Inner: Inner{
            A: 10,
        },
        S: "Hello",
    }
    fmt.Println(o.Double())
}
```

运行此代码将产生以下输出：

```go
Inner: 20
```

虽然在一个具体类型内嵌另一个类型不允许你将外部类型视为内部类型，但嵌入字段的方法确实计入包含结构体的*方法集*。这意味着它们可以使包含的结构体实现一个接口。

# 接口的快速课程

尽管 Go 的并发模型（我在第十二章中介绍）备受瞩目，但 Go 设计的真正亮点是它的隐式接口，这是 Go 中唯一的抽象类型。让我们看看它们为何如此出色。

让我们先快速看一下如何声明接口。接口在本质上很简单。像其他用户定义的类型一样，你使用`type`关键字。

这是`fmt`包中`Stringer`接口的定义：

```go
type Stringer interface {
    String() string
}
```

在接口声明中，接口文字出现在接口类型名称之后。它列出了具体类型必须实现的方法，以符合接口的要求。接口定义的方法称为接口的方法集。正如我在“指针接收器和值接收器”中所述，指针实例的方法集包含使用指针接收器和值接收器定义的方法，而值实例的方法集仅包含使用值接收器定义的方法。以下是使用先前定义的`Counter`结构的快速示例：

```go
type Incrementer interface {
    Increment()
}

var myStringer fmt.Stringer
var myIncrementer Incrementer
pointerCounter := &Counter{}
valueCounter := Counter{}

myStringer = pointerCounter    // ok
myStringer = valueCounter      // ok
myIncrementer = pointerCounter // ok
myIncrementer = valueCounter   // compile-time error!
```

尝试编译这段代码会导致错误`cannot use valueCounter (variable of type Counter) as Incrementer value in assignment: Counter does not implement Incrementer (method Increment has pointer receiver).`

你可以在[The Go Playground](https://oreil.ly/yYx8Q)上尝试这段代码，或者使用[第七章代码库](https://oreil.ly/qJQgV)中*sample_code/method_set*目录中的代码。

与其他类型一样，接口可以在任何块中声明。

接口通常以“er”结尾命名。你已经见过`fmt.Stringer`，但还有很多，包括`io.Reader`、`io.Closer`、`io.ReadCloser`、`json.Marshaler`和`http.Handler`。

# 接口是类型安全的鸭子类型

到目前为止，关于 Go 语言接口的讨论并没有什么不同于其他语言的接口。使 Go 语言接口特殊的是它们是*隐式*实现的。正如你在前面示例中使用的`Counter`结构类型和`Incrementer`接口类型所看到的那样，具体类型不声明它实现了一个接口。如果具体类型的方法集包含接口方法集中的所有方法，则具体类型实现了该接口。因此，具体类型可以赋给声明为接口类型的变量或字段。

这种隐式行为使接口成为 Go 语言中最有趣的类型特性，因为它们既实现了类型安全又实现了解耦，将静态语言和动态语言的功能桥接起来。

要理解其中的原因，让我们讨论一下为什么语言需要接口。前面我提到过*设计模式*教导开发者更倾向于组合而非继承。书中的另一个建议是“针对接口编程，而不是针对实现编程”。这样做允许你依赖行为而不是实现，使你能够根据需要交换实现。这使得你的代码能够随着需求的变化而演变。

Python、Ruby 和 JavaScript 等动态类型语言没有接口。相反，这些开发者使用*鸭子类型*，其基于“如果它走起来像鸭子，叫起来像鸭子，那它就是鸭子”的表达。该概念是，只要函数能够找到预期的方法来调用，你就可以将类型的实例作为参数传递给函数：

```go
class Logic:
    def process(self, data):
        # business logic

def program(logic):
    # get data from somewhere
    logic.process(data)

logicToUse = Logic()
program(logicToUse)
```

鸭子类型起初可能听起来很奇怪，但它已被用于构建大型且成功的系统。如果你在静态类型语言中编程，这听起来像彻头彻尾的混乱。在没有明确指定类型的情况下，很难知道应该期望什么功能。当新开发人员加入项目或现有开发人员忘记代码的作用时，他们必须跟踪代码以确定实际的依赖关系。

Java 开发人员使用不同的模式。他们定义一个接口，创建接口的实现，但仅在客户端代码中引用接口：

```go
public interface Logic {
    String process(String data);
}

public class LogicImpl implements Logic {
    public String process(String data) {
        // business logic
    }
}

public class Client {
    private final Logic logic;
    // this type is the interface, not the implementation

    public Client(Logic logic) {
        this.logic = logic;
    }

    public void program() {
        // get data from somewhere
        this.logic.process(data);
    }
}

public static void main(String[] args) {
    Logic logic = new LogicImpl();
    Client client = new Client(logic);
    client.program();
}
```

动态语言开发人员看到 Java 中的显式接口，不明白在有显式依赖关系的情况下如何随时间重构代码。从不同提供者切换到新实现意味着重写代码以依赖新接口。

Go 的开发人员认为两种方法都是正确的。如果你的应用程序会随时间增长和变化，你需要灵活性来改变实现。然而，为了让人们理解你的代码在做什么（随着时间的推移，新的人员在同一代码上工作时），你还需要指定代码依赖的内容。这就是隐式接口的用武之地。Go 代码是前两种风格的混合：

```go
type LogicProvider struct {}

func (lp LogicProvider) Process(data string) string {
    // business logic
}

type Logic interface {
    Process(data string) string
}

type Client struct{
    L Logic
}

func(c Client) Program() {
    // get data from somewhere
    c.L.Process(data)
}

main() {
    c := Client{
        L: LogicProvider{},
    }
    c.Program()
}
```

Go 代码提供了一个接口，但只有调用者（`Client`）知道它；在`LogicProvider`上没有声明任何内容来表明它符合接口。这足以允许将来引入新的逻辑提供者，并提供可执行的文档以确保传递给客户端的任何类型都与客户端的需求匹配。

###### Tip

接口指定调用者需要什么。客户端代码定义接口以指定它需要哪些功能。

这并不意味着接口不能共享。你已经在标准库中看到了几个用于输入和输出的接口。拥有标准接口是强大的；如果你的代码编写为与`io.Reader`和`io.Writer`一起工作，它将正确地运行，无论是写入本地磁盘上的文件还是内存中的值。

此外，使用标准接口鼓励*装饰器模式*。在 Go 中编写工厂函数的常见做法是，接受一个接口实例并返回实现相同接口的另一种类型。例如，假设你有以下定义的函数：

```go
func process(r io.Reader) error
```

你可以用以下代码处理来自文件的数据：

```go
r, err := os.Open(fileName)
if err != nil {
    return err
}
defer r.Close()
return process(r)
```

由 `os.Open` 返回的 `os.File` 实例符合 `io.Reader` 接口，并可以在任何读取数据的代码中使用。如果文件是 gzip 压缩的，你可以将 `io.Reader` 包装在另一个 `io.Reader` 中：

```go
r, err := os.Open(fileName)
if err != nil {
    return err
}
defer r.Close()
gz, err := gzip.NewReader(r)
if err != nil {
    return err
}
defer gz.Close()
return process(gz)
```

现在从未压缩文件读取的完全相同的代码改为从压缩文件读取。

###### Tip

如果标准库中的接口描述了你的代码所需的功能，请使用它！常用的接口包括`io.Reader`、`io.Writer`和`io.Closer`。

对于满足接口的类型而言，指定附加方法并不是不合适的做法。一组客户端代码可能不关心这些方法，但其他人可能关心。例如，`io.File` 类型也满足 `io.Writer` 接口。如果你的代码只关心从文件中读取数据，可以使用 `io.Reader` 接口来引用文件实例，并忽略其他方法。

# 嵌入和接口

嵌入不仅适用于结构体。你还可以在接口中嵌入接口。例如，`io.ReadCloser` 接口由 `io.Reader` 和 `io.Closer` 组成：

```go
type Reader interface {
        Read(p []byte) (n int, err error)
}

type Closer interface {
        Close() error
}

type ReadCloser interface {
        Reader
        Closer
}
```

###### 注意

正如你可以在结构体中嵌入具体类型一样，你也可以在结构体中嵌入接口。你会在“在 Go 中使用存根”中看到这种用法。

# 接受接口，返回结构体

你经常会听到经验丰富的 Go 开发者说，你的代码应该“接受接口，返回结构体”。这句话很可能是由 Jack Lindamood 在他 2016 年的博客文章[“Go 中的预防性接口反模式”](https://oreil.ly/OT1yi)中首次提出的。这意味着由你的函数调用的业务逻辑应通过接口调用，但你函数的输出应为具体类型。我已经解释了为什么函数应接受接口：它们使你的代码更灵活，并明确声明正在使用的确切功能。

函数应返回具体类型的主要原因是它们使得在代码的新版本中逐步更新函数的返回值变得更容易。当函数返回一个具体类型时，可以添加新方法和字段而不会破坏调用该函数的现有代码，因为新字段和方法会被忽略。但对于接口来说并非如此。向接口添加新方法意味着必须更新该接口的所有现有实现，否则你的代码就会出现问题。从语义化版本的角度来看，这是后向兼容的次要发布和后向不兼容的主要发布之间的区别。如果你正在暴露一个 API 供其他人使用（无论是在你的组织内部还是作为开源项目的一部分），避免破坏向后兼容的变更会让你的用户满意。

在某些罕见情况下，最不坏的选择是使你的函数返回接口。例如，标准库中的`database/sql/driver`包定义了一组接口，定义了数据库驱动程序必须提供的内容。数据库驱动程序的作者有责任提供这些接口的具体实现，因此标准库中`database/sql/driver`定义的所有接口上的几乎所有方法都返回接口。从 Go 1.8 开始，期望数据库驱动程序支持额外的功能。标准库承诺兼容性，因此不能更新现有接口以添加新方法，也不能更新这些接口上的现有方法以返回不同类型。解决方案是保持现有接口不变，定义描述新功能的新接口，并告诉数据库驱动程序作者他们应该在其具体类型上实现旧方法和新方法。

这引出了一个问题，即如何检查这些新方法是否存在，以及如果存在如何访问它们。你将在“类型断言和类型开关”中学习如何做到这一点。

而不是编写一个返回基于输入参数返回不同实例的接口的单个工厂函数，试着为每种具体类型编写单独的工厂函数。在某些情况下（例如可以返回一种或多种类型令牌的解析器），这是不可避免的，你别无选择，只能返回一个接口。

错误是此规则的一个例外。正如你将在第九章中看到的那样，Go 函数和方法可以声明返回`error`接口类型的返回参数。在`error`的情况下，不同的接口实现很可能会被返回，因此你需要使用接口来处理所有可能的选项，因为在 Go 中接口是唯一的抽象类型。

这种模式存在一个潜在的缺点。正如我在“减少垃圾收集器工作量”中讨论的那样，减少堆分配可以通过减少垃圾收集器的工作量来提高性能。返回一个结构体可以避免堆分配，这是很好的。然而，当调用带有接口类型参数的函数时，每个接口参数都会进行堆分配。找出更好的抽象与更好的性能之间的权衡应该在程序的整个生命周期内完成。编写你的代码使其可读和可维护。如果你发现你的程序太慢，并且你已经对其进行了分析，并且确定性能问题是由接口参数引起的堆分配导致的，那么你应该重写函数以使用具体类型参数。如果将多个接口实现传递给函数，这意味着需要创建多个带有重复逻辑的函数。

###### 注意

来自 C++或 Rust 背景的开发人员可能会尝试使用泛型作为让编译器生成专门函数的一种方式。截至 Go 1.21，这可能不会生成更快的代码。我将在“Go 语言惯用法和泛型”中详细介绍原因。

# 接口和`nil`

在讨论指针时（见第六章](ch06.html#unique_chapter_id_06)，我还谈到了`nil`，即指针类型的零值。您也可以使用`nil`来表示接口实例的零值，但这对于具体类型来说并不简单。

要理解接口与`nil`之间的关系，需要了解一些关于接口实现方式的知识。在 Go 运行时中，接口实现为一个结构体，其中包含两个指针字段，一个用于值，一个用于值的类型。只要类型字段非`nil`，接口就非`nil`。（由于不能有没有类型的变量，如果值指针非`nil`，则类型指针始终非`nil`。

要将接口视为`nil`，*类型和值*都必须为`nil`。以下代码在前两行上打印出`true`，在最后一行上打印出`false`：

```go
var pointerCounter *Counter
fmt.Println(pointerCounter == nil) // prints true
var incrementer Incrementer
fmt.Println(incrementer == nil) // prints true
incrementer = pointerCounter
fmt.Println(incrementer == nil) // prints false
```

您可以在[Go Playground](https://oreil.ly/njtz9)上运行此代码，或者使用[第七章存储库](https://oreil.ly/qJQgV)中的*sample_code/interface_nil*目录中的代码。

对于具有接口类型的变量，`nil`表示您是否可以在其上调用方法。正如我之前讨论过的，您可以在`nil`具体实例上调用方法，因此对于分配了`nil`具体实例的接口变量也可以调用方法。如果接口变量为`nil`，在其上调用任何方法将触发恐慌（我将在“恐慌和恢复”中讨论）。如果接口变量为非`nil`，则可以在其上调用方法。（但请注意，如果值为`nil`并且分配类型的方法未正确处理`nil`，仍可能触发恐慌。）

由于具有非`nil`类型的接口实例不等于`nil`，因此在类型为非`nil`时，要确定与接口关联的值是否为`nil`并不简单。您必须使用反射（我将在“使用反射检查接口值是否为 nil”中讨论）来找出。

# 接口可比较

在第三章中，您学习了可比较类型，这些类型可以使用`==`进行相等性检查。也许您会惊讶地发现接口也是可以比较的。正如一个接口仅在其类型和值字段都为`nil`时才等于`nil`一样，两个接口类型的实例仅在它们的类型和值都相等时才相等。这引发了一个问题：如果类型不可比较会发生什么？让我们通过一个简单的例子来探讨这个概念。首先定义一个接口和该接口的一些实现：

```go
type Doubler interface {
    Double()
}

type DoubleInt int

func (d *DoubleInt) Double() {
    *d = *d * 2
}

type DoubleIntSlice []int

func (d DoubleIntSlice) Double() {
    for i := range d {
        d[i] = d[i] * 2
    }
}
```

`DoubleInt` 上的 `Double` 方法声明为指针接收器，因为你要修改 `int` 的值。对于 `DoubleIntSlice` 上的 `Double` 方法，可以使用值接收器，因为如 “映射与切片的区别” 所述，可以更改参数为切片类型的项的值。`*DoubleInt` 类型是可比较的（所有指针类型都是），而 `DoubleIntSlice` 类型不可比较（切片不可比较）。

你还有一个函数，它接受两个 `Doubler` 类型的参数，并打印它们是否相等：

```go
func DoublerCompare(d1, d2 Doubler) {
    fmt.Println(d1 == d2)
}
```

现在你定义了四个变量：

```go
var di DoubleInt = 10
var di2 DoubleInt = 10
var dis = DoubleIntSlice{1, 2, 3}
var dis2 = DoubleIntSlice{1, 2, 3}
```

现在，你要调用这个函数三次。第一次调用如下：

```go
DoublerCompare(&di, &di2)
```

这会打印出 `false`。类型匹配（都是 `*DoubleInt`），但你比较的是指针而不是值，而这些指针指向不同的实例。

接下来，你将 `*DoubleInt` 与 `DoubleIntSlice` 进行比较：

```go
DoublerCompare(&di, dis)
```

这会打印出 `false`，因为类型不匹配。

最后，你遇到了问题的情况：

```go
DoublerCompare(dis, dis2)
```

此代码可以编译通过，但在运行时会触发 panic：

```go
panic: runtime error: comparing uncomparable type main.DoubleIntSlice
```

整个程序在[第七章存储库的 *sample_code/comparable* 目录](https://oreil.ly/qJQgV)中都可用。

还要注意，映射的键必须是可比较的，因此可以定义一个接口作为键的映射：

```go
m := map[Doubler]int{}
```

如果向此映射添加键值对且键不可比较，那也会触发 panic。

考虑到这种行为，在使用接口进行 `==` 或 `!=` 比较时要小心，或者将接口用作映射键时，很容易意外触发会崩溃程序的 panic。即使当前所有接口实现都是可比较的，你也不知道其他人在使用或修改你的代码时会发生什么，并且没有办法指定接口只能由可比较类型实现。如果想要更加安全，可以使用 `reflect.Value` 的 `Comparable` 方法在使用 `==` 或 `!=` 之前检查接口。（关于反射，你可以在 “反射让你在运行时操作类型” 中了解更多）。

# 空接口毫无意义

有时在静态类型语言中，你需要一种方式来表示变量可以存储任何类型的值。Go 使用 *空接口* `interface{}` 表示这种情况：

```go
var i interface{}
i = 20
i = "hello"
i = struct {
    FirstName string
    LastName string
} {"Fred", "Fredson"}
```

###### 注意

`interface{}` 并非特例语法。空接口类型仅表示变量可以存储任何类型的值，该类型实现了零个或多个方法。这恰好匹配 Go 中的每种类型。

为了提高可读性，Go 在 `interface{}` 的类型别名中添加了 `any`。遗留代码（在 Go 1.18 中添加 `any` 之前编写的代码）使用 `interface{}`，但对于新代码，请坚持使用 `any`。

因为空接口无法告诉你关于其表示的值的任何信息，所以你无法对其进行太多操作。`any` 的一个常见用途是作为从外部源（如 JSON 文件）读取的具有不确定模式的数据的占位符：

```go
data := map[string]any{}
contents, err := os.ReadFile("testdata/sample.json")
if err != nil {
    return err
}
json.Unmarshal(contents, &data)
// the contents are now in the data map
```

###### 注意

在 Go 添加泛型之前编写的用户创建的数据容器使用空接口来存储值（我将在第 8 章中讨论泛型）。标准库中的一个示例是[`container/list`](https://oreil.ly/53tmr)。现在泛型是 Go 的一部分，请为任何新创建的数据容器使用它们。

如果您看到一个接受空接口的函数，很可能使用反射（我将在第 16 章中讨论）来填充或读取值。在上面的例子中，`json.Unmarshal`函数的第二个参数声明为`any`类型。

这些情况应该相对罕见。避免使用`any`。正如您所看到的，Go 被设计为一种强类型语言，试图绕过这一点是不符合习惯的。

如果您发现自己处于必须将值存储到空接口中的情况中，您可能想知道如何再次读取该值。为了做到这一点，您需要了解类型断言和类型切换。

# 类型断言和类型切换

Go 为检查接口类型的变量是否具有特定具体类型或具体类型是否实现另一个接口提供了两种方法。让我们从看类型断言开始。*类型断言*指出实现接口的具体类型，或指出另一个同样被存储在接口中值的具体类型也实现了另一个接口。您可以在[Go Playground](https://oreil.ly/XUfuO)上试试，或在[第七章仓库](https://oreil.ly/qJQgV)的*sample_code/type_assertions*目录中的*main.go*中的`typeAssert`函数中看到：

```go
type MyInt int

func main() {
    var i any
    var mine MyInt = 20
    i = mine
    i2 := i.(MyInt)
    fmt.Println(i2 + 1)
}
```

在上述代码中，变量`i2`的类型为`MyInt`。

您可能想知道如果类型断言错误会发生什么。在这种情况下，您的代码会导致恐慌。您可以在[Go Playground](https://oreil.ly/qoXu_)上尝试或在[第七章仓库](https://oreil.ly/qJQgV)的*sample_code/type_assertions*目录中的*main.go*中的`typeAssertPanicWrongType`函数中看到这一点：

```go
i2 := i.(string)
fmt.Println(i2)
```

运行此代码会导致以下恐慌：

```go
panic: interface conversion: interface {} is main.MyInt, not string
```

正如您已经看到的，Go 对具体类型非常谨慎。即使两种类型共享底层类型，类型断言也必须匹配存储在接口中的值的类型。以下代码将导致恐慌。您可以在[Go Playground](https://oreil.ly/YUaka)上尝试或在[第七章仓库](https://oreil.ly/qJQgV)的*sample_code/type_assertions*目录中的*main.go*中的`typeAssertPanicTypeNotIdentical`函数中看到：

```go
i2 := i.(int)
fmt.Println(i2 + 1)
```

显然，崩溃不是预期的行为。您可以通过使用逗号 ok 惯用法来避免这种情况，就像在检测地图中是否有零值时所看到的“逗号 ok 惯用法”一样。您可以在[第七章仓库](https://oreil.ly/qJQgV)的*sample_code/type_assertions*目录中的*main.go*中的`typeAssertCommaOK`函数中看到这一点：

```go
i2, ok := i.(int)
if !ok {
    return fmt.Errorf("unexpected type for %v",i)
}
fmt.Println(i2 + 1)
```

如果类型转换成功，则布尔值`ok`设置为`true`。如果不成功，则`ok`设置为`false`，另一个变量（在本例中为`i2`）设置为其零值。然后，你在`if`语句中处理意外情况。我会在第九章中详细讨论错误处理。

###### 注意

类型断言与类型转换非常不同。转换将值更改为新类型，而断言则显示存储在接口中的值的类型。类型转换可以应用于具体类型和接口。类型断言只能应用于接口类型。所有类型断言在运行时都会被检查，因此如果你不使用逗号 ok 惯用法，它们可能会在运行时失败并引发恐慌。大多数类型转换在编译时检查，因此如果无效，你的代码将无法编译。（切片和数组指针之间的类型转换可能在运行时失败，并且不支持逗号 ok 惯用法，因此在使用它们时要小心！）

即使你绝对确定你的类型断言是有效的，也使用逗号 OK 惯用法版本。你不知道其他人（或者六个月后的你）将如何重用你的代码。 sooner or later, your unvalidated type assertions will fail at runtime.

当一个接口可能是多种可能类型之一时，应该使用*类型 switch*代替：

```go
func doThings(i any) {
    switch j := i.(type) {
    case nil:
        // i is nil, type of j is any
    case int:
        // j is of type int
    case MyInt:
        // j is of type MyInt
    case io.Reader:
        // j is of type io.Reader
    case string:
        // j is a string
    case bool, rune:
        // i is either a bool or rune, so j is of type any
    default:
        // no idea what i is, so j is of type any
    }
}
```

类型`switch`看起来很像你早些时候在`switch`中看到的`switch`语句。不同之处在于，你指定一个接口类型的变量，然后跟上`.(type)`。通常情况下，你将正在检查的变量分配给另一个仅在`switch`内有效的变量。

###### 注意

由于类型`switch`的目的是从现有变量派生新变量，因此将正在进行 switch 的变量分配给同名变量（`i := i.(type)`）是一种惯用法，这是少数几个使用影子变量是一个好主意的地方之一。为了使注释更易读，本例没有使用影子变量。

新变量的类型取决于哪种情况匹配。你可以在一个情况中使用`nil`来查看接口是否没有关联类型。如果在一个情况中列出多种类型，则新变量的类型为`any`。与`switch`语句一样，如果没有指定类型匹配时有一个`default`情况，则新变量的类型与匹配的情况类型相同。

到目前为止的例子都使用了带有类型断言和类型 switch 的`any`接口，你可以从所有接口类型中发现具体类型。

###### 提示

如果你*不知道*接口中存储的值的类型，你需要使用反射。我会在第十六章中详细讨论反射。

# 谨慎使用类型断言和类型 switch

虽然能从接口变量中提取具体实现看似方便，但应该少用这些技术。大多数情况下，应将参数或返回值视为所提供的类型，而不是可能的其他类型。否则，函数的 API 未能准确声明其执行任务所需的类型。如果需要不同的类型，应该明确指定。

尽管如此，类型断言和类型切换在某些情况下很有用。类型断言的一个常见用法是查看接口背后的具体类型是否还实现了另一个接口。这允许您指定可选接口。例如，标准库使用这种技术在调用 `io.Copy` 函数时允许更高效的复制。此函数具有两个类型为 `io.Writer` 和 `io.Reader` 的参数，并调用 `io.copyBuffer` 函数来执行其工作。如果 `io.Writer` 参数还实现了 `io.WriterTo`，或者 `io.Reader` 参数还实现了 `io.ReaderFrom`，则函数中的大部分工作可以跳过。

```go
// copyBuffer is the actual implementation of Copy and CopyBuffer.
// if buf is nil, one is allocated.
func copyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
    // If the reader has a WriteTo method, use it to do the copy.
    // Avoids an allocation and a copy.
    if wt, ok := src.(WriterTo); ok {
        return wt.WriteTo(dst)
    }
    // Similarly, if the writer has a ReadFrom method, use it to do the copy.
    if rt, ok := dst.(ReaderFrom); ok {
        return rt.ReadFrom(src)
    }
    // function continues...
}
```

另一个使用可选接口的地方是在演化 API 时。正如《接受接口，返回结构》中所述，数据库驱动程序的 API 随时间变化而变化。这种变化的原因之一是上下文的添加，在第十四章中有详细讨论。上下文是传递给函数的参数，提供了管理取消等功能的标准方式。它在 Go 1.7 中添加，这意味着旧代码不支持它。这包括旧的数据库驱动程序。

在 Go 1.8 中，`database/sql/driver` 包定义了现有接口的新的上下文感知模拟。例如，`StmtExecContext` 接口定义了一个名为 `ExecContext` 的方法，它是 `Stmt` 中 `Exec` 方法的上下文感知替代。当将 `Stmt` 的实现传递给标准库数据库代码时，它会检查是否还实现了 `StmtExecContext`。如果是，将调用 `ExecContext`。如果不是，Go 标准库提供了一个取消支持的后备实现：

```go
func ctxDriverStmtExec(ctx context.Context, si driver.Stmt,
                       nvdargs []driver.NamedValue) (driver.Result, error) {
    if siCtx, is := si.(driver.StmtExecContext); is {
        return siCtx.ExecContext(ctx, nvdargs)
    }
    // fallback code is here
}
```

这种可选接口技术有一个缺点。前面提到，接口的实现通常使用装饰器模式来包装相同接口的其他实现以进行层级行为。问题在于，如果一个可选接口由包装实现之一实现，你无法使用类型断言或类型切换来检测它。例如，标准库包括一个 `bufio` 包，提供了缓冲读取器。您可以通过将其传递给 `bufio.NewReader` 函数并使用返回的 `*bufio.Reader` 缓冲任何其他 `io.Reader` 实现。如果传入的 `io.Reader` 也实现了 `io.ReaderFrom`，将其包装在缓冲读取器中将阻止优化。

处理错误时也可以看到这一点。如前所述，它们实现了 `error` 接口。错误可以通过包装其他错误来包含额外信息。类型开关或类型断言无法检测或匹配已包装的错误。如果您希望采取不同的行为来处理返回错误的不同具体实现，请使用 `errors.Is` 和 `errors.As` 函数来测试和访问已包装的错误。

类型 `switch` 语句提供了区分需要不同处理的接口多个实现的能力。当只有某些可能的有效类型可以提供给接口时，它们非常有用。确保在类型 `switch` 中包含一个 `default` 情况来处理在开发时不知道的实现。这可以保护您免受在添加新接口实现时忘记更新类型 `switch` 语句的影响：

```go
func walkTree(t *treeNode) (int, error) {
    switch val := t.val.(type) {
    case nil:
        return 0, errors.New("invalid expression")
    case number:
        // we know that t.val is of type number, so return the
        // int value
        return int(val), nil
    case operator:
        // we know that t.val is of type operator, so
        // find the values of the left and right children, then
        // call the process() method on operator to return the
        // result of processing their values.
        left, err := walkTree(t.lchild)
        if err != nil {
            return 0, err
        }
        right, err := walkTree(t.rchild)
        if err != nil {
            return 0, err
        }
        return val.process(left, right), nil
    default:
        // if a new treeVal type is defined, but walkTree wasn't updated
        // to process it, this detects it
        return 0, errors.New("unknown node type")
    }
}
```

你可以在[Go Playground](https://oreil.ly/jDhqM)上看到完整的实现，或者在[第七章存储库](https://oreil.ly/qJQgV)的 *sample_code/type_switch* 目录中找到。

###### 注意

您可以通过使接口未导出并且至少一个方法未导出来进一步保护自己免受意外接口实现的影响。如果接口被导出，它可以嵌入在另一个包中的结构体中，使结构体实现接口。我将在 第十章 中更多地讨论包和导出标识符。

# 函数类型是接口的桥梁

在类型声明中，还有一件事情我没有讲到。一旦你理解了在结构体上声明方法的概念，你就能开始看到具有 `int` 或 `string` 底层类型的用户定义类型也可以有方法。毕竟，方法提供了与实例状态交互的业务逻辑，而整数和字符串也有状态。

然而，在 Go 中，方法允许在 *任何* 用户定义的类型上，包括用户定义的函数类型。这听起来像是一个学术边角案例，但它们实际上非常有用。它们允许函数实现接口。最常见的用法是用于 HTTP 处理程序。HTTP 处理程序处理 HTTP 服务器请求。它由一个接口定义：

```go
type Handler interface {
    ServeHTTP(http.ResponseWriter, *http.Request)
}
```

通过使用类型转换到 `http.HandlerFunc`，任何具有 `func(http.ResponseWriter,*http.Request)` 签名的函数都可以用作 `http.Handler`：

```go
type HandlerFunc func(http.ResponseWriter, *http.Request)

func (f HandlerFunc) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    f(w, r)
}
```

这使您可以使用函数、方法或闭包实现 HTTP 处理程序，使用与用于满足 `http.Handler` 接口的其他类型相同的代码路径。

Go 中的函数是一流的概念，因此它们经常作为参数传递给函数。同时，Go 鼓励小接口，并且仅有一个方法的接口可以轻松替代函数类型的参数。问题在于：在何时应该让您的函数或方法指定函数类型的输入参数，而何时应该使用接口？

如果您的单一函数可能依赖于许多其他函数或未在其输入参数中指定的其他状态，请使用接口参数并定义函数类型以将函数桥接到接口。这就是在`http`包中所做的；`Handler`很可能只是需要配置的一系列调用的入口点。但是，如果它是一个简单的函数（例如在`sort.Slice`中使用的函数），那么函数类型的参数是一个很好的选择。

# 隐式接口使依赖注入更加容易

在前言中，我将软件编写比作建造桥梁。软件与物理基础设施的共同点之一是，任何被多人长时间使用的程序都需要维护。虽然程序不会磨损，但开发人员经常被要求更新程序以修复错误、添加功能并在新环境中运行。因此，您应该以使其更易于修改的方式来构造程序。软件工程师讨论*解耦*代码，以便程序的不同部分的更改互不影响。

为了简化解耦，已经开发出了一种称为*依赖注入*的技术。依赖注入是指您的代码应明确指定执行任务所需的功能。它比您想象的要古老得多；1996 年，Robert Martin 写了一篇名为[《依赖倒置原则》](https://oreil.ly/6HVob)的文章。

Go 隐式接口的一个令人惊讶的好处是，它们使依赖注入成为解耦代码的一种优秀方式。尽管其他语言的开发人员通常使用大型复杂的框架来注入其依赖关系，但事实是，在 Go 中很容易实现依赖注入，而无需任何额外的库。让我们通过一个简单的示例来看看如何使用隐式接口通过依赖注入来组合应用程序。

为了更好地理解这个概念，并了解如何在 Go 中实现依赖注入，您将构建一个非常简单的 Web 应用程序。（我将在《服务器》中更详细地讨论 Go 的内置 HTTP 服务器支持；可以将其视为预览。）首先编写一个小的实用函数，一个记录器：

```go
func LogOutput(message string) {
    fmt.Println(message)
}
```

应用程序还需要的另一件事是数据存储。让我们创建一个简单的数据存储：

```go
type SimpleDataStore struct {
    userData map[string]string
}

func (sds SimpleDataStore) UserNameForID(userID string) (string, bool) {
    name, ok := sds.userData[userID]
    return name, ok
}
```

还要定义一个工厂函数来创建`SimpleDataStore`的实例：

```go
func NewSimpleDataStore() SimpleDataStore {
    return SimpleDataStore{
        userData: map[string]string{
            "1": "Fred",
            "2": "Mary",
            "3": "Pat",
        },
    }
}
```

接下来，编写一些业务逻辑来查找用户并打招呼或告别。您的业务逻辑需要一些数据来进行工作，因此它需要一个数据存储。您还希望您的业务逻辑在调用时记录日志，因此它依赖于一个记录器。但是，您不希望强制它依赖于`LogOutput`或`SimpleDataStore`，因为以后您可能希望使用不同的记录器或数据存储。您的业务逻辑需要的是描述其依赖关系的接口：

```go
type DataStore interface {
    UserNameForID(userID string) (string, bool)
}

type Logger interface {
    Log(message string)
}
```

要使你的`LogOutput`函数符合此接口，你需要定义一个带有方法的函数类型：

```go
type LoggerAdapter func(message string)

func (lg LoggerAdapter) Log(message string) {
    lg(message)
}
```

令人惊讶的是，`LoggerAdapter`和`SimpleDataStore`恰好符合你的业务逻辑所需的接口，但是这两种类型都不知道自己符合这些接口。

现在你已经定义了依赖项，让我们来看看你的业务逻辑的实现：

```go
type SimpleLogic struct {
    l  Logger
    ds DataStore
}

func (sl SimpleLogic) SayHello(userID string) (string, error) {
    sl.l.Log("in SayHello for " + userID)
    name, ok := sl.ds.UserNameForID(userID)
    if !ok {
        return "", errors.New("unknown user")
    }
    return "Hello, " + name, nil
}

func (sl SimpleLogic) SayGoodbye(userID string) (string, error) {
    sl.l.Log("in SayGoodbye for " + userID)
    name, ok := sl.ds.UserNameForID(userID)
    if !ok {
        return "", errors.New("unknown user")
    }
    return "Goodbye, " + name, nil
}
```

你有一个带有两个字段的`struct`：一个是`Logger`，另一个是`DataStore`。`SimpleLogic`中没有提到具体类型，因此对它们没有依赖。如果以后从完全不同的提供者中交换新的实现，那就没有问题，因为提供者与你的接口没有关系。这与像 Java 这样的显式接口非常不同。尽管 Java 使用接口来解耦实现与接口，但显式接口将客户端和提供者绑定在一起。这使得在 Java（以及其他具有显式接口的语言）中替换依赖关系比在 Go 中更困难。

当你想要一个`SimpleLogic`实例时，你调用一个工厂函数，传递接口并返回一个结构体：

```go
func NewSimpleLogic(l Logger, ds DataStore) SimpleLogic {
    return SimpleLogic{
        l:    l,
        ds: ds,
    }
}
```

###### 注意

`SimpleLogic`中的字段是未导出的。这意味着它们只能被与`SimpleLogic`在同一包内的代码访问。虽然在 Go 语言中不能强制实现不可变性，但限制可以访问这些字段的代码可以减少其意外修改的可能性。我将在第十章中详细讨论导出和未导出标识符。

现在你要到达你的 API。你将只有一个端点`/hello`，它向提供用户 ID 的人打招呼。（请在你的真实应用程序中不要使用查询参数作为认证信息；这只是一个快速示例。）你的控制器需要业务逻辑来打招呼，所以你为此定义了一个接口：

```go
type Logic interface {
    SayHello(userID string) (string, error)
}
```

这个方法在你的`SimpleLogic`结构体上是可用的，但是再次强调，具体类型并不知道这个接口。此外，`SimpleLogic`上的另一个方法`SayGoodbye`不在接口中，因为你的控制器不关心它。接口是由客户端代码拥有的，因此它的方法集是根据客户端代码的需求定制的：

```go
type Controller struct {
    l     Logger
    logic Logic
}

func (c Controller) SayHello(w http.ResponseWriter, r *http.Request) {
    c.l.Log("In SayHello")
    userID := r.URL.Query().Get("user_id")
    message, err := c.logic.SayHello(userID)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        w.Write([]byte(err.Error()))
        return
    }
    w.Write([]byte(message))
}
```

就像你的其他类型有工厂函数一样，让我们为`Controller`编写一个工厂函数：

```go
func NewController(l Logger, logic Logic) Controller {
    return Controller{
        l:     l,
        logic: logic,
    }
}
```

同样，你接受接口并返回结构体。

最后，在`main`函数中连接所有组件并启动服务器：

```go
func main() {
    l := LoggerAdapter(LogOutput)
    ds := NewSimpleDataStore()
    logic := NewSimpleLogic(l, ds)
    c := NewController(l, logic)
    http.HandleFunc("/hello", c.SayHello)
    http.ListenAndServe(":8080", nil)
}
```

你可以在[第七章存储库](https://oreil.ly/qJQgV)的*sample_code/dependency_injection*目录中找到此应用程序的完整代码。

`main`函数是唯一知道所有具体类型的代码部分。如果你想要切换不同的实现，这是唯一需要改变的地方。通过依赖注入来外部化依赖项意味着您限制了随时间演变代码所需的更改。

依赖注入也是简化测试的一个很好的模式。这并不令人惊讶，因为编写单元测试实际上是在不同环境中重复使用你的代码，这个环境限制输入和输出以验证功能。例如，你可以通过注入一个能捕获日志输出并符合`Logger`接口的类型来验证测试中的日志输出。我会在第十五章详细讨论这一点。

###### 注意

行`http.HandleFunc("/hello", c.SayHello)`展示了我之前讨论的两点。

首先，你将`SayHello`方法视为一个函数。

其次，`http.HandleFunc`函数接收一个函数，并将其转换为`http.HandlerFunc`函数类型，它声明了一个方法以满足`http.Handler`接口，这个类型用于表示 Go 中的请求处理程序。你将一个类型的方法转换成另一个类型的方法，这非常巧妙。

# Wire

如果你觉得手写依赖注入代码太麻烦，可以使用[Wire](https://oreil.ly/Akwt_)，这是由 Google 编写的依赖注入辅助工具。Wire 利用代码生成来自动创建你在`main`中手动编写的具体类型声明。

# Go 不是特别面向对象（这很好）

现在你已经看过 Go 中类型的惯用用法，你会发现将 Go 归类为某种特定风格的语言是困难的。它显然不是严格的过程式语言。同时，Go 缺乏方法重写、继承或对象的特性，这也使它不是特别面向对象的语言。Go 有函数类型和闭包，但它也不是一种函数式语言。如果你试图强行将 Go 归入其中一个类别，结果会是非惯用的代码。

如果要给 Go 的风格贴上标签，最合适的词是*实用*。它从多个地方借鉴概念，主要目标是创建一种简单、可读性强且适合大团队长期维护的语言。

# 练习

在这些练习中，你将构建一个程序，利用你所学的关于类型、方法和接口的知识。答案可以在[第七章仓库](https://oreil.ly/qJQgV)的*exercise_solutions*目录中找到。

1.  你被要求管理一个篮球联赛，并准备编写一个程序来帮助你。定义两种类型。第一种称为`Team`，有一个用于球队名称和一个用于球员名称的字段。第二种类型称为`League`，有一个用于联赛中的球队的`Teams`字段和一个用于映射球队名称到胜场数的`Wins`字段。

1.  给`League`添加两个方法。第一个方法名为`MatchResult`，接受四个参数：第一支队伍的名称，其在比赛中的得分，第二支队伍的名称，以及其在比赛中的得分。此方法应更新`League`中的`Wins`字段。向`League`添加第二个方法名为`Ranking`，返回按胜场排序的球队名称切片。在你的程序中构建数据结构，并从`main`函数中调用这些方法，使用一些示例数据。

1.  定义一个名为`Ranker`的接口，它有一个名为`Ranking`的方法，返回一个字符串切片。编写一个名为`RankPrinter`的函数，有两个参数，第一个参数类型为`Ranker`，第二个参数类型为`io.Writer`。使用`io.WriteString`函数将`Ranker`返回的值写入`io.Writer`，每个结果之间用换行分隔。从`main`函数中调用此函数。

# 结束语

在本章中，您学习了有关类型、方法、接口及其最佳实践的知识。在下一章中，您将学习泛型，它通过允许您重用逻辑和使用不同类型的自定义容器来提高可读性和可维护性。

¹ 针对观众中的计算机科学家，我意识到子类型化并非继承。然而，大多数编程语言使用继承来实现子类型化，因此这些定义在流行使用中经常混淆。
