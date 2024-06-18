# 第三章：通用输入/输出实例

# 3.0 简介

输入和输出（或更普遍称为 I/O）是计算机与外部世界通信的方式。I/O 是开发软件的关键部分，因此大多数编程语言，包括 Go，都有能够读取输入和写入输出的标准库。计算机的典型输入通常指的是来自键盘的按键或鼠标的点击或移动，也可以是其他外部来源，如摄像头、麦克风，或游戏手柄等。输出在许多情况下是指显示在屏幕上（或终端上）的内容，或打印在打印机上的内容。I/O 还可以指网络连接，通常也涉及文件。

在本章中，我们将探讨一些用于管理 I/O 的常见 Go 示例。我们将从一些基本的 I/O 示例开始，然后讨论文件的一般情况。在接下来的几章中，我们将继续讨论 CSV，然后是 JSON 和二进制文件。

`io` 包是 Go 中用于输入和输出的基本包。它包含了主要的 I/O 接口和一些便利函数。主要且最常用的接口是 `Reader` 和 `Writer`，但还有一些变体，如 `ReadWriter`、`TeeReader`、`WriterTo` 等等。

通常，这些接口只是一些函数的描述符，例如一个实现了 `Reader` 的结构体就是具有 `Read` 函数的结构体。一个实现了 `WriterTo` 的结构体就是具有 `WriteTo` 函数的结构体。有些接口结合了两个或更多接口，例如 `ReadWriter` 结合了 `Reader` 和 `Writer` 接口，并拥有 `Read` 和 `Write` 函数。

本章将更详细地解释这些接口的使用方式。

# 3.1 从输入中读取数据

## 问题

你想从输入中读取数据。

## 解决方案

使用 `io.Reader` 接口来从输入流中读取数据。

## 讨论

Go 使用 `io.Reader` 接口来表示从数据流中读取数据的能力。Go 标准库以及第三方包中的许多包都使用 `Reader` 接口来允许从中读取数据。

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

任何实现了 `Read` 函数的结构体都是 `Reader`。假设你有一个读取器（实现了 Reader 接口的结构体）。要从读取器中读取数据，你需要创建一个字节切片，并将该切片传递给 `Read` 方法。

```go
bytes = make([]byte, 1024)
reader.Read(bytes)
```

这可能看起来有些反直觉，并且似乎你想从 `bytes` 中读取数据到读取器中，但实际上你是从读取器中读取数据到 `bytes` 中。只需将其想象为数据从左侧的读取器流向右侧的 `bytes`。

`Read` 只会填充其容量的字节。如果你想从读取器中读取所有内容，可以使用 `io.ReadAll` 函数。

```go
bytes, err := os.ReadAll(reader)
```

这看起来更直观，因为 `ReadAll` 从传递给参数的读取器中读取数据，并将数据返回到 `bytes` 中。在这种情况下，数据从右侧的读取器流向左侧的 `bytes` 中。

你经常会发现一些函数期望一个读取器作为输入参数。假设你有一个字符串，你想将这个字符串传递给函数，你可以怎么做呢？你可以使用`strings.NewReader`函数从字符串创建一个读取器，然后将其传递给函数。

```go
str := “My String Data”
reader := strings.NewReader(str)
```

现在你可以将`reader`传递给期望一个读取器的函数。

# 3.2 写入到输出

## 问题

你想要写入到一个输出。

## 解决方案

使用`io.Writer`接口来写入到一个输出。

## 讨论

`io.Writer`接口的工作方式与`io.Reader`相同。

```go
type Writer interface {
	Write(p []byte) (n int, err error)
}
```

当你在`io.Writer`上调用`Write`时，你正在将字节写入到底层数据流中。

```go
bytes = []byte("Hello World")
writer.Write(bytes)
```

你可能注意到，这种调用模式与`io.Reader`在 8.1 章节中的方法相反。在`Reader`中，你调用`Read`方法将数据从结构体读取到`bytes`变量中，而在这里，你调用`Write`方法将数据从`bytes`变量写入到结构体中。在这种情况下，数据从右向左流动，从`bytes`流向写入器。

在 Go 语言中的一个常见模式是，一个函数以写入器作为参数。该函数然后在写入器上调用`Write`函数，稍后你可以从写入器中提取数据。让我们看一个例子。

```go
var buf bytes.Buffer
fmt.Fprintf(&buf, "Hello %s", "World")
s := buf.String() // s == "Hello World"
```

