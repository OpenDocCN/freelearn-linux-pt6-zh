# 第四章：优化网站性能

迁移到 Nginx 最受欢迎的原因之一是追求更好的性能。多年来，Nginx 因其卓越的性能而获得了一定的声誉，被誉为“银弹”和“速度怪兽”。有时，这种声誉可能会对项目产生负面影响，但它无疑是实至名归的。在许多情况下，确实会发生这样的事情：你*添加* Nginx 到网站配置中，就像添加配料一样，网站就奇迹般地变得更快。我们不会解释如何设置 Nginx 的基础知识，因为你可能已经很熟悉这些内容了。在本章中，我们将深入探讨为什么会发生这种情况，以及有哪些不太为人所知的选项可以帮助你从网站中挤出更多性能。

本章将涵盖以下主题：

+   Nginx 如何处理请求

+   Nginx 缓存子系统

+   优化上游服务器

+   一些新特性，例如线程池

+   其他性能问题

大多数使用 Nginx 的网站在性能上遇到的问题，其实大多出现在上游服务器上。我们会尽量提到一些你可以用来优化上游应用服务器的办法，但我们主要会集中讨论 Nginx 本身。你需要理解 Nginx 内部的工作原理以及反向代理的一般原理，我们将花费章节的一大部分来解释 Nginx 中的原理，正是这些原理让 Nginx 在性能上超越了其他一些较旧的 Web 服务器。

坏消息是，你可能无法对 Nginx 做太多优化。如果你已经着手通过在应用程序与用户之间插入 Nginx 来显著加速网站，那么你可能已经走完了实现目标的最重要步骤。Nginx 在避免做额外、不必要的工作方面极为高效，而这正是任何优化的核心。

仍然，一些默认配置可能为了兼容性而显得过于保守，我们将尝试讨论这一点。

# 为什么 Nginx 如此快速？

这个问题故意以过于简化的方式提出来。这就是你可能从老板或客户那里听到的话——让我们从旧技术迁移到 Nginx，因为它会让我们的网站更快，用户更开心。迁移过程在成千上万的在线文章甚至一些书籍中都有描述，我们在这里不会再写这部分内容了。我们很多读者可能已经走过这条路很多次，并且知道事实：首先，网站通常变得更快；其次，不，通常不是完全的迁移。你很少会完全弃用 Apache，直接用 Nginx 替换。虽然这种“完全转换”也会发生，但大多数情况下，你是将 Nginx 插入到 Apache 和互联网之间。要理解为什么这样做是可行的，为什么这会有帮助，并且如何从这里继续推进，继续阅读。

为了描述使用 Nginx 作为反向代理所实现的主要概念性变化，我们将简单地使用 Apache 1.x 的处理模型，也就是一种非常古老的软件，采用了预多线程传统。最新的 Apache 版本，即 2.x，可能使用另一种稍微更高效的模型，它是基于线程而不是进程的。但与 Nginx 相比，这两种模型看起来非常相似，而且较老的模型更容易理解。

这是一个简单的图示，展示了一个 HTTP 请求-响应对是如何被处理的：

![为什么 Nginx 如此快速？](img/B04329_04_01.jpg)

这里是图表的解释：

1.  用户的浏览器使用 TCP 打开与服务器的连接。

1.  一款在你的服务器上运行并监听特定 TCP 端口的 web 服务器软件，它接受连接，将一部分资源分配给该连接进行处理，处理完后再返回继续监听并接受其他传入的连接。对于 Apache 1.x 模型而言，这部分被分离出的资源是一个已经提前创建的子进程，并在进程池中等待。

1.  通常会对并发连接的处理数量设置一些限制，并在此步骤进行强制执行。理解这一点非常重要，因为这是扩展的关键所在。

1.  这部分专门的 web 服务器软件读取实际的请求 URI，解析它，并找到相关的文件或其他方式生成响应。也许生成的响应会是一个错误信息，但这并不重要。它开始将响应数据发送到连接中。

1.  用户的浏览器逐个接收响应字节，并在用户的屏幕上生成像素。这实际上是一个很长的工作。数据要通过数百公里的电线和光纤传输，转换为电磁波传播到空中，再通过感应转化为电流。从服务器的角度看，绝大多数用户都处于极其缓慢的网络环境中。Web 服务器实际上是在通过一根吸管向这些浏览器输送大量数据。

第五点是无法解决的。最后一公里总是你服务器与用户之间链路中最慢的环节。Nginx 在第二步进行了概念性的优化，从而能够更好地进行扩展。让我们详细解释一下这一点。

由于客户端连接较慢，在任何特定时间点，任何流行网站的服务器软件快照看起来大致是这样的：几个请求实际上已经在处理，即 CPU、内存和磁盘正在进行一些重要工作；然后是几千个请求，所有处理都已经完成，响应已生成，但响应会非常慢地、逐步地通过狭窄的连接发送到用户的浏览器。再次强调，这是一个简化模型，但足以说明实际发生的情况。

为了在第二步中实现扩展，原始的 Apache 1.x 使用了一种非常适合所有 UNIX 系统的机制——它进行 fork 操作。有一些优化，比如通过预先创建一组已经 fork 出来的进程池（因此有了“预先 fork”模型），而 Apache 2.x 可能使用线程代替进程（同样带有预生成的池等），但核心思想是相同的：通过将单个请求分配给一组操作系统级的实体来实现扩展，每个实体都能处理一个请求，然后将数据发送到客户端。问题在于这些实体相对庞大；你不仅仅需要一组，而是需要一大群，而且大多数时候它们执行的任务非常简单：将字节从缓冲区发送到 TCP 连接。

Nginx 和其他基于状态机的服务器通过不让大而复杂的操作系统级进程或线程执行简单任务，同时又占用大量内存，显著优化了第二步的处理。这就是 Nginx 突然能让你的网站变得更快的本质——它能够用极少的内存慢慢地处理成千上万带宽受限的客户端连接，节省了大量的 RAM。

