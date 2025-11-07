# 附录 C. 使用 JavaScript

Node.js 是一个 JavaScript 运行时环境，允许开发者编写在网页浏览器之外运行的应用程序。这导致了 JavaScript 库的增长，帮助开发者编写后端服务。我们可以使用工具连接到数据库并创建 API。让我们开始吧。

## C.1 Node 包管理器

NPM，或 Node 包管理器，是我们用于 API 的核心构建和依赖管理工具。我们将使用 Express 库。首先，确保您已从 [`nodejs.org/en/download/`](https://nodejs.org/en/download/) 安装了 Node；然后安装 Express 生成器应用程序：

```
npm install express-generator -g
express hello-api
```

代码生成完成后，使用以下列表中的代码编辑 `package.json` 文件。

列表 C.1 `package.json`

```
{
    "name": "hello-api",                          ❶
    "version": "0.0.1",
    "description": "Simple api",
    "private": true,
    "jest": {                                     ❷
        "testEnvironment": "node"
    },
    "scripts": {                                  ❸
        "test": "jest --config jest.config.js",
        "start": "node ./bin/www",
        "format": "prettier --single-quote --write --use-tabs .",
        "check-format": "prettier --single-quote --use-tabs --check .",
        "lint": "eslint \"**/*.js\" --max-warnings 0 --ignore-pattern
        ➥node_modules/"
    },
    "author": "Joel Holmes",
    "license": "MIT",
    "dependencies": {                              ❹
        "debug": "⁴.3.4",
        "express": "⁴.18.1",
        "morgan": "¹.10.0",
        "redis": "⁴.3.0"
    },
    "devDependencies": {                           ❺
        "eslint": "⁸.23.0",
        "eslint-plugin-jest": "²⁷.0.1",
        "jest": "²⁹.0.0",
        "prettier": "².7.1",
        "supertest": "⁶.2.4",
        "testcontainers": "⁸.13.1"
    }
}
```

❶ 包的名称

❷ Jest 配置设置

❸ NPM 内置了执行机制，可以根据本地安装的包运行脚本。

❹ 运行应用程序所需的库

❺ 用于测试或开发应用程序的库

`package.json` 文件包含了测试、构建和运行我们的应用程序所需的全部依赖和脚本。在做出这些更改后，输入 `npm install` 以将这些依赖项添加到我们的项目中。我们已经设置了基本结构，现在可以开始编码了。

## C.2 编码

您会注意到 Express 为您生成了很多文件。主要的一个是 `app.js`。这是您进入 API 的入口路由。打开它，并用以下列表中的代码替换它。

列表 C.2 `app.js`

```
let express = require('express');                                 ❶
let logger = require('morgan');

let translateRouter = require('./routes/translation');            ❷
const { Repository } = require('./repository/translation');       ❸

class App {
    app = express();
    repo = undefined;
    constructor(host, port) {                                     ❹
        this.app.use(logger('dev'));
        this.app.use(express.json());
        this.app.use(express.urlencoded({ extended: false }));
        this.repo = new Repository(host, port);
        this.app.get('/translate/:word', translateRouter(this.repo));
    }

    async close() {
        return this.repo.close();
    }
}

module.exports = { App };                                         ❺
```

❶ 导入 Express 框架

❷ 导入翻译处理器

❸ 导入存储库

❹ 构建应用程序类

❺ 导出应用程序以供运行中的应用程序使用

希望您能注意到生成的代码与您所写的代码之间的差异。我们使用 JavaScript 的面向对象范式来帮助我们进行未来的单元测试。

我们需要定义一个路由和存储库。让我们首先定义路由。创建一个名为 `routes` 的目录。这个目录将存放我们的 `translation.js` 文件，也将是您想要定义的任何其他路由或路由组的存放位置。创建该文件，并添加以下列表中的代码。

列表 C.3 `translation.js`

```
const translation = (translationService) => {                            ❶
    return async (req, res) => {
        let language = req.query.language || 'english';                  ❷
        const resp = await translationService.translate(language,
        ➥ req.params.word);
        resp                                                             ❸
            ? res.json({ language: language.toLowerCase(), translation:
              ➥ resp })
            : res.status(404).send('Missing translation');
    };
};

module.exports = translation;
```

❶ 处理器的功能定义

❷ 如果未传递查询参数，则默认为英语。

❸ 如果提供了响应，则返回它；否则，返回 404。

如您所见，此函数需要将翻译服务传递给函数。这与我们在其他语言中使用依赖注入和更强的类型检查所做的方法不同。相反，我们传递函数以处理业务逻辑。或者，我们也可以创建一个类，就像我们在主应用中所做的那样。

注意：我们按照功能而不是按领域组织文件，这意味着我们将所有路由放在一个目录中，并将每个目录中的存储库方法放在每个目录中。另一种方法是有一个 `translation` 目录，其中包含 `repo.js` 和 `route.js` 文件。

现在我们需要定义翻译仓库。创建一个名为`repository`的目录和一个名为`translation.js`的文件，并添加以下列表中的代码。

列表 C.4 `translation.js`

```
const redis = require('redis');

class Repository {
    constructor(host, port) {
        this.host = host ? host : process.env.DB_HOST || 'localhost';
        this.port = port ? port : process.env.DB_PORT || '6379';
        this.defaultLanguage = process.env.DEFAULT_LANGUAGE || 'english';
        const connectionURL = `redis://${this.host}:${this.port}`;
        console.log(`connecting to ${connectionURL}`);
        this.client = redis.createClient({ url: connectionURL });
        this.client.on('connect', () => {                             ❶
            console.log('connected to redis');
        });
        this.client.on('error', (err) => console.log('client error',
        ➥ err));                                                     ❷
        this.client.connect();
    }

    async translate(language, word) {
        const lang = language                                         ❸
            ? language.toLowerCase()
            : this.defaultLanguage.toLowerCase();
        const key = `${word.toLowerCase()}:${lang}`;
        const val = await this.client.get(key);
        return val;
    }
    async close() {
        this.client.quit();
    }
}

