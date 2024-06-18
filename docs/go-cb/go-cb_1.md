# 第二章。字符串示例

# 2.0 介绍

字符串操作是任何编程语言中最常见的活动之一。大多数程序以某种方式处理文本，无论是直接与用户交互还是机器之间的通信。文本可能是我们拥有的最接近通用媒介的东西，而作为数据的字符串则无处不在。作为程序员，能够操作字符串是你的必备技能之一。

Go 有几个用于字符串操作的包。`strconv` 包专注于转换到或从字符串的操作。`fmt` 包提供使用动词作为替换来格式化字符串的函数，类似于 C 语言。`unicode/utf8` 和 `unicode/utf16` 包具有用于 Unicode 编码字符串的函数。`strings` 包提供了许多我们看到的字符串操作函数，如果您不确定需要什么，那可能是最可能查找的位置。

# 2.1 创建字符串

## 问题

您希望创建字符串。

## 解决方案

使用双引号 `""` 或反引号（或反引号） `“` 创建字符串字面量。使用单引号 `”` 创建字符字面量。

## 讨论

在 Go 中，`string` 是一个只读（不可变）的字节切片。它可以是任何字节，不需要任何编码或格式。这与其他一些编程语言不同，其中字符串是字符序列。在 Go 中，一个字符可以由多个字节表示。这符合 Unicode 标准，该标准定义了一个代码点来表示代码空间内的值。在这种情况下，一个字符可以由多个代码点表示。在 Go 中，代码点也称为 *rune*，而 rune 是类型 `int32` 的别名，就像 byte 是类型 `uint8` 的别名，表示无符号 8 位整数。

因此，如果您对字符串进行索引，最终将得到一个字节而不是字符。无论如何，Go 没有字符数据类型——而是使用字节和符文。一个字节表示 ASCII 字符，一个符文表示 UTF-8 编码中的 Unicode 字符。需要明确的是，这并不意味着 Go 中没有字符，只是没有 `char` 数据类型，只有 `byte` 和 `rune`。

在 Go 中，字符使用单引号 `''` 创建。

```go
var c = 'A'
```

在这种情况下，变量 `c` 的数据类型默认为 `int32` 或 rune。如果您希望它是字节，则可以显式指定类型。

```go
var c byte = 'A'
```

在 Go 中，可以使用双引号 `""` 或反引号 ```go `` ``` 创建字符串。

```go
var str = "A simple string"
```

使用双引号创建的字符串中可以包含转义字符。例如，一个非常常见的转义字符是换行符，用反斜杠后跟 *n* 表示 — `\n`。

```go
var str = "A simple string\n"
```

转义字符的另一个常见用途是转义双引号本身，以便它可以在使用双引号创建的字符串中使用。

```go
var str = `A \"simple\" string`
```

使用反引号创建的字符串被认为是“原始”字符串。原始字符串忽略所有格式，包括转义字符。事实上，你可以使用反引号创建多行字符串。例如，使用双引号是不可能的，实际上会导致语法错误。

```go
var str = "
A
simple
string
"
```

然而，如果我们将双引号替换为反引号，则`str`将成为多行字符串。

```go
var str = `
A
simple
string
`
```

这是因为反引号之间的内容不被 Go 编译器处理（即它是“原始”的）。

# 2.2 将字符串转换为字节和字节转换为字符串

## 问题

你想将字符串转换为字节并将字节转换为字符串。

## 解决方案

使用`[]byte(str)`将字符串类型转换为字节数组，并使用`string(bytes)`将字节数组类型转换为字符串。

## 讨论

字符串是字节片，因此你可以通过类型转换直接将字符串转换为字节数组。

```go
str := "This is a simple string"
bytes := []byte(str)
```

将字节数组转换为字符串也是通过类型转换完成的。

```go
bytes := []byte{84, 104, 105, 115, 32, 105, 115, 32, 97, 32, 115, 105, 109, 112, 108,
                101, 32, 115, 116, 114, 105, 110, 103}
str := string(bytes)
```

# 2.3 从其他字符串和数据创建字符串

## 问题

你想从其他字符串或数据创建一个字符串。

## 解决方案

有多种方法可以做到这一点，包括直接连接、使用`strings.Join`、使用`fmt.Sprint`和使用`strings.Builder`。

## 讨论

有时候你想从其他字符串或数据创建字符串。一个相当直接的方法是将字符串和其他数据连接在一起。

```go
var str string = "The time is " + time.Now().Format(time.Kitchen) + " now."
```

`Now`函数返回当前时间，由`Format`方法格式化为字符串返回。当我们连接这些字符串时，我们将得到这个结果。

```go
The time is 5:28PM now.
```

另一种方法是在`string`包中使用`Join`函数。

```go
var str string = strings.Join([]string{"The time is",
                    time.Now().Format(time.Kitchen),
                    "now."}, " ")
