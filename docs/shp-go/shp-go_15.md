# 附录 A. 使用 Kotlin

在整本书中，我提到软件开发和代码维护模式并不局限于单一的语言或技术，因此以下章节将快速展示如何使用三种流行的语言（Kotlin、JavaScript 和 Python）引入这些相同的流程，并使用 HashiCorp 的工具提供另一种部署选项。首先，我们将探讨 Kotlin，这是一种建立在 JVM 之上的语言，随着其成熟度的提高而越来越受欢迎。

Kotlin 的好处是它建立在现有的 Java 技术之上，这意味着工具和模式相当成熟。我们将使用 Kotlin 特定的工具构建一个新的 CI 流水线，但在我们的 Makefile 中步骤将保持基本相同。首先，我们将使用 Kotlin 和 Redis 再次构建我们的 hello-api。

## A.1 框架

框架既帮助又阻碍开发团队。有时一个框架可以帮助你快速推出产品，但过了一段时间，你可能会发现你在与之抗争，失去了势头。花时间研究可用的框架并重新评估你正在使用的框架非常重要。寻找以下特性：

+   易用性

+   文档质量

+   可持续性

所有这三点都非常重要。你不想使用一个晦涩且不再受支持的框架，因为可能会出现错误和漏洞，而你将无法修复它们。同时，你也不想采用不支持你想要使用的技术框架。在 Java 开发世界中，一个新兴的框架 Quarkus 符合所有这些标准。幸运的是，这个框架可以与 Kotlin 一起使用，并且是为了在基于容器的生态系统中运行而构建的。由 Red Hat 开发和支持的 Quarkus 提供了许多驱动程序和开箱即用的工具，以帮助快速构建和测试 API。要设置它，请按照以下步骤操作：