module.exports = { Repository };
```

❶ 当数据库连接时添加日志记录

❷ 添加错误日志记录

❸ 检查语言；如果没有指定，则使用默认值

在这里，我们建立与 Redis 数据库的连接和一个用于检索翻译的功能。为了测试这个，打开一个终端窗口并输入`npm start`。我们有一个正在运行的 API；让我们来测试它。

## C.3 测试

我们将直接使用测试框架 Jest 和一个名为`testcontainers`的容器库进行集成测试，它将为我们提供 Redis 数据库。在我们编写测试之前，我们想要为我们的测试框架 Jest 创建一个配置文件。创建一个`jest.config.js`文件，并添加以下代码：

```
module.exports = {
    testTimeout: 30000,
};
```

这将给我们的容器提供启动时间，以便在测试之前。现在我们可以使用以下列表中的代码编写我们的集成测试。

列表 C.5 `app.test.js`

```
const { GenericContainer } = require('testcontainers');
const { App } = require('./app');
const request = require('supertest');

let container;
let app;
let api;

beforeAll(async () => {                                               ❶
    container = await new GenericContainer('redis')
        .withExposedPorts(6379)
        .withCopyFileToContainer('./data/dump.rdb', '/data/dump.rdb')
        .start();
    const port = container.getMappedPort(6379);
    const host = container.getHost();
    api = new App(host, port);                                        ❷
    app = api.app;
});

afterAll(async () => {
    await api.close();
    await container.stop();
});

