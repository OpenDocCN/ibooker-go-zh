# 第十一章：Go 工具

编程语言不是孤立存在的。为了有用，工具必须帮助开发者将源代码转化为可执行文件。由于 Go 旨在解决今天软件工程师面临的问题，并帮助他们构建高质量的软件，因此对简化通常在其他开发平台中困难的任务的工具进行了精心设计。这包括改进构建、格式化、更新、验证和分发的方式，甚至还包括用户如何安装您的代码。

我已经涵盖了许多捆绑的 Go 工具：`go vet`、`go fmt`、`go mod`、`go get`、`go list`、`go work`、`go doc`和`go build`。由`go test`工具提供的测试支持非常广泛，它在第十五章中单独进行了介绍。在本章中，您将探索使 Go 开发变得出色的其他工具，无论是来自 Go 团队还是第三方。

# 使用`go run`来尝试小程序

Go 是一种编译语言，这意味着在运行 Go 代码之前，它必须被转换成可执行文件。这与像 Python 或 JavaScript 这样的解释语言形成对比，后者允许您编写一个快速脚本来测试一个想法并立即执行它。拥有快速反馈循环非常重要，因此 Go 通过`go run`命令提供类似的功能。它一步构建和执行程序。让我们回到第一章的第一个程序。把它放在名为*hello.go*的文件中：

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, world!")
}
```

(你也可以在[第十一章的代码库](https://oreil.ly/Z_Fpg)的*sample_code/gorun*目录中找到这段代码。)

一旦文件保存，使用`go run`命令构建并执行它：

```go
go run hello.go
Hello, world!

```

运行`go run`命令后，查看目录中，你会发现没有保存任何二进制文件；目录中唯一的文件是你刚创建的*hello.go*文件。可执行文件去哪了（无意冒犯）？

`go run`命令实际上确实将您的代码编译成一个二进制文件。但是，该二进制文件是在临时目录中构建的。`go run`命令会构建该二进制文件，从该临时目录执行该二进制文件，然后在程序完成后删除该二进制文件。这使得`go run`命令非常适合测试小程序或像使用脚本语言一样使用 Go。

###### 小贴士

当你希望像运行脚本一样运行 Go 程序并立即执行源代码时，请使用`go run`。

# 使用`go install`添加第三方工具

虽然有些人选择将他们的 Go 程序作为预编译的二进制文件分发，但是用 Go 编写的工具也可以从源代码构建并通过`go install`命令安装到您的计算机上。

如您在“发布您的模块”中看到的，Go 模块通过其源代码存储库进行标识。`go install`命令接受一个参数，即模块源代码存储库中主包的路径，后跟`@`及您想要的工具版本（如果只想获取最新版本，请使用`@latest`）。然后它会下载、编译并安装该工具。

###### 警告

请务必在包名后面包含`@version`或`@latest`！如果不这样做，将触发各种令人困惑的行为，几乎肯定不是您想要的结果。如果当前目录不在模块中，或者当前目录是模块但模块的*go.mod*文件中没有引用该包，您会收到错误消息，或者安装*go.mod*文件中提到的包版本。

默认情况下，`go install`将二进制文件放置在您的主目录中的*go/bin*目录中。设置*GOBIN*环境变量以更改此位置是强烈推荐的。建议您将`go install`目录添加到可执行搜索路径中（这可以通过修改 Unix 和 Windows 上的*PATH*环境变量来完成）。为简单起见，本章中的所有示例假定您已添加了此目录。

###### 注意

其他环境变量由`go`工具识别。您可以使用`go help environment`命令获取完整的列表，以及每个变量的简要描述。其中许多控制低级行为，可以安全地忽略。在需要时，我会指出相关的变量。

一些在线资源告诉您设置`GOROOT`或`GOPATH`环境变量。`GOROOT`指定安装 Go 开发环境的位置，而`GOPATH`用于存储所有 Go 源代码，包括您自己的和第三方依赖项。设置这些变量已不再必要；`go`工具会自动确定`GOROOT`，并且基于`GOPATH`的开发已被模块取代。

让我们看一个快速的例子。Jaana Dogan 创建了一个名为`hey`的出色的 Go 工具，用于对 HTTP 服务器进行负载测试。您可以将其指向您选择的网站或您编写的应用程序。以下是使用`go install`命令安装`hey`的方法：

```go
$ go install github.com/rakyll/hey@latest
go: downloading github.com/rakyll/hey v0.1.4
go: downloading golang.org/x/net v0.0.0-20181017193950-04a2e542c03f
go: downloading golang.org/x/text v0.3.0

```

这将下载`hey`及其所有依赖项，构建程序并将二进制文件安装到您的 Go 二进制目录中。

###### 注意

正如我在“模块代理服务器”中介绍的，Go 存储库的内容被缓存在代理服务器中。根据`GOPROXY`环境变量中的存储库和值，`go install`可能会从代理服务器下载或直接从存储库下载。如果`go install`直接从存储库下载，则依赖于您计算机上已安装的命令行工具。例如，要从 GitHub 下载，必须安装 Git。

现在您已经构建并安装了`hey`，您可以使用以下命令运行它：

```go
$ hey https://go.dev

Summary:
  Total:           2.1272 secs
  Slowest:         1.4227 secs
  Fastest:         0.0573 secs
  Average:         0.3467 secs
  Requests/sec:    94.0181

```

如果您已经安装了某个工具并希望将其更新到新版本，请使用指定的更高版本或使用 `@latest` 重新运行 `go install`：

```go
$ go install github.com/rakyll/hey@latest