好奇的读者可能会问，为什么在不移除 Apache 的情况下，添加 Nginx 作为反向代理仍然能节省内存并加速网站。我们相信，你应该已经具备足够的知识来得出正确的答案。我们将提到最重要的部分作为提示：不再需要一大堆 Apache，因为 Apache 只负责生成响应——最聪明、最复杂的部分——而将将字节推送到成千上万慢速连接的简单任务交给反向代理。反向代理充当所有用户浏览器的代理客户端，且有一个非常重要的区别：这个客户端离服务器非常近，能够以闪电般的速度接收响应的字节。

所以，Nginx 性能的秘诀并非其神奇的代码质量（尽管它写得非常好），而在于它通过不为每个单独的请求复制大量数据，从而节省了系统资源，主要是内存。有趣的是，现代操作系统都有不同的低级机制来避免过度复制数据。早已不再是 `fork()`  literally 创建一个代码和数据的完整副本的时候了。随着虚拟内存和网络子系统的不断发展，我们最终可能会遇到一种系统，其中不再需要将状态机作为模型来编写紧凑的事件处理循环。目前，它们仍然能带来显著的性能提升。

# 优化单个上游

您可能还记得前几章提到过，Nginx 有两种主要的生成响应请求的方法，一种非常具体，从文件系统读取静态文件，另一种包括一整套所谓的上游模块。上游是 Nginx 代理请求的外部服务器。最流行的上游是 `ngx_proxy`，其他还有 `ngx_fastcgi`，`ngx_memcached`，`ngx_scgi` 等等。因为仅提供静态文件通常对现代网站来说不足够，上游是任何综合设置的重要部分。正如本章开头提到的，上游本身通常是您的网站性能问题的原因。您的开发人员对此负责，因为这是所有 Web 应用程序处理发生的地方。在接下来的几节中，我们将简要描述用于在 Nginx 后面实现业务逻辑的主要堆栈或平台以及您至少应该寻找优化线索的方向。

## 优化静态文件

任何 Web 应用程序都将包含静态资源，这些资源不会改变也不依赖于当前使用应用程序的用户。这些通常在网络管理员术语中称为静态文件，并包括所有静态图像、CSS、JavaScript 以及例如 `cross-domain.xml` 这样的其他额外数据，它们用于浏览器的访问控制策略。通常支持直接从应用程序中提供数据，以促进没有任何前端、中间加速服务器（如 Nginx）的简单设置。Nginx 内置的 HTTP 代理将乐意为其提供服务，并且在本地缓存的情况下，甚至可能在没有任何可察觉的性能损失的情况下完成。然而，如果您追求最大性能，长期采用这样的设置并不推荐。

我们强烈建议（或提醒）的一个普遍步骤是将尽可能多的静态数据从上游移到 Nginx 的控制之下。这将使您的应用程序更加分散，但也将是一种非常好的性能优化方法，胜过许多其他潜在且难以实现的方法。如果您的上游服务器提供静态文件，则需要将它们作为文件提供给 Nginx 并直接提供服务。当您收到一个新的遗留上游以进行优化时，这可能是您首先要做的事情。这也是一个非常容易自己完成或作为整个部署过程的一部分实现的任务。

## 优化 PHP 后端

多年来，运行 PHP 应用程序在 Nginx 前端的现代方式是使用 PHP-FPM 或 FastCGI 进程管理器。正如您可能猜到的那样，它使用 FastCGI 协议，并且需要在 Nginx 中使用 FastCGI 上游模块。然而，当处理继承的遗留 PHP 网站时，您可能仍然会遇到运行代码的旧方式，这将是您首先要优化的候选方案。

这是使用官方 Apache 方式，通过`mod_php` Apache 模块。该模块将 PHP 解释器直接嵌入到（每一个！）Apache 进程中。大多数情况下，你将继承配置为以这种方式运行的 Apache 网站。嵌入式解释器的主要优点是显而易见的——代码可以在请求之间以某种中间形式保存，而不是每次都重新解释。`mod_php` Apache 模块做得非常好，一些人称它为 PHP 在 Web 上获得流行的唯一原因。嗯，2016 年处理`mod_php`的方式是摆脱它，连同 Apache 一起。

许多 PHP 代码库可以几乎毫不费力地从`mod_php`迁移到 PHP-FPM。之后，你将改变主 Nginx 上游，从 HTTP 代理转变为直接使用 FastCGI 协议与通过 FPM 保持运行并随时准备好的 PHP 脚本通信。

有时候，你的开发人员需要投入一些资源，主要是重构和调整代码，使其能够在没有 Apache 帮助的情况下在独立进程中运行。一个特别困难的情况是，代码严重依赖于调用 Apache 内部功能。幸运的是，这在 PHP 代码库中不像在`mod_perl`代码库中那样常见。稍后我会提到如何处理基于 Perl 的网站。

另一种非常古老（且奇怪）的方法是运行 PHP 的 CGI 脚本。每一个网站管理员都做过或将会做大量的临时 CGI 脚本。你知道的，那种通过一代又一代的硬件、管理者和商业模式延续下去的临时脚本。它们很少用于面向用户的生产部分。总之，CGI 在 PHP 中并不受欢迎，因为`mod_php`和 Apache 的普及以及相当不错的质量。然而，你的遗留代码中可能会有一些 CGI 脚本，特别是如果这些代码曾经或现在有可能在 Windows 上运行的话。

CGI 脚本作为每个请求/响应对的独立进程执行，因此成本非常高。使用 CGI 的唯一好处是提高了与其他 Apache 模块的兼容性和另外一种特权分离的程度。除了在最特殊的情况下，性能的折衷完全超越了这些优点。顺便说一句，Nginx 会通过缓存输出并释放后端资源，显著提升你网站的 CGI 功能部分。你仍然需要尽早规划重写这些部分，使其能作为 FastCGI 在 FPM 下运行。

PHP-FPM 使用与 Apache 1.x 相同的预分叉（prefork）模型，这使得你可以控制一些熟悉的参数。例如，你可以配置 FPM 启动的工作进程数、每个子进程可以处理的请求上限，以及可用的子进程池大小。所有这些参数都可以通过`php-fpm.conf`文件进行设置，该文件通常直接安装在`/etc`目录下，并遵循良好的约定，包含`/etc/php-fpm.d/*.conf`。

## Java 后端

