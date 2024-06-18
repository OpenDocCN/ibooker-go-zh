# 第七章：运行 gRPC 在生产中

在之前的章节中，我们专注于设计和开发基于 gRPC 的应用程序的各个方面。现在，是时候深入探讨如何在生产环境中运行 gRPC 应用程序的细节了。在本章中，我们将讨论如何为您的 gRPC 服务和客户端开发单元测试或集成测试，以及如何将它们与持续集成工具集成。然后，我们将进入 gRPC 应用程序的持续部署，探讨虚拟机（VM）、Docker 和 Kubernetes 上的一些部署模式。最后，在生产环境中操作您的 gRPC 应用程序时，您需要一个稳固的可观察性平台。这就是我们将讨论不同的 gRPC 应用程序可观察性工具，并探索 gRPC 应用程序的故障排除和调试技术的地方。让我们从测试这些应用程序开始讨论。

# 测试 gRPC 应用程序

开发的任何软件应用程序（包括 gRPC 应用程序）都需要与应用程序一起进行关联的单元测试。由于 gRPC 应用程序总是与网络交互，因此测试还应涵盖服务器和客户端 gRPC 应用程序的网络 RPC 方面。我们将从测试 gRPC 服务器开始。

## 测试 gRPC 服务器

gRPC 服务测试通常使用 gRPC 客户端应用程序作为测试用例的一部分进行。服务器端测试包括启动具有所需 gRPC 服务的 gRPC 服务器，然后使用客户端应用程序连接到服务器，在其中实现您的测试用例。让我们看一下为我们的 `ProductInfo` 服务的 Go 实现编写的示例测试用例。在 Go 中，gRPC 测试用例的实现应作为使用 `testing` 包的通用测试用例来实现（参见 Example 7-1）。

##### Example 7-1\. 使用 Go 进行 gRPC 服务器端测试

```go
func TestServer_AddProduct(t *testing.T) { ![1](img/1.png)
	grpcServer := initGRPCServerHTTP2() ![2](img/2.png)
	conn, err := grpc.Dial(address, grpc.WithInsecure()) ![3](img/3.png)
	if err != nil {

           grpcServer.Stop()
           t.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewProductInfoClient(conn)

	name := "Sumsung S10"
	description := "Samsung Galaxy S10 is the latest smart phone, launched in
	February 2019"
	price := float32(700.0)
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	r, err := c.AddProduct(ctx, &pb.Product{Name: name,
	                                Description: description, Price: price}) ![4](img/4.png)
	if err != nil { ![5](img/5.png)
		t.Fatalf("Could not add product: %v", err)
	}

	if r.Value == "" {
		t.Errorf("Invalid Product ID %s", r.Value)
	}
	log.Printf("Res %s", r.Value)
      grpcServer.Stop()
}
```