```

这也很简单，因为函数接受一个字符串数组，并根据分隔符将它们放在一起。

到目前为止，展示的两种方法都是关于将字符串放在一起。显然，你可以在将它们连接起来之前将不同的数据类型转换为字符串，但有时你只想让 Go 来做。为此，我们有`fmt.Sprint`函数及其各种变体。让我们看看最简单和最直接的变体。

```go
var str string = fmt.Sprint("The time is ", time.Now().Format(time.Kitchen), " now.")
```

这与`Join`或直接连接似乎没有太大区别，因为所有三个参数都是字符串。实际上，`fmt.Sprint`及其变体接受`interface{}`参数，这意味着它可以接受任何数据类型。换句话说，你实际上可以直接传入`Now`返回的`Time`结构体。

```go
var str string = fmt.Sprint("The time is ", time.Now(), " now.")
```

`fmt.Sprint`的一个流行变体是格式化变体，即`fmt.Sprintf`。使用这种变体略有不同——第一个参数是格式字符串，在字符串中的不同位置可以放置不同的动词格式。从第二个参数开始是可以替换到动词中的数据值。

```go
var str string = fmt.Sprintf("The time is %v now.", time.Now())
```

对于`Time`结构体没有关联的动词，因此我们使用`%v`，它将以默认格式格式化值。

最后，`string` 包还提供了另一种创建字符串的方式，使用 `strings.Builder` 结构体。使用 `Builder` 创建字符串稍微复杂一些，因为它需要逐个添加数据片段。让我们看看如何使用 `Builder`。

```go
var builder strings.Builder
builder.WriteString("The time is ")
builder.WriteString(time.Now().Format(time.Kitchen))
builder.WriteString(" now.")
var str string = builder.String()
```

思路很简单，我们创建一个 `Builder` 结构体，然后逐步将数据写入其中，最后使用 `String` 方法提取最终的字符串。`Builder` 结构体还有一些其他方法，包括接受字节数组的 `Write` 方法，接受单个字节的 `WriteByte` 方法以及接受单个字符的 `WriteRune` 方法。然而，正如你所见，它们都是字符串或字符。其他数据类型怎么办？我们需要先将所有其他数据类型转换为字符串、字节或字符吗？不需要，因为 `Builder` 是一个 `Writer`（它实现了 `Write` 方法），你实际上可以使用另一种方法将不同的数据类型写入其中。

```go
var builder strings.Builder
fmt.Fprint(&builder, "The time is ")
fmt.Fprint(&builder, time.Now())
fmt.Fprint(&builder, " now.")
var str string = builder.String()
```

在这里，我们使用 `fmt.Fprint` 将任意数据类型写入构建器，并使用 `String` 提取最终的字符串。

我们已经看到了使用不同数据片段组合字符串的几种方法，包括字符串和其他类型的数据。有些方法非常直接（只需将它们添加在一起），而其他方法则更加深思熟虑。让我们来看看这些不同方法的性能表现。

```go
package string

import (
	"fmt"
	"strings"
	"testing"
	"time"
)

func BenchmarkStringConcat(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = "The time is " + time.Now().Format(time.Kitchen) + " now."
	}
}

func BenchmarkStringJoin(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = strings.Join([]string{"The time is", time.Now().Format(time.Kitchen),
            "now."}, " ")
	}
}

func BenchmarkStringSprint(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = fmt.Sprint("The time is ", time.Now().Format(time.Kitchen), " now.")
	}
}

func BenchmarkStringSprintDiff(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = fmt.Sprint("The time is ", time.Now(), " now.")
	}
}

func BenchmarkStringSprintf(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = fmt.Sprintf("The time is %v now.", time.Now().Format(time.Kitchen))
	}
}

func BenchmarkStringSprintfDiff(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = fmt.Sprintf("The time is %s now.", time.Now())
	}
}

func BenchmarkStringBuilderFprint(b *testing.B) {
	for i := 0; i < b.N; i++ {
		var builder strings.Builder
		fmt.Fprint(&builder, "The time is ")
		fmt.Fprint(&builder, time.Now().Format(time.Kitchen))
		fmt.Fprint(&builder, " now.")
		_ = builder.String()
	}
}

