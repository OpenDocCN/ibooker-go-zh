# 9 集成测试

本章涵盖

+   将用户需求转换为描述性测试

+   编写遵循行为驱动设计模式的测试

+   使用容器将外部依赖项集成到测试中

你和项目经理、质量保证负责人、运维负责人以及 CEO 坐在会议室里。项目经理站在房间前面开始演示。

“它被称为 strangler 应用程序。‘strangler 应用程序’这个名字来源于绞杀榕树，它围绕宿主树生长，直到宿主树死亡。虽然听起来很悲伤，但我们最终希望淘汰我们的旧应用程序。我们觉得我们的应用程序已经经过了足够的测试，可以开始向一小部分客户推出。新服务将像今天一样依赖于旧服务，但随着时间的推移，一旦我们确信没有遗漏任何东西，我们可以逐渐淘汰旧服务。”

你环顾四周，注意到每个人都点头表示同意。

“这似乎减轻了一些风险，因为我们总可以在有问题时切换回旧服务，”质量保证负责人补充道。

“确实如此。我们已经建立了一个用于编写灵活软件的系统，该系统能够满足我们的需求。我们缺少的最后一部分是什么？”你的项目经理环顾四周补充道。

运维负责人插话说：“如果我们想关闭旧系统，我们需要一个包含所有翻译的数据库。”

质量保证团队中有人补充说：“围绕这一点进行额外的测试会有所帮助。我们可以自动化吗？”

几周前，你绝对想不到这一点，但现在你在自动化测试方面建立了一些信誉。你的团队正在接受这个新的开发流程，并且效果显著。

你走到白板前写下

+   将客户调用转换为数据库调用。

+   迁移旧数据。

+   建立满足功能要求的集成测试。

“看起来我们有一个计划。大家做得很好，”CTO 站起来走出房间时说。这是你开始工作的信号。

## 9.1 逐步淘汰旧系统

Strangler 应用程序擅长逐步将旧代码转换为新代码。当我们回到第六章创建外部客户端时，我们就已经开始创建旧系统与新系统之间的链接了。如果你还记得，如果没有缓存中的值，我们会调用外部系统。我们的接口看起来是这样的：

```
type Translator interface {
    Translate(word string, language string) string
}
```

到目前为止，我们已经建立了一个灵活性的框架。这使我们能够通过使用配置管理和依赖注入来改变新系统与旧系统交互的方式，逐步淘汰旧应用程序。首先，我们需要选择一个数据库来存储我们的数据。一旦我们确信一切按预期工作，我们将移除外部客户端，并希望关闭旧系统。

为了管理这一点，我们将重载配置以包含数据库连接。如果我们看到这个连接，我们将覆盖外部客户端。我们需要向配置中添加一些值（见以下列表）。

列表 9.1 `core.go`

```
type Configuration struct {
    Port            string `json:"port"`
    DefaultLanguage string `json:"default_language"`
    LegacyEndpoint  string `json:"legacy_endpoint"`
    DatabaseURL     string `json:"database_url"`      ❶
    DatabasePort    string `json:"database_port"`     ❷
}
```

❶ 添加数据库 URL 以进行连接

❷ 在标准端口未使用的情况下添加端口

为了创建所有这些，我们应该考虑如何验证我们的更改是否有效，因此我们将编写一些集成测试来测试整个系统。到目前为止，我们主要关注基本的单元测试并模拟了外部集成。除了模拟这些集成之外，我们还应该创建一系列测试来验证这些交互。最常见的集成点通常是在应用程序和数据库之间。现在我们希望将我们的系统迁移到连接数据库而不是调用外部客户端，但我们希望有灵活性来开启和关闭客户端调用。

首先，我们将专注于创建一个新的连接，该连接与我们的现有接口相匹配。然后，根据我们的配置，我们将进行外部服务调用、数据库调用或混合调用，仅在数据库中不存在该值时才进行外部调用。在我们做任何事情之前，我们应该编写一些测试来验证现有功能，然后集成数据库并验证其是否正常工作。这些测试将验证用户体验而不是代码模块的整体功能，因此我们将采取与之前略有不同的方法。