```

当然，您不必将通过 `go install` 安装的程序留在 *go/bin* 目录中；它们是常规的可执行二进制文件，可以存储在计算机上的任何位置。同样，您不必使用 `go install` 分发用 Go 编写的程序；您可以提供一个可下载的二进制文件。但是，`go install` 对于 Go 开发者来说非常方便，已成为分发第三方开发者工具的首选方法。

# 使用 `goimports` 改善导入格式

`go fmt` 的增强版本 `goimports` 还可以清理您的导入语句。它会按字母顺序排列它们，删除未使用的导入，并尝试猜测任何未指定的导入。它的猜测有时不准确，因此您应该自己插入导入。

您可以使用命令 `go install golang.org/x/tools/cmd/goimports@latest` 下载 `goimports`。使用以下命令在您的项目中运行它：

```go
$ goimports -l -w .

```

使用 `-l` 标志告诉 `goimports` 将格式不正确的文件打印到控制台。使用 `-w` 标志告诉 `goimports` 在原地修改文件。`.` 指定要扫描的文件：当前目录及其所有子目录中的所有内容。

###### 注意

`golang.org/x` 下的包是 Go 项目的一部分，但位于主 Go 树之外。尽管有用，它们的开发与 Go 标准库相比遵循更宽松的兼容性要求，可能会引入向后不兼容的更改。一些标准库中的包，例如 第十四章 中介绍的 context 包，最初就是从 `golang.org/x` 开始的。“使用 Go Doc 注释文档化您的代码” 中涵盖的 `pkgsite` 工具也位于此处。您可以在 [“子仓库” 部分](https://oreil.ly/tuROf) 查看其他包。

# 使用代码质量扫描工具

回顾一下`go vet`，你使用了内置工具 `go vet`，它扫描源代码以查找常见的编程错误。许多第三方工具可以检查代码风格并扫描可能被 `go vet` 忽略的潜在错误。这些工具通常称为*代码检查器*。¹ 除了可能的编程错误外，这些工具建议的一些更改包括正确命名变量、格式化错误消息以及在公共方法和类型上放置注释。这些不是错误，因为它们不会阻止程序编译或使程序运行不正确，但它们会标记编写非惯用代码的情况。

当您将代码检查器添加到构建过程中时，请遵循旧的格言“信任，但验证”。由于代码检查器发现的问题类型更加模糊，它们有时会产生误报和漏报。这意味着您不一定需要按照它们的建议进行更改，但您应该认真对待这些建议。Go 开发人员期望代码看起来某种方式并遵循某些规则，如果您的代码不符合这些规则，它就会显得突兀。

如果您认为代码检查器的建议不实用，每个代码检查工具都允许您向源代码添加一个注释来阻止错误结果（注释的格式因检查工具而异，请查阅每个工具的文档了解应写入的内容）。您的注释还应包括为何忽略检查工具发现的原因，以便代码审查人员（以及未来的您，在六个月后回顾源代码时）能够理解您的推理。

## staticcheck

如果您必须选择一个第三方扫描程序，请使用[`staticcheck`](https://oreil.ly/ky8ZD)。它得到了许多活跃于 Go 社区的公司的支持，包括超过 150 种代码质量检查，并且尽可能少产生误报。您可以通过`go install honnef.co/go/tools/cmd/staticcheck@latest`安装它。使用`staticcheck ./...`来检查您的模块。

这是`staticcheck`发现的一个示例，而`go vet`没有找到：

```go
package main

import "fmt"

func main() {
    s := fmt.Sprintf("Hello")
    fmt.Println(s)
}
```

(您还可以在[第十一章存储库](https://oreil.ly/Z_Fpg)中的*sample_code/staticcheck_test*目录找到此代码。)

如果您在此代码上运行`go vet`，它不会发现任何问题。但是，`staticcheck`注意到了一个问题：

```go
$ staticcheck ./...
main.go:6:7: unnecessary use of fmt.Sprintf (S1039)

```

将带括号的代码传递给`staticcheck`时使用`-explain`标志以解释该问题：

```go
$ staticcheck -explain S1039
Unnecessary use of fmt.Sprint

Calling fmt.Sprint with a single string argument is unnecessary
and identical to using the string directly.

Available since
    2020.1

Online documentation
    https://staticcheck.io/docs/checks#S1039

```

`staticcheck`发现的另一个常见问题是对变量的未使用赋值。虽然 Go 编译器要求所有变量至少读取一次，但它不检查对变量赋值的每一个值是否都被读取。在函数内部有多个函数调用时，重复使用`err`变量是常见的做法。如果在其中一个函数调用后忘记写`if err != nil`，编译器将无法帮助您。但是，`staticcheck`可以。这段代码编译没有问题：

```go
func main() {
        err := returnErr(false)
        if err != nil {
                fmt.Println(err)
        }
        err = returnErr(true)
        fmt.Println("end of program")
}
```

(此代码位于[第十一章存储库](https://oreil.ly/Z_Fpg)中的*sample_code/check_err*目录。)

运行`staticcheck`发现了这个错误：

```go
$ staticcheck ./...
main.go:13:2: this value of err is never used (SA4006)
main.go:13:8: returnErr doesn't have side effects and its return value is
ignored (SA4017)

```

第 13 行存在两个相关问题。首先，`returnErr`返回的错误没有被读取。其次，`returnErr`函数的输出（即错误）被忽略了。

## revive

另一个很好的 lint 选项是[`revive`](https://revive.run)。它基于`golint`，这是 Go 团队曾经维护的一个工具。使用命令`go install github.com/mgechev/revive@latest`安装`revive`。默认情况下，它仅启用了`golint`中存在的规则。它可以发现像没有注释的导出标识符、不遵循命名约定的变量或不是最后一个返回值的错误返回值等风格和代码质量问题。

使用配置文件，您可以启用更多规则。例如，要启用检查宇宙块标识符屏蔽的检查，请创建名为*built_in.toml*的文件，并包含以下内容：

```go
[rule.redefines-builtin-id]
```

如果您扫描以下代码：

```go
package main

