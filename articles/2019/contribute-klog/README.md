# 给Kubernetes下的klog项目贡献代码

先前写过一篇glog源码阅读的[文章](./read-google-logging-library-glog)，在使用的过程中发现一些体验问题，偶然发现Kubernetes项目其实已经fork了glog维护着一个[klog](https://github.com/kubernetes/klog)项目，实际上Kubernetes使用klog作为其日志库。看了下，刚好也解决了默认所有配置都在`init()`中初始化带来的问题。

此外，还新引入了一个`log_file`的选项，允许将所有的日志输出到一个单独的文件上，这对调试来说更方便一些。于是尝试了一下，发现example并没有输出文件日志，比较困惑，还是翻看了下源码，发现日志默认是打在`stderr`上的，比如要将`logtostderr`设置为false才会有文件日志。

困惑消除了，解决办法也有了，但是觉得example的代码比较误导新人：如果不看代码，是很难发现这个细微的问题的。于是想着要不我也给这项目提交一个fix?

终于，在接下来一个星期的某个中午，午饭后看了一下[contribution guide](https://github.com/kubernetes/community/tree/master/contributors/guide)就开干了。

先fork了仓库到本地，按照规范的style添加了一行代码和一行注释，commit log还是琢磨了一下，参考klog项目前人的提交记录，第一行标题简要的写做了什么，然后详情中写变更原因和解决办法，以及`Signed-off-by`标签。现在想来，平常的项目commit log提交是只有标题，还是很随意的，对于回溯，好像就压根儿没有过。

接下来就在GitHub上发起了一个Pull Request，照着模版填写了一些信息提交，然后就是一系列的bot自动化验证cla是否签署和代码是否通过测试。cla好办，对于个人贡献者只需要去Linux foundation创建一个账号并签署协议然后和GitHub绑定。

一系列的检查OK后，机器人会提示你使用`/assign @github_id`指令去请求某个项目的核心贡献者去review你的代码，第一次用很高级。看了下，那些reviewers都是Google的大牛，诚惶诚恐。

到了晚上，也就是西方的工作时间，收到了review的邮件，让我改一下注释，并且给了注释应该的样子。确实，原来的注释英语不地道，修改后的可读性高多了，既说明了问题的原因，还说明了解决的办法。改完后，提交然后继续请求review。

没过过久，又收到了邮件，要求我把两条提交合并成一条。`rebase squash`这些命令很少用到，我们平时虽然也是用分支合并代码，但是几乎没有要求在合并的时候压缩commit，所以history看起来总是乱糟糟的。深深地感觉这些开源项目的流程概念很超前，很牛皮。

最后，来外很nice的approve了我的PR，还给我点了个赞，哈哈。

很荣幸，虽然只是Kubernetes下的一个小项目，虽然只有一行代码和一行注释。整个过程中，从阅读源代码，fix bug，写commit log，以及review中各种细节以及自动化流程，比起平时写的各种代码，这才是真正地做项目。

整个过程可以在 <https://github.com/kubernetes/klog/pull/103> 看到，一切都是开放的。

在开源中学习，并做出贡献。

## EOF

```yaml
title: 给Kubernetes的klog项目提交代码
summary: 在开源中学习，并做出贡献。
weather: fine
license: cc-40-by
location: 22, 114
background: ./k8s.jpg
tags: [Kubernetes, open source]
date: 2019-10-20T21:56:59+08:00
```
