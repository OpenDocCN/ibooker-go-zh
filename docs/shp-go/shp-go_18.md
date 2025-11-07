# 附录 D. 使用 Terraform

基础设施即代码不是一个新概念，但多年来，工具已经从物理硬件上的脚本和镜像转变为在云中工作的工具。Kubernetes 和无服务器平台已经将其他工具的聚光灯从构建这些不同云环境中的基础设施的工具上移开。所有不同类型的部署都有其权衡和好处。在本附录中，提供了最后一个工具的示例，以给您提供一个关于部署景观的广泛概述。

## D.1 构建镜像

我的第一份工作是在高中时的 IT 部门。在那里工作期间，我能够协助拆箱和布线计算机实验室。当所有计算机都拆箱并插上电源后，我会把它们全部打开，快速敲击键盘以启用网络模式。几分钟内，我会看到安装程序开始运行，一个小时后，我们就有了 20 台完全相同的机器。每周我都会带着惊奇重复这个魔法。

那种惊奇感还没有消失。今天我们需要在云中使用我们的应用程序执行同样的任务。我们不是拆箱物理服务器，而是将我们的镜像推送到临时服务器上。在云中创建服务器时，它是在更大的服务器集群上的一个虚拟机。它可以关闭，然后在建筑物的完全不同的部分，甚至是在完全不同的机器上重新启动。

