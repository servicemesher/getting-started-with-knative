# 规范

本文中指定了该文档的翻译规范与注意事项。

## 文档组织规则

- 所有文档使用 markdown 格式撰写；
- 所有图片都放在最顶层的 `images` 目录下，使用相对路径引用；
- 所有代码需要指明语言；

## 翻译流程

1. 首先加入到 ServiceMesher 组织中，请联系 [Jimmy Song](https://jimmysong.io/about) 加入。
2. 在 [Issues](https://github.com/servicemesher/getting-started-with-knative/issues) 中找到你想翻译的文档的标题（对应的文档路径即为标题），在 issue 中回复“认领”，确认不要与他人认领的重复。
3. 一次可以认领一个章节，也可以和其他人共同认领一个章节，但需要确定好分工，翻译完成后提交 PR。
4. 由 owner 审核后 merge 进 master 分支。
5. 每篇文章头部的 header 请注意填写。

## Header

每篇文章的头部都有一个使用 YAML 格式的 header，请在每次提交 PR 的时候填写，示例：

```yaml
owner: ["rootsongjc","loverto"]
reviewer: ["fleeto"]
description: "本章是全书的前言，用于说明本书的成因、划定目标读者、致谢等。"
publishDate: 2019-03-02
updateDate: 2019-03-03
```

## 注意事项

- 一次性只能认领一篇文章，待翻译完成后再认领其他文章；
- 尽量按照章节顺序领取；
- 若认领后超过一周没有提交 PR 则重新分配；

## 翻译规范

- 所有的英文跟中文之间要有一个空格

## 专有名词

下面的词汇作为专有名词，可以不翻译，或在出现时，同时在括号中注明翻译，如 `Servering（服务）`。

- Servering
- Eventing