## 9.2 行为驱动设计

在第三章中，我们讨论了测试驱动开发（TDD），这有助于我们关注代码单元的预期功能。这意味着我们可以专注于提供适当的输入以获得预期的输出。我们花费时间通过模拟系统部分来验证它们是否被调用，这需要我们一点技术知识来理解一切应该如何工作。我们可以采用相同的格式并进一步抽象化。想象一下，如果你的项目经理、CEO 甚至你的客户编写了测试，而你编写了实现。这正是*行为驱动设计*应该做的事情。我们从宏观层面开始审视事物，然后观察产品或功能将如何被使用的更大图景。我们不再专注于安排测试、对函数进行操作和断言值（记住第三章中的三个 A），而是专注于一个 Given、When、Then 结构，这可以用更清晰的文本编写并对其进行测试。以下列表提供了一个我们可以用于我们应用程序的示例。

列表 9.2 `app.feature`

```
Feature: Translation Service             ❶
  Users should be able to submit a word to translate words within the application

  Scenario: Translation                  ❷
    Given the word "hello"               ❸
    When I translate it to "german"
    Then the response should be "Hallo"
```

❶ 功能是可交付的项目。

❷ 场景是功能的使用方式。

❸ Given、When、Then 描述了发生了什么。

这种领域特定语言（DSL）被称为 Gherkin，用于允许非技术人员编写可以自动转换为测试的要求。一旦进入你的测试框架，这些要求就变成了你的验证标准或 Arrange, Act, Assert 模式中的断言。关于这一点很棒的是，你正在验证的是其他人编写的要求，或者可以称为你开发过程的一部分。项目经理再也不能说你没有满足要求，如果他们编写了描述并且所有测试都通过了！

注意：我们专注于使用 Go 运行器运行我们的 BDD 测试；然而，你可以使用 Selenium 或 Cypress 编写更全面的集成测试套件。

Gherkin 语言的特别之处在于多个库都可以使用它。这些功能中的每一个都可以与特定的单元测试或用于测试用户界面相关联。关键是我们的项目经理可以用这种特殊语言为我们编写代码，然后我们可以通过多种方式使用它来验证一个功能是否完整。例如，假设我们正在使用我们的 API 创建一个 UI。我们可以使用这个相同的功能文件来编写我们的 Go 后端测试、JavaScript UI 测试和自动化的端到端 QA 测试。只要功能存在但测试未实现，我们的构建就会失败。这是故意的，因为我们的系统需求已经发生了变化。

针对这个功能请求，我们可以将其集成到我们的测试流程中；但是，我们将不会测试单个包，而是测试 `main` 包，它运行整个应用程序。从高层次来看，我们可以验证我们是否满足了用户的需求。我们将使用这个功能定义来驱动我们的测试。为此，我们将使用一个名为 Godog 的库，它属于 Cucumber 项目，这是 BDD 的顶级开源项目。Cucumber 为支持 Gherkin 的其他语言编写了其他库。

## 9.3 在 Go 中编写 BDD 测试

首先，我们将通过编写我们的功能定义和测试来设置我们的 BDD 测试。我们将设置测试并编写它们，但预期它们会失败。一旦编写完成，我们将通过附加我们的数据库来修复它们。到本章结束时，我们将能够验证我们的服务完全符合预期。

我们需要做的第一件事是安装一个名为 Godog 的新工具。为此，让我们首先在我们的 Makefile 中添加一个条目（见以下列表）。

列表 9.3 Makefile

```
setup: install-go init-go install-lint install-godog
...
install-godog:
    go install github.com/cucumber/godog/cmd/godog@latest    ❶
```

❶ 安装 Godog 二进制文件

