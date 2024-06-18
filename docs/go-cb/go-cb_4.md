# 第五章 JSON 配方

# 5.0 引言

JSON（JavaScript 对象表示法）是一种轻量级的数据交换文本格式。它旨在供人类阅读，但也易于机器读取，基于 JavaScript 的一个子集。JSON 最初由 Douglas Crockford 定义，但目前由 RFC 7159 和 ECMA-404 描述。JSON 在基于 REST 的 Web 服务中广泛使用，尽管它们不一定需要接受或返回 JSON 数据。

JSON 在 RESTful web 服务中非常流行，但也经常用于配置。在许多 Web 应用程序中，从获取 Web 服务中的数据，到通过第三方身份验证服务验证 Web 应用程序，再到控制其他服务，创建和消费 JSON 是司空见惯的。

Go 使用`encoding/json`包支持标准库中的 JSON。

# 5.1 解析 JSON 数据字节数组到结构体

## 问题

您希望读取 JSON 数据字节数组并将其存储到结构体中。

## 解决方案

创建结构体以包含 JSON 数据，然后使用`encoding/json`包中的`Unmarshal`函数将数据解封装到结构体中。

## 讨论

使用`encoding/json`包解析 JSON 非常简单：

1.  创建结构体以包含 JSON 数据。

1.  将 JSON 字符串解封装为结构体