Java 生态系统如此庞大，专门讨论各种 Java Web 服务器的书架几乎都有一整本。我们无法深入探讨这个话题。如果你作为管理员从未接触过 Java Web 应用，你会很高兴地知道，大多数情况下，这些应用程序运行着自己独立的 Web 服务器，并不依赖 Apache。这是你可能在上游遇到的流行 Java Web 服务器的列表：Apache Tomcat、Jetty 和 Jboss/WildFly。Java 应用通常基于庞大且全面的框架构建，其中 Web 服务器只是一个组成部分。你的 Nginx Web 加速器将通过正常的 HTTP 协议与 Java 上游进行通信，使用 `ngx_proxy` 模块。因此，所有常见的 `ngx_proxy` 优化都会适用。稍后在本章中，关于缓存的说明会提供更多示例。

如果不深入到代码内部，几乎无法做任何事情来提高 Java 应用程序的性能。系统管理员层面可以采取的一些步骤包括：

1.  选择合适的 JVM。许多 Java Web 服务器支持几种不同的 Java 虚拟机实现。Oracle（Sun）提供的 HotSpot JVM 被认为是最好的之一，你可能会从这个开始。但也有其他的虚拟机实现，其中一些是商业化的，比如 Azul Zing VM。它们可能会为你提供一些性能提升。不幸的是，更换 JVM 供应商是一个重大步骤，容易导致不兼容错误。

1.  调整线程参数。Java 代码通常使用线程来编写，这是该语言的一个本地且自然的特性。JVM 可以自由地使用任何资源来实现线程。通常，你可以选择使用操作系统级别的线程，或者所谓的“绿色线程”，它们是在用户空间实现的。这两种方法各有优缺点。线程通常会被分组到线程池中，这些线程池的预启动方式与 Apache 1.x 在进程上所做的类似。线程池使用多种模型来优化内存和性能，作为管理员的你将能够对它们进行调整。

## 优化 Python 和 Ruby 后端

Python 和 Ruby 都是在 Web 应用已经成为部署业务逻辑的主要方式时，作为比 Perl 和 PHP 更开放、更清晰的替代方案而崛起的。它们都是后来才起步，并且有一个明确的目标——成为非常适合 Web 的语言。曾经有 `mod_python` 和 `mod_ruby` 这样的 Apache 模块，它们将解释器嵌入到 Apache Web 服务器进程中，但它们很快就过时了。Python 社区开发了**Web 服务器网关接口**（**WSGI**）协议，作为一种通用的接口，可以编写与部署选项无关的 Web 应用程序。这使得在实际 Web 服务器领域的创新得以自由进行，最终大多集中在几个独立的 WSGI 服务器或容器上（例如 gunicorn 和 uWSGI）以及 `mod_wsgi` Apache 模块。它们都可以用来运行 Python Web 应用，而无需修改任何代码。

因此，Nginx 开发了自己的 WSGI 上游模块 `ngx_wsgi`，这是你应该使用的模块，替代任何其他的 WSGI 实现。实际的迁移路径可能会更复杂。如果后端应用曾经在 Apache + `mod_wsgi` 下运行，那么，毫不犹豫地切换到 `ngx_wsgi` 并放弃 Apache。否则，为了平滑和稳定，你可以从更简单的 `ngx_proxy` 配置开始，然后再转向 `ngx_wsgi`。

你也可能遇到使用长轮询（有时称为 Comet）和 WebSockets 的应用程序，并且运行在一个特殊的 Web 服务器上，例如 Tornado（因 FriendFeed 而闻名）。这些问题主要在于 Web 服务器与客户端之间的同步通信，阻碍了 Nginx 作为加速反向代理的主要优势——处理请求的服务器部分无法通过处理字节推送到 Nginx 前端迅速为另一个请求提供服务。当然，现代的 Nginx 支持代理 Comet 请求和 WebSockets，但没有你可能已经习惯的加速效果。

Ruby 生态系统走了一条略有不同的道路，因为 Ruby 有一个所谓的“杀手级应用”，那就是 Ruby on Rails Web 应用框架。大多数 Ruby Web 应用都是基于 Ruby on Rails 构建的，甚至有人开玩笑说该把整个语言改名为 Ruby on Rails，因为没有 Rails，没人会用 Ruby。它是一个设计精妙、执行出色的 Web 框架，拥有许多革命性的理念，启发了整个行业的快速应用开发技术浪潮。它还通过提供可以立即在互联网上共享的 Web 服务器，将应用开发人员与部署工作中的问题解耦。