要构建镜像，我们将使用 Packer（[`www.packer.io/`](https://www.packer.io/）），这是 HashiCorp 的一个工具，它根据提供的规范构建服务器镜像。然后我们将使用以下列表中的代码创建一个 Packer 镜像定义。

列表 D.1 `hello-api.pkr.hcl`

```
variable "project_id" {                         ❶
  type    = string
}

variable "git_sha" {
  type    = string
  default = "UNKNOWN"
}

source "googlecompute" "hello-api" {            ❷
  project_id = ${var.project_id}
  source_image_family = "ubuntu-2204-lts"
  image_name = "hello-api-${var.git_sha}"       ❸
  ssh_username = "packer"
  zone = "us-central1-a"
}

build {                                         ❹
  sources = [sources.googlecompute.hello]

  provisioner "file" {
    destination = "/home/ubuntu/hello-api"
    source      = "api"
  }

  post-processor "manifest" {
    output = "manifest.json"
    strip_path = true
    custom_data = {
      sha = "${var.git_sha}"
    }
  }
}
```

❶ 构建所需输入变量

❷ 我们将构建的基础镜像

❸ 我们将要创建的镜像名称

❹ 特殊构建器，用于将我们的二进制文件复制到镜像中

这将构建一个以合并的提交名称命名的镜像。接下来，我们需要调整我们的账户以能够部署镜像。运行以下列表中的命令。

列表 D.2 `hello-api.pkr.hcl`

```
gcloud projects add-iam-policy-binding YOUR_GCP_PROJECT \
    --member=serviceAccount:GITHUB_SERVICE_ACCOUNT_NAME@YOUR_GCP_PROJECT.
    ➥iam.gserviceaccount.com \
    --role=roles/compute.instanceAdmin.v1

gcloud projects add-iam-policy-binding YOUR_GCP_PROJECT \
    --member=serviceAccount:GITHUB_SERVICE_ACCOUNT_NAME@YOUR_GCP_PROJECT.
    ➥iam.gserviceaccount.com \
    --role=roles/iam.serviceAccountUser

gcloud projects add-iam-policy-binding YOUR_GCP_PROJECT \
    --member=serviceAccount:GITHUB_SERVICE_ACCOUNT_NAME@YOUR_GCP_PROJECT.
    ➥iam.gserviceaccount.com \
    --role=roles/iap.tunnelResourceAccessor
```

这将允许我们的 GitHub 管道使用 Packer 构建镜像。接下来，我们将编写服务器以运行镜像。

## D.2 部署镜像

为了定义我们的基础设施，我们使用 Terraform（[https://www.terraform.io/](https://www.terraform.io/）），这是另一个 HashiCorp 的工具。Terraform 提供了一种定义服务器、负载均衡器、数据库等语言。它还跟踪您基础设施的状态，这在您需要更改或删除服务时将非常重要。我们需要在本地安装 Terraform，以便我们可以设置一些必要的组件。首先，创建一个名为`infra`的目录，然后创建一个名为`global`的子目录。在这些目录中，我们将以下列表中的代码添加到`main.tf`文件中。

列表 D.3 `main.tf`

```
resource "google_storage_bucket" "default" {    ❶
  name          = "hello-api-bucket-tfstate"
  force_destroy = false
  location      = "US"
  storage_class = "STANDARD"
  versioning {
    enabled = true
  }
}
```

❶ 定义一个状态桶来存储我们当前的基础设施值

然后运行 `terraform` `apply` 并确认。这个存储桶将保存我们整个系统的状态，以便我们可以进行更改。如果这个文件存储在笔记本电脑上，其他人就无法进行更改。如果文件丢失，你将丢失基础设施的状态，需要手动修改它。

接下来，我们将定义服务器和部署所需的变量。创建一个名为 `infra/server` 的目录，并添加以下列表中的代码。

列表 D.4 `main.tf`

```
terraform {                                                 ❶
 backend "gcs" {
   bucket  = "hello-api-bucket-tfstate
   prefix  = "terraform/state"
 }
}

resource "google_compute_instance" "hello_api" {            ❷
  name         = "hello-api"
  machine_type = "f1-micro"
  zone         = "us-east1-a"

  boot_disk {
    initialize_params {
      image = ${var.image_name}                             ❸
    }
  }
 lifecycle {
    create_before_destroy = true
  }
  metadata_startup_script = "sudo chmod +x /home/ubuntu/hello-api && sudo
  ➥/home/ubuntu/hello-api"                                 ❹
}

resource "google_compute_firewall" "hello_api" {            ❺
  name    = "hello-api-firewall"
  network = "default"

  allow {
    protocol = "tcp"
    ports    = ["8080"]
  }
  source_ranges = ["0.0.0.0/0"]
}

output "public_dns" {                                       ❻
  value = google_compute_instance.hello_api.public_dns
}
```

❶ 定义我们将使用的后端和状态文件

❷ 创建服务器实例

❸ 输入镜像名称

❹ 服务器启动时运行的脚本

❺ 在防火墙中创建一个洞以访问服务

❻ 输出端点以调用 API

现在我们可以创建我们的管道。

## D.3 创建管道

我们的管道将编译我们的二进制文件，使用 Packer 打包，然后创建一个服务器。在任何提交后，服务器将重新部署（见下一列表）。

列表 D.5 `pipeline.yml`

```
name: Terraform Depoyment

on:
  push:
    branches:
      - main

jobs:
  build-image:
    name: Build image
    runs-on: ubuntu-latest
    steps:
    - name: Set up Go 1.x #
      uses: actions/setup-go@v2
      with:
        go-version: ¹.18
    - name: Check out code into the Go module directory #
      uses: actions/checkout@v2
    - name: Build
      run: make build #
    - name: Build Artifact
      uses: hashicorp/packer-github-actions@master
      with:
          command: build
          arguments: "-color=false -on-error=abort"
          target: packer.pkr.hcl
          working_directory: infrastructure/packer
      env:
          PACKER_LOG: 1
          PKR_VAR_git_sha: $(git rev-parse --short "$GITHUB_SHA")
          PKR_VAR_project_id: ${{ secrets.GCP_PROJECT_ID }}
  deploy-server:
    name: Deploy Server
    runs-on: ubuntu-latest
    steps:
    - uses: hashicorp/setup-terraform@v2
    - name: Init
      run: terraform init
    - run: export TF_VAR_image_name=hello-api-$(git rev-parse --short
      ➥"$GITHUB_SHA")
    - name: install
      run: terraform apply -auto-approve
```

基础设施可能会变得复杂，而这个管道只展示了创建持续部署的简单方法。它与第十章不同，因为这种基础设施代码是自动部署的，而不是手动集成。在管道的输出中，你可以找到需要调用服务的端点。