describe('Translate', () => {
    test('hello translation in english to be hello', async () => {    ❸
        const response = await request(app).get('/translate/hello');
        expect(response.body).toEqual({
            translation: 'Hello',
            language: 'english',
        });
        expect(response.statusCode).toBe(200);
        return response;
    });
    test('hello translation in german to be hallo', async () => {
        const response = await request(app).get(
        ➥'/translate/hello?language=GERMAN');
        expect(response.body).toEqual({ translation: 'Hallo', language:
        ➥'german' });
        expect(response.statusCode).toBe(200);
        return response;
    });
});
```

❶ 通过启动容器来设置测试

❷ 将值传递给构建应用

❸ 编写一个测试来调用端点

现在运行`npm test`来查看测试是否通过。

## C.4 代码风格检查

我们下一步是检查格式和静态代码分析。我们将使用 Prettier 和 ESLint 来帮助我们完成这些步骤。在 package.json 代码中，我们添加了运行这些库进行格式化、检查格式和代码风格检查的脚本。要运行这些脚本，只需输入以下命令：

```
npm run format
npm run check-format
```

对于代码风格检查，我们需要为我们的代码风格检查器创建一个配置文件（见以下列表）。

列表 C.6 `\.eslintrc.json`

```
{
    "env": {                             ❶
        "node": true,
        "commonjs": true,
        "es2021": true
    },
    "extends": "eslint:recommended",
    "overrides": [                       ❷
        {
            "files": ["**/*.test.js"],
            "env": {
                "jest": true
            },
            "plugins": ["jest"],
            "rules": {
                "jest/no-disabled-tests": "warn",
                "jest/no-focused-tests": "error",
                "jest/no-identical-title": "error",
                "jest/prefer-to-have-length": "warn",
                "jest/valid-expect": "error"
            }
        }
    ],
    "parserOptions": {
        "ecmaVersion": "latest"
    },
    "rules": {                           ❸
        "indent": ["error", "tab"],
        "linebreak-style": ["error", "unix"],
        "quotes": ["error", "single"],
        "semi": ["error", "always"]
    }
}
```

❶ 定义环境

❷ 传递 Jest 格式化规则

❸ 定义额外的格式化规则

现在你可以通过输入以下命令来运行代码风格检查步骤

```
npm run lint
```

将这些步骤直接集成到我们的脚本中非常简单。这展示了拥有一个能够满足我们管道多个领域的单一工具的强大之处。在我们组装管道之前，定义容器是最后一步。

## C.5 定义容器

与附录 B 中我们创建的 Python 容器类似，JavaScript 需要设置一个环境以便脚本运行。NPM 作为基础镜像的一部分被安装，这样我们就可以运行依赖项的安装过程并运行主应用。以下列表显示了我们的容器定义。

列表 C.7 Dockerfile

```
FROM node:17                    ❶

# Create app directory
WORKDIR /usr/src/app

# Install app dependencies
COPY package*.json ./

RUN npm ci --only=production    ❷

# Bundle app source
COPY . ./
ENV PORT 8080
ENV NODE_ENV production
EXPOSE 8080
CMD [ "node", "./bin/www" ]     ❸
```

❶ 基础镜像

❷ 清理并安装生产所需的依赖包

❸ 使用 Express 提供的包装脚本运行

最后，我们可以构建我们的管道。

## C.6 构建管道

这个管道与其他章节中的管道类似，它将包含代码风格检查、测试和容器步骤，如下所示。

列表 C.8 `pipeline.yml`

```
name: JavaScript Checks

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
    - uses: actions/setup-node@v3
      with:
       node-version: 17
    - run: npm ci                ❶
    - name: Run Check Format
      run: npm run check-format  ❷
    - name: Run Check Lint
      run: npm run lint          ❸
```

❶ 清理并安装依赖

❷ 运行格式检查

❸ 运行代码风格检查

首先，在我们进行集成测试之前（见以下列表），我们将先进行简单的清理、安装和格式检查。

列表 C.9 `pipeline.yml`

```
name: JavaScript Checks
...
jobs:
...
  test:
    name: Test
    runs-on: ubuntu-latest
    needs: format-check
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-node@v3
      with:
       node-version: 17
    - run: npm ci
    - name: Test
      run: npm run test     ❶
```

❶ 使用 npm 脚本运行测试

当集成测试完成后，我们最终可以构建我们的容器以进行分发（参见下一列表）。

列表 C.10 `pipeline.yml`

```
name: JavaScript Checks
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
      run: docker build -t gcr.io/${{ secrets.GCP_PROJECT_ID }}/hello-api:
           ➥javascript-latest .         ❶
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
           ➥javascript-latest
    - name: Log in to the GHCR
      uses: docker/login-action@master
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    - name: Tag for Github
      run: docker image tag gcr.io/${{ secrets.GCP_PROJECT_ID }}/hello-api:
           ➥javascript-latest ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:
           ➥javascript-latest
    - name: Push Docker image to GCP
      run: docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:
           ➥javascript-latest
```

❶ 构建容器
