# 附录 B：使用 Python

在这个附录中，我们将使用 Python 3.8 和已安装的`pip`使用 Python 特定的工具构建一个 API 和管道。请确保你已经安装了它（[`www.python.org/downloads/`](https://www.python.org/downloads/)）。

## B.1 Poetry

在我们构建项目之前，我们需要创建一个可重复的工作环境。Python 遵循在多个项目之间共享库的常见做法。像 C、Java 甚至 Go 这样的语言使用一个中央库仓库，这些库被下载并存储在您的机器上。这个过程的问题在于，如果您没有跟踪版本，下一个设置开发环境的人可能会有不同版本的库，这可能会引入新的问题或破坏当前的功能。因此，这些语言中的大多数都存储库的版本，以便环境可以几乎完全相同地设置。

使用 Go，我们有一个`go.mod`文件，它将允许 Go 重新下载给定模块的所有依赖项。Python 为我们提供了一个非常基本的选项，称为虚拟环境，它允许开发者像在没有任何额外库的新安装的 Python 上运行一样操作。任何后续的库都是通过一个名为`pip`的工具安装的。安装完成后，你可以创建一个需求文档。这个文档是通过`pip`的`freeze`命令创建的，该命令输出所有已安装的库。如果我们想在应用程序的依赖项中更明确地指定某些内容，那么我们需要使用一个名为 Poetry 的单独工具来处理库、安装和构建。

首先，我们需要通过运行以下命令来安装 Poetry：

```
curl -sSL https://install.python-poetry.org | python3 -
```

然后我们通过输入以下命令来创建一个新的项目：

```
poetry new hello-api
```

简单到这个程度！现在让我们编写我们的 API。

## B.2 编码

我们将使用一系列工具来托管我们的 API。创建处理器的框架称为 FastAPI，这是一个相对较新的路由库，运行速度快。它将在`uvicorn`服务器上运行。和之前一样，我们的后端需要一个 Redis 数据库，因此我们还需要这些依赖项。要将这些库添加到我们的项目中，我们只需输入以下命令：

```
poetry add fastapi \
uvicorn redis
```

库将被安装，并且你的`pyproject.toml`文件将更新以包含这些依赖项。现在我们可以创建我们的应用程序。在`hello_api`目录中创建一个名为`app.py`的新文件，并添加以下列表中的代码。

列表 B.1 `app.py`

```
from fastapi import FastAPI, Depends
import hello_api.deps as deps
from hello_api.repo import RepositoryInterface

app = FastAPI()                ❶

repo = deps.redis_client       ❷

@app.get("/translate/{word}")
def translation(
    word: str, language: str = "english", repo: RepositoryInterface =
    ➥Depends(repo)
):                             ❸
    resp = repo.translate(language, word)
    return {"language": language.lower(), "translation": resp}
```

❶ 创建一个基本处理器

❷ 加载 redis 客户端

❸ 将 Redis 客户端注入到处理器中

这个处理器应该看起来很熟悉。在这里，我们创建了一个应用程序、一组依赖项和一个翻译单词的路由。我们的翻译函数需要一个接口和一个依赖项来满足这个接口，所以让我们创建这些。首先，我们在一个名为`repo.py`的文件中创建接口，代码如下所示。

列表 B.2 `repo.py`

```
class RepositoryInterface:                                  ❶
    def translate(self, language: str, word: str) -> str:
        """translates word into given language"""
        pass
```

❶ 建立一个可以鸭式类型化的接口

虽然我们称这为一个接口，但从技术上讲它并不是，因为 Python 没有显式的接口类型。这是一个利用 Python 的鸭子类型系统的简单方法，就像我们在 Go 中做的那样。这个接口更像是抽象类，其中方法不应该被实现，而只是被定义。我们只需要在我们的`redis.py`中实现接口，如下面的列表所示。

列表 B.3 `repo.py`

```
import redis
import os
from hello_api.repo import RepositoryInterface

class RedisRepository(RepositoryInterface):                     ❶

    host: str = os.environ.get("DB_HOST", "localhost")
    port: str = os.environ.get("DB_PORT", "6379")
    default_language: str = os.environ.get("DEFAULT_LANGUAGE", "english")

    def __init__(self, client=None) -> None:                    ❷
        if client is None:
            self.client = redis.Redis(host=self.host, port=self.port)
        else:
            self.client = client

    def translate(self, language: str, word: str) -> str:       ❸
        """translates word into given language"""
        lang = language.lower() if language is not None else
        ➥self.default_language
        key = f"{word.lower()}:{lang}"
        return self.client.get(key)
```

❶ 实现接口

❷ 实例化客户端或设置可选的客户端变量

❸ 满足接口

在这里，您可以看到我们扩展了接口并实现了其方法。我们可以创建我们的连接并处理请求。最后一步是通过创建一个名为`deps.py`的文件来利用 FastApi 的依赖注入工具，该文件将包含运行服务所需的依赖项的函数，如下面的列表所示。

列表 B.4 `deps.py`

```
from hello_api.repo import RepositoryInterface
from hello_api.redis import RedisRepository

def redis_client() -> RepositoryInterface:    ❶
    return RedisRepository()
```

❶ 用于 FastAPI 依赖注入的必需项

要运行您的应用程序，请输入

```
uvicorn hello_api.app:app
```

现在您应该可以开始业务了！现在让我们添加我们的检查。

## B.3 测试

我们将使用与在 Go 章节中不同的方法来测试我们的 Python 应用程序。我们不会使用 Redis 容器来模拟数据库连接，而是将使用一个内存中的 Redis 替代品，称为`redislite`。我们将使用依赖注入来替换实际连接，并且一切都应该相同。我们需要通过运行以下命令添加两个测试库，以便测试能够工作：

```
poetry add redislite \
requests
```

创建一个名为`tests`的目录，并添加下面的列表中的代码。

列表 B.5 `test_hello_api.py`

```
from hello_api.app import app, repo
from fastapi.testclient import TestClient
from redislite import Redis

from hello_api.repo import RepositoryInterface
from hello_api.redis import RedisRepository
import unittest

class AppIntegrationTest(unittest.TestCase):
    def redis_client(self) -> RepositoryInterface:            ❶
        self.fake_redis = Redis()
        self.fake_redis.set("hello:german", "Hallo")
        self.fake_redis.set("hello:english", "Hello")

        return RedisRepository(client=self.fake_redis)

    def setUp(self):                                          ❷
        self.repo = self.redis_client()
        self.client = TestClient(app)
        app.dependency_overrides[repo] = self.redis_client    ❸

    def test_english_translation(self):
        response = self.client.get("/translate/hello")
        assert response.status_code == 200
        assert response.json() == {"language": "english", "translation":
        ➥"Hello"}

    def test_german_translation(self):
        response = self.client.get("/translate/hello?language=GERMAN")
        assert response.status_code == 200
        assert response.json() == {"language": "german", "translation":
        ➥"Hallo"}
```

❶ 创建模拟 redis 客户端的内部函数

❷ 设置测试

❸ 使用内部函数覆盖依赖函数

现在我们应该能够验证我们的测试是否工作。为此，我们将使用一个工具来帮助我们保持测试和格式的标准化。

## B.4 Nox

Nox 是一个开源工具，允许您组织和标准化您的测试和代码检查脚本。要使用此工具，您必须全局安装 Nox，因此打开一个新的终端窗口，并输入以下命令：

```
pip install --user --upgrade nox
```

接下来，我们在项目的根目录下创建一个`noxfile.py`文件。然后，我们将下面的列表中的选项添加到文件中。

列表 B.6 `noxfile.py`

```
import nox

nox.options.sessions = "lint", "tests"                   ❶
locations = "hello_api", "tests", "noxfile.py"           ❷

@nox.session
def tests(session):
    session.run("poetry", "install", external=True)      ❸
    session.run("pytest")

@nox.session
def lint(session):
    args = session.posargs or locations
    session.install("flake8", "flake8-black")
    session.run("flake8", *args)
```

❶ 可以完成的作业

❷ 可以完成作业的文件

❸ 执行命令

就这样！请注意，Nox 使用`flake8`作为我们的代码检查工具。我们想要对其进行一些配置，为此，我们需要创建一个`.flake8`文件，并在下面的列表中添加代码。

列表 B.7 `\.flake8`

```
[flake8]
select = E123,W456      ❶
max-line-length = 88    ❷
```

❶ 定义代码检查规则

❷ 设置最大行宽

Nox 将为我们处理其余的工作。要执行代码检查和测试，我们需要运行以下命令：

```
nox -rs lint
nox -rs tests
```

您将看到脚本将依次通过这些阶段，并输出正确的结果。我们已经将代码检查和测试合并，这是在管道定义它应该运行的容器之前的最后步骤。

## B.5 定义容器

打包脚本语言与打包编译语言略有不同。在我们的 Go 和 Kotlin 示例中，我们构建了应用程序，并留下了一个可以分发和复制的文件。在 Python 和 JavaScript 等语言中，正确设置脚本运行的环境变得更为重要；否则，它们将在启动时或在请求过程中失败。这就是我们选择使用 Poetry 这样的打包和依赖管理器的原因。它将为我们管理所有这些。让我们定义我们的容器，如以下列表所示。

列表 B.8 Dockerfile

```
FROM python:3.10.7-slim-bullseye as base

ENV PYTHONFAULTHANDLER=1 \                                              ❶
    PYTHONHASHSEED=random \
    PYTHONUNBUFFERED=1

WORKDIR /app

FROM base as builder

ENV PIP_DEFAULT_TIMEOUT=100 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_NO_CACHE_DIR=1 \
    POETRY_VERSION=1.1.15

RUN pip install "poetry==$POETRY_VERSION"                               ❷
COPY pyproject.toml poetry.lock ./
RUN poetry config virtualenvs.create false \
  && poetry install
COPY hello_api hello_api
ENV PORT 8080
EXPOSE 8080
CMD ["uvicorn","hello_api.app:app","--port","8080","--host","0.0.0.0"]  ❸
```

❶ 为容器设置生产级别的环境变量

❷ 安装 poetry

❸ 运行服务器

我们已经有了 Dockerfile，可以继续到我们的管道。

## B.6 创建管道

此管道将进行代码检查、测试和构建容器。要开始，请在 `.github/workflows/pipeline.yml` 中创建一个新的工作流程文件，并添加以下代码。首先，我们将设置 Nox 来运行我们的代码检查器（见以下列表）。

列表 B.9 `pipeline.yml`

```
name: Python Checks

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
    - uses: actions/setup-python@v4
      with:
        python-version: '3.10'
    - run: pip install --user --upgrade nox    ❶
    - name: Run Check Lint
      run: nox -rs lint                        ❷
```

❶ 安装 Nox

❷ 运行 Nox 代码检查

然后，在下一个列表中，我们将使用 Nox 来运行我们的测试。

列表 B.10 `pipeline.yml`

```
name: Python Checks
...
jobs:
...
  test:
    name: Test
    runs-on: ubuntu-latest
    needs: format-check
    steps:
        - uses: actions/checkout@v3
    - uses: actions/setup-python@v4
      with:
        python-version: '3.10'
    - run: pip install --user --upgrade nox
    - name: Run Tests
      run: nox -rs tests    ❶
```

❶ 使用 Nox 运行测试

最后，我们将在以下列表中构建和分发我们的容器。

列表 B.11 `.pipeline.yml`

```
name: Python Checks
...
jobs:
...
  containerize:
    name: Build and Push Container
    runs-on: ubuntu-latest #
    needs: test
    steps:
    - uses: actions/checkout@v3
    - name: Build Container
      run: docker build -t gcr.io/${{ secrets.GCP_PROJECT_ID }}/hello-
      ➥api:python-latest .                                              ❶
    - name: Set up Cloud SDK
      uses: google-github-actions/setup-gcloud@main
      with:
        project_id: ${{ secrets.GCP_PROJECT_ID }}
        service_account_key: ${{ secrets.gcp_credentials }}
        export_default_credentials: true
    - name: Configure Docker
      run: gcloud auth configure-docker --quiet
    - name: Push Docker image
      run: docker push gcr.io/${{ secrets.GCP_PROJECT_ID }}/hello-
      ➥api:python-latest
    - name: Log in to the GHCR
      uses: docker/login-action@master
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    - name: Tag for Github
      run: docker image tag gcr.io/${{ secrets.GCP_PROJECT_ID }}/hello-
      ➥api:python-latest ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:
      ➥python-latest
    - name: Push Docker image to GCP
      run: docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:python-
      ➥latest
```

❶ 构建 Docker 镜像

由于 Python 生态系统很大，此时可能有多种创建和部署 Python 应用程序的方法。关键是可以通过工具和流程来保护代码并减少错误。这对于像 Python 这样的语言尤为重要，因为它们以类型安全和编译时错误为代价换取灵活性，从而允许快速开发。
