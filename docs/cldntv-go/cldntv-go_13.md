# 第十章：可管理性

> 每个人都知道调试比一开始编写程序要难两倍。所以，如果您在编写代码时尽可能聪明，您将如何调试它呢？¹
> 
> Brian Kernighan，《程序设计风格的要素》（1978 年）

在完美的世界里，您永远不需要部署新版本的服务，或者（天哪！）关闭整个系统来修复或修改以满足新的需求。

不过，在完美的世界里，独角兽会存在，五位牙医中会有四位建议我们早餐吃派。²

显然，我们不生活在一个完美的世界。但是尽管独角兽可能永远不存在，³ 您不必接受一个每次需要修改系统行为时都需要更新代码的世界。

虽然您可能总是需要进行代码更改来更新核心逻辑，但是有可能构建您的系统，以便您——或者关键时刻，其他人——可以在无需重新编码和重新部署的情况下改变多种行为。

您可能还记得我们在《“可管理性”》（ch01.xhtml#section_ch01_manageability）中介绍了这个云原生系统的重要属性，我们定义它为系统行为可以轻松修改以保持安全运行顺畅，并与变化的需求符合。

尽管听起来很简单，但可管理性实际上比您想象的要复杂得多。它远不止配置文件（尽管这当然是其中的一部分）。在本章中，我们将讨论什么是可管理的系统，以及一些技术和实现，使您能够构建一个几乎可以像其需求一样快速改变的系统。

# 什么是可管理性以及我为什么要关心它？

在考虑可管理性时，通常会从单个服务的角度思考。我的服务是否可以轻松配置？它是否拥有可能需要的所有旋钮和按钮？

然而，通过专注于组件而忽视系统，这忽略了更大的观点。可管理性不仅限于服务边界。要使系统可管理，必须考虑整个系统。

请花些时间考虑在复杂系统中重新考虑可管理性。它的行为是否可以轻松修改？它的组件是否可以相互独立地修改？如果必要，它们是否可以轻松替换？我们如何知道何时需要这样做？

可管理性包括系统行为的所有可能维度。可以说其功能可分为四个广泛的类别：⁵

配置和控制

设置和配置系统及其各个组件应该易于配置，以实现最佳的可用性和性能是非常重要的。一些系统需要定期或实时控制，因此拥有正确的“旋钮和杠杆”是绝对基础的。这将是本章大部分关注的焦点。

监控、日志和警报

这些功能跟踪系统执行其工作的能力，对于有效的系统管理至关重要。毕竟，没有它们，我们怎么知道我们的系统何时需要管理呢？尽管这些功能对可管理性至关重要，但我们不会在本章讨论它们。相反，它们将在第十一章，*可观察性*中专门讨论。

部署和更新

即使在没有代码更改的情况下，轻松部署、更新、回滚和扩展系统组件的能力也是非常有价值的，尤其是当需要管理许多系统时。显然，在初始部署期间这非常有用，但是在系统的整个生命周期中进行更新也同样重要。幸运的是，Go 语言在这方面表现出色，其缺乏外部运行时和单一的可执行构件使其非常优秀。

服务发现和库存

云原生系统的一个关键特性是其分布式性质。组件能够快速准确地检测彼此非常关键，这被称为*服务发现*功能。由于服务发现是一种架构特性而不是程序特性，我们在本书中不会深入探讨它。

因为这本书更像是一本关于 Go 语言而不是架构的书籍⁶，它主要集中于服务实现。正因如此——*并非因为它更重要*——本章的大部分内容将同样聚焦在服务级配置上。不幸的是，对这些内容的深入讨论超出了本书的范围⁷。

管理复杂的计算系统通常是困难且耗时的，其管理成本可能远远超过底层硬件和软件的成本。按定义，一个设计为可管理的系统可以更高效地进行管理，因此成本更低。即使不考虑管理成本，减少复杂性也可以极大地影响人为错误的发生概率，使其更容易和更快速地纠正。因此，可管理性直接影响可靠性、可用性和安全性，是系统可靠性的关键因素。

# 配置您的应用程序

可管理性的最基本功能是配置应用程序的能力。在一个理想的可配置应用程序中，任何在不同环境（如测试、生产、开发环境等）之间可能变化的内容都应该与代码清晰分离，并以某种外部可定义的方式定义。

您可能还记得 *The Twelve-Factor App*——我们在 第六章 中介绍的用于构建 Web 应用程序的十二条规则和指南——对这个主题有很多看法。事实上，它的十二条规则中的第三条——“III. 配置”——完全关注应用程序配置，关于这一点，它说：

> 在环境中存储配置。

正如 *The Twelve-Factor App* 所述，所有配置应存储在环境变量中。对此有很多不同看法，但自从其出版以来，行业似乎已达成普遍共识，即真正重要的是：

配置应严格与代码分离。

配置——任何可能在不同环境中有所不同的内容——应始终与代码清晰分离。虽然配置在不同部署中可能有很大差异，但代码则不然。配置不应该被硬编码到代码中。永远不要这样做。

配置应存储在版本控制中。

将配置存储在版本控制中——与代码分开——允许您在必要时快速回滚配置更改，并有助于系统的重建和恢复。一些部署框架，如 Kubernetes，通过提供像 ConfigMap 这样的配置原语，使这种区分自然且相对无缝。

如今，看到应用程序主要通过环境变量进行配置仍然很常见，但看到带有各种格式的命令行标志和配置文件也同样普遍。有时，一个应用程序甚至会支持多种这些选项。在接下来的章节中，我们将审查其中一些方法，它们的各种优缺点，以及如何在 Go 中实现它们。

## 配置良好实践

在构建应用程序时，您有很多选择来定义、实施和部署应用程序配置。然而，根据我的经验，我发现某些通用实践能够产生更好的长期和短期结果：

版本控制您的配置。

是的，我在重复，但这是值得重复的。在部署到系统之前，配置文件应存储在版本控制中。这使得在部署之前能够审查它们，在之后能够快速引用它们，并在必要时能够快速回滚更改。如果（以及当）需要重新创建和恢复系统时，这也是有帮助的。

不要自己发明格式。

使用 JSON、YAML 或 TOML 等标准格式编写配置文件。我们将在本章后面介绍其中一些。如果 *必须* 自己发明格式，请确保您对维护它的想法感到满意，并且迫使任何未来的维护人员永远处理它。

让零值有用。

不要无谓地使用非零默认值。总的来说，这是一个很好的规则；甚至有一个关于它的“Go 谚语”。⁸ 在可能的情况下，未定义配置所导致的行为应该是可接受的、合理的和不令人惊讶的。简单、最小化的配置可以减少错误的可能性。

## 使用环境变量进行配置

正如我们在 第六章 中讨论过，并且之前审查过的，使用环境变量定义配置值是《十二要素应用》中提倡的方法。这种偏好确实有其合理性：环境变量得到广泛支持，它们确保配置不会意外地被检入代码中，并且使用它们通常需要的代码比使用配置文件少。对于小型应用程序来说，它们也完全足够了。

另一方面，设置和传递环境变量的过程可能会显得丑陋、乏味和冗长。虽然一些应用程序支持在文件中定义环境变量，但这在很大程度上削弱了首次使用环境变量的初衷。

环境变量的隐式特性也可能带来一些挑战。由于无法通过查看现有配置文件或检查帮助输出轻松了解环境变量的存在和行为，依赖于它们的应用有时可能更难使用，并且其中的错误更难调试。

与大多数高级语言一样，Go 使得环境变量很容易访问。它通过标准的`os`包实现这一点，该包提供了`os.Getenv`函数用于此目的：

```go
name := os.Getenv("NAME")
place := os.Getenv("CITY")

fmt.Printf("%s lives in %s.\n", name, place)
```

`os.Getenv`函数获取由键命名的环境变量的值，但如果变量不存在，则返回一个空字符串。如果需要区分空值和未设置值，Go 还提供了`os.LookEnv`函数，它返回值和一个布尔值，如果变量未设置则为`false`：

```go
if val, ok := os.LookupEnv(key); ok {
    fmt.Printf("%s=%s\n", key, val)
} else {
    fmt.Printf("%s not set\n", key)
}
```

