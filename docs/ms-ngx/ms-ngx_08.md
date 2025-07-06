# 第八章 排错技巧

我们生活在一个不完美的世界里。尽管我们尽力而为并做好了计划，但有时事情并不会按预期的方式发展。我们需要能够退一步，看看哪里出了问题。当我们无法立即看到是什么导致了错误时，我们需要能够利用一套工具箱中的技巧来帮助我们发现问题。这个找出问题并解决问题的过程就是我们所说的排错。

本章中，我们将探索用于排错 NGINX 的不同技巧：

+   分析日志文件

+   配置高级日志记录

+   常见配置错误

+   操作系统限制

+   性能问题

+   使用 Stub Status 模块

# 分析日志文件

在开始长时间的调试会话以追踪问题的根源之前，通常首先查看日志文件是很有帮助的。它们通常会提供我们需要的线索，帮助我们追踪错误并加以修正。然而，`error_log`中出现的消息有时可能会有点难以理解，因此我们将讨论日志条目的格式，然后通过一些示例帮助你理解它们的含义。

## 错误日志文件格式

NGINX 使用几种不同的日志记录函数来生成`error_log`条目。这些函数使用的格式呈现以下模式：

*<timestamp> [log-level] <master/worker pid>#0: 消息*

例如：

```
2012/10/14 18:56:41  [notice] 2761#0: using inherited sockets from "6;" 

```

这是一个信息性消息的示例（日志级别`notice`）。在这种情况下，一个`nginx`二进制文件替换了之前运行的二进制文件，并成功继承了旧二进制文件的套接字。

错误级别的日志记录器会生成如下消息：

```
2012/10/14 18:50:34 [error] 2632#0: *1 open() "/opt/nginx/html/blog" failed (2: No such file or directory), client: 127.0.0.1, server: www.example.com, request: "GET /blog HTTP/1.0", host: "www.example.com"

```

根据错误的不同，你可能会看到来自操作系统的消息（如本例所示），或者仅来自 NGINX 本身。在这种情况下，我们看到以下组件：

+   时间戳（`2012/10/14 18:50:34`）

+   日志级别（`error`）

+   工作进程 pid（`2632`）

+   连接编号（`1`）

+   系统调用（`open`）

+   系统调用的参数（`/opt/nginx/html/blog`）

+   系统调用产生的错误消息（`2: No such file or directory`）

+   哪个客户端发起了导致错误的请求（`127.0.0.1`）

+   哪个服务器上下文负责处理请求（`www.example.com`）

+   请求本身（`GET /blog HTTP/1.0`）

+   请求中发送的`Host`头（`www.example.com`）

下面是一个关键级别日志条目的示例：

```
2012/10/14 19:11:50 [crit] 3142#0: the changing binary signal is ignored: you should shutdown or terminate before either old or new binary's process

```

关键级别消息意味着 NGINX 无法执行请求的操作。如果它尚未运行，则意味着 NGINX 将无法启动。

下面是一个紧急消息的示例：

```
2012/10/14 19:12:05 [emerg] 3195#0: bind() to 0.0.0.0:80 failed (98: Address already in use)

```

紧急消息也意味着 NGINX 无法执行请求的操作。这也意味着 NGINX 无法启动，或者如果在被要求读取配置时已经在运行，它也不会执行请求的更改。

### 注意

如果你想知道为什么配置更改没有生效，请检查错误日志。NGINX 很可能在配置中遇到了错误，并且没有应用更改。

## 错误日志文件条目示例

以下是一些在实际日志文件中发现的错误信息示例。每个示例后面跟着一个简短的解释，说明可能的含义。请注意，由于 NGINX 在新版中的改进，您看到的文本可能与示例中的有所不同。

查看以下日志文件条目示例：

```
2012/11/29 21:31:34 [error] 6338#0: *1 upstream prematurely closed connection while reading response header from upstream, client: 127.0.0.1, server: , request: "GET / HTTP/1.1", upstream: "fastcgi://127.0.0.1:8080", host: "www.example.com"

```

这里我们看到一条消息，它可以有几种解释。这可能意味着我们正在与之通信的服务器在实现上有错误，没有正确地使用 FastCGI 协议。也可能意味着我们错误地将流量指向了 HTTP 服务器，而不是 FastCGI 服务器。如果是这种情况，简单的配置更改（使用 `proxy_pass` 替代 `fastcgi_pass`，或使用正确的 FastCGI 服务器地址）可能会解决问题。

这类消息也可能仅仅意味着上游服务器生成响应的时间太长。造成这种情况的原因可能有很多，但就 NGINX 而言，解决方案相当简单：增加超时时间。根据哪个模块负责建立连接，可能需要将 `proxy_read_timeout` 或 `fastcgi_read_timeout`（或其他 `*_read_timeout`）指令的默认值 `60s` 增加。

查看以下日志文件条目示例：

```
2012/11/29 06:31:42 [error] 2589#0: *6437 client intended to send too large body: 13106010 bytes, client: 127.0.0.1, server: , request: "POST /upload_file.php HTTP/1.1", host: "www.example.com", referrer: "http://www.example.com/file_upload.html"

```

