# 第四章：安装第三方模块

本章将探讨第三方模块的安装。第三方模块是由世界各地的各种开发者开发的，并托管在各种开源代码库中，例如 GitHub 和 SourceForge。其中一些模块经过了充分测试，而其他模块可能还未准备好投入生产使用。这些模块不受 Nginx 开发者的官方支持，可能在不同的 Nginx 版本中存在问题。在本章中，我们将讨论一些最著名的 Nginx 模块。有关可用选项的更完整列表，您可以在 Nginx 网站上浏览，[`wiki.nginx.org/3rdPartyModules`](http://wiki.nginx.org/3rdPartyModules)。

到目前为止，我们讨论的所有配置指令，以及在本章和接下来的章节中讨论的指令，都是在`nginx.conf`文件中指定的。`nginx.conf`文件的默认位置是`/usr/local/conf/`。

# 编译第三方模块

我们将在本章讨论的所有第三方模块都不包含在源代码中。您需要下载源代码并在编译 Nginx 时指定其位置来进行编译。您可以通过在运行`configure`时指定`--add-module`参数来实现。例如，如果您下载了位于`/opt/downloads`的模块源代码，可以使用以下代码将其编译到 Nginx 二进制文件中：

```
configure --add-module=/opt/downloads/module-folder
```

其中一些模块可能有额外的依赖关系，您需要自行解决。请参阅您正在尝试安装的模块的文档，以确保您了解即将编译的模块的后果和依赖关系。

## 与 PostgreSQL 通信（ngx_postgres）

Nginx PostgreSQL 模块目前托管在[`labs.frickle.com/nginx_ngx_postgres/`](http://labs.frickle.com/nginx_ngx_postgres/)并由 Frickle Labs 维护。它是一个上游模块，允许与 PostgreSQL 数据库直接通信。该模块的输出以名为**Resty DBD Stream**（**RDS**）的自定义二进制格式呈现。这个模块非常有用，如果您想直接将 Nginx 连接到 PostgreSQL 数据库。您可能有多个用例需要这样做。您可能想通过直接查询表中的结果来提供页面。您还可能希望将日志记录到数据库中，或者通过查询数据库表来检查某些条件。或者，您可能想从上游 PostgreSQL 数据库中验证用户。对于所有这些情况及更多，`ngx_postgres`模块都非常有用。

一个示例配置如下：

```
http {
    upstream database {
        postgres_server  127.0.0.1 dbname=test
                         user=user password=password;
    }

    server {
        location / {
            postgres_pass   database;
            postgres_query  "select * from users";
        }
    }
}
```

### 解释指令

`ngx_postgres`模块的一些重要指令如下：

#### postgres_server

`postgres_server`指令用于设置数据库服务器的详细信息。您可以指定主机名或 IP 地址，以及端口、用户名和密码。

一个示例配置如下：

```
postgres_server  127.0.0.1 dbname=test user=test password=test;
```

#### postgres_keepalive

`postgres_keepalive` 指令用于配置 `keepalive` 参数。其语法如下：

```
postgres_keepalive off | max=count [mode=single|multi] [overflow=ignore|reject]
default: max=10 mode=single overflow=ignore
```

在这里，`max` 参数决定了 `keepalive` 连接的最大数量。`mode` 参数有两个可能的值，分别是 `multi` 和 `single`。`single` 模式意味着连接池不会区分当前块中的多个 `postgres_server` 定义，并且将对所有的 `postgres_server` 定义应用相同的池，即所有的 `postgres_server` 定义共享同一个池。在 `multi` 模式下，连接池会重用具有相同服务器主机名和端口的连接。默认值是 `single`。`overflow` 选项指定了当连接池已满且需要新数据库连接时该怎么办。可以指定 `reject` 或 `ignore`。如果选择 `reject`，则会拒绝当前请求并返回 **503 服务不可用** 错误页面。如果选择 `ignore`，该模块将创建一个新的数据库连接。

#### postgres_pass

`postgres_pass` 指令保存包含 PostgreSQL 连接配置的上游块名称。它还可以包含变量。

#### postgres_query

`postgres_query` 指令用于指定 PostgreSQL 查询。如果指定了 `GET`、`POST`、`PUT` 或 `DELETE` 等 HTTP 方法，则查询仅适用于指定的方法；否则，它将在所有方法中运行。查询可以包含变量，并且你可以在一个位置指定多个查询指令。一个示例配置如下：

```
postgres_query    GET POST  "SELECT * FROM employees";
```

#### postgres_rewrite

`postgres_rewrite` 指令应在满足某个条件时发送特定的响应代码。条件可以是以下之一：

+   `no_changes`：这是指查询没有影响任何行时的条件

+   `changes`：这是指查询至少影响一行时的条件

+   `no_rows`：这是指结果集中没有返回任何行时的条件

+   `rows`：这是指查询返回至少一行时的条件

如果你想将原始响应体发送到客户端，可以在代码前加上 `=`，如以下示例配置所示：

```
postgres_rewrite  no_rows =403;
```

#### postgres_output

`postgres_output` 指令决定响应的输出类型。可能的值包括 `rds`、`text`、`binary`、`value` 和 `none`。当你不希望有任何输出时，使用 `none` 值。`value` 用于当你希望以文本格式输出单个值时。所有响应类型都会设置适当的 HTTP 头。

#### postgres_set

`postgres_set` 指令用于从结果集中设置一个变量的值。你可以指定行和列来选择值。一个示例配置如下：

```
postgres_set $empname 00 required
```

如果你将该指令设置为 `required`，则当要设置的值为 null 或超出范围时，模块将生成 `500 内部服务器错误`。

#### postgres_escape

`postgres_escape`指令将转义并引用`$unquoted`变量中的值，并将结果存储在`$escaped`变量中，这样可以在 SQL 查询中安全使用。以下是一个示例配置：

```
postgres_escape   $user $remote_user;
postgres_escape   $pass $remote_passwd;
```

#### postgres_connect_timeout

`postgres_connect_timeout`指令设置连接数据库的超时时间。

#### postgres_result_timeout

`postgres_result_timeout`指令设置从数据库接收结果的超时时间。

## 与 MySQL 和 drizzle 进行通信（drizzle-nginx）

`drizzle-nginx`模块是一个与 MySQL 或 drizzle 服务器通信的上游模块。Drizzle 是 MySQL 的一个分支，经过优化以支持多核处理和可扩展性。此模块实质上是将`libdrizzle`集成到 Nginx 模块中。与 Nginx 的 PostgreSQL 模块类似，这个模块不会产生可读的文本输出，而是生成 Resty DB 格式，这是一种自定义的二进制格式。

你可以从 GitHub 仓库下载此模块的源代码，网址为[`github.com/chaoslawful/drizzle-nginx-module`](https://github.com/chaoslawful/drizzle-nginx-module)。请注意，成功编译此模块前，你需要安装 drizzle 和 libdrizzle。你可以从 launchpad 下载 drizzle，网址为[`launchpad.net/drizzle`](https://launchpad.net/drizzle)。

如果你希望直接将 Nginx 与 MySQL 数据库连接，这个模块会非常有用。你可能有多个原因想要这样做。例如，你可能希望通过直接查询表中的数据来提供页面。你还可能想将日志记录到数据库中，或通过查询数据库表来检查某些条件。或者，你可能希望通过上游 MySQL 数据库对用户进行身份验证。在所有这些情况下，这个模块都会很有帮助。

### 解释指令

`drizzle-nginx`模块最重要的指令如下：

#### drizzle_server

我们使用`drizzle_server`指令以 IP 地址或域名的形式指定 drizzle 服务器的名称，并可以选择性地指定端口。默认端口号是`3306`。你还可以指定用户名和密码。此指令支持以下选项：

+   `user=`: 此选项定义数据库登录的用户名。

+   `password=`: 此选项定义数据库密码，特殊字符可以选择性地用引号括起来，如下所示的示例配置所示：

    ```
    drizzle_server 127.0.0.1:3306 user=user "password=1 2 3"
            dbname=mysql protocol=mysql;
    ```

+   `dbname=`: 此选项定义默认连接所使用的数据库。

+   `protocol=`: 此选项定义目标数据库类型，可以是`drizzle`或`mysql`（默认值为`drizzle`）。

+   `charset=`: 此选项用于明确指定 MySQL 连接的字符集，如下所示的示例配置所示：

    ```
    drizzle_server localhost:3306 user=mysqluser password=passwd
                                      dbname=mydb charset=utf8;
    ```

#### drizzle_keepalive

`drizzle_keepalive`指令用于为目标数据库维护一个`keepalive`池。此指令支持以下选项：

+   `max=`：此选项默认为 `0`，意味着禁用 `keepalive` 连接池。要启用连接池，必须将此值设置为大于 0 的数值。

+   `mode=`：此参数的可选值为 `multi` 和 `single`。`single` 模式表示连接池不会区分当前块中的多个 `drizzle_server` 定义，池将应用于所有这些定义，也就是说，你有一个池供所有 `drizzle_server` 定义使用。在 `multi` 模式中，池将重用具有相同服务器主机名和端口的连接。默认值为 `single`。

+   `overflow=`：此选项指定当连接池已满，而需要新的数据库连接时应该怎么做。可以指定 `reject` 或 `ignore`。如果选择 `reject`，则会拒绝当前请求并返回 **503 服务不可用** 错误页面；对于 `ignore`，该模块将创建一个新的数据库连接。

#### drizzle_query

`drizzle_query` 指令定义了要在数据库后台运行的 SQL 查询。

你可以使用 Nginx 变量来替代查询，但必须小心 SQL 注入攻击。因此，建议你适当地清理和加引号 SQL 查询。示例配置如下：

```
location /employees {
    set_unescape_uri $name $arg_name;
    set_quote_sql_str $quoted_name $name;

    drizzle_query "select * from empl where name = $quoted_name";
    drizzle_pass my_backend;
}
```

#### drizzle_pass

使用 `drizzle_pass` 指令，你可以将当前的位置传递给另一个定义的 MySQL 或 drizzle-upstream 块。

你可以使用 Nginx 变量作为值进行动态传递。示例配置如下：

```
upstream backend { localhost:3306 dbname=mydb; }

server {
    location /emp {
        set $srv backend;

        drizzle_query ...;
        drizzle_pass $srv;
    }
 }
```

#### drizzle_connect_timeout

`drizzle_connect_timeout` 指令指定了连接到远程服务器的超时时间值。该值可以是一个整数，后面可以附加可选的时间单位，例如 s（秒）、ms（毫秒）或 m（分钟）。默认时间单位为 s，默认值为 `60 s`。

#### drizzle_send_query_timeout

`drizzle_send_query_timeout` 指令指定了向远程服务器发送 SQL 查询的超时时间值。该值可以是一个整数，后面可以附加可选的时间单位，例如 s（秒）、ms（毫秒）或 m（分钟）。默认时间单位为 s，默认值为 `60 s`。

#### drizzle_recv_cols_timeout

`drizzle_recv_cols_timeout` 指令指定了接收结果集列元数据到远程服务器的超时时间值。该值可以是一个整数，后面可以附加可选的时间单位，例如 s（秒）、ms（毫秒）或 m（分钟）。默认时间单位为 s，默认值为 `60 s`。

#### drizzle_recv_rows_timeout

`drizzle_recv_rows_timeout` 指令指定了接收结果集行数据（如果有的话）到远程服务器的超时时间值。该值可以是一个整数，后面可以附加可选的时间单位，例如 s（秒）、ms（毫秒）或 m（分钟）。默认时间单位为 `s`，默认值为 `60 s`。

#### drizzle_buffer_size

`drizzle_buffer_size`指令指定服务器输出的缓冲区大小。此指令的默认值取决于操作系统的页面大小，通常为 4K/8K。较大的缓冲区大小可以减少网络开销。然而，你需要通过实验找到适合你工作负载的正确值。

#### drizzle_module_header

`drizzle_module_header`指令控制是否在响应中输出 drizzle 头部。默认情况下，启用发送头部。可以使用以下脚本配置此指令：

```
X-Resty-DBD-Module: ngx_drizzle 0.1.0
```

## 摘要认证（ngx_http_auth_digest）

在今天的互联网环境中，HTTP 基本认证过于简单，不能为现代 Web 服务器提供足够的安全性。原因是，用户名和密码是明文传输的，除非你使用 HTTPS。`ngx_http_auth_digest`模块可以基于 RFC 2617 使用 HTTP 摘要认证来保护你的资源。

摘要认证模块已工作并且被认为相当稳定。然而，它可能没有在实际环境中进行足够的测试，因此确保它在你的环境中有效非常重要。由于这个模块涉及安全性，彻底测试软件总是一个好主意。

你可以在[`github.com/samizdatco/nginx-http-auth-digest`](https://github.com/samizdatco/nginx-http-auth-digest)下载源代码。

你可以通过将以下代码行添加到 Nginx 配置文件中的服务器部分来为目录树设置密码保护：

```
auth_digest_user_file /opt/passwd.digest;
location /members{
  auth_digest 'members area; # set the realm for this location block
}
```

当前，摘要认证模块与通过 `htdigest` 脚本生成的文件配合使用。`htdigest` 脚本可以在你的 Apache 安装或源代码中找到。此模块的源代码中也有一个 `htdigest.py` 脚本，帮助你生成兼容的文件。

### 解释指令

`ngx_http_auth_digest`模块中一些最重要的指令如下：

#### auth_digest

`auth_digest`指令可以在服务器和位置的上下文中定义。此参数定义认证的领域名称。该名称应与创建 `htdigest` 文件时使用的名称相匹配。要选择性地禁用认证，可以将 `auth_digest` 设置为 `off`。此指令的默认值为 `off`。

#### auth_digest_user_file

`auth_digest_user_file`指令可以在服务器和位置的上下文中定义。此指令用于指定密码文件的名称。密码文件应通过 Apache 的 htdigest 命令（或包含的 `htdigest.py` 脚本）创建。

#### auth_digest_timeout

`auth_digest_timeout`指令可以在服务器和位置的上下文中定义。此超时值定义了发送给客户端的挑战的过期时间。如果用户未在此时间内提供响应，则挑战被视为过期，并且在资源再次请求或响应来自客户端时，系统将向客户端发送一个新挑战。默认的超时值为 `60 s`。

#### auth_digest_expires

`auth_digest_expires`指令可以在服务器和位置的上下文中定义。此参数用于定义 nonce 值的过期时间。一旦客户端成功认证，nonce 值会被缓存，后续的请求将使用缓存的值。此参数定义了客户端可以继续使用 nonce 值的时长。默认的摘要过期值是`10 s`。

#### auth_digest_replays

`auth_digest_replays`指令可以在服务器和位置的上下文中定义。可以使用此指令根据请求的数量而非时间来指定缓存的 nonce 的有效性。较大的值将增加共享内存的需求。默认值为每个 nonce 20 次重放。

#### auth_digest_shm_size

`auth_digest_shm_size`指令只能在服务器的上下文中定义。该指令指定用于存储有关活动认证请求的信息的固定大小内存缓存。一旦缓存满了，直到活动会话过期之前将无法进行进一步的认证。默认大小约为 4 MB。默认值允许每 70 秒大约处理 82,000 个非重放请求。

一个示例配置如下：

```
auth_digest_user_file /opt/htdigest;
auth_digest_shm_size 4m;   # the storage space allocated for tracking active sessions

location /restricted {
  auth_digest 'this is a restricted location';
  auth_digest_timeout 60s;
  auth_digest_expires 10s;
  auth_digest_replays 20;
}

location / {
  auth_digest 'restricted';
  location /img {
    auth_digest off; # this location will be accessible  }
}
```

## 加速网页（ngx_pagespeed）

`ngx_pagespeed`模块优化网页及相关资源，以减少延迟和带宽。它能够重写 HTML 页面并自动消除降低网站或网页性能的缺陷。此模块由 Google 编写，类似于 Apache 的`mod_pagespeed`模块。

此模块通过自动应用网页性能最佳实践来减少页面的加载时间，优化页面及相关资产（CSS、JavaScript 和图像）。它可以执行以下类型的优化：

+   图像优化

+   CSS 和 JavaScript 优化

+   资源内联

+   HTML 重写

+   缓存生命周期延长

为了启用该模块，您必须在服务器或 HTTP 块中加入`pagespeed On`。此外，您还应该定义`FileCache`位置并指定要启用的重写过滤器。以下是一个示例配置：

```
http {
  pagespeed On;
  pagespeed FileCachePath "/var/cache/ngx_pagespeed/";
  pagespeed EnableFilters combine_css,combine_javascript, add_instrumentation;
  ...
  ...
}
```

`FileCachePath`参数提供重写文件缓存的位置，应该是一个有效路径。`EnableFilters`参数定义了特定位置将启用的优化。

### 配置处理程序

当配置并启用`ngx_pagespeed`模块时，默认处理程序会自动创建，但还有其他处理程序可以更详细地监控该模块的活动，具体如下：

+   **统计处理程序**：此处理程序显示与页面或资源优化相关的统计信息，包括到目前为止已优化的页面，以及各种延迟和缓存有效性指标。您还可以查看当前激活配置的摘要。

+   **消息处理器**：如果你启用了并为`MessageBufferSize`参数指定了大小，则此处理器将包含来自`pagespeed`的服务器范围内的最近日志输出历史记录，包括那些根据日志级别从服务器日志文件中省略的消息。

+   **控制台处理器**：这个处理器显示了问题指标随时间变化的图表。

+   **信标处理器**：这个处理器可以被`add_instrumentation`过滤器用来报告页面加载时间，你可以通过统计页面查看这些数据。

以下是模块文档页面中的一个示例配置：

```
pagespeed on;

# Needs to exist and be writable by nginx.
pagespeed FileCachePath /var/ngx_pagespeed_cache;

# Ensure requests for pagespeed optimized resources go to the pagespeed handler
# and no extraneous headers get set.
location ~ "\.pagespeed\.([a-z]\.)?[a-z]{2}\.[^.]{10}\.[^.]+" {
  add_header "" "";
}
location ~ "^/ngx_pagespeed_static/" { }
location ~ "^/ngx_pagespeed_beacon$" { }
location /ngx_pagespeed_statistics { allow 127.0.0.1; deny all; }
# Recent log messages. Like statistics, these are generally not to be shown to the public, so this has access controls as well.
pagespeed MessageBufferSize 100000;
location /ngx_pagespeed_message { allow 127.0.0.1; deny all; }
```

为了检查模块是否正在处理你的页面，你可以检查页面的源代码，并通过以下代码行查看`X-Page-Speed`头部：

```
$ curl -I 'http://localhost /index.html/'  |  grep X-Page-Speed
X-Page-Speed: 1.6.29.5-...
```

你可以在[`developers.google.com/speed/pagespeed/module/using`](https://developers.google.com/speed/pagespeed/module/using)的在线文档中找到完整的 PageSpeed 过滤器列表。

## Lua 脚本（ngx_lua）

如果你希望在 Nginx 配置文件中编写脚本，那么通过使用`ngx_lua`模块来利用 Lua 的强大功能可能是一个不错的选择。这个模块非常强大，有着广泛的应用，能够为你提供在 Nginx 配置内的完整编程能力。它具有以下优点和特点：

+   这将允许你在请求执行之前对其进行复杂的处理，或者在请求处理后更改响应。

+   你可以添加新的头部，或者移除现有的头部。

+   你可以根据复杂的程序逻辑执行重定向和路由操作。

+   你可以完全基于 Lua 脚本创建一个复杂的日志记录框架。

+   你可以阻止或允许特定的 IP 地址。

+   你可以在此基础上构建自己的认证或预处理层，而无需编写自己的 C 模块或重新编译 Nginx 代码。

Lua 是一种轻量级、可嵌入的脚本语言，非常适合在配置文件中进行脚本编写。

该模块允许你在 Nginx 请求处理的不同阶段运行 Lua 代码。在深入了解 Lua 脚本之前，值得先看看 Nginx 请求处理的不同阶段。

每个由 Nginx 处理的请求都会经历以下阶段：

| 序号 | Nginx 请求处理阶段 | 描述 |
| --- | --- | --- |
| 1 | 服务器选择 | 根据请求选择一个服务器块。 |
| 2 | 请求读取后 | 这个阶段在请求读取后执行，允许你在请求处理之前对其进行操作。例如，`HttpRealIpModule`可以使用此阶段在请求头中添加 IP 地址。 |
| 3 | 服务器重写 | 在此阶段，可以进行 URL 重写。你可以基于变量值选择配置。`HttpRewrite`模块允许你这样做。 |
| 4 | 位置选择 | 在此阶段，根据请求的 URL 选择或匹配一个位置配置块。 |
| 5 | 位置重写 | 此阶段允许你在选定的配置块内进行重写。 |
| 6 | 预访问 | 此阶段允许你进行某些过滤，即限制每个会话的请求次数。 |
| 7 | 访问 | 此阶段运行身份验证，例如`auth_basic`或`auth_digest`。你还可以根据一些条件（如 IP 地址）允许或拒绝请求。 |
| 8 | 尝试文件 | 核心模块的`try_files`指令在此阶段执行。 |
| 9 | 内容 | 实际的内容生成发生在此阶段。所有上游模块都在此阶段执行。 |
| 10 | 日志 | 在此阶段，信息会记录到日志文件中。像`access_log`这样的模块在此阶段运行。 |
| 11 | 后操作 | 在此阶段，核心模块的`post_action`指令会执行，它允许你在请求完成后向某个位置或上游发送子请求，例如，将完成的请求记录到远程 MySQL 数据库中。 |

`nginx_lua`模块通过标准的 Lua 解释器或 LuaJIT 将 Lua 嵌入到 Nginx 中。请注意，你需要先安装 Lua 或 LuaJIT，才能使用此模块。此模块还依赖于另一个名为`ngx_devel_kit`的 Nginx 模块，它促进了新 Nginx 模块的开发。我们将在本章以及第五章，*创建自己的模块*中详细了解该模块，在那里我们将学习如何编写自己的 Nginx 模块。

使用 Nginx 的 Lua API，你可以在 Lua 脚本中以非阻塞的方式与上游服务器通信。Lua 虚拟机在所有由单个 Nginx 工作进程处理的请求之间共享，以最小化内存使用。

可以与`nginx_lua`模块一起使用许多上游 Nginx 模块。这些模块如下：

+   `lua-resty-memcached`

+   `lua-resty-mysql`

+   `lua-resty-redis`

+   `lua-resty-dns`

+   `lua-resty-upload`

+   `ngx_memc`

+   `ngx_postgres`

+   `ngx_redis2`

+   `ngx_redis`

+   `ngx_proxy`

+   `ngx_fastcgi`

`ngx_lua`模块的一个示例配置如下：

```
  # set search paths for pure Lua external libraries (';;' is the default path):
    lua_package_path '/home/user/?.lua;/scripts/?.lua;;';

    # set search paths for Lua external libraries written in C (can also use ';;'):
    lua_package_cpath '/bar/for/?.so;/blah/blah/?.so;;';

    server {
        location /inline_concat {
            # MIME type determined by default_type:
            default_type 'text/plain';
            set $a "hello";
            set $b "world";
            # inline Lua script
            set_by_lua $res "return ngx.arg[1]..ngx.arg[2]" $a $b;
            echo $res;
        }
```

### 解释指令

`ngx_lua`的一些重要指令如下：

#### lua_package_path

`lua_package_path`指令用于指定 Lua 脚本的路径。此值被`set_by_lua`和`content_by_lua`等指令使用。

你可以在搜索路径字符串中使用特殊符号$prefix 或${prefix}来表示服务器前缀路径，通常该路径是通过启动 Nginx 服务器时的`-p PATH`命令行选项来确定的。

默认值来自`LUA_PATH`环境变量。如果此变量未定义，则使用默认搜索路径来定位 Lua 脚本。

#### set_by_lua 或 set_by_lua_file

`set_by_lua` 或 `set_by_lua_file` 指令用于执行一个小型的嵌入式且阻塞的 Lua 脚本，该脚本作为字符串提供。该脚本可以接受两个输入参数，并通过 `return` 变量返回结果。当执行该代码时，Nginx 事件循环会被阻塞。因此，不应使用此指令执行长时间运行的代码。

请注意，以下 API 函数在此上下文中当前被禁用。此指令一次只能将一个值写入单个 Nginx 变量，如以下代码片段所示：

```
location /testlua {
    set_by_lua $sum '
        local a = 32
        local b = 56
        return a + b;          -- return the $sum value normally
    ';

    echo "sum = $sum
}
```

此指令可以与 `HttpRewriteModule`、`HttpSetMiscModule` 和 `HttpArrayVarModule` 模块的所有指令自由混用。所有这些指令将按照它们在配置文件中出现的顺序执行。以下是一个示例配置：

```
set $num 32;
set_by_lua $num2 'tonumber(ngx.var.num) + 1';
```

你可以在此指令中提供的 Lua 脚本中自由使用 `$` 符号，因为 Nginx 变量插值已被禁用。此指令需要 `ngx_devel_kit` 模块。

`set_by_lua_file` 指令类似于 `set_by_lua` 指令。唯一的区别是，这里提供的 Lua 脚本是一个文件。该文件也可以包含 Lua 或 LuaJIT 字节码，而不仅仅是文本脚本。

当给出诸如 `path/file.lua` 这样的相对路径时，它将转换为相对于通过 `-p PATH` 命令行选项在启动 Nginx 服务器时确定的服务器前缀路径的绝对路径。

默认情况下，Lua 代码缓存是启用的。这意味着脚本文件会在第一次加载时被加载。如果你进行更改，Nginx 配置将会重新加载。如果你处于开发周期中，可以通过在配置文件中使用 `lua_code_cache_off` 参数来关闭代码缓存。以下是一个示例配置：

```
    location /rel_file_concat {
        set $a "foo";
        set $b "bar";
        # script path relative to nginx prefix
        # $ngx_prefix/conf/concat.lua contents:
        #
        #    return ngx.arg[1]..ngx.arg[2]
        #
        set_by_lua_file $res conf/concat.lua $a $b;
        echo $res;
    }
```

#### `content_by_lua` 或 `content_by_lua_file`

`content_by_lua` 或 `content_by_lua_file` 指令用于指定在内容阶段为每个请求执行的 Lua 脚本。你可以在该脚本中使用 API 调用，且该脚本会在独立的全局沙盒中执行。

由于该指令是一个内容处理器，避免在同一位置使用它和其他内容处理器指令。例如，`proxy_pass` 指令和此指令不应在同一位置使用。

`content_by_lua_file` 指令与 `content_by_lua` 等效，不同之处在于此指令要求你提供一个 Lua 脚本文件的路径，而不是编写内联代码。该文件中的代码如果启用了代码缓存，将仅加载一次，并且相对路径会解析为相对于服务器前缀的绝对路径。以下是一个示例配置：

```
    location /request_body {
         # force reading request body (default off)
         lua_need_request_body on;
         client_max_body_size 50k;
         client_body_buffer_size 50k;

         content_by_lua 'ngx.print(ngx.var.request_body)';
    }
    # transparent non-blocking I/O in Lua via subrequests
    location /lua {
        # MIME type determined by default_type:
        default_type 'text/plain';

        content_by_lua '
            local res = ngx.location.capture("/some_other_location")
            if res.status == 200 then
                ngx.print(res.body)
            end';
    }
```

#### `rewrite_by_lua` 或 `rewrite_by_lua_file`

`rewrite_by_lua` 或 `rewrite_by_lua_file` 指令在重写阶段执行 Lua 代码。Lua 代码可以使用 API 调用，并在一个独立的全局沙盒中运行。

请注意，该处理程序总是在标准的 HTTP 重写之后运行。因此，以下代码将无法按预期工作：

```
  location /foo {
      set $a 5; # create and initialize $a
      set $b 13; # create and initialize $b
      rewrite_by_lua 'ngx.var.b = tonumber(ngx.var.a) + 1';
      if ($b = '6') {
         rewrite ^ /bar redirect;
         break;
      }
      echo "res = $b";
  }
```

这是因为 `if` 条件会在 `rewrite_by_lua` 之前运行，即使它在配置脚本中位于 `rewrite_by_lua` 之后。

正确的方法是通过使用 Nginx API 调用，方式如下：

```
location /foo {
    set $a 5; # create and initialize $a
    set $b 13; # create and initialize $b
    rewrite_by_lua '
        ngx.var.b = tonumber(ngx.var.a) + 1
        if tonumber(ngx.var.b) == 6 then
            return ngx.redirect("/bar");
        end
    ';

    echo "res = $b";
}
```

`rewrite_by_lua` 代码将在重写请求处理阶段的最后运行，除非将 `rewrite_by_lua_no_postpone` 设置为 `on`。

`rewrite_by_lua_file` 指令也会在标准的 HTTP 重写阶段后运行。然而，代码是从 Lua 脚本文件或字节码文件中执行的，配置脚本如下所示：

```
    location /script {
            content_by_lua_file /path/to/script/$1.lua;
    }
```

#### access_by_lua 或 access_by_lua_file

`access_by_lua` 或 `access_by_lua_file` 指令在访问阶段执行 Lua 代码。这意味着该指令中的代码会在每次请求时运行，且没有子请求能够触发该代码。

Lua 代码会在标准的 `HttpAccessModule` 之后运行。因此，如果有任何黑名单 IP，它们将在执行此代码之前被拒绝。

你可以使用这些指令来实现更复杂的访问机制，也就是与上游服务器通信的机制，例如数据库。

现在让我们来看一个示例 ngx_lua 配置，来理解 `access_by_lua` 的使用：

```

location / {
        access_by_lua '
            local ret = ngx.location.capture("/ldap_auth")

            if ret.status == ngx.HTTP_OK then
                return
            end

            if ret.status == ngx.HTTP_FORBIDDEN then
                ngx.exit(ret.status)
            end

            ngx.exit(ngx.HTTP_INTERNAL_SERVER_ERROR)
        ';
    }
        access_by_lua '
               if ngx.var.remote_addr == "10.11.60.220" then
                   ngx.exit(ngx.HTTP_FORBIDDEN)
               end ';  
```

在上述示例配置中，Lua 代码会运行名为 `ldap_auth` 的配置，该配置将用户与 LDAP 服务器进行身份验证，并根据返回值，决定请求是正常返回还是以适当的错误代码（403）退出。

`access_by_lua` 指令允许你通过文件运行 Lua 脚本或字节码。你需要在指令中指定脚本文件的路径。

可以在文件中使用 Nginx 变量来提供灵活性。然而，这也带来了一些风险，通常不推荐这样做。

相对文件路径会使用服务器前缀转换为绝对路径。

建议在生产环境中开启代码缓存，以便 Lua 代码仅加载一次，从而提升性能。然而，在开发环境中，应该关闭代码缓存，以避免每次代码更改时都重载服务器。

`ngx_lua` 模块提供了完整的脚本能力，同时也提供了非常高的性能。这尤其适用于使用 LuaJIT 进行即时编译（**JIT**）。这使得该模块可以在非常广泛的用例中派上用场。

## 使用 GeoIP 模块（ngx_http_geoip_module）进行反向 IP 查找

`ngx_http_geoip_module` 使用 MaxMind IP 数据库对客户端的 IP 进行反向查找。它将 IP 地址解析到来源地并为变量设置一个值。

此模块默认未构建；应通过`--with-http_geoip_module`配置参数启用。如前所述，模块依赖于 MaxMind GeoIP 库。

您需要拥有 MaxMind 的账户，并且需要下载几个数据库文件，将 IP 地址映射到国家、城市甚至组织。

除了为您提供有关客户端的更多信息外，这个模块的一个关键应用还可以防御 DDoS 攻击。通过该模块查询的信息，您可以阻止或允许来自不同国家、城市、地区等的流量。这虽然有点粗糙，但它有效。您可以将此模块作为`HttpLimitReqModule`和`HttpLimitZoneModule`的补充。以下是一个示例配置：

```
http {
    geoip_country         countries.dat;
    geoip_city            city.dat;
    geoip_org             org.dat
    geoip_proxy           10.220.136.0/24;
    geoip_proxy           2331:0fb9::/32;
    geoip_proxy_recursive on;
    ...
```

### 解释指令

以下是您可以用于配置此模块的指令列表：

#### geoip_country

`geoip_country`指令允许您指定包含国家 IP 查询信息的数据库文件的名称和路径。在使用此数据库时，以下变量可用（并且由此模块设置）：

+   `$geoip_country_code`：这是一个两位字母的国家代码，例如，DE 或 US。这些代码符合 ISO 3166-1 alpha-2 标准。

+   `$geoip_country_code3`：这是一个三位字母的国家代码，例如，DEU 或 USA。这些代码符合 ISO 3166-1 alpha-3 标准。

+   `$geoip_country_name`：这是完整的国家名称，例如，俄罗斯联邦或美国。

#### geoip_city

`geoip_city`指令允许您设置用于根据客户端 IP 地址查询城市和地区信息的数据库文件的名称和路径。在使用此数据库时，以下变量也可用并已设置：

+   `$geoip_area_code`：这是与客户端 IP 地址相关的电话区号（仅限美国）。MaxMind 数据库中的此字段已被弃用，因此您可能无法获取任何信息或获取过时的信息。

+   `$geoip_city_continent_code`：这是一个两位字母的洲代码，例如，EU 或 AS。

+   `$geoip_city_country_code`：这是一个两位字母的国家代码，例如，DE 或 US。这些代码符合 ISO 3166-1 alpha-2 标准。

+   `$geoip_city_country_code3`：这是一个三位字母的国家代码，例如，DEU 或 USA。这些代码符合 ISO 3166-1 alpha-3 标准。

+   `$geoip_city_country_name`：这是国家名称，例如，俄罗斯联邦或美国。

+   `$geoip_dma_code`：这是美国的 DMA 区域代码（也称为地铁代码），可以在[`developers.google.com/adwords/api/docs/appendix/cities-DMAregions`](https://developers.google.com/adwords/api/docs/appendix/cities-DMAregions)找到。

+   `$geoip_latitude`：这是城市的纬度值。

+   `$geoip_longitude`：这是城市的经度值。

+   `$geoip_region`：这是一个两字符的国家区域代码（区域、领土、州、省、联邦土地等），例如 48 或 DC。

+   `$geoip_region_name`：此项提供国家的区域名称（区域、领土、州、省、联邦土地等），例如巴伐利亚或哥伦比亚特区。

+   `$geoip_city`：此项提供完整的城市名称，例如慕尼黑或伦敦。

+   `$geoip_postal_code`：如果可用，此项提供邮政编码信息。

#### geoip_org

`geoip_org`指令允许您指定数据库文件的名称和路径，用于将 IP 地址解析为组织名称。这可以是公司名称或机构名称。通常，这类信息可以通过`whois`数据库获取。使用此选项时，还可以使用以下变量：

+   `$geoip_org`：此项包含组织名称，例如 Facebook 公司。

#### geoip_proxy

`geoip_proxy`指令允许您指定“信任”的代理服务器的 IP 地址或 CIDR。如果客户端 IP 地址与此信任的地址匹配，则会使用 HTTP 头部`X-Forwarded-For`中发送的 IP 地址进行 IP 查找。`X-Forwarded-For`头部是代理服务器发送的标准头部，用于揭示客户端的真实 IP 地址。如果代理服务器选择不这么做，它实际上充当了一个匿名化工具。此头部中发送的 IP 地址的正确性完全取决于代理服务器；因此，如果您信任特定的代理服务器发送正确的信息，您可以使用此指令启用对`X-Forwarded-For`头部中发送的 IP 地址的查找。

#### geoip_proxy_recursive

`geoip_proxy_recursive`指令允许您启用递归 IP 查找。如果启用了递归查找，则`X-Forwarded-For`头部中发送的最后一个不受信任的地址将用于 IP 查找。

如果 Nginx 在代理后面工作，您还可以使用`HttpRealIpModule`。此模块允许您将客户端的 IP 地址更改为请求头中的某个值（例如`X-Real-IP`或`X-Forwarded-For`）。

## 进行健康检查

在这里，我们将学习如何通过各种模块跟踪健康的上游服务器。

### ngx_http_healthcheck_module

如果您的 Nginx 服务器与许多上游服务器一起工作，提供各种服务和内容，那么跟踪哪些上游服务器仍然健康和工作非常重要，尤其是当它们是第三方或外部服务器时。此模块允许您跟踪健康的后端。

它的工作原理是：当上游服务器响应一个 200+的状态码时，并且响应可选地返回一个正文，则标记为“良好”；否则，标记为“不良”。此模块还有一个 HTTP 健康检查页面，您可以查看当前的后端状态。这与`haproxy`或`varnish`健康检查非常相似。

本模块将健康检查事件插入到 Nginx 的事件树中。当该事件触发时，它会与后端建立对等连接，并发送和接收数据。当健康检查完成或超时后，它会在共享内存区域更新后端的健康状态。以下是示例配置：

```
http {

  upstream check_upstreams {
    server server1.com;
    server server2.com;
    healthcheck_enabled;
    healthcheck_delay 1000;
    healthcheck_timeout 1000;
    healthcheck_failcount 1;
    healthcheck_expected 'BACKEND_ALIVE';
    healthcheck_send "GET /health HTTP/1.0" 'Host: www.websitename.com';
 }
...
 location /health_status {
      healthcheck_status;
 }
...
}
```

### 解释指令

`ngx_http_healthcheck_module`的一些重要指令如下：

#### healthcheck_enabled

`healthcheck_enabled`模块的上下文是 upstream，并启用对特定 upstream 块中定义的上游服务器的健康检查。在前面的示例中，上游服务器是`server1`和`server2`。

#### healthcheck_delay

对于每个上游服务器，`healthcheck_delay`指令定义了两次健康检查之间的延迟时间。默认值为`1000 ms`。

#### healthcheck_timeout

`healthcheck_timeout`指令定义了健康检查操作的超时值。如果健康检查操作由于后端响应慢而花费过长时间，则在超时后该操作将停止。默认值为`2000 ms`。

#### healthcheck_failcount

`healthcheck_failcount`指令定义了需要连续多少次健康检查结果为好或坏，才能切换当前的健康状态（从好到坏或从坏到好）。默认值为`2`。

#### healthcheck_send

`healthcheck_send`指令是一个必需指令，允许你决定发送什么内容来进行健康检查。这可以是一个简单的 HTTP `GET`命令或更复杂的命令。每个参数后面跟随`\r\n`，整个块末尾再加上一个`\r\n`。以下是示例配置：

```
  healthcheck_send 'GET /health HTTP/1.0'
   'Host: www.yourhost.com';
```

请注意，你可能希望以某个指令结束健康检查，该指令会关闭连接，例如，`Connection: close`。

#### healthcheck_expected

`healthcheck_expected`指令允许你指定期望从上游服务器返回的响应，以将其标记为健康。如果返回任何其他响应或没有响应，将把该主机标记为不可用。此指令仅指 HTTP 体中的响应，而不是响应头。如果缺少此指令，则简单的 200 响应码足够。

#### healthcheck_buffer

`healthcheck_buffer`指令定义了用于临时存储后端响应以供检查的缓冲区大小。确保你分配足够的内存，不仅用于响应体，还包括你预期接收的响应头。

### 负载均衡

有许多第三方 Nginx 模块可以使用，它们允许你基于哈希机制或最少负载原则在上游服务器之间分配负载。负载均衡有多种哈希机制，其中一些通过第三方模块可用。在这里，我们将简要了解一些可用的选项。

#### 一致性哈希

`ngx_http_upstream_consistent_hash` 模块允许你使用一致性哈希环进行负载均衡。一致性哈希是一种特殊的哈希算法，当你需要频繁重哈希时特别有效，比如当一个新机器或服务器被添加到池中或从池中移除时。

该模块与 `php-memcached` 模块兼容，你可以将值存储在该模块可以读取的 `memcached` 集群中。你可以在 [`wiki.nginx.org/HttpUpstreamConsistentHash`](http://wiki.nginx.org/HttpUpstreamConsistentHash) 找到更多关于此模块的详细信息。

还有另一个类似的模块，它使用 Ketama 一致性哈希库来计算配置变量的哈希值，即请求 URI。有关更多信息，请访问 [`wiki.nginx.org/HttpUpstreamKetamaCHashModule`](http://wiki.nginx.org/HttpUpstreamKetamaCHashModule)。

#### 最不忙

`ngx_http_upstream_fair_module` 模块允许你根据哪个上游最不忙来进行负载均衡。

该模块还提供了一个状态页面，你可以在此页面查看负载均衡的当前状态。

该模块使用**加权最小连接轮询**（**WLC-RR**）算法，并提供了多种可能的变体。更多关于此模块的信息，请访问 [`wiki.nginx.org/HttpUpstreamFairModule`](http://wiki.nginx.org/HttpUpstreamFairModule)。

#### 配置变量哈希

配置变量哈希可能是你能做的最随机的哈希。你可以选择对可用的某个变量进行哈希，即 `$request_uri` 或 HTTP 头部，或者两者的组合。该模块使用 CRC-32 算法对指定变量进行哈希计算。

更多信息请访问 [`wiki.nginx.org/HttpUpstreamRequestHashModule`](http://wiki.nginx.org/HttpUpstreamRequestHashModule)。

# 总结

在本章中，我们查看了各种有用的第三方 Nginx 模块，这些模块默认不随源代码分发。还有许多其他有用的模块可供使用，你可以在 GitHub 上找到它们。请注意，确保模块足够稳定以用于生产环境。务必先进行一些测试，然后再将模块谨慎地应用到生产环境中。Nginx 社区不对使用这些模块可能遇到的任何问题承担责任。

在下一章中，我们将讨论如何开发我们自己的 Nginx 模块，这将是进入自定义模块开发世界的第一步。
