---
owner: ["andyyin"]
reviewer: ["SataQiu","haiker2011"]
description: "本章是全书的展望部分，用来说明 Knative 生态体系未来展望以及如何进行 Knative 学习。"
publishDate: 2019-03-05
updateDate: 2019-05-02
---

# 下一步

还有越来越多的的项目持续的加入到年轻的 Knative 生态系统。有些已经将其他现有的开源无服务器架构（serverless）的框架带到 Knative 上。例如，Kwsk 就是努力用 Knative 来代替大部分 Apache OpenWhisk 基础服务器组件。其他开源的无服务器架构项目专门针对 Knative 而构建，甚至帮助完善 Knative 上游体系。例如，riff 项目已经提供了一组工具来帮助简化构建函数（Function）和使用 Knative。本章将简要介绍使用 riff 项目团队的一些工具在 Knative 上构建和运行函数。

## 使用 riff 项目打包函数

通过第 2 章中的 Hello World 示例，可以看出将现存的镜像从容器仓库部署到 Knative 是非常容易的。第 3 章中的 Kaniko 示例以及示例 6-1 中的 Buildpack 方式演示了如何为 Knative 构建和部署简单的十二要素（12-factor）应用程序。到目前为止，这些例子都集中在作为软件单元的容器或应用程序之上。现在回想一下第 1 章中提及函数，试想将一个函数部署到 Knative 是什么样的？答案是它看起来几乎与容器和应用程序一样。是因为有了 Build 模块，Knative 可以将您的函数代码转换为容器，其方式与应用程序代码相似。

>什么是函数（Function）?
>
>应用程序由代码组成，函数也是如此。那么函数有什么特别之处呢？难道它不是一个应用程序吗？应用程序一般由从前端 UI 到后端数据库的许多组件以及其间的所有处理组成。相比之下，函数通常只是一小段代码，具有单一目的，可以快速和异步地运行。它通常也由事件触发，而不是由用户在请求/响应场景中直接调用。

回想一下第 6 章中的 Cloud Foundry Buildpacks 示例。例 6-1 中显示的 service.yaml 文件引用了一个完整的 Node.js Express 应用程序，该应用程序的功能是在给定端口上侦听 GET 请求然后返回 “Hello World” 信息。如果我们的程序是接受数字作为输入，返回该数字的平方作为结果的函数，而不是 Hello World 应用程序呢？此代码可能类似于我们在示例 8-1 中看到的内容。

Example 8-1. knative-function-app-demo/square-app.js

```go
const express = require('express');
const app = express();
app.post('/', function (req, res) {
    let body = '';
    req.on('data', chunk => {
        body += chunk.toString();
    });
    req.on('end', () => {
        if (isNaN(body))
            res.sendStatus(400);
        else {
            var square = body ** 2;
            res.send(square.toString());
} });
});
var port = 8080;
app.listen(port, function () {
    console.log('Listening on port',  port);
});
```

我们可以使用示例 6-1 中的相同 Buildpack 来构建此函数并将其部署到 Knative。又如例 8-2，它也是使用 Node.js 编写的一个函数，它不是一个完整的 Express 应用程序，而仅仅由一个函数组成，不包含任何其他 Node.js 模块。

Example 8-2. knative-function-demo/square.js

```javascript
module.exports = (x) => x ** 2
```

Knative 支持这一点，因为它具有 Build 模块提供的灵活性。为了构建和部署这样的代码到 Knative，需要一个自定义的构建模板将这个简单的仅含函数的代码转换为可运行的 Node.js 应用程序。例 8-2 中的代码使用了 function invokers 特别支持的编程模型，function invokers 是 riff 项目一部分的。
riff 是 Pivotal 的一个开源项目，构建于 Knative 之上，它提供了一些很棒的东西：用于安装 Knative 和管理在其上部署的函数的 CLI，以及使我们能够编写像例 8-2 中代码的 function invokers。这些 invokers 负责执行函数，例如我们见过的 Node.js 示例，或 Spring Cloud Functions，甚至是 Bash 脚本。与 Build 模板一样，invokers 也是开源的，并且随着 riff 项目的成熟，invokers 支持调用的函数种类会越来越多。请务必查看 https://project-riff.io 了解更多信息！

## 拓展阅读

在继续学习的过程中，有大量围绕 Knative 构建相关的文档、示例以及演示可以供您阅读和参考。最好的当然是 GitHub 仓库中 Knative Docs，它不仅包含有关 Knative 的每一部分如何工作的详细说明，而且还有更多的演示和加入社区的链接，例如 Knative Slack 频道或邮件列表。

我们非常感谢你花时间在我们的这本书上，希望对你开始上手使用 Knative 有帮助。我们可以留给你的最好建议就是要勤写代码并开始构建一些东西，无论大小。通过犯错并学习如何解决问题来探索和学习，与他人分享你学到的东西！Knative 的社区非常年轻，但成长速度非常快，我们希望看到你成为它的一员。