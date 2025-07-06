# 第六章：NGINX HTTP 服务器

HTTP 服务器主要是一种软件，在客户端请求时提供网页。这些网页可以是任何内容，从磁盘上的简单 HTML 文件到一个多组件框架，动态更新用户特定内容，通过 AJAX 或 WebSocket 传输。NGINX 是模块化的，旨在处理任何必要的 HTTP 服务。

在本章中，我们将研究各种模块，它们共同工作使 NGINX 成为一个可扩展的 HTTP 服务器。本章包括以下主题：

+   NGINX 的架构

+   HTTP 核心模块

+   使用限制防止滥用

+   限制访问

+   流媒体文件

+   预定义变量

+   使用 NGINX 与 PHP-FPM

+   将 NGINX 与 uWSGI 连接起来

# NGINX 的架构

NGINX 由一个主进程和多个工作进程组成。每个进程都是单线程的，旨在同时处理数千个连接。工作进程是大部分操作发生的地方，因为它是处理客户端请求的组件。NGINX 利用操作系统的事件机制迅速响应这些请求。

NGINX 的**主进程**负责读取配置、处理套接字、生成工作进程、打开日志文件以及编译嵌入的 Perl 脚本。主进程是响应管理员请求的进程，通过信号来执行。

NGINX 的**工作进程**在一个紧密的事件循环中运行，以处理传入的连接。每个 NGINX 模块都被构建到工作进程中，因此任何请求处理、过滤、代理连接的处理等都在工作进程内完成。由于这种工作进程模型，操作系统可以单独处理每个进程，并优化地调度进程在每个处理器核心上运行。如果有任何可能阻塞工作进程的过程，如磁盘 I/O，可以配置比核心更多的工作进程来分担负载。

还有少数辅助进程是由 NGINX 主进程创建的，用于处理专门的任务。其中包括**缓存加载器**和**缓存管理器**进程。缓存加载器负责为工作进程准备缓存使用的元数据。缓存管理器进程负责检查缓存项并使无效的项过期。

NGINX 是以模块化的方式构建的。主进程提供了一个基础平台，在此基础上，每个模块都可以执行其功能。每个协议和处理器都被实现为自己的模块。各个模块被串联成一个管道，用于处理连接和处理请求。请求处理完毕后，会传递给一系列的过滤器，在这些过滤器中，响应会被处理。其中一个过滤器负责处理子请求，这是 NGINX 最强大的功能之一。