+   安装 Java ([`mng.bz/e1mz`](http://mng.bz/e1mz))。

+   前往 code.quarkus.io。

+   将组命名为 `com.manning`，工件命名为 `hello-api`。

+   保持构建工具为 Maven，Java 版本 17。

+   在过滤器部分，添加 Kotlin、RESTEasy Reactive Kotlin Serialization 和 Redis 客户端。

+   点击生成您的应用程序，并打开您的项目。

注意：Spring 是 Java/Kotlin 框架中的大玩家，在 Java 和 Kotlin 中都有优秀的文档。

完成这些步骤后，我们可以解压我们的项目并开始工作。

## A.2 编码

Quarkus 给我们很多，我们不需要做任何事情。像许多现代基于 JVM 的框架一样，Quarkus 广泛使用注解来处理大部分的连接。这可以非常有帮助，但在调试时也可能是一个谜。我们首先在我们的 `com/manning/hello-api` 包中创建四个文件。首先是主函数，我们将称之为 `TranslationResource.kt`。这将是我们的翻译入口点，如下所示。

列表 A.1 `TranslationResource.kt`

```
package com.manning.hello-api

import javax.inject.Inject
import javax.ws.rs.GET
import javax.ws.rs.Path
import javax.ws.rs.PathParam
import javax.ws.rs.Produces
import javax.ws.rs.QueryParam
import javax.ws.rs.core.MediaType

@Path("/translate")                                    ❶
class TranslationResource {

    @Inject                                            ❷
    private lateinit var service: ITranslationService

    @GET                                               ❸
    @Path("/{word}")                                   ❹
    @Produces(MediaType.APPLICATION_JSON)
    fun translate(
        @PathParam("word") word: String,
        @QueryParam("language") language: String?
    ) = service.translate(language, word)              ❺
}
```

❶ 请求的基本路径

❷ 依赖注入翻译服务

❸ REST GET 方法

❹ 请求的子路径

❺ 根据路径和查询参数调用服务

当我们的应用程序启动时，Quarkus 将查找我们所有的路径并将它们添加到主控制器中，就像我们在 Go 中手动做的那样。这段代码没有特别之处，除了我们将一个接口作为我们的服务来处理翻译。这个接口将作为实际 Redis 实现背后的屏障，就像我们在 Go 中做的那样。这个接口在 `ITranslationService.kt` 中定义，如下所示。

列表 A.2 `ITranslationResource.kt`

```
package com.manning.hello-api

interface ITranslationService {
    fun translate(language: String?, word: String): Translation?    ❶
}
```

❶ 定义返回可选翻译对象的接口方法

很简单，对吧？它几乎和我们的 Go 接口完全一样。我们可选地返回一个 Translation 类型。这个类型在单独的 `Translation.kt` 文件中定义，它将复制我们的 Go 中的 Translation 结构（见以下列表）。

列表 A.3 `TranslationResource.kt`

```
package com.manning.hello-api

data class Translation(      ❶
    val language: String?,
    val translation: String?
)
```

❶ 为消息创建数据类或 DTO

最后，我们可以进入 Redis 连接。在这里，我们有一些更多的代码（见下一条列表）以及从 Redis 本身实际检索的内容。

列表 A.4 `RedisTranslationService.kt`

```
package com.manning.hello-api

import io.quarkus.redis.datasource.RedisDataSource
import org.eclipse.microprofile.config.inject.ConfigProperty
import javax.enterprise.context.ApplicationScoped
import javax.inject.Inject

@ApplicationScoped
class RedisTranslationService : ITranslationService {

    @Inject
    private lateinit var redisAPI: RedisDataSource        ❶

    @ConfigProperty(name = "default.language")
    var defaultLanguage: String? = "english"              ❷

    override fun translate(language: String?, word: String): Translation? {
        val commands = redisAPI?.string(String::class.java)
        val lang = language?.lowercase() ?: defaultLanguage
        val key = "$word:$lang"
        val translation = commands?.get(key)              ❸
        return if (translation == null) {
            null
        } else {
            Translation(language = lang, translation = translation)
        }
    }
}
```

❶ 注入 Redis 客户端

❷ 从配置中设置默认语言

❸ 从 Redis 获取翻译

为了让 Redis 连接，我们需要提供一些它可以用来自连接的属性。这些值可以被覆盖，就像在 Go 中一样，通过传递环境变量。默认情况下，我们希望使用 localhost，所以我们将填写我们的 `resources/application.properties` 文件，如下所示。

列表 A.5 `application.properties`

```
quarkus.redis.hosts=redis://localhost:6379     ❶
quarkus.redis.client-type=standalone
quarkus.datasource.jdbc=false

default.language=english
```

❶ 连接到 Redis 的默认属性

现在我们有了代码，让我们介绍一下如何构建和运行它。然后我们将用它来进行测试。最后，我们将所有这些封装在一个管道中。

## A.3 Maven

Maven 是 Apache 项目下的一个构建工具，它允许 Java 开发者管理他们的项目依赖（如我们的 `go.mod` 文件），以及整合不同的构建和测试脚本（如我们的 Makefile）。其他构建工具，如 Gradle，也可以用来完成类似任务。在本节中，我们将专注于使用 Maven。Maven 项目是通过使用一个 `pom.xml` 文件来管理的，你将在你下载的设置项目的根目录中找到它，还有一个 `mvnw` 脚本。这个脚本是一个围绕基本 Maven 工具的包装器，这样开发者就不需要担心环境特定的变量（例如，他们的操作系统和基本 Maven 文件）。

我们将使用 Maven 来构建、运行和测试我们的代码。首先，让我们运行我们编写的代码。确保你的 Redis 数据库正在运行，然后在终端窗口中输入 `./mvnw compile` `quarkus:dev`。你会看到许多文件被下载和编译，最后你会看到一个消息说服务器正在监听。在这个时候，尝试你之前章节中的可靠 `curl` 命令来测试它。

现在我们已经了解了 Maven 的样子，我们将用它来运行我们的单元测试。

## A.4 测试

对于我们的测试示例，我们将直接进入集成测试，因为它们让我们了解 Kotlin 中测试的编写方式，集成测试通常在设置和清理方面更为复杂。这将为我们提供一个很好的概述，同时提供一个起点来填充测试金字塔的其余部分。对于我们的测试，我们再次使用数据库容器和一个名为 RestAssured 的库，它将为我们提供一个用于验证的优秀的 REST 测试框架。要开始，在测试文件夹中创建一个名为 `TranslationTest.kt` 的文件（见以下列表）。

列表 A.6 `TranslationTest.kt`

```
package com.manning.hello-api

import io.quarkus.test.common.QuarkusTestResource
import io.quarkus.test.junit.QuarkusTest
import io.restassured.RestAssured.given
import org.hamcrest.CoreMatchers.equalTo
import org.junit.jupiter.api.Test

@QuarkusTestResource(RedisTestContainer::class)     ❶
@QuarkusTest                                        ❷
class TranslationTest {

    @Test
    fun testHelloEndpoint() {
        given()
            .`when`().get("/translate/hello")
            .then()
            .statusCode(200)
            .body("translation", equalTo("Hello"))
            .body("language", equalTo("english"))
    }

    @Test
    fun testHelloEndpointGerman() {
        given()                                     ❸
            .`when`().get("/translate/hello?language=GERMAN")
            .then()
            .statusCode(200)
            .body("translation", equalTo("Hallo"))
            .body("language", equalTo("german"))
    }
}
```

❶ 依赖于一个容器

❷ 告诉构建工具这是一个测试

❸ 使用流畅断言测试请求

测试应该与书中早些时候的内容相似。现在我们需要创建另一个文件，告诉我们的测试插件这个特定的测试是一个集成测试。为此，我们只需创建一个包含以下列表内容的文件。

列表 A.7 `TranslationTest.kt`

```
package com.manning.hello-api

import io.quarkus.test.junit.QuarkusIntegrationTest

@QuarkusIntegrationTest                   ❶
class TranslationIT : TranslationTest()
```

❶ 将此测试套件定义为集成测试

就这些了！最后，我们需要我们的数据库容器来运行测试。为此，我们在 `pom.xml` 文件中添加一个依赖项。在那里，在依赖项下，你将添加以下列表中的行。

列表 A.8 `pom.xml`

```
<dependency>
      <groupId>org.testcontainers</groupId>
      <artifactId>testcontainers</artifactId>
      <version>1.17.3</version>
      <scope>test</scope>
    </dependency>
```

然后我们创建一个名为 `RedisTestContainer.kt` 的文件，并添加以下列表中的代码。

列表 A.9 `RedisTestContainer.kt`

```
package com.manning.hello-api

import io.quarkus.test.common.QuarkusTestResourceLifecycleManager
import org.testcontainers.containers.BindMode
import org.testcontainers.containers.GenericContainer
import org.testcontainers.utility.DockerImageName

class RedisTestContainer : QuarkusTestResourceLifecycleManager {

    private val redisContainer = GenericContainer(DockerImageName.parse(
    ➥"redis:latest"))                                     ❶
        .withExposedPorts(6379)
        .withClasspathResourceMapping("data", "/data", BindMode.READ_ONLY)

    override fun start(): MutableMap<String, String> {     ❷
        println("STARTING redis ")
        redisContainer.start()
        println("redis://${redisContainer.getHost()}:${
        ➥redisContainer.getMappedPort(6379)}")
        return mutableMapOf(Pair("quarkus.redis.hosts", "redis://${
        ➥redisContainer.getHost()}:${
        ➥redisContainer.getMappedPort(6379)}"))
    }

    override fun stop() {
        println("STOPPING redis")
        redisContainer.stop()
    }
}
```

❶ 创建一个带有挂载数据目录的 Redis 容器

❷ 建立一个命令映射以检索消息

就这些了！要测试，运行 `./mvnw` `clean` `test`，你将看到你的测试成功运行！

## A.5 检查和初始管道

对于检查，我们使用一个名为 `ktlint` 的开源工具，它是由 Pinterest 作为开源项目编写和维护的。它支持可下载的独立应用程序以及 Maven 插件。在这里，我们将使用预构建的二进制文件，使安装更加简单直接。我们通常将检查作为管道的第一步，因为它是最简单且通常是运行最快的步骤。让我们创建我们的管道，直到构建和部署，我们将在下一节中完成。

在 `.github/workflows/pipeline.yml` 中创建一个新的工作流程文件，使用以下列表中的代码。

列表 A.10 `pipeline.kt`

```
name: Kotlin Checks

on:
  push:
    branches:
      - main
env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  format-check:
    name: Check formatting
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-java@v3           ❶
      with:
        distribution: 'temurin'
        java-version: '17'
        cache: 'maven'
    - name: Download Ktlint                 ❷
      run: curl -sSLO https://github.com/pinterest/ktlint/releases/
      ➥download/0.47.1/ktlint && chmod a+x ktlint
    - name: Lint
      run: ./ktlint
  test:
    name: Test Application
    needs:
      - format-check
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-java@v3
      with:
        distribution: 'temurin'
        java-version: '17'
        cache: 'maven'
    - name: Run Test
      run: ./mvnw clean test                ❸
```

❶ 定义 Java 版本

❷ 安装并运行 ktlint

❸ 删除所有旧文件并运行测试

如果你提交了更改并推送，你应该能看到测试运行并通过检查！

## A.6 容器化

最后一步是将这个应用程序放入容器中。为此，我们使用另一个巧妙的 Quarkus 技巧。近年来，已经开发出专门的编译器，允许 Java 代码编译成本地系统代码。这使得你的容器和包体积更小，运行速度更快。此外，你也不再需要过分关注 JVM 的安全补丁和更新；相反，你只需要担心操作系统的安全问题。Quarkus 已经将这项技术添加到 Quarkus 应用程序的构建系统中。Quarkus 甚至为你提供了 Maven 步骤和 Docker 容器来使用，因此我们可以立即将最后一个任务添加到我们的管道中，并验证它是否工作（见以下列表）。

列表 A.11 `pipeline.kt`

```
name: Kotlin Checks

jobs:
...
  containerize:
    name: Build and Push Container
    runs-on: ubuntu-latest #
    needs: test
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-java@v3
      with:
        distribution: 'temurin'
        java-version: '17'
        cache: 'maven'
    - name: Build
      run: ./mvnw package -Pnative -Dquarkus.native.container-build=true   ❶
    - name: Build Container
      run: docker build -f src/main/docker/Dockerfile.jvm -t
      ➥gcr.io/${{ secrets.GCP_PROJECT_ID }}/hello-api:kotlin-latest       ❷
    - name: Set up Cloud SDK
      uses: google-github-actions/setup-gcloud@main
      with:
        project_id: ${{ secrets.GCP_PROJECT_ID }}
        service_account_key: ${{ secrets.gcp_credentials }}
        export_default_credentials: true
    - name: Configure Docker
      run: gcloud auth configure-docker --quiet
    - name: Push Docker image
      run: docker push gcr.io/${{ secrets.GCP_PROJECT_ID }}/hello-api:
      ➥kotlin-latest
    - name: Log in to the GHCR
      uses: docker/login-action@master
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    - name: Tag for Github
      run: docker image tag gcr.io/${{ secrets.GCP_PROJECT_ID }}/hello-api:
      ➥kotlin-latest ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:
      ➥kotlin-latest
    - name: Push Docker image to GCP
      run: docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:
      ➥kotlin-latest                                                      ❸
```

❶ 构建本机 Java 二进制文件

❷ 使用内部 Dockerfile 构建容器

❸ 将容器推送到注册表

一旦你推送了最后一个更改，下载你的容器或将其部署到你的 Kubernetes 集群中，看看它是如何工作的！

Kotlin 和 Quarkus 在 Java 语言和框架的世界中都是相对较新的。这使得它们准备好应对当前软件开发中的挑战。虽然本章只是触及了表面，但我鼓励你深入挖掘并多做实验，因为 JVM 语言不会消失，你可能会在未来发现很多帮助迁移和改进 Java 应用程序的工作。