接下来，运行安装程序以验证其是否正常工作。我们应该能够将我们的功能复制到目录中以进行测试。Godog 有许多不同的方式来运行测试，但到目前为止，我们将依赖于默认行为，即在一个名为 `features` 的本地目录中查找功能文件。由于我们将测试 API 二进制文件，我们将在 `cmd` 目录中创建该目录。

一旦你创建了`cmd/features`目录并复制了我们的`app .feature`文件，我们就可以导航到`cmd`目录并输入`godog run`。你应该会看到如下列表中所示的生成代码片段。

列表 9.4 `console`

```
func iTranslateItTo(arg1 string) error {                               ❶
        return godog.ErrPending
}

func theResponseShouldBe(arg1 string) error {
        return godog.ErrPending                                        ❷
}

func theWord(arg1 string) error {
        return godog.ErrPending
}

func InitializeScenario(ctx *godog.ScenarioContext) {                  ❸
        ctx.Step(`^I translate it to "([^"]*)"$`, iTranslateItTo)      ❹
        ctx.Step(`^the response should be "([^"]*)"$`, theResponseShouldBe)
        ctx.Step(`^the word "([^"]*)"$`, theWord)
}
```

❶ 我们将文本转换为具有相似名称的函数，以捕获特定的输入。

❷ 在实现之前，我们可以使用这个特殊的错误类型。

❸ 测试进入这里来设置和运行每个场景步骤。

❹ 每个步骤都有一个特殊的捕获组，它为适当的函数提供输入。

显然，这段代码是不完整的，但它为我们提供了一个开始的基础。复制文本，并在该目录中创建一个名为`main_test.go`的新文件。我们还将创建一个结构体来帮助我们捕获测试所需的某些输入。代码如下所示。

列表 9.5 `main_test.go`

```
package main                                                          ❶

import (
    "github.com/cucumber/godog"                                       ❷
)

type apiFeature struct {}                                             ❸

func (api *apiFeature) iTranslateItTo(arg1 string) error {
    return godog.ErrPending
}

func (api *apiFeature) theResponseShouldBe(arg1 string) error {
    return godog.ErrPending
}

func (api *apiFeature) theWord(arg1 string) error {
    return godog.ErrPending
}

func InitializeScenario(ctx *godog.ScenarioContext) {
    api := &apiFeature{}

    ctx.Step(`^I translate it to "([^"]*)"$`, api.iTranslateItTo)     ❹
    ctx.Step(`^the response should be "([^"]*)"$`, api.theResponseShouldBe)
    ctx.Step(`^the word "([^"]*)"$`, api.theWord)
}
```

❶ 我们使用主包，这样我们就可以引用其中的方法来启动应用程序。

❷ 使用 Godog 库来帮助设置测试。

❸ 这个结构体将帮助在整个测试过程中存储信息。

❹ 函数现在处于我们的功能结构体的上下文中。

我们的测试已经设置好了，但我们无法访问和运行主函数，因此我们想要创建与`main()`相同的方式来启动服务器。这通常是通过创建一个包含应用程序创建逻辑的函数，并在`main`函数中调用它来完成的。我们将重构`main.go`文件以匹配此模式（见以下列表）。

列表 9.6 `main.go`

```
package main

import (
    "log"
    "net/http"

    "github.com/holmes89/hello-api/config"
    "github.com/holmes89/hello-api/handlers"
    "github.com/holmes89/hello-api/handlers/rest"
    "github.com/holmes89/hello-api/translation"
)

func main() {

    cfg := config.LoadConfiguration()
    addr := cfg.Port

    mux := API(cfg)                                        ❶

    log.Printf("listening on %s\n", addr)

    log.Fatal(http.ListenAndServe(addr, mux))
}