func BenchmarkStringBuilderWriteString(b *testing.B) {
	for i := 0; i < b.N; i++ {
		var builder strings.Builder
		builder.WriteString("The time is ")
		builder.WriteString(time.Now().Format(time.Kitchen))
		builder.WriteString(" now.")
		_ = builder.String()
	}
}
```

让我们从命令行运行基准测试。

```go
$ % go test -bench=BenchmarkString -benchmem
```

这就是结果。

```go
goos: darwin
goarch: arm64
pkg: github.com/sausheong/gocookbook/ch06_string
BenchmarkStringConcat-10                	 5787976	       206.7 ns/op
BenchmarkStringJoin-10                  	 5121637	       235.0 ns/op
BenchmarkStringSprint-10                	 3680838	       323.8 ns/op
BenchmarkStringSprintDiff-10            	 1541514	       779.9 ns/op
BenchmarkStringSprintf-10               	 4032438	       297.8 ns/op
BenchmarkStringSprintfDiff-10           	 1610212	       740.9 ns/op
BenchmarkStringBuilderFprint-10         	 2580783	       464.2 ns/op
BenchmarkStringBuilderWriteString-10    	 4866556	       247.0 ns/op
PASS
ok  	github.com/sausheong/gocookbook/ch06_string	13.025s
```

可能让你惊讶的是，最简单的方法实际上也是性能最佳的方法。使用 `fmt.Sprint` 和使用 `interface{}` 的任何方法都相对不那么高效。

# 2.4 将字符串转换为数字

## 问题

你想将字符串转换为数字。

## 解决方案

使用 `strconv` 包中的 `Atoi` 或 `Parse` 函数进行字符串转换。使用函数将字符串转换为数字，并使用 `Itoa` 或 `Format` 函数将数字转换为字符串。

## 讨论

`strconv` 包名副其实，主要用于字符串转换。有两组函数让我们可以将字符串转换为数字和将数字转换为字符串。`Parse` 函数将字符串转换为数字，`Format` 函数将数字转换为字符串。如果不确定使用哪些函数，一般来说记住解析函数读取字符串，格式化函数创建字符串。

将字符串解析为数字在使用上似乎有限，但在处理文本格式化数据时可能特别有用，例如 JSON 或 Yaml，甚至是 XML。文本格式化数据很受欢迎，因为它易于阅读，但缺点是一切最终都成为字符串。直接将字符串解析为更可用的数据类型，比如数字，在这种情况下就变得非常有用。

让我们从简单的开始。我们想要解析一个显示整数并产生实际整数的字符串。

```go
i, err := strconv.Atoi("123") // equivalent to ParseInt("123", 10, 0)
```

`strconv` 包提供了一个方便的函数，用于将字符串转换为整数。这很容易记住，因为 `Atoi` 基本上是将 *字母数字* 转换为 *整数*。

使用`Parse`函数等效于`Atoi`，使用`ParseInt(s, 10, 0)`，其中`s`是表示数字的字符串。

函数`ParseInt`正如其名称所示，将字符串解析为整数。您可以指定基数（0、2 到 36）以及位数（0 到 64）。您可以使用`ParseInt`来处理有符号或无符号整数，只需在数字前加上+或-号。

```go
i, err := strconv.ParseInt("123", 10, 0)
```

函数`ParseFloat`将字符串解析为浮点数。bitsize 参数指定了精度，32 用于`float32`，64 用于`float64`等。

```go
f, err := strconv.ParseFloat("1.234", 64)
```

在上面的代码中，`f`是一个`float64`数值。

当尝试解析代表布尔值的字符串时，`ParseBool`很有用。它接受 1、t、T、TRUE、true、True、0、f、F、FALSE、false、False 这些值。

```go
b, err := strconv.ParseBool("TRUE")
```

在上面的代码中，`b`是一个布尔值，其值为`true`。

所有`Parse`函数都会返回`NumError`，包括`Atoi`。`NumError`提供有关错误的附加信息，包括被调用的函数、传入的数字以及失败原因。

```go
str := "Not a number"
_, err := strconv.Atoi(str)
if err != nil {
    e := err.(*strconv.NumError)
    fmt.Println("Func:", e.Func)
    fmt.Println("Num:", e.Num)
    fmt.Println("Err:", e.Err)
    fmt.Println(err)
}
```

这是您运行此代码时可以看到的内容。

```go
Func: Atoi
Num: Not a number
Err: invalid syntax
strconv.Atoi: parsing "Not a number": invalid syntax
```

# 2.5 将数字转换为字符串

## 问题

您想将数字转换为字符串。

## 解决方案

使用`strconv`包中的`Itoa`或`Format`函数将数字转换为字符串。

## 讨论

在前一个示例中，我们已经讨论了`strconv`包和`Parse`函数。在本示例中，我们将讨论`Format`函数及其如何将数字转换为字符串。

将数字格式化为字符串是将字符串解析为数字的逆过程。在需要通过文本格式传递数据的情况下，将数字格式化可以很有用。将数字格式化为字符串的一个常见用法是在需要向用户展示更可读的数字时使用。例如，我们希望向用户展示`1.66666666`时，会希望显示`1.67`。这在显示货币时也很常见。

让我们从最简单的例子开始。就像解析字符串有`Atoi`一样，格式化字符串有`Itoa`。顾名思义，它是`Atoi`的逆过程，将整数转换为字符串。

```go
str := strconv.Itoa(123) // equivalent to FormatInt(int64(123), 10)
```

请注意，`Itoa`不会返回错误。事实上，`Format`函数中的任何一个都不会返回错误。这是有道理的 —— 将数字转换为字符串总是可能的，而反之则不然。

与以往一样，`Itoa`是使用`FormatInt`的便捷函数。但是，我们必须首先确保输入的数字参数始终为`int64`。`FormatInt`还需要一个基数参数，其中基数是介于 2 到 36 之间的整数，两个数字都包括在内。这意味着它可以将二进制数转换为字符串。

```go
str := strconv.FormatInt(int64(123), 10)
```

上面的代码返回字符串`"123"`。如果我们指定基数为 2 会怎么样？

```go
str := strconv.FormatInt(int64(123), 2)
```

这将返回字符串`"1111011"`。这意味着您可以使用`FormatInt`将一个基数中的数字转换为另一个基数，至少作为字符串显示。

`FormatFloat`函数比`FormatInt`函数稍微复杂一些。它根据格式和精度将浮点数转换为字符串。`FormatFloat`中可用于十进制数（基数 10）的格式有：

+   `f` - 无指数

+   `e` 和 `E` - 带有指数

+   `g` 和 `G` - 如果指数很大，则为`e`或`E`，否则将不带指数（例如`f`）

其他格式是二进制数字（基数 2）的`b`，十六进制数字（基数 16）的`x`和`X`。

精度描述的是要打印的数字位数（不包括指数）。精度值为-1 时，Go 会选择最小的数字位数，以便`ParseFloat`返回整个数字。

让我们看一些代码，这将使事情更清晰。

```go
var v float64 = 123456.123456
var s string

s = strconv.FormatFloat(v, 'f', -1, 64)
fmt.Println("f (prec=-1)\t:", s)
s = strconv.FormatFloat(v, 'f', 4, 64)
fmt.Println("f (prec=4)\t:", s)
s = strconv.FormatFloat(v, 'f', 9, 64)
fmt.Println("f (prec=9)\t:", s)

