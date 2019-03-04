---
owner: ["andyyin"]
reviewer: []
description: "本章是全书的展望部分，用来说明Knative生态体系未来展望以及如何进行Knative学习"
publishDate: 2019-03-05
updateDate: 2019-03-05
---

# 下一步
还有很多的项目加入到年轻的Knative生态系统，且趋势是不断增加的。有些已经将其他现有的开源无服务器架构（serverless）的框架带到Knative上。例如，Kwsk就是努力用Knative来代替大部分Apache OpenWhisk基础服务器组件。其他开源的无服务器架构（serverless）项目专门针对Knative而构建，甚至帮助完善Knative上游体系。例如，riff项目已经提供了一组工具来帮助简化构建函数（Functions）和使用Knative。本章将简要介绍使用riff项目团队的一些工具在Knative上构建和运行函数（Functions）。
## 使用 riff 项目打包函数（Functions）
通过第2章中的Hello World示例，可以看出将现存的镜像从容器仓库部署到Knative是非常容易的。第3章中的Kaniko示例以及示例6-1中的Buildpack方式演示了如何为Knative构建和部署简单的12-factor应用程序。到目前为止，这些例子都集中在作为软件单元的容器或应用程序上。现在回想一下第1章中提及函数（functions），试想将一个函数部署到Knative是什么样的？答案是它看起来几乎与容器和应用程序一样。是因为有了Build模块，Knative可以将您的函数（function）代码转换为容器，其方式与任何应用程序代码相似。
>什么是函数（Function）
>应用程序由代码组成，函数（Function）也是如此。那么函数（Function）有什么特别之处呢？难道它不是一个应用程序吗？应用程序一般由从前端UI到后端数据库的许多组件以及其间的所有处理组成。相比之下，函数通常只是一小段代码，具有单一目的，可以快速和异步地运行。它通常也由事件触发，而不是由用户在请求/响应场景中直接调用。

回想一下第6章中的Cloud Foundry Buildpacks示例。例6-1中显示的service.yaml文件引用了一个完整的Node.js Express应用程序，该应用程序的功能是在给定端口上侦听GET请求然后返回“Hello World”信息。如果我们的程序是接受数字作为输入，返回该数字的平方作为结果的函数，而不是Hello World应用程序呢？此代码可能类似于我们在示例8-1中看到的内容。
Example 8-1. knative-function-app-demo/square-app.js
```
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
我们可以使用示例6-1中的相同Buildpack来构建此函数并将其部署到Knative。又如例8-2，它也是使用Node.js编写的一个函数，它不是一个完整的Express应用程序，而仅仅由一个函数组成，不包含任何其他Node.js模块。

Example 8-2. knative-function-demo/square.js
```
module.exports = (x) => x ** 2
```
Knative支持这一点，因为它具有Build模块提供的灵活性。为了构建和部署这样的代码到Knative，需要一个自定义的构建模板将这个简单的仅含函数的代码转换为可运行的Node.js应用程序。例8-2中的代码使用了function invokers特别支持的编程模型，function invokers是riff项目一部分的。riff是Pivotal的一个开源项目，构建于Knative之上，它提供了一些很棒的东西：用于安装Knative和管理在其上部署的函数（functions）的CLI，以及使我们能够编写像例8-2中代码的function invokers。这些invokers负责执行函数（functions），例如我们见过的Node.js示例，或Spring Cloud Functions，甚至是Bash脚本。与Build模板一样，invokers是开源的，随着riff项目的成熟，这个列表会继续增长。请务必查看https：// project-riff.io了解更多信息！

## 拓展阅读
在您继续阅读和学习参考的过程中，围绕Knative有大量文档，示例和演示。最好的当然是GitHub仓库中Knative Docs，它不仅包含有关Knative的每一部分如何工作的详细说明，而且还有更多的演示和加入社区的链接，例如Knative Slack频道或邮件列表。
我们非常感谢您花在我们的书本上的时间，并希望对您开始上手使用Knative有帮助。我们可以留给您的最好建议就是要勤写代码并开始构建一些东西，无论大小。通过犯错并学习如何解决问题来探索和学习，与他人分享你学到的东西！ Knative的社区非常年轻，但成长速度非常快，我们希望看到你成为它的一员。