import "fmt"

func main() {
    true := false
    fmt.Println(true)
}
```

您将收到此警告：

```go
$ revive -config built_in.toml ./...
main.go:6:2: assignment creates a shadow of built-in identifier true

```

（您也可以在[第十一章代码库](https://oreil.ly/Z_Fpg)的*sample_code/revive_test*目录中找到这段代码。）

另外可以启用的规则专注于代码组织的看法，如限制函数中的行数或文件中的公共结构数量。甚至有用于评估函数逻辑复杂性的规则。查看[`revive`文档](https://oreil.ly/WGY9S)及其[支持的规则](https://revive.run/r)。

## golangci-lint

如果您更喜欢使用自助餐方式选择工具，可以考虑使用[`golangci-lint`](https://oreil.ly/p9BH4)。它旨在尽可能高效地配置和运行超过 50 个代码质量工具，包括`go vet`、`staticcheck`和`revive`。

虽然您可以使用`go install`来安装`golangci-lint`，但建议您下载二进制版本。按照[网站](https://oreil.ly/IKa_S)上的安装说明操作。安装完成后，运行`golangci-lint`：

```go
$ golangci-lint run

```

在“未使用的变量”中，您看到了一个将变量设置为从未读取过的值的程序，我提到`go vet`和 Go 编译器无法检测到这些问题。`staticcheck`和`revive`也都无法捕捉到这个问题。然而，`golangci-lint`捆绑的工具之一可以：

```go
$ golangci-lint run
main.go:6:2: ineffectual assignment to x (ineffassign)
    x := 10
    ^
main.go:9:2: ineffectual assignment to x (ineffassign)
    x = 30
    ^

```

您还可以使用`golangci-lint`提供的屏蔽检查功能，这超出了`revive`的能力。通过在运行`golangci-lint`的目录中创建名为*.golangci.yml*的文件，配置`golangci-lint`来检测自己代码中的宇宙块标识符和标识符的屏蔽：

```go
linters:
  enable:
    - govet
    - predeclared

linters-settings:
  govet:
    check-shadowing: true
    settings:
      shadow:
        strict: true
    enable-all: true
```

使用这些设置，在此代码上运行`golangci-lint`

```go
package main

import "fmt"

var b = 20

func main() {
    true := false
    a := 10
    b := 30
    if true {
        a := 20
        fmt.Println(a)
    }
    fmt.Println(a, b)
}
```

检测到以下问题：

```go
$ golangci-lint run
main.go:5:5: var `b` is unused (unused)
var b = 20
    ^
main.go:10:2: shadow: declaration of "b" shadows declaration at line 5 (govet)
    b := 30
    ^
main.go:12:3: shadow: declaration of "a" shadows declaration at line 9 (govet)
        a := 20
        ^
main.go:8:2: variable true has same name as predeclared identifier (predeclared)
    true := false
    ^

```

（您可以在[第十一章代码库](https://oreil.ly/Z_Fpg)的*sample_code/golangci-lint_test*目录中找到`golangci-lint`的代码示例。）

因为`golangci-lint`运行了很多工具（截至目前为止，默认运行了 7 种不同的工具，并允许你启用超过 50 种工具），你的团队可能会对它的一些建议持不同意见。查阅[文档](https://oreil.ly/L_mH4)以了解每个工具的功能。一旦达成对启用哪些检查器达成一致意见，请更新你模块根目录下的*.golangci.yml*文件，并提交到源代码控制。查看文件格式的[文档](https://oreil.ly/vufj1)。

###### 注意

虽然`golangci-lint`允许你在家目录下拥有配置文件，但是如果你与其他开发人员一起工作，不要把它放在那里。除非你喜欢在代码审查中添加数小时的愚蠢争论，你要确保每个人都使用相同的代码质量测试和格式化规则。

我建议你开始将`go vet`作为自动化构建流程的必需部分。接下来添加`staticcheck`，因为它几乎不会产生误报。当你对配置工具和设置代码质量标准感兴趣时，看看`revive`，但要注意它可能存在误报和漏报，所以你不能要求你的团队修复它报告的每个问题。一旦习惯了它们的建议，尝试使用`golangci-lint`并调整其设置，直到适合你的团队。

# 使用 govulncheck 扫描易受攻击的依赖关系

到目前为止，你看到的工具没有强制执行一种代码质量：软件漏洞。拥有丰富的第三方模块生态系统非常棒，但是聪明的黑客会在库中找到安全漏洞并加以利用。开发人员在报告这些漏洞时修补它们，但是如何确保使用受影响版本库的软件更新到修复版本呢？

Go 团队发布了一个名为`govulncheck`的工具来解决这个问题。它扫描你的依赖关系，并查找已知的漏洞，包括标准库和导入到你模块中的第三方库。这些漏洞在 Go 团队维护的[公共数据库](https://oreil.ly/dffxM)中报告。你可以通过以下方式安装它：

```go
$ go install golang.org/x/vuln/cmd/govulncheck@latest

```

让我们看一个小程序，看看漏洞检查器是如何运行的。首先，下载[存储库](https://oreil.ly/TcwW8)。*main.go*中的源代码非常简单。它导入了一个第三方 YAML 库，并使用它将一个小的 YAML 字符串加载到一个结构体中：

```go
func main() {
    info := Info{}

    err := yaml.Unmarshal([]byte(data), &info)
    if err != nil {
        fmt.Printf("error: %v\n", err)
        os.Exit(1)
    }
    fmt.Printf("%+v\n", info)
}
```

*go.mod*文件包含了所需的模块及其版本：

```go
module github.com/learning-go-book-2e/vulnerable

go 1.20

require gopkg.in/yaml.v2 v2.2.7

