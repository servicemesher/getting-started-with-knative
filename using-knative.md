---
owner: [loverto]
reviewer: []
description: "本章讲述了使用knative的高级用例"
publishDate: 
updateDate:
---

# 使用 Knative

已经扎实掌握Knative的组成组件了，现在是时候开始研究一些更高级的主题了。服务为如何路由流量提供了相当大的灵活性，还有其他构建模板使构建应用程序变得容易。只需几行代码即可轻松制作我们自己的事件源。在本章中，我们将深入研究这些功能，以了解如何使我们在Knative上运行代码变得更加容易。

## 创建和运行Knative Services

[第2章介绍](/serving.md) 了Knative Service的概念。

回想一下Knative中的Service是单个配置和路径集合的组合。在Knative和Kubernetes封面下，最终是Pod中的0个或更多容器以及使应用程序可寻址的组件。所有这些都由具有强大流量策略选项的路由层支持。

无论您将工作负载视为应用程序，容器还是流程，它都将在Knative中作为服务运行。这为处理许多场景提供了灵活性，具体取决于构成软件的资产。本节提供了Knative为您构建和部署软件提供的另一种选择。

### 使用Cloud Foundry Buildpack构建模板

您在[第3章中](/build.md) 看到，Kaniko构建模板允许您使用Dockerfile构建容器图像。此方法要求您负责编写和维护Dockerfile。如果您希望完全消除容器管理的负担，您可能希望使用不同的构建模板。Buildpack构建模板负责基础映像，并引入构建和运行应用程序所需的所有依赖项。

Cloud Foundry是一种开源平台即服务（PaaS），它利用buildpack来简化开发和运营。在Cloud Foundry中，buildpacks将检查您的源代码，以自动确定要下载的运行时和依赖项，构建代码以及运行应用程序。例如，使用Ruby应用程序，buildpack将下载Ruby运行时并 在Gemfile上运行 bun\-dle  instal以下载所有必需的依赖项。Java buildpack将为您的应用程序下载JVM和任何所需的依赖项。通过使用Buildpack Build Template，Knative中也提供了这个模型。

与Kaniko Build Template一样，您必须将Buildpack Build Template CRD带入环境：

```bash
kubectl apply -f https://raw.githubusercontent.com/knative/build-templates/master/buildpack/buildpack.yaml
```


Knative Services仅依赖于Serving组件。Build模块不需要在Knative中部署和运行Service。那你为什么要在你的服务中嵌入Build呢？你怎么知道在特定情况下这是一个好主意？

考虑您的软件开发生命周期过程非常重要。您是否有一个现有的，成熟的构建管道来生成容器映像并将它们推送到注册表？如果是这样，您可能不需要Knative Build为您完成工作。如果没有，在Knative Service中定义Build方法可能会使事情变得更容易。

决定使用哪个构建模板还需要检查您希望如何打包代码和依赖项。对于Docker的大量用户以及管理Dockerfiles的既定流程，Kaniko是一个很好的选择。Cloud Foundry或develop\-40的用户 

如果只喜欢编写代码并且不太关心基础设施的人会选择Buildpack Build Template。经验丰富的Java用户可能已经熟悉Jib来构建Java容器，这使得它成为正确的选择。无论您的过程如何，Knative都会提供一些不错的抽象，同时允许您选择最适合您的方法。

