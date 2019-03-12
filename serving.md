---
owner: ["junahan"]
reviewer: ["haiker2011", "SataQiu", "rootsongjc", "icyxp"]
description: "本章介绍 Knative Serving 组件，描述 Knative Serving 如何部署并为应用和函数 (funtions) 提供服务。"
publishDate: 2019-03-03
updateDate: 2019-03-12
---

# Serving（服务）

即便使用无服务器架构，处理和响应 HTTP 请求的能力依然重要。在开始写代码使用事件触发一个函数之前，您需要有地方来运行代码。

本章探讨 Knative Serving 组件，您将了解 Knative Serving 如何管理部署以及为应用和函数提供服务。通过 Serving，您可以轻松地将一个预先构建好的镜像部署到底层 Kubernetes 集群中。（在[第三章： Build](./build.md)，您将看到 Knative Build 可以帮助构建镜像以在 Serving 组件中运行）Knative Serving 维护某一时刻的快照，提供自动化伸缩功能 (既支持扩容，也支持缩容直至为零)，以及处理必要的路由和网络编排。

Serving 模块定义一组特定的对象以控制所有功能：Revision（修订版本）、Configuration （配置）、Route（路由）和 Service（服务）。Knative 使用 Kubernetes CRD（自定义资源）的方式实现这些 Kubernetes 对象。下图 2-1 展示所有 Serving 组件对象模型间的关系。在接下去的章节将具体介绍每个部分。

<div align="center">
<img src="https://ws1.sinaimg.cn/large/006tKfTcly1g0yrpiumcqj31230u0jxo.jpg" alt="Serving Object Model"/>
图 2-1: Knative Serving 对象模型
</div>

## Configuration（配置）和 Revision（修订版本）
Knative Serving 始于 Configuration。您可以在 Configuration 中为部署定义所需的状态。最小化 Configuration 至少包括一个配置名称和一个要部署容器镜像的引用。在 Knative 中，定义的引用为 Revision。Revision 代表一个不变的，某一时刻的代码和 Configuration 的快照。每个 Revision 引用一个特定的容器镜像和运行它所需要的任何特定对象（例如环境变量和卷）。然而，您不必显式创建 Revision。由于 Revision 是不变的，它们从不会被改变和删除，相反，当您修改 Configuration 的时候，Knative 会创建一个 Revision。这允许一个 Configuration 既反映工作负载的当前状态，同时也维护一个它自己的历史 Revision 列表。