这个例子比较直白。NGINX 报告文件无法上传，因为文件太大。要解决这个问题，增加 `client_body_size` 的值。请记住，由于编码原因，上传的文件大小将比实际文件大小大约 30%。例如，如果你希望允许用户上传最大为 12 MB 的文件，则应将此指令设置为 `16m`。

查看以下日志文件条目示例：

```
2012/10/14 19:51:22 [emerg] 3969#0: "proxy_pass" cannot have URI part in location given by regular expression, or inside named location, or inside "if" statement, or inside "limit_except" block in /opt/nginx/conf/nginx.conf:16

```

在这个示例中，我们看到 NGINX 无法启动，原因是配置错误。错误信息清楚地说明了为什么 NGINX 无法启动。我们看到在 `proxy_pass` 指令的参数中有一个 URI，应该没有这个 URI。NGINX 甚至告诉我们错误发生在哪个文件（这里是 `/opt/nginx/conf/nginx.conf`）的第几行（这里是 `16`）。

```
2012/10/14 18:46:26 [emerg] 2584#0: mkdir() "/home/www/tmp/proxy_temp" failed (2: No such file or directory)

```

这是一个 NGINX 无法启动的例子，因为它无法执行所要求的操作。`proxy_temp_path` 指令指定了 NGINX 在代理时存储临时文件的位置。如果 NGINX 无法创建这个目录，它就无法启动，因此请确保该目录的路径存在。

查看以下日志文件条目示例：

```
2012/10/14 18:46:54 [emerg] 2593#0: unknown directive "client_body_temp_path" in /opt/nginx/conf/nginx.conf:6

```

在前面的代码中，我们看到了一条可能令人困惑的消息。我们知道 `client_body_temp_path` 是一个有效的指令，但 NGINX 不接受它并给出了 `unknown directive` 的提示。当我们思考 NGINX 如何处理其配置文件时，我们意识到这实际上是有道理的。NGINX 是以模块化的方式构建的，每个模块负责处理自己的配置上下文。因此，我们得出结论，这个指令出现在了解析该指令的模块上下文之外的部分。

```
2012/10/16 20:56:31 [emerg] 3039#0: "try_files" directive is not allowed here in /opt/nginx/conf/nginx.conf:16

```

有时，NGINX 会给我们一些提示，帮助我们找出问题所在。在前面的例子中，NGINX 已经理解了 `try_files` 指令，但告诉我们它被用在了错误的位置。它非常方便地给出了配置文件中出错的位置，帮助我们更容易地找到问题。

```
2012/10/16 20:56:42 [emerg] 3043#0: host not found in upstream "tickets.example.com" in /opt/nginx/conf/nginx.conf:22

```

这条紧急级别的消息告诉我们，如果配置中使用了主机名，NGINX 对 DNS 的依赖有多大。如果 NGINX 无法解析 `upstream`、`proxy_pass`、`fastcgi_pass` 或其他 `*_pass` 指令中使用的主机名，那么它将无法启动。这会影响 NGINX 在系统重启后的启动顺序。确保在 NGINX 启动时，名称解析能正常工作。

```
2012/10/29 18:59:26 [emerg] 2287#0: unexpected "}" in /opt/nginx/conf/nginx.conf:40

```

这种类型的消息通常表示配置错误，NGINX 无法关闭上下文。在给定行之前的某个地方出现了问题，导致 NGINX 无法通过 `{` 和 `}` 字符形成完整的上下文。通常这意味着前一行缺少分号，导致 NGINX 将 `}` 字符错误地读取为未完成行的一部分。

```
2012/10/28 21:38:34 [emerg] 2318#0: unexpected end of file, expecting "}" in /opt/nginx/conf/nginx.conf:21

```

与之前的错误相关，这个错误意味着 NGINX 在找到匹配的闭合括号之前就已经到达了配置文件的末尾。这种错误发生在存在不匹配的 `{` 和 `}` 字符时。使用能匹配括号对的文本编辑器有助于精确定位缺失的括号。根据缺失的括号插入的位置，配置文件可能会导致与原本意图完全不同的效果。

```
2012/10/29 18:50:11 [emerg] 2116#0: unknown "exclusion" variable

```

在这里，我们看到一个没有先声明变量就使用它的例子。这意味着 `$exclusion` 在配置文件中出现在 `set`、`map` 或 `geo` 指令定义其值之前。这类错误也可能是打字错误的表现。我们可能已经定义了 `$exclusions` 变量，但后来错误地将它写作了 `$exclusion`。

```
2012/11/29 21:26:51 [error] 3446#0: *2849 SSL3_GET_FINISHED:digest check failed

```

这意味着你需要禁用 SSL 会话重用。你可以通过将 `proxy_ssl_session_reuse` 指令设置为 `off` 来实现。

# 配置高级日志记录

在正常情况下，我们希望日志记录尽可能简洁。通常，重要的是哪些 URI 被哪些客户端调用了，以及何时调用，如果出现错误，则显示相应的错误信息。如果我们希望查看更多信息，则需要配置调试日志。

## 调试日志

