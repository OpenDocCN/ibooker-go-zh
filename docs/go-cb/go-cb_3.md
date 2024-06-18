# 第四章：CSV 配方

# 4.0 简介

CSV 格式是一种文件格式，可以在文本编辑器中轻松编写和读取表格数据（数字和文本）。CSV 得到广泛支持，大多数电子表格程序，如 Microsoft Excel 和 Apple Numbers，都支持 CSV。因此，许多编程语言，包括 Go，都提供了生成和消耗 CSV 文件数据的库。

或许你会感到惊讶，CSV 已经存在了将近 50 年了。IBM Fortran 编译器在 1972 年的 OS/360 中支持它。如果你对此不是很确定，那么 OS/360 是 IBM 为他们的 System/360 主机计算机开发的批处理操作系统。因此，是的，CSV 最早的用途之一是用于 IBM 主机上的 Fortran 程序。

CSV（逗号分隔值）并没有很好地标准化，也不是所有的 CSV 格式都是用逗号分隔的。有时可能是制表符、分号或其他分隔符。但是，RFC 4180 有一个 CSV 规范，尽管并不是所有人都遵循这个标准。

Go 标准库有一个`encoding/csv`包，支持 RFC 4180，并帮助我们读写 CSV 文件。

# 4.1 读取 CSV 文件

## 问题

你想要将 CSV 文件读入内存以供使用。

## 解决方案

使用`encoding/csv`包和`csv.ReadAll`将 CSV 文件中的所有数据读取到一个二维字符串数组中。

## 讨论

假设你有这样一个文件：

```go
id,first_name,last_name,email
1,Sausheong,Chang,sausheong@email.com
2,John,Doe,john@email.com
```

第一行是标题，接下来的 2 行是用户数据。以下是打开文件并将其读入二维字符串数组的代码。

```go
file, err := os.Open("users.csv")
if err != nil {
 log.Println("Cannot open CSV file:", err)
}
defer file.Close()
reader := csv.NewReader(file)
rows, err := reader.ReadAll()
if err != nil {
 log.Println("Cannot read CSV file:", err)
}
```

首先，我们使用`os.Open`打开文件。这会创建一个`os.File`结构体（它是一个`io.Reader`），我们可以将其作为参数传递给`csv.NewReader`。`csv.NewReader`创建一个新的`csv.Reader`结构体，可以用来从 CSV 文件中读取数据。有了这个 CSV 读取器，我们可以使用`ReadAll`读取文件中的所有数据，并返回一个二维字符串数组`[][]string`。

一个二维字符串数组？也许你会感到惊讶，如果 CSV 行项是整数？或者布尔值或其他类型？你应该记住 CSV 文件是文本文件，所以除了字符串以外，没有其他方式可以区分一个值是否是其他类型。换句话说，所有的值都被假定为字符串，如果你认为有其他类型，你需要将其强制转换为其他类型。

## 将 CSV 数据解析到结构体中

### 问题

你想要将 CSV 数据解析到结构体中，而不是二维字符串数组。

### 解决方案

首先将 CSV 文件读入一个二维字符串数组，然后将其存储到结构体中。

### 讨论

对于一些其他格式，比如 JSON 或 XML，从文件（或任何地方）读取数据并将其解析到结构体中是很常见的。尽管在 CSV 中也可以做到这一点，但你需要做更多的工作。

假设你想要将数据放入一个 User 结构体中。

```go
type User struct {
   Id         int
   firstName  string
   lastName   string
   email      string
}
```

如果你想要将二维字符串数组中的数据解析到 User 结构体中，你需要自己转换每个项目。

```go
var users []User
for _, row := range rows {
 id, _ := strconv.ParseInt(row[0], 0, 0)
 user := User{Id: int(id),
   firstName: row[1],
   lastName:  row[2],
   email:     row[3],
 }
 users = append(users, user)
}
```

在上面的例子中，因为用户 ID 是整数，我在使用它创建用户结构之前使用了 `strconv.ParseInt` 将字符串转换为整数。

在 for 循环结束时，你将会有一个 User 结构体的数组。如果你打印出来，你应该会看到这样的结果。

```go
{0  first_name  last_name  email}
{1  Sausheong  Chang  sausheong@email.com}
{2  John  Doe  john@email.com}
```

# 4.2 移除标题行

## 问题

如果你的 CSV 文件有一行标头作为列标签，你也会得到它在返回的二维字符串数组或结构体数组中。你想把它移除。

## 解决方案

使用 `Read` 读取第一行，然后继续读取剩下的行。

## 讨论

当你在读取器上使用 `Read` 时，你将读取第一行，然后将光标移动到下一行。如果之后使用 `ReadAll`，你可以读取文件的其余部分到你想要的行中。