s = strconv.FormatFloat(v, 'e', -1, 64)
fmt.Println("\ne (prec=-1)\t:", s)
s = strconv.FormatFloat(v, 'E', -1, 64)
fmt.Println("E (prec=-1)\t:", s)
s = strconv.FormatFloat(v, 'e', 4, 64)
fmt.Println("e (prec=4)\t:", s)
s = strconv.FormatFloat(v, 'e', 9, 64)
fmt.Println("e (prec=9)\t:", s)

s = strconv.FormatFloat(v, 'g', -1, 64)
fmt.Println("\ng (prec=-1)\t:", s)
s = strconv.FormatFloat(v, 'G', -1, 64)
fmt.Println("G (prec=-1)\t:", s)
s = strconv.FormatFloat(v, 'g', 4, 64)
fmt.Println("g (prec=4)\t:", s)
```

我们使用 64 位精度的浮点值`float64`，并比较精度-1 和精度 4。如果你没有意识到，小写*e*和大写*E*是完全相同的，除了指数字母*e*是小写或大写的不同。

当你运行代码时，你应该看到这个。

```go
f (prec=-1)	: 123456.123456
f (prec=4)	: 123456.1235
f (prec=9)	: 123456.123456000

e (prec=-1)	: 1.23456123456e+05
E (prec=-1)	: 1.23456123456E+05
e (prec=4)	: 1.2346e+05
e (prec=9)	: 1.234561235e+05

g (prec=-1)	: 123456.123456
G (prec=-1)	: 123456.123456
g (prec=4)	: 1.235e+05
```

# 2.6 替换字符串中的多个字符

## 问题

你想用另一个字符串替换字符串的部分。

## 解决方案

有几种方法可以做到这一点。你可以使用`strings.Replace`函数或`strings.ReplaceAll`函数来替换选定的字符串。你也可以使用`strings.Replacer`类型来创建替换器。

## 讨论

有几种方法可以用来用另一个字符串替换字符串的部分。最简单的方法是使用`strings.Replace`函数。

`strings.Replace`函数非常简单。只需传递一个字符串，旧字符串你想要替换的字符串，以及你想要替换它的新字符串。让我们看看这段代码，使用查尔斯·狄更斯的《远大前程》中的这句话。

```go
var quote string = `I loved her against reason, against promise,
against peace, against hope, against happiness,
against all discouragement that could be.`
```

我们使用`Replace`运行了几次替换。

```go
replaced := strings.Replace(quote, "against", "with", 1)
fmt.Println(replaced)
replaced2 := strings.Replace(quote, "against", "with", 2)
fmt.Println("\n", replaced2)
replacedAll := strings.Replace(quote, "against", "with", -1)
fmt.Println("\n", replacedAll)
```

最后一个参数告诉`Replace`要替换的匹配数。如果最后一个参数是-1，`Replace`将匹配每一个实例。

```go
I loved her with reason, against promise,
against peace, against hope, against happiness,
against all discouragement that could be.

 I loved her with reason, with promise,
against peace, against hope, against happiness,
against all discouragement that could be.

 I loved her with reason, with promise,
with peace, with hope, with happiness,
with all discouragement that could be.
```

还有一个`ReplaceAll`函数，它更像是一个方便的函数，调用`Replace`时将最后一个参数设置为-1。

`strings`包中的`Replacer`类型允许您同时进行多个替换。如果您需要进行大量替换，这将更加方便。

```go
replacer := strings.NewReplacer("her", "him", "against", "for", "all", "some")
replaced := replacer.Replace(quote)
fmt.Println(replaced)
```

你只需要提供一组替换字符串作为参数。在上面的代码中，我们用“her”替换“him”，“against”替换“for”，“all”替换“some”。它将同时进行所有替换。如果你运行上面的代码，你将得到以下结果。

```go
I loved him for reason, for promise,
for peace, for hope, for happiness,
for some discouragement that could be.
```

所以使用`Replace`还是创建`Replacer`更好呢？显然，创建替换器需要额外的一行代码。但是它的性能如何？让我们看一看。我们将从仅替换一个单词开始。

```go
func BenchmarkOneReplace(b *testing.B) {
	for i := 0; i < b.N; i++ {
		strings.Replace(quote, "her", "him", 1)
	}
}

func BenchmarkOneReplacer(b *testing.B) {
	replacer := strings.NewReplacer("her", "him")
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		replacer.Replace(quote)
	}
}
```

如果只是一个字符串需要替换，`Replace`更快。对于简单替换，只是有更多的开销。

```go
goos: darwin
goarch: arm64
pkg: github.com/sausheong/gocookbook/ch06_string
BenchmarkOneReplace-10     	 7264310	       156.9 ns/op
BenchmarkOneReplacer-10    	 4336489	       276.0 ns/op
PASS
ok  	github.com/sausheong/gocookbook/ch06_string	3.151s
```

让我们看看是否需要进行多次替换。

```go
func BenchmarkReplace(b *testing.B) {
	for i := 0; i < b.N; i++ {
		strings.Replace(quote, "against", "with", -1)
	}
}

func BenchmarkReplacerCreate(b *testing.B) {
	for i := 0; i < b.N; i++ {
		strings.NewReplacer("against", "with")
	}
}