要启用调试日志记录，`nginx` 二进制文件必须使用 `--with-debug` 配置标志进行编译。由于该标志不推荐用于高性能的生产系统，因此我们可能希望为我们的需求提供两个独立的 `nginx` 二进制文件：一个用于生产环境，另一个则具备所有相同的配置选项，并额外添加 `--with-debug`，这样我们就可以在运行时简单地交换二进制文件来进行调试。

### 在运行时切换二进制文件

NGINX 提供了在运行时切换二进制文件的功能。在将 `nginx` 二进制文件替换为不同的版本后，无论是因为升级，还是因为我们想加载一个不同模块的 NGINX，我们可以开始替换正在运行的 `nginx` 二进制文件的过程：

1.  向正在运行的 NGINX 主进程发送 USR2 信号，指示它启动一个新的主进程。它将把其 PID 文件重命名为 `.oldbin`（例如，`/var/run/nginx.pid.oldbin`）：

    ```
    # kill -USR2 `cat /var/run/nginx.pid`

    ```

    现在将有两个 NGINX 主进程在运行，每个进程都有一组工作进程来处理传入的请求：

    ```
    root 1149 0.0 0.2 20900 11768 ?? Is Fri03PM 0:00.13 nginx: master process /usr/local/sbin/nginx
    www 36660 0.0 0.2 20900 11992 ?? S 12:52PM 0:00.19 nginx: worker process (nginx)
    www 36661 0.0 0.2 20900 11992 ?? S 12:52PM 0:00.19 nginx: worker process (nginx)
    www 36662 0.0 0.2 20900 12032 ?? I 12:52PM 0:00.01 nginx: worker process (nginx)
    www 36663 0.0 0.2 20900 11992 ?? S 12:52PM 0:00.18 nginx: worker process (nginx)
    root 50725 0.0 0.1 18844 8408 ?? I 3:49PM 0:00.05 nginx: master process /usr/local/sbin/nginx
    www 50726 0.0 0.1 18844 9240 ?? I 3:49PM 0:00.00 nginx: worker process (nginx)
    www 50727 0.0 0.1 18844 9240 ?? S 3:49PM 0:00.01 nginx: worker process (nginx)
    www 50728 0.0 0.1 18844 9240 ?? S 3:49PM 0:00.01 nginx: worker process (nginx)
    www 50729 0.0 0.1 18844 9240 ?? S 3:49PM 0:00.01 nginx: worker process (nginx)

    ```

1.  向旧的 NGINX 主进程发送 WINCH 信号，告知它停止处理新的请求，并在其完成当前请求后逐步结束工作进程：

    ```
    # kill -WINCH `cat /var/run/nginx.pid.oldbin`

    ```

    您将看到以下响应输出：

    ```
    root 1149 0.0 0.2 20900 11768 ?? Ss Fri03PM 0:00.14 nginx: master process /usr/local/sbin/nginx
    root 50725 0.0 0.1 18844 8408 ?? I 3:49PM 0:00.05 nginx: master process /usr/local/sbin/nginx
    www 50726 0.0 0.1 18844 9240 ?? I 3:49PM 0:00.00 nginx: worker process (nginx)
    www 50727 0.0 0.1 18844 9240 ?? S 3:49PM 0:00.01 nginx: worker process (nginx)
    www 50728 0.0 0.1 18844 9240 ?? S 3:49PM 0:00.01 nginx: worker process (nginx)
    www 50729 0.0 0.1 18844 9240 ?? S 3:49PM 0:00.01 nginx: worker process (nginx)

    ```

1.  向旧的 NGINX 主进程发送 `QUIT` 信号，等到所有的工作进程结束后，我们将只有新的 `nginx` 二进制文件在运行，并且响应请求：

    ```
    # kill -QUIT `cat /var/run/nginx.pid.oldbin`

    ```

如果新的二进制文件有任何问题，在向旧的二进制文件发送 `QUIT` 信号之前，我们可以回滚到旧版本：

```
# kill -HUP `cat /var/run/nginx.pid.oldbin`
# kill -QUIT `cat /var/run/nginx.pid`

```

如果新的二进制文件仍有主进程在运行，您可以向它发送 `TERM` 信号，强制它退出：

```
# kill -TERM `cat /var/run/nginx.pid`

```

同样，任何仍在运行的新的工作进程可以先通过 `KILL` 信号停止。

### 注意

请注意，某些操作系统在升级 `nginx` 包时，会自动为您执行二进制文件升级过程。

一旦我们运行了启用调试的 `nginx` 二进制文件，我们可以配置调试日志记录：

```
user www;

events {

    worker_connections 1024;

}

error_log logs/debug.log debug;

http {

   …

}
```

我们已将 `error_log` 指令放置在 NGINX 配置的主上下文中，这样它将在每个子上下文中有效，除非在其中被覆盖。我们可以有多个 `error_log` 指令，每个指向不同的文件，并且使用不同的日志级别。除了 `debug`，`error_log` 还可以使用以下值：

+   `debug_core`

+   `debug_alloc`

+   `debug_mutex`

+   `debug_event`

+   `debug_http`

+   `debug_imap`

