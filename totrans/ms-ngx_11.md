# 附录 C. NGINX 社区

NGINX 不仅得到充满活力的社区支持，现在也有一家公司支持它。NGINX 的原始作者 Igor Sysoev 于 2011 年与其他人共同创办了 NGINX, Inc.，为使用 NGINX 的公司提供专业支持。尽管如此，他和其他 NGINX 开发人员仍然为社区提供支持。本附录概述了在线可用的社区资源。

本附录涵盖以下主题：

+   邮件列表

+   IRC 频道

+   Web 资源

+   编写正确的错误报告

# 邮件列表

自 2005 年以来，`nginx@nginx.org` 邮件列表一直活跃。订阅该列表并查看提出的问题及其回答的方式是获取帮助的最佳途径。在提问之前，请先在线搜索答案。FAQ 也在 [`wiki.nginx.org/Faq`](http://wiki.nginx.org/Faq)。通过搜索 [`mailman.nginx.org/pipermail/nginx/`](http://mailman.nginx.org/pipermail/nginx/) 查找最近是否有人提出相同的问题。如果最近有人提出相同的问题，不仅会让您尴尬，也会让列表的读者感到烦恼。

# IRC 频道

IRC 频道 `#nginx` 位于 `irc.freenode.net`，是一个实时资源，供有兴趣了解开发人员并获得有帮助的简短查询响应的人使用。访问频道时，请遵守 IRC 礼仪。更大的文本块如配置文件或编译输出应放入 Pastebin，并只复制其 URL 到频道中。有关频道的更多详细信息，请访问 [`wiki.nginx.org/IRC`](http://wiki.nginx.org/IRC)。

# Web 资源

[`wiki.nginx.org`](http://wiki.nginx.org) 的 wiki 多年来一直是一个有用的资源。在这里，您将找到完整的指令参考、模块列表和许多配置示例。请记住，这是一个 wiki，因此所提供的信息不能保证准确、最新或完全符合您的需求。正如本书所见，始终在着手解决问题之前考虑您想要实现的目标非常重要。

NGINX, Inc. 维护位于 [`nginx.org/en/docs/`](http://nginx.org/en/docs/) 的官方参考文档。有一些介绍 NGINX 的文件，以及如何的指南和描述每个模块和指令的页面。

# 编写良好的错误报告

在线寻求帮助时，能够撰写良好的错误报告非常有用。如果能以清晰、可重现的方式表达问题，答案将更容易得到。本节将帮助您做到这一点。

错误报告的最困难部分实际上是定义问题本身。首先考虑您试图实现的目标将有所帮助。请如下清晰而简明地陈述您的目标：

```
I need all requests to subdomain.example.com to be served from server1.
```

避免以以下方式编写报告：

```
I'm getting requests served from the local filesystem instead of proxying them to server1 when I call subdomain.example.com.
```

你能看到这两句话之间的区别吗？在第一种情况中，你可以清楚地看到有一个特定的目标。在第二种情况中，更描述的是问题的结果，而不是目标本身。

一旦问题被定义，下一步就是描述如何重现该问题：

```
Calling http://subdomain.example.com/serverstatus yields a "404 File Not Found".
```

这将帮助任何查看此问题的人尝试解决它。它确保存在一个未解决的案例，可以在问题解决后证明其已解决。

接下来，描述观察到问题的环境是有帮助的。有些错误只会在特定操作系统或依赖库的某个版本中出现。

任何必要的配置文件，若要重现问题，都应包含在报告中。如果在软件存档中找到了某个文件，那么引用该文件即可。

在发送你的错误报告之前，一定要先阅读一遍。通常，你会发现有些信息被遗漏了。有时候，你会发现自己通过清晰地定义问题，实际上已经解决了问题！

# 概述

在本附录中，我们了解了 NGINX 背后的社区。我们看到了主要的参与者是谁，以及有哪些在线资源可用。我们还深入了解了如何编写一个有助于解决问题的错误报告。