`bytes.Buffer`结构体是一个`Writer`（它实现了`Write`函数），因此你可以轻松地创建一个，并将其传递给`fmt.Fprintf`函数，该函数将`io.Writer`作为其第一个参数。`fmt.Fprintf`函数将数据写入到缓冲区，稍后你可以从中提取数据出来。

在 Go 语言中，通过向写入器写入数据并在稍后提取出来的模式非常常见。另一个例子是在 HTTP 处理程序中使用`http.ResponseWriter`。

```go
func myHandler(w http.ResponseWriter, r *http.Request) {
    w.Write([]bytes("Hello World"))
}
```

在这里，我们将数据写入到`ResponseWriter`，然后将数据作为输入发送回浏览器。

# 3.3 从读取器复制到写入器

## 问题

你想要从一个读取器复制到一个写入器。

## 解决方案

使用`io.Copy`函数从一个读取器复制到一个写入器。

## 讨论

有时我们从一个读取器中读取，因为我们想要将其写入一个写入器。这个过程可能需要几个步骤，从读取器中读取所有内容到缓冲区，然后再将其写入到写入器中。不过，我们可以使用`io.Copy`函数来简化这个过程。`io.Copy`函数一次性从读取器复制到写入器。

让我们看看如何使用`io.Copy`。我们想要下载一个文件，因此我们使用`http.Get`来获取一个读取器，然后我们读取它，接着我们使用`os.WriteFile`来写入文件。

```go
// using a random 1MB test file
var url string = "http://speedtest.ftp.otenet.gr/files/test1Mb.db"

func readWrite() {
	r, err := http.Get(url)
	if err != nil {
		log.Println("Cannot get from URL", err)
	}
	defer r.Body.Close()
	data, _ := os.ReadAll(r.Body)
	os.WriteFile("rw.data", data, 0755)
}
```

当我们使用`http.Get`下载一个文件时，我们会得到一个`http.Response`结构体。文件的内容在`http.Response`结构体的`Body`变量中，这是一个`io.ReadCloser`。`ReadCloser`只是一个接口，它将`Reader`和`Closer`组合在一起，因此我们可以像对待读取器一样处理它。我们使用`os.ReadAll`函数从`Body`中读取数据，然后使用`os.WriteFile`将其写入文件。

这就足够简单了，但让我们看看函数的性能如何。我们使用标准 Go 工具的基准测试能力来完成这个任务。首先，我们创建一个测试文件，就像创建任何其他测试文件一样。

```go
package main

import "testing"

func BenchmarkReadWrite(b *testing.B) {
	readWrite()
}
```

在这个测试文件中，我们创建了一个以 `Benchmarkxxx` 开头的函数，而不是以 `Testxxx` 开头的函数，该函数接受一个指向 `testing.B` 的参数 `b`。

基准函数非常简单，我们只是调用我们的 `readWrite` 函数。让我们从命令行运行它，看看我们的表现如何。

```go
$ go test -bench=. -benchmem
```

我们使用 `-bench=.` 标志告诉 Go 运行所有基准测试，并使用 `-benchmem` 标志显示内存基准。这是你应该看到的内容。

```go
goos: darwin
goarch: amd64
pkg: github.com/sausheong/go-cookbook/io/copy
cpu: Intel(R) Core(TM) i7-7920HQ CPU @ 3.10GHz
BenchmarkReadWrite-8   	       1	1998957055 ns/op	 5892440 B/op
219 allocs/op
PASS
ok  	github.com/sausheong/go-cookbook/io/copy	2.327s
```

我们为一个下载 1 MB 文件的函数运行了一个基准测试。测试只运行了一次，耗时 1.99 秒。内存占用为 5.89 MB，共进行了 219 次内存分配。

可见，下载 1 MB 文件的操作相当昂贵。毕竟，下载 1 MB 文件需要近 6 MB 内存。或者，我们可以使用 `io.Copy` 来完成几乎相同的操作，但内存占用要少得多。

```go
func copy() {
	r, err := http.Get(url)
	if err != nil {
		log.Println("Cannot get from URL", err)
	}
	defer r.Body.Close()
	file, _ := os.Create("copy.data")
	defer file.Close()
	writer := bufio.NewWriter(file)
	io.Copy(writer, r.Body)
	writer.Flush()
}
```

