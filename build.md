---
owner: [haiker2011]
reviewer: []
description: "本章是全书的第三章，用于说明 Build (构建)的内容等。"
publishDate: 2019-03-03
updateDate:
---

# Build（构建）

Knative的 Serving(服务)组件是如何从容器到 URL 的，而 Build(构建)组件是如何从源代码到容器的。Build resource允许您定义如何编译代码和构建容器，而不是指向预构建的容器镜像。这确保了在将代码发送到您选择的容器镜像库之前以一致的方式编译和打包代码。有一些新的组件，我们将在本章介绍：

* Builds

> 驱动构建过程的自定义Kubernetes资源。
在定义构建时，您将定义如何获取源代码以及如何创建将运行源代码的容器镜像。

* Build Templates

> 封装可重复构建步骤集合并允许对构建进行参数化的模板。

* Service Accounts

> 允许对私有资源(如Git存储库或容器镜像库)进行身份验证。

> [备注] 在编写本文时，有一些活动的工作要迁移[Build Pipelines（构建流水线）](http://bit.ly/2SqnuTc)，对构建中的流水线进行重构更类似于 CI/CD 流水线的 Knative。这意味着除了编译和打包代码外，Knative中的构建还可以轻松地运行测试并发布这些结果。请确保密切关注Knative的未来版本，了解这一变化。

## Service Accounts（服务账户）

在开始配置构建之前，我们首先面临一个紧迫的问题：如何在构建时获得需要验证的服务？如何从私有的 Git 仓库拉取代码和如何把容器镜像推到 Docker Hub 里面。为此，我们可以利用两个 kubernet-native 组件的组合：Secrets 和 Service Accounts（服务帐户）。Secrets 允许我们安全地存储这些经过身份验证的请求所需的凭据，Service Accounts（服务帐户）允许我们灵活地为多个构建提供和维护凭据，而无需每次构建新应用程序时手动配置它们。

在 [Example 3-1]()，我们首先创建一个 Secret，命名为 `dockerhub-account`，里面包含我们需要的凭据。当然，我们可以像应用其他 YAML 一样应用它，如[Example 3-2]()所示。

*Example 3-1. knative-build-demo/secret.yaml*

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

*Example 3-2. kubectl apply*

```bash
kubectl apply -f knative-build-demo/secret.yaml
```

首先要注意的是，用户名和密码在传递给 Kubernetes 时都是 base64 编码的。我们还注意到，我们使用`basic-auth`根据 Docker Hub 进行身份验证，这意味着我们将使用用户名和密码进行身份验证，而不是类似于 access token（访问令牌）的东西。此外，Knative 还附带了开箱即用的`ssh-auth`，这允许我们使用像SSH私钥如果我们想从私有Git存储库中拉取代码。

除了将 Secret 命名为 `dockerhub-account` 之外，我们还对 Secret 进行了注释。Annotations（注释）是说明连接到特定主机时使用哪些凭据的一种方式。在 [Example 3-3]()中，我们定义了连接到 Docker Hub 时使用的基于身份的验证凭证集。

**我的凭据安全吗？**
> 使用 base64 编码对凭证进行编码不是为了安全性，而是为了可靠地将这些字符串传输到其中
Kubernetes 。在后端，Kubernetes 提供了关于如何加密机密的更多选项。有关加密的详细资料
秘密，请参考[Kubernetes文档](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)。

一旦我们创建了名为 `dockerhub-account` 的 Secret，我们必须创建将运行我们应用程序的 Service Account（服务帐户），以便它能够访问 Kubernetes 中的凭据。配置很简单，如 [Example 3-3]()所示。

*Example 3-3. knative-build-demo/serviceaccount.yaml*

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-bot
secrets:
- name: dockerhub-account
```

在这里，我们创建了一个名为 `build-bot` 的 ServiceAccount（服务账户），允许它访问`dockerhub-account` Secret。在我们的例子中当推送容器镜像时，Knative使用这些凭证对 Docker Hub 进行身份验证。

## The Build Resource

*Example 3-4. knative-helloworld/app.go*

```golang
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

*Example 3-5. knative-helloworld/Dockerfle*

```dockerfile
FROM golang

ADD . /knative-build-demo
WORKDIR /knative-build-demo

RUN go build

ENTRYPOINT ./knative-build-demo
EXPOSE 8080
```

*Example 3-6. knative-build-demo/service.yaml*

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

*Example 3-7. Install the Kaniko Build Template*

```bash
kubectl apply -f https://
raw.githubusercontent.com/knative/build-templates/master/kaniko/kaniko.yaml
```

*Example 3-8. Deploy our application*

```bash
kubectl apply -f knative-build-demo/service.yaml
```


## Build Templates（构建模板）

```bash
kubectl apply -f https://raw.githubusercontent.com/knative/
build-templates/master/kaniko/kaniko.yaml
```

```bash
kubectl apply -f knative-build-demo/service.yml

$ curl -H "Host: knative-build-demo.default.example.com"
http://$KNATIVE_INGRESS
```

*Example 3-9. https://github.com/knative/build-templates/blob/master/kaniko/kaniko.yaml*

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

## 结论

我们已经看到 Knative 中的 Builds（构建）在部署应用程序时删除了许多手动步骤。此外，Build Templates（构建模板）已经为我们提供了一些构建代码和删除手动管理组件数量的好方法。随着时间的推移，可能会有越来越多的Build Templates（构建模板）被构建并与Knative社区共享，这可能是最值得关注的事情之一。

我们已经花了很多时间来构建和运行我们的应用程序，但是serverless（无状态服务）的最大承诺之一是，它可以使您的服务轻松地连接到事件源。在下一章中，我们将研究 Knative 的Eventing（事件）组件以及开箱即用的所有可有事件源。