require gopkg.in/check.v1 v1.0.0-20201130134442-10cb98267c6c // indirect
```

让我们看看在这个项目上运行`govulncheck`时会发生什么：

```go
$ govulncheck ./...
Using go1.21 and govulncheck@v1.0.0 with vulnerability data from
    https://vuln.go.dev (last modified 2023-07-27 20:09:46 +0000 UTC).

Scanning your code and 49 packages across 1 dependent module
    for known vulnerabilities...

Vulnerability #1: GO-2020-0036
    Excessive resource consumption in YAML parsing in gopkg.in/yaml.v2
  More info: https://pkg.go.dev/vuln/GO-2020-0036
  Module: gopkg.in/yaml.v2
    Found in: gopkg.in/yaml.v2@v2.2.7
    Fixed in: gopkg.in/yaml.v2@v2.2.8
    Example traces found:
      #1: main.go:25:23: vulnerable.main calls yaml.Unmarshal

Your code is affected by 1 vulnerability from 1 module.

```

本模块使用了一个旧版且易受攻击的 YAML 包。`govulncheck`贴心地给出了调用有问题代码的确切代码行。

###### 注意

如果 `govulncheck` 知道你的代码使用了某个模块中的漏洞，但无法找到明确调用模块中有问题的部分，那么会收到较轻微的警告。该消息会告知你库的漏洞及解决问题的版本，但还会指出你的模块可能不受影响。

让我们升级到一个修复版本，并查看是否解决了问题：

```go
$ go get -u=patch gopkg.in/yaml.v2
go: downloading gopkg.in/yaml.v2 v2.2.8
go: upgraded gopkg.in/yaml.v2 v2.2.7 => v2.2.8
$ govulncheck ./...
Using go1.21and govulncheck@v1.0.0 with vulnerability data from
    https://vuln.go.dev (last modified 2023-07-27 20:09:46 +0000 UTC).

Scanning your code and 49 packages across 1 dependent module
    for known vulnerabilities...

No vulnerabilities found.

```

记住，应始终力求对项目依赖进行尽可能小的更改，因为这样可以减少依赖变更导致代码出现问题的可能性。因此，请升级到最新的 v2.2.*x* 补丁版本，即 v2.2.8\. 当再次运行 `govulncheck` 时，不会出现已知问题。

当前，`govulncheck` 需要进行 `go install` 才能下载，但很可能最终会添加到标准工具集中。与此同时，确保在构建过程中像常规一样安装并运行它以检查你的项目。你可以在宣布它的[博客文章](https://oreil.ly/uR09p)中了解更多信息。

# 将内容嵌入到您的程序中

许多程序分发时都带有支持文件目录；可能是网页模板或程序启动时加载的标准数据。如果 Go 程序需要支持文件，你可以包含一个文件目录，但这会削弱 Go 的一个优点：将代码编译为单个易于分发的二进制文件。不过，还有另一种选择。你可以通过使用 `go:embed` 注释将文件内容嵌入到你的 Go 二进制文件中。

在 GitHub 的 *sample_code/embed_passwords* 目录中可以找到演示嵌入的程序，该程序检查密码是否是最常用的 10,000 个密码之一。你不会直接将密码列表写入源代码，而是将其嵌入。

*main.go* 中的代码很简单：

```go
package main

import (
    _ "embed"
    "fmt"
    "os"
    "strings"
)

//go:embed passwords.txt
var passwords string

func main() {
    pwds := strings.Split(passwords, "\n")
    if len(os.Args) > 1 {
        for _, v := range pwds {
            if v == os.Args[1] {
                fmt.Println("true")
                os.Exit(0)
            }
        }
        fmt.Println("false")
    }
}
```

要启用嵌入，必须执行两个操作。首先，必须导入 `embed` 包。Go 编译器使用此导入作为指示启用嵌入的标志。因为这段示例代码并未引用 `embed` 包中导出的任何内容，你可以使用空白导入，这在 “尽量避免 init 函数” 中有讨论。`embed` 包中唯一导出的符号是 `FS`，你将在下一个示例中看到它。

接下来，在每个保存文件内容的包级变量前直接放置一个魔法注释。这个注释必须以`go:embed`开头，斜线和`go:embed`之间不能有空格。该注释还必须直接在变量声明的前一行。在这个示例中，你将*passwords.txt*的内容嵌入到名为`passwords`的包级变量中。习惯上，将包含嵌入值的变量视为不可变。正如前面提到的，你只能嵌入到包级变量中。变量必须是`string`、`[]byte`或`embed.FS`类型。如果只有一个文件，使用`string`或`[]byte`是最简单的。

如果需要将一个或多个目录的文件放入程序中，请使用`embed.FS`类型的变量。此类型实现了`io/fs`包中定义的三个接口：`FS`、`ReadDirFS`和`ReadFileFS`。这允许`embed.FS`的实例表示一个虚拟文件系统。下面的程序提供了一个简单的命令行帮助系统。如果没有提供帮助文件，它会列出所有可用的文件。如果指定了一个不存在的文件，则返回错误消息。

```go
package main

import (
    "embed"
    "fmt"
    "io/fs"
    "os"
    "strings"
)

//go:embed help
var helpInfo embed.FS

func main() {
    if len(os.Args) == 1 {
        printHelpFiles()
        os.Exit(0)
    }
    data, err := helpInfo.ReadFile("help/" + os.Args[1])
    if err != nil {
        fmt.Println(err)
        os.Exit(1)
    }
    fmt.Println(string(data))
}
```

你可以在[第十一章代码库](https://oreil.ly/Z_Fpg)的*sample_code/help_system*目录中找到此代码和示例帮助文件。

当构建和运行此程序时，输出如下：

```go
$ go build
$ ./help_system
contents:
advanced/topic1.txt
advanced/topic2.txt
info.txt

