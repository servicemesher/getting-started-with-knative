---
owner: [loverto]
reviewer: ["haiker2011","SataQiu","dreadbird"]
description: "本章讲述了使用 knative 的高级用例"
publishDate: 2019-3-3
updateDate: 2019-4-3
---

# 使用 Knative

通过前面的章节已经扎实掌握 Knative 的组件了，现在是时候开始研究一些更高级的主题了。Serving 为如何路由流量提供了相当大的灵活性，还有其他的构建模板使构建应用程序变得容易。只需几行代码即可轻松制作我们自己的事件源。在本章中，我们将深入研究这些功能，让我们的代码在 Knative 上更容易地运行。

## 创建和运行 Knative Services

[第 2 章](/serving.md) 介绍了 Knative Service 的概念。

回想一下，Knative 中的 Service 是单个配置和路由集合的组合。在 Knative 和 Kubernetes 体系内，它最终是 Pod 中的 0 个或多个容器以及其他使您的应用程序可寻址的组件。所有这些都由具有强大流量策略选项的路由层支持。

无论您将工作负载视为应用程序，容器还是流程，它都将在 Knative 中作为服务运行。这为处理许多场景提供了灵活性，具体取决于构成软件的资产。本节提供了 Knative 为您构建和部署软件提供的另一种选择。

### 使用 Cloud Foundry Buildpack 构建模板

您在[第 3 章](/build.md)中看到，Kaniko 构建模板允许您使用 Dockerfile 构建容器镜像。此方法要求您负责编写和维护 Dockerfile。如果您希望完全消除管理容器的负担，您可能希望使用不同的构建模板。Buildpack 构建模板负责基础镜像，并引入构建和运行应用程序所需的所有依赖项。

Cloud Foundry 是一种开源平台即服务（PaaS），它利用 buildpack 来简化开发和运维。在 Cloud Foundry 中，buildpacks 将检查您的源代码，以自动确定要下载的运行时和依赖项，构建代码以及运行应用程序。例如，使用 Ruby 应用程序，buildpack 将下载 Ruby 运行时并在 Gemfile 上运行 `bundle  install` 以下载所有必需的依赖项。Java buildpack 将为您的应用程序下载 JVM 和任何所需的依赖项。通过使用 Buildpack Build Template，这个模型在 Knative 中也可用。

与 Kaniko Build Template 一样，您必须将 Buildpack Build Template CRD 应用到环境：

```bash
kubectl apply -f https://raw.githubusercontent.com/knative/build-templates/master/buildpack/buildpack.yaml
```

Knative Services 仅依赖 Serving 组件。Build 模块不需要在 Knative 中部署和运行 Service。那你为什么要在你的服务中嵌入 Build 呢？你怎么知道在特定情况下这是一个好主意？

考虑您的软件开发生命周期的过程是非常重要的。您是否有一个现有的，成熟的构建流水线来生成容器镜像并将它们推送到 registry 仓库？如果是这样，您可能不需要 Knative Build 为您工作。如果没有，在 Knative Service 中定义 Build 方法可能会使事情变得更容易。

具体使用哪个构建模板还需要依据您希望如何打包代码和依赖项而定。对于使用既定流程管理 Dockerfile 的 Docker 重度使用者而言，Kaniko 是一个很好的选择。而 Cloud Foundry 的用户或开发者们

如果只喜欢编写代码并且不太关心基础设施会选择 Buildpack Build Template。经验丰富的 Java 用户可能已经熟悉使用 Jib 来构建 Java 容器，这使得它成为正确的选择。无论您的过程如何，Knative 都会提供一些不错的抽象，同时允许您选择最适合您的方法。

