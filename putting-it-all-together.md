---
owner: []
reviewer: []
description: ""
publishDate: 
updateDate:
---

# 演练
让我们把我们所学的一切运用起来，去创造一些东西吧!我们进行一个演练，它利用了您所学到的许多知识，并通过使用[美国地质勘探局(USGS)地震数据源](https://on.doi.gov/2GmDWNB)提供了一个服务，以可视化世界各地的地震活动。您可以在 GitHub 存储库 [gswk/earthquakedemo](https://github.com/gswk/earthquake-demo) 中找到我们将要介绍的代码。
## 架构
在深入研究代码之前，让我们先看看应用程序的体系架构，如图7-1所示。我们在这里构建三个重要的东西:事件源、服务和前端。
图中Knative内部的每一个组件都代表着我们将利用目前所学的知识来构建的内容，包括使用Kaniko构建模板的服务和用于轮询数据的自定义事件源:

*USGS 事件源*
: 我们将构建一个自定义的 ContainerSource 事件源，它将在给定的时间间隔轮询USGS提供的数据。为预构建的容器映像打包。

![arch](images/arch.png)
*图 7-1 应用程序的体系结构。事件来将自于USGS的地震数据源做为我们的事件源，这将触发我们的GeoCoder服务来保存事件。我们的前台也将使用我们的Geocoder服务来查询最近的事件。*

*Geocoder 服务*
: 这将为事件源提供发布事件的端点，并使用提供的坐标查找地址。它还将为前端提供一个用来查询和检索最近的事件的端点。我们将使用构建服务来构建容器映像。与运行在Kubernetes上的Postgres数据库通信。

*前端*
: 一个可以可视化最近的地震活动的轻量级的、持续运行的前端

我们可以使用Helm在Kubernetes集群上轻松地搭建起Postgres数据库，Helm是一个可以轻松地在Kubernetes上打包和共享应用程序包的工具。关于如何在你的Kubernetes集群上启动和运行的介绍，请务必参考Helm的文档。如果您运行在Minikube或没有任何特定的权限要求的Kubernetes集群上，那么您可以使用以下简单的命令来设置Helm:
`$ helm init`

对于像谷歌的GCP这样具有更深层安全配置的集群，请参考Helm Quickstart指南。接下来我们可以设置一个Postgres数据库和传递一些配置参数以使设置更容易:
```bash
$ helm install
--name geocodedb
--set postgresqlPassword=devPass,postgresqlDatabase
            =geocode stable/postgresql
```
这将在我们的Kubernetes集群中创建一个Postgres数据库，将用户密码设置为devPass，并创建一个名为geocode的数据库。我们已经将Postgres服务器命名为geocodedb，这意味着在Kubernetes集群中，我们可以通过geocodedbpostgresql.default.svc.cluster.local访问该服务器。现在让我们来深入了解代码吧!

## Geocoder 服务