$ ./help_system advanced/topic1.txt
This is advanced topic 1.

$ ./help_system advanced/topic3.txt
open help/advanced/topic3.txt: file does not exist

```

你应该注意到一些事情。首先，你不再需要为`embed`使用空白导入，因为你正在使用`embed.FS`。其次，目录名称是嵌入的文件系统的一部分。这个程序的用户不输入“help/”前缀，因此你必须在调用`ReadFile`时预先添加它。

`printHelpFiles`函数展示了如何像处理真实文件系统一样处理嵌入的虚拟文件系统：

```go
func printHelpFiles() {
    fmt.Println("contents:")
    fs.WalkDir(helpInfo, "help",
        func(path string, d fs.DirEntry, err error) error {
            if !d.IsDir() {
                _, fileName, _ := strings.Cut(path, "/")
                fmt.Println(fileName)
            }
            return nil
        })
}
```

使用`io/fs`中的`WalkDir`函数来遍历嵌入式文件系统。`WalkDir`接受一个`fs.FS`实例，一个起始路径和一个函数。这个函数会在文件系统中的每个文件和目录上被调用，从指定的路径开始。如果`fs.DirEntry`不是一个目录，你会打印出它的完整路径名，并通过使用`strings.Cut`来去掉`help/`前缀。

关于文件嵌入还有一些需要了解的事情。尽管所有的示例都是文本文件，你也可以嵌入二进制文件。你还可以通过将它们的名称用空格分隔来将多个文件或目录嵌入到单个`embed.FS`变量中。当嵌入一个文件或目录的名称中含有空格时，请将名称放在引号中。

除了精确的文件和目录名外，你还可以使用通配符和范围来指定你想要嵌入的文件和目录的名称。语法在标准库中的 `path` 包的 [Match 函数文档](https://oreil.ly/BQTEX) 中定义，但遵循常见约定。例如，`*` 匹配 0 或多个字符，`?` 匹配一个字符。

所有的嵌入规范，无论是否使用匹配模式，都会由编译器检查。如果它们无效，则编译失败。以下是模式可能无效的情况：

+   如果指定的名称或模式不匹配文件或目录

+   如果你为 `string` 或 `[]byte` 变量指定了多个文件名或模式

+   如果你为 `string` 或 `[]byte` 变量指定了一个模式，并且它匹配多个文件

# 嵌入隐藏文件

包括以 `.` 或 `_` 开头的目录树中的文件有点复杂。许多操作系统将其视为隐藏文件，因此在指定目录名时默认情况下不包括它们。但是，你可以通过两种方式覆盖此行为。第一种是在你想要嵌入的目录名后加上 `/*`。这将包括根目录中的所有隐藏文件，但不会包括其子目录中的隐藏文件。要包括所有子目录中的所有隐藏文件，请在目录名前加上 `all:`。

此示例程序（你可以在 [第十一章存储库](https://oreil.ly/Z_Fpg) 的 *sample_code/embed_hidden* 目录中找到）使这一点更容易理解。在示例中，目录 *parent_dir* 包含两个文件，*.hidden* 和 *visible*，以及一个子目录 *child_dir*。*child_dir* 子目录包含两个文件，*.hidden* 和 *visible*。

这是程序的代码：

```go
//go:embed parent_dir
var noHidden embed.FS

//go:embed parent_dir/*
var parentHiddenOnly embed.FS

//go:embed all:parent_dir
var allHidden embed.FS

func main() {
    checkForHidden("noHidden", noHidden)
    checkForHidden("parentHiddenOnly", parentHiddenOnly)
    checkForHidden("allHidden", allHidden)
}

func checkForHidden(name string, dir embed.FS) {
    fmt.Println(name)
    allFileNames := []string{
        "parent_dir/.hidden",
        "parent_dir/child_dir/.hidden",
    }
    for _, v := range allFileNames {
        _, err := dir.Open(v)
        if err == nil {
            fmt.Println(v, "found")
        }
    }
    fmt.Println()
}
```

程序的输出显示如下：

```go
noHidden

parentHiddenOnly
parent_dir/.hidden found

allHidden
parent_dir/.hidden found
parent_dir/child_dir/.hidden found
```

# 使用 `go generate`

`go generate` 工具有些不同，因为它本身不执行任何操作。当你运行 `go generate` 时，它会查找源代码中特殊格式的注释，并运行这些注释中指定的程序。虽然你可以使用 `go generate` 运行任何内容，但它最常用于开发者运行（顾名思义）生成源代码的工具。这可能是通过分析现有代码并添加功能，或者处理模式并生成源代码。

自动转换为代码的一个很好的例子是 [Protocol Buffers](https://protobuf.dev)，有时称为 *protobufs*。Protobuf 是一种流行的二进制格式，被 Google 用于存储和传输数据。在处理 protobuf 时，您编写一个 *schema*，它是数据结构的语言无关描述。希望编写与 protobuf 格式数据交互的程序的开发人员运行处理 schema 的工具，并生成特定于语言的数据结构以保存数据以及特定于语言的函数来读取和写入 protobuf 格式的数据。

让我们看看这在 Go 中是如何工作的。您可以在 [proto_generate 仓库](https://oreil.ly/OJdYU) 中找到一个示例模块，其中包含一个名为 *person.proto* 的 protobuf schema 文件：

```go
syntax = "proto3";

message Person {
  string name = 1;
  int32 id = 2;
  string email = 3;
}
```

尽管制作一个实现 `Person` 的结构体很容易，但编写从二进制格式转换回去的代码却很困难。让我们使用 Google 的工具来完成这些艰难的工作，并使用 `go generate` 调用它们。您需要安装两个工具。首先是适用于您计算机的 `protoc` 二进制文件（请参阅 [安装说明](https://oreil.ly/UIvZN)）。接下来，使用 `go install` 安装 Go protobuf 插件：

```go
$ go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.28

```

在 *main.go* 中，有一个由 `go generate` 处理的神奇注释：

```go
//go:generate protoc -I=. --go_out=.
  --go_opt=module=github.com/learning-go-book-2e/proto_generate
  --go_opt=Mperson.proto=github.com/learning-go-book-2e/proto_generate/data
  person.proto
```

（如果您在 GitHub 上查看源代码，您会看到这应该是一行。它被换行以适应打印页面的约束。）

输入以下内容运行 `go generate`：

```go
$ go generate ./...

```

运行 `go generate` 后，您会看到一个名为 *data* 的新目录，其中包含一个名为 *person.pb.go* 的文件。它包含 `Person` 结构体的源代码，以及在 `google.golang.org/protobuf/proto` 模块中由 `Marshal` 和 `Unmarshal` 函数使用的一些方法和函数。您在 `main` 函数中调用这些函数：

```go
func main() {
    p := &data.Person{
        Name:  "Bob Bobson",
        Id:    20,
        Email: "bob@bobson.com",
    }
    fmt.Println(p)
    protoBytes, _ := proto.Marshal(p)
    fmt.Println(protoBytes)
    var p2 data.Person
    proto.Unmarshal(protoBytes, &p2)
    fmt.Println(&p2)
}
```

像往常一样构建和运行程序：

```go
$ go build
$ ./proto_generate
name:"Bob Bobson"  id:20  email:"bob@bobson.com"
[10 10 66 111 98 32 66 111 98 115 111 110 16 20 26 14 98
  111 98 64 98 111 98 115 111 110 46 99 111 109]
name:"Bob Bobson"  id:20  email:"bob@bobson.com"
```

另一个常与 `go generate` 一起使用的工具是 `stringer`。正如我在 “iota 用于枚举——有时” 中讨论的那样，Go 中的枚举缺少许多其他语言中的枚举所具有的特性之一是为枚举中的每个值自动生成可打印的名称。 `stringer` 工具与 `go generate` 一起使用，为您枚举的值添加一个 `String` 方法，以便可以打印它们。

使用 `go install golang.org/x/tools/cmd/stringer@latest` 安装 `stringer`。在 [第十一章仓库](https://oreil.ly/Z_Fpg) 的 *sample_code/stringer_demo* 目录中提供了一个非常简单的示例，展示了如何使用 `stringer`。以下是 *main.go* 中的源代码：

```go
type Direction int

const (
    _ Direction = iota
    North
    South
    East
    West
)

//go:generate stringer -type=Direction

func main() {
    fmt.Println(North.String())
}
```

运行 `go generate ./...`，您将看到生成一个名为 *direction_string.go* 的新文件。使用 `go build` 构建 `string_demo` 二进制文件，并在运行时，您将获得输出：

```go
North
```

您可以以多种方式配置 `stringer` 及其输出。Arjun Mahishi 写了一篇很棒的 [博客文章](https://oreil.ly/2YVE2) 描述了如何使用 `stringer` 并自定义其输出。

# 使用 `go generate` 和 Makefile 进行工作

由于 `go generate` 的工作是运行其他工具，你可能会想知道在项目中已经有一个完全正常的 Makefile 时是否值得使用它。`go generate` 的优势在于它创建了职责的分离。使用 `go generate` 命令来机械地创建源代码，并使用 Makefile 来验证和编译源代码。

将由 `go generate` 创建的源代码提交到版本控制是一种最佳实践。（在 [第十一章的存储库](https://oreil.ly/Z_Fpg) 的示例项目中不包括生成的源代码，因此你可以看到 `go generate` 的工作。）这使得浏览你源代码的人可以看到每个被调用的东西，甚至是生成的部分。它也意味着他们不需要安装诸如 `protoc` 这样的工具来构建你的代码。

技术上来说，提交你生成的源代码意味着你不一定*需要*运行 `go generate`，除非它会产生不同的输出，例如处理修改的 protobuf 定义或更新的枚举。然而，自动化在 `go build` 之前调用 `go generate` 仍然是一个好主意。依赖手动流程会带来麻烦。一些生成器工具，如 `stringer`，包含聪明的技巧来阻止编译，如果你忘记重新运行 `go generate`，但这并非普遍。在测试中试图理解为什么变更没有显示之前，你会浪费时间，最终发现你忘记调用 `go generate`。 （在学习教训之前，我犯了这个错误多次。）因此，最好是在你的 Makefile 中添加一个 `generate` 步骤，并将其作为你的 `build` 步骤的依赖项。

然而，在两种情况下我会忽略这些建议。首先是当在相同输入上调用 `go generate` 产生具有轻微差异的源文件时，例如时间戳。一个良好编写的 `go generate` 工具每次在相同输入上运行时应该产生相同的输出，但不能保证你需要使用的每个工具都是良好编写的。你不希望不断地检查功能上相同的新版本文件，因为它们会混乱你的版本控制系统并使得代码审查更加嘈杂。

如果 `go generate` 花费很长时间才能完成，这是第二种情况。快速构建是 Go 的一个特性，因为它们允许开发者保持专注并获得快速反馈。如果因为生成相同文件而显著减慢构建速度，那么这种开发者生产力的损失是不值得的。在这两种情况下，你只能留下大量注释来提醒人们在变更时重新构建，并希望你团队的每个人都很勤奋。

# 在 Go 二进制文件内读取构建信息

随着公司开发越来越多的自己的软件，他们越来越希望确切地了解他们已部署到他们的数据中心和云环境中的内容，包括版本和依赖关系。您可能会想知道为什么要从编译代码中获取这些信息。毕竟，公司已经在版本控制中拥有这些信息。

具有成熟的开发和部署流水线的公司可以在部署程序之前捕获此信息，以确保信息的准确性。然而，许多公司，如果不是大多数公司，不跟踪已部署的内部软件的确切版本。在某些情况下，软件可以部署多年而无需更换，并且没有人记得太多关于它。如果在流行的 Log4j 库中发现了严重的漏洞报告，需要找到某种方法扫描已部署的软件并确定已部署的第三方库的版本，或者为了安全起见重新部署所有内容。在 Java 的世界中，这个确切的问题就发生了。

幸运的是，Go 为您解决了这个问题。使用 `go build` 创建的每个 Go 二进制文件都自动包含了构建信息，包括构成该二进制文件的模块的版本以及使用的构建命令、版本控制系统以及代码在版本控制系统中的修订版本。您可以使用 `go version -m` 命令查看此信息。以下是在 Apple Silicon Mac 上构建 `vulnerable` 程序的输出：

```go
$ go build
go: downloading gopkg.in/yaml.v2 v2.2.7
$ go version -m vulnerable
vulnerable: go1.20
    path     github.com/learning-go-book-2e/vulnerable
    mod      github.com/learning-go-book-2e/vulnerable    (devel)
    dep      gopkg.in/yaml.v2  v2.2.7  h1:VUgggvou5XRW9mHwD/yXxIYSMtY0zoKQf/v...
    build    -compiler=gc
    build    CGO_ENABLED=1
    build    CGO_CFLAGS=
    build    CGO_CPPFLAGS=
    build    CGO_CXXFLAGS=
    build    CGO_LDFLAGS=
    build    GOARCH=arm64
    build    GOOS=darwin
    build    vcs=git
    build    vcs.revision=623a65b94fd02ea6f18df53afaaea3510cd1e611
    build    vcs.time=2022-10-02T03:31:05Z
    build    vcs.modified=false

```

由于这些信息嵌入到每个二进制文件中，`govulncheck`能够扫描 Go 程序，检查是否存在已知漏洞的库：

```go
$ govulncheck -mode binary vulnerable
Using govulncheck@v1.0.0 with vulnerability data from
    https://vuln.go.dev (last modified 2023-07-27 20:09:46 +0000 UTC).

Scanning your binary for known vulnerabilities...

Vulnerability #1: GO-2020-0036
    Excessive resource consumption in YAML parsing in gopkg.in/yaml.v2
  More info: https://pkg.go.dev/vuln/GO-2020-0036
  Module: gopkg.in/yaml.v2
    Found in: gopkg.in/yaml.v2@v2.2.7
    Fixed in: gopkg.in/yaml.v2@v2.2.8
    Example traces found:
      #1: yaml.Unmarshal

Your code is affected by 1 vulnerability from 1 module.

```

请注意，`govulncheck` 在检查二进制文件时无法追踪到确切的代码行。如果 `govulncheck` 在二进制文件中发现问题，请使用 `go version -m` 查找确切的部署版本，从版本控制中检出代码，并再次针对源代码运行以准确定位问题。

如果您想构建自己的工具来读取构建信息，请查看标准库中的 [`debug/buildinfo`](https://oreil.ly/M5Jmq) 包。

# 为其他平台构建 Go 二进制文件

像 Java、JavaScript 或 Python 这样基于虚拟机的语言的一个优点是，您可以将您的代码放在安装了虚拟机的任何计算机上运行。这种可移植性使得使用这些语言的开发人员能够在 Windows 或 Mac 计算机上构建程序，并在 Linux 服务器上部署，即使操作系统和可能的 CPU 架构不同。

Go 程序被编译成本地代码，因此生成的二进制文件只能与单个操作系统和 CPU 架构兼容。但是，这并不意味着 Go 开发人员需要维护多台机器（虚拟或其他）以在多个平台上发布。`go build` 命令使得跨平台编译变得简单，或者为不同的操作系统和/或 CPU 创建二进制文件。运行 `go build` 时，目标操作系统由 `GOOS` 环境变量指定。类似地，`GOARCH` 环境变量指定 CPU 架构。如果没有显式设置它们，`go build` 默认使用您当前计算机的值，这就是为什么您以前从未担心过这些变量的原因。

您可以在 [安装文档](https://oreil.ly/Zf1lx) 中找到 `GOOS` 和 `GOARCH` 的有效值和组合（有时发音为“GOOSE”和“GORCH”）。一些支持的操作系统和 CPU 有点奇特，而其他可能需要一些翻译。例如，`darwin` 指的是 macOS（Darwin 是 macOS 内核的名称），而 `amd64` 表示 64 位兼容 Intel 的 CPU。

让我们最后再回到 `vulnerable` 程序。当使用 Apple Silicon Mac（具有 ARM64 CPU）时，运行 `go build` 默认为 `darwin` 的 `GOOS` 和 `arm64` 的 `GOARCH`。您可以使用 `file` 命令确认这一点：

```go
$ go build
$ file vulnerable
vulnerable: Mach-O 64-bit executable arm64

```

这里是如何在 64 位 Intel CPU 的 Linux 上构建二进制文件：

```go
$ GOOS=linux GOARCH=amd64 go build
$ file vulnerable
vulnerable: ELF 64-bit LSB executable, x86-64, version 1 (SYSV),
    statically linked, Go BuildID=IDHVCE8XQPpWluGpMXpX/4VU3GpRZEifN
    8TzUrT_6/1c30VcDYNVPfSSN-zCkz/JsZSLAbWkxqIVhPkC5p5, with debug_info,
    not stripped

```

# 使用构建标签

当编写需要在多个操作系统或 CPU 架构上运行的程序时，有时需要为不同平台编写不同的代码。您可能还希望编写一个模块，利用最新的 Go 特性，但仍与旧版 Go 编译器兼容。

您可以通过两种方式创建目标代码。第一种方法是使用文件名来指示何时应将文件包含在构建中。您可以在 *.go* 之前的文件名中添加目标 *GOOS* 和 *GOARCH*，用 `_` 分隔。例如，如果您有一个只希望在 Windows 上编译的文件，您将文件命名为 *something_windows.go*，但如果您希望在构建 ARM64 Windows 时编译它，则将文件命名为 *something_windows_arm64.go*。

*构建标签*（也称为*构建约束*）是您可以使用的另一种指定文件何时编译的选项。与嵌入和生成类似，构建标签利用了一个魔术注释。在这种情况下，它是 `//go:build`。此注释必须放置在文件中包声明之前的行上。

构建标签使用布尔运算符（`||`、`&&` 和 `!`）和括号来指定针对架构、操作系统和 Go 版本的确切构建规则。构建标签 `//go:build (!darwin && !linux) || (darwin && !go1.12)` —— 这实际上出现在 Go 标准库中 —— 指定该文件不应在 Linux 或 macOS 上编译，除非在 macOS 上的 Go 版本是 1.11 或更早版本。

还可以使用一些元构建约束。约束`unix`匹配任何类 Unix 平台，而`cgo`则在当前平台支持且启用了`cgo`时匹配（我在“Cgo 用于集成而非性能”中讨论过`cgo`）。

问题在于你应该何时使用文件名来指示在哪里运行代码，何时应该使用构建标签。因为构建标签允许使用二进制运算符，你可以用它们来指定更具体的平台集。Go 标准库有时采用一种“双重保险”的方法。标准库中的`internal/cpu`包针对 CPU 特征检测具有特定于平台的源代码。文件*internal/cpu/cpu_arm64_darwin.go*的名称表明它仅适用于使用苹果 CPU 的计算机。该文件还在其内部使用了`//go:build arm64 && darwin && !ios`行，以指示它应仅在构建苹果 Silicon Macs 时编译，而不适用于 iPhone 或 iPad。构建标签能够更详细地指定目标平台，但遵循文件名约定使人们能够轻松找到适合特定平台的正确文件。

除了表示 Go 版本、操作系统和 CPU 架构的内置构建标签之外，您还可以使用任何字符串作为自定义构建标签。然后，您可以使用`-tags`命令行标志来控制该文件的编译。例如，如果在包声明前的行上放置了`//go:build gopher`，则除非在`go build`、`go run`或`go test`命令中包含`-tags gopher`标志，否则它将不会被编译。

自定义构建标签非常方便。如果有一个源文件暂时不想构建（可能是因为它尚未编译或者是一个尚未准备好包含的实验），在包声明前的行上放置`//go:build ignore`是一种惯用方法。在查看“使用集成测试和构建标签”中的集成测试时，您将看到自定义构建标签的另一个用途。

###### 警告

写你的构建标签时，请确保`//`和`go:build`之间没有任何空格。如果有空格，Go 将不会将其视为构建标签。

# 测试 Go 版本

尽管 Go 提供了强大的向后兼容性保证，但仍会出现 bug。自然而然地希望确保新版本不会破坏您的程序。您还可能会收到用户反馈，称您的库在旧版本的 Go 上运行时出现了意外行为。一个选项是安装第二个 Go 环境。例如，如果您想尝试版本 1.19.2，可以使用以下命令：

```go
$ go install golang.org/dl/go1.19.2@latest
$ go1.19.2 download

```

然后可以使用命令`go1.19.2`而不是`go`命令来查看版本 1.19.2 是否适用于您的程序：

```go
$ go1.19.2 build

```

一旦验证了你的代码可以工作，你可以卸载辅助环境。Go 将辅助 Go 环境存储在你的主目录的*sdk*目录中。要卸载，请从*sdk*目录中删除该环境，并从*go/bin*目录中删除二进制文件。以下是如何在 macOS、Linux 和 BSD 上执行此操作的步骤：

```go
$ rm -rf ~/sdk/go.19.2
$ rm ~/go/bin/go1.19.2

```

# 使用 go help 了解更多关于 Go 工具的信息

通过`go help`命令可以了解更多关于 Go 工具和运行时环境的信息。它包含了关于这里提到的所有命令的详尽信息，以及诸如模块、导入路径语法和处理非公开源码的内容。例如，你可以通过输入`go help importpath`获取有关导入路径语法的信息。

# 练习

这些练习涵盖了本章介绍的一些工具。你可以在[第十一章存储库](https://oreil.ly/Z_Fpg)的*exercise_solutions*目录中找到答案。

1.  前往[联合国《世界人权宣言》页面](https://oreil.ly/-q7Cn)，并将《世界人权宣言》的文本复制到名为*english_rights.txt*的文本文件中。点击“其他语言”链接，并将该文件的文本用几种其他语言分别复制到名为*LANGUAGE_rights.txt*的文件中。创建一个程序，将这些文件嵌入包级别变量中。你的程序应接受一个命令行参数，即语言名称。然后，它应该打印出该语言的《世界人权宣言》。

1.  使用`go install`安装`staticcheck`。运行它来检查你的程序并修复它找到的任何问题。

1.  在 Windows 上为 ARM64 交叉编译你的程序。如果你使用的是 ARM64 Windows 计算机，则在 Linux 上为 AMD64 进行交叉编译。

# 总结

在本章中，你学习了 Go 提供的工具，以改进软件工程实践和第三方代码质量工具。在下一章中，你将探索 Go 中的一个标志性特性：并发。

¹“linter”这个术语源于贝尔实验室 Unix 团队成员史蒂夫·约翰逊编写的原始`lint`程序，并在他的[1978 年论文](https://oreil.ly/RgZbU)中描述。这个名字来自于干衣机中衣物脱落的微小纤维被滤网捕捉的现象。他将他的程序比作捕捉小错误的滤网。