让我们来看一个示例 JSON 文件，包含来自 SWAPI（星球大战 API）的卢克·天行者角色数据。您可以直接访问此处的数据 — [*https://swapi.dev/api/people/1*](https://swapi.dev/api/people/1)。我已经将数据存储在名为`skywalker.json`的文件中。

```go
{
	"name": "Luke Skywalker",
	"height": "172",
	"mass": "77",
	"hair_color": "blond",
	"skin_color": "fair",
	"eye_color": "blue",
	"birth_year": "19BBY",
	"gender": "male",
	"homeworld": "https://swapi.dev/api/planets/1/",
	"films": [
		"https://swapi.dev/api/films/1/",
		"https://swapi.dev/api/films/2/",
		"https://swapi.dev/api/films/3/",
		"https://swapi.dev/api/films/6/"
	],
	"species": [],
	"vehicles": [
		"https://swapi.dev/api/vehicles/14/",
		"https://swapi.dev/api/vehicles/30/"
	],
	"starships": [
		"https://swapi.dev/api/starships/12/",
		"https://swapi.dev/api/starships/22/"
	],
	"created": "2014-12-09T13:50:51.644000Z",
	"edited": "2014-12-20T21:17:56.891000Z",
	"url": "https://swapi.dev/api/people/1/"
}
```

要将数据存储在 JSON 中，我们可以创建这样一个结构体。

```go
type Person struct {
	Name      string        `json:"name"`
	Height    string        `json:"height"`
	Mass      string        `json:"mass"`
	HairColor string        `json:"hair_color"`
	SkinColor string        `json:"skin_color"`
	EyeColor  string        `json:"eye_color"`
	BirthYear string        `json:"birth_year"`
	Gender    string        `json:"gender"`
	Homeworld string        `json:"homeworld"`
	Films     []string      `json:"films"`
	Species   []string      `json:"species"`
	Vehicles  []string      `json:"vehicles"`
	Starships []string      `json:"starships"`
	Created   time.Time     `json:"created"`
	Edited    time.Time     `json:"edited"`
	URL       string        `json:"url"`
}
```

在结构体定义每个字段后的字符串字面量称为*结构标记*。Go 使用这些结构标记确定结构字段与 JSON 元素之间的映射。如果映射完全相同，则不需要它们。然而正如你所见，JSON 通常使用蛇形命名法（用下划线替换空格），使用小写字符，而在 Go 中我们使用驼峰命名法（变量无空格，但用一个大写字母表示分隔）。

如您从结构体中看到的，我们可以定义字符串切片以存储 JSON 中的数组，并使用`time.Time`等数据类型。事实上，我们可以使用大多数 Go 数据类型，甚至是映射（尽管只支持具有字符串键的映射）。

将数据解封装到结构体中只需一个函数调用，使用`json.Unmarshal`。

```go
func unmarshal() (person Person) {
	file, err := os.Open("skywalker.json")
	if err != nil {
		log.Println("Error opening json file:", err)
	}
	defer file.Close()

	data, err := io.ReadAll(file)
	if err != nil {
		log.Println("Error reading json data:", err)
	}

	err = json.Unmarshal(data, &person)
	if err != nil {
		log.Println("Error unmarshalling json data:", err)
	}
	return
}
```

在上述代码中，从文件读取数据后，我们创建一个`Person`结构体，然后使用`json.Unmarshal`将数据解封装到其中。

JSON 数据来自 Star Wars API，所以让我们通过 API 直接获取并稍作乐趣。我们使用`http.Get`函数传入 URL，但其他一切都一样。

```go
func unmarshalAPI() (person Person) {
	r, err := http.Get("https://swapi.dev/api/people/1")
	if err != nil {
		log.Println("Cannot get from URL", err)
	}
	defer r.Body.Close()

	data, err := io.ReadAll(r.Body)
	if err != nil {
		log.Println("Error reading json data:", err)
	}

	err = json.Unmarshal(data, &person)
	if err != nil {
		log.Println("Error unmarshalling json data:", err)
	}
	return
}
```

如果你打印`Person`结构体，这就是我们应该得到的结果（输出已美化）。

```go
json.Person{
    Name:      "Luke Skywalker",
    Height:    "172",
    Mass:      "77",
    HairColor: "blond",
    SkinColor: "fair",
    EyeColor:  "blue",
    BirthYear: "19BBY",
    Gender:    "male",
    Homeworld: "https://swapi.dev/api/planets/1/",
    Films:     {"https://swapi.dev/api/films/1/", "https://swapi.dev/api/films/2/",
    "https://swapi.dev/api/films/3/", "https://swapi.dev/api/films/6/"},
    Species:   {},
    Vehicles:  {"https://swapi.dev/api/vehicles/14/",
      "https://swapi.dev/api/vehicles/30/"},
    Starships: {"https://swapi.dev/api/starships/12/",
      "https://swapi.dev/api/starships/22/"},
    Created:   time.Date(2014, time.December, 9, 13, 50, 51, 644000000, time.UTC),
    Edited:    time.Date(2014, time.December, 20, 21, 17, 56, 891000000, time.UTC),
    URL:       "https://swapi.dev/api/people/1/",
}
```

# 5.2 解析非结构化 JSON 数据

## 问题

您想解析一些 JSON 数据，但不知道 JSON 数据的结构或属性足够提前来构建结构体，或者键到值的映射是动态的。

## 解决方案

我们使用与之前相同的方法，但是不使用预定义的结构体，而是使用字符串映射到空接口来存储数据。

## 讨论

明确星球大战 API 的结构。但并非总是如此。有时我们根本不知道结构足够清晰以创建结构体，而且没有可用的文档。此外，有时键到值的映射可能是动态的。看看这个 JSON。

```go
{
    "Luke Skywalker": [
        "https://swapi.dev/api/films/1/",
        "https://swapi.dev/api/films/2/",
        "https://swapi.dev/api/films/3/",
        "https://swapi.dev/api/films/6/"
       ],
    "C-3P0": [
        "https://swapi.dev/api/films/1/",
        "https://swapi.dev/api/films/2/",
        "https://swapi.dev/api/films/3/",
        "https://swapi.dev/api/films/4/",
        "https://swapi.dev/api/films/5/",
        "https://swapi.dev/api/films/6/"
       ],
    "R2D2": [
        "https://swapi.dev/api/films/1/",
        "https://swapi.dev/api/films/2/",
        "https://swapi.dev/api/films/3/",
        "https://swapi.dev/api/films/4/",
        "https://swapi.dev/api/films/5/",
        "https://swapi.dev/api/films/6/"
       ],
    "Darth Vader": [
        "https://swapi.dev/api/films/1/",
        "https://swapi.dev/api/films/2/",
        "https://swapi.dev/api/films/3/",
        "https://swapi.dev/api/films/6/"
       ]
}
```

显然，从 JSON 中，键不一致，并且可以随着字符的添加而改变。对于这种情况，我们如何解析 JSON 数据？我们可以使用字符串映射到空接口，而不是预定义的结构体。让我们看一下代码。

```go
func unstructured() (output map[string]interface{}) {
	file, err := os.Open("unstructured.json")
	if err != nil {
		log.Println("Error opening json file:", err)
	}
	defer file.Close()

	data, err := io.ReadAll(file)
	if err != nil {
		log.Println("Error reading json data:", err)
	}

	err = json.Unmarshal(data, &output)
	if err != nil {
		log.Println("Error unmarshalling json data:", err)
	}
	return
}
```

让我们看看输出。

```go
map[string]interface {}{
    "C-3P0": []interface {}{
        "https://swapi.dev/api/films/1/",
        "https://swapi.dev/api/films/2/",
        "https://swapi.dev/api/films/3/",
        "https://swapi.dev/api/films/4/",
        "https://swapi.dev/api/films/5/",
        "https://swapi.dev/api/films/6/",
    },
    "Darth Vader": []interface {}{
        "https://swapi.dev/api/films/1/",
        "https://swapi.dev/api/films/2/",
        "https://swapi.dev/api/films/3/",
        "https://swapi.dev/api/films/6/",
    },
    "Luke Skywalker": []interface {}{
        "https://swapi.dev/api/films/1/",
        "https://swapi.dev/api/films/2/",
        "https://swapi.dev/api/films/3/",
        "https://swapi.dev/api/films/6/",
    },
    "R2D2": []interface {}{
        "https://swapi.dev/api/films/1/",
        "https://swapi.dev/api/films/2/",
        "https://swapi.dev/api/films/3/",
        "https://swapi.dev/api/films/4/",
        "https://swapi.dev/api/films/5/",
        "https://swapi.dev/api/films/6/",
    },
}
```

让我们尝试将同样的代码应用于早期的卢克·天行者 JSON 数据，并看看输出。

```go
map[string]interface {}{
    "birth_year": "19BBY",
    "created":    "2014-12-09T13:50:51.644000Z",
    "edited":     "2014-12-20T21:17:56.891000Z",
    "eye_color":  "blue",
    "films":      []interface {}{
        "https://swapi.dev/api/films/1/",
        "https://swapi.dev/api/films/2/",
        "https://swapi.dev/api/films/3/",
        "https://swapi.dev/api/films/6/",
    },
    "gender":     "male",
    "hair_color": "blond",
    "height":     "172",
    "homeworld":  "https://swapi.dev/api/planets/1/",
    "mass":       "77",
    "name":       "Luke Skywalker",
    "skin_color": "fair",
    "species":    []interface {}{
    },
    "starships": []interface {}{
        "https://swapi.dev/api/starships/12/",
        "https://swapi.dev/api/starships/22/",
    },
    "url":      "https://swapi.dev/api/people/1/",
    "vehicles": []interface {}{
        "https://swapi.dev/api/vehicles/14/",
        "https://swapi.dev/api/vehicles/30/",
    },
}
```

您可能认为这比尝试弄清楚结构体要容易和简单得多！而且它更具宽容性和灵活性，为什么不使用这个呢？实际上使用结构体具有其优势。使用空接口基本上使数据结构无类型。结构体可以捕获 JSON 中的错误，而空接口则简单地让其通过。

从结构体中检索数据比从映射中更容易，因为您确切知道哪些字段是可用的。此外，您需要进行类型断言才能从接口中获取数据。例如，假设我们想要获取上面提到的出现达斯·维达的电影，所以您可能认为可以这样做。

```go
unstruct := unstructured()
vader := unstruct["Darth Vader"]
first := vader[0]
```

你不能 — 你会看到这个错误而不是。

```go
invalid operation: vader[0] (type interface {} does not support indexing)
```

这是因为变量`vader`是一个空接口，您必须首先对其进行类型断言，然后才能执行任何操作。

```go
unstruct := unstructured()
vader := unstruct["Darth Vader"].([]interface{})
first := vader[0]
```

通常情况下，您应该尽量使用结构体，而只有在最后一种情况下才使用映射到空接口。

# 5.3 将 JSON 数据流解析为结构体

## 问题

您想从流中解析 JSON 数据。

## 解决方案

创建结构体以包含 JSON 数据。在`encoding/json`包中使用`NewDecoder`创建解码器，然后在解码器上调用`Decode`将数据解码为结构体。

## 讨论

对于 JSON 文件或 API 数据，使用`Unmarshal`简单而直接。但是如果 API 是流式 JSON 数据，会发生什么？在这种情况下，我们不能再使用`Unmarshal`，因为`Unmarshal`需要一次性读取整个文件。相反，`encoding/json`包提供了一个`Decoder`函数来处理数据。

可能很难理解 JSON 数据和流 JSON 数据之间的区别，所以让我们通过比较两个不同的 JSON 文件来看一下它们之间的区别。

在第一个 JSON 文件中，我们有一个包含 3 个 JSON 对象的数组（我截取了部分数据以便更容易阅读）。

```go
[{
"name": "Luke Skywalker",
"height": "172",
"mass": "77",
"hair_color": "blond",
"skin_color": "fair",
"eye_color": "blue",
"birth_year": "19BBY",
"gender": "male"
},
{
"name": "C-3PO",
"height": "167",
"mass": "75",
"hair_color": "n/a",
"skin_color": "gold",
"eye_color": "yellow",
"birth_year": "112BBY",
"gender": "n/a"
},
{
"name": "R2-D2",
"height": "96",
"mass": "32",
"hair_color": "n/a",
"skin_color": "white, blue",
"eye_color": "red",
"birth_year": "33BBY",
"gender": "n/a"
}]
```

要读取这个，我们可以使用`Unmarshal`将其解码为一个`Person`结构体数组。

```go
func unmarshalStructArray() (people []Person) {
	file, err := os.Open("people.json")
	if err != nil {
		log.Println("Error opening json file:", err)
	}
	defer file.Close()

	data, err := io.ReadAll(file)
	if err != nil {
		log.Println("Error reading json data:", err)
	}

	err = json.Unmarshal(data, &people)
	if err != nil {
		log.Println("Error unmarshalling json data:", err)
	}
	return
}
```

这将导致如下输出。

```go
[]json.Person{
    {
        Name:      "Luke Skywalker",
        Height:    "172",
        Mass:      "77",
        HairColor: "blond",
        SkinColor: "fair",
        EyeColor:  "blue",
        BirthYear: "19BBY",
        Gender:    "male",
        Homeworld: "",
        Films:     nil,
        Species:   nil,
        Vehicles:  nil,
        Starships: nil,
        Created:   time.Date(1, time.January, 1, 0, 0, 0, 0, time.UTC),
        Edited:    time.Date(1, time.January, 1, 0, 0, 0, 0, time.UTC),
        URL:       "",
    },
    {
        Name:      "C-3PO",
        Height:    "167",
        Mass:      "75",
        HairColor: "n/a",
        SkinColor: "gold",
        EyeColor:  "yellow",
        BirthYear: "112BBY",
        Gender:    "n/a",
        Homeworld: "",
        Films:     nil,
        Species:   nil,
        Vehicles:  nil,
        Starships: nil,
        Created:   time.Date(1, time.January, 1, 0, 0, 0, 0, time.UTC),
        Edited:    time.Date(1, time.January, 1, 0, 0, 0, 0, time.UTC),
        URL:       "",
    },
    {
        Name:      "R2-D2",
        Height:    "96",
        Mass:      "32",
        HairColor: "n/a",
        SkinColor: "white, blue",
        EyeColor:  "red",
        BirthYear: "33BBY",
        Gender:    "n/a",
        Homeworld: "",
        Films:     nil,
        Species:   nil,
        Vehicles:  nil,
        Starships: nil,
        Created:   time.Date(1, time.January, 1, 0, 0, 0, 0, time.UTC),
        Edited:    time.Date(1, time.January, 1, 0, 0, 0, 0, time.UTC),
        URL:       "",
    },
}
```

这是一个 Person 结构体的数组，在我们解码单个 JSON 数组后得到的结果。但是，当我们得到一系列 JSON 对象的数据流时，这就不再可能了。让我们看看另一个 JSON 文件，这个文件代表了一个 JSON 数据流。

```go
{
"name": "Luke Skywalker",
"height": "172",
"mass": "77",
"hair_color": "blond",
"skin_color": "fair",
"eye_color": "blue",
"birth_year": "19BBY",
"gender": "male"
}
{
"name": "C-3PO",
"height": "167",
"mass": "75",
"hair_color": "n/a",
"skin_color": "gold",
"eye_color": "yellow",
"birth_year": "112BBY",
"gender": "n/a"
}
{
"name": "R2-D2",
"height": "96",
"mass": "32",
"hair_color": "n/a",
"skin_color": "white, blue",
"eye_color": "red",
"birth_year": "33BBY",
"gender": "n/a"
}
```

请注意，这不是单个 JSON 对象，而是连续的 3 个 JSON 对象。这不再是一个有效的 JSON 文件，而是在读取 `http.Response` 结构体的 `Body` 时可能得到的内容。如果尝试使用 `Unmarshal` 读取这些内容，将会得到一个错误输出。

```go
Error unmarshalling json data: invalid character '{' after top-level value
```

然而，你可以通过使用 `Decoder` 来解析它。

```go
func decode(p chan Person) {
	file, err := os.Open("people_stream.json")
	if err != nil {
		log.Println("Error opening json file:", err)
	}
	defer file.Close()

	decoder := json.NewDecoder(file)
	for {
		var person Person
		err = decoder.Decode(&person)
		if err == io.EOF {
			break
		}
		if err != nil {
			log.Println("Error decoding json data:", err)
			break
		}
		p <- person
	}
	close(p)
}
```

首先，我们使用 `json.NewDecoder` 创建一个解码器，并将文件作为读取器传递给它。然后在 `for` 循环中，我们在解码器上调用 `Decode`，将要存储数据的结构体传递给它。如果一切顺利，每次循环时，都会从数据中创建一个新的 `Person` 结构体。然后我们可以使用这些数据。如果读取器中没有更多数据了，即我们遇到了 `io.EOF`，我们将从 `for` 循环中退出。

在上述代码中，我们传入一个通道，在其中每个循环中存储 `Person` 结构体。当我们完成读取文件中的所有 JSON 数据后，我们将关闭该通道。

```go
func main() {
	p := make(chan Person)
	go decode(p)
	for {
		person, ok := <-p
		if ok {
            fmt.Printf("%# v\n", pretty.Formatter(person))
		} else {
			break
		}
	}
}
```

这是上述代码的输出。

```go
json.Person{
    Name:      "Luke Skywalker",
    Height:    "172",
    Mass:      "77",
    HairColor: "blond",
    SkinColor: "fair",
    EyeColor:  "blue",
    BirthYear: "19BBY",
    Gender:    "male",
    Homeworld: "",
    Films:     nil,
    Species:   nil,
    Vehicles:  nil,
    Starships: nil,
    Created:   time.Date(1, time.January, 1, 0, 0, 0, 0, time.UTC),
    Edited:    time.Date(1, time.January, 1, 0, 0, 0, 0, time.UTC),
    URL:       "",
}
json.Person{
    Name:      "C-3PO",
    Height:    "167",
    Mass:      "75",
    HairColor: "n/a",
    SkinColor: "gold",
    EyeColor:  "yellow",
    BirthYear: "112BBY",
    Gender:    "n/a",
    Homeworld: "",
    Films:     nil,
    Species:   nil,
    Vehicles:  nil,
    Starships: nil,
    Created:   time.Date(1, time.January, 1, 0, 0, 0, 0, time.UTC),
    Edited:    time.Date(1, time.January, 1, 0, 0, 0, 0, time.UTC),
    URL:       "",
}
json.Person{
    Name:      "R2-D2",
    Height:    "96",
    Mass:      "32",
    HairColor: "n/a",
    SkinColor: "white, blue",
    EyeColor:  "red",
    BirthYear: "33BBY",
    Gender:    "n/a",
    Homeworld: "",
    Films:     nil,
    Species:   nil,
    Vehicles:  nil,
    Starships: nil,
    Created:   time.Date(1, time.January, 1, 0, 0, 0, 0, time.UTC),
    Edited:    time.Date(1, time.January, 1, 0, 0, 0, 0, time.UTC),
    URL:       "",
}
```

此处可以看到，这里连续打印了 3 个 `Person` 结构体，一个接一个地输出，与之前的数组形式不同。

有时候会出现的一个问题是，什么时候应该使用 `Unmarshal`，什么时候应该使用 `Decode`？

`Unmarshal` 更适合用于单个 JSON 对象，但当你从读取器中获得一系列 JSON 对象时，它将无法正常工作。此外，它的简单性意味着它不太灵活，你只能一次性获取整个 JSON 数据。

`Decode` 另一方面，适用于单个 JSON 对象或流式 JSON 数据。此外，使用 `Decode`，你可以在不需要先完整获取 JSON 数据的情况下，在更细的层次上操作 JSON 数据。这是因为你可以在其传入时检查 JSON，甚至在标记级别上。然而，稍微的缺点是它更冗长。

此外，`Decode` 速度稍快。让我们对两者进行快速基准测试。

```go
var luke []byte = []byte(
	`{
 "name": "Luke Skywalker",
 "height": "172",
 "mass": "77",
 "hair_color": "blond",
 "skin_color": "fair",
 "eye_color": "blue",
 "birth_year": "19BBY",
 "gender": "male"
}`)

func BenchmarkUnmarshal(b *testing.B) {
	var person Person
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		json.Unmarshal(luke, &person)
	}
}

func BenchmarkDecode(b *testing.B) {
	var person Person
	data := bytes.NewReader(luke)
	decoder := json.NewDecoder(data)
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		decoder.Decode(&person)
		data.Seek(0, 0)
	}
}
```

这里我们使用标准的 Go 基准测试工具来对 `Unmarshal` 和 `Decode` 进行基准测试。为了确保我们进行了正确的基准测试，我们在运行测试 `Unmarshal` 和 `Decode` 性能的迭代之前重置计时器。我们在基准测试之前创建解码器，因为我们只需要创建一次解码器，它将包装在流式数据输入的读取器周围。但是因为一旦调用 `Decode`，我们需要将偏移量移动到下一次基准测试循环的起始位置。

我们在命令行中运行此命令来启动基准测试。

```go
$ go test -bench=. -benchmem
```

这就是结果。

```go
goos: darwin
goarch: amd64
pkg: github.com/sausheong/gocookbook/ch10_json
cpu: Intel(R) Core(TM) i7-7920HQ CPU @ 3.10GHz
BenchmarkUnmarshal-8   	  437274	  2494 ns/op     272 B/op    12 allocs/op
BenchmarkDecode-8      	  486051	  2368 ns/op      48 B/op     8 allocs/op
PASS
ok  	github.com/sausheong/gocookbook/ch10_json	6.242s
```

正如你所看到的，`Decode` 只快了一点点，每次操作花费 2258 纳秒（每操作纳秒），而 `Unmarshal` 则需花费 2418 纳秒。然而，`Decode` 每次操作只使用 48 B/op（每次操作字节），比 `Unmarshal` 的 272 B/op 要少得多。

# 5.4 从结构体创建 JSON 数据字节数组

## 问题

您希望从结构体创建 JSON 数据。

## 解决方案

创建结构体，然后使用 `json.Marshal` 包将数据编组成 JSON 字节数组。

## 讨论

创建 JSON 数据本质上是其解析的反向过程：

1.  创建您将从中编组数据的结构体

1.  使用 `json.Marshal` 或 `json.MarshalIndent` 将数据编组为 JSON 字符串

我们将重复使用前一个配方中相同的结构体来解析 JSON。我们还将使用用于从 Star Wars API 解析 JSON 数据的函数。

```go
func main() {
	person := get("https://swapi.dev/api/people/14")

	data, err := json.Marshal(&person)
	if err != nil {
		log.Println("Cannot marshal person:", err)
	}
	err = os.WriteFile("han.json", data, 0644)
	if err != nil {
		log.Println("Cannot write to file", err)
	}
}

func get(url string) Person {
	r, err := http.Get(url)
	if err != nil {
		log.Println("Cannot get from URL", err)
	}
	defer r.Body.Close()

	data, err := os.ReadAll(r.Body)
	if err != nil {
		log.Println("Error reading json data:", err)
	}

	var person Person
	json.Unmarshal(data, &person)
	return person
}
```

`get` 函数返回一个 `Person` 结构体，我们可以用它来将数据编组到文件中。`json.Marshal` 函数将结构体中的数据转换为包含 JSON 字符串的字节数组 `data`。如果您只希望它作为字符串，可以将其转换为字符串并使用。在这里，我们将其传递给 `os.WriteFile` 来创建一个新的 JSON 文件。

```go
{"name":"Han Solo","height":"180","mass":"80","hair_color":"brown",
"skin_color":"fair","eye_color":"brown","birth_year":"29BBY","gender":"male",
"homeworld":"https://swapi.dev/api/planets/22/","films":
["https://swapi.dev/api/films/1/","https://swapi.dev/api/films/2/",
"https://swapi.dev/api/films/3/"],"species":[],"vehicles":[],"starships":
["https://swapi.dev/api/starships/10/","https://swapi.dev/api/starships/22/"],
"created":"2014-12-10T16:49:14.582Z","edited":"2014-12-20T21:17:50.334Z",
"url":"https://swapi.dev/api/people/14/"}
```

这实际上并不是很可读。如果您想要一个更可读的版本，可以改用 `json.MarshalIndent`。您需要添加两个额外的参数，第一个是前缀，第二个是缩进。通常，如果您想要一个干净的 JSON 输出，前缀就是一个空字符串，而缩进则是一个空格。

```go
data, err := json.MarshalIndent(&person, "", " ")
```

这将产生一个更可读的版本。

```go
{
 "name": "Han Solo",
 "height": "180",
 "mass": "80",
 "hair_color": "brown",
 "skin_color": "fair",
 "eye_color": "brown",
 "birth_year": "29BBY",
 "gender": "male",
 "homeworld": "https://swapi.dev/api/planets/22/",
 "films": [
  "https://swapi.dev/api/films/1/",
  "https://swapi.dev/api/films/2/",
  "https://swapi.dev/api/films/3/"
 ],
 "species": [],
 "vehicles": [],
 "starships": [
  "https://swapi.dev/api/starships/10/",
  "https://swapi.dev/api/starships/22/"
 ],
 "created": "2014-12-10T16:49:14.582Z",
 "edited": "2014-12-20T21:17:50.334Z",
 "url": "https://swapi.dev/api/people/14/"
}
```

# 5.5 从结构体创建 JSON 数据流

## 问题

您想要从结构体创建流式 JSON 数据。

## 解决方案

在 `encoding/json` 包中使用 `NewEncoder` 创建一个编码器，传递一个 `io.Writer`。然后在编码器上调用 `Encode` 来将结构体数据编码到流中。

## 讨论

`io.Writer` 接口有一个 `Write` 方法，用于向底层数据流写入字节。我们使用 `NewEncoder` 创建一个围绕写入器的编码器。当我们在编码器上调用 `Encode` 时，它将将 JSON 结构写入写入器。

为了正确显示这一点，我们需要有一些 JSON 结构体。我将使用与之前相同的 Star Wars 人物 API 来创建这些结构体。

```go
func get(url string) (person Person) {
	r, err := http.Get(r, err := http.Get("https://swapi.dev/api/people/" +
      strconv.Itoa(n)))
	if err != nil {
		log.Println("Cannot get from URL", err)
	}
	defer r.Body.Close()

	data, err := os.ReadAll(r.Body)
	if err != nil {
		log.Println("Error reading json data:", err)
	}

	json.Unmarshal(data, &person)
	return
}
```

此 `get` 函数将调用 API 并返回请求的 `Person` 结构体。接下来我们将需要使用这个 `Person` 结构体。

```go
func main() {
	encoder := json.NewEncoder(os.Stdout)
	for i := 1; i < 4; i++ { // we're just retrieving 3 records
		person := get(i)
		encoder.Encode(person)
	}
}
```

正如您所看到的，我们将 `os.Stdout` 用作写入器。实际上，`os.Stdout` 是一个 `os.File` 结构体，但 `File` 也是一个写入器，所以这没问题。它的作用是一次将编码写入 `os.Stdout`。首先，我们使用 `json.NewEncoder` 创建一个编码器，将 `os.Stdout` 作为写入器传递。接下来，在循环中获取一个 `Person` 结构体，并将其传递给 `Encode` 以写入 `os.Stdout`。

当你运行程序时，应该会看到类似这样的东西，但每个 JSON 编码将依次出现。

```go
{"name":"Luke Skywalker","height":"172","mass":"77","hair_color":"blond",
"skin_color":"fair","eye_color":"blue","birth_year":"19BBY","gender":"male",
"homeworld":"https://swapi.dev/api/planets/1/","films":
["https://swapi.dev/api/films/1/","https://swapi.dev/api/films/2/",
"https://swapi.dev/api/films/3/","https://swapi.dev/api/films/6/"],
"species":[],"vehicles":["https://swapi.dev/api/vehicles/14/",
"https://swapi.dev/api/vehicles/30/"],"starships":
["https://swapi.dev/api/starships/12/","https://swapi.dev/api/starships/22/"],
"created":"2014-12-09T13:50:51.644Z","edited":"2014-12-20T21:17:56.891Z",
"url":"https://swapi.dev/api/people/1/"}
{"name":"C-3PO","height":"167","mass":"75","hair_color":"n/a","skin_color":
"gold","eye_color":"yellow","birth_year":"112BBY","gender":"n/a","homeworld":
"https://swapi.dev/api/planets/1/","films":["https://swapi.dev/api/films/1/",
"https://swapi.dev/api/films/2/","https://swapi.dev/api/films/3/",
"https://swapi.dev/api/films/4/","https://swapi.dev/api/films/5/",
"https://swapi.dev/api/films/6/"],"species":["https://swapi.dev/api/species/2/"],
"vehicles":[],"starships":[],"created":"2014-12-10T15:10:51.357Z","edited":
"2014-12-20T21:17:50.309Z","url":"https://swapi.dev/api/people/2/"}
{"name":"R2-D2","height":"96","mass":"32","hair_color":"n/a","skin_color":
"white, blue","eye_color":"red","birth_year":"33BBY","gender":"n/a",
"homeworld":"https://swapi.dev/api/planets/8/","films":
["https://swapi.dev/api/films/1/","https://swapi.dev/api/films/2/",
"https://swapi.dev/api/films/3/","https://swapi.dev/api/films/4/",
"https://swapi.dev/api/films/5/","https://swapi.dev/api/films/6/"],
"species":["https://swapi.dev/api/species/2/"],"vehicles":[],"starships":[],
"created":"2014-12-10T15:11:50.376Z","edited":"2014-12-20T21:17:50.311Z",
"url":"https://swapi.dev/api/people/3/"}
```

如果您对此处的凌乱输出感到恼火，并且想知道是否有类似于`MarshalIndent`的等效方法，答案是肯定的。只需像这样设置编码器，使用`SetIndent`，您就可以开始了。

```go
encoder.SetIndent("", " ")
```

如果您想知道这与`Marshal`的区别是什么——使用`Marshal`您无法做到以上的操作。要使用`Marshal`，您需要将所有内容放入一个对象中，并一次性将其全部编组为 JSON — 您无法逐个流式传输 JSON 编码。

换句话说，如果有 JSON 结构数据传入，您不知道全部数据什么时候会完全传入，或者如果您想要先写出 JSON 编码，那么您需要使用`Encode`。只有当您拥有所有 JSON 数据时，才能使用`Marshal`。

当然，`Encode`也比`Marshal`快。让我们再来看看一些基准测试。

```go
var jsonBytes []byte = []byte(jsonString)
var person Person

func BenchmarkMarshal(b *testing.B) {
    json.Unmarshal(jsonBytes, &person)
    b.ResetTimer()
	for i := 0; i < b.N; i++ {
		data, _ := json.Marshal(person)
		io.Discard.Write(data)
	}
}

func BenchmarkEncoder(b *testing.B) {
	json.Unmarshal(jsonBytes, &person)
	b.ResetTimer()
    encoder := json.NewEncoder(io.Discard)
	for i := 0; i < b.N; i++ {
		encoder.Encode(person)
	}
}
```

在测试之前，我们需要准备好 JSON 结构，因此我在运行基准测试循环之前将数据解组为`Person`结构。我还使用`io.Discard`作为写入器。`io.Discard`是一个写入器，其所有写入调用都将成功，并且在这里使用是最方便的。

要对`Marshal`进行基准测试，我将`Person`结构编组为 JSON，然后将其写入`io.Discard`。要对`Encode`进行基准测试，我创建了一个编码器，它包裹在`io.Discard`周围，然后将`Person`结构编码到其中。与解码器的基准测试一样，我在迭代之前放置了编码器的创建，因为我们只需要创建一次。

这是基准测试的结果。

```go
goos: darwin
goarch: amd64
pkg: github.com/sausheong/go-recipes/io/json
cpu: Intel(R) Core(TM) i7-7920HQ CPU @ 3.10GHz
BenchmarkMarshal-8   	 1983175     614.6 ns/op     288 B/op     2 allocs/op
BenchmarkEncoder-8   	 2284209     500.3 ns/op     128 B/op     1 allocs/op
PASS
ok  	github.com/sausheong/go-recipes/io/json	3.852s
```

与以前一样，`Encode`更快，而且内存使用更少，大约为 128 B/op，而`Marshal`则为 288 B/op。

# 5.6 在结构体中省略字段

## 问题

当将 JSON 结构编组为 JSON 编码时，有时候某些结构变量根本就没有数据。我们希望创建的 JSON 编码会省略掉那些没有任何数据的变量。

## 解决方案

使用`omitempty`标签来定义在编组时可以省略的结构变量。

## 讨论

让我们再来看看`Person`结构。

```go
type Person struct {
	Name      string        `json:"name"`
	Height    string        `json:"height"`
	Mass      string        `json:"mass"`
	HairColor string        `json:"hair_color"`
	SkinColor string        `json:"skin_color"`
	EyeColor  string        `json:"eye_color"`
	BirthYear string        `json:"birth_year"`
	Gender    string        `json:"gender"`
	Homeworld string        `json:"homeworld"`
	Films     []string      `json:"films"`
	Species   []string      `json:"species"`
	Vehicles  []string      `json:"vehicles"`
	Starships []string      `json:"starships"`
	Created   time.Time     `json:"created"`
	Edited    time.Time     `json:"edited"`
	URL       string        `json:"url"`
}
```

您可能会注意到，当人物是人类时，API 没有指定物种，也有许多人物没有交通工具或星际飞船的标签。因此，当我们将结构编组时，它将作为空数组输出。

```go
{
 "name": "Owen Lars",
 "height": "178",
 "mass": "120",
 "hair_color": "brown, grey",
 "skin_color": "light",
 "eye_color": "blue",
 "birth_year": "52BBY",
 "gender": "male",
 "homeworld": "https://swapi.dev/api/planets/1/",
 "films": [
  "https://swapi.dev/api/films/1/",
  "https://swapi.dev/api/films/5/",
  "https://swapi.dev/api/films/6/"
 ],
 "species": [],
 "vehicles": [],
 "starships": [],
 "created": "2014-12-10T15:52:14.024Z",
 "edited": "2014-12-20T21:17:50.317Z",
 "url": "https://swapi.dev/api/people/6/"
}
```

如果您不想显示物种、交通工具或星际飞船，您可以在 JSON 结构标签上使用`omitempty`标签。

```go
Species   []string      `json:"species,omitempty"`
Vehicles  []string      `json:"vehicles,omitempty"`
Starships []string      `json:"starships,omitempty"`
```

当您再次运行相同的代码时，输出中将不再有它们。

```go
{
 "name": "Owen Lars",
 "height": "178",
 "mass": "120",
 "hair_color": "brown, grey",
 "skin_color": "light",
 "eye_color": "blue",
 "birth_year": "52BBY",
 "gender": "male",
 "homeworld": "https://swapi.dev/api/planets/1/",
 "films": [
  "https://swapi.dev/api/films/1/",
  "https://swapi.dev/api/films/5/",
  "https://swapi.dev/api/films/6/"
 ],
 "created": "2014-12-10T15:52:14.024Z",
 "edited": "2014-12-20T21:17:50.317Z",
 "url": "https://swapi.dev/api/people/6/"
}
```

您为什么要这样做呢？在此 API 中，身高和体重是字符串。但是，如果它们是整数，并且您不知道它们的身高或体重，那么默认值将是 0。在这种情况下，这些值是错误的，有时可能是正确的（例如，绝地武士力量的幽灵既没有身高也没有体重），但您无法分辨哪个是正确的。在这种情况下，根本不显示它可能是更好的选择。
