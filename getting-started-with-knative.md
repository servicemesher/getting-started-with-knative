# Knative入门——构建基于 Kubernetes 的现代化Serverless应用

**本书是 Getting Started with Knative - Building Modern Serverless Workloads on Kubernetes 的中文版，由 [ServiceMesher 社区](http://www.servicemesher.com)翻译。**

<img src="cover.jpg" width="50%" height="50%">

## 关于本书

*Getting Started with Knative* 是一本由 Pivotal 公司赞助 O’Reilly 出品的免费电子书，英文版下载地址：<https://content.pivotal.io/ebooks/getting-started-with-knative>。

在线阅读：<http://www.servicemesher.com/getting-started-with-knative/>

Github：<https://github.com/servicemesher/getting-started-with-knative/>

## 关于 Knative

[Knative](https://github.com/knative) 是一个基于 Kubernetes 的，用于构建、部署和管理现代 serverless 应用的平台。

## 关于 ServiceMesher 社区

ServiceMesher社区，关注内容涵盖Kubernetes、Service Mesh、Istio、Serverless、Knative等云原生技术，社区分享开源技术干货，推动服务网格和云原生在企业的落地。

## 译者名单

感谢参与本书翻译和审校的所有[贡献者](https://github.com/servicemesher/getting-started-with-knative/graphs/contributors)（按字母顺序排序）：

- [SataQiu](https://github.com/SataQiu)：邱世达
- [andyyin](https://github.com/andyyin)：殷琦
- [dishuihengxin](https://github.com/dishuihengxin)：韦世滴
- [dreadbird](https://github.com/dreadbird)：王刚
- [eportzxp](https://github.com/eportzxp)：张晓鹏
- [fleeto](https://github.com/fleeto)：崔秀龙
- [haiker2011](https://github.com/haiker2011)：孙海洲
- [icyxp](https://github.com/icyxp)：徐鹏
- [jordanchenCN](https://github.com/jordanchenCN)：陈佳栋
- [junahan](https://github.com/junahan)：杨铁党
- [lou-lan](https://github.com/lou-lan)：翟怀楼
- [loverto](https://github.com/loverto)：殷龙飞
- [mathlsj](https://github.com/mathlsj)：李寿景
- [rootsongjc](https://github.com/rootsongjc)：宋净超
- [shaobai](https://github.com/shaobai)：陈冬

### 各章节详细译者信息

| 章节             | 译者   | 审校者                               |
| ---------------- | ------ | ------------------------------------ |
| 前言             | 宋净超 | 孙海洲，徐鹏                         |
| Knative 概述     | 陈佳栋 | 宋净超，孙海洲，徐鹏，邱世达，陈冬   |
| Serving（服务）  | 杨铁党 | 孙海洲，邱世达，宋净超，徐鹏         |
| Build（构建）    | 孙海洲 | 邱世达，陈冬，杨铁党，宋净超，翟怀楼 |
| Eventing（事件） | 韦世滴 | 孙海洲，邱世达，王刚，周雨青，宋净超 |
| Knative 安装     | 李寿景 | 邱世达，孙海洲，徐鹏                 |
| Knative 使用     | 殷龙飞 | 孙海洲，邱世达，王刚，宋净超         |
| 演练             | 张晓鹏 | 孙海洲，邱世达，宋净超               |
| 下一步           | 殷琦   | 邱世达，孙海洲                       |

## 反馈与参与本书

关于本书有任何问题可以[提交 Issue](https://github.com/servicemesher/getting-started-with-knative/issues/new) 反馈，参与本书翻译请参阅[规范](CODE_OF_CONDUCT.md)。

## 版权

**本书经 Pivotal 公司授权 [ServiceMesher 社区](http://www.servicemesher.com)翻译，请勿擅自印刷出版，引用本书中文版中的内容请注明出处。**

![ServiceMesher](images/006tKfTcly1g0cz6429t2j31jt0beq9s.jpg)

# 前言

Kubernetes 赢了。这不是夸大其词，事实就是如此。越来越多的人开始基于容器部署，而 Kubernetes 已经成为容器编排的事实标准。但是，Kubernetes 自己也承认，它是一个*容器*平台而不是*代码*平台。它可以作为一个运行和管理容器的很好的平台，但是这些容器是如何构建、运行、扩展和路由很大程度上是由用户自己决定的。这些是 Knative 想要补充的缺失部分。

也许您已经在生产上使用 Kubernetes，或者您是一个前沿技术爱好者，梦想着将您基于 OS/2 运行的组织现代化。不管怎样，本报告都没有假定太多东西，只是要求您知道容器是什么，具有 Kubernetes 的一些基本知识，可以访问 Kubernetes 集群。如果这些您都不具备的话，那么 Minikube 是一个很好的选择。

我们将使用大量代码示例和预先构建的容器镜像，这些镜像我们都为读者开源，您可以从 http://github.com/gswk 找到所有代码示例，并在 http://hub.docker.com/u/gswk 找到所有容器镜像。您还可以在 http://gswkbook.com 找到这两个存储库以及其他重要参考资料的链接。

我们对 Knative 的未来十分期待。虽然我们来自 Pivotal——Knative 最大的贡献者之—— 但本报告仅出自于对 Knative 的发展前景充满期待的我们。报告中包含了我们的观点，有的读者可能不认同这些观点，还可能会热情地告诉我们为什么我们错了。没关系！这个领域非常新，并且不断重新定义自己。至少，本报告将让您思考无服务器架构（serverless），您会和我们一样对 Knative 感到兴奋。

## 目标读者

我们本质上是开发人员，所以这份报告主要是针对开发人员编写的。在整个报告中，我们将探索 serverless 架构模式，并向开发人员展示自服务用例示例（例如构建和部署代码）。然而，Knative 吸引了不同角色的技术人员。特别是，将 Knative 组件作为更大平台的一部分或与他们的系统集成的想法会引起运维和平台构建者们的极大兴趣。当这些受众探索如何使用 Knative 来实现其特定目的时，本报告将对他们非常有用。

## 您将学到什么

尽管本报告并不旨在详解 Knative 的全部功能，但已足够深入，可以带您入门 Knative，了解它的工作原理和使用方式。初步了解了 Knative 后，我们将花一些时间研究如何使用它的每个主要组件。然后转到一些高级用例，最后通过构建一个真实的示例应用来结束，该应用将充分利用您在本报告中学到的所有知识。

## 致谢

我们要感谢 Pivotal。我们都是第一次合作写书，如果没有 Pivotal 团队的支持，可能就不会有本书。技术营销总监 Dan Baskette（我们的老板）和产品营销副总裁 Richard Seroter 在 Pivotal 和领导者的成长中发挥了重要作用。我们要感谢 Jared Ruckle、Derrick Harris 和 Jeff Kelly，他们给予我们很多帮助。我们还要感谢 Tevin Rawls，他是 Pivotal 团队的一名优秀实习生，帮助我们在第 7 章中为我们的演示构建了前端。当然，我们要感谢 O’Reilly 团队所有人的支持和指导。非常感谢整个 Knative 社区，尤其是那些在 Pivotal 帮助我们的人。最后我们要感谢 Virginia Wilson、Nic Williams 博士、Mark Fisher、Nate Schutta、Michael Kehoe 和 Andrew Martin 花时间审阅我们的稿件。

*Brian McClain*：我要感谢我的妻子 Sarah 在我写作过程中给予我不断的支持和鼓励。我还要感谢我们的两只狗，Tony 和 Brutus，让我几乎把所有时间都用在这份报告上。还要感谢我们的三只猫 Tyson、Marty 和 Doc，它们有时候会趴在我的笔记本电脑上呼呼大睡，这让我可以更积极地投入写作，感谢它们的陪伴。最后，感谢我的合著者 Bryan Friedman，没有他，我是不可能完成这份报告的。Pivotal 告诉我，合作是 1+1 大于 2。

*Bryan Friedman*：感谢我的妻子 Alison，她十分支持我的写作，还是我们家最有才华的作家。我还要感谢我两个漂亮的女儿，Madelyn 和 Arielle，她们让我每天都变得更好。我也有一个忠诚的办公室伙伴，我的狗 Princeton，它大多只是喜欢待在沙发上，但偶尔会看着我的脸，暗示他为我在这份报告上的工作感到自豪。当然，我无法独自完成这一切，所以我要感谢我的合著者 Brian McClain，他的技术实力和富有感染力的激情在整个过程中给了我极大的帮助。与他合作真是太荣幸了。