func BenchmarkReplacer(b *testing.B) {
	replacer := strings.NewReplacer("against", "with")
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		replacer.Replace(quote)
	}
}
```

我们不需要每次创建一个`Replacer`，因此如果您想对不同的字符串进行相同的替换，可以多次重用替换器。同时，如果您有很多替换，使用替换器肯定更容易。

```go
goos: darwin
goarch: arm64
pkg: github.com/sausheong/gocookbook/ch06_string
BenchmarkReplace-10           	 2250291	       532.1 ns/op
BenchmarkReplacerCreate-10    	31878366	        37.13 ns/op
BenchmarkReplacer-10          	 4671319	       255.0 ns/op
PASS
ok  	github.com/sausheong/gocookbook/ch06_string	4.547s
```

正如您所见，要替换更多的字符串，使用`Replacer`会更有效率。

# 2.7 创建子字符串

## 问题

您想从字符串中创建子字符串。

## 解决方案

将字符串视为数组或切片，并从字符串中取出一个子字符串。

## 讨论

在 Go 语言中，字符串是字节的切片。因此，如果您想从字符串中取出一个子字符串，可以像对待任何切片一样操作。让我们使用前面的配方中相同的引语，查尔斯·狄更斯的《远大前程》。

```go
var quote string = `I loved her against reason, against promise,
against peace, against hope, against happiness,
against all discouragement that could be.`
```

如果您想从引语中提取“against reason”这几个词，可以这样做。

```go
quote[12:26]
```

这很简单，但是如何在不手动计数引语中的字母的情况下知道单词的位置呢？像许多编程语言一样，我们只需找到子字符串的索引。

```go
strings.Index(quote, "against reason")
```

`strings.Index`函数给出了与第二个参数匹配的第一个子字符串的索引，本例中将是 12。这将给我们子字符串的位置。要找到子字符串的结束位置，我们只需将子字符串的长度添加到索引即可。

```go
i := strings.Index(quote, "against reason")
j := i + len("against reason")
fmt.Println(quote[i:j])
```

# 2.8 检查字符串是否包含另一个字符串

## 问题

您想检查一个字符串是否包含另一个字符串。

## 解决方案

使用`strings`包中的`Contains`函数。如果您要检查的字符串是后缀或前缀，也可以使用`HasSuffix`或`HasPrefix`函数。

## 讨论

在 Go 语言中检查字符串是否包含另一个字符串非常简单。您可以使用`strings.Contains`函数，传入字符串和子字符串，它会相应地返回 true 或 false。我们将再次使用查尔斯·狄更斯的《远大前程》中的引语。

```go
var quote string = `I loved her against reason, against promise,
against peace, against hope, against happiness,
against all discouragement that could be.`
```

`Contains`函数检查引语是否包含字符串“against”。

```go
var has bool = strings.Contains(quote, "against")
```

或者，您也可以始终使用`strings.Index`，如果返回的结果是 < 0，这意味着字符串中找不到子字符串。两种函数的性能是一样的，这并不奇怪，因为`Contains`只是围绕`Index`函数提供的方便功能。另一种选择是使用`Count`函数，它返回子字符串在字符串中出现的次数，但这通常是一个较差的选择（除非您无论如何都需要知道计数），因为性能比任一函数都要差。

如果您想查找子字符串是否是字符串的前缀，可以使用`HasPrefix`函数。

```go
strings.HasPrefix(quote, "I loved")
```

当然，您可以直接从字符串中切出前缀的长度并自行检查。

```go
prefix := "I loved"
if quote[:len(prefix)] == prefix {
    ... // do whatever you wanted if the string has the prefix
}
```

您可以使用`HasSuffix`函数对后缀执行相同的操作。

```go
strings.HasSuffix(quote, "could be.")
```

您还可以直接切片字符串以获取后缀并进行比较。

```go
suffix := "could be."
if quote[len(quote)-len(suffix):] != suffix {
    ... // do whatever you wanted if the string has the prefix
}
```

# 2.9 将字符串分割为字符串数组或将字符串数组组合成字符串

## 问题

你想通过拆分字符串创建一个字符串数组，或者通过组合字符串数组创建一个字符串。

## 解决方案

使用`strings`包中的`Split`函数来拆分字符串，并使用`Join`函数将字符串数组组合成单个字符串。

## 讨论

许多函数接受一个字符串数组。你可能想要处理字符串中的单词而不是单独的字节。你可能在处理像 CSV 或 TSV 这样的分隔符分隔的数据。无论是哪种情况，能够快速地将一个字符串拆分成字符串数组是很有用的。

在 Go 语言中，你可以使用`strings.Split`函数来做到这一点。让我们再次使用查尔斯·狄更斯的《远大前程》的引语。

```go
var quote string = `I loved her against reason, against promise,
against peace, against hope, against happiness,
against all discouragement that could be.`
```

`Split`函数根据指定的分隔符将字符串拆分为字符串数组。

```go
array := strings.Split(quote, " ")
fmt.Printf("%q", array)
```

在上面的代码中，我们使用空格作为分隔符，所以这是你会看到的。

```go
["I" "loved" "her" "against" "reason," "against" "promise," "\nagainst" "peace,"
"against" "hope," "against" "happiness," "\nagainst" "all" "discouragement" "that"
"could" "be."]
```

你可能注意到一些元素有换行符，因为原始字符串包含它。这可能不是你想要的，那么我们如何移除换行符？或者更糟糕的是，如果有多个空格，你的数组看起来会非常凌乱，有许多额外的空字符串元素。当然，你可以稍后用笨方法清理它们，但有一个更简单的方法。`strings`包中有一个名为`Fields`的函数，可以将一个字符串拆分为考虑一个或多个连续空格的字符串数组，由`uniform.isSpace`定义。

如果我们使用`Fields`而不是`Split`来看一下。

```go
array := strings.Fields(quote)
fmt.Printf("%q", array)
```

现在你应该能看到这个了。

```go
["I" "loved" "her" "against" "reason," "against" "promise," "against" "peace,"
"against" "hope," "against" "happiness," "against" "all" "discouragement" "that"
"could" "be."]
```

换行符已经被去掉了。如果我们想要连标点符号（在这种情况下是逗号和句号在引号末尾）也移除掉呢？这有点复杂，我们需要使用`FieldsFunc`函数并传入一个函数来确定是否应该将其作为分隔符的一部分。让我们看看。

```go
f := func(c rune) bool {
    return unicode.IsPunct(c) || !unicode.IsLetter(c)
}
array := strings.FieldsFunc(quote, f)
fmt.Printf("%q", array)
```

在上面的代码中，我们创建了一个函数`f`，它将连续的标点符号和非字母字符视为分隔符的一部分。然后我们将这个函数传递给`FieldsFunc`来对字符串执行。这是我们应该看到的。

```go
["I" "loved" "her" "against" "reason" "against" "promise" "against" "peace"
"against" "hope" "against" "happiness" "against" "all" "discouragement" "that"
"could" "be"]
```

如你所见，我们也去掉了标点符号。`FieldsFunc`函数非常灵活，我只给出了一个非常简单的例子。如果你经常处理字符串拆分，这将是一个非常强大的函数，你可以用它做很多事情。

如果我们只想把字符串拆分为前 9 个元素并将剩余部分放入单个字符串呢？`SplitN`函数正好可以做到这一点，其中`n`是生成数组中元素数量，在这种情况下是 10。

```go
array := strings.SplitN(quote, " ", 10)
fmt.Printf("%q", array)
```

你将会看到结果数组中有 10 个元素。

```go
["I" "loved" "her" "against" "reason," "against" "promise," "\nagainst" "peace,"
"against hope, against happiness, \nagainst all discouragement that could be."]
```

有时候你希望在拆分字符串后保留分隔符。Go 语言有一个名为`SplitAfter`的函数可以做到这一点。

```go
array := strings.SplitAfter(quote, " ")
fmt.Printf("%q", array)
```

如你所见，每个元素都以空格结尾（这是分隔符），除了最后一个元素。

```go
["I " "loved " "her " "against " "reason, " "against " "promise, " "\nagainst "
"peace, " "against " "hope, " "against " "happiness, " "\nagainst " "all "
"discouragement " "that " "could " "be."]
```

# 2.10 字符串修剪

## 问题

你想要移除字符串的开头和结尾的字符。

## 解决方案

使用 `strings` 包中的 `Trim` 函数。

## 讨论

在处理字符串时，经常会遇到尾随或前导的空白或其他不必要的字符。我们经常希望在存储或进一步处理字符串之前移除这些字符。因此，字符串修剪的概念也非常普遍。字符串修剪基本上是从字符串的开头或结尾移除字符，而不是在字符串内部。

在 Go 中，有许多 `strings` 包中的 `Trim` 函数可以帮助我们修剪字符串。

让我们从 `Trim` 函数开始。它接受一个字符串和一个*切割集*，这是一个由一个或多个 Unicode 代码点组成的字符串，并返回一个删除了所有前导和尾随这些代码点的字符串。

```go
var str string = ", and that is all."
var cutset string = ",. "
trimmed := strings.Trim(str, cutset) // "and that is all"
```

在上面的代码中，我们想要移除前导逗号和空格，以及尾随句点。为此，我们使用包含这 3 个 Unicode 代码点的切割集，因此切割集字符串最终为 `",. "`。

`Trim` 函数同时移除前导和尾随字符，但如果您只想移除尾随字符，可以使用 `TrimRight` 函数；如果只想移除前导字符，可以使用 `TrimLeft` 函数。

```go
trimmedLeft := strings.TrimLeft(str, cutset)   // "and that is all."
trimmedRight := strings.TrimRight(str, cutset) // ", and that is all"
```

之前的 `Trim` 函数会移除切割集中的任何字符。但是，如果你想要移除整个前导子字符串（或者前缀），你可以使用 `TrimPrefix` 函数。

```go
trimmedPrefix := strings.TrimPrefix(str, ", and ")	// "that is all."
```

同样地，如果你想要移除整个尾部子字符串（或者后缀），你可以使用 `TrimSuffix` 函数。

```go
trimmedSuffix := strings.TrimSuffix(str, " all.") // ", and that is"
```

`Trim` 函数允许您移除任何前导或尾随字符或字符串。但通常被移除的字符是空白字符，如换行符 (`\n`) 或制表符 (`\r`) 或回车符 (`\r`)。为方便起见，Go 提供了一个 `TrimSpace` 函数，仅移除前导和尾随空白字符。

```go
trimmed := strings.TrimSpace("\r\n\t Hello World \t\n\r") // Hello World
```

最后一组 `Trim` 函数是 `TrimFunc`、`TrimLeftFunc` 和 `TrimRightFunc` 函数。从名称就可以猜到，这允许您用函数替换切割集字符串，该函数将检查前导或尾随或两者都检查的 Unicode 代码点，并确保它们满足条件。

```go
f := func(c rune) bool {
	return unicode.IsPunct(c) || !unicode.IsLetter(c)
}
trimmed := strings.TrimFunc(str, f) // "and that is all"
```

这些 `TrimFunc` 函数允许您对字符串修剪进行更精细的控制，如果您有意外的规则用于删除前导或尾随字符，则非常有用。

# 2.11 从命令行捕获字符串输入

## 问题

您想要从命令行捕获用户输入的字符串数据。

## 解决方案

使用 `fmt` 包中的 `Scan` 函数从标准输入读取单个字符串。要读取由空格分隔的字符串，请在包装在 `os.Stdin` 周围的 `Reader` 上使用 `ReadString`。

## 讨论

如果您的 Go 程序从命令行运行，可能会遇到需要从用户获取字符串输入的情况。这就是 `fmt` 包中的 `Scan` 函数派上用场的地方。

您可以使用 `Scan` 来从用户获取输入，方法是创建一个变量，然后将该变量的引用传递给 `Scan`。

```go
package main