func API(cfg config.Configuration) *http.ServeMux {        ❷

    mux := http.NewServeMux()

    var translationService rest.Translator
    translationService = translation.NewStaticService()
    if cfg.LegacyEndpoint != "" {
        log.Printf("creating external translation client: %s", cfg.LegacyEndpoint)
        client := translation.NewHelloClient(cfg.LegacyEndpoint)
        translationService = translation.NewRemoteService(client)
    }

    translateHandler := rest.NewTranslateHandler(translationService)

    mux.HandleFunc("/translate/hello", translateHandler.TranslateHandler)
    mux.HandleFunc("/health", handlers.HealthCheck)

    return mux                                            ❸
}
```

❶ 主函数现在只是运行服务器，而不是配置服务的 HTTP 和端点。

❷ 这个函数将组装服务及其 HTTP 端点，以便传递给服务器。

❸ 返回 mux 路由器以附加到 HTTP 服务器。

你应该能够启动你的应用程序，并且它仍然可以正常工作。现在我们可以连接我们的 Godog 测试。为了验证我们的结果，我们将调用我们的 API 并解析结果。虽然 Go 有调用 HTTP 端点的功能，但我们将使用库来帮助我们使代码更容易阅读。为此，安装`go` `get` `github.com/go-resty/resty/v2`。Resty 帮助使编写 API 调用更加清晰。例如，如果我们想使用 Resty 调用我们的 API，它将类似于以下列表。

列表 9.7 Resty 示例

```
resp, err := resty.New().R().                             ❶
        SetHeader("Content-Type", "application/json").    ❷
        SetQueryParams(map[string]string{                 ❸
            "language": "german",
        }).
        Get("http://localhost:8080/translate/hello")      ❹
```

❶ 创建一个新的请求

❷ 设置头部为 JSON

❸ 设置查询参数为德语

❹ 使用 GET 调用端点

我们需要这个调用的输入来验证我们的测试。记得我们之前查看 Godog 为我们生成的那些方法吗？它们有字符串输入，我们可以在功能测试结构体中设置这些输入。让我们将以下列表中的代码添加到我们的结构体中。

列表 9.8 `main_test.go`

```
type apiFeature struct {
    client   *resty.Client          ❶
    server   *httptest.Server       ❷
    word     string                 ❸
    language string                 ❹
}
```

❶ 测试的共享客户端

❷ 创建一个测试服务器以避免端口冲突

❸ 正在使用的单词

❹ 正在翻译到的语言

现在我们可以将值存储在各个步骤中（见以下列表）。

列表 9.9 `main_test.go`

```
func (api *apiFeature) iTranslateItTo(arg1 string) error {
    api.language = arg1     ❶
    return nil
}