每个级别都是为了调试 NGINX 内部的特定模块。

同样，为每个虚拟服务器配置单独的错误日志也是有意义的。这样，仅与该服务器相关的错误将被记录在特定的日志中。这个概念也可以扩展到 `core` 和 `http` 模块：

```
error_log logs/core_error.log;

events {

    worker_connections 1024;

}

http {

    error_log logs/http_error.log;

    server {

        server_name www.example.com;

        error_log logs/www.example.com_error.log;

    }

    server {

        server_name www.example.org;

        error_log logs/www.example.org_error.log;

    }

}
```

使用这种模式，我们能够调试特定的虚拟主机，如果这是我们感兴趣的区域：

```
    server {

        server_name www.example.org;

        error_log logs/www.example.org_debug.log debug_http;

    }
```

以下是单个请求的 `debug_http` 级别输出示例。每个点上发生的事情在中间穿插了一些注释：

```
<timestamp> [debug] <worker pid>#0: *<connection number> http cl:-1 max:1048576
```

`rewrite` 模块在请求处理阶段非常早期就被激活：

```
<timestamp> [debug] <worker pid>#0: *<connection number> rewrite phase: 3
<timestamp> [debug] <worker pid>#0: *<connection number> post rewrite phase: 4
<timestamp> [debug] <worker pid>#0: *<connection number> generic phase: 5
<timestamp> [debug] <worker pid>#0: *<connection number> generic phase: 6
<timestamp> [debug] <worker pid>#0: *<connection number> generic phase: 7

```

检查访问限制：

```
<timestamp> [debug] <worker pid>#0: *<connection number> access phase: 8
<timestamp> [debug] <worker pid>#0: *<connection number> access: 0100007F FFFFFFFF 0100007F

```

接下来解析 `try_files` 指令。文件的路径是通过任何字符串（`http script copy`）加上任何变量（`http script var`）的值来构建的，这些变量在 `try_files` 指令的参数中：

```
<timestamp> [debug] <worker pid>#0: *<connection number> try files phase: 11
<timestamp> [debug] <worker pid>#0: *<connection number> http script copy: "/"
<timestamp> [debug] <worker pid>#0: *<connection number> http script var: "ImageFile.jpg"

```

然后，评估后的参数与该 `location` 的 `alias` 或 `root` 连接，找到文件的完整路径：

```
<timestamp> [debug] <worker pid>#0: *<connection number> trying to use file: "/ImageFile.jpg" "/data/images/ImageFile.jpg"
<timestamp> [debug] <worker pid>#0: *<connection number> try file uri: "/ImageFile.jpg"

```

一旦文件被找到，它的内容将被处理：

```
<timestamp> [debug] <worker pid>#0: *<connection number> content phase: 12
<timestamp> [debug] <worker pid>#0: *<connection number> content phase: 13
<timestamp> [debug] <worker pid>#0: *<connection number> content phase: 14
<timestamp> [debug] <worker pid>#0: *<connection number> content phase: 15
<timestamp> [debug] <worker pid>#0: *<connection number> content phase: 16

```

`http filename` 是要发送的文件的完整路径：

```
<timestamp> [debug] <worker pid>#0: *<connection number> http filename: "/data/images/ImageFile.jpg"

```

`static` 模块接收到此文件的文件描述符：

```
<timestamp> [debug] <worker pid>#0: *<connection number> http static fd: 15

```

响应体中的任何临时内容不再需要：

```
<timestamp> [debug] <worker pid>#0: *<connection number> http set discard body

```

一旦知道了文件的所有信息，NGINX 就可以构建完整的响应头：

```
<timestamp> [debug] <worker pid>#0: *<connection number> HTTP/1.1 200 OK
Server: nginx/<version>
Date: <Date header>
Content-Type: <MIME type>
Content-Length: <filesize>
Last-Modified: <Last-Modified header>
Connection: keep-alive
Accept-Ranges: bytes

```

接下来的阶段涉及对文件进行任何变换，这可能是由于输出过滤器的作用：

