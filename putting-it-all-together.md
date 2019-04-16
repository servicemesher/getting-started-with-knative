---
owner: ["eportzxp"]
reviewer: ["haiker2011","SataQiu","rootsongjc"]
description: "通过一个可视化地展示世界各地的地震活动的演练，将前几章所学知识进行了一个串联。存在问题: 原文中 bit.ly 链接无法打开，前端容器镜像也无法访问到。"
publishDate: 
updateDate:
---

# 演练

让我们把我们所学的一切运用起来去创造一些东西吧!我们进行一个演练，它利用了您前面所学到的许多知识，并通过使用[美国地质勘探局 (USGS) 地震数据源](https://on.doi.gov/2GmDWNB)的数据提供了一个服务，以可视化地展示世界各地的地震活动。您可以在 GitHub 存储库 [gswk/earthquakedemo](https://github.com/gswk/earthquake-demo) 中找到我们将要介绍的代码。

## 架构

在深入研究代码之前，让我们先看看应用程序的体系架构，如 [图7-1](#pic-7-1) 所示。我们在这里构建三个重要的东西:事件源、服务和前端。
图中 Knative 内部的每一个组件都代表着我们将利用目前所学的知识来构建的内容，包括使用 Kaniko 构建模板的服务和用于轮询数据的自定义事件源:

*USGS 事件源*
: 我们将构建一个自定义的 ContainerSource 事件源，它将在给定的时间间隔轮询 USGS 提供的数据。为预构建的容器镜像打包。

<span id="pic-7-1">![arch](https://ws4.sinaimg.cn/large/006tKfTcly1g1o90h02s4j30qs0idgng.jpg)</span>
*图 7-1 应用程序的体系结构。来自于 USGS 的地震数据源作为事件进入我们的事件源，这将触发我们的 GeoCoder 服务来持久化事件。我们的前台也将使用我们的 Geocoder 服务来查询最近的事件。*

*Geocoder 服务*
: 这将为事件源提供 POST 事件的节点，并使用提供的坐标查找地址。它还将作为前端用来查询和检索最近的事件的节点。我们将使用 Build 服务来构建容器镜像。与运行在 Kubernetes 上的 Postgres 数据库通信。

*前端*
: 一个可以可视化最近的地震活动的轻量级的、持续运行的前端

我们可以使用 [Helm](https://helm.sh) 在 Kubernetes 集群上轻松地搭建起 Postgres 数据库，Helm 是一个可以轻松地在 Kubernetes 上打包和共享应用程序包的工具。关于如何在你的 Kubernetes 集群上启动和运行的介绍，请务必参考 Helm 的文档。如果您运行在 Minikube 或没有任何特定的权限要求的 Kubernetes 集群上，那么您可以使用以下简单的命令来设置 Helm:
    `$ helm init`

对于像谷歌的 GCP 这样具有更深层安全配置的集群，请参考 [Helm Quickstart 指南](https://helm.sh/docs/using_helm/#quickstart)。接下来我们可以设置一个 Postgres 数据库并且传递一些配置参数以使设置更容易:

```bash
$ helm install
--name geocodedb
--set postgresqlPassword=devPass,postgresqlDatabase
                        =geocode stable/postgresql
```

这将在我们的 Kubernetes 集群中创建一个 Postgres 数据库，将用户密码设置为 devPass ，并创建一个名为 geocode 的数据库。我们已经将 Postgres 服务器命名为 geocodedb ，这意味着在 Kubernetes 集群中，我们可以通过 geocodedb-postgresql.default.svc.cluster.local 访问该服务器。现在让我们来深入了解代码吧!

## Geocoder 服务

如应用程序体系结构图所示，我们的事件源和前端都将向 Geocoder 服务发送请求，后者将与 Postgres 数据库通信。这将我们的服务置于应用程序的中心位置。对我们服务的 HTTP POST 请求将会在数据库中记录事件，而 GET 请求将检索过去24小时内发生的事件。让我们来看一下 [示例 7-1](#example-7-1) 中我们服务的代码。

*<span id="example-7-1">示例 7-1 geocoder/app.rb</span>*

```ruby
require 'geocoder'
require 'json'
require 'pg'
require 'sinatra'

set :bind, '0.0.0.0'

# DB connection credentials are passed via environment
# variables
DB_HOST = ENV["DB_HOST"] || 'localhost'
DB_DATABASE = ENV["DB_DATABASE"] || 'geocode'
DB_USER = ENV["DB_USER"] || 'postgres'
DB_PASS = ENV["DB_PASS"] || 'password'

# Connect to the database and create table if it doesn't exist
conn = PG.connect( dbname: DB_DATABASE, host: DB_HOST, 
    password: DB_PASS, user: DB_USER)
conn.exec "CREATE TABLE IF NOT EXISTS events (
    id varchar(20) NOT NULL PRIMARY KEY,
    timestamp timestamp,
    lat double precision,
    lon double precision,
    mag real,
    address text
);"

# Store an event
post '/' do
    d = JSON.parse(request.body.read.to_s)
    address = coords_to_address(d["lat"], d["long"])
    id = d["id"]
    
    conn.prepare("insert_#{id}", 
        'INSERT INTO events VALUES ($1, $2, $3, $4, $5, $6)')
    conn.exec_prepared("insert_#{id}", [d["id"], d["time"], 
        d["lat"], d["long"], d["mag"], address.to_json])
end

# Get all events from the last 24 hours
get '/' do
    select_statement = "select * from events where 
        timestamp > 'now'::timestamp - '24 hours'::interval;"
    results = conn.exec(select_statement)
    jResults = []
    results.each do |row|
        jResults << row
    end

    content_type 'application/json'
    headers 'Access-Control-Allow-Origin' => "*"
    return jResults.to_json
end

# Get the address from a given set of coordinates
def coords_to_address(lat, lon)
    coords = [lat, lon]
    results = Geocoder.search(coords)

    a = results.first
    address = {
        address: a.address,
        house_number: a.house_number,
        street: a.street,
        county: a.county,
        city: a.city,
        state: a.state,
        state_code: a.state_code,
        postal_code: a.postal_code,
        country: a.country,
        country_code: a.country_code,
        coordinates: a.coordinates
    }

    return address
end
```

我们将使用 Knative 为我们构建容器镜像，将连接到 Postgres 数据库所需的信息传递给它，并运行我们的服务。我们可以在 [示例7-2](#example-7-2) 中看到这是如何设置的。

*<span id="example-7-2">示例 7-2. earthquake-demo/geocoder-service.yaml</span>*

```YAML
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: geocoder
  namespace: default
spec:
  runLatest:
    configuration:
      build:
        serviceAccountName: build-bot
        source:
          git:
           url: https://github.com/gswk/geocoder.git
           revision: master
        template:
          name: kaniko
          arguments:
          - name: IMAGE
            value: docker.io/gswk/geocoder
      revisionTemplate:
        spec:
          container:
            image: docker.io/gswk/geocoder
            env:
            - name: DB_HOST
              value: "geocodedb-postgresql.default.svc.cluster.local"
            - name: DB_DATABASE
              value: "geocode"
            - name: DB_USER
              value: "postgres"
            - name: DB_PASS
              value: "devPass"
```

`kubectl apply -f earthquake-demo/geocoder-service.yaml`

因为我们已经通过环境变量传递了所有连接信息给我们的 Postgres 数据库，这是我们的服务运行需要的所有信息。接下来，我们将获取事件源并运行它，以便我们可以开始向新部署的服务发送事件。

## USGS 事件源

我们的事件源将负责在指定的时间间隔内轮询 USGS 地震活动的数据，解析它，并将其发送到我们定义的接收器。由于我们需要轮询数据，并且没有由 USGS 将其推送给我们的可能，因此它非常适合使用 ContainerSource 编写自定义事件源。

在设置事件源之前，还需要一个事件发送的通道。虽然我们可以直接将事件从事件源发送到我们的服务，但如果我们希望将来能够将事件发送到另一个服务，这将给我们带来一些灵活性。我们只需要一个简单的通道，我们将在 [示例 7-3](#example-7-3) 中定义它。

*<span id="example-7-3">示例 7-3. earthquake-demo/channel.yaml</span>*

```YAML
apiVersion: eventing.knative.dev/v1alpha1
kind: Channel
metadata:
  name: geocode-channel
spec:
  provisioner:
    apiVersion: eventing.knative.dev/v1alpha1
    kind: ClusterChannelProvisioner
name: in-memory-channel
```

`kubectl apply -f earthquake-demo/channel.yaml`

正如我们在第6章中构建自定义事件源一样，我们的这个事件源也是由一个脚本构成，在本例中是一个 Ruby 脚本，它接受两个命令行标志位: --sink 和 --interval。让我们在 [示例7-4](#example-7-4)中看看这个。

*<span id="example-7-4">示例 7-4. usgs-event-source/usgs-event-source.rb</span>*

```Ruby
require 'date'
require "httparty"
require 'json'
require 'logger'
require 'optimist'

$stdout.sync = true
@logger = Logger.new(STDOUT)
@logger.level = Logger::DEBUG

# Poll the USGS feed for real-time earthquake readings
def pull_hourly_earthquake(lastTime, sink)
    # Get all detected earthquakes in the last hour
    url = "https://earthquake.usgs.gov/earthquakes/feed/v1.0/" \
    + "summary/all_hour.geojson"
    response = HTTParty.get(url)
    j = JSON.parse(response.body)

    # Keep track of latest recorded event, reporting all
    # if none have been tracked so far
    cycleLastTime = lastTime

    # Parse each reading and emit new ones as events
    j["features"].each do |f|
        time = f["properties"]["time"]

        if time > lastTime
            msg = {
                time: DateTime.strptime(time.to_s,'%Q'),
                id: f["id"],
                mag: f["properties"]["mag"],
                lat: f["geometry"]["coordinates"][1],
                long: f["geometry"]["coordinates"][0]
            }

            publish_event(msg, sink)
        end

        # Keep track of latest reading
        if time > cycleLastTime
            cycleLastTime = time
        end
    end

    lastTime = cycleLastTime
    return lastTime
end

# POST event to provided sink
def publish_event(message, sink)
    @logger.info("Sending #{message[:id]} to #{sink}")
    puts message.to_json
    r = HTTParty.post(sink, 
        :headers => {'Content-Type'=>'text/plain'}, 
        :body => message.to_json)
    
    if r.code != 200
        @logger.error("Error! #{r}")
    end
end


# Parse CLI flags
opts = Optimist::options do
    banner <<-EOS
Poll USGS Real-Time Earthquake data
Usage:
  ruby usgs-event-source.rb
EOS
    opt :interval, "Poll Frequenvy", 
        :default => 10
    opt :sink, "Sink to send events", 
        :default => "http://localhost:8080"
end

# Begin polling USGS data
lastTime = 0
@logger.info("Polling every #{opts[:interval]} seconds")
while true do 
    @logger.debug("Polling . . .")
    lastTime = pull_hourly_earthquake(lastTime, opts[:sink])
    sleep(opts[:interval])
end
```

像往常一样，Knative 在作为 ContainerSource 事件源运行时将处理 --sink 标志位。我们还提供了一个额外的标记 --interval，我们将定义这个标记，因为我们编写的代码将允许用户定义自己的轮询间隔。脚本被打包为 Docker 镜像并上传到 Dockerhub 上的 [gswk/usgs-event-source](https://hub.docker.com/r/gswk/usgs-event-source) 下。剩下的就是创建 [示例 7-5](#example-7-5) 中所示的我们的事件源的 YAML，并创建订阅，以便将事件从通道发送到 [示例 7-6](#example-7-6) 中所示的服务。

*<span id="example-7-5">示例 7-5. earthquake-demo/usgs-event-source.yaml</span>*

```YAML
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: ContainerSource
metadata:
  labels:
    controller-tools.k8s.io: "1.0"
  name: usgs-event-source
spec:
  image: docker.io/gswk/usgs-event-source:latest
  args: 
    - "--interval=10"
  sink:
    apiVersion: serving.knative.dev/v1alpha1
    kind: Service
name: geocoder
```

`$ kubectl apply -f earthquake-demo/subscription.yaml`

一旦我们应用这个 YAML，事件源将启动一个持续运行的容器，该容器将轮询事件并将它们发送到我们创建的通道中。另外，我们需要将 Geocoder 服务连接到通道中。

*<span id="example-7-6">示例 7-6. earthquake-demo/subscription.yaml</span>*

```YAML
apiVersion: eventing.knative.dev/v1alpha1
kind: Subscription
metadata:
  name: geocode-subscription
spec:
  channel:
    apiVersion: eventing.knative.dev/v1alpha1
    kind: Channel
    name: geocode-channel
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1alpha1
      kind: Service
name: geocoder
```

`$ kubectl apply -f earthquake-demo/subscription.yaml`

创建了订阅之后，我们已经将所有内容连接起来，以便将事件通过自定义事件源带到环境中，然后将它们发送到服务中，服务将把它们持久化到 Postgres 数据库中。我们还有最后一个要部署的部分，那就是我们的前端，用来可视化所有东西。

## 前端

最后，我们需要把我们收集的所有数据一起放在前端来进行可视化。我们创建了一个简单的网站，并将其打包在一个容器镜像中，该容器镜像将使用 [Nginx](https://hub.docker.com/_/nginx/) 提供服务。当页面加载时，它将调用 Geocoder 服务，返回一个地震事件的数组，包括坐标和震级，并在地图上显示它们。我们还将把它设置为 Knative 服务，这样我们就可以免费获得简易的路由和度量。同样，我们将像其他 Knative 服务一样编写一个 YAML，并使用 Kaniko 构建模板，如 [示例 7-7](#example-7-7) 所示。

*<span id="example-7-7">示例 7-7. earthquake-demo/frontend/frontend-service.yaml</span>*

```YAML
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: earthquake-demo
  namespace: default
spec:
  runLatest:
    configuration:
      build:
        serviceAccountName: build-bot
        source:
          git:
            url: https://github.com/gswk/earthquake-demo-frontend.git
            revision: master
        template:
          name: kaniko
          arguments:
          - name: IMAGE
            value: docker.io/gswk/earthquake-demo-frontend
      revisionTemplate:
        spec:
          container:
            image: docker.io/gswk/earthquake-demo-frontend
            env:
            - name: EVENTS_API
value: "http://geocoder.default.svc.cluster.local"
```
`$ kubectl apply -f earthquake-demo/frontend-service.yaml`

我们定义 EVENTS_API 环境变量，前端将使用该变量来了解 Geocoder 服务的位置。最后这一部分就绪后，我们就可以启动并运行整个系统了!我们的应用程序如 [图 7-2](#pic-7-2) 所示。

<span id="pic-7-2">![前端界面](https://ws3.sinaimg.cn/large/006tKfTcly1g1o90iornuj31i00u0492.jpg)</span>
*图 7-2 我们的应用程序启动起来了*

当请求进入我们的前端应用程序时，它将从 Geocoder 服务中提取事件，当新事件进入时，它们将被我们的自定义事件源接收。此外，Knative 还提供了一些额外的工具，通过内置的日志记录、度量和跟踪功能，帮助您保持应用程序和服务的正常运行。

## 度量及日志纪录

任何在生产环境中运行过代码的人都知道我们的故事还没有结束。仅仅因为编写了代码和部署了应用程序，就需要对管理和运维负责。正确地了解代码如何处理日志及度量是该运维流程的一部分，幸运的是 Knative 附带了许多工具来提供这些信息。更好的是，它的大部分功能已经自动绑定到您的代码中，而不需要您做任何特殊的事情。

让我们从深入研究 Geocoder 服务的日志开始，这个功能由 Kibana 提供，Kibana 是在我们设置 Knative 的服务组件时安装的。在我们访问任何东西之前，我们需要在我们的 Kubernetes 集群中设置一个代理，只需一个命令就可以轻松完成:

`$ kubectl proxy`

这将为访问整个 Kubernetes 集群中打开一个代理，并可以在我们机器的8001端口上访问它。这也包括 Kibana，我们可以通过 http://localhost:8001/api/v1/namespaces/knative-monitoring/services/kibana-logging/proxy/app/kibana 访问它。

我们需要提供一个索引模板，我们可以简单地使用 * 和 timestamp_millis 的时间过滤器。最后，如果我们转到 Kibana 的 Discover 选项卡，我们将看到系统中的每个日志！让我们看一下通过如下搜索方式发送到 Geocoder 服务的请求及其结果，如 [图7-3](#pic-7-3) 所示。

*localEndpoint.serviceName = geocoder*

<span id="pic-7-3">![Geocoder](https://ws4.sinaimg.cn/large/006tKfTcly1g1o90g3npwj30w70ddmzd.jpg)</span>
*图 7-3。展示我们的Geocoder服务日志的Kibana仪表板*

那么，如果只想看粗略的度量标准呢?看看某些指标比如失败的请求和响应时间可以提供解决我们应用程序问题的线索，Knative 还通过与 Grafana 一起提供非常多的度量指标（从响应代码的分布到我们的服务使用了多少 CPU）来帮助我们解决这个问题。Knative 甚至包括一个仪表盘，用于可视化当前集群的使用情况，以帮助进行容量规划。在加载 Grafana 之前，我们需要使用以下命令将端口转发到 Kubernetes 集群:

```
$ kubectl port-forward
    --namespace knative-monitoring $(kubectl get pods
    --namespace knative-monitoring
    --selector=app=grafana
    --output=jsonpath="{.items..metadata.name}") 3000
```

一旦转发，我们可以通过 http://localhost:3000 访问仪表板。在 [图7-4](pic-7-4) 中，我们可以看到发送到 Geocoder 服务的请求的图，看起来很好很健康!

<span id="pic-7-4">![Geocoder](https://ws3.sinaimg.cn/large/006tKfTcly1g1o90hfexlj30uo07e753.jpg)</span>
*图 7-4 对Geocoder服务的成功和失败请求对比的图表*

最后，Knative 还附带了 Zipkin 来帮助跟踪我们的请求。当请求通过我们的 ingress 网关进入，并到达数据库时，通过一些简单的仪表化，我们可以很好地了解我们的应用程序内部情况。在按照前述设置好代理之后，我们可以通过 http://localhost:8001/api/v1/namespaces/istio-system/services/zipkin:9411/proxy/Zipkin 来访问 Zipkin。一旦进入，我们就可以通过它看到请求如何发送到我们的 Geocoder服务上的，如 [图 7-5](#pic-7-5) 和 [图 7-6](#pic-7-6) 所示。

<span id="pic-7-5">![Geocoder_zipkin1](https://ws1.sinaimg.cn/large/006tKfTcly1g1o90hy3czj30wb0da769.jpg)
*图7-5 对一个到Geocoder服务请求的简单跟踪*</span>

<span id="pic-7-6">![Geocoder_zipkin2](https://ws2.sinaimg.cn/large/006tKfTcly1g1o90fp7b3j30wb0fxwhe.jpg)
*图 7-6 我们的服务请求堆栈时间分解*</span>

## 结论

成功了！一个完整的应用程序，带有我们自己定制的事件源。这在很大程度上总结了我们在本书中要学习的内容，但是 Knative 还可以提供更多。同时，Knative 也在不断地发展和完善。当你继续你的旅程时，还有很多资源值得关注，所以在我们结束之前，我们需要知道我们在[第8章](./what-is-next.md)中还提供其他一些参考资料。
