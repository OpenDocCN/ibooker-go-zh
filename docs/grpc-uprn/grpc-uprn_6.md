# 第六章 安全的 gRPC

基于 gRPC 的应用程序在网络上远程通信。这要求每个 gRPC 应用程序向需要与其通信的其他应用程序公开其入口点。从安全性的角度来看，这并不是一件好事。我们拥有的入口点越多，攻击面就越广，被攻击的风险就越高。因此，保护通信并保护入口点对于任何真实世界的使用案例都是至关重要的。每个 gRPC 应用程序必须能够处理加密消息，加密所有节点间的通信，并验证和签署所有消息等。

在本章中，我们将讨论一组安全基础知识和模式，以解决在启用应用级安全时面临的挑战。简单来说，我们将探讨如何保护微服务之间的通信通道，并验证和控制用户的访问权限。

所以让我们从保护通信通道开始。

# 使用 TLS 验证 gRPC 通道

*传输层安全* (TLS) 旨在为两个通信应用程序提供隐私和数据完整性保护。在这里，它关注的是为 gRPC 客户端和服务器应用程序之间提供安全连接。根据[传输层安全协议规范](https://oreil.ly/n4iIE)，当客户端和服务器之间的连接是安全的时，它应具有以下一个或多个属性：

连接是私密的

对数据加密使用对称加密技术。这是一种加密类型，其中仅使用一个密钥（秘密密钥）进行加密和解密。这些密钥根据会话开始时协商的共享秘密生成，对每个连接都是唯一的。

连接是可靠的

这是因为每条消息都包含消息完整性检查，以防止在传输过程中发生未检测到的数据丢失或更改。

因此，通过安全连接发送数据非常重要。使用 TLS 保护 gRPC 连接并不困难，因为这种认证机制已内置于 gRPC 库中。它还推广了 TLS 的使用，用于验证和加密交换。

那么，我们如何在 gRPC 连接中启用传输层安全呢？客户端和服务器之间的安全数据传输可以实现为单向或双向（也称为双向 TLS 或 mTLS）。在接下来的部分中，我们将讨论如何以这些方式启用安全性。

## 启用单向安全连接

在单向连接中，只有客户端验证服务器，以确保它从预期的服务器接收数据。在建立客户端和服务器之间的连接时，服务器会向客户端共享其公共证书，客户端然后验证接收到的证书。对于由证书颁发机构（CA）签名的证书，这是通过证书颁发机构（CA）完成的。一旦证书验证通过，客户端就使用密钥加密发送数据。

CA 是一个受信任的实体，负责管理和签发用于公共网络中安全通信的安全证书和公钥。由该受信任实体签署或颁发的证书称为 CA 签名证书。

要启用 TLS，首先我们需要创建以下证书和密钥：

`server.key`

用于签署和验证公共密钥的私有 RSA 密钥。

`server.pem/server.crt`

自签名的 X.509 公钥用于分发。

###### 注意

缩写 *RSA* 代表三位发明者的名字：Rivest、Shamir 和 Adleman。RSA 是最流行的公钥密码系统之一，在安全数据传输中被广泛使用。在 RSA 中，公钥（可被所有人知道）用于加密数据。然后使用私钥解密数据。其思想是只有使用私钥才能在合理的时间内解密用公钥加密的消息。

要生成这些密钥，我们可以使用 OpenSSL 工具，这是一个开源工具包，用于 TLS 和安全套接字层（SSL）协议。它支持生成不同大小和密码短语的私钥、公共证书等。还有其他工具如 [mkcert](https://mkcert.dev) 和 [certstrap](https://oreil.ly/Mu4Q6)，也可以用来轻松生成密钥和证书。

我们不会在此描述如何生成自签名证书的密钥，因为在源代码仓库的 README 文件中已经详细描述了生成这些密钥和证书的逐步详细过程。

假设我们已经创建了私钥和公共证书。让我们使用它们，为我们在第 1 和 2 章讨论的在线产品管理系统中的 gRPC 服务器和客户端之间的通信进行安全保护。

### 启用 gRPC 服务器中的单向安全连接

这是加密客户端和服务器通信的最简单方法。这里服务器需要用公共/私有密钥对初始化。我们将解释如何在我们的 gRPC Go 服务器中执行这个过程。

要在安全 Go 服务器中启用安全连接，让我们根据 示例 6-1 更新服务器实现的主函数。

##### 示例 6-1\. 用于托管 ProductInfo 服务的 gRPC 安全服务器实现

```go
package main

import (
  "crypto/tls"
  "errors"
  pb "productinfo/server/ecommerce"
  "google.golang.org/grpc"
  "google.golang.org/grpc/credentials"
  "log"
  "net"
)

var (
  port = ":50051"
  crtFile = "server.crt"
  keyFile = "server.key"
)

func main() {
  cert, err := tls.LoadX509KeyPair(crtFile,keyFile) ![1](img/1.png)
  if err != nil {
     log.Fatalf("failed to load key pair: %s", err)
  }
  opts := []grpc.ServerOption{
     grpc.Creds(credentials.NewServerTLSFromCert(&cert)) ![2](img/2.png)
  }

  s := grpc.NewServer(opts...) ![3](img/3.png)
  pb.RegisterProductInfoServer(s, &server{}) ![4](img/4.png)

  lis, err := net.Listen("tcp", port) ![5](img/5.png)
  if err != nil {
     log.Fatalf("failed to listen: %v", err)
  }

  if err := s.Serve(lis); err != nil { ![6](img/6.png)
     log.Fatalf("failed to serve: %v", err)
  }
}
```

![1](img/#co_secured_grpc_CO1-1)

读取和解析公共/私有密钥对，并创建证书以启用 TLS。

![2](img/#co_secured_grpc_CO1-2)

通过添加证书作为 TLS 服务器凭据，为所有传入连接启用 TLS。

![3](img/#co_secured_grpc_CO1-3)

通过传递 TLS 服务器凭据创建一个新的 gRPC 服务器实例。

![4](img/#co_secured_grpc_CO1-4)

通过调用生成的 API，将实现的服务注册到新创建的 gRPC 服务器。

![5](img/#co_secured_grpc_CO1-5)

在端口（50051）上创建一个 TCP 监听器。

![6](img/#co_secured_grpc_CO1-6)

将 gRPC 服务器绑定到侦听器并开始侦听端口（50051）上的传入消息。

现在我们已经修改了服务器，使其接受可以验证服务器证书的客户端的请求。让我们修改我们的客户端代码，与这个服务器通信。

### 在 gRPC 客户端启用单向安全连接

为了使客户端连接成功，客户端需要服务器的自签名公钥。我们可以修改我们的 Go 客户端代码，以连接示例 6-2 中的服务器。

##### 示例 6-2. gRPC 安全客户端应用程序

```go
package main

import (
  "log"

  pb "productinfo/server/ecommerce"
  "google.golang.org/grpc/credentials"
  "google.golang.org/grpc"
)

var (
  address = "localhost:50051"
  hostname = "localhost
  crtFile = "server.crt"
)

func main() {
  creds, err := credentials.NewClientTLSFromFile(crtFile, hostname) ![1](img/1.png) if err != nil {
     log.Fatalf("failed to load credentials: %v", err)
  }
  opts := []grpc.DialOption{
     grpc.WithTransportCredentials(creds), ![2](img/2.png) }

  conn, err := grpc.Dial(address, opts...) ![3](img/3.png) if err != nil {
     log.Fatalf("did not connect: %v", err)
  }
  defer conn.Close() ![5](img/5.png)
  c := pb.NewProductInfoClient(conn) ![4](img/4.png)

  .... // Skip RPC method invocation. }
```

![1](img/#co_secured_grpc_CO2-1)

读取和解析公共证书并创建证书以启用 TLS。

![2](img/#co_secured_grpc_CO2-2)

将传输凭证添加为 `DialOption`。

![3](img/#co_secured_grpc_CO2-3)

使用拨号选项与服务器建立安全连接。

![4](img/#co_secured_grpc_CO2-5)

传递连接并创建存根。该存根实例包含所有远程方法，以调用服务器。

![5](img/#co_secured_grpc_CO2-4)

当一切都完成时，关闭连接。

这是一个相当简单直接的过程。我们只需从原始代码中添加三行并修改一行。首先，我们从服务器公钥文件创建一个凭证对象，然后将传输凭证传递给 gRPC 拨号器。这将在每次客户端建立与服务器之间的连接时启动 TLS 握手。

在单向 TLS 中，我们只验证服务器身份。让我们在下一节中同时验证双方（客户端和服务器）。

## 启用 mTLS 安全连接

客户端和服务器之间的 mTLS 连接的主要目的是控制连接到服务器的客户端。与单向 TLS 连接不同，服务器配置为接受来自一组经过验证的客户端的连接。在这种连接中，双方都彼此分享其公共证书并验证对方。连接的基本流程如下：

1.  客户端发送请求以访问服务器上的受保护信息。

1.  服务器向客户端发送其 X.509 证书。

1.  客户端通过 CA 验证接收到的证书以验证 CA 签名的证书。

1.  如果验证成功，客户端将其证书发送给服务器。

1.  服务器还通过 CA 验证客户端的证书。

1.  一旦成功，服务器允许访问受保护的数据。

要在我们的示例中启用 mTLS，我们需要解决如何处理客户端和服务器证书的问题。我们需要创建一个带有自签名证书的 CA，为客户端和服务器分别创建证书签名请求，并使用我们的 CA 签名它们。与之前的单向安全连接一样，我们可以使用 OpenSSL 工具生成密钥和证书。

假设我们拥有所有必需的证书来启用客户端与服务器的 mTLS 通信。如果您正确生成了它们，您的工作空间中将创建以下密钥和证书：

server.key

服务器的私有 RSA 密钥。

server.crt

服务器的公共证书。

client.key

客户端的私有 RSA 密钥。

client.crt

客户端的公共证书。

ca.crt

用于签署所有公共证书的 CA 的公共证书。

让我们首先修改示例中的服务器代码，直接创建 X.509 密钥对，并基于 CA 公钥创建证书池。

### 在 gRPC 服务器中启用 mTLS

在 Go 服务器中启用 mTLS，请按照示例 6-3 中显示的方式更新服务器实现的主函数。

##### 示例 6-3\. 用于在 Go 中托管 ProductInfo 服务的 gRPC 安全服务器实现

```go
package main

import (
  "crypto/tls"
  "crypto/x509"
  "errors"
  pb "productinfo/server/ecommerce"
  "google.golang.org/grpc"
  "google.golang.org/grpc/credentials"
  "io/ioutil"
  "log"
  "net"
)

var (
  port = ":50051"
  crtFile = "server.crt"
  keyFile = "server.key"
  caFile = "ca.crt"
)

func main() {
  certificate, err := tls.LoadX509KeyPair(crtFile, keyFile) ![1](img/1.png)
  if err != nil {
     log.Fatalf("failed to load key pair: %s", err)
  }

  certPool := x509.NewCertPool() ![2](img/2.png)
  ca, err := ioutil.ReadFile(caFile)
  if err != nil {
     log.Fatalf("could not read ca certificate: %s", err)
  }

  if ok := certPool.AppendCertsFromPEM(ca); !ok { ![3](img/3.png)
     log.Fatalf("failed to append ca certificate")
  }

  opts := []grpc.ServerOption{
     // Enable TLS for all incoming connections.
     grpc.Creds( ![4](img/4.png)
        credentials.NewTLS(&tls.Config {
           ClientAuth:   tls.RequireAndVerifyClientCert,
           Certificates: []tls.Certificate{certificate},
           ClientCAs:    certPool,
           },
        )),
  }

  s := grpc.NewServer(opts...) ![5](img/5.png)
  pb.RegisterProductInfoServer(s, &server{}) ![6](img/6.png)

  lis, err := net.Listen("tcp", port) ![7](img/7.png)
  if err != nil {
     log.Fatalf("failed to listen: %v", err)
  }

  if err := s.Serve(lis); err != nil { ![8](img/8.png)
     log.Fatalf("failed to serve: %v", err)
  }
}
```

![1](img/#co_secured_grpc_CO3-1)

直接从服务器证书和密钥创建 X.509 密钥对。

![2](img/#co_secured_grpc_CO3-2)

从 CA 创建证书池。

![3](img/#co_secured_grpc_CO3-3)

将来自 CA 的客户端证书附加到证书池。

![4](img/#co_secured_grpc_CO3-4)

通过创建 TLS 凭据为所有传入连接启用 TLS。

![5](img/#co_secured_grpc_CO3-5)

通过传递 TLS 服务器凭据创建新的 gRPC 服务器实例。

![6](img/#co_secured_grpc_CO3-6)

通过调用生成的 API 将 gRPC 服务注册到新创建的 gRPC 服务器。

![7](img/#co_secured_grpc_CO3-7)

在端口（50051）上创建 TCP 监听器。

![8](img/#co_secured_grpc_CO3-8)

将 gRPC 服务器绑定到侦听器，并开始在端口（50051）上监听传入的消息。

现在我们已经修改了服务器以接受来自验证客户端的请求。让我们修改我们的客户端代码以与此服务器通信。

### 在 gRPC 客户端中启用 mTLS

为了使客户端连接，客户端需要按照与服务器类似的步骤进行修改。我们可以修改我们的 Go 客户端代码，如示例 6-4 所示。

##### 示例 6-4\. 在 Go 中的 gRPC 安全客户端应用程序

```go
package main

import (
  "crypto/tls"
  "crypto/x509"
  "io/ioutil"
  "log"

  pb "productinfo/server/ecommerce"
  "google.golang.org/grpc"
  "google.golang.org/grpc/credentials"
)

var (
  address = "localhost:50051"
  hostname = "localhost"
  crtFile = "client.crt"
  keyFile = "client.key"
  caFile = "ca.crt"
)

func main() {
  certificate, err := tls.LoadX509KeyPair(crtFile, keyFile) ![1](img/1.png)
  if err != nil {
     log.Fatalf("could not load client key pair: %s", err)
  }

  certPool := x509.NewCertPool() ![2](img/2.png)
  ca, err := ioutil.ReadFile(caFile)
  if err != nil {
     log.Fatalf("could not read ca certificate: %s", err)
  }

  if ok := certPool.AppendCertsFromPEM(ca); !ok { ![3](img/3.png)
     log.Fatalf("failed to append ca certs")
  }

  opts := []grpc.DialOption{
     grpc.WithTransportCredentials( credentials.NewTLS(&tls.Config{ ![4](img/4.png)
        ServerName:   hostname, // NOTE: this is required!
        Certificates: []tls.Certificate{certificate},
        RootCAs:      certPool,
     })),
  }

  conn, err := grpc.Dial(address, opts...) ![5](img/5.png)
  if err != nil {
     log.Fatalf("did not connect: %v", err)
  }
  defer conn.Close()![7](img/7.png)
  c := pb.NewProductInfoClient(conn) ![6](img/6.png)

  .... // Skip RPC method invocation. }
```

![1](img/#co_secured_grpc_CO4-1)

直接从服务器证书和密钥创建 X.509 密钥对。

![2](img/#co_secured_grpc_CO4-2)

从 CA 创建证书池。

![3](img/#co_secured_grpc_CO4-3)

将来自 CA 的客户端证书附加到证书池。

![4](img/#co_secured_grpc_CO4-4)

将传输凭据作为连接选项添加。这里的 `ServerName` 必须等于证书上的 `Common Name`。

![5](img/#co_secured_grpc_CO4-5)

通过传递选项与服务器建立安全连接。

![6](img/#co_secured_grpc_CO4-7)

将连接传递并创建一个存根。该存根实例包含调用服务器的所有远程方法。

![7](img/#co_secured_grpc_CO4-6)

在一切完成后关闭连接。

现在，我们已经使用基本的单向 TLS 和 mTLS 保护了 gRPC 应用程序的客户端和服务器之间的通信通道。下一步是在每个调用的基础上启用身份验证，这意味着凭据将附加到调用中。每个客户端调用都有身份验证凭据，服务器端检查调用的凭据，并决定是否允许客户端调用或拒绝。

# 验证 gRPC 调用

gRPC 设计用于使用严格的身份验证机制。在前一节中，我们介绍了如何使用 TLS 加密客户端和服务器之间交换的数据。现在，我们将讨论如何验证调用者的身份，并使用不同的调用凭据技术（如基于令牌的身份验证等）应用访问控制。

为了方便验证调用者，gRPC 提供了客户端在每个调用中注入其凭据（如用户名和密码）的能力。gRPC 服务器能够拦截来自客户端的请求，并为每个传入的调用检查这些凭据。

首先，我们将回顾一个简单的身份验证场景，以解释每个客户端调用如何工作的身份验证。

## 使用基本身份验证

*基本身份验证* 是最简单的身份验证机制。在这种机制中，客户端发送带有 `Authorization` 头的请求，该头的值以单词 `Basic` 开头，后跟一个空格和一个 base64 编码的字符串 `username:password`。例如，如果用户名是 `admin`，密码是 `admin`，则头的值如下所示：

```go
Authorization: Basic YWRtaW46YWRtaW4=
```

通常情况下，gRPC 不鼓励我们使用用户名/密码来验证服务。这是因为与其他令牌（JSON Web Tokens [JWTs]、OAuth2 访问令牌）相比，用户名/密码在时间上没有控制。这意味着当我们生成一个令牌时，我们可以指定其有效期。但是对于用户名/密码，我们无法指定有效期。它的有效性会一直持续，直到我们修改密码。如果您需要在应用程序中启用基本身份验证，建议在客户端和服务器之间的安全连接中共享基本凭据。我们选择基本身份验证是因为这样更容易解释 gRPC 中身份验证的工作原理。

让我们首先讨论如何将用户凭据（在基本身份验证中）注入到调用中。由于 gRPC 中没有基本身份验证的内置支持，我们需要将其作为自定义凭据添加到客户端上下文中。在 Go 中，我们可以通过定义一个凭据结构体并实现 `PerRPCCredentials` 接口来轻松实现此操作，如 示例 6-5 所示。

##### 示例 6-5. 实现 PerRPCCredentials 接口以传递自定义凭据

```go
type basicAuth struct { ![1](img/1.png)
  username string
  password string
}

func (b basicAuth) GetRequestMetadata(ctx context.Context,
  in ...string)  (map[string]string, error) { ![2](img/2.png)
  auth := b.username + ":" + b.password
  enc := base64.StdEncoding.EncodeToString([]byte(auth))
  return map[string]string{
     "authorization": "Basic " + enc,
  }, nil
}

func (b basicAuth) RequireTransportSecurity() bool { ![3](img/3.png)
  return true
}
```

![1](img/#co_secured_grpc_CO5-1)

定义一个结构体来保存您想要在 RPC 调用中注入的字段集合（在我们的情况下，这些是像用户名和密码这样的用户凭据）。

![2](img/#co_secured_grpc_CO5-2)

实现`GetRequestMetadata`方法并将用户凭据转换为请求元数据。在我们的情况下，“Authorization”是键，其值是“Basic”后跟 base64 编码（<用户名>:<密码>）。

![3](img/#co_secured_grpc_CO5-3)

指定是否需要通过这些凭据传递通道安全性。如前所述，建议使用通道安全性。

一旦实现了凭据对象，我们需要使用有效的凭据初始化它，并在创建连接时传递，如示例 6-6 所示。

##### 示例 6-6\. 带有基本身份验证的 gRPC 安全客户端应用程序

```go
package main

import (
  "log"
  pb "productinfo/server/ecommerce"
  "google.golang.org/grpc/credentials"
  "google.golang.org/grpc"
)

var (
  address = "localhost:50051"
  hostname = "localhost"
  crtFile = "server.crt"
)

func main() {
  creds, err := credentials.NewClientTLSFromFile(crtFile, hostname)
  if err != nil {
     log.Fatalf("failed to load credentials: %v", err)
  }

  auth := basicAuth{ ![1](img/1.png)
    username: "admin",
    password: "admin",
  }

  opts := []grpc.DialOption{
     grpc.WithPerRPCCredentials(auth), ![2](img/2.png)
     grpc.WithTransportCredentials(creds),
  }

  conn, err := grpc.Dial(address, opts...)
  if err != nil {
     log.Fatalf("did not connect: %v", err)
  }
  defer conn.Close()
  c := pb.NewProductInfoClient(conn)

  .... // Skip RPC method invocation. }
```

![1](img/#co_secured_grpc_CO6-1)

使用有效的用户凭据（用户名和密码）初始化`auth`变量。`auth`变量保存我们将要使用的值。

![2](img/#co_secured_grpc_CO6-2)

将`auth`变量传递给`grpc.WithPerRPCCredentials`函数。`grpc.WithPerRPCCredentials()`函数接受一个接口作为参数。由于我们定义了符合该接口的认证结构，我们可以传递我们的变量。

现在客户端在其调用期间推送额外的元数据，但服务器不关心。因此，我们需要告诉服务器检查元数据。让我们更新我们的服务器以读取如示例 6-7 所示的元数据。

##### 示例 6-7\. 使用基本用户凭据验证的 gRPC 安全服务器

```go
package main

import (
  "context"
  "crypto/tls"
  "encoding/base64"
  "errors"
  pb "productinfo/server/ecommerce"
  "google.golang.org/grpc"
  "google.golang.org/grpc/codes"
  "google.golang.org/grpc/credentials"
  "google.golang.org/grpc/metadata"
  "google.golang.org/grpc/status"
  "log"
  "net"
  "path/filepath"
  "strings"
)

var (
  port = ":50051"
  crtFile = "server.crt"
  keyFile = "server.key"
  errMissingMetadata = status.Errorf(codes.InvalidArgument, "missing metadata")
  errInvalidToken    = status.Errorf(codes.Unauthenticated, "invalid credentials")
)

type server struct {
  productMap map[string]*pb.Product
}

func main() {
  cert, err := tls.LoadX509KeyPair(crtFile, keyFile)
  if err != nil {
     log.Fatalf("failed to load key pair: %s", err)
  }
  opts := []grpc.ServerOption{
     // Enable TLS for all incoming connections.
     grpc.Creds(credentials.NewServerTLSFromCert(&cert)),

     grpc.UnaryInterceptor(ensureValidBasicCredentials), ![1](img/1.png)
  }

  s := grpc.NewServer(opts...)
  pb.RegisterProductInfoServer(s, &server{})

  lis, err := net.Listen("tcp", port)
  if err != nil {
     log.Fatalf("failed to listen: %v", err)
  }

  if err := s.Serve(lis); err != nil {
     log.Fatalf("failed to serve: %v", err)
  }
}

func valid(authorization []string) bool {
  if len(authorization) < 1 {
     return false
  }
  token := strings.TrimPrefix(authorization[0], "Basic ")
  return token == base64.StdEncoding.EncodeToString([]byte("admin:admin"))
}

func ensureValidBasicCredentials(ctx context.Context, req interface{}, info
*grpc.UnaryServerInfo,
     handler grpc.UnaryHandler) (interface{}, error) { ![2](img/2.png)
  md, ok := metadata.FromIncomingContext(ctx) ![3](img/3.png)
  if !ok {
     return nil, errMissingMetadata
  }
  if !valid(md["authorization"]) {
     return nil, errInvalidToken
  }
  // Continue execution of handler after ensuring a valid token.
  return handler(ctx, req)
}
```

![1](img/#co_secured_grpc_CO7-1)

使用 TLS 服务器证书初始化新的服务器选项（`grpc.ServerOption`）。`grpc.UnaryInterceptor`是一个函数，我们在其中添加拦截器以拦截来自客户端的所有请求。我们将引用传递给一个函数（`ensureValidBasicCredentials`），因此拦截器将所有客户端请求传递给该函数。

![2](img/#co_secured_grpc_CO7-2)

定义一个名为`ensureValidBasicCredentials`的函数来验证调用者身份。在这里，`context.Context`对象包含我们需要的元数据，并且在请求的生命周期内将存在。

![3](img/#co_secured_grpc_CO7-3)

从上下文中提取元数据，获取`authentication`的值，并验证凭据。`metadata.MD`中的键被规范化为小写。因此，我们需要检查小写键的值。

现在服务器在每次调用中验证客户端身份。这是一个非常简单的例子。您可以在服务器拦截器内部具有复杂的认证逻辑来验证客户端身份。

既然你已经对客户端认证工作原理有基本了解，让我们来讨论常用和推荐的基于令牌的认证（OAuth 2.0）。

## 使用 OAuth 2.0

[OAuth 2.0](https://oauth.net/2) 是一个访问委托框架。它允许用户代表他们授予对服务的有限访问权限，而不像使用用户名和密码那样给予他们总访问权限。在这里我们不打算详细讨论 OAuth 2.0。如果您对 OAuth 2.0 的工作原理有一些基本的了解，则有助于在您的应用程序中启用它。

###### 注意

在 OAuth 2.0 流程中，有四个主要角色：客户端、授权服务器、资源服务器和资源所有者。*客户端*希望访问资源服务器中的资源。为了访问资源，客户端需要从*授权服务器*获取一个令牌（这是一个任意字符串）。这个令牌必须是适当长度的，并且不可预测。一旦客户端收到令牌，客户端可以使用这个令牌向*资源服务器*发送请求。资源服务器随后与相应的授权服务器交流并验证令牌。如果这个*资源所有者*验证通过，客户端就可以访问资源。

gRPC 具有内置支持，可以在 gRPC 应用程序中启用 OAuth 2.0。让我们首先讨论如何将令牌注入到调用中。由于我们的示例中没有授权服务器，我们将为令牌值硬编码一个任意的字符串。示例 6-8 展示了如何向客户端请求添加 OAuth 令牌。

##### 示例 6-8\. 在 Go 中使用 OAuth 令牌的 gRPC 安全客户端应用程序

```go
package main

import (
  "google.golang.org/grpc/credentials"
  "google.golang.org/grpc/credentials/oauth"
  "log"

  pb "productinfo/server/ecommerce"
  "golang.org/x/oauth2"
  "google.golang.org/grpc"
)

var (
  address = "localhost:50051"
  hostname = "localhost"
  crtFile = "server.crt"
)

func main() {
  auth := oauth.NewOauthAccess(fetchToken()) ![1](img/1.png)

  creds, err := credentials.NewClientTLSFromFile(crtFile, hostname)
  if err != nil {
     log.Fatalf("failed to load credentials: %v", err)
  }

  opts := []grpc.DialOption{
     grpc.WithPerRPCCredentials(auth), ![2](img/2.png)
     grpc.WithTransportCredentials(creds),
  }

  conn, err := grpc.Dial(address, opts...)
  if err != nil {
     log.Fatalf("did not connect: %v", err)
  }
  defer conn.Close()
  c := pb.NewProductInfoClient(conn)

  .... // Skip RPC method invocation. }

func fetchToken() *oauth2.Token {
  return &oauth2.Token{
     AccessToken: "some-secret-token",
  }
}
```

![1](img/#co_secured_grpc_CO8-1)

设置连接的凭据。我们需要提供一个 OAuth2 令牌值来创建凭据。这里我们使用一个硬编码的字符串值作为令牌。

![2](img/#co_secured_grpc_CO8-2)

配置 gRPC 拨号选项，以在同一连接上的所有 RPC 调用中应用单个 OAuth 令牌。如果要每次调用应用一个 OAuth 令牌，则需要使用 `CallOption` 配置 gRPC 调用。

注意，我们还启用了通道安全，因为 OAuth 要求底层传输是安全的。在 gRPC 内部，提供的令牌被前缀为令牌类型，并附加到具有键`authorization`的元数据中。

在服务器端，我们添加了类似的拦截器来检查和验证客户端请求中带有的令牌，如示例 6-9 所示。

##### 示例 6-9\. 使用 OAuth 用户令牌验证的 gRPC 安全服务器

```go
package main

import (
  "context"
  "crypto/tls"
  "errors"
  "log"
  "net"
  "strings"

  pb "productinfo/server/ecommerce"
  "google.golang.org/grpc"
  "google.golang.org/grpc/codes"
  "google.golang.org/grpc/credentials"
  "google.golang.org/grpc/metadata"
  "google.golang.org/grpc/status"
)

// server is used to implement ecommerce/product_info. type server struct {
  productMap map[string]*pb.Product
}

var (
  port = ":50051"
  crtFile = "server.crt"
  keyFile = "server.key"
  errMissingMetadata = status.Errorf(codes.InvalidArgument, "missing metadata")
  errInvalidToken    = status.Errorf(codes.Unauthenticated, "invalid token")
)

func main() {
  cert, err := tls.LoadX509KeyPair(crtFile, keyFile)
  if err != nil {
     log.Fatalf("failed to load key pair: %s", err)
  }
  opts := []grpc.ServerOption{
     grpc.Creds(credentials.NewServerTLSFromCert(&cert)),
     grpc.UnaryInterceptor(ensureValidToken), ![1](img/1.png)
  }

  s := grpc.NewServer(opts...)
  pb.RegisterProductInfoServer(s, &server{})

  lis, err := net.Listen("tcp", port)
  if err != nil {
     log.Fatalf("failed to listen: %v", err)
  }

  if err := s.Serve(lis); err != nil {
     log.Fatalf("failed to serve: %v", err)
  }
}

func valid(authorization []string) bool {
  if len(authorization) < 1 {
     return false
  }
  token := strings.TrimPrefix(authorization[0], "Bearer ")
  return token == "some-secret-token"
}

func ensureValidToken(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo,
     handler grpc.UnaryHandler) (interface{}, error) { ![2](img/2.png)
  md, ok := metadata.FromIncomingContext(ctx)
  if !ok {
     return nil, errMissingMetadata
  }
  if !valid(md["authorization"]) {
     return nil, errInvalidToken
  }
  return handler(ctx, req)
}
```

![1](img/#co_secured_grpc_CO9-1)

添加新的服务器选项（`grpc.ServerOption`）以及 TLS 服务器证书。通过 `grpc.UnaryInterceptor` 函数，我们添加一个拦截器来拦截来自客户端的所有请求。

![2](img/#co_secured_grpc_CO9-2)

定义一个名为`ensureValidToken`的函数来验证令牌。如果令牌丢失或无效，拦截器将阻止执行并返回错误。否则，拦截器调用下一个处理程序，传递上下文和接口。

可以使用拦截器为所有 RPC 配置令牌验证。服务器可以根据服务类型配置 `grpc.UnaryInterceptor` 或 `grpc.StreamInterceptor`。

类似于 OAuth 2.0 认证，gRPC 也支持基于 JSON Web Token（JWT）的认证。在接下来的章节中，我们将讨论如何进行配置以启用基于 JWT 的认证。

## 使用 JWT

JWT 定义了一个容器，用于在客户端和服务器之间传输身份信息。签名的 JWT 可以用作自包含的访问令牌，这意味着资源服务器无需与认证服务器交互来验证客户端令牌。它可以通过验证签名来验证令牌。客户端从认证服务器请求访问权限，认证服务器验证客户端凭据，创建一个 JWT 并将其发送给客户端。带有 JWT 的客户端应用允许访问资源。

gRPC 对 JWT 有内置支持。如果您从认证服务器获取了 JWT 文件，则需要传递该令牌文件并创建 JWT 凭据。示例 6-10 中的代码片段演示了如何从 JWT 令牌文件（*token.json*）创建 JWT 凭据，并将其作为 `DialOptions` 传递给 Go 客户端应用程序。

##### 示例 6-10\. 在 Go 客户端应用程序中使用 JWT 设置连接

```go
jwtCreds, err := oauth.NewJWTAccessFromFile(“token.json”) ![1](img/1.png)
if err != nil {
  log.Fatalf("Failed to create JWT credentials: %v", err)
}

creds, err := credentials.NewClientTLSFromFile("server.crt",
     "localhost")
if err != nil {
    log.Fatalf("failed to load credentials: %v", err)
}
opts := []grpc.DialOption{
  grpc.WithPerRPCCredentials(jwtCreds),
  // transport credentials.
  grpc.WithTransportCredentials(creds), ![2](img/2.png)
}

// Set up a connection to the server. conn, err := grpc.Dial(address, opts...)
if err != nil {
  log.Fatalf("did not connect: %v", err)
}
  .... // Skip Stub generation and RPC method invocation.
```

![1](img/#co_secured_grpc_CO10-1)

调用 `oauth.NewJWTAccessFromFile` 来初始化 `credentials.PerRPCCredentials`。我们需要提供一个有效的令牌文件来创建凭据。

![2](img/#co_secured_grpc_CO10-2)

使用 `DialOption WithPerRPCCredentials` 配置 gRPC 拨号，以在同一连接上为所有 RPC 调用应用 JWT 令牌。

除了这些认证技术外，我们还可以通过在客户端扩展 RPC 凭据并在服务器端添加新的拦截器来添加任何认证机制。gRPC 还为调用部署在 Google Cloud 中的 gRPC 服务提供了特殊的内置支持。在接下来的章节中，我们将讨论如何调用这些服务。

## 使用 Google 基于令牌的认证

用户身份验证以及决定是否允许他们使用部署在 Google 云平台上的服务是由可扩展服务代理（ESP）控制的。ESP 支持多种认证方法，包括 Firebase、Auth0 和 Google ID 令牌。在每种情况下，客户端需要在其请求中提供有效的 JWT。为了生成认证 JWT，我们必须为每个部署的服务创建一个服务账号。

一旦我们获取了服务的 JWT 令牌，我们可以在请求中发送令牌来调用服务方法。我们可以创建通道并传递凭据，如 示例 6-11 所示。

##### 示例 6-11\. 在 Go 客户端应用程序中设置与 Google 终端点的连接

```go
perRPC, err := oauth.NewServiceAccountFromFile("service-account.json", scope) ![1](img/1.png)
if err != nil {
  log.Fatalf("Failed to create JWT credentials: %v", err)
}

pool, _ := x509.SystemCertPool()
creds := credentials.NewClientTLSFromCert(pool, "")

opts := []grpc.DialOption{
  grpc.WithPerRPCCredentials(perRPC),
  grpc.WithTransportCredentials(creds), ![2](img/2.png)
}

conn, err := grpc.Dial(address, opts...)
if err != nil {
  log.Fatalf("did not connect: %v", err)
}
.... // Skip Stub generation and RPC method invocation.
```

![1](img/#co_secured_grpc_CO11-1)

调用 `oauth.NewServiceAccountFromFile` 来初始化 `credentials.PerRPCCredentials`。我们需要提供一个有效的令牌文件来创建凭证。

![2](img/#co_secured_grpc_CO11-2)

与之前讨论的认证机制类似，我们配置了一个 gRPC 连接的 `DialOption WithPerRPCCredentials`，以便将认证令牌作为元数据应用于同一连接上的所有 RPC 调用。

# 摘要

在构建一个生产就绪的 gRPC 应用程序时，至少需要满足 gRPC 应用程序的最低安全要求，以确保客户端和服务器之间的安全通信。gRPC 库设计用于与不同类型的认证机制一起工作，并能够通过添加自定义认证机制来扩展支持。这使得安全地使用 gRPC 与其他系统进行通信变得简单。

gRPC 支持两种类型的凭证：通道凭证和调用凭证。通道凭证附加到通道上，如 TLS 等。调用凭证附加到调用上，如 OAuth 2.0 令牌、基本认证等。我们甚至可以将这两种凭证类型应用到 gRPC 应用程序中。例如，我们可以启用 TLS 连接客户端和服务器之间的连接，并且还可以将凭证附加到在该连接上进行的每个 RPC 调用。

在本章中，您学习了如何为您的 gRPC 应用程序启用这两种凭证类型。在下一章中，我们将扩展您学到的概念和技术，以便在生产环境中构建和运行真实的 gRPC 应用程序。我们还将讨论如何为服务和客户端应用程序编写测试用例，如何在 Docker 和 Kubernetes 上部署应用程序，以及在生产环境中运行时如何观察系统。