![1](img/#co_running_grpc_in_production_CO1-1)

传统测试启动 gRPC 服务器和客户端，以测试通过 RPC 提供服务。

![2](img/#co_running_grpc_in_production_CO1-2)

启动运行在 HTTP/2 上的传统 gRPC 服务器。

![3](img/#co_running_grpc_in_production_CO1-3)

连接到服务器应用程序。

![4](img/#co_running_grpc_in_production_CO1-4)

发送用于 `AddProduct` 方法的 RPC。

![5](img/#co_running_grpc_in_production_CO1-5)

验证响应消息。

由于 gRPC 测试用例基于标准语言测试用例，你执行它们的方式与标准测试用例没有区别。关于服务器端的 gRPC 测试的一个特别之处是，它们要求服务器应用程序打开一个客户端应用程序连接到的端口。如果你不希望这样做，或者你的测试环境不允许这样做，你可以使用一个库来帮助避免使用真实端口号启动服务。在 Go 中，你可以使用[`bufconn`包](https://oreil.ly/gOq46)，它提供了一个由缓冲区实现的`net.Conn`及相关的拨号和监听功能。你可以在本章的源代码存储库中找到完整的代码示例。如果你使用 Java，你可以使用一个测试框架，如*JUnit*，并按照完全相同的过程编写服务器端的 gRPC 测试用例。但是，如果你希望在不启动 gRPC 服务器实例的情况下编写测试用例，那么你可以使用 Java 实现的 gRPC 内部服务器。你可以在本书的代码存储库中找到这种用 Java 编写的完整代码示例。

你可以在不经过 RPC 网络层的情况下，直接测试你开发的远程函数的业务逻辑。你可以直接调用这些函数来进行测试，而不使用 gRPC 客户端。

通过这样，我们已经学会了如何为 gRPC 服务编写测试。现在让我们谈谈如何测试你的 gRPC 客户端应用程序。

## 测试 gRPC 客户端

当我们为 gRPC 客户端开发测试时，其中一种可能的测试方法是启动一个 gRPC 服务器并实现一个模拟服务。然而，这并不是一项非常直接的任务，因为它会产生打开端口和连接服务器的开销。因此，为了在不连接到真实服务器的情况下测试客户端逻辑，你可以使用一个模拟框架。通过对 gRPC 服务器端进行模拟，开发人员可以编写轻量级单元测试，以检查客户端侧的功能，而无需调用服务器的 RPC 调用。

如果你正在使用 Go 开发 gRPC 客户端应用程序，你可以使用[Gomock](https://oreil.ly/8GAWB)来模拟客户端接口（使用生成的代码），并编程设置其方法来期望并返回预定值。使用 Gomock，你可以为 gRPC 客户端应用程序生成模拟接口：

```go
mockgen github.com/grpc-up-and-running/samples/ch07/grpc-docker/go/proto-gen \
ProductInfoClient > mock_prodinfo/prodinfo_mock.go
```

在这里，我们将`ProductInfoClient`指定为要模拟的接口。然后，你编写的测试代码可以导入`mockgen`生成的包以及`gomock`包，围绕客户端逻辑编写单元测试。如示例 7-2 所示，你可以创建一个模拟对象来期待其方法调用并返回响应。

##### 示例 7-2\. 使用 Gomock 进行 gRPC 客户端测试

```go
func TestAddProduct(t *testing.T) {
	ctrl := gomock.NewController(t)
	defer ctrl.Finish()
	mocklProdInfoClient := NewMockProductInfoClient(ctrl) ![1](img/1.png)
     ...
	req := &pb.Product{Name: name, Description: description, Price: price}

	mocklProdInfoClient. ![2](img/2.png)
	 EXPECT().AddProduct(gomock.Any(), &rpcMsg{msg: req},). ![3](img/3.png)
	 Return(&wrapper.StringValue{Value: "ABC123" + name}, nil) ![4](img/4.png)

	testAddProduct(t, mocklProdInfoClient) ![5](img/5.png)
}

func testAddProduct(t *testing.T, client pb.ProductInfoClient) {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	...

	r, err := client.AddProduct(ctx, &pb.Product{Name: name,
    Description: description, Price: price})

	// test and verify response. }
```

![1](img/#co_running_grpc_in_production_CO2-1)

创建一个模拟对象来期待对远程方法的调用。

![2](img/#co_running_grpc_in_production_CO2-2)

编写模拟对象。

![3](img/#co_running_grpc_in_production_CO2-3)

预期调用`AddProduct`方法。

![4](img/#co_running_grpc_in_production_CO2-4)

返回产品 ID 的模拟值。

![5](img/#co_running_grpc_in_production_CO2-5)

调用实际的测试方法，调用客户端存根的远程方法。

如果您使用 Java，可以使用[Mockito](https://site.mockito.org)测试客户端应用程序，并使用 gRPC Java 实现的进程内服务器实现。您可以参考源代码库以获取这些示例的更多详情。一旦您在服务器端和客户端完成所需的测试，就可以将它们与您使用的持续集成工具集成起来。

需要注意的是，模拟 gRPC 服务器不会像真实的 gRPC 服务器那样提供完全相同的行为。因此，除非重新实现 gRPC 服务器中存在的所有错误逻辑，否则某些功能可能无法通过测试进行验证。实际上，您可以通过模拟验证一组选定的功能，其余功能需要与实际的 gRPC 服务器实现进行验证。现在让我们看看如何对您的 gRPC 应用程序进行负载测试和基准测试。

## 负载测试

对于 gRPC 应用程序，使用传统工具进行负载测试和基准测试比较困难，因为这些应用程序更多或多或少地受限于特定的协议，例如 HTTP。因此，对于 gRPC，我们需要定制的负载测试工具，可以通过向服务器生成 RPC 的虚拟负载来对 gRPC 服务器进行负载测试。

[ghz](https://ghz.sh)是这样一款负载测试工具；它是一个使用 Go 语言实现的命令行实用程序。它可用于本地测试和调试服务，并且在自动化持续集成环境中用于性能回归测试。例如，使用 ghz 可以通过以下命令运行负载测试：

```go
ghz --insecure \
  --proto ./greeter.proto \
  --call helloworld.Greeter.SayHello \
  -d '{"name":"Joe"}'\
  -n 2000 \
  -c 20 \

  0.0.0.0:50051
```

在此，我们不安全地调用`Greeter`服务的`SayHello`远程方法。我们可以指定请求的总数（`-n 2000`）和并发数（20 个线程）。结果也可以生成各种输出格式。

一旦您在服务器端和客户端完成所需的测试，就可以将它们与您使用的持续集成工具集成起来。

## 持续集成

如果您对*持续集成*（CI）还不熟悉，它是一种开发实践，要求开发人员频繁将代码集成到共享存储库中。在每次提交代码时，自动化构建会对代码进行验证，使团队能够早期发现问题。在涉及 gRPC 应用程序时，通常服务器端和客户端应用程序是独立的，可能使用不同的技术构建。因此，在 CI 过程中，您将需要使用我们在前一节学习的单元测试和集成测试技术来验证 gRPC 客户端或服务器端代码。然后，根据您用来构建 gRPC 应用程序的语言，您可以将这些应用程序的测试（例如，Go 测试或 Java JUnit）集成到您选择的 CI 工具中。例如，如果您使用 Go 编写了测试，则可以轻松地将您的 Go 测试与[Jenkins](https://jenkins.io)，[TravisCI](https://travis-ci.com)，[Spinnaker](https://www.spinnaker.io)等工具集成。

一旦您为您的 gRPC 应用程序建立了测试和持续集成（CI）流程，接下来需要关注的是部署您的 gRPC 应用程序。

# 部署

现在，让我们看看我们开发的 gRPC 应用程序的不同部署方法。如果您打算在本地或虚拟机上运行 gRPC 服务器或客户端应用程序，则部署主要取决于您为 gRPC 应用程序的相应编程语言生成的二进制文件。对于本地或基于虚拟机的部署，通常使用标准的部署实践来实现 gRPC 服务器应用程序的扩展性和高可用性，例如使用支持 gRPC 协议的负载均衡器。

大多数现代应用程序现在都以容器方式部署。因此，看看如何在容器上部署您的 gRPC 应用程序是非常有用的。Docker 是基于容器的应用程序部署的标准平台。

## 在 Docker 上部署

[Docker](https://www.docker.com) 是一个用于开发、发布和运行应用程序的开放平台。使用 Docker，您可以将应用程序与基础设施分离开来。它提供了在被称为*容器*的隔离环境中打包和运行应用程序的能力，以便您可以在同一主机上运行多个容器。容器比传统的虚拟机更轻量级，并直接在主机机器的内核中运行。

让我们看一些将 gRPC 应用程序部署为 Docker 容器的示例。

###### 注意

Docker 的基础知识超出了本书的范围。因此，如果您对 Docker 不熟悉，我们建议您参考[Docker 文档](https://docs.docker.com)和其他资源。

开发完 gRPC 服务器应用程序后，您可以为其创建一个 Docker 容器。示例 7-3 展示了一个基于 Go 的 gRPC 服务器的 Dockerfile。在 Dockerfile 中有许多特定于 gRPC 的结构。在这个例子中，我们使用了多阶段的 Docker 构建，在第一阶段构建应用程序，然后在第二阶段作为更轻量级的运行时运行应用程序。生成的服务器端代码也被添加到构建应用程序之前的容器中。

##### 示例 7-3\. Go gRPC 服务器的 Dockerfile

```go
# Multistage build 
# Build stage I: ![1](img/1.png)
FROM golang AS build
ENV location /go/src/github.com/grpc-up-and-running/samples/ch07/grpc-docker/go
WORKDIR ${location}/server

ADD ./server ${location}/server
ADD ./proto-gen ${location}/proto-gen

RUN go get -d ./... ![2](img/2.png)
RUN go install ./... ![3](img/3.png)

RUN CGO_ENABLED=0 go build -o /bin/grpc-productinfo-server ![4](img/4.png)

# Build stage II: ![5](img/5.png)
FROM scratch
COPY --from=build /bin/grpc-productinfo-server /bin/grpc-productinfo-server ![6](img/6.png)

ENTRYPOINT ["/bin/grpc-productinfo-server"]
EXPOSE 50051
```

![1](img/#co_running_grpc_in_production_CO3-1)

只需使用 Go 语言和 Alpine Linux 来构建程序。

![2](img/#co_running_grpc_in_production_CO3-2)

下载所有的依赖项。

![3](img/#co_running_grpc_in_production_CO3-3)

安装所有的包。

![4](img/#co_running_grpc_in_production_CO3-4)

构建服务器应用程序。

![5](img/#co_running_grpc_in_production_CO3-5)

Go 二进制文件是自包含的可执行文件。

![6](img/#co_running_grpc_in_production_CO3-6)

将在上一个阶段构建的二进制文件复制到新位置。

创建 Dockerfile 后，您可以使用以下命令构建 Docker 镜像：

```go
docker image build -t grpc-productinfo-server -f server/Dockerfile
```

gRPC 客户端应用程序可以使用相同的方法创建。这里唯一的例外是，由于我们在 Docker 上运行服务器应用程序，客户端应用程序用于连接 gRPC 的主机名和端口现在不同了。

当我们在 Docker 上同时运行服务器和客户端的 gRPC 应用程序时，它们需要通过主机机器与彼此和外部世界进行通信。因此，必须涉及网络层。Docker 支持不同类型的网络，每种适用于特定的用例。因此，当我们运行服务器和客户端 Docker 容器时，我们可以指定一个共同的网络，以便客户端应用程序可以根据主机名发现服务器应用程序的位置。这意味着客户端应用程序的代码必须更改，以便连接到服务器的主机名。例如，我们的 Go gRPC 应用程序必须修改为调用服务的主机名，而不是 localhost：

```go
conn, err := grpc.Dial("productinfo:50051", grpc.WithInsecure())
```

您可以从环境中读取主机名，而不是在客户端应用程序中硬编码它。完成对客户端应用程序的更改后，您需要重新构建 Docker 镜像，然后像这样运行服务器和客户端镜像：

```go
docker run -it --network=my-net --name=productinfo \
    --hostname=productinfo
    -p 50051:50051  grpc-productinfo-server ![1](img/1.png)

docker run -it --network=my-net \
    --hostname=client grpc-productinfo-client ![2](img/2.png)
```

![1](img/#co_running_grpc_in_production_CO4-1)

在 Docker 网络 *my-net* 上运行带有主机名为 `productinfo`，端口为 50051 的 gRPC 服务器。

![2](img/#co_running_grpc_in_production_CO4-2)

在 Docker 网络 *my-net* 上运行 gRPC 客户端。

在启动 Docker 容器时，可以指定容器所在的 Docker 网络。如果服务共享同一个网络，那么客户端应用程序可以使用与 `docker run` 命令一起提供的主机名发现主机服务的实际地址。

当您运行的容器数量较少且它们的交互相对简单时，您可能完全可以在 Docker 上构建解决方案。然而，大多数现实场景需要管理多个容器及其交互。仅基于 Docker 构建这样的解决方案非常繁琐。这就是容器编排平台发挥作用的地方。

## 部署在 Kubernetes 上

Kubernetes 是一个开源平台，用于自动化部署、扩展和管理容器化应用程序。当您使用 Docker 运行容器化的 gRPC 应用程序时，默认情况下不提供可扩展性或高可用性保证。您需要在 Docker 容器之外构建这些功能。Kubernetes 提供了广泛的功能，使您能够将大多数容器管理和编排任务卸载到底层 Kubernetes 平台上。

###### 注意

[Kubernetes](https://kubernetes.io) 提供了一个可靠且可扩展的平台，用于运行容器化的工作负载。Kubernetes 负责处理扩展需求、故障转移、服务发现、配置管理、安全性、部署模式等等。

Kubernetes 的基础知识超出了本书的范围。因此，我们建议您参考 [Kubernetes 文档](https://oreil.ly/csW_8) 和其他类似资源以获取更多信息。

让我们看看如何将您的 gRPC 服务器应用程序部署到 Kubernetes 中。

### 用于 gRPC 服务器的 Kubernetes 部署资源

要在 Kubernetes 中部署，您需要做的第一件事是为您的 gRPC 服务器应用程序创建一个 Docker 容器。我们在前一节中已经做到了这一点，您可以在这里使用相同的容器。您可以将容器镜像推送到像 Docker Hub 这样的容器注册表中。

对于本示例，我们已将 gRPC 服务器 Docker 镜像推送到 Docker Hub，并使用标签 `kasunindrasiri/grpc-productinfo-server`。Kubernetes 平台不直接管理容器，而是使用称为 *pod* 的抽象。Pod 是一个逻辑单元，可以包含一个或多个容器；它是 Kubernetes 中的复制单位。例如，如果您需要多个 gRPC 服务器应用程序的实例，那么 Kubernetes 将创建更多的 pod。在给定 pod 上运行的容器共享相同的资源和本地网络。然而，在我们的情况下，我们只需要在 pod 中运行一个 gRPC 服务器容器。因此，这是一个只有单个容器的 pod。Kubernetes 不直接管理 pod。而是使用另一个称为 *deployment* 的抽象。部署指定了应该同时运行的 pod 数量。创建新部署时，Kubernetes 会启动部署中指定数量的 pod。

要在 Kubernetes 中部署我们的 gRPC 服务器应用程序，我们需要使用 示例 7-4 中显示的 YAML 描述符创建一个 Kubernetes 部署。

##### 示例 7-4\. Go gRPC 服务器应用程序的 Kubernetes 部署描述符

```go
apiVersion: apps/v1
kind: Deployment ![1](img/1.png)
metadata:
  name: grpc-productinfo-server ![2](img/2.png)
spec:
  replicas: 1 ![3](img/3.png)
  selector:
    matchLabels:
      app: grpc-productinfo-server
  template:
    metadata:
      labels:
        app: grpc-productinfo-server
    spec:
      containers:
      - name: grpc-productinfo-server ![4](img/4.png)
        image: kasunindrasiri/grpc-productinfo-server ![5](img/5.png)
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 50051
          name: grpc
```

![1](img/#co_running_grpc_in_production_CO5-1)

声明一个 Kubernetes `Deployment`对象。

![2](img/#co_running_grpc_in_production_CO5-2)

部署的名称。

![3](img/#co_running_grpc_in_production_CO5-3)

同时运行的 gRPC 服务器 pod 的数量。

![4](img/#co_running_grpc_in_production_CO5-4)

关联的 gRPC 服务器容器的名称。

![5](img/#co_running_grpc_in_production_CO5-5)

gRPC 服务器容器的镜像名称和标签。

当您在 Kubernetes 中使用`kubectl apply -f server/grpc-prodinfo-server.yaml`应用此描述符时，您将在 Kubernetes 集群中获得一个运行中的 gRPC 服务器 pod 的部署。然而，如果 gRPC 客户端应用程序必须访问运行在同一 Kubernetes 集群中的 gRPC 服务器 pod，则必须找到 pod 的确切 IP 地址和端口并发送 RPC。但是，当 pod 重新启动时，IP 地址可能会更改，并且如果您运行多个副本，则必须处理每个副本的多个 IP 地址。为了克服这个限制，Kubernetes 提供了一个称为*service*的抽象。

### 用于 gRPC 服务器的 Kubernetes 服务资源

您可以创建一个 Kubernetes 服务，并将其与匹配的 pod（在本例中为 gRPC 服务器 pod）关联起来，您将获得一个 DNS 名称，该名称将自动将流量路由到任何匹配的 pod。因此，您可以将服务视为一个 Web 代理或负载均衡器，用于将请求转发到底层的 pod。示例 7-5 显示了 gRPC 服务器应用程序的 Kubernetes 服务描述符。

##### 示例 7-5\. Go gRPC 服务器应用程序的 Kubernetes 服务描述符

```go
apiVersion: v1
kind: Service ![1](img/1.png)
metadata:
  name: productinfo ![2](img/2.png)
spec:
  selector:
    app: grpc-productinfo-server ![3](img/3.png)
  ports:
  - port: 50051 ![4](img/4.png)
    targetPort: 50051
    name: grpc
  type: NodePort
```

![1](img/#co_running_grpc_in_production_CO6-1)

指定一个`Service`描述符。

![2](img/#co_running_grpc_in_production_CO6-2)

服务的名称。客户端应用程序连接到服务时将使用此名称。

![3](img/#co_running_grpc_in_production_CO6-3)

这告诉服务将请求路由到具有匹配标签`grpc-productinfo-server`的 pod。

![4](img/#co_running_grpc_in_production_CO6-4)

服务运行在端口 50051，并将请求转发到目标端口 50051。

所以，一旦您创建了`Deployment`和`Service`描述符，您可以使用`kubectl apply -f server/grpc-prodinfo-server.yaml`将此应用部署到 Kubernetes 中（您可以将两个描述符放在同一个 YAML 文件中）。成功部署这些对象应该会给您一个运行中的 gRPC 服务器 pod，一个用于 gRPC 服务器的 Kubernetes 服务以及一个部署。

下一步是将 gRPC 客户端部署到 Kubernetes 集群。

### 用于运行 gRPC 客户端的 Kubernetes Job

当您在 Kubernetes 集群上成功运行 gRPC 服务器后，还可以在同一集群中运行 gRPC 客户端应用程序。客户端可以通过我们在前一步创建的名为 `productinfo` 的 gRPC 服务访问 gRPC 服务器。因此，在客户端的代码中，您应将 Kubernetes 服务名称用作主机名，并使用 gRPC 服务器的服务端口作为端口名称。因此，客户端在连接 Go 实现的服务器时将使用 `grpc.Dial("productinfo:50051", grpc.WithInsecure())`。如果假设我们的客户端应用程序需要运行指定次数（即仅调用 gRPC 服务、记录响应并退出），则我们可能不使用 Kubernetes 部署，而是使用 Kubernetes *job*。Kubernetes job 设计用于指定次数运行 Pod。

您可以以与 gRPC 服务器相同的方式创建客户端应用程序容器。一旦容器推送到 Docker 注册表中，然后您可以按照 示例 7-6 中显示的 Kubernetes `Job` 描述符指定它。

##### 示例 7-6\. gRPC 客户端应用程序作为 Kubernetes job 运行

```go
apiVersion: batch/v1
kind: Job ![1](img/1.png)
metadata:
  name: grpc-productinfo-client ![2](img/2.png)
spec:
  completions: 1 ![3](img/3.png)
  parallelism: 1 ![4](img/4.png)
  template:
    spec:
      containers:
      - name: grpc-productinfo-client ![5](img/5.png)
        image: kasunindrasiri/grpc-productinfo-client ![6](img/6.png)
      restartPolicy: Never
  backoffLimit: 4
```

![1](img/#co_running_grpc_in_production_CO7-1)

指定一个 Kubernetes `Job`。

![2](img/#co_running_grpc_in_production_CO7-2)

作业的名称.

![3](img/#co_running_grpc_in_production_CO7-3)

Pod 成功运行的次数，作业才被视为完成。

![4](img/#co_running_grpc_in_production_CO7-4)

应并行运行的 Pod 数量。

![5](img/#co_running_grpc_in_production_CO7-5)

关联的 gRPC 客户端容器的名称。

![6](img/#co_running_grpc_in_production_CO7-6)

与此作业关联的容器镜像。

接下来，您可以使用 `kubectl apply -f client/grpc-prodinfo-client-job.yaml` 部署 gRPC 客户端应用程序的作业，并检查 Pod 的状态。

该作业执行成功后，会向我们的 `ProductInfo` gRPC 服务发送添加产品的 RPC。因此，您可以观察服务器和客户端 Pod 的日志，查看是否获得了预期的信息。

然后我们可以使用入口资源将您的 gRPC 服务暴露到 Kubernetes 集群之外。

### 用于外部暴露 gRPC 服务的 Kubernetes Ingress

到目前为止，我们已经在 Kubernetes 上部署了一个 gRPC 服务器，并使其对同一集群中作为作业运行的另一个 Pod 可访问。如果我们想将 gRPC 服务暴露给 Kubernetes 集群之外的外部应用程序，会怎样？正如您所了解的那样，Kubernetes 服务构造仅用于将给定的 Kubernetes Pod 暴露给集群中运行的其他 Pod。因此，Kubernetes 服务无法被位于 Kubernetes 集群之外的外部应用程序访问。Kubernetes 提供了另一个称为 *ingress* 的抽象来实现此目的。

我们可以将 Ingress 想象成位于 Kubernetes 服务与外部应用程序之间的负载均衡器。Ingress 将外部流量路由到服务；服务然后在匹配的 Pod 之间路由内部流量。Ingress 控制器管理给定 Kubernetes 集群中的 Ingress 资源。Ingress 控制器的类型和行为可能会根据您使用的集群而变化。此外，当您向外部应用程序公开 gRPC 服务时，支持在 Ingress 层面进行 gRPC 路由是必需的条件之一。因此，我们需要选择一个支持 gRPC 的 Ingress 控制器。

在这个示例中，我们将使用基于 [Nginx](https://oreil.ly/0UC0a) 负载均衡器的 [Nginx](https://www.nginx.com) Ingress 控制器。根据您使用的 Kubernetes 集群，您可以选择支持 gRPC 的最合适的 Ingress 控制器。[Nginx Ingress](https://oreil.ly/wZo5w) 支持将外部流量路由到内部服务的 gRPC。

因此，为了将我们的 `ProductInfo` gRPC 服务器应用程序暴露给外部世界（即 Kubernetes 集群之外），我们可以创建如 示例 7-7 所示的 `Ingress` 资源。

##### 示例 7-7\. Go gRPC 服务器应用程序的 Kubernetes Ingress 资源

```go
apiVersion: extensions/v1beta1
kind: Ingress ![1](img/1.png)
metadata:
  annotations: ![2](img/2.png)
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
  name: grpc-prodinfo-ingress ![3](img/3.png)
spec:
  rules:
  - host: productinfo ![4](img/4.png)
    http:
      paths:
      - backend:
          serviceName: productinfo ![5](img/5.png)
          servicePort: grpc ![6](img/6.png)
```

![1](img/#co_running_grpc_in_production_CO8-1)

指定 `Ingress` 资源。

![2](img/#co_running_grpc_in_production_CO8-2)

与 Nginx Ingress 控制器相关的注解，并指定 gRPC 作为后端协议。

![3](img/#co_running_grpc_in_production_CO8-3)

`Ingress` 资源的名称。

![4](img/#co_running_grpc_in_production_CO8-4)

这是向外界公开的主机名。

![5](img/#co_running_grpc_in_production_CO8-5)

关联的 Kubernetes 服务的名称。

![6](img/#co_running_grpc_in_production_CO8-6)

在 Kubernetes 服务中指定的服务端口的名称。

在部署上述的入口资源之前，您需要安装 Nginx Ingress 控制器。您可以在 Kubernetes 的 [Ingress-Nginx](https://oreil.ly/l-vFp) 仓库中找到有关安装和使用 Nginx Ingress 与 gRPC 的更多详细信息。一旦您部署了这个 `Ingress` 资源，任何外部应用程序都可以通过主机名 (`productinfo`) 和默认端口 (80) 调用 gRPC 服务器。

通过以上内容，您已经学习了关于在 Kubernetes 上部署生产就绪的 gRPC 应用程序的所有基础知识。正如您所见，由于 Kubernetes 和 Docker 提供的能力，我们不必过多担心大多数非功能性需求，如可伸缩性、高可用性、负载均衡、故障转移等，因为 Kubernetes 已作为底层平台的一部分提供了这些功能。因此，一些我们在 第 6 章 中学习的概念，如负载均衡、gRPC 代码层面的名称解析等，在运行 gRPC 应用程序时在 Kubernetes 上是不必要的。

一旦您的基于 gRPC 的应用程序运行起来，您需要确保生产环境中应用程序的平稳运行。为了实现这一目标，您需要持续观察您的 gRPC 应用程序，并在需要时采取必要的行动。让我们深入探讨 gRPC 应用程序的可观测性方面的细节。

# 可观测性

正如我们在上一节中讨论的那样，gRPC 应用程序通常部署和运行在容器化环境中，其中有多个此类容器通过网络进行通信。然后出现了如何跟踪每个容器并确保它们实际工作的问题。这就是*可观测性*的作用所在。

正如[维基百科的定义](https://oreil.ly/FVPTN)所述，“可观测性是指从系统外部输出的知识中推断出系统内部状态的程度。”基本上，有关系统可观测性的目的是回答问题：“系统当前有问题吗？”如果答案是肯定的，我们还应该能够回答一系列其他问题，如“出了什么问题？”和“为什么会发生这种情况？”如果我们能够在任何给定时间和系统的任何部分回答这些问题，我们可以说我们的系统是可观测的。

还需要注意的是，可观测性是系统的一个属性，与效率、可用性和可靠性一样重要。因此，在构建 gRPC 应用程序时，必须从一开始考虑它。

谈论可观测性时，我们通常谈论三个主要支柱：指标、日志记录和跟踪。这些是用于获得系统可观测性的主要技术。让我们在接下来的部分分别讨论每一个。

## 指标

*指标*是对数据在时间间隔内的数值表示。谈到指标时，我们可以收集两种类型的数据。一种是系统级指标，如 CPU 使用率、内存使用率等。另一种是应用级指标，如入站请求速率、请求错误率等。

当应用程序运行时通常会捕获系统级指标。如今，有许多工具用于捕获这些指标，通常由 DevOps 团队捕获。但是应用程序级指标在不同的应用程序之间有所不同。因此，在设计新应用程序时，应用程序开发人员需要决定需要捕获哪些类型的应用程序级指标，以了解系统行为。在本节中，我们将重点介绍如何在我们的应用程序中启用应用程序级指标。

### 使用 gRPC 的 OpenCensus

对于 gRPC 应用程序，[OpenCensus](https://oreil.ly/EMfF-) 库提供了标准指标。我们可以通过在客户端和服务器应用程序中添加处理程序来轻松启用它们。我们还可以添加自己的指标收集器（示例 7-8）。

###### 注意

[OpenCensus](https://opencensus.io) 是一个用于收集应用程序指标和分布式跟踪的开源库集合；它支持多种语言。它从目标应用程序收集指标并实时传输数据到您选择的后端。目前支持的后端包括 Azure Monitor、Datadog、Instana、Jaeger、SignalFX、Stackdriver 和 Zipkin。我们也可以为其他后端编写自定义导出器。

##### 示例 7-8\. 为 gRPC Go 服务器启用 OpenCensus 监控

```go
package main

import (
  "errors"
  "log"
  "net"
  "net/http"

  pb "productinfo/server/ecommerce"
  "google.golang.org/grpc"
  "go.opencensus.io/plugin/ocgrpc" ![1](img/1.png)
  "go.opencensus.io/stats/view"
  "go.opencensus.io/zpages"
  "go.opencensus.io/examples/exporter"
)

const (
  port = ":50051"
)

// server is used to implement ecommerce/product_info. type server struct {
  productMap map[string]*pb.Product
}

func main() {

  go func() { ![7](img/7.png)
     mux := http.NewServeMux()
     zpages.Handle(mux, "/debug")
     log.Fatal(http.ListenAndServe("127.0.0.1:8081", mux))
  }()

   view.RegisterExporter(&exporter.PrintExporter{}) ![2](img/2.png)

  if err := view.Register(ocgrpc.DefaultServerViews...); err != nil { ![3](img/3.png)
     log.Fatal(err)
  }

  grpcServer := grpc.NewServer(grpc.StatsHandler(&ocgrpc.ServerHandler{})) ![4](img/4.png)
  pb.RegisterProductInfoServer(grpcServer, &server{}) ![5](img/5.png)

  lis, err := net.Listen("tcp", port)
  if err != nil {
     log.Fatalf("Failed to listen: %v", err)
  }

  if err := grpcServer.Serve(lis); err != nil { ![6](img/6.png)
     log.Fatalf("failed to serve: %v", err)
  }
}
```

![1](img/#co_running_grpc_in_production_CO9-1)

指定我们需要添加的外部库以启用监控。gRPC OpenCensus 提供了一组预定义的处理程序，以支持 OpenCensus 监控。在这里，我们将使用这些处理程序。

![2](img/#co_running_grpc_in_production_CO9-3)

注册统计导出器以导出收集的数据。这里我们添加 `PrintExporter`，它将导出的数据记录到控制台。这仅用于演示目的；通常不建议记录所有生产负载。

![3](img/#co_running_grpc_in_production_CO9-4)

注册视图以收集服务器请求计数。这些是预定义的默认服务视图，用于收集每个 RPC 的接收字节、发送字节、RPC 的延迟和已完成的 RPC。我们可以编写自己的视图来收集数据。

![4](img/#co_running_grpc_in_production_CO9-5)

创建一个带有统计处理程序的 gRPC 服务器。

![5](img/#co_running_grpc_in_production_CO9-6)

将我们的 `ProductInfo` 服务注册到 gRPC 服务器。

![6](img/#co_running_grpc_in_production_CO9-7)

开始监听端口（50051）上传入的消息。

![7](img/#co_running_grpc_in_production_CO9-2)

启动 z-Pages 服务器。一个 HTTP 端点从端口 8081 开始，用于可视化指标的上下文为 `/debug`。

类似于 gRPC 服务器，我们可以使用客户端端处理程序在 gRPC 客户端中启用 OpenCensus 监控。示例 7-9 提供了向 Go 编写的 gRPC 客户端添加度量处理程序的代码片段。

##### 示例 7-9\. 为 gRPC Go 服务器启用 OpenCensus 监控

```go
package main

import (
  "context"
  "log"
  "time"

  pb "productinfo/server/ecommerce"
  "google.golang.org/grpc"
  "go.opencensus.io/plugin/ocgrpc" ![1](img/1.png)
  "go.opencensus.io/stats/view"
  "go.opencensus.io/examples/exporter"
)

const (
  address = "localhost:50051"
)

func main() {
  view.RegisterExporter(&exporter.PrintExporter{}) ![2](img/2.png)

   if err := view.Register(ocgrpc.DefaultClientViews...); err != nil { ![3](img/3.png)
       log.Fatal(err)
   }

  conn, err := grpc.Dial(address, ![4](img/4.png)
        grpc.WithStatsHandler(&ocgrpc.ClientHandler{}),
          grpc.WithInsecure(),
          )
  if err != nil {
     log.Fatalf("Can't connect: %v", err)
  }
  defer conn.Close() ![6](img/6.png)

  c := pb.NewProductInfoClient(conn) ![5](img/5.png)

  .... // Skip RPC method invocation. }
```

![1](img/#co_running_grpc_in_production_CO10-1)

指定我们需要添加的外部库以启用监控。

![2](img/#co_running_grpc_in_production_CO10-2)

注册统计和跟踪导出器以导出收集的数据。这里我们将添加 `PrintExporter`，它将导出的数据记录到控制台。这仅用于演示目的。通常不建议记录所有生产负载。

![3](img/#co_running_grpc_in_production_CO10-3)

注册视图以收集服务器请求计数。这些是预定义的默认服务视图，用于收集每个 RPC 的接收字节、发送字节、RPC 的延迟和已完成的 RPC。我们可以编写自己的视图来收集数据。

![4](img/#co_running_grpc_in_production_CO10-4)

设置与服务器的连接，并注册客户端状态处理程序。

![5](img/#co_running_grpc_in_production_CO10-6)

使用服务器连接创建客户端存根。

![6](img/#co_running_grpc_in_production_CO10-5)

在一切都完成后关闭连接。

一旦运行服务器和客户端，我们就可以通过创建的 HTTP 端点访问服务器和客户端度量（例如，RPC 度量位于[*http://localhost:8081/debug/rpcz*](http://localhost:8081/debug/rpcz)，跟踪位于[*http://localhost:8081/debug/tracez)*](http://localhost:8081/debug/tracez)）。

如前所述，我们可以使用预定义的导出器将数据发布到支持的后端，也可以编写自己的导出器将跟踪和度量发送到能够消费它们的任何后端。

在下一节中，我们将讨论另一个流行的技术，[Prometheus](https://prometheus.io)，通常用于为 gRPC 应用程序启用度量。

### Prometheus 与 gRPC

Prometheus 是一个用于系统监控和警报的开源工具包。您可以使用[grpc Prometheus 库](https://oreil.ly/nm84_)为您的 gRPC 应用程序启用度量。通过为客户端和服务器应用程序添加拦截器，我们可以轻松实现此功能，还可以添加自定义度量收集器。

###### 注意

Prometheus 通过调用以上下文`/metrics`开头的 HTTP 端点从目标应用程序收集度量。它存储所有收集的数据，并对这些数据运行规则，以聚合并记录新的时间序列或生成警报。我们可以使用诸如[Grafana](https://grafana.com)之类的工具可视化这些聚合结果。

示例 7-10 演示了如何在我们使用 Go 编写的产品管理服务器中添加度量拦截器和自定义度量收集器。

##### 示例 7-10\. 为 gRPC Go 服务器启用 Prometheus 监控

```go
package main

import (
  ...
  "github.com/grpc-ecosystem/go-grpc-prometheus" ![1](img/1.png)
  "github.com/prometheus/client_golang/prometheus"
  "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
  reg = prometheus.NewRegistry() ![2](img/2.png)

  grpcMetrics = grpc_prometheus.NewServerMetrics() ![3](img/3.png)

   customMetricCounter = prometheus.NewCounterVec(prometheus.CounterOpts{
       Name: "product_mgt_server_handle_count",
       Help: "Total number of RPCs handled on the server.",
   }, []string{"name"}) ![4](img/4.png)
)

func init() {
   reg.MustRegister(grpcMetrics, customMetricCounter) ![5](img/5.png)
}

func main() {
  lis, err := net.Listen("tcp", port)
  if err != nil {
     log.Fatalf("failed to listen: %v", err)
  }

  httpServer := &http.Server{
      Handler: promhttp.HandlerFor(reg, promhttp.HandlerOpts{}),
        Addr:  fmt.Sprintf("0.0.0.0:%d", 9092)} ![6](img/6.png)

  grpcServer := grpc.NewServer(
     grpc.UnaryInterceptor(grpcMetrics.UnaryServerInterceptor()), ![7](img/7.png)
  )

  pb.RegisterProductInfoServer(grpcServer, &server{})
  grpcMetrics.InitializeMetrics(grpcServer) ![8](img/8.png)

  // Start your http server for prometheus.
  go func() {
     if err := httpServer.ListenAndServe(); err != nil {
        log.Fatal("Unable to start a http server.")
     }
  }()

  if err := grpcServer.Serve(lis); err != nil {
     log.Fatalf("failed to serve: %v", err)
  }
}
```

![1](img/#co_running_grpc_in_production_CO11-1)

指定我们需要添加的外部库以启用监控。gRPC 生态系统提供了一组预定义的拦截器来支持 Prometheus 监控。在这里，我们将使用这些拦截器。

![2](img/#co_running_grpc_in_production_CO11-2)

创建度量注册表。该注册表保存系统中注册的所有数据收集器。如果需要添加新的收集器，我们需要在此注册表中注册它。

![3](img/#co_running_grpc_in_production_CO11-3)

创建标准客户端度量。这些是库中定义的预定义度量。

![4](img/#co_running_grpc_in_production_CO11-4)

创建名为`product_mgt_server_handle_count`的自定义度量计数器。

![5](img/#co_running_grpc_in_production_CO11-5)

在第 2 步创建的注册表中注册标准服务器度量和自定义度量收集器。

![6](img/#co_running_grpc_in_production_CO11-6)

为 Prometheus 创建一个 HTTP 服务器。用于度量收集的 HTTP 端点在端口 9092 上以上下文`/metrics`开头。

![7](img/#co_running_grpc_in_production_CO11-7)

创建一个带有度量拦截器的 gRPC 服务器。这里我们使用 `grpcMetrics.UnaryServerInterceptor`，因为我们有一个一元服务。还有一个叫做 `grpcMetrics.StreamServerInterceptor()` 的拦截器用于流式服务。

![8](img/#co_running_grpc_in_production_CO11-8)

初始化所有标准度量。

使用第 4 步创建的自定义度量计数器，我们可以添加更多用于监控的度量指标。比如说，我们想要收集添加到产品管理系统中具有相同名称的产品的数量。如示例 7-11 所示，在 `addProduct` 方法中我们可以向 `customMetricCounter` 添加新的度量指标。

##### 示例 7-11\. 向自定义度量计数器添加新的度量指标

```go
// AddProduct implements ecommerce.AddProduct
func (s *server) AddProduct(ctx context.Context,
     in *pb.Product) (*wrapper.StringValue, error) {
     customMetricCounter.WithLabelValues(in.Name).Inc()
  ...
}
```

类似于 gRPC 服务器，我们可以使用客户端拦截器在 gRPC 客户端中启用 Prometheus 监控。示例 7-12 提供了在 Go 中编写的 gRPC 客户端添加度量拦截器的代码片段。

##### 示例 7-12\. 为 gRPC Go 客户端启用 Prometheus 监控

```go
package main

import (
  ...
  "github.com/grpc-ecosystem/go-grpc-prometheus" ![1](img/1.png)
  "github.com/prometheus/client_golang/prometheus"
  "github.com/prometheus/client_golang/prometheus/promhttp"
)

const (
  address = "localhost:50051"
)

func main() {
  reg := prometheus.NewRegistry() ![2](img/2.png)
  grpcMetrics := grpc_prometheus.NewClientMetrics() ![3](img/3.png)
  reg.MustRegister(grpcMetrics) ![4](img/4.png)

  conn, err := grpc.Dial(address,
        grpc.WithUnaryInterceptor(grpcMetrics.UnaryClientInterceptor()), ![5](img/5.png)
          grpc.WithInsecure(),
          )
  if err != nil {
     log.Fatalf("did not connect: %v", err)
  }
  defer conn.Close()

   // Create a HTTP server for prometheus.
   httpServer := &http.Server{
        Handler: promhttp.HandlerFor(reg, promhttp.HandlerOpts{}),
          Addr: fmt.Sprintf("0.0.0.0:%d", 9094)} ![6](img/6.png)

   // Start your http server for prometheus.
   go func() {
       if err := httpServer.ListenAndServe(); err != nil {
           log.Fatal("Unable to start a http server.")
       }
   }()

  c := pb.NewProductInfoClient(conn)
  ...
}
```

![1](img/#co_running_grpc_in_production_CO12-1)

指定我们需要添加的外部库以启用监控。

![2](img/#co_running_grpc_in_production_CO12-2)

创建度量注册表。与服务器代码类似，这个注册表包含系统中注册的所有数据收集器。如果我们需要添加一个新的收集器，我们需要将其注册到这个注册表中。

![3](img/#co_running_grpc_in_production_CO12-3)

创建标准服务器度量。这些是库中定义的预定义度量指标。

![4](img/#co_running_grpc_in_production_CO12-4)

将标准客户端度量注册到第 2 步创建的注册表中。

![5](img/#co_running_grpc_in_production_CO12-5)

使用度量拦截器建立与服务器的连接。这里我们使用 `grpcMetrics.UnaryClientInterceptor`，因为我们有一个一元客户端。另一个拦截器叫做 `grpcMetrics.StreamClientInterceptor()`，用于流式客户端。

![6](img/#co_running_grpc_in_production_CO12-6)

为 Prometheus 创建一个 HTTP 服务器。一个 HTTP 端点从 `/metrics` 开始，监听 9094 端口用于度量收集。

一旦服务器和客户端运行，我们可以通过创建的 HTTP 端点访问服务器和客户端的度量（例如，服务器度量在 [*http://localhost:9092/metrics*](http://localhost:9092/metrics)，客户端度量在 [*http://localhost:9094/metrics)*](http://localhost:9094/metrics)）。

正如之前提到的，Prometheus 可以通过访问上述 URL 收集度量。Prometheus 将所有度量数据存储在本地，并应用一组规则来聚合和创建新的记录。通过将 Prometheus 作为数据源，我们可以使用像 Grafana 这样的工具在仪表板中可视化度量数据。

###### 注意

Grafana 是一个开源的度量仪表板和图形编辑器，用于 Graphite、Elasticsearch 和 Prometheus。它允许您查询、可视化和理解您的度量数据。

系统中基于度量的监控的一个优势是处理度量数据的成本不会随系统活动的增加而增加。例如，应用程序流量的增加不会增加处理成本，如磁盘利用率、处理复杂性、可视化速度、运营成本等。它具有恒定的开销。此外，一旦收集到度量数据，我们可以进行多种数学和统计转换，并得出有关系统的有价值结论。

观察性的另一个支柱是日志，在下一节中我们将讨论它。

## 日志

日志是不可变的、带有时间戳的离散事件记录，记录了系统在时间上发生的具体事件。作为应用程序开发者，我们通常将数据转储到日志中，以便在特定时间点告知系统的内部状态在何处以及是什么。日志的好处在于它们最容易生成，并且比度量更加精细化。我们可以附加特定操作或一堆上下文信息，例如唯一标识符、我们将要执行的操作、堆栈跟踪等。不利之处在于它们非常昂贵，因为我们需要以易于搜索和使用的方式存储和索引它们。

在 gRPC 应用程序中，我们可以使用拦截器启用日志记录。正如我们在 第五章 中讨论的那样，我们可以在客户端和服务器端都附加新的日志拦截器，并记录每次远程调用的请求和响应消息。

###### 注意

gRPC 生态系统为 Go 应用程序提供了一组预定义的日志拦截器。其中包括 `grpc_ctxtags`，一个库，它将一个标签映射添加到上下文中，并从请求体中填充数据；`grpc_zap`，将 [zap](https://oreil.ly/XMlIg) 日志库集成到 gRPC 处理程序中；以及 `grpc_logrus`，将 [logrus](https://oreil.ly/oKJX5) 日志库集成到 gRPC 处理程序中。有关这些拦截器的更多信息，请查看 [gRPC Go Middleware 仓库](https://oreil.ly/8lNaH)。

一旦在您的 gRPC 应用程序中添加了日志，它们将根据您的日志配置打印到控制台或日志文件中。如何配置日志取决于您使用的日志框架。

我们现在已经讨论了观察性的两个支柱：度量和日志。这些足以理解单个系统的性能和行为，但不足以理解跨多个系统遍历的请求的生命周期。分布式追踪是一种技术，它提供了对请求生命周期在多个系统中的可见性。

## 追踪

跟踪是一系列相关事件的表示，构建了通过分布式系统的端到端请求流。正如我们在章节 “使用 gRPC 进行微服务通信” 中讨论的，在现实场景中，我们有多个微服务提供不同和特定的业务能力。因此，从客户端发起的请求通常会经过多个服务和不同的系统，然后响应返回给客户端。所有这些中间事件都是请求流的一部分。通过跟踪，我们可以看到请求所经过的路径以及请求的结构。

在跟踪中，一个跟踪是 *跨度* 的树形结构，跨度是分布式跟踪的主要构建块。跨度包含任务的元数据、延迟（完成任务所花费的时间）和任务的其他相关属性。一个跟踪有自己的 ID，称为 TraceID，它是一个唯一的字节序列。这个 TraceID 将不同的跨度分组和区分开来。让我们尝试在我们的 gRPC 应用程序中启用跟踪。

类似于指标，OpenCensus 库支持在 gRPC 应用程序中启用跟踪。我们将使用 OpenCensus 在我们的产品管理应用程序中启用跟踪。正如前面所述，我们可以将任何支持的导出器插入到不同的后端以导出跟踪数据。我们将使用 Jaeger 作为分布式跟踪样例的后端。

gRPC Go 默认启用了跟踪功能。因此，只需注册一个导出器即可开始在 gRPC Go 集成中收集跟踪数据。让我们在客户端和服务器应用程序中启动一个 Jaeger 导出器。示例 7-13 演示了如何使用库初始化 OpenCensus Jaeger 导出器。

##### 示例 7-13\. 初始化 OpenCensus Jaeger 导出器

```go
package tracer

import (
  "log"

  "go.opencensus.io/trace" ![1](img/1.png)
  "contrib.go.opencensus.io/exporter/jaeger"

)

func initTracing() {

   trace.ApplyConfig(trace.Config{DefaultSampler: trace.AlwaysSample()})
   agentEndpointURI := "localhost:6831"
   collectorEndpointURI := "http://localhost:14268/api/traces" ![2](img/2.png)
    exporter, err := jaeger.NewExporter(jaeger.Options{
            CollectorEndpoint: collectorEndpointURI,
            AgentEndpoint: agentEndpointURI,
            ServiceName:    "product_info",

    })
    if err != nil {
       log.Fatal(err)
    }
    trace.RegisterExporter(exporter) ![3](img/3.png)

}
```

![1](img/#co_running_grpc_in_production_CO13-1)

导入 OpenTracing 和 Jaeger 库。

![2](img/#co_running_grpc_in_production_CO13-2)

创建 Jaeger 导出器，并指定收集器端点、服务名称和代理端点。

![3](img/#co_running_grpc_in_production_CO13-3)

将导出器注册到 OpenCensus 追踪器中。

一旦我们在服务器上注册了导出器，我们就可以通过跟踪来为服务器添加仪表。示例 7-14 展示了如何在服务方法中添加仪表。

##### 示例 7-14\. 为 gRPC 服务方法添加仪表

```go
// GetProduct implements ecommerce.GetProduct func (s *server) GetProduct(ctx context.Context, in *wrapper.StringValue) (
         *pb.Product, error) {
  ctx, span := trace.StartSpan(ctx, "ecommerce.GetProduct") ![1](img/1.png)
  defer span.End() ![2](img/2.png)
  value, exists := s.productMap[in.Value]
  if exists {
     return value, status.New(codes.OK, "").Err()
  }
  return nil, status.Errorf(codes.NotFound, "Product does not exist.", in.Value)
}
```

![1](img/#co_running_grpc_in_production_CO14-1)

使用跨度名称和上下文启动新跨度。

![2](img/#co_running_grpc_in_production_CO14-2)

当所有操作完成时停止跨度。

类似于 gRPC 服务器，我们可以像示例 7-15 中显示的那样通过跟踪来为客户端添加仪表。

##### 示例 7-15\. 为 gRPC 客户端添加仪表

```go
package main

import (
  "context"
  "log"
  "time"

  pb "productinfo/client/ecommerce"
  "productinfo/client/tracer"
  "google.golang.org/grpc"
  "go.opencensus.io/plugin/ocgrpc" ![1](img/1.png)
  "go.opencensus.io/trace"
  "contrib.go.opencensus.io/exporter/jaeger"

)

const (
  address = "localhost:50051"
)

func main() {
  tracer.initTracing() ![2](img/2.png)

  conn, err := grpc.Dial(address, grpc.WithInsecure())
  if err != nil {
     log.Fatalf("did not connect: %v", err)
  }
  defer conn.Close()
  c := pb.NewProductInfoClient(conn)

  ctx, span := trace.StartSpan(context.Background(),
          "ecommerce.ProductInfoClient") ![3](img/3.png)

  name := "Apple iphone 11"
  description := "Apple iphone 11 is the latest smartphone,
            launched in September 2019"
  price := float32(700.0)
  r, err := c.AddProduct(ctx, &pb.Product{Name: name,
      Description: description, Price: price}) ![5](img/5.png)
  if err != nil {
     log.Fatalf("Could not add product: %v", err)
  }
  log.Printf("Product ID: %s added successfully", r.Value)

  product, err := c.GetProduct(ctx, &pb.ProductID{Value: r.Value}) ![6](img/6.png)
  if err != nil {
    log.Fatalf("Could not get product: %v", err)
  }
  log.Printf("Product: ", product.String())
  span.End() ![4](img/4.png)

}
```

![1](img/#co_running_grpc_in_production_CO15-1)

导入 OpenTracing 和 Jaeger 库。

![2](img/#co_running_grpc_in_production_CO15-2)

调用`initTracing`函数并初始化 Jaeger 导出器实例，并注册到追踪器。

![3](img/#co_running_grpc_in_production_CO15-3)

使用 span 名称和上下文启动新的跨度。

![4](img/#co_running_grpc_in_production_CO15-6)

当一切都完成时停止跨度。

![5](img/#co_running_grpc_in_production_CO15-4)

通过传递新产品详细信息调用 addProduct 远程方法。

![6](img/#co_running_grpc_in_production_CO15-5)

通过传递 productID 调用 getProduct 远程方法。

一旦我们运行了服务器和客户端，追踪跨度将被发布到 Jaeger 代理，其中一个守护进程用作缓冲，将批处理处理和路由从客户端抽象出来。一旦 Jaeger 代理接收到来自客户端的追踪日志，它将其转发到收集器。收集器处理日志并存储它们。通过 Jaeger 服务器，我们可以可视化追踪。

从这里我们将结束可观察性的讨论。日志、度量和跟踪各自具有其独特的目的，最好在系统中启用这三个支柱，以获取对内部状态的最大可见性。

一旦您在生产环境中运行了基于 gRPC 的可观察应用程序，您可以随时观察其状态，并在出现问题或系统中断时轻松找到问题所在。在诊断系统问题时，找到解决方案、测试并尽快部署到生产环境非常重要。为了实现这个目标，您需要有良好的调试和故障排除机制。让我们深入了解 gRPC 应用程序的这些机制的细节。

# 调试和故障排除

调试和故障排除是查找问题根本原因并解决应用程序中出现问题的过程。为了调试和排除问题，我们首先需要在较低的环境（称为开发或测试环境）中重现相同的问题。因此，我们需要一套工具来生成类似的请求负载，就像在生产环境中一样。

这个过程在 gRPC 服务中相对较难，因为工具需要支持根据服务定义编码和解码消息，并且能够支持 HTTP/2。常见的工具如 curl 或 Postman，用于测试 HTTP 服务，不能用于测试 gRPC 服务。

但是有很多有趣的工具可用于调试和测试 gRPC 服务。您可以在[awesome gRPC repository](https://oreil.ly/Ki2aZ)中找到这些工具的列表。它包含了大量可用于 gRPC 的资源集合。调试 gRPC 应用程序的最常见方式之一是使用额外的日志记录。

## 启用额外的日志记录

我们可以启用额外的日志和跟踪来诊断您的 gRPC 应用程序问题。在 gRPC Go 应用程序中，我们可以通过设置以下环境变量来启用额外的日志：

```go
GRPC_GO_LOG_VERBOSITY_LEVEL=99 ![1](img/1.png)
GRPC_GO_LOG_SEVERITY_LEVEL=info ![2](img/2.png)
```

![1](img/#co_running_grpc_in_production_CO16-1)

*详细程度*意味着每五分钟任何单个信息消息应该打印多少次。默认情况下，详细程度设置为 0。

![2](img/#co_running_grpc_in_production_CO16-2)

将日志严重级别设置为 `info`。所有信息性消息将被打印出来。

在 gRPC Java 应用程序中，没有环境变量来控制日志级别。我们可以通过提供一个 *logging.properties* 文件来打开额外的日志，其中包含日志级别更改。假设我们想要在我们的应用程序中调试传输级别帧，我们可以在应用程序中创建一个新的 *logging.properties* 文件，并将较低的日志级别设置为特定的 Java 包（netty transport package）如下：

```go
handlers=java.util.logging.ConsoleHandler
io.grpc.netty.level=FINE
java.util.logging.ConsoleHandler.level=FINE
java.util.logging.ConsoleHandler.formatter=java.util.logging.SimpleFormatter
```

然后使用 JVM 标志启动 Java 二进制文件：

```go
-Djava.util.logging.config.file=logging.properties
```

一旦我们在应用程序中设置了较低的日志级别，所有日志级别等于或高于配置的日志级别的日志将在控制台/日志文件中打印出来。通过阅读日志，我们可以深入了解系统的状态。

通过以上内容，我们已经覆盖了在生产环境中运行 gRPC 应用程序时应知道的大部分内容。

# 概要

制作生产就绪的 gRPC 应用程序需要我们关注与应用程序开发相关的多个方面。我们首先通过设计服务契约并为服务或客户端生成代码来开始，然后实现我们服务的业务逻辑。一旦我们实现了服务，我们需要专注于以下几点，以使 gRPC 应用程序达到生产就绪状态。*测试* gRPC 服务器和客户端应用程序是必不可少的。

gRPC 应用程序的*部署*遵循标准的应用程序开发方法。对于本地和虚拟机部署，只需使用服务器或客户端程序的生成二进制文件。您可以将 gRPC 应用程序作为 Docker 容器运行，并找到用于在 Docker 上部署 Go 和 Java 应用程序的标准 Dockerfile 示例。在 Kubernetes 上运行 gRPC 类似于标准的 Kubernetes 部署。当您在 Kubernetes 上运行 gRPC 应用程序时，您会使用底层特性，如负载均衡、高可用性、入口控制器等。使 gRPC 应用程序可观察对于在生产中使用它们至关重要，当 gRPC 应用程序在生产中运行时，通常使用 gRPC 应用程序级指标。

在 gRPC 中最受欢迎的指标支持实现之一，即 gRPC-Prometheus 库中，我们在服务器和客户端端使用拦截器来收集指标，同时还使用拦截器启用 gRPC 日志记录。对于生产中的 gRPC 应用程序，您可能需要通过启用额外的日志记录来进行故障排除或调试。在接下来的章节中，我们将探讨一些对于构建 gRPC 应用程序很有用的 gRPC 生态系统组件。
