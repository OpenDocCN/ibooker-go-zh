# 第四章。块、遮蔽和控制结构

现在我已经讲解了变量、常量和内置类型，你已经准备好学习编程逻辑和组织了。我将首先解释块及其控制标识符何时可用。然后我会介绍 Go 语言的控制结构：`if`、`for`和`switch`。最后，我将讨论`goto`及其唯一应用场景。

# 块

在许多地方，Go 语言允许你声明变量。你可以在函数外部声明它们，作为函数的参数，以及在函数内部作为局部变量。

###### 注意

到目前为止，你只写了`main`函数，但在下一章节中你将编写带参数的函数。

每次声明发生的地方称为*块*。变量、常量、类型和在任何函数外声明的函数位于*包* 块中。你在程序中使用`import`语句来访问打印和数学函数（我将在第十章详细讨论它们）。它们为其他包定义了在包含`import`语句的文件中有效的名称。这些名称位于*文件* 块中。函数的顶级所有变量（包括函数的参数）都位于一个块中。在函数内部，每一组大括号（`{}`）定义另一个块，在稍后你将看到 Go 语言中的控制结构定义了它们自己的块。

你可以从任何外部块中访问在其中一个内部块中定义的标识符。这引出了一个问题：当你在一个包含块中有一个与标识符同名的声明时会发生什么？如果这样做，你会*遮蔽* 外部块中创建的标识符。

# 遮蔽变量