在 Knative 中，Buildpack 构建模板将使用 Cloud Foundry 的相同构建包，包括自动检测要应用于代码的构建包。如果您参考[例6\-1](#example-6-1)

你会看到很像 Kaniko Buildpack，你只能定义代码的位置以及推送容器镜像的位置。最大的区别是没必要提供 Dockerfile。

相反，构建模板知道如何为此应用程序构建容器。

#### 例6-1.`knative-buildpack-demo/service.yaml`{#example-6-1}

```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: knative-buildpack-demo
  namespace: default
spec:
  runLatest:
    configuration:
      build:
        serviceAccountName: build-bot
          source:
            git:
              url: https://github.com/gswk/knative-buildpack-demo.git 
              revision: master
          template:
            name: buildpack
            arguments:
            - name: IMAGE
              value: docker.io/gswk/knative-buildpack-demo:latest 
      revisionTemplate:
        spec:
          container:
            image: docker.io/gswk/knative-buildpack-demo:latest 
```

在此创建和运行 Knative Services ,除了代码和容器注册表的路由之外，与示例3-6中的 Kaniko 构建的唯一不同之处在于模板的名称已从 kaniko 更改为 buildpack。

![指数52_1.png](https://ws1.sinaimg.cn/large/61411417ly1g0pw1ph83cj20sg0sg4co.jpg)

例如，git 存储库是 Node.js 应用程序 hello.js，以及定义应用程序的依赖关系和元数据的 package.json 文件。在这种情况下，Build Template 将下载 Node 运行时和 npm 可执行文件，运行`npm install`，最后构建容器并将其推送到 Docker Hub。

您可能发现已应用了大量 YAML 文件，并且不确定是否已创建所有的 Knative 对象。有一个命令方便你确认这些问题，命令如下所示：

```bash
kubectl get knative -n {NAMESPACE} 
```

返回给定命名空间中所有的 Knative 对象列表。

```bash
kubectl get knative --all-namespaces
```

返回集群上存在的所有 Knative 对象。

## 部署注意事项

Knative 还提供不同的部署方法，具体取决于最适合您服务的方案。我们在[第 2 章](/serving.md)展示了一个 Knative 路由如何可以用来将流量发送到特定的修订。以下部分将详细介绍 Knative Routes 如何实现蓝绿部署和增量部署。

### 零停机部署

在[第 2 章](/serving.md)中，您了解了如何将单个路由指向多个修订版以及如何实现零停机部署。由于修订是不可变的，并且可以同时运行多个版本，因此可以在为旧版本提供流量时调出新版本。然后，一旦准备好将流量引导到新版本，请立即更新路由切换。这有时被称为蓝绿部署，蓝和绿代表不同的版本。

[例6\-2](#example-6-2) 重新审视了[ 第 2 章中](/serving.md)的 Route 定义。

#### 例6\-2。所有流量都将路由到 00001 修订版{#example-6-2}

```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Route
metadata:
  name: knative-helloworld
  namespace: default
spec:
  traffic:
  - revisionName: knative-routing-demo-00001
    name: v1
    percent: 100
```

如[第 2 章](/serving.md)所述，您可以在 `knative-helloworld.default.example.com` 和 `v1.knative-helloworld.default.example.com` 上访问修订版 `knative-routing-demo-00001`。让我们考虑一个场景，你已经在代码中添加了一些新功能或修复了一些错误，然后构建并将其推送到 Knative。这导致一个名为 `knative-routing-demo-00002` 的新版本。

但是，在开始向应用程序发送生产流量之前，我们希望确保它正常运行。
[在例6\-3](#example-6-3)中有一个名为 v2 的新路由，但没有路由到它的生产流量。

#### 例6\-3。我们的新版本{#example-6-3} 

```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Route
metadata:
  name: knative-helloworld
  namespace: default
spec:
  traffic:
  - revisionName: knative-routing-demo-00001
    name: v1
    percent: 100
  - revisionName: knative-routing-demo-00002
    name: v2
    percent: 0
```

这些修订版已命名为 v1 和 v2（尽管您可以选择任何名称，例如蓝和绿）。这意味着您可以在 `v2.knative-helloworld.default` 访问新版本。example.com 路由仍然只会将流量发送到 00001 修订版。在更改流量之前，请访问新版本并对其进行测试以确保它已准备好用于生产流量。当新版本准备好接收生产流量时，请再次更新路由，如[例6\-4所示](#example-6-4)。

#### 例6\-4。将所有实时流量发送到我们的新版本{#example-6-4}

```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Route
metadata:
  name: knative-helloworld
  namespace: default
spec:
  traffic:
  - revisionName: knative-routing-demo-00002
    name: v2
    percent: 100
```

应用后，新的配置流量将立即路由到 00002 版本而不会出现任何停机。发现代码中的新错误并需要回滚？可以很容易的再次更新 Route 配置以指向原始版本。因为修订版是不可变的，而 Knative 会存储过去的版本 yaml 配置，您可以随时路由它们。这包括对特定容器镜像、配置以及与修订版相关的任何构建信息的引用。

### 增量部署

Knative Routes 支持的另一种部署模式是逐步部署新版本的代码。这可以用于 AB 测试，或者在为每个用户释放功能之前将功能推广到用户子集。在 Knative 中，这是通过使用基于百分比的路由来实现的。

虽然类似于蓝绿部署示例 [6\-4，你可以在例6\-5中看到](#example-6-5) 而不是路由0% 对于 v2的流量，我们在 v1和 v2上均匀分配负载。您也可以选择使用`80-20`之类的其他拆分，甚至可以拆分三个修订版。每个修订版仍可通过指定的子域访问，但用户流量将按百分比值进行拆分。

#### 例6\-5。部分负载路由{#example-6-5}

```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Route
metadata:
  name: knative-helloworld
  namespace: default
spec:
  traffic:
  - revisionName: knative-routing-demo-00001
    name: v1
    percent: 50
  - revisionName: knative-routing-demo-00002
    name: v2
    percent: 50
```

### 自定义域

到目前为止所涵盖的每个示例都使用了通用示例域。

这不是用于生产应用程序的 URL。不仅如此，还不可能路由到 example.com。值得庆幸的是，Knative 提供了使用自定义域的选项。开箱即用，Knative 为每个 Route 使用{route}.{namespace}.{domain}方案，并为 example.com 使用默认域。

使用 `knative-custom-domain` 示例作为[示例6\-6](#example-6-6)中显示的起始位置，默认情况下它接收 `knative-custom-domain.default.example.com` 的 Route。

#### 例6\-6。 `knative-custom-domain/configuration.yaml`{#example-6-6}

```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Configuration
metadata:
  name: knative-custom-domain
  namespace: default
spec:
  revisionTemplate:
    spec:
      container:
        image: docker.io/gswk/knative-helloworld
        imagePullPolicy: Always
```

由于我们已将此定义为配置而非服务，因此我们还需要为应用程序定义路由，如[例6\-7](#example-6-7)。将这两种配置分开将为我们提供更高级别的定制，例如我们在讨论零停机部署时所说的那些定制，但也将让我们更新我们的域和路由，而无需重新部署整个应用程序。

#### 例6\-7。`knative-custom-domain/route.yaml`{#example-6-7}
 
```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Route
metadata:
  name: knative-custom-domain
  namespace: default
spec:
  traffic:
  - revisionName: knative-custom-domain-00001
    name: v1
    percent: 100
```

正如预期的那样，这将创建一个服务并在 `knative-custom-domain.default.example.com` 上创建一个 Route。现在来看看如何将默认 URL 方案中的域名从 example.com 更改为您实际可以路由到的域名。此示例使用本书的网站 `dev.gswkbook.com` 的子域。这可以通过更新配置域 ConfigMap 轻松完成，该配置域由 Knative 的默认配置，[如例 6\-8](#example-6-8) 所示。

#### 例6\-8。`knative-custom-domain/domain.yaml`{#example-6-8}

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-domain
  namespace: knative-serving
data:
  dev.gswkbook.com: ""
```

Knative 最终应该协调对域的更改。重新创建 Route 将加快进程：

```bash
$ kubectl delete -f route.yaml

$ kubectl apply -f route.yaml
```

看一下更改后的路由，您会看到它现在已获得 `knative-custom-domain.default.dev` 的更新网址。

`gswkbook.com`。如果您的入口网关可公开访问（即，在 Google 的 GKE 或类似的托管 Kubernetes 产品上设置），您可以为 `*.dev.gswkbook.com` 创建一个通配符 DNS 条目， 以便路由到您的 Knative 安装并访问您的服务和功能直接通过互联网，如[例 6\-9](#example-6-9) 所示。

```bash
$ kubectl get route knative-custom-domain -o yaml

apiVersion: serving.knative.dev/v1alpha1
kind: Route
metadata:
  ...
status:
  ...
  domain: knative-custom-domain.default.dev.gswkbook.com
  ...
  traffic:
  - name: v1
    percent: 100
    revisionName: knative-custom-domain-00001
```

您可能还希望为不同的部署设置不同的域。例如，默认情况下，您可能希望将所有内容部署到开发域，然后在测试后将其转发到生产域。Knative 提供了一种简单的启用此功能的机制，允许您定义多个域并标记路由以确定它们所在的域。让我们再次更新 `config-domain` ConfigMap，设置另一个用于生产的域名 `*.prod.gswkbook.com`，如图所示

[例6\-10。](#example-6-10)

#### 例6\-10。`knative-custom-domain/domain.yaml`{#example-6-10}

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-domain
  namespace: knative-serving
data:
  prod.gswkbook.com: |
    selector:
      environment: prod
  dev.gswkbook.com: ""
```

在[例 6\-11](#example-6-11) 中，我们已经定义了具有 `environment: prod` 标签的 Route 将被放置在 `prod.gswkbook.com`  域上，否则它将默认置在 `dev.gswkbook.com` 域中。您需要做的就是将应用程序移动到这个新域，然后在配置的元数据部分中使用这个新标签更新您的 Route。

#### 例6\-11。`knative-custom-domain/route-label.yaml`{#example-6-11}
 
```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Route
metadata:
  name: knative-custom-domain
  namespace: default
  labels:
    environment: prod
spec:
  traffic:
  - revisionName: knative-custom-domain-00001
    name: v1
    percent: 100
```

应用后，您的路由会自动更新，并且假设您的 DNS 配置正确，可以立即在新域中访问。您可以通过在检索 Route CRD 时确保域条目匹配来验证这一点，如[例 6\-12](#example-6-12) 所示。

```bash
kubectl describe route knative-custom-domain
```

#### 例6\-12。`knative-custom-domain` 路由{#example-6-12}

```yaml
description Name:         knative-custom-domain
Namespace:    default
...
Kind:         Route
Metadata:
  ...
Spec:
  ...
Status:
  ...
  Domain:          knative-custom-domain.default.prod.gswkbook.com 
  Domain Internal: knative-custom-domain.default.svc.cluster.local
  ...
```

## 构建自定义事件源

假设我们希望应用程序从没有事件源的源接收事件。例如，我们可能希望定期检查文件服务器是否有新文件，或者请求 URL 来监视更改。将这些代码组合在一起很容易，但是运行它的最佳方法是什么？像其他 Knative 应用程序一样运行它是没有意义的，因为它需要永久运行，但是如果我们在 Knative 之外运行它，那么我们就需要手动管理和配置的新组件。

Knative 通过使用 ContainerSource 轻松创建自己的事件源来解决这个问题。使用此事件源，我们提供 Knative 容器，Knative 将为容器提供 POST 事件的 URL。只要满足这几个简单的要求，我们就可以用我们喜欢的任何语言编写事件源：

它可以打包成一个容器，并有一个 ENTRYPOINT 定义，所以它知道如何运行我们的代码。

2.它希望接收一个 `--sink` CLI 标志，Knative 将提供一个指向已配置目标的 URL。

让我们看看它是如何工作的，通过构建一个事件源，它将在给定的时间间隔内发出当前时间，称为[时间事件源](http://bit.ly/2SSpX80)。首先，让我们看看事件源的代码[例 6\-13](#example-6-13)。

#### 例6\-13。`time-event-source/app.rb`{#example-6-13}

```ruby
require 'optparse'
require 'net/http'
require 'uri'
# 默认 CLI 选项（如果省略）
options = {
  :sink => "http://localhost:8080",
  :interval => 1
}

# 解析 CLI 标志
opt_parser = OptionParser.new do |opt|
  opt.on("-s", "--sink SINK", "Location to send events") 
    do |sink| options[:sink] = sink
  end
  opt.on("-i", "--interval INTERVAL", "Poll frequency") 
    do |interval| options[:interval] = interval.to_i 
  end
end
opt_parser.parse!
# 按给定间隔发送当前时间到给定的接收器
uri = URI.parse(options[:sink])
header = {'Content-Type': 'text/plain'}
loop do
  http = Net::HTTP.new(uri.host, uri.port)
  request = Net::HTTP::Post.new(uri.request_uri, header) 
  request.body = Time.now.to_s
  response = http.request(request)
  sleep options[:interval]
end
```

解析 CLI 标志后，这个 Ruby 应用程序进入一个死循环，不断地将当前时间 POST 到由 `--sink` 标志提供的 URL，然后根据 `--interval` 标志提供的秒数休眠。由于这需要打包为容器，我们还有一个用于构建它的 Dockerfile，如[例 6\-14](#example-6-14) 所示。

#### 例6\-14。`time-event-source/Dockerfile`{#example-6-14}

```Dockerfile
FROM ruby:2.5.3-alpine3.8
ADD . /time-event-source
WORKDIR /time-event-source
ENTRYPOINT ["ruby", "app.rb"]
```

这里并不奇怪。我们使用官方 Ruby 镜像作为基础，添加我们的代码，并定义如何运行我们的代码。我们可以构建我们的容器并将其发送到 Docker Hub。在我们运行事件源之前，我们需要一个发送事件的地方。我们已经开始构建一个非常简单的 Knative 应用程序，它记录了所有它收到的 HTTP POST 的主体请求。

#### 例6\-15。`time-event-source/service.yaml`{#example-6-15}

```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: logger
spec:
  runLatest:
    configuration:
      revisionTemplate:
        spec:
          container:
            image: docker.io/gswk/logger:latest
```

我们将应用在[例 6\-15](#example-6-15) 中所示的 YAML ，命名为 logger 并部署它。

```bash
kubectl apply -f service.yaml
```

剩下的就是让我们的事件源在 Knative 中运行。YAML 与其他事件源的概述相同，我们在[例 6\-16](#example-6-16) 中可以看到。

#### 例6\-16。`time-event-source/source.yaml`{#example-6-16}

```yaml
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: ContainerSource
metadata:
  labels:
    controller-tools.k8s.io: "1.0"
  name: time-eventsource
spec:
  image: docker.io/gswk/time-event-source:latest
  args:
    - '--interval=1'
  sink:
    apiVersion: serving.knative.dev/v1alpha1
    kind: Service
    name: logger
```

这里有几个需要注意的点。首先，我们在 image 参数中为 Knative 提供了 Event Source 容器的位置，就像我们部署 Service 时一样。其次，我们在 args 数组中提供了 `--interval` 标志，但我们省略了 `--sink` 标志。这是因为 Knative 将查看我们提供的接收器（在本例中为我们的日志服务），查找 URL 到该资源，并自动将其提供给我们的事件源。

这意味着，在 Event Source 容器的内部，最后我们的代码将使用类似于以下内容的命令调用：

```bash
ruby app.rb
--interval=1
--sink=http://logger.default.example.com
```

我们将使用我们期望的通常的 kubectl apply 命令来部署它：

```bash
kubectl apply -f source.yaml
```

很快，我们会看到一个新的 Pod 被创建出来，但重要的区别在于它将永久运行而不会降低到零个。我们可以查看记录器务的日志，以验证我们的事件是否符合预期，如[例 6\-17](#example-6-17) 所示。

#### 例6\-17。从我们的记录器中检索日志服务{#example-6-17}

```bash
$ kubectl get pods -l app = logger-00001 -o name pod/logger-00001-deployment-57446ffb59-vzg97
$ kubectl logs logger-00001-deployment-57446ffb59-vzg97 -c user-container

2018/12/26 21:12:59 Starting server on port 8080...
[Wed 26 Dec 2018 21:13:00 UTC] 2018-12-26 21:13:00 +0000
[Wed 26 Dec 2018 21:13:01 UTC] 2018-12-26 21:13:01 +0000
[Wed 26 Dec 2018 21:13:02 UTC] 2018-12-26 21:13:02 +0000
[Wed 26 Dec 2018 21:13:03 UTC] 2018-12-26 21:13:03 +0000
[Wed 26 Dec 2018 21:13:04 UTC] 2018-12-26 21:13:04 +0000
[Wed 26 Dec 2018 21:13:05 UTC] 2018-12-26 21:13:05 +0000
```

这种简单的抽象使我们能够快速轻松地提供自己的事件源。不仅如此，而且与构建模板非常相似，您可以设想如何轻松地与 Knative 社区共享这些事件源，因为它们很容易插入到您的环境中。我们还将在[第七章](/putting-it-all-together.md) 中查看另一个自定义事件源 

## 结论

到目前为止，我们的内容已经涵盖了相当多的用例，从初级的到高级的，但我们只是自己看了这些概念，一次展示一个功能。如果你像我们一样，真正有用的是将它们全部放在一个现实世界的例子中，观察每个组件如何相互作用。在下一章中，我们将做到这一点并构建一个利用我们所学知识的应用程序！