func (api *apiFeature) theWord(arg1 string) error {
    api.word = arg1
    return nil
}
```

❶ 将值保存到结构体中

让我们使用下面的列表中的代码初始化我们的功能结构体，以便每个场景都有一个服务器启动和关闭。

列表 9.10 `main_test.go`

```
func InitializeScenario(ctx *godog.ScenarioContext) {

    client := resty.New()                    ❶
    api := &apiFeature{                      ❷
        client: client,
    }

    ctx.Before(func(ctx context.Context, sc *godog.Scenario) 
      (context.Context, error) {             ❸
        cfg := config.Configuration{}
        cfg.LoadFromEnv()                    ❹

        mux := API(cfg)                      ❺
        server := httptest.NewServer(mux)    ❻

        api.server = server
        return ctx, nil
    })

    ctx.After(func(ctx context.Context, sc *godog.Scenario, err error) 
      (context.Context, error) {
        api.server.Close()                   ❼
        return ctx, nil
    })

    ctx.Step(`^I translate it to "([^"]*)"$`, api.iTranslateItTo)
    ctx.Step(`^the response should be "([^"]*)"$`, api.theResponseShouldBe)
    ctx.Step(`^the word "([^"]*)"$`, api.theWord)
}
```

❶ 创建共享客户端

❷ 创建一个新的用于共享的功能结构体

❸ 使用前后钩子来管理服务器

❹ 从环境变量中加载配置（也可以使用默认值）

❺ 创建与主函数相同的 mux

❻ 创建测试服务器

❼ 在场景结束后关闭服务器

最后，我们可以测试这个调用。我们将在`theResponseShouldBe`函数中这样做。在其中，我们组装 API 调用并验证结果，如下面的列表所示。

列表 9.11 `main_test.go`

```
func (api *apiFeature) theResponseShouldBe(arg1 string) error {
    url := fmt.Sprintf("%s/translate/%s", api.server.URL, api.word)    ❶

    resp, err := api.client.R().
        SetHeader("Content-Type", "application/json").
        SetQueryParams(map[string]string{
            "language": api.language,                                  ❷
        }).
        SetResult(&rest.Resp{}).                                       ❸
        Get(url)

    if err != nil {
        return err
    }

    res := resp.Result().(*rest.Resp)
    if res.Translation != arg1 {                                       ❹
        return fmt.Errorf("translation should be set to %s", arg1)
    }

    return nil
}
```

❶ 根据单词创建要调用的 URL

❷ 设置翻译的语言

❸ 将结果捕获到已知的结构体中

❹ 验证单词

我们就到这里吧！再次输入`godog run`，看看结果如何。现在如果你想要的话，可以创建另一个语言的场景！它工作了吗？接下来，让我们添加数据库的要求。

## 9.4 添加数据库

首先，我们将添加一个新的要求，这个要求需要我们离开我们的静态数据集，转而使用外部数据库。这组相同的测试原本也可以用于连接我们的外部服务。想象一下，如果 QA 团队最初是在旧系统上编写所有这些要求，然后你随着你的 strangler 应用开始，将他们移动到新项目中。当你所有的测试都通过时，你就知道你已经达到了平衡。一旦我们将配置翻转以使用数据库，我们就可以再次验证一切是否都已准备好部署（见下面的列表）。

列表 9.12 `app.feature`

```
Feature: Translation Service
  Users should be able to submit a word to translate words within the application

  Scenario: Translation
    Given the word "hello"
    When I translate it to "german"
    Then the response should be "Hallo"

  Scenario: Translation
    Given the word "hello"
    When I translate it to "bulgarian"
    Then the response should be "Здравейте"
```

如果我们现在运行我们的测试，我们应该看到失败。回想一下第三章，并记住我们的失败、通过、失败的模式。这意味着我们的项目经理或其他人可以在我们开发功能时监控我们的进度。可以生成报告来显示特定功能中所有场景的覆盖率和完成进度，并可以被称为*功能完成*。

还有一点非常重要，这个应用功能集可以在后端 API 和前端屏幕上测试。一个功能的完整性可以通过一系列集成测试来验证，而不仅仅是单一的一个测试！

对于我们的功能，我们可以想象一个更大的翻译集合需要我们验证。我们当前在代码中的 switch 语句中保留所有翻译的解决方案既不可扩展，也不允许我们在不重新启动服务的情况下添加或删除语言，因此我们将向我们的系统添加数据库。数据库是专门的数据存储应用程序，在管理和处理我们的各种数据方面做得更好。然后我们将测试我们的服务与外部依赖项（在这种情况下是数据库）之间的集成。

有许多数据库选项，但我们将使用一个非常简单（但强大）的键值存储，称为 Redis，它非常轻量级，并且将非常类似于我们之前实现的缓存机制。我们将把这个工作分解为开发和测试。在我们设置测试之前，我们需要有一种建立连接的方法。记住，我们只需要一个实现我们的 `Translator` 接口的服务，然后我们就可以将其直接放入现有的处理器中。

首先，让我们将 Redis 添加到我们的基础设施中。记得在第七章中我们引入了 `docker-compose` 来帮助我们构建容器吗？我们将使用相同的技术来管理我们的依赖项。让我们将 Redis 作为依赖项添加到我们的 `docker-compose.yml` 文件中，如下所示。

列表 9.13 `docker-compose.yml`

```
version: "3.8"
services:
  ...
  database:                   ❶
    image: redis:latest       ❷
    ports:
     - '6379:6379'            ❸
    volumes:
     - "./data/:/data/"       ❹