首先，我们需要为数据创建一个文件，这里使用 `os.Create`。接下来，我们使用 `bufio.NewWriter` 创建一个缓冲写入器，将其包装在文件周围。这将在 `Copy` 函数中使用，将响应的内容复制到缓冲写入器中。最后，我们刷新写入器的缓冲区，并使底层写入器将内容写入文件。

如果你运行这个 `copy` 函数，它的工作方式是相同的，但性能如何呢？让我们回到我们的基准测试中，并为这个 `copy` 函数添加另一个基准函数。

```go
package main

import "testing"

func BenchmarkReadWrite(b *testing.B) {
	readWrite()
}

func BenchmarkCopy(b *testing.B) {
	copy()
}
```

我们再次运行基准测试，你应该看到的结果如下。

```go
goos: darwin
goarch: amd64
pkg: github.com/sausheong/go-cookbook/io/copy
cpu: Intel(R) Core(TM) i7-7920HQ CPU @ 3.10GHz
BenchmarkReadWrite-8   	       1	2543665782 ns/op	 5895360 B/op
227 allocs/op
BenchmarkCopy-8        	       1	1426656774 ns/op	   42592 B/op
61 allocs/op
PASS
ok  	github.com/sausheong/go-cookbook/io/copy	4.103s
```

这次，`readWrite` 函数耗时 2.54 秒，使用了 5.89 MB 内存，进行了 227 次内存分配。然而，`copy` 函数只耗时 1.43 秒，使用了 42.6 kB 内存，进行了 61 次内存分配。

`copy` 函数速度大约快了 80%，但内存使用量只有原来的一小部分（不到 1%）。对于非常大的文件，如果你使用 `os.ReadAll` 和 `os.WriteFile`，内存可能会迅速耗尽。

# 3.4 从文本文件读取

## 问题

你想将文本文件读入内存。

## 解决方案

你可以使用 `os.Open` 函数打开文件，然后在文件上使用 `Read`。或者，你也可以使用更简单的 `os.ReadFile` 函数在一个函数调用中完成它。

## 讨论

读取和写入文件系统是编程语言需要做的基本操作之一。当然，你可以始终存储在内存中，但如果需要在关闭后持久保存数据，你需要将其存储在某个地方。数据可以以多种方式持久化，但最常用的可能是本地文件系统。

### 一次性读取所有内容

读取文本文件最简单的方法是使用 `os.ReadFile`。假设我们要从名为 `data.txt` 的文本文件中读取。

```go
hello world!
```

要读取文件，只需将文件名作为参数传递给`os.ReadFile`，然后您就完成了！

```go
data, err := os.ReadFile("data.txt")
if err != nil {
  log.Println("Cannot read file:", err)
}
fmt.Println(string(data))
```

这将打印出`hello world!`。

### 打开文件并从中读取

通过打开文件然后对其进行读取更灵活，但需要更多步骤。首先，您需要打开文件。

```go
// open the file
file, err := os.Open("data.txt")
if err != nil {
  log.Println("Cannot open file:", err)
}
// close the file when we are done with it
defer file.Close()
```

这可以使用`os.Open`完成，它以只读形式返回一个`File`结构。如果要以不同模式打开它，可以使用`os.OpenFile`。使用`defer`关键字设置文件以在函数返回前关闭是一个良好的实践。

接下来，我们需要创建一个字节数组来存储数据。

```go
// get some info from the file
stat, err := file.Stat()
if err != nil {
  log.Println("Cannot read file stats:", err)
}
// create the byte array to store the read data
data := make([]byte, stat.Size())
```

为此，我们需要知道字节数组应该有多大，这应该是文件的大小。我们使用文件上的`Stat`方法获取`FileInfo`结构，可以调用`Size`方法获取文件的大小。

一旦我们有了字节数组，我们可以将其作为参数传递给文件结构上的`Read`方法。

```go
// read the file
bytes, err := file.Read(data)
if err != nil {
  log.Println("Cannot read file:", err)
}
fmt.Printf("Read %d bytes from file\n", bytes)
fmt.Println(string(data))
```

这将把读取的数据存储到字节数组中并返回读取的字节数。如果一切顺利，您应该从输出中看到类似这样的内容。

```go
Read 13 bytes from file
Hello World!
```

还有一些步骤，但您可以在打开文件并读取它之间的过程中灵活读取整个文档的部分，并且还可以做其他事情。

# 3.5 写入文本文件

## 问题

您想要将数据写入文本文件。