```
<timestamp> [debug] <worker pid>#0: *<connection number> http write filter: l:0 f:0 s:219
<timestamp> [debug] <worker pid>#0: *<connection number> http output filter "/ImageFile.jpg?file=ImageFile.jpg"
<timestamp> [debug] <worker pid>#0: *<connection number> http copy filter: "/ImageFile.jpg?file=ImageFile.jpg"
<timestamp> [debug] <worker pid>#0: *<connection number> http postpone filter "/ImageFile.jpg?file=ImageFile.jpg" 00007FFF30383040
<timestamp> [debug] <worker pid>#0: *<connection number> http write filter: l:1 f:0 s:480317
<timestamp> [debug] <worker pid>#0: *<connection number> http write filter limit 0
<timestamp> [debug] <worker pid>#0: *<connection number> http write filter 0000000001911050
<timestamp> [debug] <worker pid>#0: *<connection number> http copy filter: -2 "/ImageFile.jpg?file=ImageFile.jpg"
<timestamp> [debug] <worker pid>#0: *<connection number> http finalize request: -2, "/ImageFile.jpg?file=ImageFile.jpg" a:1, c:1
<timestamp> [debug] <worker pid>#0: *<connection number> http run request: "/ImageFile.jpg?file=ImageFile.jpg"
<timestamp> [debug] <worker pid>#0: *<connection number> http writer handler: "/ImageFile.jpg?file=ImageFile.jpg"
<timestamp> [debug] <worker pid>#0: *<connection number> http output filter "/ImageFile.jpg?file=ImageFile.jpg"
<timestamp> [debug] <worker pid>#0: *<connection number> http copy filter: "/ImageFile.jpg?file=ImageFile.jpg"
<timestamp> [debug] <worker pid>#0: *<connection number> http postpone filter "/ImageFile.jpg?file=ImageFile.jpg" 0000000000000000
<timestamp> [debug] <worker pid>#0: *<connection number> http write filter: l:1 f:0 s:234338
<timestamp> [debug] <worker pid>#0: *<connection number> http write filter limit 0
<timestamp> [debug] <worker pid>#0: *<connection number> http write filter 0000000000000000
<timestamp> [debug] <worker pid>#0: *<connection number> http copy filter: 0 "/ImageFile.jpg?file=ImageFile.jpg"
<timestamp> [debug] <worker pid>#0: *<connection number> http writer output filter: 0, "/ImageFile.jpg?file=ImageFile.jpg"
<timestamp> [debug] <worker pid>#0: *<connection number> http writer done: "/ImageFile.jpg?file=ImageFile.jpg"

```

一旦输出过滤器运行完毕，请求就完成了：

```
<timestamp> [debug] <worker pid>#0: *<connection number> http finalize request: 0, "/ImageFile.jpg?file=ImageFile.jpg" a:1, c:1

```

`keepalive` 处理器负责判断连接是否应该保持打开：

```
<timestamp> [debug] <worker pid>#0: *<connection number> set http keepalive handler
<timestamp> [debug] <worker pid>#0: *<connection number> http close request

```

在请求处理完毕后，它可以被记录：

```
<timestamp> [debug] <worker pid>#0: *<connection number> http log handler
<timestamp> [debug] <worker pid>#0: *<connection number> hc free: 0000000000000000 0
<timestamp> [debug] <worker pid>#0: *<connection number> hc busy: 0000000000000000 0
<timestamp> [debug] <worker pid>#0: *<connection number> tcp_nodelay

```

客户端已关闭连接，因此 NGINX 也将关闭连接：

```
<timestamp> [debug] <worker pid>#0: *<connection number> http keepalive handler
<timestamp> [info] <worker pid>#0: *<connection number> client <IP address> closed keepalive connection
<timestamp> [debug] <worker pid>#0: *<connection number> close http connection: 3

```

如你所见，这里包含了相当多的信息。如果你在弄清楚为什么某个配置不起作用时遇到困难，查看调试日志的输出可能会有所帮助。你可以立即看到各种过滤器运行的顺序，以及哪些处理器参与了请求的服务。

## 使用访问日志进行调试

当我在学习编程时，无法找到问题的源头时，我的一个朋友告诉我“到处加上 printf”。这就是他最迅速找到问题源头的方法。他的意思是，在每个代码分支点放一个会打印消息的语句，这样我们可以看到哪个代码路径被执行，逻辑在哪里出错。通过这样做，我们能够可视化发生了什么，并更容易找到问题所在。

同样的原理也可以应用于配置 NGINX。我们可以用 `log_format` 和 `access_log` 指令来代替 `printf()`，以可视化请求流并分析请求处理过程中的情况。使用 `log_format` 指令可以查看配置中不同点的变量值：

```
http {

    log_format sentlog '[$time_local] "$request" $status $body_bytes_sent ';
    log_format imagelog '[$time_local] $image_file $image_type '
                    '$body_bytes_sent $status';

    log_format authlog '[$time_local] $remote_addr $remote_user '
                    '"$request" $status';

}
```

使用多个 `access_logs` 来查看哪些位置在什么时间被调用。通过为每个位置配置不同的 `access_log`，我们可以轻松看到哪些位置未被使用。对这些位置的任何更改都不会影响请求处理；应首先检查处理层次结构中较高的那些位置。

```
http {

    log_format sentlog '[$time_local] "$request" $status $body_bytes_sent ';

    log_format imagelog '[$time_local] $image_file $image_type '
                    '$body_bytes_sent $status';

    log_format authlog '[$time_local] $remote_addr $remote_user '
                    '"$request" $status';

    server {

        server_name .example.com;

        root /home/www;

        location / {

            access_log logs/example.com-access.log combined;

            access_log logs/example.com-root_access.log sentlog;

            rewrite ^/(.*)\.(png|jpg|gif)$ /images/$1.$2;

            set $image_file $1;

            set $image_type $2;
        }

        location /images {

            access_log logs/example.com-images_access.log imagelog;

        }

        location /auth {

            auth_basic "authorized area";

            auth_basic_user_file conf/htpasswd;

            deny all;

            access_log logs/example.com-auth_access.log authlog;

        }

    }

}
```