在Knative中，Buildpack构建模板将使用Cloud Foundry使用的相同构建包，包括自动检测要应用于代码的构建包。如果您参考 [例6\-1](#p51)

你会看到很像Kaniko Buildpack，你只能定义代码的位置以及推送容器图像的位置。最大的区别是没有必要提供Dockerfile。

相反，构建模板知道如何为此应用程序构建容器。

例6\-1。`knative-buildpack-demo/service.yaml`

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

除了代码和容器注册表的路径之外，唯一的

[与示例3\-6中的Kaniko构建的不同之处 ](#p31) 在于模板的名称已从kaniko更改为buildpack。在此 创建和运行Knative Services中

![指数52_1.png](https://ws1.sinaimg.cn/large/61411417ly1g0pw1ph83cj20sg0sg4co.jpg)

例如，git存储库是Node.js应用程序hello.js，以及  定义应用程序的依赖关系和元数据的package.json 文件。在这种情况下，Build Template将下载Node运行时和npm可执行文件，运行npm install，最后构建容器并将其推送到Docker Hub。

获得所有的Knative对象

您可能会发现已应用了大量YAML文件，并且不确定所有已创建的Knative对象。有一个方便的命令来帮助你

只是。只需输入

```bash
kubectl get knative -n {NAMESPACE} 
```

返回所有Knative的分类列表

给定命名空间中的对象或使用kubectl get

knative \-\-all\-namespaces返回所有Knative

群集上存在的对象。

## 部署注意事项

Knative还提供不同的部署方法，具体取决于最适合您服务的方案。我们在[第2章显示](/serving.md)  一个Knative路线如何可以用来将流量发送到特定的修订。以下部分将详细介绍Knative Routes如何实现蓝绿部署和增量部署。

### 零停机时间部署

[在第2章中](/serving.md)，您了解了如何将单个路由指向多个修订版以及如何实现零停机部署。由于修订是不可变的，并且可以同时运行多个版本，因此可以在为旧版本提供流量时调出新版本。然后，一旦准备好将流量引导到新版本，请更新路由以立即切换。这有时被称为蓝绿色部署，蓝色和绿色代表不同的版本。

[例6\-2](#p52) 重新审视了 [ 第2章中](/serving.md) 的Route定义。

例6\-2。所有流量都将路由到修订版00001

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

如[第2章所述](/serving.md)，您可以在`knative-helloworld.default.example.com`和 `v1.knative-helloworld.default.example.com` 上访问Revision knative\-routing\-demo\-00001。让我们考虑一个场景，你已经在代码中添加了一些新功能或修复了一些错误，然后构建并将其推送到Knative。这导致一个名为`knative-routing-demo-00002`的新版本。

但是，在开始向应用程序发送实时流量之前，我们希望确保它正常运行[在例6\-3](#p53)中有

一个名为v2 的新路由，但没有路由到它的实时流量。

例6\-3。我们的新版本 

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

这些修订版已命名为 v1和v2（尽管您可以选择任何名称，例如蓝色和绿色）。这意味着您可以在`v2.knative-helloworld.default`访问新版本。考试ple.com但是直播路线仍然只会将流量发送到修订版00001.在更改流量之前，请访问新版本并对其进行测试以确保它已准备好用于生产流量。当新版本准备好接收实时流量时，请再次更新路由，如 [例6\-4所示](#p54)。

部署注意事项

 例6\-4。将所有实时流量发送到我们的新版本

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

应用后，新的配置流量将立即开始路由到版本00002而不会出现任何停机。发现代码中的新错误并需要回滚？再次更新Route配置以指向原始版本同样容易。由于Revisions是不可变的，但是简单，轻量级的YAML配置，Knative将存储过去的版本，您可以随时路由它们。这包括对特定容器映像，配置以及与Revision相关的任何构建信息的引用。

### 增量部署

Knative Routes支持的另一种部署模式是逐步部署新版本的代码。这可以用于AB测试，或者在为每个用户释放功能之前将功能推广到用户子集。在Knative中，这是通过使用基于百分比的路由来实现的。

虽然类似于蓝绿色部署示例

[例6\-4，你可以在例6\-5中看到](#p54) 而不是路由0％

对于v2的流量，我们在v1和v2上均匀分配负载。您也可以选择使用80\-20之类的其他拆分，甚至可以拆分三个修订版。每个修订版仍可通过指定的子域访问，但用户流量将按百分比值进行拆分。

例6\-5。部分负载路由

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

这不是用于生产应用程序的很棒的URL。不仅如此，还不可能路由到 example.com。值得庆幸的是，Knative提供了使用自定义域的选项。开箱即用，Knative为每个Route使用{route}。{namespace}。{domain}方案，并为 example.com 使用默认域。

使用`knative-custom-domain`示例作为[示例6\-6中](#p55) 显示的起始位置，默认情况下它接收  `knative-custom-domain.default.example.com` 的Route。

例6\-6。 `knative-custom-domain/configuration.yaml`

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

由于我们已将此定义为配置而非服务，因此我们还需要为应用程序定义路由，如

[例6\-7](#p56)。将这两种配置分开将为我们提供更高级别的定制，例如我们在讨论零停机部署时所说的那些定制，但也将让我们更新我们的域和路由，而无需重新部署整个应用程序。

部署注意事项

 例6\-7。`knative-custom-domain/route.yaml`
 
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

正如预期的那样，这将创建一个服务并在`knative-custom-domain.default.example.com`上 创建一个Route。现在来看看如何将默认URL方案中的域从 example.com 更改为您实际可以路由到的域。此示例使用本书的网站 dev.gswkbook.com 的子域。这可以通过更新配置域ConfigMap轻松完成，该配置域由Knative by defa [ult](#p56) 配置[，如例6\-8所示](#p56)。

例6\-8。`knative-custom-domain/domain.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-domain
  namespace: knative-serving
data:
  dev.gswkbook.com: ""
```

Knative最终应该协调对域的更改。重新创建Route将加快进程：

```bash
$ kubectl delete -f route.yaml

$ kubectl apply -f route.yaml
```

看一下更改后的路线，您会看到它现在已获得knative\-custom\-domain.default.dev 的更新网址。

gswkbook.com。如果您的入口网关可公开访问（即，在Google的GKE或类似的托管Kubernetes产品上设置），您可以为\* .dev.gswkbook.com 创建一个通配符DNS条目， 以便路由到您的Knative安装并访问您的服务和功能直接通过互联网，如 [例6\-9](#p57) 所示。

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

您可能还希望为不同的部署设置不同的域。例如，默认情况下，您可能希望将所有内容部署到开发域，然后在测试后将其转发到生产域。Knative提供了一种简单的启用此功能的机制，允许您定义多个域并标记路由以确定它们所在的域。让我们再次更新config\-domain ConfigMap，设置另一个用于生产的域名 \* .prod.gswkbook.com，如图所示

[例6\-10。](#p57)

例6\-10。`knative-custom-domain/domain.yaml`

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

[在例6\-11中](#p58)，我们已经定义了具有标签environment ：prod的任何Route将被放置在 prod.gswkbook.com  域上，否则它将  默认 放置在 dev.gswkbook.com 域中。您需要做的就是将应用程序移动到这个新域，然后在配置的元数据部分中使用这个新标签更新您的Route。

 例6\-11。`knative-custom-domain/route-label.yaml`
 
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

应用后，您的路由会自动更新，并且假设您的DNS配置正确，可以立即在新域中访问。您可以通过在检索Route CRD时确保域条目匹配来验证这一点[，如例6\-12所示。](#p58)

```bash
kubectl describe route knative-custom-domain
```

例6\-12。knative\-custom\-domain路由

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

假设我们希望我们的应用程序从我们没有事件源的源接收事件。例如，我们可能希望定期检查文件服务器是否有新文件，或者请求URL来监视更改。将这些代码组合在一起很容易，但是运行它的最佳方法是什么？像其他Knative应用程序一样运行它是没有意义的，因为它需要永久运行，但是如果我们在Knative之外运行它，那么我们就可以了。

会有我们需要手动管理和配置的新组件。

Knative通过使用ContainerSource事件源轻松创建自己的事件源来解决这个问题。使用此事件源，我们提供Knative容器，Knative将为容器提供POST事件的URL。只要满足几个要求，我们就可以用我们喜欢的任何语言编写事件源：

它可以打包成一个容器，并有一个ENTRYPOINT

定义所以它知道如何运行我们的代码。

2.它希望接收一个\-\-sink CLI标志，Knative将提供一个指向已配置目标的URL。

让我们看看它是如何工作的，通过构建一个事件源，它将在给定[的时间间隔内](http://bit.ly/2SSpX80) 发出当前时间 [，称为时间事件 \-](http://bit.ly/2SSpX80)

[来源](http://bit.ly/2SSpX80)。首先，让我们看看我们的事件源的代码

[例6\-13。](#p59)

例6\-13。`time-event-source/app.rb`

```ruby
require 'optparse'
require 'net/http'
require 'uri'
# 默认CLI选项（如果省略）
options = {
  :sink => "http://localhost:8080",
  :interval => 1
}

# 解析CLI标志
opt_parser = OptionParser.new do |opt|
  opt.on("-s", "--sink SINK", "Location to send events") 
    do |sink| options[:sink] = sink
  end
  opt.on("-i", "--interval INTERVAL", "Poll frequency") 
    do |interval| options[:interval] = interval.to_i 
  end
end
opt_parser.parse!
# 按给定间隔发送当前时间
＃给定的接收器
uri = URI.parse(options[:sink])
header = {'Content-Type': 'text/plain'}
loop do
  http = Net::HTTP.new(uri.host, uri.port)
  request = Net::HTTP::Post.new(uri.request_uri, header) request.body = Time.now.to_s
  response = http.request(request)
  sleep options[:interval]
end
```

解析CLI标志后，这个Ruby应用程序进入一个无限循环，不断地将当前时间POST到由。提供的URL

\-\-sink标志，然后休眠\-\-interval标志提供的秒数。由于这需要打包为容器，我们还有一个用于构建它的Dockerfile，如图所示

[例6\-14。](#p60)

例6\-14。`time-event-source/Dockerfile`

```Dockerfile
FROM ruby:2.5.3-alpine3.8
ADD . /time-event-source
WORKDIR /time-event-source
ENTRYPOINT ["ruby", "app.rb"]
```

这里并不奇怪。我们使用官方Ruby图像作为基础，添加我们的代码，并定义如何运行我们的代码。我们可以构建我们的容器并将其发送到Docker Hub。在我们运行事件源之前，我们需要一个发送事件的地方。我们已经开始构建一个非常简单的Knative应用程序，它记录了所有HTTP POST的主体

要求它收到。我们将应用所示的YAML

[在例6\-15](#p60)  中部署它，命名为logger。

例6\-15。`time-event-source/service.yaml`

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



```bash
kubectl apply -f service.yaml
```

剩下的就是让我们的事件源在Knative中运行。YAML与其他事件源的概述相同，

[我们在例6\-16中可以看到。](#p61)

例6\-16。`time-event-source/source.yaml`

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

这里有几点需要注意。首先，我们在image参数中为Knative提供了Event Source容器的位置，就像我们部署Service时一样。其次，我们在args数组中提供了\-\-interval标志，但我们省略了\-\-sink标志。这是因为Knative将查看我们提供的接收器（在本例中为我们的记录器服务），查找URL

到该资源，并自动将其提供给我们的事件源。

这意味着，在Event Source容器的内部，最后我们的代码将使用类似于以下内容的命令调用：

```bash
ruby app.rb
--interval=1
--sink=http://logger.default.example.com
```

我们将使用我们期望的通常的kubectl apply命令来部署它：

```bash
kubectl apply -f source.yaml
```

很快，我们会看到一个新的Pod旋转，但重要的区别在于它将永久运行而不是从下降到零。我们可以查看记录器 服务 中的日志，以验证我们的事件是否 [符合预期，如例6\-17所示](#p62)。

例6\-17。从我们的记录器中检索日志服务 

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

这种简单的抽象使我们能够快速轻松地提供自己的事件源。不仅如此，而且与构建模板非常相似，您可以设想如何轻松地与Knative社区共享这些事件源，因为它们很容易插入到您的环境中。我们还将在[第七章中](/putting-it-all-together.md) 查看另一个自定义事件源 

## 结论

到目前为止，我们已经涵盖了相当多的基础，从基本的基础到高级用例，但我们只是自己看了这些概念，一次展示一个功能。如果你像我们一样，真正有用的是将它们全部放在一个真实世界的例子中，观察每个组件如何相互作用。在下一章中，我们将做到这一点并构建一个利用我们所学知识的应用程序！