在我解释什么是遮蔽之前，让我们看一些代码（参见示例 4-1）。你可以在[The Go Playground](https://oreil.ly/50t6b)上运行它，或者在[第四章示例代码/遮蔽变量](https://oreil.ly/Fqw5B)目录中的仓库中运行它。

##### 示例 4-1。遮蔽变量

```go
func main() {
    x := 10
    if x > 5 {
        fmt.Println(x)
        x := 5
        fmt.Println(x)
    }
    fmt.Println(x)
}
```

Before you run this code, try to guess what it’s going to print out:

+   什么也不打印；代码无法编译

+   10 on line one, 5 on line two, 5 on line three

+   10 on line one, 5 on line two, 10 on line three

下面是发生的事情：

```go
10
5
10
```

*遮蔽变量* 是指与包含块中的变量同名的变量。只要遮蔽变量存在，就无法访问被遮蔽的变量。

在这种情况下，你几乎肯定不想在`if`语句内部创建一个全新的`x`。相反，你可能想要将 5 赋给函数块顶层声明的`x`。在`if`语句内部的第一个`fmt.Println`处，你可以访问函数块顶层声明的`x`。然而，在接下来的一行中，你通过在`if`语句体内部创建的块中*声明同名新变量*来*遮蔽*了`x`。在第二个`fmt.Println`处，当你访问名为`x`的变量时，你得到的是具有值 5 的遮蔽变量。`if`语句体的闭合括号结束了包含遮蔽`x`的块，在第三个`fmt.Println`处，当你访问名为`x`的变量时，你得到的是函数顶层声明的变量，其值为 10。注意，这个`x`并没有消失或被重新赋值；一旦在内部块中遮蔽了它，就无法访问它。

我在上一章提到过，在某些情况下，我避免使用`:=`，因为这可能会使使用的变量不清晰。这是因为在使用`:=`时很容易意外地遮蔽一个变量。记住，你可以使用`:=`同时创建和赋值多个变量。此外，左侧并不是所有变量都必须是新变量才能合法使用`:=`。让我们看另一个程序（见示例 4-2），你可以在[The Go Playground](https://oreil.ly/U_m4B)或[第四章代码库](https://oreil.ly/Fqw5B)的*sample_code/shadow_multiple_assignment*目录中找到它。

##### 示例 4-2\. 使用多重赋值进行遮蔽

```go
func main() {
    x := 10
    if x > 5 {
        x, y := 5, 20
        fmt.Println(x, y)
    }
    fmt.Println(x)
}
```

运行这段代码会得到以下结果：

```go
5 20
10
```

尽管外部块中存在`x`的定义，但`x`仍然在`if`语句内部被遮蔽了。这是因为`:=`只重新使用当前块中声明的变量。在使用`:=`时，请确保左侧没有来自外部作用域的变量，除非你打算遮蔽它们。

你还需要注意确保不要遮蔽导入的包。我会在第十章更详细地讨论导入包的内容，但你一直在导入`fmt`包来打印我们程序的结果。让我们看看当你在`main`函数内部声明一个名为`fmt`的变量时会发生什么，正如示例 4-3 所示。你可以在[The Go Playground](https://oreil.ly/CKQvm)或[第四章代码库](https://oreil.ly/Fqw5B)的*sample_code/shadow_package_names*目录中尝试运行它。

##### 示例 4-3\. 遮蔽包名

```go
func main() {
    x := 10
    fmt.Println(x)
    fmt := "oops"
    fmt.Println(fmt)
}
```

当你尝试运行这段代码时，会得到一个错误：

```go
fmt.Println undefined (type string has no field or method Println)
```

请注意问题不是你将变量命名为`fmt`，而是你尝试访问本地变量`fmt`没有的内容。一旦声明了本地变量`fmt`，它将屏蔽文件块中的`fmt`包，使得在`main`函数的其余部分中无法使用`fmt`包。

由于在某些情况下屏蔽是有用的（稍后章节将指出），`go vet`不会将其报告为可能的错误。在“使用代码质量扫描工具”中，您将学习有关可以检测代码中意外屏蔽的第三方工具。

# if

Go 中的`if`语句与大多数编程语言中的`if`语句非常相似。因为它是如此熟悉的结构，所以我在之前的示例代码中使用它时并不担心会造成困惑。示例 4-5 展示了一个更完整的示例。

##### 示例 4-5\. `if`和`else`

```go
n := rand.Intn(10)
if n == 0 {
    fmt.Println("That's too low")
} else if n > 5 {
    fmt.Println("That's too big:", n)
} else {
    fmt.Println("That's a good number:", n)
}
```

Go 中`if`语句与其他语言的最显著区别是你不需要在条件周围加括号。但 Go 在`if`语句中添加了另一个功能，帮助您更好地管理变量。

正如我在“变量屏蔽”中所讨论的，任何在`if`或`else`语句的大括号内声明的变量仅存在于该块中。这并不罕见；大多数语言都是如此。Go 增加的是能够声明仅在条件和`if`以及`else`块中有效的变量。看看示例 4-6，它重新编写了我们先前的示例以使用这种作用域。

##### 示例 4-6\. 将变量作用域限制为`if`语句

```go
if n := rand.Intn(10); n == 0 {
    fmt.Println("That's too low")
} else if n > 5 {
    fmt.Println("That's too big:", n)
} else {
    fmt.Println("That's a good number:", n)
}
```

拥有这种特殊作用域非常方便。它允许你创建仅在需要时可用的变量。一旦`if/else`语句系列结束，`n`就变得未定义。你可以尝试在示例 4-7 中的[Go Playground](https://oreil.ly/rz671)或[第四章代码库](https://oreil.ly/Fqw5B)中的*sample_code/if_bad_scope*目录运行此代码来测试。

##### 示例 4-7\. 超出范围…​

```go
if n := rand.Intn(10); n == 0 {
    fmt.Println("That's too low")
} else if n > 5 {
    fmt.Println("That's too big:", n)
} else {
    fmt.Println("That's a good number:", n)
}
fmt.Println(n)
```

尝试运行此代码会产生编译错误：

```go
undefined: n
```

###### 注意

技术上，您可以在`if`语句中的比较之前放置任何*简单语句*，包括不返回值的函数调用或将新值分配给现有变量。但不要这样做。只使用此功能定义仅限于`if/else`语句的新变量；其他任何操作都可能会令人困惑。

还要注意，就像任何其他块一样，作为`if`语句的一部分声明的变量将屏蔽包含块中具有相同名称的变量。

# 四种`for`

与 C 语言家族中的其他语言一样，Go 使用`for`语句进行循环。使 Go 与其他语言不同的是，`for`是该语言中*唯一*的循环关键字。Go 通过四种格式使用`for`关键字来实现这一点：

+   完整的，类似 C 风格的`for`

+   仅有条件的`for`

+   一个无限的`for`

+   `for-range`

## 完整的`for`语句

第一个 `for` 循环风格是你可能从 C、Java 或 JavaScript 中熟悉的完整 `for` 声明，如 示例 4-8 所示。

##### 示例 4-8\. 一个完整的 `for` 语句

```go
for i := 0; i < 10; i++ {
    fmt.Println(i)
}
```

你可能对发现这个程序打印出从 0 到 9 的数字感到不惊讶。

就像 `if` 语句一样，`for` 语句在其各部分周围不使用括号。否则，它看起来应该很熟悉。`if` 语句有三个部分，用分号分隔。第一部分是初始化，在循环开始前设置一个或多个变量。关于初始化部分，你应该记住两个重要的细节。首先，你必须使用 `:=` 来初始化变量；在这里不合法使用 `var`。其次，就像 `if` 语句中的变量声明一样，你可以在这里遮蔽一个变量。

第二部分是比较。这必须是一个评估为 `bool` 的表达式。它在每次循环迭代之前立即检查。如果表达式评估为 `true`，则执行循环体。

标准 `for` 语句的最后部分是增量。通常你会在这里看到像 `i++` 这样的东西，但任何赋值都是有效的。它在每次循环迭代之后立即运行，然后再评估条件。

Go 允许你省略 `for` 语句的三个部分中的一个或多个。最常见的情况是，如果它是基于循环前计算的值：

```go
i := 0
for ; i < 10; i++ {
    fmt.Println(i)
}
```

或者你会因为在循环内有更复杂的增量规则而省略增量：

```go
for i := 0; i < 10; {
    fmt.Println(i)
    if i % 2 == 0 {
        i++
    } else {
        i+=2
    }
}
```

## 仅有条件的 `for` 语句

当你在 `for` 语句中省略初始化和增量时，请不要包含分号。（如果你这样做，`go fmt` 会将它们删除。）这留下了一个像在 C、Java、JavaScript、Python、Ruby 和许多其他语言中找到的 `while` 语句一样工作的 `for` 语句。看起来像 示例 4-9。

##### 示例 4-9\. 仅有条件的 `for` 语句

```go
i := 1
for i < 100 {
        fmt.Println(i)
        i = i * 2
}
```

## 无限 `for` 语句

第三种 `for` 语句格式也取消了条件。Go 有一个版本的 `for` 循环，永远循环。如果你在 1980 年代学习编程，你的第一个程序可能是在 BASIC 中的一个无限循环，永远打印 `HELLO` 到屏幕上：

```go
10 PRINT "HELLO"
20 GOTO 10
```

示例 4-10 展示了这个程序的 Go 版本。你可以在本地运行它，或者在 [The Go Playground](https://oreil.ly/whOi-) 或 [第四章存储库](https://oreil.ly/Fqw5B) 的 *sample_code/infinite_for* 目录中试用它。

##### 示例 4-10\. 无限循环怀旧

```go
package main

import "fmt"

func main() {
    for {
        fmt.Println("Hello")
    }
}
```

运行这个程序会给你同样的输出，这个输出填满了数百万台 Commodore 64 和 Apple ] 的屏幕：

```go
Hello
Hello
Hello
Hello
Hello
Hello
Hello
...
```

当你厌倦了走在记忆的长廊上时，请按 Ctrl-C。

###### 注意

如果在 The Go Playground 上运行 [示例 4-10，你会发现它会在几秒钟后停止执行。作为共享资源，Playground 不允许任何一个程序运行太长时间。

## break 和 continue

如何在不使用键盘或关闭计算机的情况下退出无限`for`循环？这是`break`语句的任务。它立即退出循环，就像其他语言中的`break`语句一样。当然，你可以在任何`for`语句中使用`break`，不仅限于无限`for`语句。

###### 注意

在 Java、C 和 JavaScript 中，Go 没有等同于`do`关键字。如果你想至少迭代一次，最简洁的方法是使用以`if`语句结束的无限`for`循环。例如，如果你有一些 Java 代码使用了`do/while`循环：

```go
do {
    // things to do in the loop
} while (CONDITION);
```

Go 版本如下：

```go
for {
    // things to do in the loop
    if !CONDITION {
        break
    }
}
```

注意条件前面有一个`!`用于*否定*Java 代码中的条件。Go 代码指定如何*退出*循环，而 Java 代码则指定如何*保持*在循环中。

Go 还包括`continue`关键字，它跳过`for`循环的其余部分，直接进入下一次迭代。技术上，你不需要`continue`语句。你可以像示例 4-11 那样编写代码。

##### 示例 4-11\. 令人困惑的代码

```go
for i := 1; i <= 100; i++ {
    if i%3 == 0 {
        if i%5 == 0 {
            fmt.Println("FizzBuzz")
        } else {
            fmt.Println("Fizz")
        }
    } else if i%5 == 0 {
        fmt.Println("Buzz")
    } else {
        fmt.Println(i)
    }
}
```

但这并不是惯用写法。Go 鼓励短小的`if`语句体，尽可能左对齐。嵌套代码更难以跟踪。使用`continue`语句使理解正在发生的事情变得更容易。示例 4-12 展示了从前面示例中重写为使用`continue`的代码。

##### 示例 4-12\. 使用`continue`使代码更清晰

```go
for i := 1; i <= 100; i++ {
    if i%3 == 0 && i%5 == 0 {
        fmt.Println("FizzBuzz")
        continue
    }
    if i%3 == 0 {
        fmt.Println("Fizz")
        continue
    }
    if i%5 == 0 {
        fmt.Println("Buzz")
        continue
    }
    fmt.Println(i)
}
```

如你所见，用一系列使用`continue`语句的`if`语句来替换`if/else`语句链，可以使条件对齐。这改善了条件的布局，意味着你的代码更易读和理解。

## `for-range`语句

第四个`for`语句格式用于遍历一些 Go 内置类型中的元素。它被称为`for-range`循环，类似于其他语言中的迭代器。本节展示如何在字符串、数组、切片和映射中使用`for-range`循环。当我在第十二章中讲解通道时，我会讨论如何在其中使用`for-range`循环。

###### 注意

你只能用`for-range`循环来迭代内置复合类型和基于它们的用户定义类型。

首先，让我们看看如何在切片中使用`for-range`循环。你可以在示例 4-13 中尝试这段代码，也可以在[Go Playground](https://oreil.ly/XwuTL)或[第四章代码库](https://oreil.ly/Fqw5B)的*sample_code/for_range*目录中的*main.go*中的`forRangeKeyValue`函数中进行尝试。

##### 示例 4-13\. `for-range`循环

```go
evenVals := []int{2, 4, 6, 8, 10, 12}
for i, v := range evenVals {
    fmt.Println(i, v)
}
```

运行此代码将产生以下输出：

```go
0 2
1 4
2 6
3 8
4 10
5 12
```

使 `for-range` 循环有趣的是你会得到两个循环变量。第一个变量是正在迭代的数据结构中的位置，而第二个变量是该位置上的值。这两个循环变量的惯用名称取决于正在循环的内容。当遍历数组、切片或字符串时，通常使用 `i` 作为 *index*。当遍历映射时，则使用 `k`（表示 *key*）。

第二个变量通常称为 `v`，表示 *value*，但有时会根据正在迭代的值的类型命名。当然，你可以给变量任意名称。如果循环体只包含几条语句，单字母变量名效果很好。对于更长（或嵌套的）循环，应使用更具描述性的名称。

如果你在 `for-range` 循环中不需要使用第一个变量怎么办？请记住，Go 要求你访问所有声明的变量，这条规则也适用于 `for` 循环中声明的变量。如果你不需要访问键，可以使用下划线 (`_`) 作为变量名。这告诉 Go 忽略该值。

让我们重写切片遍历代码，不打印位置信息。你可以在 示例 4-14 中运行代码，也可以在 [Go Playground](https://oreil.ly/2fO12) 上运行，或者在 [第四章代码库](https://oreil.ly/Fqw5B) 的 *sample_code/for_range* 目录中的 *main.go* 中的 `forRangeIgnoreKey` 函数中运行。

##### 示例 4-14\. 在 `for-range` 循环中忽略切片索引

```go
evenVals := []int{2, 4, 6, 8, 10, 12}
for _, v := range evenVals {
    fmt.Println(v)
}
```

运行此代码将产生以下输出：

```go
2
4
6
8
10
12
```

###### 提示

在任何返回值但要忽略它的情况下，请使用下划线来隐藏值。当我在 第五章 谈到函数和 第十章 谈到包时，你会再次看到下划线的模式。

如果你想要键而不想要值怎么办？在这种情况下，Go 允许你只留下第二个变量。这是有效的 Go 代码：

```go
uniqueNames := map[string]bool{"Fred": true, "Raul": true, "Wilma": true}
for k := range uniqueNames {
    fmt.Println(k)
}
```

遍历键的最常见原因是在映射用作集时。在这些情况下，值并不重要。然而，当遍历数组或切片时，也可以省略值。这种情况很少见，因为遍历线性数据结构的常见原因是访问数据。如果你发现自己在数组或切片上使用此格式，很可能选择了错误的数据结构，应考虑重构。

###### 注意

当你在 第十二章 看 channels 时，你会看到 `for-range` 循环每次迭代只返回单个值的情况。

### 遍历映射

`for-range` 循环遍历映射的方式非常有趣。你可以在 示例 4-15 中运行代码，也可以在 [Go Playground](https://oreil.ly/VplnA) 上运行，或者在 [第四章代码库](https://oreil.ly/Fqw5B) 的 *sample_code/iterate_map* 目录中运行。

##### 示例 4-15\. Map 迭代顺序变化

```go
m := map[string]int{
    "a": 1,
    "c": 3,
    "b": 2,
}

for i := 0; i < 3; i++ {
    fmt.Println("Loop", i)
    for k, v := range m {
        fmt.Println(k, v)
    }
}
```

当你构建并运行这个程序时，输出会有所不同。以下是一种可能性：

```go
Loop 0
c 3
b 2
a 1
Loop 1
a 1
c 3
b 2
Loop 2
b 2
a 1
c 3
```

键和值的顺序变化；某些运行可能是相同的。这实际上是一个安全功能。在早期的 Go 版本中，如果你向 map 中插入相同的项，键的迭代顺序通常（但不总是）是相同的。这导致了两个问题：

+   人们可能编写假设顺序固定的代码，但这些代码会在奇怪的时候出错。

+   如果 map 总是将项哈希到完全相同的值，并且你知道服务器在 map 中存储用户数据，你可以通过发送具有所有哈希到同一个桶的密钥的特殊制作数据来减慢服务器，这称为 *Hash DoS* 攻击。

为了防止这两个问题，Go 团队对 map 实现进行了两个更改。首先，他们修改了 map 的哈希算法，以包括每次创建 map 变量时生成的随机数。其次，他们使得 `for-range` 在 map 上的迭代顺序每次都有所变化。这两个变化使得实现 Hash DoS 攻击变得更加困难。

###### 注意

这个规则有一个例外。为了更容易调试和记录 map，格式化函数（如 `fmt.Println`）总是按升序输出它们的键。

### 迭代字符串

如前所述，你也可以使用 `for-range` 循环来处理字符串。让我们来看看。你可以在你的计算机上或者在[The Go Playground](https://oreil.ly/C3LRy)或在[第四章仓库](https://oreil.ly/Fqw5B)的*sample_code/iterate_string*目录中运行示例 4-16 中的代码。

##### 示例 4-16\. 迭代字符串

```go
samples := []string{"hello", "apple_π!"}
for _, sample := range samples {
    for i, r := range sample {
        fmt.Println(i, r, string(r))
    }
    fmt.Println()
}
```

当代码迭代单词“hello”时，输出没有什么意外：

```go
0 104 h
1 101 e
2 108 l
3 108 l
4 111 o
```

在第一列是索引；第二列是字母的数值；第三列是转换为字符串的字母类型的数值。

查看“apple_π!”的结果更有趣：

```go
0 97 a
1 112 p
2 112 p
3 108 l
4 101 e
5 95 _
6 960 π
8 33 !
```

注意此输出的两个问题。首先，注意第一列跳过了编号 7\. 其次，位置 6 的值是 960\. 这远远超过了一个字节可以容纳的范围。但是在第三章中，你看到字符串是由字节组成的。这是怎么回事？

当你用 `for-range` 循环迭代字符串时，会出现特殊的行为。它迭代的是 *runes* 而不是 *bytes*。每当 `for-range` 循环遇到字符串中的多字节 rune 时，它将 UTF-8 表示转换为一个单一的 32 位数值并赋给该值。偏移量按 rune 的字节长度增加。如果 `for-range` 循环遇到一个不能表示有效 UTF-8 值的字节，则返回 Unicode 替换字符（十六进制值为 0xfffd）。

###### 提示

使用 `for-range` 循环按顺序访问字符串中的字符。第一个变量保存从字符串开头的字节数，但第二个变量的类型是 rune。

### `for-range` 的值是一个副本

你应该注意，每次 `for-range` 循环迭代你的复合类型时，它会将复合类型的值复制到值变量中。*修改值变量不会修改复合类型中的值*。示例 4-17 展示了一个快速演示这一点的程序。你可以在 [The Go Playground](https://oreil.ly/ShwR0) 或 [第四章代码库](https://oreil.ly/Fqw5B) 的 *sample_code/for_range* 目录中的 `forRangeIsACopy` 函数中尝试。

##### 示例 4-17\. 修改值不会修改源

```go
evenVals := []int{2, 4, 6, 8, 10, 12}
for _, v := range evenVals {
    v *= 2
}
fmt.Println(evenVals)
```

运行此代码会得到以下输出：

```go
[2 4 6 8 10 12]
```

在 Go 1.22 版本之前，值变量只会被创建一次，并在 `for` 循环的每次迭代中重用。从 Go 1.22 开始，默认行为是在 `for` 循环的每次迭代中创建一个新的索引和值变量。这个变更可能看起来不重要，但它可以防止一个常见的 bug。当我在 “Goroutines, for Loops, and Varying Variables” 中讨论 goroutines 和 `for-range` 循环时，你会看到在 Go 1.22 之前，如果在 `for-range` 循环中启动 goroutines，你需要小心如何传递索引和值给 goroutines，否则可能会得到意外的结果。

因为这是一个向后兼容性的变更（即使是消除了一个常见的错误），你可以通过在你的模块的 *go.mod* 文件中指定 Go 版本来控制是否启用此行为。我在 “Using go.mod” 中详细讨论了这个问题。

与 `for` 语句的其他三种形式一样，你可以在 `for-range` 循环中使用 `break` 和 `continue`。

## 给你的 for 语句加上标签

默认情况下，`break` 和 `continue` 关键字适用于直接包含它们的 `for` 循环。如果你有嵌套的 `for` 循环，并且想要退出或跳过外部循环的迭代器，该怎么办？我们来看一个例子。你将修改早期的字符串迭代程序，在字符串中第一次遇到字母“l”时停止迭代。你可以在 示例 4-18 中运行这段代码，也可以在 [The Go Playground](https://oreil.ly/ToDkq) 或 [第四章代码库](https://oreil.ly/Fqw5B) 的 *sample_code/for_label* 目录中运行。

##### 示例 4-18\. 标签

```go
func main() {
    samples := []string{"hello", "apple_π!"}
outer:
    for _, sample := range samples {
        for i, r := range sample {
            fmt.Println(i, r, string(r))
            if r == 'l' {
                continue outer
            }
        }
        fmt.Println()
    }
}
```

注意，标签 `outer` 被 `go fmt` 缩进到与周围函数相同的级别。标签总是缩进到与块的大括号相同的级别，这样更容易注意到。运行程序会得到以下输出：

```go
0 104 h
1 101 e
2 108 l
0 97 a
1 112 p
2 112 p
3 108 l
```

带标签的嵌套 `for` 循环很少见。它们最常用于实现类似以下伪代码的算法：

```go
outer:
    for _, outerVal := range outerValues {
        for _, innerVal := range outerVal {
            // process innerVal
            if invalidSituation(innerVal) {
                continue outer
            }
        }
        // here we have code that runs only when all of the
        // innerVal values were successfully processed
    }
```

## 选择正确的 for 语句

现在我已经覆盖了所有形式的`for`语句，你可能想知道何时使用哪种格式。大多数情况下，你会使用`for-range`格式。`for-range`循环是遍历字符串的最佳方式，因为它能正确地返回符文而不是字节。你还看到了`for-range`循环在遍历切片和映射时表现良好，并且在第十二章中你将看到`for-range`与通道的自然结合。

###### 提示

在遍历所有内置复合类型实例的内容时，最好使用`for-range`循环。它避免了在使用数组、切片或映射时所需的大量样板代码。

当应该使用完整的`for`循环？最佳场景是在复合类型中不从第一个元素到最后一个元素进行迭代时。虽然你可以在`for-range`循环中使用一些`if`、`continue`和`break`的组合，但标准的`for`循环更清晰地表示你的迭代的起始和结束。比较下面这两段代码片段，它们都是在数组中从第二个到倒数第二个元素进行迭代。首先是`for-range`循环：

```go
evenVals := []int{2, 4, 6, 8, 10}
for i, v := range evenVals {
    if i == 0 {
        continue
    }
    if i == len(evenVals)-1 {
        break
    }
    fmt.Println(i, v)
}
```

下面是相同的代码，使用了标准的`for`循环：

```go
evenVals := []int{2, 4, 6, 8, 10}
for i := 1; i < len(evenVals)-1; i++ {
    fmt.Println(i, evenVals[i])
}
```

标准的`for`循环代码既更短又更易于理解。

###### 警告

这种模式无法跳过字符串的开头。记住，标准的`for`循环不能正确处理多字节字符。如果你想跳过字符串中的某些符文，你需要使用`for-range`循环，这样它才能正确地为你处理符文。

另外两种`for`语句格式使用较少。仅条件的`for`循环就像替代它的`while`循环一样，在基于计算值进行循环时非常有用。

无限的`for`循环在某些情况下非常有用。`for`循环的主体应始终包含`break`或`return`，因为你很少希望无限循环。现实世界的程序应限制迭代，并在无法完成操作时优雅地失败。如前所示，无限的`for`循环可以与`if`语句结合使用，以模拟其他语言中存在的`do`语句。无限的`for`循环还用于实现一些版本的*迭代器*模式，在我回顾标准库时你将会看到“io and Friends”。

# 开关

像许多 C 衍生语言一样，Go 语言有`switch`语句。在这些语言中，大多数开发者避免使用`switch`语句，因为其对可以切换的值有限制，并且有默认的穿透行为。但是 Go 语言不同，它使得`switch`语句变得很有用。

###### 注意

对于那些熟悉 Go 语言的读者，我将在本章中介绍*表达式 switch*语句。当我在第七章讨论接口时，我将讨论*类型 switch*语句。

乍一看，在 Go 中，`switch` 语句看起来并不比在 C/C++、Java 或 JavaScript 中的样子有多大的不同，但其中还是有些意外的地方。让我们看一个 `switch` 语句的示例。你可以在 Example 4-19 中的代码或在 [第四章仓库](https://oreil.ly/Fqw5B) 中的 *sample_code/switch* 目录下的 `main.go` 文件中的 `basicSwitch` 函数中运行这段代码。

##### 示例 4-19\. `switch` 语句

```go
words := []string{"a", "cow", "smile", "gopher",
    "octopus", "anthropologist"}
for _, word := range words {
    switch size := len(word); size {
    case 1, 2, 3, 4:
        fmt.Println(word, "is a short word!")
    case 5:
        wordLen := len(word)
        fmt.Println(word, "is exactly the right length:", wordLen)
    case 6, 7, 8, 9:
    default:
        fmt.Println(word, "is a long word!")
    }
}
```

当你运行这段代码时，会得到以下输出：

```go
a is a short word!
cow is a short word!
smile is exactly the right length: 5
anthropologist is a long word!
```

我将详细介绍 `switch` 语句的特性以解释其输出。和 `if` 语句一样，在 `switch` 中，你不需要在比较的值周围加上括号。同样如 `if` 语句一样，你可以在 `switch` 语句中声明一个在所有分支中都可见的变量。在这种情况下，你将变量 `size` 的作用域限定在 `switch` 语句的所有 `case` 中。

所有的 `case` 子句（和可选的 `default` 子句）都包含在一对大括号内。但请注意，你不需要在 `case` 子句的内容周围加上大括号。你可以在 `case`（或 `default`）子句内有多行代码，它们都被认为是同一个代码块的一部分。

在 `case 5:` 内，你声明了一个新变量 `wordLen`。由于这是一个新的代码块，你可以在其中声明新的变量。和其他任何代码块一样，`case` 子句的代码块内声明的变量只在该代码块内可见。

如果你习惯在 `switch` 语句中的每个 `case` 结尾放置 `break` 语句，那么你会高兴地注意到它们已经消失了。在 Go 中，默认情况下，`switch` 语句中的 `case` 不会穿透。这与 Ruby 或者（如果你是老派程序员的话）Pascal 的行为更加一致。

这引出了一个问题：如果 `case` 不再穿透，那么如果有多个值应该触发相同的逻辑，你该怎么办？在 Go 中，你用逗号分隔多个匹配值，就像匹配 1、2、3 和 4 或者 6、7、8 和 9 一样。这就是为什么对于 `a` 和 `cow` 你会得到相同的输出。

这就引出了下一个问题：如果你没有了穿透，而且你有一个空的 `case`（就像在样例程序中当你的参数长度是 6、7、8 或 9 个字符时一样），会发生什么？在 Go 中，*空的 `case` 意味着什么也不会发生*。这就是为什么当你使用 *octopus* 或 *gopher* 作为参数时，程序不会输出任何内容。

###### 提示

出于完整性考虑，Go 确实包含了 `fallthrough` 关键字，它允许一个 `case` 继续执行下一个 `case`。在使用 `fallthrough` 的算法之前，请三思。如果你发现自己需要使用 `fallthrough`，尝试重构你的逻辑以消除 `case` 之间的依赖关系。

在示例程序中，您正在根据整数值进行`switch`，但这并不是您可以做的全部。您可以对任何可以使用`==`进行比较的类型进行`switch`，其中包括除切片、映射、通道、函数和包含这些类型字段的结构体之外的所有内置类型。

即使在每个`case`子句的末尾不需要放置`break`语句，但当您希望从`case`中早期退出时，可以使用它们。但是，需要`break`语句可能表明您正在做某些过于复杂的事情。考虑重构您的代码以去除它。

还有一个地方可能会使用`break`语句在`switch`语句的`case`中。如果您在`for`循环内部有一个`switch`语句，并且希望跳出`for`循环，请在`for`语句上放置一个标签，并在`break`上放置标签的名称。如果不使用标签，Go 会假设您想跳出`case`。让我们看一个快速示例，您希望一旦达到 7 就跳出`for`循环。您可以在 Example 4-20 中运行代码，在[The Go Playground](https://oreil.ly/xH2Ij)或在[第四章存储库](https://oreil.ly/Fqw5B)的*sample_code/switch*目录中的*main.go*文件中的`missingLabel`函数中运行它。

##### 示例 4-20\. 缺少标签的情况

```go
func main() {
    for i := 0; i < 10; i++ {
        switch i {
        case 0, 2, 4, 6:
            fmt.Println(i, "is even")
        case 3:
            fmt.Println(i, "is divisible by 3 but not 2")
        case 7:
            fmt.Println("exit the loop!")
            break
        default:
            fmt.Println(i, "is boring")
        }
    }
}
```

运行此代码会产生以下输出：

```go
0 is even
1 is boring
2 is even
3 is divisible by 3 but not 2
4 is even
5 is boring
6 is even
exit the loop!
8 is boring
9 is boring
```

这不是预期的行为。目标是在遇到 7 时跳出`for`循环，但`break`被解释为退出`case`。要解决此问题，您需要引入一个标签，就像在跳出嵌套的`for`循环时所做的那样。首先，给`for`语句加上标签：

```go
loop:
    for i := 0; i < 10; i++ {
```

然后您可以在您的`break`上使用标签：

```go
break loop
```

您可以在[The Go Playground](https://oreil.ly/PeaBI)上查看这些更改，或者在[第四章存储库](https://oreil.ly/Fqw5B)中的*sample_code/switch*目录中的*main.go*文件中的`labeledBreak`函数中查看它。再次运行时，您将获得预期的输出：

```go
0 is even
1 is boring
2 is even
3 is divisible by 3 but not 2
4 is even
5 is boring
6 is even
exit the loop!
```

## 空白`switch`

您可以以另一种更强大的方式使用`switch`语句。正如 Go 允许您从`for`语句的声明中省略部分内容一样，您可以编写一个不指定要比较的值的`switch`语句。这被称为*空白`switch`*。常规的`switch`只允许您检查相等的值。空白`switch`允许您对每个`case`使用任何布尔比较。您可以在 Example 4-21 中尝试这段代码，在[The Go Playground](https://oreil.ly/v7qI5)或在[第四章存储库](https://oreil.ly/Fqw5B)的*sample_code/blank_switch*目录中的*main.go*文件中的`basicBlankSwitch`函数中。

##### 示例 4-21\. 空白的`switch`

```go
words := []string{"hi", "salutations", "hello"}
for _, word := range words {
    switch wordLen := len(word); {
    case wordLen < 5:
        fmt.Println(word, "is a short word!")
    case wordLen > 10:
        fmt.Println(word, "is a long word!")
    default:
        fmt.Println(word, "is exactly the right length.")
    }
}
```

当您运行此程序时，会得到以下输出：

```go
hi is a short word!
salutations is a long word!
hello is exactly the right length.
```

就像普通的`switch`语句一样，你可以选择在空的`switch`中包含一个短变量声明。但与普通的`switch`不同的是，你可以为你的情况编写逻辑测试。空的`switch`很酷，但不要过度使用。如果你发现自己写了一个所有情况都是对同一变量的相等比较的空`switch`：

```go
switch {
case a == 2:
    fmt.Println("a is 2")
case a == 3:
    fmt.Println("a is 3")
case a == 4:
    fmt.Println("a is 4")
default:
    fmt.Println("a is ", a)
}
```

你应该用一个表达式`switch`语句替换它：

```go
switch a {
case 2:
    fmt.Println("a is 2")
case 3:
    fmt.Println("a is 3")
case 4:
    fmt.Println("a is 4")
default:
    fmt.Println("a is ", a)
}
```

## 在`if`和`switch`之间的选择

就功能而言，一系列`if/else`语句和空的`switch`语句之间没有太大区别。两者都允许一系列比较。那么，何时应该使用`switch`，何时应该使用一组`if`或`if/else`语句呢？`switch`语句，即使是空的`switch`，表明每个`case`中的值或比较之间存在关系。为了演示清晰度的差异，可以使用空的`switch`重新编写 FizzBuzz 程序（参见示例 4-22）。你也可以在[第四章代码库](https://oreil.ly/Fqw5B)的*sample_code/simplest_fizzbuzz*目录中找到这段代码。

##### 示例 4-22。用空的`switch`重写一系列`if`语句

```go
for i := 1; i <= 100; i++ {
    switch {
    case i%3 == 0 && i%5 == 0:
        fmt.Println("FizzBuzz")
    case i%3 == 0:
        fmt.Println("Fizz")
    case i%5 == 0:
        fmt.Println("Buzz")
    default:
        fmt.Println(i)
    }
}
```

大多数人都会同意这是最可读的版本。不再需要`continue`语句，`default`情况下的默认行为也变得明确。

当然，在 Go 语言中，没有任何东西能阻止你在空`switch`的每个`case`上做各种无关的比较。然而，这并不符合习惯用法。如果你发现自己处于这样一种情况下，应该使用一系列`if/else`语句（或者考虑重构你的代码）。

###### 提示

当你有多个相关的情况时，更倾向于使用空的`switch`语句而不是`if/else`链。使用`switch`可以使比较更加明显，并强调它们是一组相关的问题。

# `goto`——是的，`goto`

Go 语言有第四种控制语句，但很可能你永远不会使用它。自从 Edsger Dijkstra 在 1968 年写下[“谈谈`goto`语句的害处”](https://oreil.ly/YK2tl)以来，`goto`语句一直是编码家族中的害群之马。这有充分的理由。传统上，`goto`很危险，因为它可以跳转到程序中的几乎任何地方；你可以跳进或跳出循环，跳过变量定义，或者跳入`if`语句中一组语句的中间。这使得理解使用`goto`的程序变得困难。

大多数现代语言不包括`goto`。然而 Go 语言有一个`goto`语句。你仍然应该尽量避免使用它，但它有一些用途，而 Go 语言对它施加的限制使其更适合结构化编程。

在 Go 语言中，`goto`语句指定了一个带标签的代码行，并且执行跳转到它。但你不能随处跳转。Go 语言禁止跳过变量声明和进入内部或并行块的跳转。

示例 4-23 中的程序展示了两个非法的`goto`语句。你可以尝试在[Go Playground](https://oreil.ly/l016p)或[第四章存储库](https://oreil.ly/Fqw5B)的*sample_code/broken_goto*目录中运行它。

##### Example 4-23\. Go 的`goto`有规则。

```go
func main() {
    a := 10
    goto skip
    b := 20
skip:
    c := 30
    fmt.Println(a, b, c)
    if c > a {
        goto inner
    }
    if a < b {
    inner:
        fmt.Println("a is less than b")
    }
}
```

尝试运行这个程序会产生以下错误：

```go
goto skip jumps over declaration of b at ./main.go:8:4
goto inner jumps into block starting at ./main.go:15:11
```

那么`goto`应该用在什么地方？大多数情况下，不应该用。带标签的`break`和`continue`语句允许您跳出深度嵌套的循环或跳过迭代。示例 4-24 中的程序具有合法的`goto`，展示了少数几个有效使用场景之一。你还可以在[第四章存储库](https://oreil.ly/Fqw5B)的*sample_code/good_goto*目录中找到这段代码。

##### 示例 4-24\. 使用`goto`的理由。

```go
func main() {
    a := rand.Intn(10)
    for a < 100 {
        if a%5 == 0 {
            goto done
        }
        a = a*2 + 1
    }
    fmt.Println("do something when the loop completes normally")
done:
    fmt.Println("do complicated stuff no matter why we left the loop")
    fmt.Println(a)
}
```

这个例子是人为构造的，但它展示了`goto`如何使程序更清晰。在这个简单的情况下，有一些逻辑你不想在函数中间运行，但你希望在函数结束时运行。有方法可以在没有`goto`的情况下做到这一点。你可以设置一个布尔标志或在`for`循环之后复制复杂的代码，而不是使用`goto`，但这两种方法都有缺点。在你的代码中满布布尔标志以控制逻辑流的功能与`goto`语句几乎相同，只是更冗长。复制复杂的代码是有问题的，因为它使得你的代码更难维护。这些情况很少见，但如果你找不到重构逻辑的方法，像这样使用`goto`实际上会改进你的代码。

如果你想看一个实际的例子，你可以看一下标准库中`strconv`包的文件*atof.go*中的`floatBits`方法。这个方法太长了，无法完全包含，但方法以这段代码结束：

```go
overflow:
    // ±Inf
    mant = 0
    exp = 1<<flt.expbits - 1 + flt.bias
    overflow = true

out:
    // Assemble bits.
    bits := mant & (uint64(1)<<flt.mantbits - 1)
    bits |= uint64((exp-flt.bias)&(1<<flt.expbits-1)) << flt.mantbits
    if d.neg {
        bits |= 1 << flt.mantbits << flt.expbits
    }
    return bits, overflow
```

在这些行之前，有几个条件检查。有些需要运行`overflow`标签后的代码，而其他条件则需要跳过该代码直接到`out`。根据条件，有`goto`语句跳转到`overflow`或`out`。你可能可以想出一种避免`goto`语句的方法，但它们都会使代码更难理解。

###### 提示

你应该尽力避免使用`goto`。但在它使你的代码更易读的罕见情况下，它是一个选项。

# 练习

现在是时候应用你在 Go 中学到的所有关于控制结构和块的知识了。你可以在[第四章存储库](https://oreil.ly/Fqw5B)中找到这些练习的答案。

1.  编写一个`for`循环，将 100 个 0 到 100 之间的随机数放入一个`int`切片中。

1.  循环遍历你在练习 1 中创建的切片。对于切片中的每个值，应用以下规则：

    1.  如果值可被 2 整除，打印“二！”

    1.  如果值可被 3 整除，打印“三！”

    1.  如果值可同时被 2 和 3 整除，打印“六！”。不打印其他任何内容。

    1.  否则，打印“不要紧”。

1.  开始一个新程序。在`main`函数中声明一个名为`total`的`int`变量。编写一个`for`循环，使用名为`i`的变量从 0（包含）迭代到 10（不包含）。`for`循环的主体应如下所示：

    ```go
    total := total + i
    fmt.Println(total)
    ```

    在`for`循环结束后，打印出`total`的值。会打印出什么？这段代码中可能存在的 bug 是什么？

# 总结

本章覆盖了许多编写符合惯例的 Go 语言的重要主题。你已经学习了关于代码块、变量屏蔽和控制结构的知识，并学会了如何正确使用它们。此时，你已经能够编写简单的 Go 程序，适合在`main`函数中运行。是时候转向更大的程序了，使用函数来组织你的代码。