在前面的示例中，每个位置都有一个`access_log`声明，并且每个`access_log`声明都有一个不同的`log_format`。我们可以根据相应`access_log`中的条目确定哪些请求到达了各个位置。例如，如果在`example.com-images_access.log`文件中没有条目，那么我们就知道没有请求到达`/images`位置。我们可以比较不同日志文件的内容，看看变量是否被设置为正确的值。例如，如果`$image_file`和`$image_type`变量为空，`imagelog`格式的`access_log`中相应的占位符也会为空。

# 常见的配置错误

故障排除的下一步是检查配置，看看它是否确实达到了你想要实现的目标。NGINX 配置已经在互联网上流传了多年。通常，这些配置是为较旧版本的 NGINX 设计的，用于解决特定问题。不幸的是，这些配置常常是被复制而没有真正理解它们最初设计的目的。有时，可以使用*更新*的配置来更好地解决同样的问题。

## 使用`if`代替`try_files`

其中一个例子是用户希望在文件系统中找到静态文件时提供该文件，如果未找到，则将请求传递给 FastCGI 服务器：

```
server {

    root /var/www/html;

    location / {

        if (!-f $request_filename) {

            include fastcgi_params;

            fastcgi_pass 127.0.0.1:9000;

            break;

        }

    }

}
```

这是在 NGINX 没有`try_files`指令（该指令出现在版本 0.7.27）之前，通常解决此问题的方式。之所以认为这是一个配置错误，是因为它涉及在`location`指令中使用`if`。正如在第四章的*将`if`-fy 配置转换为更现代的解释*部分中详细说明的那样，*NGINX 作为反向代理*，这可能导致意外结果，甚至可能引发崩溃。正确解决此问题的方法如下：

```
server {

    root /var/www/html;

    location / {

        try_files $uri $uri/ @fastcgi;

    }

    location @fastcgi {
        include fastcgi_params;

        fastcgi_pass 127.0.0.1:9000;

    }

}
```

`try_files`指令用于检查文件是否存在于文件系统中，如果没有找到，则将请求传递给 FastCGI 服务器，而无需使用`if`。

## 将`if`用作主机名切换

有许多配置示例，使用`if`根据 HTTP `Host`头部重定向请求。这些类型的配置充当选择器，并会为每个请求进行评估：

```
server {

    server_name .example.com;

    root /var/www/html;

    if ($host ~* ^example\.com) {

        rewrite ^/(.*)$ http://www.example.com/$1 redirect;

    }

}
```

与其为每个请求评估`if`而承担处理开销，NGINX 的常规请求匹配机制可以将请求路由到正确的虚拟服务器。然后，可以将重定向放在适当的位置，甚至无需重写：

```
server {

    server_name example.com;

    return 301 $scheme://www.example.com;

}
server {

    server_name www.example.com;

    root /var/www/html;

    location / {

        …

    }

}
```

## 未充分利用服务器上下文

复制的配置片段经常导致不正确配置的另一个地方是`server`上下文区域。`server`上下文描述了整个虚拟服务器（应该在特定的`server_name`下处理的所有内容）。在这些复制的配置片段中，它的使用不够充分。

通常，我们会看到`root`和`index`在每个`location`中指定：

```
server {

    server_name www.example.com;

    location / {

        root /var/www/html;

        index index.php index.html index.htm;

    }

    location /ftp{

        root /var/www/html;

        index index.php index.html index.htm;

    }

}
```

当添加新位置时，这可能会导致配置错误，如果没有将指令复制到这些新位置，或者复制不正确。使用`root`和`index`指令的目的是分别指示虚拟服务器的文档根目录和在 URI 中给定目录时应该尝试的文件。这些值随后会被继承到该`server`上下文中的任何`location`。

```
server {

    server_name www.example.com;

    root /var/www/html;

    index  index.php index.html index.htm;

    location / {

        ...

    }

    location /ftp{

        ...

    }

}
```

在这里，我们已指定所有文件将位于`/var/www/html`下，并且`index.php index.html index.htm`将按顺序作为任何位置的`index`文件进行尝试。

# 操作系统限制

操作系统通常是我们寻找问题的最后一个地方。我们假设设置系统的人已经根据我们的工作负载调整了操作系统，并在类似场景下进行了测试。但实际上，这并非总是如此。有时我们需要深入操作系统本身，以识别瓶颈。

与 NGINX 一样，我们可以首先从两个主要领域查找性能问题：**文件描述符限制**和**网络限制**。

## 文件描述符限制

NGINX 以多种方式使用文件描述符。主要用途是响应客户端连接，每个连接都使用一个文件描述符。每个外部连接（尤其是在代理配置中）都需要一个唯一的 IP:TCP 端口对，NGINX 通过文件描述符来引用它。若 NGINX 正在提供任何静态文件或其缓存中的响应，也会使用文件描述符。正如您所看到的，随着并发用户数量的增加，文件描述符的数量会迅速增加。NGINX 可以使用的文件描述符总数受到操作系统的限制。

典型的类 UNIX 操作系统为超级用户（`root`）与普通用户设定了不同的限制，因此请确保以运行 NGINX 的非特权用户身份执行以下命令（该用户由`--user`编译时选项或`user`配置指令指定）。