import "fmt"

func main() {
	var input string
	fmt.Print("Please enter a word: ")
	n, err := fmt.Scan(&input)
	if err != nil {
		fmt.Println("error with user input:", err, n)
	} else {
		fmt.Println("You entered:", input)
	}
}
```

如果您运行上面的代码，程序将在 `fmt.Scan` 处等待您的输入，并且只有在您输入数据后才会继续。一旦您输入了一些数据，`Scan` 将把数据存储到 `input` 变量中。

如果您运行上面的代码，这就是您应该看到的。

```go
% go run scan.go
Please enter a word: Hello
You entered: Hello
```

文档没有明确提到您需要传递一个变量的引用。您可以按值传递一个变量，它将被编译。但是，如果您这样做，将会出现错误。

```go
n, err := fmt.Scan(input)
if err != nil {
	fmt.Println("error with user input:", err, n)
}
```

如果您运行上面的代码，您将看到这个结果。

```go
% go run scan.go
Please enter a word: error with user input: type not a pointer: string 0
```

`Scan` 函数可以接受多个参数，每个参数表示由空格分隔的用户输入。让我们再做两个输入。

```go
func main() {
	var input1, input2 string
	fmt.Print("Please enter two words: ")
	n, err := fmt.Scan(&input1, &input2)
	if err != nil {
		fmt.Println("error with user input:", err, n)
	} else {
		fmt.Println("You entered:", input1, "and", input2)
	}
}
```

如果您运行此代码，并输入 *Hello* 和 *World*，它们将分别被捕获并存储到 `input1` 和 `input2` 中。如果您运行上面的代码，您将看到这个结果。

```go
% go run scan.go
Please enter two words: Hello World
You entered: Hello and World
```

这似乎有些局限性。如果您想捕获一个带有空格的字符串怎么办？例如，您想要让用户输入一个句子。在这种情况下，您可以在包装在 `os.Stdin` 周围的 `Reader` 上使用 `ReadString` 函数。

```go
func main() {
	reader := bufio.NewReader(os.Stdin)
	fmt.Print("Please enter many words: ")
	input, err := reader.ReadString('\n')
	if err != nil {
		fmt.Println("error with user input:", err)
	} else {
		fmt.Println("You entered:", input)
	}
}
```

如果您运行上面的代码，这就是您应该看到的结果。

```go
% go run scan.go
Please enter many words: Many words here and still more to go
You entered: Many words here and still more to go
```

您应该知道 `Scan` 不仅可以用于从用户获取字符串输入，还可以用于获取数字等。

# 2.12 转义和反转义 HTML 字符串

## 问题

您想要转义或反转义 HTML 字符串。

## 解决方案

使用 `html` 包中的 `EscapeString` 和 `UnescapeString` 函数来转义或反转义 HTML 字符串。

## 讨论

HTML 是一种基于文本的标记语言，用于结构化网页及其内容。它通常由浏览器解释并显示。HTML 的大部分内容都在 HTML 标签内描述，例如 `<a>` 是锚点标签，`<img>` 是图像标签。类似地，还有其他像 `&` 和 `"` 这样具有在 HTML 中特定意义的字符。