## 解决方案

您可以使用`os.Open`函数打开文件，然后在文件上执行`Write`。或者，您可以使用`os.WriteFile`函数一次性完成。

## 讨论

就像读取文件一样，有几种方法可以向文件中写入内容。

### 一次性写入文件

根据数据，您可以使用`os.WriteFile`一次性写入文件。

```go
data := []byte("Hello World!\n")

err := os.WriteFile("data.txt", data, 0644)
if err != nil {
  log.Println("Cannot write to file:", err)
}
```

第一个参数是文件名，数据在一个字节数组中，最后一个参数是您要给文件的 Unix 文件权限。如果文件不存在，这将创建一个新文件。如果存在，它将删除文件中的所有数据并写入新数据，但不更改权限。

### 创建文件并写入

通过创建文件然后写入文件稍微复杂，但也更灵活。首先，您需要使用`os.Create`函数创建或打开文件。

```go
data := []byte("Hello World!\n")
// write to file and read from file using the File struct
file, err := os.Create("data.txt")
if err != nil {
  log.Println("Cannot create file:", err)
}
defer file.Close()
```

如果文件不存在，将创建具有给定名称和模式`0666`的新文件。如果文件已存在，则会删除其中的所有数据。与以前一样，您希望设置文件在函数结束时关闭，使用`defer`关键字。

一旦您拥有文件，您可以直接使用`Write`方法将数据写入其中，并将包含数据的字节数组传递给它。

```go
bytes, err := file.Write(data)
if err != nil {
  log.Println("Cannot write to file:", err)
}
fmt.Printf("Wrote %d bytes to file\n", bytes)
```

这将返回写入文件的字节数。与之前一样，虽然需要更多步骤，但在创建文件和写入文件之间分步骤可以让您以较小的块而不是一次性写入，从而更灵活。

# 3.6 使用临时文件

## 问题

您想要创建一个临时文件供使用，然后处理它后再将其丢弃。

## 解决方案

使用`os.CreateTemp`函数创建一个临时文件，然后在不再需要时删除它。

## 讨论

临时文件是在程序执行某些操作时创建的用于暂存数据的文件。完成任务后，它应该被删除或复制到永久存储。在 Go 中，我们可以使用`os.CreateTemp`函数创建临时文件，然后在需要时删除它。

不同的操作系统将它们的临时文件存储在不同的位置。无论它在哪里，Go 都可以使用`os.TempDir`函数告诉你它在哪里。

```go
fmt.Println(os.TempDir())
```

我们需要知道`os.CreateTemp`创建的临时文件将被创建在哪里。通常我们不关心，但因为我们要逐步分析临时文件的创建过程，我们想确切地知道它在哪里。执行这条语句时，我们应该看到像这样的东西。

```go
/var/folders/nj/2xd4ssp94zz41gnvsyvth38m0000gn/T/
```

这是你的计算机告诉 Go（和其他一些程序）要使用作为临时目录的目录。我们可以直接使用这个目录，或者使用`os.MkdirTemp`函数在这里创建我们自己的目录。

```go
tmpdir, err := os.MkdirTemp(os.TempDir(), "mytmpdir_*")
if err != nil {
	log.Println("Cannot create temp directory:", err)
}
defer os.RemoveAll(tmpdir)
```

`os.MkdirTemp`的第一个参数是临时目录，第二个参数是一个模式字符串。该函数将应用一个随机字符串来替换模式字符串中的`*`。使用`os.RemoveAll`推迟清理临时目录也是一个好习惯。

接下来，我们使用`os.CreateTemp`实际创建临时文件，向其传递我们刚刚创建的临时目录以及文件名的模式字符串，其工作方式与临时目录相同。

```go
tmpfile, err := os.CreateTemp(tmpdir, "mytmp_*")
if err != nil {
	log.Println("Cannot create temp file:", err)
}
```

有了这个，我们有了一个文件，其他操作与任何其他文件一样。

```go
data := []byte("Some random stuff for the temporary file")
_, err = tmpfile.Write(data)
if err != nil {
	log.Println("Cannot write to temp file:", err)
}
err = tmpfile.Close()
if err != nil {
	log.Println("Cannot close temp file:", err)
}
```

如果你选择不将临时文件放入单独的目录中（完成后删除该目录及其中的所有内容），你可以像这样使用`os.Remove`与临时文件名。

```go
defer os.Remove(tmpfile.Name())
```