```
ulimit -n

```

此命令将显示该用户允许的打开文件描述符的数量。通常，这个数字保守地设置为 1024 甚至更低。由于我们知道 NGINX 将是机器上文件描述符的主要使用者，我们可以将此数字设置得更高。如何设置取决于具体的操作系统。可以按照以下方式进行：

+   Linux

    ```
    vi /etc/security/limits.conf

    www-run  hard  nofile  65535
    $ ulimit -n 65535

    ```

+   FreeBSD

    ```
    vi /etc/sysctl.conf

    kern.maxfiles=65535
    kern.maxfilesperproc=65535
    kern.maxvnodes=65535
    # /etc/rc.d/sysctl reload

    ```

+   Solaris

    ```
    # projadd -c "increased file descriptors"  -K "process.max-file-descriptor=(basic,65535,deny)" resource.file

    # usermod -K project=resource.file www

    ```

上述两个命令将增加以`www`用户身份运行的新进程允许的最大文件描述符数量。此设置在重启后也会保持有效。

以下两个命令将增加运行中的 NGINX 进程允许的最大文件描述符数：

```
# prctl -r -t privileged -n process.max-file-descriptor -v 65535 -i process `pgrep nginx`

# prctl -x -t basic -n process.max-file-descriptor -i process `pgrep nginx`

```

这些方法中的每一种都会改变操作系统的限制，但对正在运行的 NGINX 进程没有影响。为了让 NGINX 使用指定数量的文件描述符，可以将 `worker_rlimit_nofile` 指令设置为这个新限制：

```
worker_rlimit_nofile  65535;

worker_processes    8;

events {

    worker_connections	8192;

}
```

现在，向正在运行的 `nginx` 主进程发送 `HUP` 信号：

```
# kill -HUP `cat /var/run/nginx.pid`

```

NGINX 将能够处理超过 65,000 个并发客户端、连接到上游服务器以及任何本地静态或缓存文件。如果您实际拥有 8 个 CPU 核心或面临严重的 I/O 限制，那么使用如此多的 `worker_processes` 是有意义的。如果不是这种情况，则应减少 `worker_processes` 的数量，以匹配 CPU 核心数，并增加 `worker_connections`，使得两者的乘积接近 65,000。

当然，您可以将总文件描述符数量和 `worker_connections` 增加到适合硬件和使用场景的合理限制。只要操作系统限制和配置正确设置，NGINX 完全有能力处理数百万个并发连接。

## 网络限制

如果您发现自己处于没有可用网络缓冲区的情况，您很可能只能通过控制台登录（如果能登录的话）。这种情况可能发生在 NGINX 接收到大量客户端连接，以至于所有可用的网络缓冲区都被用尽。增加网络缓冲区的数量也是特定于操作系统的，通常可以通过以下方式进行：

+   FreeBSD

    ```
    vi /boot/loader.conf

    kern.ipc.nmbclusters=262144

    ```

+   Solaris

    ```
    # ndd -set /dev/tcp tcp_max_buf 16777216

    ```

当 NGINX 作为邮件或 HTTP 代理时，它需要与上游服务器建立许多连接。为了尽可能多地启用连接，应该将临时 TCP 端口范围调整到最大值。

+   Linux

    ```
    vi /etc/sysctl.conf

    net.ipv4.ip_local_port_range = 1024 65535
    # sysctl -p /etc/sysctl.conf

    ```

+   FreeBSD

    ```
    vi /etc/sysctl.conf

    net.inet.ip.portrange.first=1024
    net.inet.ip.portrange.last=65535
    # /etc/rc.d/sysctl reload

    ```

+   Solaris

    ```
    # ndd -set /dev/tcp tcp_smallest_anon_port 1024
    # ndd -set /dev/tcp tcp_largest_anon_port 65535

    ```

调整了这些基本值后，我们将在下一节中查看更具体的与性能相关的参数。

# 性能问题

在设计应用程序并配置 NGINX 来提供服务时，我们期望它能良好运行。然而，当我们遇到性能问题时，我们需要检查可能导致问题的原因。问题可能出在应用程序本身，也可能出在我们的 NGINX 配置上。我们将研究如何发现问题所在。

在代理时，NGINX 大部分工作是在网络上进行的。如果在网络层面存在任何限制，NGINX 就无法达到最佳性能。网络调优同样是特定于运行 NGINX 的操作系统和网络环境的，因此在具体情况下应检查这些调优参数。

与网络性能相关的最重要的值之一是新 TCP 连接的 `listen` 队列大小。此数值应增加，以便支持更多的客户端。如何进行此操作以及使用什么值，取决于操作系统和优化目标。

+   Linux

    ```
    vi /etc/sysctl.conf

    net.core.somaxconn = 3240000
    # sysctl -p /etc/sysctl.conf

    ```

+   FreeBSD

    ```
    vi /etc/sysctl.conf

    kern.ipc.somaxconn=4096
    # /etc/rc.d/sysctl reload

    ```