这个功能非常基本，但对于许多（如果不是大多数）用途来说完全足够了。如果你需要更复杂的选项，如默认值或类型化变量，有几个出色的第三方包可以提供这些功能。 [Viper（spf13/viper）](https://oreil.ly/bS4RY)——我们将在 “Viper: The Swiss Army Knife of Configuration Packages” 中讨论它——特别受欢迎。

## 使用命令行参数进行配置

作为配置方法，命令行参数绝对值得考虑，至少对于较小、不太复杂的应用程序来说是如此。毕竟，它们是显式的，并且它们的存在和用法细节通常可以通过`--help`选项获得。

### 标准标志包

Go 语言包含`flag`包，这是其标准库中的基本命令行解析包。虽然`flag`包功能不是特别丰富，但使用起来相当简单，并且——不像`os.Getenv`——支持开箱即用的类型。

举个例子，下面的程序使用`flag`实现了一个基本命令，读取并输出命令行标志的值：

```go
package main

import (
    "flag"
    "fmt"
)

func main() {
    // Declare a string flag with a default value "foo"
    // and a short description. It returns a string pointer.
    strp := flag.String("string", "foo", "a string")

    // Declare number and Boolean flags, similar to the string flag.
    intp := flag.Int("number", 42, "an integer")
    boolp := flag.Bool("boolean", false, "a boolean")

    // Call flag.Parse() to execute command-line parsing.
    flag.Parse()

    // Print the parsed options and trailing positional arguments.
    fmt.Println("string:", *strp)
    fmt.Println("integer:", *intp)
    fmt.Println("boolean:", *boolp)
    fmt.Println("args:", flag.Args())
}
```

正如你从前面的代码中看到的，`flag`包允许你注册带有类型、默认值和简短描述的命令行标志，并将这些标志映射到变量。我们可以通过运行程序并传递`-help`标志来查看这些标志的摘要：

```go
$ go run . -help
Usage of /var/folders/go-build618108403/exe/main:
  -boolean
        a boolean
  -number int
        an integer (default 42)
  -string string
        a string (default "foo")
```

帮助输出向我们展示了所有可用标志的列表。运行所有这些标志，结果如下：

```go
$ go run . -boolean -number 27 -string "A string." Other things.
string: A string.
integer: 27
boolean: true
args: [Other things.]
```

它有效果！但是，`flag`包似乎有一些限制其实用性的问题。

首先，你可能已经注意到，生成的标志语法似乎有点不太标准。我们许多人已经习惯了命令行界面遵循[GNU 参数标准](https://oreil.ly/evqk4)，长名称选项由两个破折号（`--version`）前缀，而短的单字母等效项（`-v`）。

其次，`flag`的功能仅限于解析标志（虽然公平地说，它并没有声称做得更多），虽然这很好，但它的功能远不及它可能达到的水平。如果我们能够将命令映射到函数，那就太好了，不是吗？

### Cobra 命令行解析器

如果你只需解析标志，`flags`包完全可以胜任，但如果你需要更强大的东西来构建命令行界面，你可能会考虑[Cobra 包](https://oreil.ly/4oCyH)。Cobra 有许多功能，使它成为构建功能齐全的命令行界面的热门选择。它被用于许多知名项目，包括 Kubernetes、CockroachDB、Docker、Istio 和 Helm。

除了提供完全符合 POSIX 标准的标志（长*和*短版本），Cobra 还支持嵌套子命令，并自动生成各种 Shell 的帮助（`--help`）输出和自动完成。它还与 Viper 集成，我们将在“Viper：配置包的瑞士军刀”中介绍它。

你可能会想到，Cobra 的主要缺点是相对于`flags`包来说相当复杂。使用 Cobra 来实现“标准标志包”中的程序看起来像这样：

```go
package main

import (
    "fmt"
    "os"
    "github.com/spf13/cobra"
)

var strp string
var intp int
var boolp bool

var rootCmd = &cobra.Command{
    Use:  "flags",
    Long: "A simple flags experimentation command, built with Cobra.",
    Run:  flagsFunc,
}

func init() {
    rootCmd.Flags().StringVarP(&strp, "string", "s", "foo", "a string")
    rootCmd.Flags().IntVarP(&intp, "number", "n", 42, "an integer")
    rootCmd.Flags().BoolVarP(&boolp, "boolean", "b", false, "a boolean")
}

func flagsFunc(cmd *cobra.Command, args []string) {
    fmt.Println("string:", strp)
    fmt.Println("integer:", intp)
    fmt.Println("boolean:", boolp)
    fmt.Println("args:", args)
}

func main() {
    if err := rootCmd.Execute(); err != nil {
        fmt.Println(err)
        os.Exit(1)
    }
}
```

与`flags`包版本相比，后者基本上只是读取一些标志并打印结果，Cobra 程序具有更复杂的结构，包含多个不同的部分。

首先，我们使用包范围声明目标变量，而不是在函数内部局部声明。这是必需的，因为它们必须在`init`函数和实现命令逻辑的函数之间可访问。

接下来，我们创建一个`cobra.Command`结构体，`rootCmd`，代表*根命令*。一个单独的`cobra.Command`实例用于表示 CLI 提供的每个命令和子命令。`Use`字段说明命令的一行使用消息，`Long`是显示在帮助输出中的长消息。`Run`是一个类型为`func(cmd *Command, args []string)`的函数，用于实现命令执行时要执行的实际工作。

通常，命令是在`init`函数中构建的。在我们的情况下，我们将三个标志—`string`，`number`和`boolean`—添加到我们的根命令中，以及它们的短标志、默认值和描述。

每个命令都有一个自动生成的帮助输出，我们可以使用`--help`标志来获取它：

```go
$ go run . --help
A simple flags experimentation command, built with Cobra.

Usage:
  flags [flags]

Flags:
  -b, --boolean         a boolean
  -h, --help            help for flags
  -n, --number int      an integer (default 42)
  -s, --string string   a string (default "foo")
```

这是有道理的，而且也很漂亮！但它是否按我们的预期运行？执行命令（使用标准标志样式），会给我们带来以下输出：

```go
$ go run . --boolean --number 27 --string "A string." Other things.
string: A string.
integer: 27
boolean: true
args: [Other things.]
```

输出是相同的；我们已经实现了一致性。但这只是一个单一的命令。Cobra 的一个好处是它还允许*子命令*。

这意味着什么？以`git`命令为例。在这个例子中，`git`将是根命令。它本身不做太多事情，但它有一系列子命令—`git clone`，`git init`，`git blame`等—它们相关，但每个都是独立的操作。

Cobra 通过将命令视为树结构来提供此功能。每个命令和子命令（包括根命令）都由一个独立的`cobra.Command`值表示。它们使用`(c *Command) AddCommand(cmds ...*Command)`函数彼此连接。我们在以下示例中演示了这一点，通过将`flags`命令转换为新根命令的子命令，我们称之为`cng`（用于*Cloud Native Go*）。

要做到这一点，我们首先必须将原始的`rootCmd`重命名为`flagsCmd`。我们添加了一个`Short`属性来定义其在帮助输出中的简短描述，但它在其他方面是相同的。但现在我们需要一个新的根命令，因此我们也创建了它：

```go
var flagsCmd = &cobra.Command{
    Use:   "flags",
    Short: "Experiment with flags",
    Long:  "A simple flags experimentation command, built with Cobra.",
    Run:   flagsFunc,
}

var rootCmd = &cobra.Command{
    Use:  "cng",
    Long: "A super simple command.",
}
```

现在我们有两个命令：根命令`cng`和一个单一的子命令`flags`。下一步是将`flags`子命令添加到根命令中，使其直接位于命令树的根下。这通常在`init`函数中完成，我们在这里演示：

```go
func init() {
    flagsCmd.Flags().StringVarP(&strp, "string", "s", "foo", "a string")
    flagsCmd.Flags().IntVarP(&intp, "number", "n", 42, "an integer")
    flagsCmd.Flags().BoolVarP(&boolp, "boolean", "b", false, "a boolean")

    rootCmd.AddCommand(flagsCmd)
}
```

在前述的`init`函数中，我们保留了三个`Flags`方法，只是现在我们在`flagsCmd`上调用它们。

然而，新的是`AddCommand`方法，它允许我们将`flagsCmd`作为子命令添加到`rootCmd`中。我们可以多次重复`AddCommand`，使用多个`Command`值添加尽可能多的子命令（或子子命令，或子子子命令）。

现在我们已经告诉 Cobra 关于新的`flags`子命令，其信息反映在生成的帮助输出中：

```go
$ go run . --help
A super simple command.

Usage:
  cng [command]

Available Commands:
  flags       Experiment with flags
  help        Help about any command

Flags:
  -h, --help   help for cng

Use "cng [command] --help" for more information about a command.
```

现在，根据此帮助输出，我们有一个顶级根命令名为`cng`，有*两个*可用的子命令：我们的`flags`命令，以及一个自动生成的`help`子命令，允许用户查看任何子命令的帮助。例如，`help flags`为我们提供了有关`flags`子命令的信息和说明：

```go
$ go run . help flags
A simple flags experimentation command, built with Cobra.

Usage:
  cng flags [flags]

Flags:
  -b, --boolean         a boolean
  -h, --help            help for flags
  -n, --number int      an integer (default 42)
  -s, --string string   a string (default "foo")
```

相当不错，是吗？

这只是 Cobra 库能够完成的微小示例，但已经足以让我们构建一组强大的配置选项。如果您有兴趣了解更多关于 Cobra 及其如何用于构建强大命令行界面的信息，请查看其[GitHub 仓库](https://oreil.ly/oy7EN)和其[GoDoc 列表](https://oreil.ly/JOeoJ)。

## 使用文件配置

最后但同样重要的是，我们可能使用最广泛的配置选项：配置文件。

对于更复杂的应用程序，配置文件比环境变量具有许多优势。它们通常更为明确和可理解，通过允许将行为逻辑地分组和注释来实现。通常，理解如何使用配置文件只是查看其结构或其使用示例的问题。

配置文件在管理大量选项时特别有用，这是它们优于环境变量和命令行标志的优势。特别是命令行标志有时可能会导致一些冗长且构造繁琐的语句。

文件并非完美的解决方案。根据您的环境，在大规模分发文件并在集群中保持一致性可能会面临挑战。可以通过拥有单一的“真相源”来改善这种情况，例如像 etcd 或 HashiCorp Consul 这样的分布式键/值存储，或者从中部署自动提取其配置的中央源代码仓库，但这会增加复杂性并依赖于其他资源。

幸运的是，大多数编排平台提供专门的配置资源，例如 Kubernetes 的`ConfigMap`对象，大大缓解了分发问题。

近年来可能有数十种用于配置的文件格式，但特别是两种最为突出：JSON 和 YAML。在接下来的几节中，我们将更详细地讨论这两种格式以及如何在 Go 中使用它们。

### 我们的配置数据结构

在我们讨论文件格式及其如何解码之前，我们应该讨论配置可以解码的两种一般方式：

+   配置键和值可以映射到特定结构类型中的相应字段。例如，包含属性`host: localhost`的配置可以解码为具有`Host string`字段的结构类型。

+   配置数据可以解码和解组成一个或多个可能嵌套的`map[string]interface{}`类型的映射。当处理任意配置时这非常方便，但在操作时却有些棘手。

如果您预先知道您的配置可能是什么样子（通常是这样），那么解码配置的第一种方法，将它们映射到为此目的创建的数据结构中，显然是最简单的方法。尽管可以解码并处理任意配置模式，并做有用的工作，但这样做可能非常乏味，并且不适合大多数配置目的。

因此，在本节的其余部分，我们的示例配置将对应以下`Config`结构：

```go
type Config struct {
    Host string
    Port uint16
    Tags map[string]string
}
```

###### 警告

对于结构体字段要能被*任何*编码包编组或解组，它*必须*以大写字母开头，以指示它被包的导出。

对于我们的每一个例子，我们将从`Config`结构开始，偶尔会用格式特定的标签或其他装饰增强它。

### 使用 JSON

JSON（JavaScript 对象表示法）是在 21 世纪初发明的，是为了取代 XML 和当时使用的其他格式而需要的现代数据交换格式。它基于 JavaScript 脚本语言的一个子集，使其相对易于阅读，并对机器生成和解析都非常高效，同时提供了 XML 中缺少的列表和映射的语义。

尽管 JSON 非常常见和成功，但它确实有一些缺点。它通常被认为不如 YAML 用户友好。它的语法尤其严格，一个错位（或缺失）的逗号就可能轻易地破坏它，甚至它不支持注释。

然而，在本章介绍的格式中，它是 Go 标准库唯一支持的格式。

下面是一个关于将数据编码和解码到 JSON 的非常简要的介绍。想要更全面地了解，请参阅 Andrew Gerrand 在[*Go 博客*](https://oreil.ly/6Uvl2)上的“JSON 和 Go”。

#### JSON 编码

理解如何解码 JSON（或任何配置格式）的第一步是理解如何*编码*它。这可能看起来很奇怪，特别是在一个关于读取配置文件的部分，但编码对于 JSON 编码的一般主题非常重要，并提供了一个方便的方法来生成、测试和调试您的配置文件。⁹

Go 的标准库支持 JSON 编码和解码，提供了许多有用的辅助函数，用于编码、解码、格式化、验证和其他处理 JSON 数据。

其中之一是`json.Marshal`函数，它接受一个`interface{}`类型的值`v`，并返回一个包含`v`的 JSON 编码表示的`[]byte`数组：

```go
func Marshal(v interface{}) ([]byte, error)
```

换句话说，一个值进去，JSON 出来。

这个函数确实像它看起来那样容易使用。例如，如果我们有一个`Config`的实例，我们可以将它传递给`json.Marshal`以获取其 JSON 编码：

```go
c := Config{
    Host: "localhost",
    Port: 1313,
    Tags: map[string]string{"env": "dev"},
}

bytes, err := json.Marshal(c)

fmt.Println(string(bytes))
```

如果一切按预期进行，则`err`将为`nil`，并且`bytes`将是包含 JSON 的`[]byte`值。`fmt.Println`的输出将类似于以下内容：

```go
{"Host":"localhost","Port":1313,"Tags":{"env":"dev"}}
```

###### 提示

`json.Marshal`函数递归遍历`v`的值，因此任何内部结构都将被编码，以及嵌套的 JSON。

这相当轻松，但如果我们生成配置文件，如果文本格式化为人类可读，那将非常好。幸运的是，`encoding/json`还提供以下`json.MarshalIndent`函数，它返回“漂亮打印”的 JSON：

```go
func MarshalIndent(v interface{}, prefix, indent string) ([]byte, error)
```

正如你所看到的，`json.MarshalIndent`的工作方式与`json.Marshal`非常相似，只是还接受`prefix`和`indent`字符串，如此处所示：

```go
bytes, err := json.MarshalIndent(c, "", "   ")
fmt.Println(string(bytes))
```

前面的片段打印出了我们希望看到的内容：

```go
{
   "Host": "localhost",
   "Port": 1313,
   "Tags": {
      "env": "dev"
   }
}
```

结果是漂亮打印的 JSON，为像你和我这样的人类阅读格式化。这是一个非常有用的方法来引导配置文件！

#### 解码 JSON

现在我们知道如何将数据结构编码为 JSON 后，让我们看看如何将 JSON 解码为现有的数据结构。

要实现这一点，我们使用方便命名的`json.Unmarshal`函数：

```go
func Unmarshal(data []byte, v interface{}) error
```

`json.Unmarshal`函数解析`data`数组中包含的 JSON 编码文本，并将结果存储在`v`指向的值中。重要的是，如果`v`为`nil`或不是指针，则`json.Unmarshal`将返回错误。

但是，`v`应该是什么类型呢？理想情况下，它应该是指向数据结构的指针，其字段恰好对应 JSON 结构。虽然可以将任意 JSON 解组为非结构化映射，正如我们将在“解码任意 JSON”中讨论的那样，但如果真的没有其他选择，就应该这样做。

正如我们将看到的那样，如果您有一个反映 JSON 结构的数据类型，那么`json.Unmarshal`能够直接更新它。要做到这一点，我们首先必须创建一个实例，用于存储我们解码后的数据：

```go
c := Config{}
```

现在我们有了存储值，我们可以调用`json.Unmarshal`，将包含我们的 JSON 数据和指向`c`的指针的`[]byte`传递给它：

```go
bytes := []byte(`{"Host":"127.0.0.1","Port":1234,"Tags":{"foo":"bar"}}`)
err := json.Unmarshal(bytes, &c)
```

如果`bytes`包含有效的 JSON，则`err`将为`nil`，并且来自`bytes`的数据将存储在结构体`c`中。现在打印`c`的值应该提供以下输出：

```go
{127.0.0.1 1234 map[foo:bar]}
```

不错！但是当 JSON 的结构与 Go 类型不完全匹配时会发生什么呢？让我们找出来：

```go
c := Config{}
bytes := []byte(`{"Host":"127.0.0.1", "Food":"Pizza"}`)
err := json.Unmarshal(bytes, &c)
```

有趣的是，这段代码片段并不像你预期的那样产生错误。相反，`c`现在包含以下值：

```go
{127.0.0.1 0 map[]}
```

看起来`Host`的值已经设置，但`Config`结构中没有对应值的`Food`被忽略了。事实证明，`json.Unmarshal`只会解码目标类型中能找到的字段。如果你只想从一个大的 JSON 块中挑选几个特定的字段，这种行为实际上非常有用。

#### 使用结构体字段标签进行字段格式化

在内部，编组通过使用反射来检查一个值并为其类型生成适当的 JSON 来工作。对于结构体，结构体的字段名直接用作默认的 JSON 键，而结构体的字段值成为 JSON 值。解组基本上以相同的方式工作，只是反向操作。

当您编组一个零值结构体时会发生什么？嗯，事实证明，当您编组一个例如`Config{}`值时，您得到的 JSON 如下：

```go
{"Host":"","Port":0,"Tags":null}
```

这不太美观。或者高效。难道真的有必要输出所有空值吗？

同样地，结构体字段必须导出（因此大写）才能进行写入或读取。这是否意味着我们必须使用大写字段名？

幸运的是，这两个问题的答案都是“否”。

Go 支持使用*结构体字段标签*——出现在字段类型声明后的结构体中的短字符串，允许在特定结构字段上添加元数据。字段标签最常由编码包使用，以修改字段级别的编码和解码行为。

Go 结构字段标签是特殊字符串，包含一个或多个键/值对，这些键/值对在字段类型声明后的反引号中：

```go
type User struct {
    Name string `example:"name"`
}
```

在这个例子中，结构体的`Name`字段被标记为`example:"name"`。这些标签可以通过运行时反射使用`reflect`包访问，但它们最常见的用例是提供编码和解码指令。

`encoding/json`包支持多种此类标签。一般格式使用结构体字段标签中的`json`键，并指定字段的名称，可能跟随以逗号分隔的选项列表。名称可以为空，以便在不覆盖默认字段名称的情况下指定选项。

`encoding/json`支持的可用选项如下所示：

自定义 JSON 键

默认情况下，结构体字段将大小写敏感地映射到与字段名完全相同的 JSON 键。通过设置标签的选项列表中的第一个（或唯一的）值，可以覆盖此默认名称。

示例：`` CustomKey string `json:"custom_key"` ``

忽略空值

默认情况下，即使字段为空，它也会出现在 JSON 中。使用`omitempty`选项将导致如果字段包含零值，则跳过它们。请注意`omitempty`前面的逗号！

示例：`` OmitEmpty string `json:",omitempty"` ``

忽略字段

使用`-`（破折号）选项的字段在编码和解码期间始终完全被忽略。

示例：`` IgnoredName string `json:"-"` ``

一个使用了所有前述标签的结构体可能如下所示：

```go
type Tagged struct {
    // CustomKey will appear in JSON as the key "custom_key".
    CustomKey   string `json:"custom_key"`

    // OmitEmpty will appear in JSON as "OmitEmpty" (the default),
    // but will only be written if it contains a nonzero value.
    OmitEmpty   string `json:",omitempty"`

    // IgnoredName will always be ignored.
    IgnoredName string `json:"-"`

    // TwoThings will appear in JSON as the key "two_things",
    // but only if it isn't empty.
    TwoThings   string `json:"two_things,omitempty"`
}
```

欲了解有关`json.Marshal`如何编码数据的更多信息，请参阅[golang.org 上该函数的文档](https://oreil.ly/5QeJ4)。

### 使用 YAML

YAML（YAML Ain’t Markup Language^(11）是一种可扩展的文件格式，在依赖于复杂分层配置的项目（如 Kubernetes）中很受欢迎。它非常表现力强，尽管其语法可能有点脆弱，并且随着规模扩大，使用它的配置可能开始遭遇可读性问题。

与最初作为数据交换格式创建的 JSON 不同，YAML 在本质上主要是一种配置语言。然而，有趣的是，YAML 1.2 是 JSON 的超集，这两种格式可以相互转换。不过，相比 JSON，YAML 确实有一些优势：它可以自引用，允许嵌入块文字，支持注释和复杂数据类型。

不像 JSON，Go 的核心库不支持 YAML。虽然有几个 YAML 包可供选择，但标准选择是[Go-YAML](https://oreil.ly/yhERJ)。Go-YAML 的第一个版本始于 2014 年，是 Canonical 内部项目，旨在将著名的`libyaml` C 库移植到 Go。作为一个项目，它非常成熟且得到良好维护。其语法与`encoding/json`非常相似，非常方便。

#### 编码 YAML

使用 Go-YAML 编码数据与编码 JSON 非常相似。实际上，两个包的`Marshal`函数的签名完全相同。与其`encoding/json`等效项一样，Go-YAML 的`yaml.Marshal`函数也接受`interface{}`值，并将其 YAML 编码作为`[]byte`值返回：

```go
func Marshal(v interface{}) ([]byte, error)
```

正如我们在“编码 JSON”中所做的那样，我们通过创建`Config`的实例来演示其用法，然后将其传递给`yaml.Marshal`以获取其 YAML 编码：

```go
c := Config{
    Host: "localhost",
    Port: 1313,
    Tags: map[string]string{"env": "dev"},
}

bytes, err := yaml.Marshal(c)
```

再次地，如果一切按预期工作，`err`将是`nil`，`bytes`将是包含 YAML 的`[]byte`值。打印`bytes`的字符串值将提供如下内容：

```go
host: localhost
port: 1313
tags:
  env: dev
```

此外，就像`encoding/json`提供的版本一样，Go-YAML 的`Marshal`函数会递归遍历值`v`。它发现的任何复合类型——数组、切片、映射和结构体——都将被适当编码，并作为嵌套的 YAML 元素出现在输出中。

#### 解码 YAML

与我们已建立的主题相一致，与`encoding/json`和 Go-YAML 相似的`Marshal`函数之间也表现出了相同的一致性：

```go
func Unmarshal(data []byte, v interface{}) error
```

再次，`yaml.Unmarshal`函数解析`data`数组中的 YAML 编码数据，并将结果存储在指向的`v`值中。如果`v`为`nil`或不是指针，则`yaml.Unmarshal`会返回错误。如下所示，这些相似之处非常明显：

```go
// Caution: Indent this YAML with spaces, not tabs.
bytes := []byte(`
host: 127.0.0.1
port: 1234
tags:
 foo: bar
`)

c := Config{}
err := yaml.Unmarshal(bytes, &c)
```

就像我们在“解码 JSON”中所做的那样，我们将一个指向`Config`实例的指针传递给`yaml.Unmarshal`，其字段与 YAML 中找到的字段对应。打印`c`的值应该（再次）会提供以下输出：

```go
{127.0.0.1 1234 map[foo:bar]}
```

`encoding/json`和 Go-YAML 之间还有其他行为上的相似之处：

+   两者都会忽略源文档中无法映射到`Unmarshal`函数的属性。同样，如果你忘记导出结构字段，`Unmarshal`将会默默地忽略它，从而导致其永远不会被设置。

+   通过将`interface{}`值传递给`Unmarshal`，这两者都能够解码任意数据。然而，`json.Unmarshal`将提供一个`map[string]interface{}`，而`yaml.Unmarshal`则会返回一个`map[interface{}]interface{}`。这是一个细微的差别，但可能会导致问题！

#### 用于 YAML 的结构字段标签

除了“标准”结构字段标签外——自定义键，`omitempty`和`-`（破折号）——在“使用结构字段标签进行字段格式化”中详细说明，Go-YAML 还支持两个特定于 YAML 编组格式的附加标签：

流样式

使用`flow`选项的字段将使用[流样式](https://oreil.ly/zyUpd)进行编组，这对于结构体、序列和映射非常有用。

示例：`` Flow map[string]string `yaml:"flow"` ``

内联结构体和映射

`inline`选项会导致结构体或映射的所有字段或键都被处理为外部结构的一部分。对于映射，键必须与其他结构字段的键不冲突。

示例：`` Inline map[string]string `yaml:",inline"` ``

一个结构体可以同时使用这两个选项的例子如下：

```go
type TaggedMore struct {
    // Flow will be marshalled using a "flow" style
    // (useful for structs, sequences and maps).
    Flow map[string]string `yaml:"flow"`

    // Inlines a struct or a map, causing all of its fields
    // or keys to be processed as if they were part of the outer
    // struct. For maps, keys must not conflict with the yaml
    // keys of other struct fields.
    Inline map[string]string `yaml:",inline"`
}
```

如你所见，标签语法也是一致的，只是不再使用`json`前缀，而是使用`yaml`前缀。

### 监视配置文件变化

在处理配置文件时，你将不可避免地面对一个情况：必须对正在运行的程序的配置进行更改。如果程序没有明确地监视并重新加载更改，那么通常必须重新启动以重新读取其配置，这可能在最好的情况下会带来不便，而在最坏的情况下则可能导致停机。

在某个时刻，你必须决定如何让你的程序对这些变化做出响应。

第一种（也是最简单的）选择是什么也不做，只是期望程序在配置更改时重新启动。这实际上是一个相当常见的选择，因为它确保了旧配置的任何痕迹都不存在。它还允许程序在配置文件中引入错误时“快速失败”：程序只需输出一条愤怒的错误消息并拒绝启动。

但是，你可能更喜欢在程序中添加逻辑来检测配置文件（或文件）的更改，并适当地重新加载它们。

#### 使您的配置可重新加载

如果您希望在底层文件更改时重新加载内部配置表示形式，则需要事先进行一些规划。

首先，你需要有一个全局唯一的配置结构体实例。目前，我们将使用我们在“我们的配置数据结构”中介绍的`Config`实例。在稍大一点的项目中，你甚至可以将其放在一个`config`包中：

```go
var config Config
```

经常会看到这样的代码，其中几乎每个方法和函数都会传递一个显式的`config`参数。我经常看到这种情况；足够多，以至于我知道这种特定的反模式只会让生活变得更加困难。此外，由于现在配置存在于多个地方而不是一个地方，这也倾向于使得配置重新加载变得更加复杂。

一旦我们有了我们的`config`值，我们将希望添加读取配置文件并将其加载到结构体中的逻辑。类似以下的`loadConfiguration`函数将完美地胜任：

```go
func loadConfiguration(filepath string) (Config, error) {
    dat, err := ioutil.ReadFile(filepath)   // Ingest file as []byte
    if err != nil {
        return Config{}, err
    }

    config := Config{}

    err = yaml.Unmarshal(dat, &config)      // Do the unmarshal
    if err != nil {
        return Config{}, err
    }

    return config, nil
}
```

我们的`loadConfiguration`函数几乎与我们在“使用 YAML”中讨论过的方式相同，只是它使用了`io/ioutil`标准库中的`ioutil.ReadFile`函数来获取传递给`yaml.Unmarshal`的字节。在这里使用 YAML 的选择完全是任意的。¹² JSON 配置的语法几乎相同。

现在我们有了将配置文件加载到规范结构体中的逻辑，我们需要在文件发生更改时调用它。为此，我们有`startListening`，它监视一个`updates`通道：

```go
func startListening(updates <-chan string, errors <-chan error) {
    for {
        select {
        case filepath := <-updates:
            c, err := loadConfiguration(filepath)
            if err != nil {
                log.Println("error loading config:", err)
                continue
            }
            config = c

        case err := <-errors:
            log.Println("error watching config:", err)
        }
    }
}
```

正如你所看到的，`startListening`接受两个通道：`updates`，当文件（假设是配置文件）更改时会发出文件名，以及一个`errors`通道。

它在一个无限循环的`select`中监视两个通道，以便如果配置文件发生更改，`updates`通道发送其名称，然后传递给`loadConfiguration`。如果`loadConfiguration`没有返回非`nil`错误，则其返回的`Config`值将替换当前值。

再退一步，我们有一个`init`函数，它从`watchConfig`函数中获取通道，并将它们传递给`startListening`，然后作为一个 goroutine 运行：

```go
func init() {
    updates, errors, err := watchConfig("config.yaml")
    if err != nil {
        panic(err)
    }

    go startListening(updates, errors)
}
```

那么这个`watchConfig`函数是什么呢？嗯，我们还不太清楚细节。我们将在接下来的几节中弄清楚。我们知道它实现了一些配置监视逻辑，并且它有一个函数签名看起来像下面这样：

```go
func watchConfig(filepath string) (<-chan string, <-chan error, error)
```

`watchConfig`函数，无论其实现如何，都会返回两个通道——一个`string`通道，发送更新的配置文件路径，以及一个`error`通道，通知无效配置——还有一个`error`值，报告启动时是否发生致命错误。

`watchConfig`的确切实现可以有几种不同的方式，每种方式都有其利弊。现在让我们看看其中两种最常见的方式。

#### 轮询配置更改

轮询，您可以在一定的时间间隔内检查配置文件的更改，这是一种常见的观察配置文件的方式。标准的实现使用`time.Ticker`每隔几秒重新计算配置文件的哈希值，并在哈希值更改时重新加载。

Go 在其`crypto`包中提供了许多常见的哈希算法，每个算法都位于`crypto`的子包中，并满足`crypto.Hash`和`io.Writer`接口。

例如，Go 的`SHA256`标准实现可以在`crypto/sha256`中找到。要使用它，您可以使用其`sha256.New`函数获取一个新的`sha256.Hash`值，然后像对待任何`io.Writer`一样将要计算哈希的数据写入其中。完成后，使用其`Sum`方法检索生成的哈希总和：

```go
func calculateFileHash(filepath string) (string, error) {
    file, err := os.Open(filepath)  // Open the file for reading
    if err != nil {
        return "", err
    }
    defer file.Close()              // Be sure to close your file!

    hash := sha256.New()            // Use the Hash in crypto/sha256

    if _, err := io.Copy(hash, file); err != nil {
        return "", err
    }

    sum := fmt.Sprintf("%x", hash.Sum(nil))  // Get encoded hash sum

    return sum, nil
}
```

为配置生成哈希有三个明确的部分。首先，我们以`io.Reader`的形式获取一个`[]byte`源。在本例中，我们使用`io.File`。接下来，我们将这些字节从`io.Reader`复制到我们的`sha256.Hash`实例中，使用`io.Copy`方法完成。最后，我们使用`Sum`方法从`hash`中检索哈希总和。

现在我们有了我们的`calculateFileHash`函数，创建我们的`watchConfig`实现只是使用`time.Ticker`并发地在某个节奏上检查它，并将任何积极的结果（或错误）发送到适当的通道的问题：

```go
func watchConfig(filepath string) (<-chan string, <-chan error, error) {
    errs := make(chan error)
    changes := make(chan string)
    hash := ""

    go func() {
        ticker := time.NewTicker(time.Second)

        for range ticker.C {
            newhash, err := calculateFileHash(filepath)
            if err != nil {
                errs <- err
                continue
            }

            if hash != newhash {
                hash = newhash
                changes <- filepath
            }
        }
    }()

    return changes, errs, nil
}
```

轮询方法有一些好处。它并不特别复杂，这总是一个大的优点，并且适用于任何操作系统。也许最有趣的是，因为哈希只关心配置的内容，它甚至可以泛化到检测像远程键/值存储这样的地方的更改，这些地方在技术上不是文件。

不幸的是，轮询方法可能会有些计算资源浪费，尤其是对于非常大或文件数量众多的情况。由其性质决定，它还会在文件更改后和检测到该更改之间存在短暂的延迟。如果您确实要处理本地文件，可能更高效的方法是监视操作系统级别的文件系统通知，我们将在下一节讨论。

#### 监视操作系统文件系统通知

虽然轮询变化的方法效果足够好，但这种方法也有一些缺点。根据您的使用情况，您可能会发现监视操作系统级别的文件系统通知更为高效。

实际上，这样做的复杂性在于每个操作系统具有不同的通知机制。幸运的是，[fsnotify](https://oreil.ly/ziw4J) 包提供了一个可行的抽象，支持大多数操作系统。

要使用此包监视一个或多个文件，您可以使用 `fsnotify.NewWatcher` 函数获取一个新的 `fsnotify.Watcher` 实例，并使用 `Add` 方法注册更多要监视的文件。`Watcher` 提供两个通道，`Events` 和 `Errors`，分别发送文件事件和错误的通知。

例如，如果我们想要监视我们的配置文件，我们可以像下面这样做：

```go
func watchConfigNotify(filepath string) (<-chan string, <-chan error, error) {
    changes := make(chan string)

    watcher, err := fsnotify.NewWatcher()         // Get an fsnotify.Watcher
    if err != nil {
        return nil, nil, err
    }

    err = watcher.Add(filepath)                    // Tell watcher to watch
    if err != nil {                                // our config file
        return nil, nil, err
    }

    go func() {
        changes <- filepath                        // First is ALWAYS a change

        for event := range watcher.Events {        // Range over watcher events
            if event.Op&fsnotify.Write == fsnotify.Write {
                changes <- event.Name
            }
        }
    }()

    return changes, watcher.Errors, nil
}
```

注意语句 `event.Op & fsnotify.Write == fsnotify.Write`，它使用按位与 (`&`) 来筛选“写入”事件。我们这样做是因为 `fsnotify.Event` 可能包含多个操作，每个操作在无符号整数中表示为一个位。例如，同时发生 `fsnotify.Write` (`2`，二进制 `0b00010`) 和 `fsnotify.Chmod` (`16`，二进制 `0b10000`) 会导致 `event.Op` 值为 `18` (二进制 `0b10010`)。因为 `0b10010 & 0b00010 = 0b00010`，按位与允许我们确保操作包括 `fsnotify.Write`。

## Viper：配置包的瑞士军刀

[Viper (spf13/viper)](https://oreil.ly/pttZM) 自称是 Go 应用程序的完整配置解决方案，这确实如此。除其他功能外，它允许通过多种机制和格式进行应用程序配置，包括优先顺序为：

明确设置值

这比所有其他方法优先，测试期间非常有用。

命令行标志

Viper 被设计为 Cobra 的伴侣，我们在 “Cobra 命令行解析器” 中介绍过。

环境变量

Viper 对环境变量有全面支持。重要的是，Viper 将环境变量视为区分大小写！

配置文件，支持多种文件格式

Viper 出厂支持 JSON 和 YAML 与我们之前介绍的包，以及 TOML、HCL、INI、envfile 和 Java Properties 文件。它还可以编写配置文件以帮助启动配置，并且可选择支持配置文件的实时监视和重新读取。

远程键值存储

Viper 可以访问诸如 etcd 或 Consul 之类的键值存储，并监视其变化。

它还支持诸如默认值和类型化变量等特性，这些通常标准包不提供。

但请记住，尽管 Viper 做了很多事情，但它也是一个引入许多依赖的大锤。如果您尝试构建一个精简、流畅的应用程序，Viper 可能会超出您的需求。

### 明确在 Viper 中设置值

Viper 允许您使用 `viper.Set` 函数从命令行标志或应用程序逻辑显式设置值。这在测试期间非常方便：

```go
viper.Set("Verbose", true)
viper.Set("LogFile", LogFile)
```

明确设置的值具有最高优先级，并覆盖其他机制设置的值。

### 在 Viper 中处理命令行标志

Viper 的设计目标是成为 [Cobra 库](https://oreil.ly/67LFI) 的伴侣，我们在 “The Cobra command-line parser” 的构建命令行界面的上下文中简要讨论过。与 Cobra 的密切集成使得将命令行标志绑定到配置键变得非常简单。

Viper 提供了 `viper.BindPFlag` 函数，允许将单独的命令行标志绑定到命名键，并且 `viper.BindPFlags` 可以绑定完整的标志集，使用每个标志的长名称作为键。

因为只有在访问绑定时才设置配置值的实际值，而不是在调用时，所以可以在 `init` 函数中调用 `viper.BindPFlag`，正如我们在这里所做的：

```go
var rootCmd = &cobra.Command{ /* omitted for brevity */ }

func init() {
    rootCmd.Flags().IntP("number", "n", 42, "an integer")
    viper.BindPFlag("number", rootCmd.Flags().Lookup("number"))
}
```

在上述代码片段中，我们声明了一个 `&cobra.Command` 并定义了一个名为“number”的整数标志。请注意，我们使用 `IntP` 方法而不是 `IntVarP`，因为在这种方式下，当使用 Cobra 时无需将标志的值存储在外部值中。然后，使用 `viper.BindPFlag` 函数，我们将“number”标志绑定到相同名称的配置键。

绑定后（和命令行标志解析后），可以通过使用 `viper.GetInt` 函数从 Viper 获取绑定键的值：

```go
n := viper.GetInt("number")
```

### 在 Viper 中处理环境变量

Viper 提供了几个函数来处理环境变量作为配置源的情况。其中第一个是 `viper.BindEnv`，用于将配置键绑定到环境变量：

```go
viper.BindEnv("id")                     // Bind "id" to var "ID"
viper.BindEnv("port", "SERVICE_PORT")   // Bind "port" to var "SERVICE_PORT"

id := viper.GetInt("id")
id := viper.GetInt("port")
```

如果仅提供了一个键，`viper.BindEnv` 将绑定到与键匹配的环境变量。可以提供更多参数来指定要绑定的一个或多个环境变量。在这两种情况下，Viper 自动假设环境变量的名称全部大写。

Viper 还提供了几个额外的辅助函数来处理环境变量。更多详细信息请参见 [Viper GoDoc](https://oreil.ly/CGpPS)。

### 在 Viper 中处理配置文件

Viper 默认支持使用我们之前介绍的包支持的 JSON 和 YAML，以及 TOML、HCL、INI、envfile 和 Java Properties 文件。它还可以编写配置文件来帮助启动配置，并且甚至可以选择支持实时观察和重新读取配置文件。

在云原生书籍中讨论本地配置文件可能看起来出乎意料，但文件在任何情境中仍然是一个常用的结构。毕竟，共享文件系统——无论是 Kubernetes ConfigMaps 还是 NFS 挂载——都很常见，甚至云原生服务也可以通过配置管理系统部署，该系统为所有服务副本安装一个只读本地文件副本以供读取。配置文件甚至可以以一种看起来——就容器化服务而言——与任何其他本地文件完全相同的方式被烘焙或挂载到容器镜像中。

#### 读取配置文件

要从文件中读取配置，Viper 只需要知道文件的名称和位置。此外，如果无法从文件扩展名中推断出类型，则还需要知道其类型。`viper.ReadInConfig` 函数指示 Viper 查找并读取配置文件，如果出现问题，则可能返回一个 `error` 值。所有这些步骤在这里演示：

```go
viper.SetConfigName("config")

// Optional if the config has a file extension
viper.SetConfigType("yaml")

viper.AddConfigPath("/etc/service/")
viper.AddConfigPath("$HOME/.service")
viper.AddConfigPath(".")

if err := viper.ReadInConfig(); err != nil {
    panic(fmt.Errorf("fatal error reading config: %w", err))
}
```

如您所见，Viper 可以搜索多个路径以查找配置文件。不幸的是，此时，单个 Viper 实例仅支持读取单个配置文件。

#### 在 Viper 中监视和重新读取配置文件

Viper 本身允许您的应用程序监视配置文件的修改并在检测到更改时重新加载，这意味着配置可以在不必重新启动服务器的情况下进行更改以生效。

默认情况下，此功能处于关闭状态。`viper.WatchConfig` 函数可用于启用它。此外，`viper.OnConfigChange` 函数允许您指定在更新配置文件时调用的函数：

```go
viper.WatchConfig()
viper.OnConfigChange(func(e fsnotify.Event) {
    fmt.Println("Config file changed:", e.Name)
})
```

###### 警告

确保在调用 `viper.WatchConfig` 之前进行任何对 `viper.AddConfigPath` 的调用。

有趣的是，Viper 实际上在幕后使用了 `fsnotify/fsnotify` 包，这是我们在 “监视配置文件更改” 中详细介绍的相同机制。

### 使用 Viper 与远程键/值存储

Viper 最有趣的功能之一可能是其能够从远程键/值存储中的路径读取以任何支持的格式编写的配置字符串，例如 [etcd](https://etcd.io) 或 [HashiCorp Consul](https://consul.io)。这些值优先于默认值，但会被从磁盘、命令行标志或环境变量中检索的配置值覆盖。

要在 Viper 中启用远程支持，首先必须对 `viper/remote` 包进行空白导入：

```go
import _ "github.com/spf13/viper/remote"
```

然后可以使用 `viper.AddRemoteProvider` 方法注册远程键/值配置源，其签名如下：

```go
func AddRemoteProvider(provider, endpoint, path string) error
```

+   `provider` 参数可以是 `etcd`、`consul` 或 `firestore` 中的一个。

+   `endpoint` 是远程资源的 URL。Viper 的一个奇怪之处是 etcd 提供程序要求 URL 包含方案 (`http://ip:port`)，而 Consul 则要求*没有方案* (ip:port)。

+   `path`是从键值存储中检索配置的路径。

例如，要从 etcd 服务中读取一个 JSON 格式的配置文件，你可以这样做：

```go
viper.AddRemoteProvider("etcd", "http://127.0.0.1:4001","/config/service.json")
viper.SetConfigType("json")
err := viper.ReadRemoteConfig()
```

请注意，尽管配置路径包括文件扩展名，但我们还是使用`viper.SetConfigType`来明确定义配置类型。这是因为从 Viper 的角度来看，资源只是一串字节流，因此无法自动推断格式。¹³ 在撰写本文时，支持的格式包括 *json*、*toml*、*yaml*、*yml*、*properties*、*props*、*prop*、*env* 和 *dotenv*。

可以添加多个提供程序，按添加的顺序进行搜索。

这只是对 Viper 在使用远程键值存储时的基本介绍。要了解更多关于如何使用 Viper 从 Consul 中读取、监视配置更改或读取加密配置的详细信息，请参阅[Viper 的 README](https://oreil.ly/1iE2y)。

### 在 Viper 中设置默认值

与本章中审查的所有其他包不同，Viper 可选地允许通过`SetDefault`函数为键定义默认值。

默认值有时可能很有用，但在使用此功能时需要小心。如在“配置良好实践”中提到的，有用的零值通常比隐式默认值更可取，因为后者可能会在不加思索地应用时导致意外行为。

在下面的示例中，我们展示了 Viper 中显示默认值的片段示例：

```go
viper.BindEnv("id")             // Will be upper-cased automatically
viper.SetDefault("id", "13")    // Default value is "13"

id1 := viper.GetInt("id")
fmt.Println(id1)                // 13

os.Setenv("ID", "50")           // Explicitly set the envvar

id2 := viper.GetInt("id")
fmt.Println(id2)                // 50
```

默认值优先级最低，仅在未通过其他机制明确设置键时才会生效。

# 使用特性标志进行特性管理

*特性标记*（或*特性切换*^(14）是一种软件开发模式，旨在通过允许在运行时打开或关闭特定功能，而无需部署新代码，从而提高开发和交付新功能的速度和安全性。

特性标志本质上是您代码中的条件语句，根据某些外部条件（通常但并非总是配置设置）启用或禁用功能。通过设置不同的配置值，开发人员可以选择为测试启用不完整的功能，并对其他用户禁用它。

具备发布未完成特性的能力，带来了许多强大的好处。

首先，特性标志允许交付许多小的增量软件版本，而无需使用特性分支带来的分支和合并开销。换句话说，特性标志将特性的发布与其部署解耦。结合特性标志的本质，要求尽早集成代码更改，这既鼓励又促进了持续部署和交付。因此，开发人员对其代码获得更快速的反馈，从而允许更小、更快和更安全的迭代。

其次，特性标志不仅可以在准备发布之前更轻松地进行测试，而且还可以动态进行。例如，可以使用逻辑来构建反馈循环，可以与类似断路器的模式结合使用，在特定条件下自动启用或禁用标志。

最后，逻辑执行标志甚至可以用来针对特定用户子集推出特性。这种技术称为*特性门控*，可以用作金丝雀部署和分阶段或地理基础的推出的替代方案。结合可观察性技术，特性门控甚至可以让您更轻松地执行像 A/B 测试或有针对性的追踪这样的实验，仪器化特定用户基础或甚至单个客户的特定切片。

## 特性标志的演变

在本节中，我们将通过直接从我们在第五章构建的键值 REST 服务中取出的函数迭代实现一个特性标志。从基线函数开始，我们将逐步进入几个演变阶段，从无特性标志到为特定用户子集开启的动态特性标志。

在我们的场景中，我们决定要能够扩展我们的键值存储，因此我们希望更新逻辑，使其支持一种精密的分布式数据结构，而不是本地映射。

## 第一代：初始实现

对于我们的第一个迭代，我们将从“实现读函数”中的`keyValueGetHandler`函数开始。您可能还记得`keyValueGetHandler`是一个 HTTP 处理函数，满足`net/http`包中定义的`HandlerFunc`接口。如果您对这意味着什么有些生疏，您可能想回顾一下“使用 net/http 构建 HTTP 服务器”。

初始处理函数几乎直接从第五章复制过来（为简洁起见省略了部分错误处理），如下所示：

```go
func keyValueGetHandler(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)                     // Retrieve "key" from the request
    key := vars["key"]

    value, err := Get(key)                  // Get value for key
    if err != nil {                         // Unexpected error!
        http.Error(w,
            err.Error(),
            http.StatusInternalServerError)
        return
    }

    w.Write([]byte(value))                  // Write the value to the response
}
```

正如您所见，此函数没有特性切换逻辑（或确实没有任何切换的东西）。它所做的只是从请求变量中检索键，使用`Get`函数检索与该键关联的值，并将该值写入响应。

在我们的下一个实现中，我们将开始测试一个新功能：一个高级分布式数据结构，用于取代本地的`map[string]string`，这将允许服务扩展到不止一个实例。

## 第一代：硬编码特性标志

在这个实现中，我们假设我们已经构建了新的实验性分布式后端，并通过`NewGet`函数使其可访问。

我们首次尝试创建特性标志时引入了一个条件，允许我们使用简单的布尔值`useNewStorage`在两种实现之间进行切换：

```go
// Set to true if you're working on the new storage backend
const useNewStorage bool = false;

func keyValueGetHandler(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    key := vars["key"]

    var value string
    var err error

    if useNewStorage {
        value, err = NewGet(key)
    } else {
        value, err = Get(key)
    }

    if err != nil {
        http.Error(w,
            err.Error(),
            http.StatusInternalServerError)
        return
    }

    w.Write([]byte(value))
}
```

这个第一次迭代显示了一些进展，但离我们想要的还有很远。在代码中将标志条件作为硬编码值固定使得我们可以在本地测试中很好地切换实现，但在自动化和持续的测试中同时测试两者将不容易。

另外，每当你想要在已部署的实例中更改算法时，你都必须重新构建和部署服务，这在很大程度上抵消了一开始引入特性标志的好处。

###### 小贴士

良好的特性标志使用规范很重要！如果你有一段时间没有更新特性标志，考虑移除它。

## 第二代：可配置标志

一段时间过去了，硬编码特性标志的缺点显而易见。首先，如果我们能够使用外部机制更改标志值以便在测试中同时测试*两种*算法，那将非常好。

在这个示例中，我们使用 Viper 绑定和读取一个环境变量，现在我们可以在运行时启用或禁用功能。配置机制的选择在这里并不重要。重要的是，我们能够在不重新构建代码的情况下外部更新标志：

```go
func keyValueGetHandler(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    key := vars["key"]

    var value string
    var err error

    if FeatureEnabled("use-new-storage", r) {
        value, err = NewGet(key)
    } else {
        value, err = Get(key)
    }

    if err != nil {
        http.Error(w,
            err.Error(),
            http.StatusInternalServerError)
        return
    }

    w.Write([]byte(value))
}

func FeatureEnabled(flag string, r *http.Request) bool {
    return viper.GetBool(flag)
}
```

除了使用 Viper 读取设置`use-new-storage`标志的环境变量外，我们还引入了一个新函数：`FeatureEnabled`。目前，它的作用只是执行`viper.GetBool(flag)`，但更重要的是，它还将标志读取逻辑集中在一个地方。我们将在下一次迭代中看到这样做的好处。

你可能会想为什么`FeatureEnabled`接受一个`*http.Request`。嗯，目前它还没有使用，但在下一个迭代中就会有意义。

## 第三代：动态特性标志

该特性现在已经部署，但在特性标志背后关闭。现在我们希望能够在用户基础的特定子集上在生产环境中测试它。显然，我们无法通过配置设置来实现这种标志。相反，我们将不得不构建可以*自行判断*是否应该设置的动态标志，这意味着将标志与函数关联起来。

### 动态标志作为函数

构建动态标志函数的第一步是决定函数的签名。虽然不是严格要求，但明确定义这一点，例如使用如下所示的函数类型，是很有帮助的：

```go
type Enabled func(flag string, r *http.Request) (bool, error)
```

`Enabled`函数类型是所有我们动态特性标志函数的原型。其合同定义了一个接受标志名称作为`string`和`*http.Request`的函数，并返回一个`bool`，如果请求的标志已启用，则返回`true`。

### 实现动态标志函数

使用由`Enabled`类型提供的合同，我们现在可以实现一个函数，用于通过比较请求的远程地址与专门为私有网络分配的标准 IP 范围列表，确定请求是否来自私有网络：

```go
// The list of CIDR ranges associated with internal networks.
var privateCIDRs []*net.IPNet

// We use an init function to load the privateCIDRs slice.
func init() {
    for _, cidr := range []string{
        "10.0.0.0/8",
        "172.16.0.0/12",
        "192.168.0.0/16",
    } {
        _, block, _ := net.ParseCIDR(cidr)
        privateCIDRs = append(privateCIDRs, block)
    }
}

// fromPrivateIP receives the flag name (which it ignores) and the
// request. If the request's remote IP is in a private range per
// RFC1918, it returns true.
func fromPrivateIP(flag string, r *http.Request) (bool, error) {
    // Grab the host portion of the request's remote address
    remoteIP, _, err := net.SplitHostPort(r.RemoteAddr)
    if err != nil {
        return false, err
    }

    // Turn the remote address string into a *net.IPNet
    ip := net.ParseIP(remoteIP)
    if ip == nil {
        return false, errors.New("couldn't parse ip")
    }

    // Loopbacks are considered "private."
    if ip.IsLoopback() {
        return true, nil
    }

    // Search the CIDRs list for the IP; return true if found.
    for _, block := range privateCIDRs {
        if block.Contains(ip) {
            return true, nil
        }
    }

    return false, nil
}
```

正如您所见，`fromPrivateIP`函数符合`Enabled`，通过接收`string`值（标志名称）和`*http.Request`（具体来说是与发起请求关联的实例），如果请求来自私有 IP 范围（由[RFC 1918](https://oreil.ly/lZ5PQ)定义），则返回`true`。

要做出这种确定，`fromPrivateIP`函数首先从`*http.Request`中检索包含发送请求的网络地址的远程地址。通过`net.SplitHostPort`解析主机 IP，然后使用`net.ParseIP`将其解析为`*net.IP`值，它将发起 IP 与`privateCIDRs`中包含的每个私有 CIDR 范围进行比较，如果找到匹配，则返回`true`。

###### 警告

如果请求正在通过负载均衡器或反向代理，则此函数还会返回`true`。生产级实现需要意识到这一点，并最好是[代理协议感知](https://oreil.ly/S3btg)的。

当然，这个函数只是一个例子。我之所以使用它，是因为它相对简单，但类似的技术可以用来为地理区域、用户固定百分比，甚至特定客户启用或禁用标志。

### 标志函数查找

现在我们有了`fromPrivateIP`形式的动态标志函数，我们必须实现一些机制来通过名称将标志与之关联起来。也许最直接的方法是使用标志名称字符串到`Enabled`函数的映射：

```go
var enabledFunctions map[string]Enabled

func init() {
    enabledFunctions = map[string]Enabled{}
    enabledFunctions["use-new-storage"] = fromPrivateIP
}
```

以这种方式使用映射间接引用函数，为我们提供了很大的灵活性。如果需要，我们甚至可以将函数与多个标志关联起来。如果我们希望一组相关特性在相同条件下始终处于活动状态，这可能非常有用。

您可能已经注意到，我们正在使用一个`init`函数来填充`enabledFunctions`映射。但等等，我们不是已经有一个`init`函数了吗？

是的，我们确实这样做了，这没问题。`init`函数很特别：如果需要，您可以拥有多个`init`函数。

### 路由器功能

最后，我们将一切联系起来。

我们通过重构`FeatureEnabled`函数来查找适当的动态标志函数，并在找到时调用它并返回结果：

```go
func FeatureEnabled(flag string, r *http.Request) bool {
    // Explicit flags take precedence
    if viper.IsSet(flag) {
        return viper.GetBool(flag)
    }

    // Retrieve the flag function, if any. If none exists,
    // return false
    enabledFunc, exists := enabledFunctions[flag]
    if !exists {
        return false
    }

    // We now have the flag function: call it and return
    // the result
    result, err := enabledFunc(flag, r)
    if err != nil {
        log.Println(err)
        return false
    }

    return result
}
```

此时，`FeatureEnabled` 已经成为一个完整的路由器函数，可以根据显式的特性标志设置和标志函数的输出动态控制哪些代码路径是活动的。在此实现中，显式设置的标志优先于其他一切。这允许自动化测试验证标记特性的两侧。

我们的实现使用简单的内存查找来确定特定标志的行为，但这也可以轻松地实现为数据库或其他数据源，甚至是像 LaunchDarkly 这样的复杂托管服务。不过，请记住，这些解决方案确实引入了新的依赖。

# 摘要

管理性可能不是云原生世界中最引人注目的主题——或者任何世界中的——但我仍然非常喜欢我们在这一章中详细处理细节的方式。

我们深入研究了各种配置样式的细节，包括环境变量、命令行标志和各种格式的文件。我们甚至讨论了一些检测配置更改以触发重新加载的策略。更不用说 Viper，它几乎做了所有这些事情及更多。

我觉得在某些事情上可能有很大的深入潜力，如果不是时间和空间的限制的话，我可能会有更多收获。例如，特性标志和特性管理是一个相当大的主题，我肯定希望能够更深入地探讨一下。有些主题，如部署和服务发现，我们甚至完全无法覆盖。我想我们在下一版中有一些值得期待的事情，对吧？

尽管我很喜欢这一章，但我对第十一章特别感兴趣，我们将深入探讨总体可观察性，特别是 OpenTelemetry。

最后，我给你一些建议：始终做自己，记住幸运来自努力工作。

¹ Kernighan, Brian W. 和 P. J. Plauger. *《程序设计风格的要素》*。McGraw-Hill，1978 年。

² America’s Test Kitchen 的工作人员。*《完美派：经典与现代派饼、馅饼、盖莱特及更多的终极指南》*。America’s Test Kitchen，2019 年。[*https://oreil.ly/rl5TP*](https://oreil.ly/rl5TP)。

³ 他们在基因工程方面做了一些非常惊人的事情。不要停止相信。

⁴ “系统与软件工程：词汇”。ISO/IEC/IEEE 24765:2010(E)，2010 年 12 月 15 日。[*https://oreil.ly/NInvC*](https://oreil.ly/NInvC)。

⁵ Radle，Byron 等人。《什么是可管理性？》*NI*，国家仪器，2019 年 3 月 5 日。[*https://oreil.ly/U3d7Q*](https://oreil.ly/U3d7Q)。

⁶ 我告诉过我的编辑们。嗨，阿米莉亚！嗨，赞！

⁷ 这让我很伤心。这些是重要的话题，但我们必须集中精力。

⁸ Pike，Rob。《Go Proverbs》。Gopherfest，2015 年 11 月 18 日，YouTube。[*https://oreil.ly/5bOxW*](https://oreil.ly/5bOxW)。

⁹ 很巧妙的技巧，对吧？

¹⁰ 嗯，就像你一样。

¹¹ 真的，那确实是它的含义。

¹² 而且，我真的非常喜欢 JSON *so much*。

¹³ 或者那个特性还没有被实现。我不知道。

¹⁴ 我也见过“特性开关”、“特性翻转”、“条件特性”等术语。行业似乎正在选择“标志”和“切换”，可能是因为其他名称有点滑稽。