以下[示例 2-1](#example-2-1) 展示了一个完整的 Configuration 定义。它指定一个 Revision，该 Revision 使用一个容器镜像仓库 URI 引用一个特定的镜像并且指定其版本标签。

<span id="example-2-1">*示例 2-1. knative-helloworld/configuration.yml* </span>

```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Configuration
metadata:
  name: knative-helloworld
  namespace: default
spec:
  revisionTemplate:
    spec:
      container:
        image: docker.io/gswk/knative-helloworld:latest
        env:
          - name: MESSAGE
            value: "Knative!"
```

现在，您可以用一个简单的命令启用该 YAML 文件：

```shell
$ kubectl apply -f configuration.yaml
```

**自定义端口**

默认情况下，Knative 将假定您的应用程序监听 8080 端口。但是，如果不是这样的话，您可以通过 `containerPort` 参数自定义一个端口：

```yaml
    spec:
      revisionTemplate:
        spec:
          container:
            image: docker.io/gswk/knative-helloworld:latest
            env:
              - name: MESSAGE
                value: "Knative!"
            ports:
              - containerPort: 8081
```

就像任意 Kubernetes 对象一样，您可以在系统中使用命令行工具（CLI）查阅 Revision 和 Configuration。您可以使用 `kubectl get revisions` 和 `kubectl get configurations` 得到它们的列表。获取我们刚刚创建[示例 2-1](#example-2-1) 的 Configuration，可以使用命令 `kubectl get configuration knative-helloworld -oyaml`。这将以 YAML 形式显示该 Configuration 完整详情（如下[示例 2-2](#example-2-2)）。

<span id="example-2-2">*示例 2-2. 命令 `kubectl get configuration knative-hellworld -oyaml` 的输出*</span>

```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Configuration
metadata:
  creationTimestamp: YYYY-MM-DDTHH:MM:SSZ
  generation: 1
  labels:
    serving.knative.dev/route: knative-helloworld
    serving.knative.dev/service: knative-helloworld
  name: knative-helloworld
  namespace: default
  ownerReferences:
  - apiVersion: serving.knative.dev/v1alpha1
    blockOwnerDeletion: true
    controller: true
    kind: Service
    name: knative-helloworld
    uid: 9835040f-f29c-11e8-a238-42010a8e0068
  resourceVersion: "374548"
  selfLink: "/apis/serving.knative.dev/v1alpha1/namespaces\
    /default/configurations/knative-helloworld"
  uid: 987101a0-f29c-11e8-a238-42010a8e0068
spec:
  generation: 1
  revisionTemplate:
    metadata:
      creationTimestamp: null
    spec:
      container:
        image: docker.io/gswk/knative-helloworld:latest
        name: ""
        resources: {}
status:
  conditions:
  - lastTransitionTime: YYYY-MM-DDTHH:MM:SSZ
    status: "True"
    type: Ready
  latestCreatedRevisionName: knative-helloworld-00001
  latestReadyRevisionName: knative-helloworld-00001
  observedGeneration: 1
```

注意[示例 2-2](#example-2-2) 中 `status` 小节，Configuration 控制器保持对最近创建和就绪 Revison 的追踪。它也包含了 Revision 的适用条件，表明它是否就绪以接收流量。

> **NOTE**
>
> Configuration 可以指定一个已有的容器镜像，如[示例 2-1](#example-2-1) 中所示。或者，它也可以选择指向一个 Build 资源以从源代码创建一个容器镜像。[第三章：Build](./build.md) 将介绍 Knative Build 组件的详情并提供一些示例。
>

那么在 Kubernetes 集群内部发生了什么？我们在 Configuration 中指定的容器镜像是什么样子？Knative 转换 Configuration 定义为一些 Kubernetes 对象并在集群中创建它们。在启用 Configuration 后，可以看到相应的 Deployment、ReplicaSet 和 Pod。[示例 2-3](#example-2-3) 展示了所有来自[示例 2-1](#example-2-1) 所创建的对象。

<span id="example-2-3">*示例 2-3. Knative 创建的 Kubernetes 对象* </span>

```shell
$ kubectl get deployments -oname
deployment.extensions/knative-helloworld-00001-deployment

$ kubectl get replicasets -oname
replicaset.extensions/knative-helloworld-00001-deployment-5f7b54c768

$ kubectl get pods -oname
pod/knative-helloworld-00001-deployment-5f7b54c768-lrqt5
```

现在我们有了用于运行我们应用的 Pod，但是我们怎么知道该向哪里发送请求？这正是 Route 用武之地。

## Route（路由）
Knative 中的 Route 提供了一种将流量路由到正在运行的代码的机制。它将一个命名的，HTTP 可寻址端点映射到一个或者多个 Revision。Configuration 本身并不定义 Route。[示例 2-4](#example-2-4) 展示了一个将流量发送到指定 Configuration 最新 Revision 的最基本路由定义。

*<span id="example-2-4">示例 2-4. knative-helloworld/route.yml</span>*

```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Route
metadata:
  name: knative-helloworld
  namespace: default
spec:
  traffic:
  - configurationName: knative-helloworld
percent: 100
```

就像我们对 Configuration 所做的那样，我们可以运行一个简单的命令应用该 YAML 文件：

```shell
kubectl apply -f route.yaml
```

这个定义中，Route 发送 100% 流量到由 `configurationName` 属性指定 Configuration 的最新就绪 Revision，该 Revision 由 Configuration YAML 中 `latestReadyRevisionName` 属性定义。您可以通过发送如下 `curl` 命令来测试这些 Route 和 Configuration ：

```shell
curl -H "Host: knative-routing-demo.default.example.com"
http://$KNATIVE_INGRESS
```

通过使用 `revisionName` 替代 `latestReadyRevisionName` ，您可以锁定一个 Route 以发送流量到一个指定的 Revision。使用 `name` 属性，您也可以通过可寻址子域名访问 Revision。[示例 2-5](#example-2-5) 同时展示两种场景。

<span id="example-2-5">*示例 2-5. knative-routing-demo/route.yml*</span>

```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Route
metadata:
  name: knative-routing-demo
  namespace: default
spec:
  traffic:
  - revisionName: knative-routing-demo-00001
    name: v1
    percent: 100
```

我们可以再一次使用简单命令应用该 YAML 文件：

```shell
kubectl apply -f route.yaml
```

指定的 Revision 可以使用 `v1` 子域名访问，如下 `curl` 命令所示：

```shell
curl -H "Host: v1.knative-routing-demo.default.example.com"
http://$KNATIVE_INGRESS
```

> **NOTE**
>
> Knative 默认使用 `example.com` 域名，但不适合生产使用。您会注意到在 `curl` 命令 (v1.knative-routing-demo.default.example.com) 中作为一个主机头传递的 URL 包含该默认值作为域名后缀。URL 格式遵循模式 `{REVISION_NAME}.{SERVICE_NAME}.{NAMESPACE}.{DOMAIN}` 。
>
> 在这个案例中，子域名中 `default` 部分指的是命名空间。您将在[第六章：部署注意事项](./using-knative.md/#6.2)一节中学习到如何改变这些值以及如何使用自定义域名。
>

Knative 也允许以百分比的方式跨 Revision 进行流量分配。支持诸如增量发布、蓝绿部署或者其他复杂的路由场景。您将在[第六章](./using-knative.md)看到这些以及其他案例。

### Autoscaler（自动伸缩器）和 Activator（激活器）
Serverless 的一个关键原则是可以按需扩容以满足需要和缩容以节省资源。Serverless 负载应当可以一直缩容至零。这意味着如果没有请求进入，则不会运行容器实例。Knative 使用两个关键组件以实现该功能。它将 Autoscaler 和 Activator 实现为集群中的 Pod。您可以看到它们伴随其他 Serving 组件一起运行在 `knative-serving` 命名空间中（参见[示例 2-6](#example-2-6)）。

<span id="example-2-6">*示例 2-6. `kubectl get pods -n knative-serving` 输出* </span>

```shell
NAME                          READY     STATUS    RESTARTS   AGE
activator-69dc4755b5-p2m5h    2/2       Running   0          7h
autoscaler-7645479876-4h2ds   2/2       Running   0          7h
controller-545d44d6b5-2s2vt   1/1       Running   0          7h
webhook-68fdc88598-qrt52      1/1       Running   0          7h
```

Autoscaler 收集打到 Revision 并发请求数量的有关信息。为了做到这一点，它在 Revision Pod 内运行一个称之为 `queue-proxy` 的容器，该 Pod 中也运行用户提供的 (user-provided) 镜像。可以在相应 Revision Pod 上，通过运行 `kubectl describe` 命令可以看到这些容器 (参见[示例 2-7](#example-2-7))。

<span id="example-2-7">*示例 2-7. `kubectl describe pod knative-helloworld-00001-deployment-id` 输出片段* </span>

```yaml
...
Containers:
  user-container:
    Container ID:  docker://f02dc...
    Image:         index.docker.io/gswk/knative-helloworld...
...
  queue-proxy: 
    Container ID:  docker://1afcb...
    Image:         gcr.io/knative-releases/github.com/knative...
...
```

`queue-proxy` 检测该 Revision 上观察到的并发量，然后它每隔一秒将此数据发送到 Autoscaler。Autoscaler 每两秒对这些指标进行评估。基于评估的结果，它增加或者减少 Revision 部署的规模。

默认情况下，Autoscaler 尝试维持每 Pod 每秒平均 100 个并发请求。这些并发目标和平均并发窗口均可以变化。Autoscaler 也能够被配置为利用 Kubernets HPA (Horizontal Pod Autoscaler) 来替代该默认配置。这将基于 CPU 使用率来自动伸缩但不支持缩容至零。这些设定都能够通过 Revision 元数据注解 (annotations) 定制。有关这些注解的详情，请参阅 [Knative 文档](https://github.com/knative/docs/blob/master/serving/samples/autoscale-go/README.md)。

例如，一个 Revision 每秒收到 350 个请求并且每次请求大约需要处理 0.5 秒。使用默认设置 (每 Pod 100 个并发请求)，这个 Revision 将扩展至两个 Pod：

```
350 * .5 = 175
175 / 100 = 1.75
ceil(1.75) = 2 pods
```

Autoscaler 也负责缩容至零。Revision 处于 Active (激活) 状态才接受请求。当一个 Revision 停止接受请求时，Autoscaler 将其置为 Reserve (待命) 状态，条件是每 Pod 平均并发必须持续 30 秒保持为 0 (这是默认设置，但可以配置)。

处于 Reserve 状态下，一个 Revision 底层部署缩容至零并且所有到它的流量均路由至 Activator。Activator 是一个共享组件，其捕获所有到待命 Revisios 的流量。当它收到一个到某一待命 Revision 的请求后，它转变 Revision 状态至 Active。然后代理请求至合适的 Pods。

> **Autoscaler 如何伸缩**
>
> Autoscaler 采用的伸缩算法针对两个独立的时间间隔计算所有数据点的平均值。它维护两个时间窗，分别是 60 秒和 6 秒。Autoscaler 使用这些数据以两种模式运作：Stable Mode (稳定模式) 和 Panic Mode (忙乱模式)。在 Stable 模式下，它使用 60 秒时间窗平均值决定如何伸缩部署以满足期望的并发量。
>
> 如果 6 秒窗口的平均并发量两次到达期望目标，Autoscaler 转换为 Panic Mode 并使用 6 秒时间窗。这让它更加快捷的响应瞬间流量的增长。它也仅仅在 Panic Mode 期间扩容以防止 Pod 数量快速波动。如果超过 60 秒没有扩容发生，Autoscaler 会转换回 Stable Mode。
>

<span id="fingure-2-2">*图 2-2 显示 Autoscaler 和 Activator 如何和 Routes 及 Revisions 协同工作。*</span>

<div align="center">
<img src="https://ws2.sinaimg.cn/large/006tKfTcly1g0yrmo1t2cj31z70u0afi.jpg" alt="Autoscaler and Activator with Route and Revision" />
图 2-2: Autoscaler 和 Activator 如何和 Routes 及 Revisions 互动。
</div>

> **WARN**
>
> Autoscaler 和 Activator 均是 Knative 中快速演化的部分。参阅[最新 Knative 文档](https://github.com/knative/docs)获取最近改进。
>

## 服务
在 Knative 中，Service 管理负责的整个生命周期。包括部署、路由和回滚。（不要将 Knative Service 和 Kubernetes [Service](https://kubernetes.io/docs/concepts/services-networking/service/) 混淆。它们是不同的资源。） Knative Service 控制一系列组成软件的 Route 和 Configuration。Knative Service 可以被看作是一段代码 —— 您正在部署的应用或者函数。

一个 Service 注意确保一个应用有一个 Route、一个 Configuation，以及为每次 Service 更新产生的一个新 Revision。当创建一个 Service 时，您没有特别定义一个 Route，Knative 创建一个发送流量到最新 Revision 的路由。您可以选择一个特定的 Revision 以路由流量到该 Revision。

不要求您明确创建一个 Service。Route 和 Configuration 可以被分开在不同的 YAML 文件（如[示例 2-1](#example-2-1) 和 [示例 2-4](#example-2-4)）。在这种情形下，您可以应用每个单独的对象到集群。然而，推荐的方式使用一个 Service 来编排 Route 和 Configuration。[示例 2-8](#example-2-8) 所示文件用于替换来自[示例 2-1](#example-2-1) 和[示例 2-4](#example-2-4) 定义的 `configuation.yml` 和 `route.yml`。

<span id="example-2-8">*示例 2-8. knative-helloworld/service.yml* </span>

```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: knative-helloworld
  namespace: default
spec:
  runLatest:
    configuration:
      revisionTemplate:
        spec:
          container:
            image: docker.io/gswk/knative-helloworld:latest
```

注意这个 `service.yml` 文件和 `configuration.yml` 非常相似。这个文件定义 Configuration 并且是最小化 Service 定义。由于这里没有 Route 定义，一个默认 Route 指向最新 Revision。Service 控制器整体追踪它所有的 configuration 和 Route 的状态。然后反映这些状态在它的 `ConfigurationsReady` 和 `RoutesReady` conditions 属性里。当通过 CLI 使用 `kubectl get ksvc` 命令请求 Knative Service 信息的时候，这些状态可以被看到。

<span id="example-2-9">*示例 2-9. `kubectl get ksvc knative-helloworld -oyaml` 命令输出片段* </span>

```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
...
  name: knative-helloworld
  namespace: default
...
spec:
...
status:
conditions:
  - lastTransitionTime: YYYY-MM-DDTHH:MM:SSZ
    status: "True"
    type: ConfigurationsReady
  - lastTransitionTime: YYYY-MM-DDTHH:MM:SSZ
    status: "True"
    type: Ready
  - lastTransitionTime: YYYY-MM-DDTHH:MM:SSZ
    status: "True"
    type: RoutesReady
  domain: knative-helloworld.default.example.com
  domainInternal: knative-helloworld.default.svc.cluster.local
  latestCreatedRevisionName: knative-helloworld-00001
  latestReadyRevisionName: knative-helloworld-00001
  observedGeneration: 1
  targetable:
    domainInternal: knative-helloworld.default.svc.cluster.local
  traffic:
  - percent: 100
    revisionName: knative-helloworld-00001
```

[示例 2-9](#example-2-9) 显示这个命令的输出。

## 小结
至此已经向您介绍了 Service、Route、Configuration 和 Revision。Revision 是不变的并且只能经由 Configuration 改变而被创建。您可以分别单独创建 Configuration 和 Route，或者把它们组合在一起并定义为一个 Service。理解 Serving 组件的这些构建块是使用 Knative 的基础。您部署的应用均需要一个 Service 或者 Configuration 以在 Knative 中作为容器运行。

但是，如何打包您的源代码进入一个容器镜像以使用本章介绍的方式进行部署？[第三章](./build.md)将回答这些问题并且向您介绍 Knative Build 组件。