```

❶ 创建一个新的名为 database 的服务

❷ 使用最新的 Redis 容器定义

❸ 将 Redis 端口暴露给 API 使用

❹ 将数据库备份挂载用于测试

如果你运行 `docker-compose` `up` `-d database` 然后输入 `docker` `exec` `-it` `database redis-cli`，你应该会看到一个提示出现。如果你输入 `ping`，你应该得到一个 `pong` 响应。恭喜！你刚刚启动了一个数据库！

让我们创建一个连接。我们已经更新了我们的配置来处理数据库的连接字符串。现在我们将在 `translation` 包中创建一个新的文件。我们将称之为 `database.go`。我们首先创建一个返回连接结构体的函数。我们将使用这个结构体来实现 `Translator` 接口。我们将创建足够的代码以便开始编写我们的测试。让我们在以下列表中编写代码。

列表 9.14 `database.go`

```
package translation

import (
    "context"
    "fmt"

    "github.com/go-redis/redis/v9"
    "github.com/holmes89/hello-api/config"
    "github.com/holmes89/hello-api/handlers/rest"
)

var _ rest.Translator = &Database{}                                    ❶

type Database struct {
    conn *redis.Client
}

func NewDatabaseService(cfg config.Configuration) *Database {          ❷
    rdb := redis.NewClient(&redis.Options{
        Addr:     fmt.Sprintf("%s:%s", cfg.DatabaseURL, cfg.DatabasePort),
        Password: "", // no password set
        DB:       0,  // use default DB
    })
    return &Database{
        conn: rdb,
    }
}

func (s *Database) Close() error {                                     ❸
    return s.conn.Close()
}

func (s *Database) Translate(word string, language string) string {
    return ""                                                          ❹
}
```

❶ 这是一个类型验证，这样我们知道我们的服务满足接口。

❷ 使用数据库配置返回一个新的连接结构体

❸ 需要一个关闭函数来清理连接。

❹ 只做最少的努力以开始。

现在我们可以创建我们的集成测试了。多亏了我们的运维团队，我们能够获取到生产数据库的备份，因此作为我们测试的一部分，我们将把备份的数据库加载到 Docker 容器中，并针对它运行我们的服务。这将尽可能地模拟生产环境进行测试。让我们为我们的测试套件创建设置（见以下列表）。

列表 9.15 `main_test.go`

```
var (
    pool     *dockertest.Pool
    database *dockertest.Resource
)

func InitializeTestSuite(sc *godog.TestSuiteContext) {                 ❶

    var err error

    sc.BeforeSuite(func() {
        pool, err = dockertest.NewPool("")                             ❷
        if err != nil {
            panic(fmt.Sprintf("unable to create connection pool %s", err))
        }

        wd, err := os.Getwd()
        if err != nil {
            panic(fmt.Sprintf("unable to get working directory %s", err))
        }

        mount := fmt.Sprintf("%s/data/:/data/", filepath.Dir(wd))      ❸

        redis, err := pool.RunWithOptions(&dockertest.RunOptions{      ❹
            Repository: "redis",
            Mounts:     []string{mount},
        })
        if err != nil {
            panic(fmt.Sprintf("unable to create container: %s", err))
        }
        if err := redis.Expire(600); err != nil {
            panic("unable to set expiration on container")
        } //Destroy container if it takes too long
        database = redis
    })

    sc.AfterSuite(func() {
        database.Close()                                               ❺
    })
}
```

❶ 这将在每个测试套件之前运行，这与在每个场景上运行的设置不同。

❷ 创建一个新的 Docker 连接池

❸ 将数据库备份挂载

❹ 运行 Docker 容器

❺ 完成后关闭容器

现在我们还需要更新我们的场景设置（见以下列表）。

列表 9.16 `main_test.go`

```
func InitializeScenario(ctx *godog.ScenarioContext) {

...
    ctx.Before(func(ctx context.Context, sc *godog.Scenario) (context.Context, error) {
        cfg := config.Configuration{}
        cfg.LoadFromEnv()

                cfg.DatabaseURL = "localhost"               ❶
                cfg.DatabasePort = database.Port("6379")    ❷

        mux := API(cfg)
        server := httptest.NewServer(mux)

        api.server = server
        return ctx, nil
    })
...
}
```

❶ 将数据库设置为连接到您机器上的 Docker

❷ Docker 库随机创建一个端口进行连接。

最后，我们编辑我们的主文件以使用我们刚刚启动的数据库的新 URL（见以下代码列表）。

列表 9.17 `main.go`

```
if cfg.DatabaseURL != "" {                            ❶
        db := translation.NewDatabaseService(cfg)
        translationService = db
    }
