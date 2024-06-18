# 第十三章。标准库

使用 Go 开发的最大优势之一是能够利用其标准库。像 Python 一样，Go 也有“电池包含”的哲学，提供了构建应用程序所需的许多工具。由于 Go 是一种相对较新的语言，它附带了一个专注于现代编程环境中遇到的问题的库。

我无法覆盖所有标准库包，幸运的是，我也不需要，因为有许多优秀的信息源涵盖了标准库，从[文档](https://oreil.ly/g970a)开始。相反，我将重点放在几个最重要的包上，以及它们的设计和使用如何展示 Go 的成语风格的原则。一些包（`errors`，`sync`，`context`，`testing`，`reflect`和`unsafe`）在它们自己的章节中进行了介绍。在这一章中，你将看到 Go 对 I/O、时间、JSON 和 HTTP 的内置支持。

# io 及其伙伴们

要使程序有用，它需要读取和写入数据。Go 的输入/输出哲学的核心可以在`io`包中找到。特别是在该包中定义的两个接口可能是 Go 中使用第二和第三最多的接口：`io.Reader`和`io.Writer`。

###### 注意

第一名是什么？那将是`error`，你已经在第九章中看过了。

`io.Reader`和`io.Writer`都定义了一个单一方法：

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

`io.Writer`接口上的`Write`方法接受一个字节切片，将其写入接口实现。它返回写入的字节数以及如果出现错误则返回错误。`io.Reader`上的`Read`方法更有趣。与通过返回参数返回数据不同，一个切片被传递到实现中并进行修改。最多会写入`len(p)`字节到切片中。该方法返回写入的字节数。这可能看起来有点奇怪。你可能期望这样：

```go
type NotHowReaderIsDefined interface {
    Read() (p []byte, err error)
}
```

`io.Reader`被定义为它的样子有一个非常好的原因。让我们编写一个代表如何使用`io.Reader`的函数来说明：

```go
func countLetters(r io.Reader) (map[string]int, error) {
    buf := make([]byte, 2048)
    out := map[string]int{}
    for {
        n, err := r.Read(buf)
        for _, b := range buf[:n] {
            if (b >= 'A' && b <= 'Z') || (b >= 'a' && b <= 'z') {
                out[string(b)]++
            }
        }
        if err == io.EOF {
            return out, nil
        }
        if err != nil {
            return nil, err
        }
    }
}
```

有三件事需要注意。首先，你只需创建一次缓冲区，并在每次调用`r.Read`时重复使用它。这允许你使用单个内存分配从可能很大的数据源中读取。如果`Read`方法被编写为返回一个`[]byte`，则每次调用都需要一个新的分配。每个分配都将最终在堆上，这将为垃圾收集器造成相当多的工作。

如果你想进一步减少分配，可以在程序启动时创建一个缓冲池。然后，在函数开始时从池中取出一个缓冲区，在函数结束时归还它。通过将一个切片传递给`io.Reader`，内存分配就在开发者的控制之下。

第二，你使用从`r.Read`返回的`n`值来知道写入缓冲区的字节数，并迭代处理被读取的`buf`切片的子切片数据。

最后，当从`r.Read`返回的错误是`io.EOF`时，你知道已经完成从`r`的读取。这种错误有点奇怪，因为它并不是真正的错误。它表示从`io.Reader`中没有剩余可读取的内容。当返回`io.EOF`时，你完成处理并返回你的结果。

`io.Reader`中的`Read`方法有一个不寻常的方面。在大多数情况下，当函数或方法有一个错误返回值时，你在处理非错误返回值之前会先检查错误。但是对于`Read`来说，你要做相反的操作，因为字节可能已经被复制到缓冲区中，然后才会因数据流结束或意外条件而触发错误。

###### 提示

如果意外地到达了`io.Reader`的末尾，会返回不同的哨兵错误(`io.ErrUnexpectedEOF`)。请注意，它以`Err`开头，表示这是一个意外的状态。

因为`io.Reader`和`io.Writer`是如此简单的接口，它们可以有很多种实现方式。你可以使用`strings.NewReader`函数从字符串创建一个`io.Reader`：

```go
s := "The quick brown fox jumped over the lazy dog"
sr := strings.NewReader(s)
counts, err := countLetters(sr)
if err != nil {
    return err
}
fmt.Println(counts)
```

正如我在“接口是类型安全的鸭子类型”中讨论的那样，`io.Reader`和`io.Writer`的实现通常在装饰器模式中链接在一起。因为`countLetters`依赖于`io.Reader`，你可以使用完全相同的`countLetters`函数来计算 gzip 压缩文件中的英文字母数。首先，编写一个函数，给定文件名后返回一个`*gzip.Reader`：

```go
func buildGZipReader(fileName string) (*gzip.Reader, func(), error) {
    r, err := os.Open(fileName)
    if err != nil {
        return nil, nil, err
    }
    gr, err := gzip.NewReader(r)
    if err != nil {
        return nil, nil, err
    }
    return gr, func() {
        gr.Close()
        r.Close()
    }, nil
}
```

此函数演示了正确包装实现`io.Reader`的类型的方法。你创建一个`*os.File`（符合`io.Reader`接口），在确保它有效后，将其传递给`gzip.NewReader`函数，该函数返回一个`*gzip.Reader`实例。如果有效，返回`*gzip.Reader`和一个闭包，当调用时会适当地清理你的资源。

因为`*gzip.Reader`实现了`io.Reader`，所以你可以像之前使用`*strings.Reader`一样使用它来与`countLetters`配合使用：

```go
r, closer, err := buildGZipReader("my_data.txt.gz")
if err != nil {
    return err
}
defer closer()
counts, err := countLetters(r)
if err != nil {
    return err
}
fmt.Println(counts)
```

你可以在[第十三章存储库](https://oreil.ly/XOPbD)的*sample_code/io_friends*目录中找到`countLetters`和`buildGZipReader`的代码。

因为有用于读取和写入的标准接口，`io`包中有一个用于从`io.Reader`复制到`io.Writer`的标准函数，`io.Copy`。还有其他标准函数用于为现有的`io.Reader`和`io.Writer`实例添加新功能。其中包括以下内容：

`io.MultiReader`

返回一个从多个`io.Reader`实例依次读取的`io.Reader`。

`io.LimitReader`

返回一个从提供的`io.Reader`读取指定字节数的`io.Reader`。

`io.MultiWriter`

返回一个 `io.Writer`，可以同时向多个 `io.Writer` 实例写入数据。

标准库中的其他包提供了它们自己的类型和函数来处理 `io.Reader` 和 `io.Writer`。你已经看到了其中的一些，但还有更多。这些包括压缩算法、存档、加密、缓冲区、字节切片和字符串。

`io` 包中还定义了其他单方法接口，例如 `io.Closer` 和 `io.Seeker`：

```go
type Closer interface {
        Close() error
}

type Seeker interface {
        Seek(offset int64, whence int) (int64, error)
}
```

`io.Closer` 接口由像 `os.File` 这样的类型实现，它们在读取或写入完成后需要进行清理。通常情况下，通过 `defer` 调用 `Close`：

```go
f, err := os.Open(fileName)
if err != nil {
    return nil, err
}
defer f.Close()
// use f
```

###### 警告

如果在循环中打开资源，请不要使用 `defer`，因为它不会在函数退出之前运行。相反，应在循环迭代结束前调用 `Close`。如果存在可能导致退出的错误，则也必须在那里调用 `Close`。

`io.Seeker` 接口用于对资源进行随机访问。`whence` 的有效值为常量 `io.SeekStart`、`io.SeekCurrent` 和 `io.SeekEnd`。如果使用自定义类型来明确此点会更好，但出人意料的设计疏忽，`whence` 的类型却是 `int`。

`io` 包定义了几种方式将这四种接口组合起来的接口。它们包括 `io.ReadCloser`、`io.ReadSeeker`、`io.ReadWriteCloser`、`io.ReadWriteSeeker`、`io.ReadWriter`、`io.WriteCloser` 和 `io.WriteSeeker`。使用这些接口来指定你的函数将如何处理数据。例如，不要只将 `os.File` 作为参数传递，而是使用接口精确地指定你的函数将如何处理参数。这不仅使得你的函数更具通用性，也使你的意图更加清晰。此外，如果你正在编写自己的数据源和接收器，应使你的代码兼容这些接口。总的来说，应力求创建与 `io` 包定义的接口一样简单和解耦的接口。它们展示了简单抽象的力量。

除了 `io` 包中的接口外，还有几个用于常见操作的辅助函数。例如，`io.ReadAll` 函数从 `io.Reader` 中读取所有数据到一个字节切片中。`io` 中的一种更聪明的函数展示了一种向 Go 类型添加方法的模式。如果你有一个实现了 `io.Reader` 但没有实现 `io.Closer` 的类型（例如 `strings.Reader`），并且需要将其传递给期望 `io.ReadCloser` 的函数，则可以将你的 `io.Reader` 传递给 `io.NopCloser`，从而得到一个实现了 `io.ReadCloser` 的类型。如果查看其实现，你会发现它非常简单：

```go
type nopCloser struct {
    io.Reader
}

func (nopCloser) Close() error { return nil }

func NopCloser(r io.Reader) io.ReadCloser {
    return nopCloser{r}
}
```

如果需要为某种类型添加额外的方法以满足某个接口，可以使用嵌入类型模式。

###### 注意

`io.NopCloser` 函数违反了不从函数返回接口的一般规则，但它是一个简单的适配器，用于保证接口保持不变，因为它是标准库的一部分。

`os`包包含与文件交互的函数。函数`os.ReadFile`和`os.WriteFile`分别将整个文件读入字节片段并将字节片段写入文件。这些函数（以及`io.ReadAll`）适用于小数据量，但不适合大数据源。处理较大数据源时，请使用`os`包中的`Create`、`NewFile`、`Open`和`OpenFile`函数。它们返回一个实现了`io.Reader`和`io.Writer`接口的`*os.File`实例。您可以将`*os.File`实例与`bufio`包中的`Scanner`类型一起使用。

# 时间

与大多数语言一样，Go 标准库包含时间支持，通常在`time`包中。用于表示时间的两个主要类型是`time.Duration`和`time.Time`。

时间段由`time.Duration`表示，这是基于`int64`的类型。Go 能表示的最小时间单位是纳秒，但`time`包定义了`time.Duration`类型的常量，表示纳秒、微秒、毫秒、秒、分钟和小时。例如，表示 2 小时 30 分钟的持续时间如下：

```go
d := 2 * time.Hour + 30 * time.Minute // d is of type time.Duration
```

这些常量使得使用`time.Duration`既可读性强又类型安全。它们展示了类型化常量的良好使用。

Go 定义了一种合理的字符串格式，一系列数字，可以使用`time.ParseDuration`函数解析为`time.Duration`。此格式在标准库文档中有描述：

> 时间段字符串是一系列可能带有可选小数部分和单位后缀的十进制数字，例如“300ms”、“-1.5h”或“2h45m”。有效的时间单位有“ns”、“us”（或“µs”）、“ms”、“s”、“m”、“h”。
> 
> [Go 标准库文档](https://oreil.ly/wmZdy)

`time.Duration`上定义了几种方法。它符合`fmt.Stringer`接口，并通过`String`方法返回格式化的持续时间字符串。它还具有将值作为小时数、分钟数、秒数、毫秒数、微秒数或纳秒数返回的方法。`Truncate`和`Round`方法将`time.Duration`截断或四舍五入到指定的`time.Duration`单位。

时间的瞬间由`time.Time`类型表示，带有时区信息。使用`time.Now`函数获取当前时间的引用，返回一个设置为当前本地时间的`time.Time`实例。

###### 提示

`time.Time`实例包含时区信息，因此不应使用`==`来检查两个`time.Time`实例是否指向同一时刻。相反，请使用`Equal`方法，该方法会校正时区。

`time.Parse`函数将`string`转换为`time.Time`，而`Format`方法将`time.Time`转换为`string`。虽然 Go 通常采纳在过去表现良好的想法，但它使用[自己的日期和时间格式语言](https://oreil.ly/yfm_V)。它依赖于格式化日期和时间为 2006 年 1 月 2 日下午 3:04:05PM MST（山区标准时间）来指定您的格式。

###### 注意

为什么选择这个日期？因为它的每一部分都按照顺序代表从 1 到 7 的数字，也就是说，01/02 03:04:05PM ’06 -0700（MST 比 UTC 提前 7 小时）。

例如，以下代码

```go
t, err := time.Parse("2006-01-02 15:04:05 -0700", "2023-03-13 00:00:00 +0000")
if err != nil {
    return err
}
fmt.Println(t.Format("January 2, 2006 at 3:04:05PM MST"))
```

打印出以下输出：

```go
March 13, 2023 at 12:00:00AM UTC
```

尽管用于格式化的日期和时间旨在成为一个聪明的记忆技巧，但我发现很难记住，每次想要使用它时都必须查找。幸运的是，在`time`包中，最常用的日期和时间格式都已经被赋予了自己的常量。

就像在`time.Duration`上有提取部分的方法一样，在`time.Time`上也定义了类似的方法，包括`Day`、`Month`、`Year`、`Hour`、`Minute`、`Second`、`Weekday`、`Clock`（返回`time.Time`的时间部分作为单独的小时、分钟和秒`int`值）和`Date`（返回年、月和日作为单独的`int`值）。您可以使用`After`、`Before`和`Equal`方法比较一个`time.Time`实例与另一个。

`Sub`方法返回一个表示两个`time.Time`实例之间经过的时间的`time.Duration`，而`Add`方法返回比当前时间晚`time.Duration`的`time.Time`，`AddDate`方法返回增加指定年、月和日数的新`time.Time`实例。与`time.Duration`一样，还定义了`Truncate`和`Round`方法。所有这些方法都在值接收器上定义，因此它们不会修改`time.Time`实例。

## **单调时间**

大多数操作系统会跟踪两种类型的时间：*挂钟*，它对应于当前时间，以及*单调时钟*，它从计算机启动时开始计数。跟踪两个时钟的原因是挂钟不会均匀地增长。夏令时、闰秒和网络时间协议（NTP）更新可能会使挂钟意外地前进或后退。这在设置定时器或查找经过的时间时可能会导致问题。

为了解决这个潜在问题，Go 语言使用单调时间来跟踪经过的时间，每当设置定时器或使用`time.Now`创建`time.Time`实例时，都会用到单调时间。这种支持是透明的；定时器会自动使用它。`Sub`方法使用单调时钟来计算`time.Duration`，如果两个`time.Time`实例都已设置。如果它们没有（因为一个或两个实例没有使用`time.Now`创建），那么`Sub`方法会使用实例中指定的时间来计算`time.Duration`。

###### 注意

如果你想了解在未正确处理单调时间时可能发生的问题的类型，请查看 Cloudflare 的[博文](https://oreil.ly/IxS2D)，详细描述了 Go 早期版本中由于缺乏单调时间支持而导致的错误。

## 计时器和超时

正如我在“超时代码”中所介绍的，`time`包包含返回通道的函数，这些通道在指定时间后输出值。`time.After`函数返回一个仅输出一次的通道，而`time.Tick`返回的通道在每个指定的`time.Duration`经过后返回一个新值。这些函数与 Go 的并发支持一起使用，以实现超时或定期任务。

你还可以使用`time.AfterFunc`函数在指定的`time.Duration`后触发单个函数运行。不要在非平凡程序之外使用`time.Tick`，因为底层的`time.Ticker`无法关闭（因此无法被垃圾回收）。相反，请使用`time.NewTicker`函数，它返回一个`*time.Ticker`，该类型具有用于监听的通道以及重置和停止计时器的方法。

# encoding/json

REST API 已经将 JSON 确立为服务之间通信的标准方式，而 Go 的标准库包含了将 Go 数据类型与 JSON 相互转换的支持。*Marshaling*一词表示从 Go 数据类型到编码的转换，*unmarshaling*则表示从编码到 Go 数据类型的转换。

## 使用结构体标签添加元数据

假设你正在构建一个订单管理系统，并且必须读取和写入以下 JSON：

```go
{
    "id":"12345",
    "date_ordered":"2020-05-01T13:01:02Z",
    "customer_id":"3",
    "items":[{"id":"xyz123","name":"Thing 1"},{"id":"abc789","name":"Thing 2"}]
}
```

你定义了类型来映射这些数据：

```go
type Order struct {
    ID            string        `json:"id"`
    DateOrdered   time.Time     `json:"date_ordered"`
    CustomerID    string        `json:"customer_id"`
    Items         []Item        `json:"items"`
}

type Item struct {
    ID   string `json:"id"`
    Name string `json:"name"`
}
```

你可以使用*结构体标签*来指定处理 JSON 的规则，这些标签写在结构体字段后面。尽管结构体标签是用反引号标记的字符串，但它们不能超过单行。结构体标签由一个或多个标签/值对组成，写成*`tagName:"tagValue"`*，并用空格分隔。因为它们只是字符串，编译器无法验证它们的格式，但`go vet`可以。还要注意，所有这些字段都是可导出的。与任何其他包一样，`encoding/json`包中的代码无法访问另一个包中未导出的结构体字段。

对于 JSON 处理，使用标签`json`来指定应与结构体字段关联的 JSON 字段的名称。如果未提供`json`标签，则默认行为是假定 JSON 对象字段的名称与 Go 结构体字段的名称相匹配。尽管存在此默认行为，最好还是使用结构体标签显式指定字段的名称，即使字段名称相同。

###### 注意

当将 JSON 解组到没有`json`标签的结构体字段时，名称匹配是不区分大小写的。当将没有`json`标签的结构体字段编组回 JSON 时，JSON 字段的首字母始终大写，因为该字段是可导出的。

如果在编组或解组时应忽略某个字段，请使用破折号（`-`）作为字段名。如果该字段在为空时应从输出中省略，则在字段名后添加`,omitempty`。例如，在`Order`结构中，如果不希望在输出中包含`CustomerID`，如果其设置为空字符串，则结构标记应为`json:"customer_id,omitempty"`。

###### 警告

不幸的是，“空”这一定义与零值并不完全对齐，正如你所期望的那样。结构体的零值并不算作空，但零长度的切片或映射则属于空。

结构标记允许您使用元数据来控制程序的行为。其他语言，尤其是 Java，鼓励开发者在各种程序元素上放置注解，以描述处理方式，而不显式指定处理内容。虽然声明性编程能够实现更简洁的程序，但元数据的自动处理使得理解程序行为变得困难。在一个大型 Java 项目中，有注解的开发者在出现问题时，往往会陷入一片茫然，不知道哪段代码处理了特定的注解，以及做了哪些改变。Go 更倾向于显式代码而不是短小的代码。结构标记永远不会自动评估；它们在将结构实例传递给函数时进行处理。

## 解组和编组

`encoding/json`包中的`Unmarshal`函数用于将字节切片转换为结构体。如果有一个名为`data`的字符串，则以下代码将`data`转换为类型为`Order`的结构体：

```go
var o Order
err := json.Unmarshal([]byte(data), &o)
if err != nil {
    return err
}
```

`json.Unmarshal`函数将数据填充到输入参数中，就像`io.Reader`接口的实现一样。正如我在“指针是最后的选择”中所讨论的，这样可以有效地重复使用同一个结构体，从而控制内存使用。

使用`encoding/json`包中的`Marshal`函数将`Order`实例写回为 JSON，存储在字节切片中：

```go
out, err := json.Marshal(o)
```

这引出了一个问题：你如何能够评估结构标记？你可能还想知道`json.Marshal`和`json.Unmarshal`如何能够读取和写入任何类型的结构体。毕竟，你编写的其他方法仅与程序编译时已知的类型一起工作（即使在类型开关中列出的类型也是预先枚举的）。对这两个问题的答案是反射。你可以在第十六章了解更多关于反射的信息。

## JSON、读者和写者

`json.Marshal` 和 `json.Unmarshal` 函数在字节片上工作。正如你刚才看到的，Go 中的大多数数据源和汇合都实现了 `io.Reader` 和 `io.Writer` 接口。虽然你可以使用 `io.ReadAll` 将 `io.Reader` 的整个内容复制到字节片中，以便 `json.Unmarshal` 读取，但这是低效的。类似地，你可以使用 `json.Marshal` 写入到内存中的字节片缓冲区，然后将该字节片写入网络或磁盘，但最好直接写入到 `io.Writer`。

`encoding/json` 包含两种类型，允许你处理这些情况。`json.Decoder` 和 `json.Encoder` 类型分别从符合 `io.Reader` 和 `io.Writer` 接口的任何地方读取和写入。让我们快速看看它们的工作方式。

从你的数据 `toFile` 开始，它实现了一个简单的结构体：

```go
type Person struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}
toFile := Person {
    Name: "Fred",
    Age:  40,
}
```

`os.File` 类型实现了 `io.Reader` 和 `io.Writer` 接口，因此可以用来演示 `json.Decoder` 和 `json.Encoder`。首先，通过将临时文件传递给 `json.NewEncoder` 将 `toFile` 写入临时文件，该方法返回一个临时文件的 `json.Encoder`。然后，将 `toFile` 传递给 `Encode` 方法：

```go
tmpFile, err := os.CreateTemp(os.TempDir(), "sample-")
if err != nil {
    panic(err)
}
defer os.Remove(tmpFile.Name())
err = json.NewEncoder(tmpFile).Encode(toFile)
if err != nil {
    panic(err)
}
err = tmpFile.Close()
if err != nil {
    panic(err)
}
```

一旦 `toFile` 写入完成，你可以通过将临时文件的引用传递给 `json.NewDecoder`，然后在返回的 `json.Decoder` 上调用 `Decode` 方法，类型为 `Person` 的变量进行 JSON 读取：

```go
tmpFile2, err := os.Open(tmpFile.Name())
if err != nil {
    panic(err)
}
var fromFile Person
err = json.NewDecoder(tmpFile2).Decode(&fromFile)
if err != nil {
    panic(err)
}
err = tmpFile2.Close()
if err != nil {
    panic(err)
}
fmt.Printf("%+v\n", fromFile)
```

你可以在 [Go Playground](https://oreil.ly/eLk64) 上看到完整的示例，或在 [第十三章示例代码/json 目录](https://oreil.ly/XOPbD) 中找到。

## 编码和解码 JSON 流

当你有多个 JSON 结构需要一次性读取或写入时，我们的朋友 `json.Decoder` 和 `json.Encoder` 也可以用于这些情况。

假设你有以下数据：

```go
{"name": "Fred", "age": 40}
{"name": "Mary", "age": 21}
{"name": "Pat", "age": 30}
```

为了本示例，假设它存储在一个名为 `streamData` 的字符串中，但它可以在文件中或甚至是传入的 HTTP 请求中（稍后你将看到 HTTP 服务器如何工作）。

你将这些数据逐个 JSON 对象地存储到你的 `t` 变量中。

就像之前一样，你用数据源初始化你的 `json.Decoder`，但这次你使用 `for` 循环并运行直到出现错误。如果错误是 `io.EOF`，则成功读取所有数据。如果不是，则 JSON 流存在问题。这让你能够逐个 JSON 对象地读取和处理数据：

```go
var t struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

dec := json.NewDecoder(strings.NewReader(streamData))
for {
    err := dec.Decode(&t)
    if err != nil {
        if errors.Is(err, io.EOF) {
            break
        }
        panic(err)
    }
    // process t
}
```

使用 `json.Encoder` 写出多个值的方式与使用它写出单个值的方式完全相同。在这个示例中，你将写入到一个 `bytes.Buffer`，但任何符合 `io.Writer` 接口的类型都可以工作：

```go
var b bytes.Buffer
enc := json.NewEncoder(&b)
for _, input := range allInputs {
    t := process(input)
    err = enc.Encode(t)
    if err != nil {
        panic(err)
    }
}
out := b.String()
```

你可以在 [Go Playground](https://oreil.ly/XGbRQ) 上运行这个示例，或在 [第十三章示例代码/encode_decode 目录](https://oreil.ly/XOPbD) 中找到。

此示例在数据流中有多个未包装在数组中的 JSON 对象，但你也可以使用`json.Decoder`从数组中读取单个对象，而无需一次性加载整个数组到内存中。这可以极大地提高性能并减少内存使用。在[Go 文档](https://oreil.ly/_LTZQ)中有一个示例。

## 自定义 JSON 解析

尽管默认功能通常足够使用，但有时你需要进行覆盖。虽然`time.Time`默认支持 RFC 3339 格式的 JSON 字段，但你可能需要处理其他时间格式。你可以通过创建一个实现`json.Marshaler`和`json.Unmarshaler`两个接口的新类型来处理这个问题：

```go
type RFC822ZTime struct {
    time.Time
}

func (rt RFC822ZTime) MarshalJSON() ([]byte, error) {
    out := rt.Time.Format(time.RFC822Z)
    return []byte(`"` + out + `"`), nil
}

func (rt *RFC822ZTime) UnmarshalJSON(b []byte) error {
    if string(b) == "null" {
        return nil
    }
    t, err := time.Parse(`"`+time.RFC822Z+`"`, string(b))
    if err != nil {
        return err
    }
    *rt = RFC822ZTime{t}
    return nil
}
```

你将一个`time.Time`实例嵌入到一个称为`RFC822ZTime`的新结构体中，这样你仍然可以访问`time.Time`上的其他方法。正如在“指针接收器和值接收器”中讨论的那样，读取时间值的方法声明为值接收器，而修改时间值的方法声明为指针接收器。

然后你改变了`DateOrdered`字段的类型，并可以使用 RFC 822 格式化时间进行处理：

```go
type Order struct {
    ID          string      `json:"id"`
    DateOrdered RFC822ZTime `json:"date_ordered"`
    CustomerID  string      `json:"customer_id"`
    Items       []Item      `json:"items"`
}
```

你可以在[Go Playground](https://oreil.ly/I_cSY)上运行此代码，或在[第十三章的代码库](https://oreil.ly/XOPbD)的*sample_code/custom_json*目录中找到它。

这种方法存在一个哲学上的缺点：JSON 的日期格式决定了数据结构中字段的类型。这是`encoding/json`方法的一个缺点。你可以让`Order`实现`json.Marshaler`和`json.Unmarshaler`，但这需要你编写代码来处理所有字段，即使那些不需要定制支持的字段也是如此。结构体标签格式并没有提供一种指定解析特定字段的函数的方式。这使得你不得不为字段创建一个自定义类型。

另一种选择在 Ukiah Smith 的[博客文章](https://oreil.ly/Jl05c)中有描述。它允许你仅重新定义那些不匹配默认编组行为的字段，通过利用结构体嵌入（在“使用嵌入进行组合”中介绍过）。如果嵌入结构体的字段与包含结构体的同名，那么在 JSON 编组或解组时，该字段将被忽略。

在这个例子中，`Order`的字段看起来像这样：

```go
type Order struct {
    ID          string    `json:"id"`
    Items       []Item    `json:"items"`
    DateOrdered time.Time `json:"date_ordered"`
    CustomerID  string    `json:"customer_id"`
}
```

`MarshalJSON`方法看起来像这样：

```go
func (o Order) MarshalJSON() ([]byte, error) {
    type Dup Order

    tmp := struct {
        DateOrdered string `json:"date_ordered"`
        Dup
    }{
        Dup: (Dup)(o),
    }
    tmp.DateOrdered = o.DateOrdered.Format(time.RFC822Z)
    b, err := json.Marshal(tmp)
    return b, err
}
```

对于`Order`的`MarshalJSON`方法，你定义了一个类型为`Dup`，其基础类型是`Order`。创建`Dup`的原因是，基于另一个类型的类型具有与基础类型相同的字段，但不具有方法。如果没有`Dup`，在调用`json.Marshal`时将会导致`MarshalJSON`的无限循环调用，最终导致堆栈溢出。

你定义了一个具有`DateOrdered`字段和嵌入式`Dup`的匿名结构体。然后将`Order`实例分配给`tmp`中的嵌入字段，在`tmp`中为`DateOrdered`字段分配 RFC822Z 格式的时间，并在`tmp`上调用`json.Marshal`。这会产生所需的 JSON 输出。

`UnmarshalJSON`中也有类似的逻辑：

```go
func (o *Order) UnmarshalJSON(b []byte) error {
    type Dup Order

    tmp := struct {
        DateOrdered string `json:"date_ordered"`
        *Dup
    }{
        Dup: (*Dup)(o),
    }

    err := json.Unmarshal(b, &tmp)
    if err != nil {
        return err
    }

    o.DateOrdered, err = time.Parse(time.RFC822Z, tmp.DateOrdered)
    if err != nil {
        return err
    }
    return nil
}
```

在`UnmarshalJSON`中，对`json.Unmarshal`的调用填充了`o`中的字段（除了`DateOrdered`），因为它被嵌入到`tmp`中。然后，你使用`time.Parse`处理`tmp`中的`DateOrdered`字段，并在`o`中填充`DateOrdered`。

你可以在[Go Playground](https://oreil.ly/JHsdO)上运行此代码，或在[第十三章存储库](https://oreil.ly/XOPbD)的*sample_code/custom_json2*目录中找到它。

这样做可以使`Order`不与 JSON 格式绑定，但`Order`上的`MarshalJSON`和`UnmarshalJSON`方法与 JSON 中时间字段的格式耦合在一起。你无法重用`Order`来支持其他时间格式的 JSON。

为了限制关心 JSON 外观的代码量，定义两个结构体。使用一个结构体进行 JSON 的转换，另一个进行数据处理。将 JSON 读入你的 JSON-aware 类型，然后将其复制到另一个类型。当你想要写出 JSON 时，做相反的操作。这确实会产生一些重复，但它可以使你的业务逻辑不依赖于通信协议。

你可以将`map[string]any`传递给`json.Marshal`和`json.Unmarshal`，在 JSON 和 Go 之间进行双向转换，但在编码探索阶段保存它，并在理解正在处理的数据后用具体类型替换它。Go 使用类型是有原因的；它们文档化了预期数据和预期数据的类型。

尽管 JSON 可能是标准库中最常用的编码器，Go 还包含其他编码器，包括 XML 和 Base64。如果你有一种数据格式需要编码，而标准库或第三方模块中找不到支持，你可以自己编写一个。你将会学习如何在“使用反射编写数据编组器”中实现我们自己的编码器。

###### 警告

标准库包括`encoding/gob`，这是 Go 特有的二进制表示，有点像 Java 中的序列化。就像 Java 序列化是 Enterprise Java Beans 和 Java RMI 的传输协议一样，gob 协议旨在成为`net/rpc`包中 Go 特有 RPC（远程过程调用）实现的传输格式。不要使用`encoding/gob`或`net/rpc`。如果你想要在 Go 中进行远程方法调用，请使用像[GRPC](https://grpc.io)这样的标准协议，这样你就不会被绑定到特定的语言。无论你有多么喜欢 Go，如果你希望你的服务有用，让其他语言的开发者能够调用它们。

# net/http

每种语言都附带了一个标准库，但随着时间的推移，标准库应该包含的预期也在变化。作为在 2010 年代推出的语言，Go 的标准库包括一些其他语言分发版本认为应该由第三方负责的内容：一个高质量的 HTTP/2 客户端和服务器。

## 客户端

`net/http`包定义了一个`Client`类型，用于发出 HTTP 请求和接收 HTTP 响应。在`net/http`包中可以找到一个默认的客户端实例（巧妙地命名为`DefaultClient`），但是在生产应用程序中应避免使用它，因为它默认没有超时设置。相反，应实例化自己的客户端。对于整个程序，只需要创建一个`http.Client`，它可以正确处理跨 goroutine 的多个同时请求：

```go
client := &http.Client{
    Timeout: 30 * time.Second,
}
```

当您想要发出请求时，可以使用`http.NewRequestWithContext`函数创建一个新的`*http.Request`实例，传递一个上下文、方法和要连接的 URL。如果要进行`PUT`、`POST`或`PATCH`请求，请将请求体作为最后一个参数作为`io.Reader`指定。如果没有请求体，请使用`nil`：

```go
req, err := http.NewRequestWithContext(context.Background(),
    http.MethodGet, "https://jsonplaceholder.typicode.com/todos/1", nil)
if err != nil {
    panic(err)
}
```

###### 注意

在第十四章中，我将讨论上下文的概念。

一旦您有了`*http.Request`实例，您可以通过实例的`Headers`字段设置任何标头。使用`http.Client`的`Do`方法和您的`http.Request`，结果将返回为`http.Response`：

```go
req.Header.Add("X-My-Client", "Learning Go")
res, err := client.Do(req)
if err != nil {
    panic(err)
}
```

响应具有几个字段，其中包含有关请求的信息。响应状态的数值代码在`StatusCode`字段中，响应代码的文本在`Status`字段中，响应头在`Header`字段中，任何返回的内容在`Body`类型为`io.ReadCloser`的字段中。这使您可以与`json.Decoder`一起处理 REST API 响应：

```go
defer res.Body.Close()
if res.StatusCode != http.StatusOK {
    panic(fmt.Sprintf("unexpected status: got %v", res.Status))
}
fmt.Println(res.Header.Get("Content-Type"))
var data struct {
    UserID    int    `json:"userId"`
    ID        int    `json:"id"`
    Title     string `json:"title"`
    Completed bool   `json:"completed"`
}
err = json.NewDecoder(res.Body).Decode(&data)
if err != nil {
    panic(err)
}
fmt.Printf("%+v\n", data)
```

您可以在[第十三章的代码库](https://oreil.ly/XOPbD)的*sample_code/client*目录中找到此代码。

###### 警告

`net/http`包中有用于进行`GET`、`HEAD`和`POST`调用的函数。应避免使用这些函数，因为它们使用默认客户端，这意味着它们不会设置请求超时。

## 服务器

HTTP 服务器围绕着`http.Server`和`http.Handler`接口的概念构建。就像`http.Client`发送 HTTP 请求一样，`http.Server`负责监听 HTTP 请求。它是一个性能优越的支持 TLS 的 HTTP/2 服务器。

服务器对请求的处理由分配给`Handler`字段的`http.Handler`接口的实现来处理。此接口定义了一个方法：

```go
type Handler interface {
    ServeHTTP(http.ResponseWriter, *http.Request)
}
```

`*http.Request`应该看起来很熟悉，因为它正是用于向 HTTP 服务器发送请求的类型。`http.ResponseWriter`是一个接口，有三个方法：

```go
type ResponseWriter interface {
        Header() http.Header
        Write([]byte) (int, error)
        WriteHeader(statusCode int)
}
```

这些方法必须按特定顺序调用。首先，调用`Header`以获取`http.Header`的实例，并设置任何你需要的响应头。如果不需要设置任何头部，可以跳过此步骤。接下来，使用你的响应的 HTTP 状态码调用`WriteHeader`。（所有状态码都在`net/http`包中定义为常量。这里本应是定义自定义类型的好地方，但没有这样做；所有状态码常量都是无类型整数。）如果要发送的响应具有 200 状态码，可以跳过`WriteHeader`。最后，调用`Write`方法设置响应体。以下是一个简单处理程序的示例：

```go
type HelloHandler struct{}

func (hh HelloHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Hello!\n"))
}
```

实例化新的`http.Server`与实例化其他结构体一样简单：

```go
s := http.Server{
    Addr:         ":8080",
    ReadTimeout:  30 * time.Second,
    WriteTimeout: 90 * time.Second,
    IdleTimeout:  120 * time.Second,
    Handler:      HelloHandler{},
}
err := s.ListenAndServe()
if err != nil {
    if err != http.ErrServerClosed {
        panic(err)
    }
}
```

`Addr`字段指定服务器监听的主机和端口。如果不指定，服务器将默认监听所有主机的标准 HTTP 端口 80。你可以使用`time.Duration`值设置服务器的读取、写入和空闲超时，以正确处理恶意或损坏的 HTTP 客户端，因为默认行为是根本不设置超时。最后，使用`Handler`字段为服务器指定`http.Handler`。

你可以在[第十三章代码库](https://oreil.ly/XOPbD)的*sample_code/server*目录中找到这段代码。

一个只处理单个请求的服务器并不是非常有用，因此 Go 标准库包含一个请求路由器`*http.ServeMux`。你可以使用`http.NewServeMux`函数创建一个实例。它满足`http.Handler`接口，因此可以分配给`http.Server`的`Handler`字段。它还包含两个方法用于调度请求。第一个方法叫做`Handle`，接受两个参数，一个路径和一个`http.Handler`。如果路径匹配，就会调用`http.Handler`。

虽然可以创建`http.Handler`的实现，但更常见的模式是在`*http.ServeMux`上使用`HandleFunc`方法：

```go
mux.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Hello!\n"))
})
```

此方法接受一个函数或闭包，并将其转换为`http.HandlerFunc`。你在“函数类型是接口的桥梁”中探讨了`http.HandlerFunc`类型。对于简单的处理程序，闭包足够了。对于依赖于其他业务逻辑的更复杂的处理程序，请使用结构体的方法，如“隐式接口使依赖注入更容易”中所示。

Go 1.22 扩展了路径语法，可选择允许 HTTP 动词和路径通配符变量。通配符变量的值通过`http.Request`的`PathValue`方法读取：

```go
mux.HandleFunc("GET /hello/{name}", func(w http.ResponseWriter,
                                         r *http.Request) {
    name := r.PathValue("name")
    w.Write([]byte(fmt.Sprintf("Hello, %s!\n", name)))
})
```

###### 警告

包级别的函数`http.Handle`、`http.HandleFunc`、`http.ListenAndServe`和`http.ListenAndServeTLS`与名为`http.DefaultServeMux`的包级别实例一起工作。不要在非常简单的测试程序之外使用它们。`http.Server`实例是在`http.ListenAndServe`和`http.ListenAndServeTLS`函数中创建的，因此你无法配置服务器属性如超时。此外，第三方库可能已经使用`http.DefaultServeMux`注册了它们自己的处理程序，而没有扫描所有依赖项（直接和间接的）就无法知道这一点。通过避免共享状态，保持你的应用程序受控制。

因为`*http.ServeMux`分派请求给`http.Handler`实例，而且`*http.ServeMux`本身也实现了`http.Handler`接口，所以你可以创建一个`*http.ServeMux`实例，并注册它到父`*http.ServeMux`中：

```go
person := http.NewServeMux()
person.HandleFunc("/greet", func(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("greetings!\n"))
})
dog := http.NewServeMux()
dog.HandleFunc("/greet", func(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("good puppy!\n"))
})
mux := http.NewServeMux()
mux.Handle("/person/", http.StripPrefix("/person", person))
mux.Handle("/dog/", http.StripPrefix("/dog", dog))
```

在这个例子中，请求`/person/greet`由附加到`person`的处理程序处理，而`/dog/greet`由附加到`dog`的处理程序处理。当你将`person`和`dog`注册到`mux`时，使用`http.StripPrefix`辅助函数来移除已经被`mux`处理过的路径部分。你可以在[第十三章代码库](https://oreil.ly/XOPbD)的*sample_code/server_mux*目录中找到这段代码。

### 中间件

HTTP 服务器最常见的需求之一是在多个处理程序中执行一组操作，如检查用户是否已登录、计时请求或检查请求头。Go 使用*中间件模式*处理这些横切关注点。

中间件模式不是使用特殊类型，而是使用一个接受`http.Handler`实例并返回`http.Handler`实例的函数。通常，返回的`http.Handler`是一个转换为`http.HandlerFunc`的闭包。这里有两个中间件生成器，一个提供请求的定时，另一个使用可能是最糟糕的访问控制：

```go
func RequestTimer(h http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        h.ServeHTTP(w, r)
        dur := time.Since(start)
        slog.Info("request time",
            "path", r.URL.Path,
            "duration", dur)
    })
}

var securityMsg = []byte("You didn't give the secret password\n")

func TerribleSecurityProvider(password string) func(http.Handler) http.Handler {
    return func(h http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if r.Header.Get("X-Secret-Password") != password {
                w.WriteHeader(http.StatusUnauthorized)
                w.Write(securityMsg)
                return
            }
            h.ServeHTTP(w, r)
        })
    }
}
```

这两个中间件实现展示了中间件的作用。首先，进行设置操作或检查。如果检查未通过，则在中间件中编写输出（通常使用错误代码）并返回。如果一切正常，则调用处理程序的`ServeHTTP`方法。当返回时，运行清理操作。

`TerribleSecurityProvider`展示了如何创建可配置的中间件。你传递配置信息（在本例中是密码），函数返回使用该配置信息的中间件。这有点令人费解，因为它返回一个返回闭包的闭包。

###### 注意

也许你想知道如何通过中间件层传递值。这通过上下文（context）完成，在第十四章中会详细介绍。

通过将中间件链接起来，你可以将中间件添加到请求处理程序中：

```go
terribleSecurity := TerribleSecurityProvider("GOPHER")

mux.Handle("/hello", terribleSecurity(RequestTimer(
    http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("Hello!\n"))
    }))))
```

我们从 `TerribleSecurityProvider` 获取你的中间件，然后通过一系列函数调用包装你的处理程序。首先调用 `terribleSecurity` 闭包，然后调用 `RequestTimer`，然后调用你的实际请求处理程序。

因为 `*http.ServeMux` 实现了 `http.Handler` 接口，你可以将一组中间件应用于注册到单个请求路由器的所有处理程序：

```go
terribleSecurity := TerribleSecurityProvider("GOPHER")
wrappedMux := terribleSecurity(RequestTimer(mux))
s := http.Server{
    Addr:    ":8080",
    Handler: wrappedMux,
}
```

你可以在 [第十三章仓库](https://oreil.ly/XOPbD) 的 *sample_code/middleware* 目录中找到这段代码。

### 使用第三方模块增强服务器功能。

仅仅因为服务器是生产质量的，并不意味着你不应该使用第三方模块来提升其功能。如果你不喜欢中间件的函数链，可以使用一个名为 [`alice`](https://oreil.ly/_cS1w) 的第三方模块，它允许你使用以下语法：

```go
helloHandler := func(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Hello!\n"))
}
chain := alice.New(terribleSecurity, RequestTimer).ThenFunc(helloHandler)
mux.Handle("/hello", chain)
```

虽然在 Go 1.22 中 `*http.ServeMux` 增加了一些广受欢迎的功能，但其路由和变量支持仍然很基础。嵌套 `*http.ServeMux` 实例也有些笨拙。如果你发现自己需要更高级的功能，比如基于头部值进行路由、使用正则表达式指定路径变量或更好的处理程序嵌套，那么有许多第三方请求路由器可供选择。其中两个最受欢迎的是 [gorilla mux](https://oreil.ly/CrQ4i) 和 [chi](https://oreil.ly/twYcG)。它们都被认为是惯用的，因为它们与 `http.Handler` 和 `http.HandlerFunc` 实例配合使用，展示了使用与标准库相容的可组合库的 Go 哲学。它们还与惯用的中间件一起工作，这两个项目还提供了常见问题的可选中间件实现。

几个流行的 Web 框架还实现了自己的处理程序和中间件模式。其中两个最受欢迎的是 [Echo](https://oreil.ly/7UdEi) 和 [Gin](https://oreil.ly/vvTve)。它们通过包含自动绑定请求或响应数据到 JSON 等功能，简化了 Web 开发。它们还提供了适配器函数，使你可以使用 `http.Handler` 实现，提供了迁移路径。

## ResponseController

在 “接受接口，返回结构体” 中，你学到了修改接口会破坏向后兼容性。你还学到了可以通过定义新接口并使用类型开关和类型断言来检查是否实现了新接口，随时间演变接口。创建这些额外接口的缺点是很难知道它们的存在，使用类型开关来检查它们也很冗长。

你可以在`http`包中找到此类示例。在设计该包时，选择将`http.ResponseWriter`设为接口。这意味着不能在未来的发布版本中向其添加额外的方法，否则将破坏 Go 兼容性保证。为了表示`http.ResponseWriter`实例的新可选功能，`http`包包含了几个可能由`http.ResponseWriter`实现的接口：`http.Flusher`和`http.Hijacker`。这些接口上的方法用于控制响应的输出。

在 Go 1.20 中，`http`包新增了一个具体类型`http.ResponseController`。它展示了向现有 API 添加方法的另一种方式：

```go
func handler(rw http.ResponseWriter, req *http.Request) {
    rc := http.NewResponseController(rw)
    for i := 0; i < 10; i++ {
        result := doStuff(i)
        _, err := rw.Write([]byte(result))
        if err != nil {
            slog.Error("error writing", "msg", err)
            return
        }
        err = rc.Flush()
        if err != nil && !errors.Is(err, http.ErrNotSupported) {
            slog.Error("error flushing", "msg", err)
            return
        }
    }
}
```

在这个示例中，如果`http.ResponseWriter`支持`Flush`，则需要将计算后的数据即时返回给客户端。否则，在所有数据都计算完毕后再返回。工厂函数`http.NewResponseController`接收一个`http.ResponseWriter`并返回一个指向`http.ResponseController`的指针。这个具体类型具有用于`http.ResponseWriter`可选功能的方法。通过将返回的错误与`http.ErrNotSupported`比较，使用`errors.Is`来检查底层`http.ResponseWriter`是否实现了可选方法。你可以在[第十三章存储库](https://oreil.ly/XOPbD)的*sample_code/response_controller*目录中找到这段代码。

因为`http.ResponseController`是一个具体类型，它包装了对`http.ResponseWriter`实现的访问，所以可以随着时间的推移向其添加新方法，而不会破坏现有的实现。这使得新功能可以被发现，并提供了一种标准错误检查的方法来检查可选方法的存在或不存在。这种模式是处理接口需要演变的情况的一种有趣方式。事实上，`http.ResponseController`包含两个没有对应接口的方法：`SetReadDeadline`和`SetWriteDeadline`。未来可能会通过这种技术向`http.ResponseWriter`添加新的可选方法。

# 结构化日志

自其首次发布以来，Go 标准库包含了一个简单的日志包`log`。虽然对于小型程序来说很好用，但它不容易生成*结构化*日志。现代 Web 服务可能有数百万同时在线用户，在这种规模下，您需要软件来处理日志输出以理解发生的情况。结构化日志使用每个日志条目的文档化格式，使得编写处理日志输出并发现模式和异常的程序变得更加容易。

JSON 通常用于结构化日志，但甚至是使用空格分隔的键值对比起不将值分隔为字段的非结构化日志更易处理。虽然您当然可以通过使用`log`包来编写 JSON，但它不提供任何简化结构化日志创建的支持。`log/slog`包解决了这个问题。

将`log/slog`添加到标准库展示了几个良好的 Go 库设计实践。第一个良好的决定是在标准库中包含结构化记录。拥有标准化的结构化记录器使得编写协同工作的模块变得更加容易。已发布了几个第三方结构化记录器来解决`log`的不足，包括[zap](https://oreil.ly/gkd0p)，[logrus](https://oreil.ly/7QpFC)，[go-kit log](https://oreil.ly/Obk0L)等等。碎片化的记录生态系统的问题在于您希望控制日志输出的位置以及记录的消息级别。如果您的代码依赖于使用不同记录器的第三方模块，这将变得不可能。避免记录分片的通常建议是不要在作为库的模块中记录，但这是不可强制执行的，并且使得监视第三方库中发生的情况更加困难。`log/slog`包在 Go 1.21 中是新推出的，但它解决了这些不一致性的事实使得它可能在未来几年内被广泛应用于大多数 Go 程序中。

第二个良好的决定是将结构化记录作为其自己的包，而不是`log`包的一部分。尽管这两个包有着类似的目的，但它们有着非常不同的设计哲学。试图将结构化记录添加到非结构化记录包中会混淆 API。通过将它们作为独立的包，您一眼就能知道`slog.Info`是结构化记录，而`log.Print`是非结构化记录，即使您记不清`Info`是用于结构化还是非结构化记录。

下一个良好的决定是使`log/slog` API 可扩展化。它从简单开始，通过函数提供默认记录器：

```go
func main() {
    slog.Debug("debug log message")
    slog.Info("info log message")
    slog.Warn("warning log message")
    slog.Error("error log message")
}
```

这些函数允许您以各种记录级别记录简单消息。输出如下所示：

```go
2023/04/20 23:13:31 INFO info log message
2023/04/20 23:13:31 WARN warning log message
2023/04/20 23:13:31 ERROR error log message
```

有两件事需要注意。首先，默认记录器默认抑制调试消息。稍后在讨论如何创建自己的记录器时，您将看到如何控制记录级别。

第二点则更加微妙。虽然这是纯文本输出，但它利用空白来生成结构化日志。第一列是年/月/日格式的日期。第二列是 24 小时制的时间。第三列是记录级别。最后是消息内容。

结构化记录的强大之处在于能够添加具有自定义值的字段。通过一些自定义字段更新您的日志：

```go
userID := "fred"
loginCount := 20
slog.Info("user login",
    "id", userID,
    "login_count", loginCount)
```

你可以像之前一样使用相同的函数，但现在可以添加可选参数。可选参数成对出现。第一部分是键，应为字符串。第二部分是值。此日志行输出以下内容：

```go
2023/04/20 23:36:38 INFO user login id=fred login_count=20
```

在消息之后，你有键-值对，再次以空格分隔。

虽然这种文本格式比非结构化日志更容易解析，但你可能希望使用类似 JSON 的东西。你可能还希望自定义日志的写入位置或日志级别。为此，你可以创建一个结构化日志实例：

```go
options := &slog.HandlerOptions{Level: slog.LevelDebug}
handler := slog.NewJSONHandler(os.Stderr, options)
mySlog := slog.New(handler)
lastLogin := time.Date(2023, 01, 01, 11, 50, 00, 00, time.UTC)
mySlog.Debug("debug message",
    "id", userID,
    "last_login", lastLogin)
```

你正在使用`slog.HandlerOptions`结构来定义新日志记录器的最低日志级别。然后，你使用`slog.HandlerOptions`上的`NewJSONHandler`方法来创建一个`slog.Handler`，将日志使用 JSON 写入指定的`io.Writer`。在这种情况下，你使用标准错误输出。最后，你使用`slog.New`函数创建一个包装了`slog.Handler`的`*slog.Logger`。然后，你创建一个`lastLogin`值来记录，还有一个用户 ID。这将产生以下输出：

```go
{"time":"2023-04-22T23:30:01.170243-04:00","level":"DEBUG",
 "msg":"debug message","id":"fred","last_login":"2023-01-01T11:50:00Z"}
```

如果 JSON 和文本不能满足你的输出需求，你可以定义自己实现`slog.Handler`接口的实现，并将其传递给`slog.New`。

最后，`log/slog`包考虑了性能问题。如果你不小心，你的程序可能会花更多时间写日志，而不是执行它设计进行的工作。你可以选择以多种方式将数据写入`log/slog`。你已经看到了最简单（但最慢）的方法，即在`Debug`、`Info`、`Warn`和`Error`方法上交替使用键和值。为了提高性能并减少分配次数，建议使用`LogAttrs`方法：

```go
mySlog.LogAttrs(ctx, slog.LevelInfo, "faster logging",
                slog.String("id", userID),
                slog.Time("last_login", lastLogin))
```

第一个参数是`context.Context`，接下来是日志级别，然后是零个或多个`slog.Attr`实例。对于最常用的类型，有工厂函数可用，对于没有提供函数的类型，你可以使用`slog.Any`。

由于兼容性承诺，`log`包不会被移除。使用它的现有程序将继续工作，同样适用于使用第三方结构化日志记录器的程序。如果你的代码使用了`log.Logger`，`slog.NewLogLogger`函数提供了一个桥接到原始`log`包的方法。它创建一个使用`slog.Handler`来写输出的`log.Logger`实例：

```go
myLog := slog.NewLogLogger(mySlog.Handler(), slog.LevelDebug)
myLog.Println("using the mySlog Handler")
```

这会产生以下输出：

```go
{"time":"2023-04-22T23:30:01.170269-04:00","level":"DEBUG",
 "msg":"using the mySlog Handler"}
```

你可以在[第十三章代码库](https://oreil.ly/XOPbD)的*sample_code/structured_logging*目录中找到所有`log/slog`的编码示例。

`log/slog` API 包括更多功能，包括动态日志级别支持、上下文支持（上下文在第十四章中讨论），值分组和创建共同的值头。你可以通过查看其[API 文档](https://oreil.ly/LRhGf)来了解更多信息。最重要的是，看看`log/slog`是如何组合的，以便学习如何构建自己的 API。

# 练习

现在你已经更多地了解了标准库，通过这些练习来巩固你所学的知识。解决方案在[第十三章代码库](https://oreil.ly/XOPbD)的*exercise_solutions*目录中。

1.  编写一个小的 Web 服务器，当你发送`GET`命令时返回当前时间的 RFC 3339 格式。如果愿意，可以使用第三方模块。

1.  编写一个小的中间件组件，使用 JSON 结构化日志记录每个传入请求到你的 Web 服务器的 IP 地址。

1.  添加以 JSON 格式返回时间的功能。使用`Accept`头来控制返回 JSON 还是文本（默认为文本）。JSON 应按以下结构进行组织：

    ```go
    {
        "day_of_week": "Monday",
        "day_of_month": 10,
        "month": "April",
        "year": 2023,
        "hour": 20,
        "minute": 15,
        "second": 20
    }
    ```

# 结语

在本章中，你看到了标准库中一些最常用的包，并了解了它们如何体现应在你的代码中效仿的最佳实践。你还看到了其他合理的软件工程原则：在经验丰富的情况下可能会做出不同的决策，以及如何尊重向后兼容性，从而可以在坚实的基础上构建应用程序。

在下一章中，你将看到上下文、通过 Go 代码传递状态和计时器的包和模式。