+   Solaris

    ```
    # ndd -set /dev/tcp tcp_conn_req_max_q 1024
    # ndd -set /dev/tcp tcp_conn_req_max_q0 4096

    ```

下一个要更改的参数是发送和接收缓冲区的大小。请注意，这些值仅用于示范目的——它们可能导致过多的内存使用，因此一定要在特定场景下进行测试。

+   Linux

    ```
    vi /etc/sysctl.conf

    net.ipv4.tcp_wmem = 8192 87380 1048576
    net.ipv4.tcp_rmem = 8192 87380 1048576
    # sysctl -p /etc/sysctl.conf

    ```

+   FreeBSD

    ```
    vi /etc/sysctl.conf

    net.inet.tcp.sendspace=1048576
    net.inet.tcp.recvspace=1048576
    # /etc/rc.d/sysctl reload

    ```

+   Solaris

    ```
    # ndd -set /dev/tcp tcp_xmit_hiwat 1048576
    # ndd -set /dev/tcp tcp_recv_hiwat 1048576

    ```

你还可以直接在 NGINX 的配置中更改这些缓冲区，以使其仅对 NGINX 有效，而不影响机器上运行的其他软件。当你有多个服务运行时，这可能是必要的，但你希望确保 NGINX 能够充分利用你的网络堆栈。

```
server {

    listen 80 sndbuf=1m rcvbuf=1m;

}
```

根据你的网络设置，你会注意到性能有显著变化。不过，你应该检查你的具体设置，并一次只做一个更改，观察每个更改后的结果。性能调优可以在很多不同的层次上进行，这里简要的讨论并没有充分涵盖这一主题。如果你有兴趣深入了解性能调优，应该阅读一些相关书籍和在线资源。

### 提示

**在 Solaris 中使网络调优更改持久化**

在前两个部分中，我们更改了多个 TCP 级别的参数。在 Linux 和 FreeBSD 中，由于这些更改也会在系统配置文件中进行（例如，`/etc/sysctl.conf`），因此这些更改在重启后会持久化。对于 Solaris，情况则不同。这些更改并没有在`sysctls`中进行，因此无法在该文件中持久化。

Solaris 10 及以上版本提供了**服务管理框架**（**SMF**）。这是一种独特的服务管理方式，并确保重启时服务的启动顺序。（当然，它不仅仅是这个，但这种过度简化在此处是适用的。）为了使前面提到的 TCP 级别更改持久化，我们可以编写一个 SMF 清单和相应的脚本来应用这些更改。

这些内容在附录 D，*持久化 Solaris 网络调优*中有详细说明。

# 使用 Stub Status 模块

NGINX 提供了一个内省模块，输出有关其运行方式的某些统计信息。此模块称为**Stub Status**，通过`--with-http_stub_status_module`配置标志启用。

要查看此模块生成的统计信息，`stub_status`指令需要设置为`on`。应为此模块创建一个单独的`location`指令，以便可以应用访问控制列表（ACL）：

```
location /nginx_status {

    stub_status on;

    access_log off;

    allow 127.0.0.1;

    deny all;

}
```

从本地主机调用此 URI（例如，使用`curl` `http://localhost/nginx_status`）将显示类似以下内容的输出：

```
Active connections: 2532
server accepts handled requests
 1476737983 1476737983 3553635810
Reading: 93 Writing: 13 Waiting: 2426

```

在这里我们看到有 2,532 个打开的连接，其中 NGINX 当前正在读取 93 个请求头，13 个连接处于 NGINX 正在读取请求体、处理请求或向客户端写响应的状态。剩下的 2,426 个请求被视为`keepalive`连接。自从启动此`nginx`进程以来，它已经接受并处理了 1,476,737,983 个连接，意味着没有一个连接在被接受后立即关闭。通过这 1,476,737,983 个连接，共处理了 3,553,635,810 个请求，意味着每个连接平均处理了约 2.4 个请求。

这类数据可以使用你喜欢的系统监控工具链进行收集和绘制图表。Munin、Nagios、collectd 等都有插件，利用`stub_status`模块收集统计信息。随着时间推移，你可能会注意到某些趋势，并能够将其与特定因素关联，但前提是数据已被收集。用户流量的激增以及操作系统的变化应该在这些图表中可见。

# 总结

在将一款新软件投入生产环境时，会在多个层面上出现问题。有些错误可以在测试环境中进行测试并消除；而另一些问题只有在实际负载和真实用户下才会显现。为了发现这些问题的原因，NGINX 提供了非常详细的日志记录，涵盖多个层级。某些消息可能有多种解释，但总体模式是可以理解的。通过调整配置并查看产生的错误信息，我们可以大致了解如何解读错误日志中的条目。操作系统对 NGINX 的运行有影响，因为它对多用户系统的默认设置施加了一些限制。理解 TCP 层面的情况有助于在实际负载下调整这些参数以适应条件。通过我们排除故障的介绍，我们看到`stub_status`模块能够提供哪些信息。这些数据有助于我们大致了解 NGINX 的性能表现。

接下来是附录。第一个附录是指令参考，列出了 NGINX 的所有配置指令，包括默认值以及它们可以在哪些上下文中使用。