```

❶ 如果数据库已设置，我们通过依赖注入使用这个服务。

现在我们可以运行我们的测试，并看到它们失败。太好了！让我们更新我们的实现代码来检索文件。根据文档，翻译的值存储在`language:word`的格式中，其中所有`word`变量都是英文，所以我们的逻辑变得相当简单，如下所示。

列表 9.18 `database.go`

```
func (s *Database) Translate(word string, language string) string {
    out := s.conn.Get(context.Background(), fmt.Sprintf("%s:%s", 
      word, language))         ❶
    return out.Val()           ❷
}
```

❶ 通过从单词和语言构建键来查询数据库

❷ 返回字符串值

再次运行我们的测试，你应该会看到它们通过！

备注：希望你们中的一些人在想为什么我们没有创建专门测试数据库的测试，而是依赖于集成测试。这是因为这超出了本章的范围，但它是一个很好的练习。你可以使用之前相同的数据库设置，或者查看一些其他内存数据库测试解决方案。

你能想到其他可以添加的测试吗？或者是我们可能没有涵盖的其他功能？

## 9.5 发布

让我们看看我们的测试金字塔（图 9.1）。

![](img/CH09_F01_Holmes4.png)

图 9.1 端到端测试在顶部较小，因为它们成本更高且可靠性不高。它们应该由更大的集成和单元测试套件支持。每一层都应该独立运行，从单元测试开始，然后在不同阶段向上推进金字塔。

我们已经涵盖了金字塔的所有层级，从第三章的单元测试开始，第六章的验收测试，以及现在本章的集成测试。这意味着我们完成了吗？不，还远远没有。这是你和你团队需要开始监控覆盖率以及测试所需时间的时候。理想情况下，你不想让你的测试套件运行超过 5-10 分钟，这样它们才是有效的。集成测试可以根据速度分成不同的组。例如，测试子集可以针对核心功能运行，而较长的套件可以用于所有回归（旧问题）。这个测试组通常被称为*功能测试*，或验证应用程序规格的测试。表 9.1 给出了这些不同类型的简要概述。

表 9.1 功能测试类型

| 类型 | 描述 | 回答问题 |
| --- | --- | --- |
| 烟雾测试 | 检查基本功能的初步测试 | 它能启动吗？ |
| 精神测试 | 验证高级计算，如聚合或数学计算 | 项目数量正确吗？ |
| 回归测试 | 验证之前报告的缺陷是否已解决 | 这之前能工作吗？ |
| 可用性测试 | 评估客户与产品的互动 | 人们如何使用这个功能？ |

在我们的链中何时运行这些测试是合适的呢？在上一章中，我们讨论了发布：我们希望在代码稳定时才进行发布，因此我们希望使用这些测试作为一种方式来确认一切都已经稳定到可以发布。从理论上讲，我们所有的单元测试和验收测试都应该支持我们的集成测试，所以当我们打标签或发布时，不应该有任何意外。然而，我们还想增加一个最后的防护措施来防止发布有缺陷的代码，因此我们将在构建过程中添加一个集成测试阶段（见下面的代码列表），这个阶段将在发布完成但推送到生产之前发生。

列表 9.19 `ci.yaml`

```
smoke-test:
        name: Smoke Test Application
        needs:
        - test
        runs-on: ubuntu-latest
        steps:
        - name: Set up Go 1.x
        uses: actions/setup-go@v2
        with:
            go-version: ¹.18
        - name: Check out code into the Go module directory
        uses: actions/checkout@v2
        - name: Install Godog
        run: go install github.com/cucumber/godog/cmd/godog@latest    ❶
        - name: Run Smoke Tests
        run: |
            go get ./...
            godog run --tags=smoke-test
  build:
    name: Build App
    runs-on: ubuntu-latest #
    needs: smoke-test                                                 ❷
    steps:
    ...
  containerize-buildpack:
    name: Build Container buildpack
    runs-on: ubuntu-latest #
    needs: smoke-test                                                 ❸
