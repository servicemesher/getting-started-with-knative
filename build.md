---
owner: ["haiker2011"]
reviewer: ["SataQiu", "shaobai", "junahan", "rootsongjc"]
description: "本章是全书的第三章，主要介绍 Build （构建）的几个相关的组件，通过示例展示了如何进行配合来减少手动构建，更轻松的完成代码的打包和构建。"
publishDate: 2019-03-04
updateDate: 2019-03-14
---

# Build（构建）

Knative 的 Serving（服务）组件是解决如何从容器到 URL 的，而 Build 组件是解决如何从源代码到容器的。Build resource 允许您定义如何编译代码和构建容器，而不是指向预构建的容器镜像。这确保了在将代码发送到容器镜像库之前以一致的方式编译和打包代码。在本章中将会向你介绍一些新的组件：

* Build

> 驱动构建过程的自定义 Kubernetes 资源。在定义构建时，您将定义如何获取源代码以及如何创建将运行源代码的容器镜像。

* Build Template

> 封装可重复构建步骤集合并允许对构建进行参数化的模板。

* Service Account

> 允许对私有资源（如 Git 存储库或容器镜像库）进行身份验证。

> [备注] 在编写本文时，有一些活动的工作要迁移 [Build Pipeline（构建流水线）](https://github.com/knative/build-pipeline)，对构建中的流水线进行重构更类似于 CI/CD 流水线的 Knative。这意味着除了编译和打包代码外，Knative 中的构建还可以轻松地运行测试并发布这些结果。请密切关注 Knative 的未来版本，了解这一变化。

## Service Account（服务账户）

在开始配置构建之前，你首先会面临一个紧迫的问题：如何在构建时获得需要验证的服务？如何从私有的 Git 仓库拉取代码和如何把容器镜像推到 Docker Hub 里面？为此，你可以利用两个 Kubernetes 原生组件的组合：Secret 和 Service Account 。Secret 可以让你安全地存储这些经过身份验证的请求所需的凭据，Service Account 可以让你灵活地为多个构建提供和维护凭据，而无需每次构建新应用程序时手动配置它们。

在 [Example 3-1](#example-3-1) 中，首先创建一个 Secret ，命名为 `dockerhub-account`，里面包含需要使用的凭据。当然，可以像应用其他 YAML 一样应用它，如 [Example 3-2](#example-3-2) 所示。

<span id="example-3-1">*Example 3-1. knative-build-demo/secret.yaml*</span>

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dockerhub-account
  annotations:
    build.knative.dev/docker-0: https://index.docker.io/v1/
type: kubernetes.io/basic-auth
data:
  # 'echo -n "username" | base64'
  username: dXNlcm5hbWUK
  # 'echo -n "password" | base64'
  password: cGFzc3dvcmQK
```

<span id="example-3-2">*Example 3-2. kubectl apply*</span>

```bash
kubectl apply -f knative-build-demo/secret.yaml
```

首先要注意的是，`username` 和 `password` 在传递给 Kubernetes 时都是 base64 编码的。还注意到，使用 `basic-auth` 根据 Docker Hub 进行身份验证，这意味着将使用用户名和密码进行身份验证，而不是类似于 access token（访问令牌）的东西。此外，Knative 还附带了开箱即用的 `ssh-auth`，这允许使用 SSH 私钥从私有 Git 存储库中拉取代码。

除了将 Secret 命名为 `dockerhub-account` 之外，还对 Secret 进行了注解。Annotation（注解）是说明连接到特定主机时使用哪些凭据的一种方式。在 [Example 3-3](#example-3-3) 中，定义了连接到 Docker Hub 时使用的基于身份的验证凭证集。

**我的凭据安全吗？**
> 使用 base64 编码对凭证进行编码不是为了安全性，而是为了可靠地将这些字符串传输到其中 Kubernetes 。在后端，Kubernetes 提供了关于如何加密机密的更多选项。有关加密的详细资料 Secret，请参考 [Kubernetes 文档](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) 。

一旦创建了名为 `dockerhub-account` 的 Secret，接下来必须创建要运行应用程序的 Service Account ，以便它能够访问 Kubernetes 中的凭据。配置很简单，如 [Example 3-3](#example-3-3) 所示。

<span id="example-3-3">*Example 3-3. knative-build-demo/serviceaccount.yaml*</span>

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-bot
secrets:
- name: dockerhub-account
```

示例中创建了一个名为 `build-bot` 的 ServiceAccount ，允许它访问 `dockerhub-account` Secret 。在示例中当推送容器镜像时，Knative 使用这些凭证对 Docker Hub 进行身份验证。

## The Build Resource（构建资源）

首先从 Hello World 应用程序开始。这是一个简单的 Go 应用程序，它监听端口8080并以 “Hello from Knative!” 作为 HTTP GET 请求的回应。代码如 [Example 3-4](#example-3-4) 所示。

<span id="example-3-4">*Example 3-4. knative-helloworld/app.go*</span>

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func handlePost(rw http.ResponseWriter, req *http.Request) {
    fmt.Fprintf(rw, "%s", "Hello from Knative!")
}

func main() {
    log.Print("Starting server on port 8080...")
    http.HandleFunc("/", handlePost)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

然后编写一个 Dockerfile 来构建代码和容器，如 [Example 3-5](#example-3-5) 所示。

<span id="example-3-5">*Example 3-5. knative-helloworld/Dockerfle*</span>

```docker
FROM golang

ADD . /knative-build-demo
WORKDIR /knative-build-demo

RUN go build

ENTRYPOINT ./knative-build-demo
EXPOSE 8080
```

在前面的[第2章](serving.md)中，你已经在本地构建了容器并手动将其推送到容器镜像库。然而，Knative 为在 Kubernetes 集群中使用 Build 来完成这些步骤提供了一种更好的方式。与 Configuration （配置）和 Route（路由）一样，Build 也可以简单地作为 Kubernetes 自定义资源（CRD）来通过 YAML 定义的方式实现。在深入研究每个组件之前，先来看一看 [Example 3-6](#example-3-6) ，看看 Build 的配置是什么样的。

<span id="example-3-6">*Example 3-6. knative-build-demo/service.yaml*</span>

```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: knative-build-demo
  namespace: default
spec:
  runLatest:
    configuration:
      build:
        serviceAccountName: build-bot
        source:
          git:
            url: https://github.com/gswk/knative-helloworld.git
            revision: master
         template:
           name: kaniko
           arguments:
           - name: IMAGE
             value: docker.io/gswk/knative-build-demo:latest
      revisionTemplate:
        spec:
          container:
            image: docker.io/gswk/knative-build-demo:latest
```

在构建步骤之前，你还会看到定义源代码位置的源代码部分。目前，Knative 发布了三个代码源选项：

* git：Git 仓库，可以选择使用参数来定义分支、标记或提交 SHA 。

* gcs：位于谷歌云存储中的存档文件。

* 自定义：任意容器镜像仓库。这允许用户编写自己的源代码，只要将源代码放在 `/work space` 目录中即可。

只需要安装一个额外的组件，即 Build Template（构建模板）。将会在 “Build template” 一节中向你更深入地介绍这些内容，但是现在，先将继续使用在 YAML 中定义的方式，在本例中是 Kaniko Build Template 如 [Example 3-7](#example-3-7) 所示。

<span id="example-3-7">*Example 3-7. Install the Kaniko Build Template*</span>

```bash
kubectl apply -f https://
raw.githubusercontent.com/knative/build-templates/master/kaniko/kaniko.yaml
```

通过应用模板，可以像在 Serving 示例中那样部署服务，配置如 [Example 3-8](#example-3-8) 所示。

<span id="example-3-8">*Example 3-8. Deploy our application*</span>

```bash
kubectl apply -f knative-build-demo/service.yaml
```

然后，该构建将运行以下步骤：

1. 从 [gswk/knative-helloworld](https://github.com/gswk/knative-helloworld) 的 GitHub repo 中拉取代码。
2. 在 repo 中使用 Kaniko Build Template (下一节将详细描述)。
3. 使用前面设置的 “build-bot” Service Account 将容器推送到 [gswk/knative-build-demo](https://hub.docker.com/r/gswk/knative-build-demo) 上的 Docker Hub。
4. 使用新构建的容器部署应用程序。

## Build Template（构建模板）

在 [Example 3-6](#example-3-6) 中，使用了一个 Build Template ，但从未真正解释过 Build Template 是什么或它做什么。简单来说，Build Template 是可共享的、封装的、参数化的构建步骤集合。目前，Knative 已经支持多个 Build Template ，包括：

* Kaniko

> 在运行的容器中构建容器镜像，而不依赖于运行 Docker daemon 。

* Jib

> 为Java应用程序构建容器镜像。

* Buildpack

> 自动检测应用程序的运行时，并建立一个容器镜像使用 Cloud Foundry Buildpack。

虽然这并不是可用模板的完整列表，但是可以轻松地集成 Knative 社区开发的新模板。安装 Build Template 和应用 YAML 文件安装 Service（服务）、Route（路由）或 Build configuration（构建配置）一样简单：

```bash
kubectl apply -f https://raw.githubusercontent.com/knative/
build-templates/master/kaniko/kaniko.yaml
```

然后可以像其他configura配置一样应用 [Example 3-6](#example-3-6) 来部署应用程序，同时向它发送请求，就像在[第2章](serving)中所做的那样：

```bash
kubectl apply -f knative-build-demo/service.yml

$ curl -H "Host: knative-build-demo.default.example.com"
http://$KNATIVE_INGRESS
```

在 [Example 3-9](#example-3-9) 中继续使用 Kaniko 作为参考来进一步观察 Build Template 。

<span id="example-3-9">*Example 3-9. https://github.com/knative/build-templates/blob/master/kaniko/kaniko.yaml*</span>

```yaml
apiVersion: build.knative.dev/v1alpha1
kind: BuildTemplate
metadata:
  name: kaniko
spec:
  parameters:
  - name: IMAGE
    description: The name of the image to push
  - name: DOCKERFILE
    description: Path to the Dockerfile to build.
    default: /workspace/Dockerfile
  steps:
  - name: build-and-push
    image: gcr.io/kaniko-project/executor
    args:
    - --dockerfile=${DOCKERFILE}
    - --destination=${IMAGE}
```

Build Template 的 `steps` 部分具有与 Build 完全相同的语法，只是使用命名变量进行模板化。实际上，除了用变量替换路径之外，`steps` 部分看起来非常类似于 [Example 3-6](#example-3-6) 的模板部分。`parameters` 部分在 Build Template 所期望的参数周围放置了一些结构。Kaniko Build Template 需要一个定义在何处推送容器镜像的 `IMAGE` 参数，但是有一个可选的 `DOCKERFILE` 参数，如果没有定义该参数，则提供一个默认值。

## 结论

Knative 中的 Build 在部署应用程序时删除了许多手动步骤。此外，Build Template 提供了一些构建代码和删除手动管理组件数量的好方法。随着时间的推移，可能会有越来越多的 Build Template 被构建并与 Knative 社区共享，这可能是最值得关注的事情之一。

我们已经花了很多时间来构建和运行应用程序，但是 serverless 的最大承诺之一是，它可以使您的服务轻松地连接到事件源。在下一章中，将研究 Knative 的 Eventing（事件）组件以及开箱即用的所有可用事件源。