但是如果想在 HTML 中显示这些字符怎么办？例如，和符号（`&`）是一个常用字符。小于号和大于号（`<` 和 `>`）也是常用的。如果我们不打算让这些符号在 HTML 中具有任何意义，我们需要将它们转换为 HTML 字符实体。例如：

+   小于号变成了 <

+   > 大于号变成了 >

+   &（和符号）变成了 &

等等。将 HTML 字符转换为实体的过程称为 HTML 转义，反之则称为 HTML 反转义。

Go 语言在 `html` 包中有一对函数，名为 `EscapeString` 和 `UnescapeString`，可用于转义或反转义 HTML。

```go
str := "<b>Rock & Roll</b>"
escaped := html.EscapeString(str) // "&lt;b&gt;Rock &amp; Roll&lt;/b&gt;"
```

反转义将转义的 HTML 字符串恢复为原始字符串。

```go
unescaped := html.UnescapeString(escaped) // "<b>Rock & Roll</b>"
```

您可能注意到 `html/template` 包中还有一个 `HTMLEscapeString` 函数。这两个函数的结果是相同的。

# 2.13 使用正则表达式

## 问题

您想要使用正则表达式进行字符串操作。

## 解决方案

使用`regex`包，并使用`Compile`函数解析正则表达式以返回一个`Regexp`结构体。然后使用`Find`函数匹配模式并返回字符串。

## 讨论

正则表达式是一种描述字符串中搜索模式的表示法。当特定字符串在正则表达式描述的集合中时，正则表达式与该字符串匹配。正则表达式在许多语言中都很流行且可用。

Go 语言有一个专门用于正则表达式的标准包叫做`regex`。正则表达式的语法与 Perl、Python 和其他语言使用的一般语法相同。

使用`regex`包非常简单。你必须先从正则表达式创建一个`Regexp`结构体。有了这个结构体，你可以调用任意数量的`Find`函数来返回匹配的字符串或字符串的索引。