```

❶ 安装 Godog 并运行你的烟雾测试

❷ 仅在烟雾测试成功后构建应用程序

❸ 仅在烟雾测试成功后构建容器

我们决定在单元测试之后放置功能测试。这允许我们在开始构建之前提升测试树。我们构建集成测试的方式意味着我们可以在系统启动时获得洞察。在手工焊接电路板的时代，这被称为*烟雾测试*，因为如果插入时冒烟，就存在问题。今天的烟雾测试验证系统是否启动并具有基本功能。因此，我们的集成测试也可以是我们的烟雾测试。

通常，标签会被赋予某些测试以表示它们是否是烟雾测试的一部分或更大的*回归测试*套件的一部分。这些可以对应于功能测试类型。我们可能在烟雾测试运行后运行额外的检查。烟雾测试失败将停止管道，但回归套件可能是一个标志，提示某人验证是否有所变化。这变成了一种自动化的质量保证系统，可以允许质量保证成员花大部分时间探索系统以找到额外的错误。在下一个列表中，让我们调整我们的场景以使用标签，并更新 CI 的最后一个标志以进行烟雾测试。

列表 9.20 `api.feature`

```
Feature: Translate API
  Users should be able to submit a word to translate

  @smoke-test
  Scenario: Translation
    Given the word "hello"
    When I translate it to "german"
    Then the response should be "Hallo"
  @smoke-test
  Scenario: Translation unknown
    Given the word "goodbye"
    When I translate it to "german"
    Then the response should be ""
  @smoke-test
  Scenario: Translation Bulgarian
    Given the word "hello"
    When I translate it to "bulgarian"
    Then the response should be "Здравейте"
  @regression-test                      ❶
  Scenario: Translation Czech
    Given the word "hello"
    When I translate it to "Czech"
    Then the response should be "Ahoj"
```

❶ 运行一个单独的测试类型

编辑我们的 CI 以首先运行烟雾测试，然后进行第二个步骤以测试回归（见下面的列表）。

列表 9.21 `ci.yaml`

```
smoke-test:
...
        run: |
            go get ./...
            godog run --tags=smoke-test
    regression-test:                              ❶
        name: Regression Test Application
        needs:
        - test
        runs-on: ubuntu-latest
        steps:
        - name: Set up Go 1.x
        uses: actions/setup-go@v2
        with:
            go-version: ¹.18
        - name: Check out code into the Go module directory
        uses: actions/checkout@v2
        - name: Install Godog
        run: go install github.com/cucumber/godog/cmd/godog@latest
        - name: Run Smoke Tests
        run: |
            go get ./...
            godog run --tags=regression-test      ❷
```

❶ 建立一个更长时间或更全面的测试套件，定期运行

❷ 指定回归套件

当你看到你的管道变绿时，质量保证经理走过。现在是展示你所能完成的事情的大好时机。你展示了不同的功能测试，并解释了我们如何将他们想要的全部要求放入各种测试套件中。当他们看到绿色的场景时，他们笑了，这是你第一次看到他们这样做。

## 摘要

+   使用行为驱动开发有助于整个团队建立需求。

+   Gherkin 提供了一种通用的语言来编写行为驱动测试，这些测试可以被不同的团队实现。

+   集成测试的外部依赖可以通过使用容器来复制现实世界的服务来提供。

+   标签可用于帮助聚焦测试套件并缩短测试的整体运行时间。
