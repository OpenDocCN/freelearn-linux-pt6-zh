# 前言

NGINX 是一款高性能的 web 服务器，旨在使用极少的系统资源。网上有很多关于如何配置 NGINX 的教程和示例配置文件。本指南将帮助你理清 NGINX 配置的模糊领域。在此过程中，你将学习如何根据不同的情况调整 NGINX，了解一些较为晦涩的配置选项的作用，以及如何设计一个合适的配置以满足你的需求。

你将不再需要复制粘贴配置片段，因为你将理解如何构建一个配置文件来实现你想要的功能。这是一个过程，途中可能会遇到一些困难，但通过本书中解释的技巧，你将能够自信地手写 NGINX 配置文件。如果出现问题，你将能够自己调试问题，或者至少可以在请求帮助时，不会感觉自己没有尝试过。

本书以模块化的方式编写。它的结构旨在帮助你尽可能快速地找到所需的信息。每一章基本上都是一个独立的部分。你可以随时跳到任何你想深入了解的部分。如果你觉得遗漏了什么重要的内容，可以回过头去阅读前面的章节。它们的构建方式是帮助你一步步构建配置文件的。

# 本书内容概述

第一章，*安装 NGINX 和第三方模块*，教你如何在你选择的操作系统上安装 NGINX，以及如何将第三方模块包含在安装中。

第二章，*配置指南*，解释了 NGINX 配置文件的格式。你将学习不同上下文的用途，如何配置全局参数，以及 location 的用途。

第三章，*使用邮件模块*，探索了 NGINX 的邮件代理模块，详细讲解了其配置的各个方面。本章的代码中包含了一个示例认证服务。

第四章，*NGINX 作为反向代理*，介绍了反向代理的概念，并描述了 NGINX 如何充当这个角色。

第五章，*反向代理高级话题*，深入探讨了如何使用 NGINX 作为反向代理来解决扩展性和性能问题。

第六章，*NGINX HTTP 服务器*，描述了如何使用 NGINX 附带的各种模块来解决常见的 web 服务问题。

第七章，*NGINX 面向开发者*，展示了如何将 NGINX 集成到你的应用程序中，以便更快速地为用户提供内容。

第八章，*故障排除技巧*，探讨了一些常见的配置问题，如何调试问题并提出性能调优的建议。

附录 A，*指令参考*，提供了书中使用的配置指令的便捷参考，以及一些之前未涉及的其他指令。

附录 B，*重写规则指南*，描述了如何使用 NGINX 重写模块，并介绍了将 Apache 风格的重写规则转换为 NGINX 可以处理的规则的一些简单步骤。

附录 C，*社区*，介绍了可用于获取更多信息的在线资源。

附录 D，*持久化 Solaris 网络调优*，详细说明了在 Solaris 10 及以上版本中，如何持久化不同的网络调优更改。

# 本书所需的内容

任何现代的 Linux PC 都足以运行书中的代码示例。每章中使用代码示例的部分都会提供安装说明。基本上，它归结为：

+   **构建环境**：编译器、头文件等

+   **NGINX**：最新版本应该没问题

+   **Ruby**：最好从 [`rvm.io`](https://rvm.io) 安装

+   **Perl**：默认版本应该没问题

# 本书适用人群

本书适用于有经验的系统管理员或系统工程师，他们熟悉安装和配置服务器以满足特定需求。你不需要已经有使用 NGINX 的经验。

# 约定

在本书中，你会发现许多不同类型的信息，它们通过不同的文本样式来区分。以下是一些样式示例，以及它们的含义解释。

文本中的代码词汇如下所示：“如果你在配置中包括 `––with-<library>=<path>` 选项，NGINX 将尝试静态构建依赖库。”

代码块设置如下：

```
$ export BUILD_DIR=`pwd`
$ export NGINX_INSTALLDIR=/opt/nginx
$ export VAR_DIR=/home/www/tmp
$ export LUAJIT_LIB=/opt/luajit/lib
$ export LUAJIT_INC=/opt/luajit/include/luajit-2.0
```

当我们希望引起你对代码块中特定部分的注意时，相关行或项目会用粗体显示：

```
$ export BUILD_DIR=`pwd`
$ export NGINX_INSTALLDIR=/opt/nginx
$ export VAR_DIR=/home/www/tmp
$ export LUAJIT_LIB=/opt/luajit/lib
$ export LUAJIT_INC=/opt/luajit/include/luajit-2.0
```

任何命令行输入或输出都如下所示：

```
$ mkdir $HOME/build
$ cd $HOME/build && tar xzf nginx-<version-number>.tar.gz

```

**新术语** 和 **重要词汇** 用粗体显示。你在屏幕上看到的、在菜单或对话框中出现的词汇，例如，以这种方式出现在文本中：“点击 **下一步** 按钮可以将你带到下一屏幕”。

### 注意

警告或重要提示以类似于这样的框显示：

### 提示

提示和技巧通常是这样展示的。

# 读者反馈

我们始终欢迎读者的反馈。告诉我们你对这本书的看法——你喜欢什么，或者可能不喜欢什么。读者反馈对我们非常重要，它有助于我们开发出真正能让你获益的书籍。

要向我们提供一般反馈，只需发送电子邮件至 `<feedback@packtpub.com>`，并通过邮件主题提及书名。

如果你在某个领域拥有专业知识，并且有兴趣参与编写或为书籍贡献内容，请参阅我们的作者指南，网址为：[www.packtpub.com/authors](http://www.packtpub.com/authors)。

# 客户支持

既然你已经成为一本 Packt 图书的骄傲拥有者，我们为你提供了一些帮助，以便你能从购买中获得最大收益。

## 下载示例代码

你可以从 [`www.PacktPub.com`](http://www.PacktPub.com) 的帐户中下载你购买的所有 Packt 图书的示例代码文件。如果你是在其他地方购买的此书，可以访问 [`www.PacktPub.com/support`](http://www.PacktPub.com/support) 并注册，让我们直接通过电子邮件将文件发送给你。

## 勘误

虽然我们已尽一切努力确保内容的准确性，但错误是难免的。如果你发现我们书中的错误——可能是文本或代码中的错误——我们将非常感激你向我们报告。通过这样做，你可以帮助其他读者避免困扰，并帮助我们改进后续版本的书籍。如果你发现任何勘误，请通过访问 [`www.packtpub.com/support`](http://www.packtpub.com/support)，选择你的书籍，点击**勘误** **提交** **表单**链接，并输入勘误的详细信息。一旦勘误被验证，你的提交将被接受，并且勘误将上传至我们的网站，或添加到该书勘误列表中的勘误部分。你可以通过从 [`www.packtpub.com/support`](http://www.packtpub.com/support) 选择你的书名，查看任何现有的勘误。

## 盗版

网络上侵犯版权的盗版问题在所有媒体中都是一个持续存在的问题。在 Packt，我们非常重视保护我们的版权和许可。如果你在网上发现任何我们作品的非法复制版本，请立即提供网址或网站名称，以便我们采取相应的措施。

如有疑问，请通过 `<copyright@packtpub.com>` 联系我们，并附上涉嫌盗版材料的链接。

我们感谢你在保护我们的作者和我们提供有价值内容的能力方面所做的贡献。

## 问题

如果你在书的任何部分遇到问题，可以通过 `<questions@packtpub.com>` 与我们联系，我们将尽力解决问题。
