---
owner: ["mathlsj"]
reviewer: ["SataQiu","haiker2011","icyxp"]
description: "本书第五章节，讲述了 Knative 的安装。"
publishDate: 2019-03-04
updateDate: 2019-03-27
---

# Knative 安装

在开始使用 Knative 构建和托管工作负载前，你需要安装它。你还应该运行一些命令来验证它是否正常运行并按预期工作。本章将介绍从 Mac 或 Linux shell 安装和验证 Knative 的必要步骤。

## 建立一个 Knative 集群

首先，你需要已经有一个 Kubernetes 集群。 Knative 要求 Kubernetes 的版本在1.11以上。你必须在集群上启用 [MutatingAdmissionWebhook admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#how-do-i-turn-on-an-admission-controller)。为了简单，你可以在本地机器上使用 [Minikube](https://kubernetes.io/docs/setup/minikube/) 或者在云上运行集群。

> ### 为什么我们需要安装 Istio

迄今为止，我们还没讨论过 Istio，但是它作为安装的一部分出现了。Istio 是什么？Knative 为什么需要它？

Istio 是一个服务网络。它在 Kubernetes 之上提供了很多特性，包括流量管理、网络策略执行和可观察性。我们不认为 Istio 是 Knative 的组件，而是它的依赖项之一，就像 Kubernetes 一样。所以 Knative 最终使用 Istio 运行在 Kubernetes 集群之上。

---

本报告的目的不是详细说明 Istio 的内部工作。在这一章中，我们将介绍 Istio 与 Knative 一起使用时要用到的关于 Istio 的所有知识。如果你想了解更多，可以从 [What is Istio?](https://istio.io/docs/concepts/what-is-istio/) 以及 [Istio documentation](https://istio.io/docs/) 开始。

> ### 须知

建立 Kubernetes 集群有许多选择。决定使用哪种工具取决于您的需求和提供者对特定工具集的熟悉程度。在 GitHub 中参考 [Knative’s installation documentation](https://github.com/knative/docs/tree/master/docs/install) 以获得特定提供者的指令。

在您的本地机器上，请确保您已经安装了 kubectl v1.11或更高版本。将上下文设置指向 Knative 设置的集群。您将使用 kubectl 以 YAML 文件的形式应用于所有的 crd。

---

集群建立之后，使用以下两个命令设置 Istio：

```bash
kubectl apply --filename https://storage.googleapis.com/knative-releases/serving/latest/istio.yaml

kubectl label namespace default istio-injection=enabled
```

第一个命令将所有必需的 Istio 对象导入集群。第二个命令在 default 命名空间中启用 Istio 自动注入。这可以确保 Istio 在 default 命名空间中为每个 Pod 创建时自动注入边车（sidecar）。（你会注意到所有 Pod 至少都有两个容器。一个是用户的容器；一个是 istio-proxy。) Knative 依赖于 Istio 组件。使用以下命令验证 Istio 安装，直到所有 Pod 显示为运行或完成：

```bash
kubectl get pods -n istio-system --watch
```

现在您已经使用 Istio 运行了集群，可以开始安装 Knative。使用 kubectl 命令安装 Knative 的核心组件 Serving 和 Build。YAML 声明文件可以从 Google Storage 或 GitHub 获取。您可以使用特定的版本或使用最新的 release 版本。下面的命令将从 Google Storage 中获取最新的版本：

```bash
kubectl apply --filename https://storage.googleapis.com/knative-releases/serving/latest/release.yaml

kubectl apply --filename https://storage.googleapis.com/knative-releases/build/latest/release.yaml
```

再次验证这些对象是否正确导入。使用以下命令监视它们，直到所有 Pod 显示为运行：

```bash
kubectl get pods --namespace knative-serving --watch

kubectl get pods --namespace knative-build --watch
```

> ### 小贴士： 轻量安装

如果您正在本地机器上安装 Knative 或刚刚开始安装，您可能希望在不使用内置监控（在 monitoring 命名空间下）组件的情况下安装 Knative。在这种情况下，您可以使用以下命令来安装服务：

```bash
kubectl apply --filename https://storage.googleapis.com/knative-releases/serving/latest/release-no-mon.yaml
```

这避免了在 monitoring 命名空间中安装任何组件。想要获取更多信息，请看 *第7章指标和日志*。

---

Serving 和 Build 安装完成并运行后，按照类似的步骤启动 Eventing 模块：

```bash
kubectl apply --filename https://storage.googleapis.com/knative-releases/eventing/latest/release.yaml

kubectl get pods --namespace knative-eventing --watch
```

最后，你可以选择安装你需要的 Build 模板。这一步与第三章中的步骤完全相同。下面是安装 Kaniko 和 Buildpacks 模板的命令：

```bash
kubectl apply -f https://raw.githubusercontent.com/knative/build-templates/master/kaniko/kaniko.yaml

kubectl apply -f https://raw.githubusercontent.com/knative/build-templates/master/buildpack/buildpack.yaml
```

> ### 须知

如果你计划使用 Build 模块将源代码打包到镜像中，你需要一个容器仓库来推送。在安装 Knative 的同时，考虑设置对所选容器仓库的访问。容器仓库你可以选择 Docker Hub 或谷歌容器仓库这样的公共托管方式，或者你也可以设置自己的私有仓库。有关访问和将镜像推送到仓库的更多信息，请参阅第4章中的 Build 组件。

---

您可以使用 kubectl get buildtemplates 命令验证 Build 模板是否已成功安装。这将返回 default 命名空间中安装的所有构建模板的列表：

```bash
$ kubectl get knative
NAME AGE
buildpack 1m
kaniko 1m
```

> ### 删除 Knative 对象

您可能希望在添加某些 Knative 组件之后删除它们。可以使用 kubectl delete 命令从集群中删除任何 Knative 对象。正如您使用 kubectl apply 引用 YAML 文件一样，您也可以使用 kubectl delete 执行相同的操作：

```bash
kubectl delete -f https://raw.githubusercontent.com/knative/build-templates/master/kaniko/kaniko.yaml
```

---

Kubernetes 集群已启动。Istio 已经安装。已添加了 Serving 、 Build 和 Eventing 组件。您可能已经添加了一两个 Build 模板。您几乎已经准备好开始使用 Knative 了。剩下的只需要获取一些关于如何在网络上访问它的信息。

> ### 安装方法选择

本章中的步骤展示了如何使用本地的 kubectl apply 命令分别安装 Knative 组件。然而，一些构建在 Knative 之上的无服务器框架也可能包含安装系统的快捷方式。例如， riff 提供了一个在 Kubernetes 集群上安装和运行 Knative 的一行程序：

```bash
riff system install
```

这需要 riff CLI 和 Kubernetes 集群已经建立且 kubectl 已经指向正确的上下文。

---

## 访问 Knative 集群

设置好 Knative 集群之后，就可以将应用程序部署到 Knative 上了。但你需要知道如何使用它们。它们如何暴露在集群中？ Knative 在 istio-system 命名空间中使用 LoadBalancer 方式。使用以下命令获取外部 IP 地址，列名： EXTERNAL-IP

```bash
$ kubectl get svc istio-ingressgateway --namespace istio-system
NAME                 TYPE         CLUSTER-IP   EXTERNAL-IP
istio-ingressgateway LoadBalancer 10.23.247.74 35.203.155.229
```

正如您将在第6章中看到的，这个 IP 地址加上正确的 HTTP HOST 头就可以向集群上的应用程序发起请求。为了方便使用，可以把外部 IP 地址设置为 KNATIVE_INGRESS 这个环境变量：

```bash
$ export KNATIVE_INGRESS=$(kubectl get svc istio-ingressgateway --namespace istio-system --output 'jsonpath={.status.loadBalancer.ingress[0].ip}')

$ echo $KNATIVE_INGRESS
35.203.155.229
```

现在，使用与我们在第2章中看到的类似的 curl 命令，我们可以使用这个环境变量向 Knative 环境中的服务发出请求：

```bash
curl -H "Host: my-knative-service-name.default.example.com" http://$KNATIVE_INGRESS
```

> ### 不支持 Load Balancer？

如果你的 Kubernetes 实例不支持 load balancer（比如： Minikube），命令会略有不同因为 EXTERNAL-IP 会显示为 <pending>。使用以下命令返回节点的 IP 和端口：

```bash
$ export KNATIVE_INGRESS=$(kubectl get node  --output 'jsonpath={.items[0].status.addresses[0].address}'):$(kubectl get svc istio-ingressgateway>  --namespace istio-system --output 'jsonpath={.spec.ports[?(@.port==80)].nodePort}')

$ echo $KNATIVE_INGRESS
10.10.0.10:32380
```

---

## 结论

现在已经设置好了一切，可以将应用程序部署到 Knative 了。在第6章，您将看到一些不同的示例。您还将了解通过设置静态 IP、自定义域名以及 DNS 配置来公开集群的更健壮的方式。