尽管`regex`包中看起来有很多`Find`函数，大多数都作为`Regexp`结构体的方法附加在上面，但它们有一个通用的模式。特别是没有`All`的那些函数只会返回第一个匹配项，而带有`All`的函数将根据`n`参数在字符串中可能返回所有匹配项。带有`String`的函数将返回字符串或字符串片段，而没有的函数将返回字节数组`[]byte`。带有`Index`的函数返回匹配项的索引。

在下面的代码片段中，我们将使用查尔斯·狄更斯《远大前程》中的同一引用，就像本章节中的其他示例一样。

```go
var quote string = `I loved her against reason, against promise,
against peace, against hope, against happiness,
against all discouragement that could be.`
```

让我们先创建一个`Regexp`结构体，以便稍后使用。

```go
re, err := regexp.Compile(`against [\w]+`)
```

这里的正则表达式是`` `against [\w]+` ``。我们使用反引号来创建正则表达式字符串，因为正则表达式使用了很多反斜杠，如果我们使用双引号，这些反斜杠会被解释得不同。我们使用的正则表达式匹配以*against*开头并在其后有一个单词的字符串模式。

`Compile`的一个方便替代是`MustCompile`函数。这个函数与`Compile`做的事情完全一样，只是它不会返回错误。相反，如果正则表达式无法编译，该函数将会引发 panic。

一旦我们设置了正则表达式，我们就可以用它来查找匹配项。有许多`Find`方法，但在这个示例中我们只介绍了其中几个。其中一个最直接的方法是`MatchString`，它简单地告诉我们正则表达式在字符串中是否有任何匹配项。

```go
re.MatchString(quote) // true
```

有时除了检查正则表达式是否有任何匹配项外，我们还想返回匹配的字符串。我们可以使用`FindString`来实现这一点。

```go
str := re.FindString(quote) // "against reason"
```

在这里，我们从`quote`字符串中找到一个字符串，使用我们之前设置的正则表达式。它返回第一个匹配项，因此返回的字符串是`against reason`。如果我们想返回所有匹配项，我们必须使用带有`All`的方法，例如`FindAllString`。

```go
strs := re.FindAllString(quote, -1)
fmt.Println(strs)
```

`FindAllString`中的第二个参数，像所有的`All`方法一样，表示我们希望返回的匹配数。如果我们想返回所有匹配项，我们需要使用一个负数，例如`-1`。返回的值是一个字符串数组。

如果我们运行上面的代码，同时打印出字符串数组，我们将得到这样的结果。

```go
[against reason against promise against peace against hope against happiness
against all]
```

除了返回匹配的字符串，有时我们还想知道匹配项的位置。在这种情况下，我们可以使用`Index`函数，例如`FindStringIndex`。

```go
locs := re.FindStringIndex(quote) // [12 26]
```

这返回一个包含两个整数的切片，表示第一个匹配的位置。如果我们使用这两个整数在`quote`本身上，我们将能够提取匹配的子字符串。

```go
quote[locs[0]:locs[1]] // against reason
```

和以前一样，要获取所有匹配项，我们需要使用`All`方法，因此在这种情况下我们可以使用`FindAllStringIndex`方法。

```go
allLocs := re.FindAllStringIndex(quote, -1)
fmt.Println(allLocs)
```

这将返回所有匹配项的二维切片。

```go
[[12 26] [28 43] [46 59] [61 73] [75 92] [95 106]]
```

除了查找和索引正则表达式，我们还可以使用`ReplaceAllString`完全替换匹配的字符串。这是一个简单的例子。

```go
replaced := re.ReplaceAllString(quote, "anything")
fmt.Println(replaced)
```

如果你运行上面的代码，你应该看到这个结果。

```go
I loved her anything, anything,
anything, anything, anything,
anything discouragement that could be.
```

除了简单的替换，我们还可以使用一个接受匹配字符串并生成另一个字符串作为输出的函数来替换匹配的字符串。让我们看一个快速的例子。

```go
replaced = re.ReplaceAllStringFunc(quote, strings.ToUpper)
fmt.Println(replaced)
```

在这里，我们通过使用`strings.ToUpper`函数将所有匹配的字符串替换为大写版本。这是运行代码的结果。

```go
I loved her AGAINST REASON, AGAINST PROMISE,
AGAINST PEACE, AGAINST HOPE, AGAINST HAPPINESS,
AGAINST ALL discouragement that could be.
```

假设我们不是将匹配字符串中的两个单词都大写，而是只想让第二个单词大写。我们可以创建一个简单的函数来实现这个目标。

```go
f := func(in string) string {
	split := strings.Split(in, " ")
	split[1] = strings.ToUpper(split[1])
	return strings.Join(split, " ")
}
replaced = re.ReplaceAllStringFunc(quote, f)
fmt.Println(replaced)
```

如果我们运行这段代码，我们将得到这个结果。

```go
I loved her against REASON, against PROMISE,
against PEACE, against HOPE, against HAPPINESS,
against ALL discouragement that could be.
```

正则表达式非常强大，并且在许多地方都有用途。在 Go 语言中，它可以用于非常强大的字符串操作。然而，需要注意的是，Go 语言中的`regex`包支持 RE2 接受的正则表达式语法。这保证了正则表达式在输入大小线性时间内运行。因此，一些像`lookahead`和`lookbehind`这样的语法不被支持。如果你更熟悉 PCRE 库支持的语法，你可能需要检查并确保你的正则表达式按照你希望的方式工作。