子请求是 NGINX 返回与客户端发送的 URI 不同的请求结果的方式。根据配置，它们可能会嵌套多次，并调用其他子请求。过滤器可以收集来自多个子请求的响应，并将它们合并为一个响应发送给客户端。然后，响应会被最终确定并发送给客户端。在这个过程中，多个模块会发挥作用。有关 NGINX 内部的详细解释，请参阅 [`www.aosabook.org/en/nginx.html`](http://www.aosabook.org/en/nginx.html)。

在本章余下部分，我们将探索 `http` 模块及一些辅助模块。

# HTTP 核心模块

`http` 模块是 NGINX 的核心模块，负责处理所有通过 HTTP 与客户端的交互。我们已经在 第二章，*配置指南* 中讨论了该模块的以下方面：

+   客户端指令

+   文件 I/O 指令

+   哈希指令

+   套接字指令

+   `listen` 指令

+   匹配请求到 `server_name` 和 `location` 指令

我们将在本节剩余部分查看其余的指令，仍然按照类型进行划分。

## 服务器

`server` 指令启动一个新的上下文。到目前为止，我们已经看到过它的使用示例。尚未深入探讨的一个方面是**默认服务器**的概念。

在 NGINX 中，默认服务器意味着它是某个配置中第一个定义的，具有与其他服务器相同 `listen` IP 地址和端口的服务器。默认服务器也可以通过 `listen` 指令的 `default_server` 参数来表示。

默认服务器用于定义一组通用指令，这些指令将被重复使用，用于后续监听相同 IP 地址和端口的服务器：

```
server {

    listen 127.0.0.1:80;

    server_name default.example.com;

    server_name_in_redirect on;

}

server {

    listen 127.0.0.1:80;

    server_name www.example.com;

}
```

在这个示例中，`www.example.com` 服务器将设置 `server_name_in_redirect` 指令为 `on`，以及 `default.example.com` 服务器。请注意，如果这两个服务器没有 `listen` 指令，它们仍然会匹配相同的 IP 地址和端口号（即 `*:80` 的默认值）。不过，继承性并不保证。只有少数几个指令是会继承的，且哪些指令继承会随时间变化。

默认服务器的一个更好用途是处理所有进入该 IP 地址和端口的请求，并且没有 `Host` 头。如果你不希望默认服务器处理没有 `Host` 头的请求，可以定义一个空的 `server_name` 指令。该服务器将会匹配这些请求。

```
server {

    server_name "";

}
```

以下表格总结了与 `server` 相关的指令：

### 表：HTTP 服务器指令

| 指令 | 解释 |
| --- | --- |
| `port_in_redirect` | 决定是否在 NGINX 发出的重定向中指定端口。 |
| `server` | 创建一个新的配置上下文，定义虚拟主机。`listen`指令指定 IP 地址和端口；`server_name`指令列出此上下文匹配的`Host`头值。 |
| `server_name` | 配置虚拟主机可以响应的名称。 |
| `server_name_in_redirect` | 在此上下文中，激活 NGINX 在任何重定向中使用`server_name`指令的第一个值。 |
| `server_tokens` | 禁用在错误消息和`Server`响应头中发送 NGINX 版本字符串（默认值为`on`）。 |

## 日志

NGINX 拥有非常灵活的日志模型。每个配置级别都可以有一个访问日志。此外，每个级别可以指定多个不同的`log_format`的访问日志。`log_format`指令允许您精确指定将记录什么内容，并且需要在`http`部分中定义。

日志文件路径本身可以包含变量，因此可以构建动态配置。以下示例描述了如何将其付诸实践：

```
http {

    log_format vhost '$host $remote_addr - $remote_user [$time_local] ' 
                    '"$request" $status $body_bytes_sent ' 
                    '"$http_referer" "$http_user_agent"';

    log_format downloads '$time_iso8601 $host $remote_addr '
                    '"$request" $status $body_bytes_sent $request_time';

    open_log_file_cache max=1000 inactive=60s;

    access_log logs/access.log;

    server {

        server_name ~^(www\.)?(.+)$;

        access_log logs/combined.log vhost;

        access_log logs/$2/access.log;

        location /downloads {

            access_log logs/downloads.log downloads;

        }

    }

}
```

以下表格描述了前述代码中使用的指令：

### 表格：HTTP 日志指令

| 指令 | 解释 |
| --- | --- |
| `access_log` | 描述如何以及在哪里写入访问日志。第一个参数是存储日志的文件路径。可以使用变量来构建路径。特殊值`off`禁用访问日志。第二个可选参数指示将用于编写日志的`log_format`。如果未配置第二个参数，则使用预定义的组合格式。第三个可选参数指示缓冲区的大小，如果使用写入缓冲来记录日志，则此大小不能超过该文件系统的原子磁盘写入大小。如果第三个参数为`gzip`，则会在写入时即时压缩缓冲日志，前提是`nginx`二进制文件已使用`zlib`库构建。最后的`flush`参数指示在刷新到磁盘之前，缓冲的日志数据可以保留在内存中的最大时间长度。 |
| `log_format` | 指定日志文件中应出现的字段及其格式。有关日志特定变量的描述，请参阅下表。 |
| `log_not_found` | 禁用在错误日志中报告 404 错误（默认值为`on`）。 |
| `log_subrequest` | 启用在访问日志中记录子请求（默认值为`off`）。 |

| `open_log_file_cache` | 存储带有路径中变量的`access_logs`中使用的打开文件描述符的缓存。使用的参数为：

+   `max`：缓存中的最大文件描述符数

+   `inactive`：NGINX 将等待一段时间以便将内容写入此日志文件，然后关闭其文件描述符。

+   `min_uses`：在`inactive`期间，文件描述符必须使用此次数才能保持打开状态

+   `valid`：NGINX 会经常检查是否文件描述符仍然与具有相同名称的文件匹配

+   `off`：禁用缓存

|

在下面的示例中，日志条目将以 gzip 等级 4 进行压缩。缓冲区大小为默认的 64 KB，并且至少每分钟刷新一次到磁盘。

```
access_log /var/log/nginx/access.log.gz combined gzip=4 flush=1m;
```

请注意，当指定 `gzip` 时，`log_format` 参数是必需的。

默认的合并 `log_format` 是这样构建的：

```
log_format combined '$remote_addr - $remote_user [$time_local] ' 
 '"$request" $status $body_bytes_sent ' 
 '"$http_referer" "$http_user_agent"';

```

如你所见，换行符可以用来提高可读性，但不会影响 `log_format` 本身。`log_format` 指令中可以使用任何变量。下表中带有星号（`*`）的变量是特定于日志记录的，只能在 `log_format` 指令中使用。其他变量也可以在配置中的其他地方使用。

### 表格：日志格式变量

| 变量名 | 值 |
| --- | --- |
| `$body_bytes_sent` | 发送给客户端的字节数，不包括响应头。 |
| `$bytes_sent` | 发送给客户端的字节数。 |
| `$connection` | 序列号，用于标识唯一的连接。 |
| `$connection_requests` | 通过特定连接发起的请求数量。 |
| `$msec` | 秒数，带有毫秒精度。 |
| `$pipe *` | 指示请求是否使用流水线处理（`p`），如果没有使用流水线处理则为（`.`）。 |
| `$request_length *` | 请求的长度，包括 HTTP 方法、URI、HTTP 协议、头信息和请求体。 |
| `$request_time` | 请求处理时间，带有毫秒精度，从接收到客户端的第一个字节到发送给客户端的最后一个字节。 |
| `$status` | 响应状态。 |
| `$time_iso8601 *` | ISO8601 格式中的本地时间。 |
| `$time_local *` | 常见日志格式中的本地时间（`%d/%b/%Y:%H:%M:%S %z`）。 |

在本节中，我们仅关注 `access_log` 以及如何进行配置。你还可以配置 NGINX 来记录错误。`error_log` 指令在第八章，*故障排除*中有描述。

## 查找文件

为了让 NGINX 响应请求，它会将请求传递给内容处理器，内容处理器由 `location` 指令的配置决定。首先会尝试无条件的内容处理器：`perl`、`proxy_pass`、`flv`、`mp4` 等。如果没有匹配项，请求会依次传递给以下内容之一：`random index`、`index`、`autoindex`、`gzip_static`、`static`。带有尾部斜杠的请求由某个索引处理器处理。如果没有启用 gzip，则由静态模块处理请求。这些模块如何在文件系统中查找适当的文件或目录是由某些指令的组合决定的。`root` 指令最好在默认的 `server` 指令中定义，或者至少在特定的 `location` 指令外部定义，这样它对整个服务器都是有效的：

```
server {

    root /home/customer/html;

    location / {

        index index.html index.htm;

    }
    location /downloads {

        autoindex on;

    }

}
```

在前面的示例中，所有待服务的文件都位于根目录 `/home/customer/html` 下。如果客户端只输入了域名，NGINX 将尝试提供 `index.html` 文件。如果该文件不存在，则 NGNIX 会提供 `index.htm` 文件。当用户在浏览器中输入 `/downloads` URI 时，他们将看到一个以 HTML 格式呈现的目录列表。这使得用户能够轻松访问托管软件下载站点。NGINX 会自动重写目录的 URI，使其包含尾部斜杠，并发出 HTTP 重定向。NGINX 将 URI 附加到 `root`，以查找要交付给客户端的文件。如果该文件不存在，客户端将收到 **404 Not Found** 错误信息。如果你不想让错误信息返回给客户端，一个可选的方案是尝试从不同的文件系统位置交付文件，如果这些选项都不可用，则退回到一个通用页面。`try_files` 指令可以按如下方式使用：

```
location / {

    try_files $uri $uri/ backups/$uri /generic-not-found.html;

}
```

作为安全预防措施，NGINX 可以检查其即将交付的文件的路径，如果路径中包含符号链接，它将向客户端返回错误信息：

```
server {

    root /home/customer/html;

    disable_symlinks if_not_owner from=$document_root;

}
```

在上述示例中，如果在 `/home/customer/html` 后发现符号链接，且该符号链接与指向的文件不属于同一用户 ID，NGINX 将返回“权限拒绝”错误。

以下表格总结了这些指令：

### 表格：HTTP 文件路径指令

| 指令 | 说明 |
| --- | --- |

| `disable_symlinks` | 确定 NGINX 是否在将文件交付给客户端之前，对文件路径进行符号链接检查。以下参数被识别：

+   `off`：禁用符号链接检查（默认设置）

+   `on`：如果路径中的任何部分是符号链接，访问将被拒绝

+   `if_not_owner`：如果路径中的任何部分包含符号链接，并且该链接和引用目标文件的所有者不同，则访问该文件将被拒绝

+   `from=part`：当指定时，`part`之前的路径不检查符号链接，之后的路径根据`on`或`if_not_owner`参数进行检查。

|

| `root` | 设置文档根目录的路径。文件通过将 URI 附加到该指令的值来找到。 |
| --- | --- |
| `try_files` | 测试作为参数给定的文件是否存在。如果没有找到任何先前的文件，则使用最后一项作为回退，因此确保该路径或命名的 `location` 存在，或者设置为返回由 `=<status code>` 指定的状态代码。 |

## 名称解析

如果在 `upstream` 或 `*_pass` 指令中使用逻辑名称而非 IP 地址，NGINX 默认会使用操作系统的解析器来获取 IP 地址，这是它连接到服务器所需要的。此过程仅发生一次，即第一次请求 `upstream` 时。如果在 `*_pass` 指令中使用变量，则无法生效。然而，您可以为 NGINX 配置一个单独的解析器。通过这样做，您可以覆盖 DNS 返回的 TTL，并且在 `*_pass` 指令中使用变量。

```
server {

    resolver 192.168.100.2 valid=300s;

}
```

### 表格：名称解析指令

| 指令 | 说明 |
| --- | --- |
| `resolver` | 配置一个或多个名称服务器，用于将上游服务器名称解析为 IP 地址。可选的 `valid` 参数可以覆盖域名记录的 TTL。 |

为了让 NGINX 重新解析一个 IP 地址，可以将逻辑名称放入一个变量。当 NGINX 解析该变量时，它会隐式地执行 DNS 查找以找到该 IP 地址。要使此操作生效，必须配置 `resolver` 指令：

```
server {

    resolver 192.168.100.2;

    location / {

        set $backend upstream.example.com;

        proxy_pass http://$backend;

    }

}
```

当然，通过依赖 DNS 查找上游服务器，你的操作会依赖于解析器始终可用。当解析器不可达时，会发生网关错误。为了尽量缩短客户端等待时间，`resolver_timeout` 参数应设置得较低。然后，可以通过专门设计的 `error_page` 来处理网关错误。

```
server {

    resolver 192.168.100.2;

    resolver_timeout 3s;

    error_page 504 /gateway-timeout.html;
    location / {

        proxy_pass http://upstream.example.com;

    }

}
```

## 客户端交互

NGINX 与客户端的交互方式有多种。这些方式可以包括连接本身的属性（IP 地址、超时、保持连接等）到内容协商头。以下表格列出了如何设置各种头部和响应码，以使客户端请求正确的页面或从缓存中提供该页面：

### 表格：HTTP 客户端交互指令

| 指令 | 说明 |
| --- | --- |
| `default_type` | 设置响应的默认 MIME 类型。如果文件的 MIME 类型无法与 `types` 指令中指定的类型匹配，则会使用此默认值。 |
| `error_page` | 定义当遇到错误级别响应码时需要提供的 URI。通过添加 `=` 参数，可以更改响应码。如果该参数为空，则响应码将从 URI 中获取，并且此 URI 必须由某种上游服务器提供。 |
| `etag` | 禁用自动生成静态资源的 `ETag` 响应头（默认是 `on`）。 |

| `if_modified_since` | 控制如何将响应的修改时间与 `If-Modified-Since` 请求头的值进行比较：

+   `off`: 忽略 `If-Modified-Since` 头部

+   `exact`: 进行精确匹配（默认）

+   `before`: 响应的修改时间小于或等于 `If-Modified-Since` 头部的值

|

| `ignore_invalid_headers` | 禁用忽略具有无效名称的头部（默认值为`on`）。有效的名称由 ASCII 字母、数字、连字符和可能的下划线组成（由`underscores_in_headers`指令控制）。 |
| --- | --- |
| `merge_slashes` | 禁用多个斜杠的删除。默认值为`on`，这意味着 NGINX 会将两个或多个`/`字符压缩为一个。 |
| `recursive_error_pages` | 启用通过`error_page`指令进行多个重定向（默认值为`off`）。 |
| `types` | 设置 MIME 类型与文件扩展名的映射。NGINX 自带一个`conf`/`mime.types`文件，包含了大多数 MIME 类型的映射。通过`include`加载此文件应当足以满足大多数需求。 |
| `underscores_in_headers` | 启用在客户端请求头中使用下划线字符。如果保持默认值`off`，则此类头部的评估将受`ignore_invalid_headers`指令的值控制。 |

`error_page`指令是 NGINX 最灵活的指令之一。使用该指令时，我们可以在出现错误条件时提供任何页面。此页面可以位于本地机器上，也可以是由应用服务器生成的动态页面，甚至可以是完全不同站点上的页面。

```
http {

    # a generic error page to handle any server-level errors
    error_page 500 501 502 503 504 share/examples/nginx/50x.html;

    server {

        server_name www.example.com;

        root /home/customer/html;

        # for any files not found, the page located at 
        #  /home/customer/html/404.html will be delivered
        error_page 404 /404.html;
        location / {

            # any server-level errors for this host will be directed
            # to a custom application handler
            error_page 500 501 502 503 504 = @error_handler;

        }

        location /microsite {

            # for any non-existent files under the /microsite URI,
            #  the client will be shown a foreign page
            error_page 404 http://microsite.example.com/404.html;

        }

        # the named location containing the custom error handler
        location @error_handler {

            # we set the default type here to ensure the browser
            # displays the error page correctly
            default_type text/html;

            proxy_pass http://127.0.0.1:8080;

        }

    }

}
```

# 使用限制来防止滥用

我们建立和托管网站是因为我们希望用户能访问它们。我们希望我们的网站始终能够为合法访问提供服务。这意味着我们可能需要采取措施来限制对恶意用户的访问。我们可能将“恶意”定义为从每秒一个请求到同一 IP 地址的多个连接。滥用还可以表现为**DDOS**（**分布式拒绝服务攻击**），即全球各地运行的机器人在尽可能多的时间内同时访问网站。在本节中，我们将探讨应对每种滥用行为的方法，以确保我们的网站可用。

首先，让我们来看一下不同的配置指令，这些指令将帮助我们实现目标：

## 表格：HTTP 限制指令

| 指令 | 解释 |
| --- | --- |
| `limit_conn` | 指定一个共享内存区域（通过`limit_conn_zone`配置）以及每个键值允许的最大连接数。 |
| `limit_conn_log_level` | 当 NGINX 由于`limit_conn`指令限制连接时，此指令指定报告该限制的日志级别。 |
| `limit_conn_zone` | 在`limit_conn`中指定要限制的键作为第一个参数。第二个参数`zone`表示用于存储该键及每个键当前连接数的共享内存区域的名称，以及该区域的大小（`name:size`）。 |
| `limit_rate` | 限制客户端下载内容的速率（以字节每秒为单位）。速率限制在连接级别生效，意味着单个客户端可以通过打开多个连接来增加其吞吐量。 |
| `limit_rate_after` | 在传输此字节数后，启动`limit_rate`限制。 |
| `limit_req` | 在共享内存存储区（通过`limit_req_zone`配置）中设置特定键的请求数量限制，并支持突发功能。突发量可以通过第二个参数指定。如果请求之间不应有延迟，直到达到突发量，则需要配置第三个参数`nodelay`。 |
| `limit_req_log_level` | 当 NGINX 因`limit_req`指令限制请求数量时，该指令指定了该限制在日志中报告的日志级别。延迟会在比此处指示的级别低一个级别的日志中记录。 |
| `limit_req_zone` | 在`limit_req`中，指定要限制的键作为第一个参数。第二个参数，区域，表示用于存储键、当前每个键请求数量以及该区域大小（`name:size`）的共享内存区域的名称。第三个参数，`rate`，配置每秒（r/s）或每分钟（r/m）的请求数量，在达到限制之前。 |
| `max_ranges` | 设置字节范围请求中允许的最大范围数。指定`0`将禁用字节范围支持。 |

在这里，我们限制每个独立 IP 地址最多 10 个连接。对于正常浏览，这应该足够了，因为现代浏览器每个主机会打开两到三个连接。然而，请记住，任何位于代理后面的用户都会显得来自同一个地址。所以要观察错误代码 503（服务不可用）的日志，意味着这个限制已生效：

```
http {

    limit_conn_zone $binary_remote_addr zone=connections:10m;

    limit_conn_log_level notice;

    server {

        limit_conn connections 10;

    }

}
```

基于速率限制访问看起来几乎一样，但其工作方式略有不同。限制用户每单位时间可请求的页面数量时，NGINX 会在第一次页面请求后插入延迟，直到突发量达到为止。这可能是你需要的，也可能不是，所以 NGINX 提供了使用`nodelay`参数移除延迟的可能性：

```
http {

    limit_req_zone $binary_remote_addr zone=requests:10m rate=1r/s;

    limit_req_log_level warn;

    server {

        limit_req zone=requests burst=10 nodelay;

    }

}
```

### 提示

**使用 $binary_remote_addr**

我们在前面的示例中使用了`$binary_remote_addr`变量，以准确知道存储一个 IP 地址需要多少空间。该变量在 32 位平台上占用 32 个字节，在 64 位平台上占用 64 个字节。因此，我们之前配置的`10m`区域在 32 位平台上最多可以容纳 320,000 个状态，或在 64 位平台上最多可以容纳 160,000 个状态。

我们还可以限制每个客户端的带宽。这样可以确保少数客户端不会占用所有的可用带宽。不过有一个注意事项：`limit_rate`指令是基于连接工作的。即使一个客户端被允许打开多个连接，它仍然可以绕过这个限制：

```
location /downloads {

    limit_rate 500k;

}
```

另外，我们可以允许一种突发模式，允许自由下载较小的文件，但确保较大的文件受到限制：

```
location /downloads {

    limit_rate_after 1m;

    limit_rate 500k;

}
```

结合这些不同的速率限制，我们可以创建一个非常灵活的配置，以控制客户端在哪些地方和如何被限制：

```
http {

    limit_conn_zone $binary_remote_addr zone=ips:10m;

    limit_conn_zone $server_name zone=servers:10m;

    limit_req_zone $binary_remote_addr zone=requests:10m rate=1r/s;

    limit_conn_log_level notice;

    limit_req_log_level warn;

    reset_timedout_connection on;

    server {

        # these limits apply to the whole virtual server
        limit_conn ips 10;
        # only 1000 simultaneous connections to the same server_name
        limit_conn servers 1000;

        location /search {

            # here we want only the /search URL to be rate-limited
            limit_req zone=requests burst=3 nodelay;

        }

        location /downloads {

            # using limit_conn to ensure that each client is # bandwidth-limited
            #  with no getting around it
            limit_conn connections 1;

            limit_rate_after 1m;

            limit_rate 500k;

        }

    }

}
```

# 限制访问

在上一节中，我们探讨了限制 NGINX 运行的网站免受恶意访问的方法。现在，我们将看看如何限制访问整个网站或其特定部分。访问限制有两种形式：限制某一特定 IP 地址集合的访问，或限制某一特定用户集合的访问。这两种方法也可以结合使用，以满足某些用户可以从特定 IP 地址集合中访问网站，或者在通过有效的用户名和密码进行身份验证后访问的要求。

以下指令将帮助我们实现这些目标：

## 表格：HTTP 访问模块指令

| 指令 | 说明 |
| --- | --- |
| `allow` | 允许来自该 IP 地址、网络或`all`的访问。 |
| `auth_basic` | 启用使用 HTTP 基本认证的身份验证。参数字符串用于作为区域名称。如果使用特殊值`off`，则表示父级配置的`auth_basic`值被否定。 |
| `auth_basic_user_file` | 指定用于身份验证用户的`username:password:comment`元组文件的位置。`password`字段需要使用 crypt 算法加密。`comment`字段是可选的。 |
| `deny` | 拒绝来自该 IP 地址、网络或`all`的访问。 |
| `satisfy` | 如果前面的`all`或`any`指令授权访问，则允许访问。默认值`all`表示用户必须来自特定网络地址并输入正确的密码。 |

要限制来自某一特定 IP 地址集合的客户端访问，可以按如下方式使用`allow`和`deny`指令：

```
location /stats {

    allow 127.0.0.1;
    deny all;

}
```

此配置将只允许从本地主机访问`/stats` URI。

要限制只有经过身份验证的用户才能访问，可以按如下方式使用`auth_basic`和`auth_basic_user_file`指令：

```
server {

    server_name restricted.example.com;

    auth_basic "restricted";

    auth_basic_user_file conf/htpasswd;

}
```

任何想要访问`restricted.example.com`的用户，都需要提供与 NGINX 根目录下`conf`目录中的`htpasswd`文件匹配的凭证。`htpasswd`文件中的条目可以使用任何支持标准 UNIX `crypt()`函数的工具生成。例如，以下 Ruby 脚本将生成适当格式的文件：

```
#!/usr/bin/env ruby

# setup the command-line options
require 'optparse'

OptionParser.new do |o|

  o.on('-f FILE') { |file| $file = file }

  o.on('-u', "--username USER") { |u| $user = u }

  o.on('-p', "--password PASS") { |p| $pass = p }

  o.on('-c', "--comment COMM (optional)") { |c| $comm = c }

  o.on('-h') { puts o; exit }

  o.parse!

  if $user.nil? or $pass.nil?
    puts o; exit

  end

end

# initialize an array of ASCII characters to be used for the salt
ascii = ('a'..'z').to_a + ('A'..'Z').to_a + ('0'..'9').to_a + [ ".", "/" ]

$lines = []

begin

  # read in the current http auth file
  File.open($file) do |f|

    f.lines.each { |l| $lines << l }

  end

rescue Errno::ENOENT

  # if the file doesn't exist (first use), initialize the array
  $lines = ["#{$user}:#{$pass}\n"]

end

# remove the user from the current list, since this is the one we're editing
$lines.map! do |line|

  unless line =~ /#{$user}:/

    line

  end

end

# generate a crypt()ed password
pass = $pass.crypt(ascii[rand(64)] + ascii[rand(64)])
# if there's a comment, insert it
if $comm

  $lines << "#{$user}:#{pass}:#{$comm}\n"

else

  $lines << "#{$user}:#{pass}\n"

end

# write out the new file, creating it if necessary

File.open($file, File::RDWR|File::CREAT) do |f|

  $lines.each { |l| f << l}

end
```

将此文件保存为`http_auth_basic.rb`，并指定一个文件名（`-f`）、一个用户（`-u`）和一个密码（`-p`），它将生成适用于 NGINX `auth_basic_user_file`指令的条目：

```
$ ./http_auth_basic.rb -f htpasswd -u testuser -p 123456
```

要处理在某些 IP 地址集合之外才需要输入用户名和密码的场景，NGINX 有一个`satisfy`指令。`any`参数用于表示这种“任一或”的场景：

```
server {

    server_name intranet.example.com;

    location / {

        auth_basic "intranet: please login";

        auth_basic_user_file conf/htpasswd-intranet;

        allow 192.168.40.0/24;

        allow 192.168.50.0/24;

        deny all;
          satisfy any;

    }
```

如果要求配置为用户必须来自某一特定 IP 地址并提供身份验证，则默认值为`all`。因此，我们省略`satisfy`指令本身，只包括`allow`、`deny`、`auth_basic`和`auth_basic_user_file`：

```
server {

    server_name stage.example.com;

    location / {

        auth_basic "staging server";

        auth_basic_user_file conf/htpasswd-stage;

        allow 192.168.40.0/24;

        allow 192.168.50.0/24;

        deny all;

    }
```

# 流媒体文件

NGINX 能够提供某些视频媒体类型的服务。包括在基础发行版中的 `flv` 和 `mp4` 模块可以执行所谓的 **伪流媒体** 功能。这意味着 NGINX 会跳转到视频文件中的某个位置，位置由 `start` 请求参数指定。

为了使用伪流媒体功能，需要在编译时包含相应的模块：`--with-http_flv_module` 用于 Flash Video (FLV) 文件，`--with-http_mp4_module` 用于 H.264/AAC 文件。然后，以下指令将变为可配置项：

## 表格：HTTP 流媒体指令

| 指令 | 解释 |
| --- | --- |
| `flv` | 为此位置激活 `flv` 模块。 |
| `mp4` | 为此位置激活 `mp4` 模块。 |
| `mp4_buffer_size` | 设置传送 MP4 文件的初始缓冲区大小。 |
| `mp4_max_buffer_size` | 设置用于处理 MP4 元数据的缓冲区最大大小。 |

激活 FLV 伪流媒体功能只需要在配置中加入 `flv` 关键字：

```
location /videos {

    flv;

}
```

MP4 伪流媒体有更多的选项，因为 H.264 格式包含需要解析的元数据。一旦播放器解析了“moov atom”，就可以进行跳转。所以为了优化性能，请确保元数据位于文件的开头。如果日志中出现如下错误信息，则需要增大 `mp4_max_buffer_size`：

```
mp4 moov atom is too large

```

`mp4_max_buffer_size` 可以通过以下方式增大：

```
location /videos {

    mp4;

    mp4_buffer_size 1m;

    mp4_max_buffer_size 20m;

}
```

# 预定义变量

NGINX 使得根据变量的值构建配置变得简单。你不仅可以使用 `set` 或 `map` 指令来实例化自己的变量，而且还有一些在 NGINX 中预定义的变量。它们经过优化以便快速评估，并且在请求的生命周期内缓存其值。你可以将它们中的任何一个作为 `if` 语句的键，或传递给代理。如果你定义自己的日志文件格式，一些变量可能会非常有用。不过，如果你尝试重新定义它们，你将会收到如下的错误信息：

```
<timestamp> [emerg] <master pid>#0: the duplicate "<variable_name>" variable in <path-to-configuration-file>:<line-number>

```

这些变量也不适用于宏扩展配置——它们主要在运行时使用。

以下是 `http` 模块中定义的变量及其值：

## 表格：HTTP 变量

| 变量名 | 值 |
| --- | --- |
| `$arg_name` | 请求参数中的 `name` 参数。 |
| `$args` | 所有请求参数。 |
| `$binary_remote_addr` | 客户端的 IP 地址（以二进制形式表示，始终为 4 字节）。 |
| `$content_length` | `Content-Length` 请求头的值。 |
| `$content_type` | `Content-Type` 请求头的值。 |
| `$cookie_name` | 名为 `name` 的 cookie。 |
| `$document_root` | 当前请求的 `root` 或 `alias` 指令的值。 |
| `$document_uri` | `$uri` 的别名。 |
| `$host` | 请求头中 `Host` 的值（如果存在）。如果该头部不存在，则其值等于与请求匹配的 `server_name`。 |
| `$hostname` | 运行 NGINX 的主机名。 |
| `$http_name` | `name` 请求头的值。如果该头部有连字符，它们将被转换为下划线；大写字母将转换为小写字母。 |
| `$https` | 如果连接是通过 SSL 建立的，则该变量的值为 `on`。否则，它是一个空字符串。 |
| `$is_args` | 如果请求包含参数，则该变量的值为 `?`。否则，它是一个空字符串。 |
| `$limit_rate` | `limit_rate` 指令的值。如果未设置，则允许使用此变量设置速率限制。 |
| `$nginx_version` | 正在运行的 `nginx` 二进制文件的版本。 |
| `$pid` | 工作进程的进程 ID。 |
| `$query_string` | `$args` 的别名。 |
| `$realpath_root` | 当前请求的 `root` 或 `alias` 指令的值，所有符号链接已解析。 |
| `$remote_addr` | 客户端的 IP 地址。 |
| `$remote_port` | 客户端的端口。 |
| `$remote_user` | 使用 HTTP 基本认证时，设置为用户名。 |
| `$request` | 完整的请求，包括客户端发送的 HTTP 方法、URI、HTTP 协议、头部和请求体。 |
| `$request_body` | 请求体，用于 `*_pass directive` 指令处理的区域。 |
| `$request_body_file` | 请求体保存的临时文件路径。要保存该文件，`client_body_in_file_only` 指令需要设置为 `on`。 |
| `$request_completion` | 如果请求已完成，则该变量的值为 `OK`。否则，它是一个空字符串。 |
| `$request_filename` | 当前请求的文件路径，基于 `root` 或 `alias` 指令的值加上 URI。 |
| `$request_method` | 当前请求中使用的 HTTP 方法。 |
| `$request_uri` | 完整的请求 URI，包括客户端发送的参数。 |
| `$scheme` | 当前请求的协议类型，HTTP 或 HTTPS。 |
| `$sent_http_name` | `name` 响应头的值。如果该头部有连字符，它们将被转换为下划线；大写字母将转换为小写字母。 |
| `$server_addr` | 接受请求的服务器地址的值。 |
| `$server_name` | 接受请求的虚拟主机的 `server_name`。 |
| `$server_port` | 接受请求的服务器端口的值。 |
| `$server_protocol` | 当前请求中使用的 HTTP 协议。 |
| `$status` | 响应的状态。 |
| `$tcpinfo_rtt``$tcpinfo_rttvar``$tcpinfo_snd_cwnd``$tcpinfo_rcv_space` | 如果系统支持 `TCP_INFO` 套接字选项，这些变量将填充相关信息。 |
| `$uri` | 当前请求的标准化 URI。 |

# 使用 NGINX 配合 PHP-FPM

Apache 一直被认为是提供 PHP 网站服务的唯一选择，因为 `mod_php` Apache 模块使得将 PHP 直接集成到 Web 服务器中变得非常容易。随着 **PHP-FPM** 被纳入 PHP 核心，现在 PHP 分发包中就包含了一个替代方案。PHP-FPM 是一种在 FastCGI 服务器下运行 PHP 的方式。PHP-FPM 主进程负责创建工作进程、适应站点使用情况，并在必要时重新启动子进程。它使用 FastCGI 协议与其他服务进行通信。你可以在 [`php.net/manual/en/install.fpm.php`](http://php.net/manual/en/install.fpm.php) 了解更多关于 PHP-FPM 的信息。 |

NGINX 有一个 `fastcgi` 模块，能够与不仅仅是 PHP-FPM，还能与任何符合 FastCGI 协议的服务器进行通信。默认情况下此模块已启用，因此无需额外的配置即可开始使用 NGINX 与 FastCGI 服务器。 |

## 表格：FastCGI 指令 |

| 指令 | 说明 |
| --- | --- |
| `fastcgi_buffer_size` | 用于 FastCGI 服务器响应的第一部分的缓冲区大小，其中包含响应头。 |
| `fastcgi_buffers` | 用于从 FastCGI 服务器接收响应的缓冲区的数量和大小，针对单个连接。 |
| `fastcgi_busy_buffers_size` | 分配给向客户端发送响应时仍在从 FastCGI 服务器读取的缓冲区空间的总大小。通常设置为两个 `fastcgi_buffers`。 |
| `fastcgi_cache` | 定义一个共享内存区域，用于缓存。 |
| `fastcgi_cache_bypass` | 一个或多个字符串变量，当其非空或非零时，会导致响应直接从 FastCGI 服务器获取，而不是从缓存中获取。 |
| `fastcgi_cache_key` | 用作存储和检索缓存值的键的字符串。 |
| `fastcgi_cache_lock` | 启用此指令将防止多个请求进入相同的缓存键。 |
| `fastcgi_cache_lock_timeout` | 请求等待缓存中出现条目或等待 `fastcgi_cache_lock` 被释放的时间长度。 |
| `fastcgi_cache_min_uses` | 缓存响应之前，针对某个特定键所需的请求次数。 |

| `fastcgi_cache_path` | 放置缓存响应的目录以及一个共享内存区域（`keys_zone = name:size`）用于存储活动的键和响应元数据。可选参数包括： |

+   `levels`：每一层次子目录名称的长度，用冒号分隔（最多两层），最多三层深度。 |

+   `inactive`：一个非活动响应在缓存中停留的最大时间，超过该时间后会被清除。 |

+   `max_size`：缓存的最大大小；当大小超过此值时，缓存管理进程会删除最久未使用的项目。 |

+   `loader_files`：每次缓存加载器进程迭代时加载的最大缓存文件数（仅元数据）。 |

+   `loader_sleep`：缓存加载器进程每次迭代之间暂停的毫秒数。 |

+   `loader_threshold`：缓存加载器迭代可能花费的最长时间。

|

| `fastcgi_cache_use_stale` | 在访问 FastCGI 服务器时发生错误时，接受提供过期缓存数据的情况。`updating` 参数表示正在加载新数据的情况。 |
| --- | --- |
| `fastcgi_cache_valid` | 指示响应代码为 200、301 或 302 的缓存响应有效的时间长度。如果在时间参数之前给出可选的响应代码，则该时间仅适用于该响应代码。特殊参数 `any` 表示任何响应代码应缓存该时间长度。 |
| `fastcgi_connect_timeout` | NGINX 在向 FastCGI 服务器发起请求时，等待连接被接受的最长时间。 |
| `fastcgi_hide_header` | 不应传递给客户端的头字段列表。 |
| `fastcgi_ignore_client_abort` | 如果设置为 `on`，当客户端中止连接时，NGINX 将不会中止与 FastCGI 服务器的连接。 |
| `fastcgi_ignore_headers` | 设置在处理来自 FastCGI 服务器的响应时可以忽略哪些头部。 |
| `fastcgi_index` | 设置要附加到 `$fastcgi_script_name` 的文件名，该文件名以斜杠结尾。 |
| `fastcgi_intercept_errors` | 如果启用，NGINX 将显示配置的 `error_page`，而不是直接显示来自 FastCGI 服务器的响应。 |
| `fastcgi_keep_conn` | 通过指示服务器不立即关闭连接，启用与 FastCGI 服务器的 `keepalive` 连接。 |
| `fastcgi_max_temp_file_size` | 溢出文件的最大大小，当响应无法放入内存缓冲区时，写入该文件。 |

| `fastcgi_next_upstream` | 指示在什么条件下将选择下一个 FastCGI 服务器进行响应。如果客户端已经收到了响应，则不会使用此条件。条件由以下参数指定：

+   `error`：与 FastCGI 服务器通信时发生了错误。

+   `timeout`：与 FastCGI 服务器通信时发生了超时。

+   `invalid_header`：FastCGI 服务器返回了一个空的或其他无效的响应。

+   `http_500`：FastCGI 服务器响应了 500 错误代码。

+   `http_503`：FastCGI 服务器响应了 503 错误代码。

+   `http_404`：FastCGI 服务器响应了 404 错误代码。

+   `off`：当发生错误时，禁用将请求传递给下一个 FastCGI 服务器。

|

| `fastcgi_no_cache` | 一个或多个字符串变量，当它们非空或非零时，将指示 NGINX 不将来自 FastCGI 服务器的响应保存在缓存中。 |
| --- | --- |
| `fastcgi_param` | 设置一个参数及其值，并将其传递给 FastCGI 服务器。如果参数只应在值非空时传递，则应设置 `if_not_empty` 附加参数。 |
| `fastcgi_pass` | 指定将请求传递给的 FastCGI 服务器，可以是 `address:port` 组合，或 `unix:path` 用于 UNIX 域套接字。 |
| `fastcgi_pass_header` | 覆盖在 `fastcgi_hide_header` 中禁用的头部，允许它们发送给客户端。 |
| `fastcgi_read_timeout` | 指定从 FastCGI 服务器进行两次连续读取操作之间需要经过的时间长度，超过该时间后连接会被关闭。 |
| `fastcgi_send_timeout` | 两次连续写入 FastCGI 服务器操作之间需要经过的时间长度，超过该时间后连接会被关闭。 |
| `fastcgi_split_path_info` | 定义一个具有两个捕获组的正则表达式。第一个捕获组将是 `$fastcgi_script_name` 变量的值。第二个捕获组将成为 `$fastcgi_path_info` 变量的值。仅对于依赖 `PATH_INFO` 的应用程序是必需的。 |
| `fastcgi_store` | 启用将从 FastCGI 服务器获取的响应存储为磁盘上的文件。`on` 参数将使用 `alias` 或 `root` 指令作为存储文件的基础路径。也可以给定一个字符串，指示存储文件的替代位置。 |
| `fastcgi_store_access` | 设置新创建的 `fastcgi_store` 文件的访问权限。 |
| `fastcgi_temp_file_write_size` | 限制每次写入临时文件时缓冲的数据量，以便 NGINX 不会在单个请求上阻塞过长时间。 |
| `fastcgi_temp_path` | 用于缓冲从 FastCGI 服务器代理的临时文件的目录，可以选择多层深度。 |

## 一个 Drupal 配置示例：

Drupal ([`drupal.org`](http://drupal.org)) 是一个流行的开源内容管理平台，拥有大量安装用户基础，许多流行网站都运行在 Drupal 上。像大多数 PHP 网络框架一样，Drupal 通常在 Apache 下使用 `mod_php` 运行。我们将探讨如何配置 NGINX 来运行 Drupal。

有一个非常全面的 NGINX 配置指南，适用于 Drupal，可以在 [`github.com/perusio/drupal-with-nginx`](https://github.com/perusio/drupal-with-nginx) 上找到。它比我们这里能够深入探讨的内容更详细，但我们将指出一些提到的特性，并讲解 Drupal 6 与 Drupal 7 之间的一些差异：

```
## Defines the $no_slash_uri variable for drupal 6.
map $uri $no_slash_uri {

    ~^/(?<no_slash>.*)$ $no_slash;
}

server { 

    server_name www.example.com; 

    root /home/customer/html; 

    index index.php;

    # keep alive to the FastCGI upstream (used in conjunction with
    #  the "keepalive" directive in the upstream section)
    fastcgi_keep_conn on;

    # The 'default' location.
    location / {
        ## (Drupal 6) Use index.html whenever there's no index.php.
        location = / {
            error_page 404 =200 /index.html;
        }
        # Regular private file serving (i.e. handled by Drupal).
        location ^~ /system/files/ {

            include fastcgi_private_files.conf;

            fastcgi_pass 127.0.0.1:9000;
            # For not signaling a 404 in the error log whenever the
            # system/files directory is accessed add the line below.
            # Note that the 404 is the intended behavior.
            log_not_found off;

        }

        # Trying to access private files directly returns a 404.
        location ^~ /sites/default/files/private/ {
            internal;
        }

        ## (Drupal 6) If accessing an image generated by imagecache,
        ## serve it directly if available, if not relay the request to# Drupal
        ## to (re)generate the image.
        location ~* /imagecache/ {

            access_log off;

            expires 30d;

            try_files $uri /index.php?q=$no_slash_uri&$args;

        }

        # Drupal 7 image handling, i.e., imagecache in core
        location ~* /files/styles/ {

            access_log off;

            expires 30d;

            try_files $uri @drupal;

        }
```

接下来的高级聚合模块配置只在所用的 `location` 上有所不同。CSS 的高级聚合模块配置如下：

```
        # Advanced Aggregation module CSS support.
        location ^~ /sites/default/files/advagg_css/ {
            location ~* /sites/default/files/advagg_css/css_[[:alnum:]]+\.css$ {
```

对于 JavaScript，配置如下：

```
        # Advanced Aggregation module JS
        location ^~ /sites/default/files/advagg_js/ {
            location ~* /sites/default/files/advagg_js/js_[[:alnum:]]+\.js$ {
```

两个部分的共同配置项如下：

```
                access_log off;

                add_header Pragma '';

                add_header Cache-Control 'public, max-age=946080000';

                add_header Accept-Ranges '';

                # This is for Drupal 7
                try_files $uri @drupal;

                ## This is for Drupal 6 (use only one)
                try_files $uri /index.php?q=$no_slash_uri&$args;

            }

        }

        # All static files will be served directly.
        location ~* ^.+\.(?:css|cur|js|jpe?g|gif|htc|ico|png|html|xml)$ {

            access_log off;

            expires 30d;

            # Send everything all at once.
            tcp_nodelay off;

            # Set the OS file cache.
            open_file_cache max=3000 inactive=120s;
            open_file_cache_valid 45s;

            open_file_cache_min_uses 2;

            open_file_cache_errors off;

        }

        # PDFs and powerpoint files handling.
        location ~* ^.+\.(?:pdf|pptx?)$ {

            expires 30d;

            # Send everything all at once.
            tcp_nodelay off;

        }
```

服务音频文件展示了 AIO 的使用。MP3 的 `location` 配置如下：

```
        # MP3 files are served using AIO where supported by the OS.
        location ^~ /sites/default/files/audio/mp3 {

            location ~* ^/sites/default/files/audio/mp3/.*\.mp3$ {
```

Ogg/Vorbis 的 `location` 配置如下：

```
        # Ogg/Vorbis files are served using AIO where supported by theOS.
        location ^~ /sites/default/files/audio/ogg {

            location ~* ^/sites/default/files/audio/ogg/.*\.ogg$ {
```

这些配置项之间有以下共同点：

```
                directio 4k; # for XFS

                tcp_nopush off;
                aio on;
                output_buffers 1 2M;
            }

        }
        # Pseudo-streaming of FLV files
        location ^~ /sites/default/files/video/flv {

            location ~* ^/sites/default/files/video/flv/.*\.flv$ {

                flv;

            }

        }
```

接下来的两个伪流媒体部分也很相似。H264 文件的伪流媒体配置如下代码：

```
        # Pseudo-streaming of H264 files.
        location ^~ /sites/default/files/video/mp4 {

            location ~* ^/sites/default/files/video/mp4/.*\.(?:mp4|mov)$ {
```

对于 AAC 文件的伪流媒体配置如下代码：

```
        # Pseudo-streaming of AAC files.
        location ^~ /sites/default/files/video/m4a {

            location ~* ^/sites/default/files/video/m4a/.*\.m4a$ {
```

这些配置项之间有以下共同点：

```
                mp4;

                mp4_buffer_size         1M;

                mp4_max_buffer_size 5M;

            }

        }

        # Advanced Help module makes each module-provided
        # README available.
        location ^~ /help/ {

            location ~* ^/help/[^/]*/README\.txt$ {
                include fastcgi_private_files.conf;

                fastcgi_pass 127.0.0.1:9000;

            }
        }

        # Replicate the Apache <FilesMatch> directive of Drupal # standard
        # .htaccess. Disable access to any code files. Return a 404 to# curtail
        # information disclosure. Also hide the text files.
        location ~* ^(?:.+\.(?:htaccess|make|txt|engine|inc|info|install|module|profile|po|sh|.*sql|test|theme|tpl(?:\.php)?|xtmpl)|code-style\.pl|/Entries.*|/Repository|/Root|/Tag|/Template)$ {

            return 404;

        }

        #First we try the URI and relay to the /index.php?q=$uri&$argsif not found.
        try_files $uri @drupal;

        ## (Drupal 6) First we try the URI and relay to the /index.php?q=$no_slash_uri&$args if not found. (use only one)
        try_files $uri /index.php?q=$no_slash_uri&$args;

    } # default location ends here

    # Restrict access to the strictly necessary PHP files. Reducingthe
    # scope for exploits. Handling of PHP code and the Drupal eventloop.
    location @drupal {

        # Include the FastCGI config.
        include fastcgi_drupal.conf;

        fastcgi_pass 127.0.0.1:9000;

    }

    location @drupal-no-args {
        include fastcgi_private_files.conf;

        fastcgi_pass 127.0.0.1:9000;

    }

    ## (Drupal 6)
    ## Restrict access to the strictly necessary PHP files. Reducing# the
    ## scope for exploits. Handling of PHP code and the Drupal event # loop.
    ## (use only one)
    location = /index.php {

        # This is marked internal as a pro-active security practice.
        # No direct access to index.php is allowed; all accesses are# made
        #  by NGINX from other locations or internal redirects.
        internal;

        fastcgi_pass 127.0.0.1:9000;

    }
```

以下`locations`都设置了`return 404`以拒绝访问：

```
    # Disallow access to .git directory: return 404 as not to disclose
    # information.
    location ^~ /.git { return 404; }
    # Disallow access to patches directory.
    location ^~ /patches { return 404; }
    # Disallow access to drush backup directory.
    location ^~ /backup { return 404; }
    # Disable access logs for robots.txt.
    location = /robots.txt {

        access_log off;

    }

    # RSS feed support.
    location = /rss.xml {

        try_files $uri @drupal-no-args;
        ## (Drupal 6: use only one)
        try_files $uri /index.php?q=$uri;

    }

    # XML Sitemap support.
    location = /sitemap.xml {
        try_files $uri @drupal-no-args;

        ## (Drupal 6: use only one)
        try_files $uri /index.php?q=$uri;
    }

    # Support for favicon. Return an 1x1 transparent GIF if it doesn't
    # exist.
    location = /favicon.ico {

        expires 30d;

        try_files /favicon.ico @empty;

    }

    # Return an in-memory 1x1 transparent GIF.
    location @empty {

        expires 30d;

        empty_gif;

    }

    # Any other attempt to access PHP files returns a 404.
    location ~* ^.+\.php$ {

        return 404;

    }

} # server context ends here
```

上述提到的`include`文件为了简洁起见没有在此重复。它们可以在本节开头提到的 Perusio 的 GitHub 仓库中找到。

# 将 NGINX 和 uWSGI 连接起来

Python 的**WSGI**（**Web 服务器网关接口**）是根据 PEP-3333（[`www.python.org/dev/peps/pep-3333/`](http://www.python.org/dev/peps/pep-3333/)）正式化的接口规范。它的目的是提供一个“Web 服务器和 Python Web 应用程序或框架之间的标准接口，以促进 Web 应用程序在多种 Web 服务器上的可移植性”。由于其在 Python 社区中的广泛使用，许多其他语言也有符合 WSGI 规范的实现。虽然 uWSGI 服务器并非专为 Python 编写，但它提供了一种运行符合此规范的应用程序的方式。与 uWSGI 服务器通信的原生协议称为 uwsgi。有关 uWSGI 服务器的更多详细信息，包括安装说明、示例配置和其他支持的语言，请参阅[`projects.unbit.it/uwsgi/`](http://projects.unbit.it/uwsgi/)和[`github.com/unbit/uwsgi-docs`](https://github.com/unbit/uwsgi-docs)。

NGINX 的`uwsgi`模块可以使用类似于前一节中讨论的`fastcgi_*`指令的方式配置与此服务器通信。大多数指令的意义与它们的 FastCGI 对应指令相同，显著的区别是它们以`uwsgi_`而不是`fastcgi_`开头。然而，也有少数例外——`uwsgi_modifier1`和`uwsgi_modifier2`，以及`uwsgi_string`。前两个指令分别设置 uwsgi 包头的第一个或第二个修饰符。`uwsgi_string`允许 NGINX 向 uWSGI 或任何其他支持 eval 修饰符的 uwsgi 服务器传递任意字符串。这些修饰符特定于 uwsgi 协议。有效值及其含义的表格可以在[`uwsgi-docs.readthedocs.org/en/latest/Protocol.html`](http://uwsgi-docs.readthedocs.org/en/latest/Protocol.html)找到。

## 一个 Django 配置示例

Django（[`www.djangoproject.com/`](https://www.djangoproject.com/)）是一个 Python Web 框架，开发人员可以利用它快速创建高性能的 Web 应用程序。它已成为一个流行的框架，用于编写各种类型的 Web 应用程序。

以下配置示例展示了如何将 NGINX 连接到在 Emperor 模式下运行并启用 FastRouter 的多个 Django 应用程序。有关如何像这样运行 uWSGI 的更多信息，请参阅以下代码中的注释中嵌入的 URL：

```
http {
    # spawn a uWSGI server to connect to
    # uwsgi --master --emperor /etc/djangoapps --fastrouter 127.0.0.1:3017 --fastrouter-subscription-server 127.0.0.1:3032
    # see http://uwsgi-docs.readthedocs.org/en/latest/Emperor.html
    # and http://projects.unbit.it/uwsgi/wiki/Example
    upstream emperor {
        server 127.0.0.1:3017;
    }

    server {
        # the document root is set with a variable so that multiple # sites
        # may be served - note that all static application files are
        # expected to be found under a subdirectory "static" and all# user
        # uploaded files under a subdirectory "media"
        # see https://docs.djangoproject.com/en/dev/howto/static-files/
        root /home/www/sites/$host;

        location / {
            # CSS files are found under the "styles" subdirectory
            location ~* ^.+\.$ {
                root /home/www/sites/$host/static/styles;
                expires 30d;
            }
            # any paths not found under the document root get passed # to
            #  the Django running under uWSGI
            try_files $uri @django;
        }

        location @django {
            # $document_root needs to point to the application code
            root /home/www/apps/$host;
            # the uwsgi_params file from the nginx distribution
            include uwsgi_params;
            # referencing the upstream we defined earlier, a uWSGI # server
            #  running in Emperor mode with FastRouter
            uwsgi_param UWSGI_FASTROUTER_KEY $host;
            uwsgi_pass emperor;
        }

        # the robots.txt file is found under the "static" subdirectory
        # an exact match speeds up the processing

        location = /robots.txt {
            root /home/www/sites/$host/static;
            access_log off;
        }

        # again an exact match
        location = /favicon.ico {
            error_page 404 = @empty;
            root /home/www/sites/$host/static;
            access_log off;
            expires 30d;
        }

        # generates the empty image referenced above
        location @empty {
            empty_gif;
        }

        # if anyone tries to access a '.py' file directly,
        #  return a File Not Found code
        location ~* ^.+\.py$ {
            return 404;
        }
    }
}
```

这使得可以在不更改 NGINX 配置的情况下动态托管多个站点。

# 总结

在本章中，我们探讨了用于通过 HTTP 提供文件的若干指令。`http` 模块不仅提供了这一功能，还有许多对于 NGINX 正常运行至关重要的辅助模块。这些辅助模块默认启用。结合这些不同模块的指令，我们能够构建出满足需求的配置。我们探讨了 NGINX 如何根据请求的 URI 查找文件。我们研究了不同指令如何控制 HTTP 服务器与客户端的交互，以及如何使用 `error_page` 指令来处理多种需求。基于带宽使用、请求频率和连接数量限制访问都是可能的。

我们还看到，如何根据 IP 地址或要求认证来限制访问。我们探讨了如何使用 NGINX 的日志功能，只捕获我们想要的信息。同时，伪流媒体也进行了简要的研究。NGINX 提供了许多变量，可以用来构建我们的配置。我们还探讨了使用 `fastcgi` 模块连接 PHP-FPM 应用程序，以及使用 `uwsgi` 模块与 uWSGI 服务器进行通信的可能性。示例配置结合了本章讨论的指令，以及其他章节中讨论的部分内容。

下一章将介绍一些模块，帮助开发者将 NGINX 集成到他们的应用程序中。