当前 Ruby on Rails 推荐的部署选项是使用 Phusion Passenger 或者运行一组 Unicorn web 服务器。两种方式都适用于将系统迁移到 Nginx 的任务。Phusion Passenger 是一个成熟的例子，它提供了自己的内嵌代码，包含了 Apache 和 Nginx web 服务器的模块。所以，如果运气好，你可以毫不费力地在两者之间切换。Passenger 会在你的主 Nginx 工作进程外运行工作进程，但该模块允许 Nginx 自由地进行通信。这是一个自定义上游模块的好例子。查看 [`www.phusionpassenger.com/library/deploy/nginx/deploy/ruby/`](https://www.phusionpassenger.com/library/deploy/nginx/deploy/ruby/) Passenger 指南获取实际的部署说明。Passenger 还可以以独立模式运行，向外暴露 HTTP 服务。这也是 Unicorn 部署 Ruby 应用的方式。你知道如何处理这个问题——通用助手 `ngx_proxy`。

## 优化 Perl 后端

Perl 是第一个广泛用于 Web 服务器端编程的语言。我们可以说，正是 Perl 将动态生成网页的概念推广开来，为今天的 Web 应用程序的蓬勃发展铺平了道路。如今仍然有许多 Perl 支持的 Web 企业，规模不等，从像 [`www.booking.com`](https://www.booking.com) 这样的大型公司，到像 DuckDuckGo 这样的小型、充满活力且雄心勃勃的初创公司。你可能也见过几个 MovableType 支持的博客。这是一个由 SixApart 开发的专业博客平台，后来被多次转售。

Perl 也是编写 CGI 脚本最流行的语言，这也是它被认为较慢的唯一原因。CGI 是一个简单的接口，用于从 Web 服务器内部运行外部程序。它相当低效，因为通常涉及到分叉一个操作系统级进程，然后在处理完一个请求后关闭它。这种模式加上 Perl 的解释性特性意味着 Perl CGI 脚本效率低下，成为低效 Web 开发平台的代表。

如果你有一个由 CGI 脚本生成的面向用户的动态网页，并且该脚本是由 Apache 运行的，那么你必须将其移除。具体细节请见下文。

有多种更先进的方法可以在生产环境中运行 Perl 代码。部分受到`mod_php`成功的启发，有一个长期运行的项目叫做`mod_perl`，它是一个将 Perl 解释器嵌入 Apache 进程中的 Apache 模块。它同样非常成功，因为它稳定且健壮，支持许多高负载网站的运行。然而，它也相当复杂，对于开发者和管理员来说都不容易。与 `mod_php` Apache 模块的另一个不同之处在于，`mod_perl`未能提供强有力的环境隔离，而这对于虚拟主机业务至关重要。

无论如何，如果你继承了一个基于`mod_perl`的网站，你有几个选择。首先，可能有一种便宜的方式可以迁移到 PSGI 或 FastCGI 模型，这将使你能够摆脱 Apache。模块 Apache::Registry，它在`mod_perl`中模拟了 CGI 环境，可能是这种情况的一个良好迹象。其次，代码可能是以与 Apache 紧密耦合的方式编写的。`mod_perl`模块提供了一个接口，可以深入钩入 Apache 的内部，尽管为开发者提供了几个有趣的功能，但也使得迁移变得更加困难。开发者将不得不调查软件中使用的方法，并做出最终决定。他们可能决定保留 Apache + `mod_perl`并继续使用它作为一个重型且功能过剩的进程管理器。

如今将 CGI 迁移到`mod_perl`从来不是一个好的选择，我们不推荐这样做。

有许多类似于 PHP-FPM 的 Perl FastCGI 管理器。作为 Nginx 管理员，这些选项对你来说是非常幸运的，因为大多数情况下迁移会非常顺利且简单。

运行 Perl 代码在 Web 服务器中的一种有趣的最新方式是所谓的**Perl 服务器网关接口**（**PSGI**）。它或多或少是将 Ruby 堆栈中的 Rack 架构直接移植到 Perl 的版本。有趣的是，PSGI 是在 Nginx 已经流行的世界中发明并实现的。因此，如果你有一个使用 PSGI 的 Web 应用程序，它很可能已经在 Nginx 后面进行了测试和运行。无需移植任何东西。PSGI 可能是升级 CGI 或`mod_perl`应用程序的最重要的目标架构。

较大的 Perl Web 框架通常提供多种方式来运行应用程序。例如，Dancer 和较早的 Catalyst 都提供了将同一个应用程序作为独立 Web 服务器运行的连接脚本（你可以借助 Nginx 的`ngx_proxy`上游将其暴露给外部），也可以作为`mod_perl`应用程序，甚至作为 CGI 脚本运行。并非所有这些方法都适合生产环境，但它们肯定会在迁移过程中有所帮助。在权衡其他选择之前，永远不要接受开发者的“我们应该从头重写所有内容”这一建议。如果应用程序是在过去 3 至 4 年内编写的，它应该直接或通过其框架实现了 PSGI。

PSGI 应用程序通过特殊的 PSGI 服务器运行，如 Starman 或 Starlet，这些服务器对外使用简单的 HTTP 协议。Nginx 将使用`ngx_proxy`上游来处理这些应用程序。

# 在 Nginx 中使用线程池

使用异步事件驱动架构非常适合 Nginx，因为它可以在处理成千上万、甚至数百万个慢速客户端的独立连接时节省宝贵的内存和 CPU 上下文切换。不幸的是，像 Nginx 这样的事件循环，在面对阻塞操作时很容易失败。Nginx 最初诞生于 FreeBSD，它相对于 Linux 有几个优势，其中之一就是强大的异步输入/输出实现。基本上，操作系统内核通过拥有自己的内核级后台线程，能够避免在传统的阻塞操作（如从磁盘读取数据）上发生阻塞。而 Linux 需要更多来自应用程序的工作，最近在版本 1.7.11 中，Nginx 团队发布了自己的线程池功能，以便更好地在 Linux 上运行。你可以在这篇官方 Nginx 博客文章中找到问题和解决方案的详细介绍：[`www.nginx.com/blog/thread-pools-boost-performance-9x/`](https://www.nginx.com/blog/thread-pools-boost-performance-9x/)。我们将提供一个你可以在 Web 服务器上启用线程池的配置示例。记住，只有在 Linux 上你才需要这个功能。

为了启用后台线程，这些线程将在不阻塞主事件循环的情况下执行阻塞的输入/输出操作，你可以这样使用 `aio` 指令：

```
server {
    location /file {
        root /mnt/huge-static-storage;
        aio threads;
    }
}
```

你可能知道 `aio` 指令，它用来启用异步 I/O 接口，因此它的使用扩展成这样是非常自然的。

从一个非常高的层面来看，Nginx 的实现非常简单易懂。对你来说，Nginx 会运行一组后台的、用户级的线程来完成输入/输出任务，而主事件循环则会继续运行，并等待慢速磁盘或网络。

# Nginx 的缓存层

如果有一种被普遍认可并推崇的加速算法，那就是缓存。实用来说，缓存是一个避免多次重复做相同工作过程。理想情况下，每个独立的计算单元应该只执行一次。当然，这在现实世界中从未发生过。不过，通过重新安排工作或使用已保存的结果来减少重复的技术是非常流行的，它们形成了一个庞大的学科，名为“动态规划”。

在 Web 服务器的上下文中，缓存通常意味着将生成的响应保存在文件中，这样下次收到相同请求时，就可以通过读取该文件来处理，而不需要再次计算响应。现在请参考本章第一部分概述的步骤。对于许多实际的网页，响应的实际计算并不是瓶颈；将这些响应传输到慢速客户端才是瓶颈。这就是为什么最有效的缓存通常发生在浏览器端，或者开发者更喜欢说的，是在客户端。

## 发出缓存头

所有浏览器（甚至许多非浏览器的 HTTP 客户端）都支持客户端缓存。它是 HTTP 标准的一部分，尽管它是最复杂的部分之一。显然，Web 服务器不能完全控制客户端缓存，但它们可以通过特殊的 HTTP 响应头以推荐的方式告诉客户端缓存什么以及如何缓存。这是一个在许多优秀文章和指南中深入讨论的话题，因此我们将在此简要提及，重点是您可能面临的问题以及如何进行故障排除。

尽管浏览器至少在过去 20 年里就已支持客户端缓存，但配置缓存头总是有些令人困惑，主要是因为有两套为相同目的设计的头部，它们的作用范围不同，格式也完全不同。

有一个`Expires:`头，它被设计为一个快速且粗糙的解决方案，还有一个新的（相对来说）几乎全能的`Cache-Control:`头，它试图支持所有可能的 HTTP 缓存工作方式。

这是一个现代 HTTP 请求-响应对的示例，包含了缓存头。这些是从浏览器（此处为 Firefox 41，但版本无关紧要）发送的请求头：

```
User-Agent:"Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:41.0) Gecko/20100101 Firefox/41.0"
Accept:"text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8"
Accept-Encoding:"gzip, deflate"
Connection:"keep-alive"
Cache-Control:"max-age=0"

```

然后，响应头为：

```
Cache-Control:"max-age=1800"
Content-Encoding:"gzip"
Content-Type:"text/html; charset=UTF-8"
Date:"Sun, 10 Oct 2015 13:42:34 GMT"
Expires:"Sun, 10 Oct 2015 14:12:34 GMT"
Server:"nginx"
X-Cache:"EXPIRED"
```

我们已经突出了相关部分。请注意，某些指令可能由对话的双方发送。浏览器发送了`Cache-Control: max-age=0`头，因为用户按下了*F5*键。这表示用户希望接收一个新的响应。通常，请求不会包含此头，而是允许任何中间缓存返回一个过时但尚未过期的响应。

在这种情况下，我们与之交互的服务器返回了一个经过 gzip 压缩并采用 UTF-8 编码的 HTML 页面，并指出该响应可以使用半小时。它使用了两种可用的机制，现代的`Cache-Control:max-age=1800`头和非常老旧的`Expires:Sun, 10 Oct 2015 14:12:34 GMT`头。

`X-Cache: "EXPIRED"`头不是一个标准的 HTTP 头，但很可能（无法从外部确认）由 Nginx 发出的。它可能表示客户端和服务器之间确实有中间缓存代理，其中之一为调试目的添加了此头。该头也可能表明后端软件使用了某些内部缓存。

另一个可能的来源是用于调试技术，帮助找出 Nginx 缓存配置中的问题。其思路是使用缓存命中或未命中的状态，这些状态可以通过 Nginx 内部的某些变量获取，作为额外头的一部分，然后您可以从客户端监控这一状态。这段代码将添加这样一个头：

```
add_header X-Cache $upstream_cache_status;
```

Nginx 有一个特殊的指令，可以透明地设置标准的缓存控制头，它的名称是`expires`。这是使用`expires`指令的`nginx.conf`文件的一部分：

```
location ~* \.(?:css|js)$ {
  expires 1y;
  add_header Cache-Control "public";
}
```

该模式使用了所谓的非捕获括号，这是 Perl 正则表达式中首次出现的特性。这个正则表达式的效果与更简单的 `\.(css|js)$` 模式相同，但正则表达式引擎被特别指示不创建包含括号内实际字符串的变量。这是一个简单的优化。

然后，`expires` 指令声明 `css` 和 `js` 文件的内容将在存储一年后过期。客户端接收到的实际头部将如下所示：

```
Server: nginx/1.9.8 (Ubuntu)
Date: Fri, 11 Mar 2016 22:01:04 GMT
Content-Type: text/css
Last-Modified: Thu, 10 Mar 2016 05:45:39 GMT
Expires: Sat, 11 Mar 2017 22:01:04 GMT
Cache-Control: max-age=31536000
```

最后的两行包含相同的信息，但形式上大相径庭。`Expires:` 头的时间是 `Date:` 头时间的恰好一年后，而 `Cache-Control:` 则指定了以秒为单位的年龄，这样客户端就可以自行进行日期运算。

提供的配置片段中的最后一个指令添加了另一个 `Cache-Control:` 头部，值为 `public`。这意味着 HTTP 资源的内容没有访问控制，因此不仅可以为特定用户缓存，还可以在其他地方缓存。这是一种简单且有效的策略，在办公室中广泛使用，以最小化带宽消耗：设立一个办公室范围的缓存代理服务器。当一个用户请求一个来自互联网的资源并且该资源有 `Cache-Control: public` 标记时，公司缓存服务器将会存储该资源，以便其他用户在办公室网络中使用。

今天，由于廉价带宽的普及，这种做法可能不再那么流行，但由于历史往往会重演，你需要了解 `Cache-Control: public` 是如何以及为何起作用的。

Nginx 的 `expires` 指令出奇地表达力强。它可以接受多种不同的值。请参阅此表：

| `off` | 该值关闭 Nginx 缓存头部逻辑。不会添加任何内容，更重要的是，来自上游的现有头部将不会被修改。 |
| --- | --- |
| `epoch` | 这是一个人为的值，用于通过将 `Expires` 头设置为 **"1970 年 1 月 1 日 00:00:01 GMT"** 来从所有缓存中清除存储的资源。 |
| `max` | 这是“epoch”值的相反。**Expires** 头将设置为 **"2037 年 12 月 31 日 23:59:59 GMT"**，并且 **Cache-Control max-age** 设置为 10 年。这基本上意味着 HTTP 响应保证永远不会改变，因此客户端可以自由地不再请求相同的内容，并可以使用其自己存储的值。 |
| 特定持续时间 | 实际的特定持续时间值意味着从相关请求时间起的到期时间。例如，`expires 10w`。该指令的负值会发出一个特殊的头部 `Cache-Control: no-cache`。 |
| `"modified" 特定时间` | 如果在时间值之前添加关键字 "modified"，则过期时间将相对于提供文件的修改时间来计算。 |
| `"@" 具体时间` | 带有 @ 前缀的时间指定的是绝对的时间到期。这应该少于 24 小时。例如，Expires @17h;。 |

许多 Web 应用程序选择自行发出缓存头部，这是件好事。它们对哪些资源经常变化、哪些资源永不变化有更多的了解。篡改你从上游收到的头部可能是你希望做的事，也可能不是。有时，在代理响应时添加头部可能会产生冲突的头部集合，从而导致不可预测的行为。

你通过 Nginx 提供的静态文件应该设置 `expires` 指令。然而，关于上游的普遍建议是，始终检查你收到的缓存头部，并避免通过设置更激进的缓存策略来进行过度优化。

我们在本章前面描述的企业缓存代理配置，再加上对非公开资源错误的`public`缓存策略，可能会导致某些用户看到为其他用户生成的页面，这些页面通过相同的缓存代理缓存。使这种情况发生的方式出奇的简单。假设你的客户是一个书店，他们的 Web 应用程序既提供包含书籍详情、封面图片等的公共页面，也提供包含推荐页面和购物车等的私人资源。它们可能会有相同的 URL，且一旦由于错误被声明为`public`并设置了很远未来的过期时间，它们可能会被中间代理自由缓存。一些更智能的代理会自动注意到 cookies，并将其添加到缓存键中，或者避免缓存。但也存在一些不那么智能的代理，因此有不少报告显示它们会展示属于其他用户的页面。

甚至有技术方法，比如给所有 URL 添加随机数，通过让所有 URL 唯一来击败这样的缓存配置。

我们还想描述一种当前广泛使用的独特 URL 和长过期时间的组合。现代网站非常动态，既体现在文档加载后发生的变化，也体现在客户端代码的更新频率上。每天甚至每小时发布的情况并不罕见。这是 Web 作为应用程序交付机制的奢侈之处，人们正在抓住这一机会。那么，如何将快速发布与缓存结合起来呢？最初的想法是将版本编码到 URL 中。这种方法出奇地有效。每次发布后，所有的 URL 都会变化；旧的 URL 会开始在不同层级的缓存存储中慢慢过期，而新的 URL 会直接从源服务器请求。

基于这个方案，开发了一种聪明的技巧，它使用资源内容的哈希值而不是版本号作为 URL 的唯一元素。这样可以减少在新版本发布时，某些文件未更改而导致的额外缓存失效。

实现这个技巧是在应用层进行的。Nginx 管理员只需负责通过使用 `expires max` 指令等方式设置长时间的过期日期。

限制客户端缓存效果的一个显而易见的因素是，许多不同的用户可能会发出相同或类似的请求，这些请求都会到达 Web 服务器。避免重复劳动的下一步是服务器端的缓存。

## Nginx 上游模块中的缓存

缓存基础设施作为上游接口的一部分来实现，如果你允许我们使用面向对象编程的术语的话。每一个上游模块都有一组非常相似的指令，这些指令允许你配置该特定上游的响应本地缓存。

基本的工作原理非常简单——一旦一个请求被确定为上游请求，它就会被重定向到相关模块。如果该上游模块已配置缓存，首先会检查缓存中是否有此请求的响应。如果缓存中没有找到响应，才会执行实际的代理操作。之后，生成的新的响应会在发送给客户端时保存到缓存中。

有趣的是，虽然反向代理上的缓存已经被知道了一段时间，但 Nginx 却因其作为一个神奇的加速器而声名鹊起，而并没有实现这一点。原因应该从第一部分中就能看出——仅仅是 RAM 消耗的根本性变化，就带来了大量的性能提升。直到 0.7.44 版本的发布，Nginx 才内置了任何缓存功能。在那时，网站管理员要么使用著名的 squid HTTP 代理来进行缓存，要么使用 Apache 的 `mod_accel` 模块。顺便提一下，`mod_accel` 模块是由 Nginx 的作者 Igor Sysoev 创建的，并且最终成为了所有关于正确反向代理思想的试验平台，这些思想后来在 Nginx 中得到了实现。

让我们来看一下最流行的上游模块 `ngx_proxy` 的缓存指令。提醒一下，这个模块将请求转发到另一个 HTTP 服务器。这正是 Nginx 作为反向代理位于 Apache 前面时的工作方式。完整的描述可以在优秀的 Nginx 文档中找到，网址是 [`nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache`](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache)。我们不会重复文档内容，而是提供一些额外的事实和想法。

| 指令 | 附加信息 |
| --- | --- |
| `proxy_cache_path` | 该指令显然是整个缓存系列中的主要指令。它指定了缓存存储的路径参数，从文件系统中的路径开始。您应当熟悉所有选项。最重要的选项是`inactive`和`max_size`，它们控制 Nginx 缓存管理器如何从缓存存储中删除未使用的数据。此指令中的一个必需参数是`keys_zone`，它将缓存存储与“zone”关联。详情见后文。 |
| `proxy_cache` | 这是主开关指令。如果您希望启用缓存，它是必需的。它有一个稍显晦涩的参数，名为“zone”，稍后会详细解释。“off”值将禁用缓存。当在作用域栈上方有`proxy_cache`指令时，可能需要使用此指令。 |
| `proxy_cache_bypass` | 该指令允许您轻松指定一些响应永远不被缓存的条件。 |
| `proxy_cache_key` | 该指令创建一个用于识别缓存中对象的键。默认情况下，使用的是 URL，但人们通常会添加一些其他内容。不同的响应不应拥有相同的键。任何可能改变页面内容的内容都应该包含在键中。除了明显的 Cookie 值，您还可能希望在键中添加客户端 IP 地址，如果您的页面依赖于此（例如，通过 GeoIP 数据库进行地理定位）。 |
| `proxy_cache_lock` | 这是一个开/关的二进制开关，默认为关闭。如果开启它，那么对相同（这里的“相同”是指“具有相同缓存键”）资源的并发请求将不会并行执行。只有第一个请求会被执行，其他请求会被阻塞并等待。`proxy_cache_lock_*`系列指令在生成一些非常耗费资源的响应时可能会很有用。 |
| `proxy_cache_lock_age``proxy_cache_lock_timeout` | 这两个指令指定了额外的锁定参数。详情请参考文档。 |
| `proxy_cache_methods` | 这是一个可缓存的 HTTP 方法列表。除了明显的“GET”和“HEAD”方法外，您有时可能希望缓存一些不太常用的方法，如 WebDAV 中的“OPTIONS”或“PROPFIND”。在某些情况下，您可能希望缓存对“POST”，“PUT”和“DELETE”的响应，尽管那将是对规则的严重弯曲，您需要非常清楚自己在做什么。 |
| `proxy_cache_min_uses` | 这个数字参数的默认值为“1”，在优化庞大的缓存存储时可能会有用，它可以避免缓存稀有请求的响应。记住，真正有效的缓存不是存储更多内容的缓存，而是存储那些被反复请求的有用内容。 |
| `proxy_cache_purge` | 此指令指定了在对象过期之前从缓存存储中删除对象的额外条件。它可以作为强制使缓存条目无效的方式。一个好的缓存键设计应该不需要使其无效，但我们都知道在现实生活中，好的设计有时是多么罕见。 |
| `proxy_cache_revalidate` | 这也是一个布尔值指令。带有“If-None-Match”或“If-Modified-Since”头的 HTTP 条件请求即使没有返回任何新内容，也可以更新缓存中对象的有效性。如果需要这样做，请指定“on”。 |
| `proxy_cache_use_stale` | 这是一个有趣的指令，它有时允许用缓存中的过期响应来回应请求。这样做的主要情况是上游失败。有时，返回过期内容比基于著名的“内部服务器错误”拒绝请求要好。从用户的角度来看，这种情况非常常见。 |
| `proxy_cache_valid` | 这是一个非常粗略的缓存过期规范。通常，您应该通过响应头来控制缓存数据的有效性。但是，如果你需要一些快速或广泛的设置，这个指令会帮助你。 |

在所有上游模块中使用的一个非常重要的概念是缓存子系统中的缓存区域。一个区域是一个命名的内存区域，可以通过名称从所有 Nginx 进程中访问。熟悉 System V 共享内存或通过 mmap 映射区域进行 IPC 的读者将立刻看出其中的相似性。选择区域作为缓存状态存储的抽象层，因为它应该在所有工作进程之间共享。你可以在 Nginx 实例中配置多个缓存，但你始终需要为每个缓存指定一个区域。你可以将不同的缓存链接到同一个区域，关于缓存对象的信息将会被共享。区域还充当封装实际缓存存储配置的对象，配置内容包括缓存对象将在文件系统中的何处持久化，存储层次结构如何组织，何时清除过期对象，以及如何在重启时从磁盘加载对象到内存中。

总结一下，管理员首先使用指令`*_cache_path`设置至少一个包含所有相关存储参数的区域，然后通过指令`*_cache`将整个 URL 空间的子树连接到这些区域。

区域通常在全局范围内设置，通常在 `http` 范围内，而单个缓存则通过相关上下文中的简单 `*_cache` 指令链接到区域，例如，路径树下的定位或整个服务器块。

我们需要提醒你，所描述的缓存子系统指令家族适用于所有 Nginx 的上游模块。你将用 `proxy_` 替换其他上游标识符，从而得到另一个完全相同的指令家族，可能会对不同类型上游生成的响应有些许变化。例如，关于如何缓存 FastCGI 响应的详细信息可以参考[`nginx.org/en/docs/http/ngx_http_fastcgi_module.html#fastcgi_cache`](http://nginx.org/en/docs/http/ngx_http_fastcgi_module.html#fastcgi_cache)。

让我们提供一些现实世界的缓存配置示例，帮助你更好地理解这个概念：

```
http {
    proxy_cache_path /var/db/cache/nginx levels=1:2 keys_zone=cache1:1m max_size=1000m inactive=600m;
    proxy_temp_path /var/db/cache/tmp;

    server {
        listen 80;
        server_name example.com;

        location / {
            proxy_pass http://localhost:8080/;
            proxy_cache cache1;
            proxy_cache_valid 200 302 24h;
            proxy_cache_valid 404 5m;
        }
        }
}
```

这是一个规范的简单缓存配置，包含一个名为 `cache1` 的区域和在一个服务器中的 `/` 路径下配置的缓存。值得提到几个重要细节。使用 `proxy_temp_path` 指令配置的临时文件目录，强烈建议与主缓存存储在同一文件系统上，否则，Nginx 将无法快速地在临时存储和永久存储之间移动文件，而是会执行耗费资源的文件复制操作。

`key_zone` 大小指定了分配给该区域的内存量。此内存用于存储缓存中对象的键及元数据信息，而不是实际的缓存响应（对象）。对象存储的限制由 `max_size` 参数指定。Nginx 会启动一个名为 `cache manager` 的独立进程，该进程会不断扫描所有缓存区域，当 `max_size` 超过时，删除最少使用的对象。

`proxy_cache_valid` 指令组合指定了负面 404 结果的更短有效期。其背后的理念是，404 错误实际上可能被修复，至少其中一些可能由于配置错误而出现。因此，频繁地重试这些请求是有意义的。在决定有效期时，你还应考虑上游的负载。许多计算密集型的搜索算法需要更多资源来给出负面答案。为了确保查找的实体不存在，可能需要在所有地方进行检查，而不仅仅是在找到第一个实例后就停止。虽然这是对搜索算法的简化描述，但它足够简短，你应该记得在缩短缓存有效期之前，始终检查日志中负面响应的请求处理时间及其相对数量。

上述配置中遗漏了缓存的两个重要参数，这意味着你将使用默认值。`proxy_cache_methods` 默认只缓存 GET 和 HEAD 请求，这可能对你的 Web 应用程序并不最优。而 `proxy_cache_key` 默认值为 `$scheme$proxy_host$request_uri,`，如果你的 Web 应用程序对不同用户发出类似请求，这可能会很危险。了解这些指令，并为键添加唯一性，或者通过 `proxy_cache_bypass` 回退到未缓存行为。

我们想介绍的另一个示例要复杂得多。我们将专门为它提供一个单独的章节。

## 缓存静态文件

在水平扩展网站时，你不可避免地会发现自己处于一种情况，即有许多相同的 Nginx 服务器位于低层负载均衡器后面。它们都会将请求代理到相同的上游服务器集群，且在同步网站所服务的活跃、动态内容时不会出现问题。但如果你遵循将所有静态内容本地化以允许 Nginx 以最原生、最高效的方式提供服务的建议，你将面临一个任务，即在各处拥有相同文件的多个副本。

完成同一任务的另一种方法是使用一组 Nginx 实例来提供一个巨大的静态文件库，比如视频或音乐。因为文件库太大，所以在每个 Nginx 节点上都拥有一份副本是不可能的。

和往常一样，这两种情况有很多可能的解决方案。一个选择是拥有一个较小的 Nginx 服务器集群，向主集群提供文件，这样主集群将使用 `ngx_proxy` 上游进行缓存。

另一个有趣的解决方案是使用挂载在节点上的网络文件系统。传统的 Unix NFS 名声不好，但实际上，在当前的 Linux 内核上，它稳定到足以在生产环境中使用。两种替代方案是 AFS 和 SMBFS。挂载点下的文件对于 Nginx 看起来是本地文件，但它们仍然需要通过网络下载，这比读取一块好的本地 SSD 要慢得多。幸运的是，现代 Linux 内核具有从 NFS 和 AFS 本地缓存文件的能力。这种功能被称为 FS-Cache，并且使用一个独立的用户空间守护进程`cachefilesd`来存储来自网络文件系统的本地副本。你可以在[`people.redhat.com/dhowells/fscache/FS-Cache.pdf`](https://people.redhat.com/dhowells/fscache/FS-Cache.pdf)了解有关 FS-Cache 的更多信息。

FS-Cache 的配置相当简单，我们在这里不重点讨论。还有一种方法更紧密地遵循 Nginx 的理念。SlowFS 是一个第三方的类似上游模块，为 Nginx 提供了一个简单的文件系统子树接口。该接口包括缓存功能，这与其他所有 Nginx 上游相同。

**SlowFS** 是开源的，采用非常宽松的许可证，可以通过作者的网站或直接从 GitHub 仓库获取。参考 [`labs.frickle.com/nginx_ngx_slowfs_cache`](http://labs.frickle.com/nginx_ngx_slowfs_cache)。

这里是一个 SlowFS 配置的例子：

```
http {
    slowfs_cache_path /var/db/cache/nginx levels=1:2 keys_zone=cache2:20m;
  slowfs_temp_path /var/db/cache/tmp 1 2;
    location / {
        root /var/www/nfs;
        slowfs_cache cache2;
        slowfs_cache_key $uri;
        slowfs_cache_valid 5d;
    }
}
```

该配置在 `/var/www/nfs` 中可用的文件上安装了一个透明缓存层。文件的实际存储方式并不重要，它们仍然会根据 `slowfs_*` 系列指令指定的参数进行缓存。但显然，只有当 `/var/db/cache` 比 `/var/www/nfs` 快得多时，你才能注意到加速效果。

# 用内部重定向替代外部重定向

随着现代前端框架变得越来越复杂，所谓的客户端重定向的数量也在令人担忧地增加。Nginx 有一个很好的功能，可以帮助你节省一些流量，并减少客户端因重定向而等待的时间。首先，让我们简要回顾一下这些重定向。

所有的 HTTP 响应都是由三部分主要内容组成的文档：

+   这里有一些 HTTP 状态码（200: 成功，404: 未找到，等等）

+   这里有一些松散结构的键值对，形式为头部信息

+   这里有一个相对较大的、不透明的可选主体

互联网上有很多关于 HTTP 响应代码的文档（还有一些搞笑的内容，可以在[`httpstatusdogs.com/`](http://httpstatusdogs.com/)找到）——与我们讨论相关的代码在四百以内，即 300 到 399 之间。

带有这些状态码的响应表示浏览器应该立即发起另一个请求，而不是原始请求。这就是为什么它们被称为重定向。不同 3xx 状态码之间的语义差异在这里并不重要。

重要的是，许多重定向是多余的。HTTP 客户端（例如浏览器）在重定向上花费了时间，而这些重定向并没有特别的意义，除了清理地址栏中的 URL。雅虎真的需要将我从 `yahoo.de` 重定向到 `ru.yahoo.com`、`www.yahoo.com`，以及 [`www.yahoo.com`](https://www.yahoo.com) 吗？这让我浏览器发出三次额外的请求，完全可以避免。如果你控制的网站有这种行为，你可以向相应的开发人员提出这个问题。你也可以建议一个简单的修复方法，稍后在文中会提到。

有一个很棒的小型网络服务，可以让你查看重定向链条以及一些可能对调试有用的其他元信息。你可以在[`httpstatus.io/`](https://httpstatus.io/)查看它。

你可以去检查一下你的网站是否存在不必要的重定向，这可能会使得移动用户在真正加载网站内容之前浪费宝贵的几秒钟。

![用内部重定向替代外部重定向](img/B04329_04_02.jpg)

Nginx 有一个名为“内部重定向”的功能。其思路是所有中间的 HTTP 请求-响应对都在服务器内部处理。客户端从链条的末端获得内容，以响应原始请求。有多种方法可以在 Nginx 中启用内部重定向，但最灵活的方式可能是由 Nginx 后的上游生成的 `X-Accel-Redirect` 响应头。

为了让内部重定向能够使用这种方法工作，你需要更改上游软件的配置。你需要生成早前提到的 `X-Accel-Redirect:` 头，而不是通过 HTTP 3xx 响应码和 `Location:` 响应头生成真正的重定向。这实际上是你需要做的唯一修改。这里有一些需要小心的地方，全部与浏览器的安全模型有关。像雅虎示例中所展示的地理重定向在现在已经相当少见，因此优化它们可能不值得，因为你在错误的域上发布 cookies 会引发麻烦。然而，`example.com` 到 `www.example.com` 的重定向仍然非常流行，且看起来是内部重定向的完美候选。

# 总结

在本章中，我们讨论了几种方法来查找你 Nginx 安装中的性能问题。我们主要集中在处理你可能继承并正在优化的遗留网站。之所以这样做，是因为 Nginx 本身很少会有性能不足的问题。

作为一名运营专家，你通过学习如何加速现有的、已经有负载和客户的工作网站，提升了自己在业务中的价值，但这些网站基于一些早期的 Nginx 技术，成为了限制因素。