```go
file, err := os.Open("users.csv")
if err != nil {
 log.Println("Cannot open CSV file:", err)
}
defer file.Close()
reader := csv.NewReader(file)
reader.Read() // use Read to remove the first line
rows, err := reader.ReadAll()
if err != nil {
 log.Println("Cannot read CSV file:", err)
}
```

这将给我们带来这样的东西：

```go
{1  Sausheong  Chang  sausheong@email.com}
{2  John  Doe  john@email.com}
```

# 4.3 使用不同的分隔符

## 问题

CSV 并不一定需要使用逗号作为分隔符。你想读取一个 CSV 文件，其分隔符不是逗号。

## 解决方案

设置 `csv.Reader` 结构中的 `Comma` 变量为文件中使用的分隔符，然后像以前一样读取。

## 讨论

假设我们要读取的文件以分号作为分隔符。

```go
id;first_name;last_name;email
1;Sausheong;Chang;sausheong@email.com
2;John;Doe;john@email.com
```

我们只需在之前创建的 `csv.Reader` 结构中设置 `Comma`，然后像以前一样读取文件。

```go
file, err := os.Open("users2.csv")
if err != nil {
  log.Println("Cannot open CSV file:", err)
}
defer file.Close()
reader := csv.NewReader(file)
reader.Comma = ';' // change Comma to the delimiter in the file
rows, err := reader.ReadAll()
if err != nil {
  log.Println("Cannot read CSV file:", err)
}
```

# 4.4 忽略行

## 问题

当你读取 CSV 文件时，你想忽略某些行。

## 解决方案

在文件中使用注释来指示要忽略的行。然后在 `csv.Reader` 中启用编码，并像以前一样读取文件。

## 讨论

假设你想忽略某些行；你想做的就是简单地注释掉那些行。在 CSV 中你不能这样做，因为注释不是标准。但是使用 Go 的 `encoding/csv` 包，你可以指定一个注释符号，如果你把它放在行的开头，整行都会被忽略。

所以说你有这个 CSV 文件。

```go
id,first_name,last_name,email
1,Sausheong,Chang,sausheong@email.com
# 2,John,Doe,john@email.com
```

要启用注释，只需在我们从 `csv.NewReader` 获取的 `csv.Reader` 结构中设置 `Comment` 变量。

```go
file, err := os.Open("users.csv")
if err != nil {
  log.Println("Cannot open CSV file:", err)
}
defer file.Close()
reader := csv.NewReader(file)
reader.Comment = '#' // lines that start with this will be ignored
rows, err := reader.ReadAll()
if err != nil {
  log.Println("Cannot read CSV file:", err)
}
```

当你运行这个时，你会看到：

```go
{0 first_name last_name email}
{1 Sausheong Chang sausheong@email.com}
```

# 4.5 写入 CSV 文件

## 问题

你想将内存中的数据写入 CSV 文件。

## 解决方案

使用 `encoding/csv` 包和 `csv.Writer` 来写入文件。

## 讨论

我们乐在读取 CSV 文件，现在我们必须写一个。写入与读取非常相似。首先需要创建一个文件（一个 `io.Writer`）。

```go
file, err := os.Create("new_users.csv")
if err != nil {
  log.Println("Cannot create CSV file:", err)
}
defer file.Close()
```

写入文件的数据需要是二维字符串数组。记住，如果数据不是字符串，只需在这之前将其转换为字符串。创建一个带有文件的 `csv.Writer` 结构。之后，你可以在写入器上调用 `WriteAll`，文件就会被创建。这将所有数据写入你的二维字符串数组中的文件。

```go
data := [][]string{
 {"id", "first_name", "last_name", "email"},
 {"1", "Sausheong", "Chang", "sausheong@email.com"},
 {"2", "John", "Doe", "john@email.com"},
}
writer := csv.NewWriter(file)
err = writer.WriteAll(data)
if err != nil {
 log.Println("Cannot write to CSV file:", err)
}
```

# 4.6 一次写入一行到文件

## 问题

不要把所有东西写在我们的二维字符串中，我们想一次写入一行到文件中。

## 解决方案

使用 `csv.Writer` 上的 `Write` 方法来写入单行。

## 讨论

将每一行逐行写入文件几乎相同，只是你需要迭代二维字符串数组以获取每一行，然后调用`Write`方法传递该行。每当需要将缓冲数据写入`Writer`（即文件）时，还需要调用`Flush`方法。在上面的示例中，我在将所有数据写入写入器后调用了`Flush`，但那是因为我没有很多数据。如果有很多行数据，你可能会想定期将数据刷新到文件中。要检查写入或刷新时是否出现问题，可以调用`Error`方法。

```go
writer := csv.NewWriter(file)
for _, row := range data {
 err = writer.Write(row)
 if err != nil {
  log.Println("Cannot write to CSV file:", err)
 }
}
writer.Flush()
```
