# 第三章 安装和配置 HTTP 模块

在本章中，我们将探讨标准 HTTP 模块的安装与配置。标准 HTTP 模块默认内置于 Nginx 中，除非在运行配置脚本时明确禁用它们。可选的 HTTP 模块只有在运行 `configure` 时明确指定才会安装。这些模块处理如 SSL、HTTP 认证、HTTP 代理、gzip 压缩等多种功能。我们将在下一章介绍一些可选的 HTTP 模块。

到目前为止我们讨论的所有配置指令以及在本章和后续章节中讨论的指令，都在 `nginx.conf` 文件中指定。该文件的默认位置是 `/usr/local/conf/nginx.conf`。

# 标准 HTTP 模块

如前所述，标准 HTTP 模块默认内置于 Nginx 中，除非你明确禁用它们。如其名所示，这些模块为 web 服务器提供标准的 HTTP 功能。接下来，我们将查看一些重要的标准 HTTP 模块。

## 核心模块（HttpCoreModule）

核心模块处理核心的 HTTP 功能。这包括协议版本、HTTP keepalive、位置（基于 URI 的不同配置）、文档根目录等。与 HTTP 核心模块相关的配置指令超过 74 个，环境变量超过 30 个。我们将简要讨论其中最重要的一些。

## 指令解释

以下是一些核心模块指令的说明。此列表并不详尽，你可以在[`wiki.nginx.org/HttpCoreModule`](http://wiki.nginx.org/HttpCoreModule)找到完整的列表。

### server

`server` 指令定义服务器上下文。它在配置文件中定义为 `server {...}` 块。每个 `server` 块都代表一个虚拟服务器。你必须在 `server` 块内指定 `listen` 指令来定义该虚拟服务器的主机 IP 和端口。或者，你可以指定一个 `server_name` 指令来定义该虚拟服务器的所有主机名。

```
server {
  server_name www.acme.com *.acme.com www.acme.org;
  ....
  ....
}
server {
  listen myserver.com:8001;
  ....
  ....
}
```

### server_name

`server_name` 指令定义虚拟服务器的名称。它可以包含多个主机名，第一个主机名将成为默认的服务器名称。主机名可以是精确的字符串文字、通配符、正则表达式，或这些的组合。你还可以定义一个空主机名为 `""`，这允许在主机 HTTP 头为空时处理请求。

通配符名称只能在点边界上以及名称的开始或结尾处使用星号（`*`）。例如，`*.example.com` 是一个有效的名称；然而，`ac*e.example.com` 是无效的名称。

正则表达式服务器名可以是任何兼容 PCRE 的正则表达式，且必须以 `~` 开头。

```
server_name ~^www\d+\.acme\.org$
```

如果在此指令中指定环境变量 `$hostname`，则使用机器的主机名。

#### listen

`listen` 指令指定了服务器的 `listen` 地址。`listen` 地址可以是 IP 地址和端口的组合、主机名和端口的组合，或者仅仅是一个端口。

```
server {
  listen 8001
  server_name www.acme.com *.acme.com www.acme.org
  ...
}
```

如果在 `listen` 指令中未指定端口，当 Nginx 服务器以 `superuser` 权限运行时，默认使用端口 80，否则使用端口 8000。

Nginx 还可以使用以下语法监听 UNIX 套接字：

```
listen unix:/var/lock/nginx
```

IPv6 地址可以使用 `[]` 括号来指定：

```
listen [::]:80
listen [2001:db8::1]
```

指定 IPv6 地址还可以启用 IPv4 地址。在前面的示例中，当您在 `listen` 指令中启用 `[::]:80` 地址（通过 IPv6 绑定端口 80）时，Linux 默认也会启用 IPv4 的端口 80。

`listen` 指令还接受多个参数，以下段落列出了其中一些重要的参数。

##### SSL

`listen` 参数允许您指定在此 `listen` 地址上接受的连接将以 SSL 模式工作。

##### default_server

`default_server` 参数将 `listen` 地址设置为默认位置。如果没有 `listen` 地址指定默认值，第一条 `listen` 声明将成为默认值。对于 HTTP 请求，Nginx 会检查请求头字段 `Host` 来确定该请求应该路由到哪个服务器。如果其值与任何服务器名称都不匹配，或者请求根本没有此头字段，Nginx 将把请求路由到默认服务器。

```
listen  8001
listen  443 default_server ssl
```

`ssl` 选项指定该地址上的所有连接都应使用 SSL。只有在服务器编译时启用了 SSL 支持，`ssl` 选项才会生效。

`listen` 指令还有其他与 `listen` 和 `bind` 系统调用相关的参数。例如，您可以通过提供 `rcvbuf` 和 `sndbuf` 参数来修改监听套接字的发送和接收缓冲区。您可以在官方文档中详细了解它们，网址为 [`nginx.org/en/docs/http/ngx_http_core_module.html`](http://nginx.org/en/docs/http/ngx_http_core_module.html)。

#### location

`location` 指令是服务器上下文配置。在 `server` 块中可以包含多个位置配置块，每个配置块都引用该服务器内的一个独特 URI。它是最重要且最常用的指令之一，允许您基于 URI 指定配置。与用户请求的 URI 匹配的位置将导致该特定配置块处理用户请求。您可以灵活地指定配置，可以使用字符串字面量，也可以使用正则表达式。正则表达式可以用于大小写敏感的比较（以 `~` 为前缀）或大小写不敏感的比较（以 `~*` 为前缀）。您还可以通过在字符串前添加 `^~` 来禁用正则表达式匹配。

匹配的顺序如下：

1.  首先，带有 `=` 的字符串字面量会被评估，并且匹配时停止搜索。

1.  剩余的字符串会被匹配；遇到`^~`时，搜索会停止。在所有非正则表达式字符串中，会选择最长匹配的前缀。

1.  正则表达式的搜索顺序是按照它们在`nginx.conf`文件中出现的顺序进行的。

1.  如果有两个匹配项，一个来自正则表达式，另一个来自字符串，则使用字符串。

    ```
    location = / matches only /
    location / matches any URI
    location ~/index matches a lower case /index as a subsring in anyposition
    ```

配置定义的顺序并不重要，它们始终会按照之前提到的顺序进行评估。

```
location ^~/index/main.jpg
location ~^/index/.*\.jpg$
```

在这个示例中，像`/index/main.jpg`这样的 URI 将选择第一个规则，尽管两个模式都匹配。这是由于`^~`前缀，它禁用了正则表达式搜索。

也可以使用`@`定义命名位置，这些位置供内部使用。例如：

```
location @internalerror (
  proxy_pass http://myserver/internalerror.html
)
```

然后，你可以在其他配置中使用`@internalerror`，例如：

```
location / (
  error_page /500.html @internalerror;
)
```

#### server_names_hash_bucket_size

Nginx 将静态数据存储在哈希表中，以便快速访问。每一组静态数据（如服务器名称）都会有一个哈希表。相同的名称会进入同一个哈希桶，`server_names_hash_bucket_size`参数控制服务器名称哈希表中哈希桶的大小。

此参数（以及其他`hash_bucket_size`参数）应该是处理器缓存行大小的倍数。这有助于优化哈希桶内的搜索，确保任何条目最多只需两次内存读取即可找到。在 Linux 系统中，你可以通过以下方法查找缓存行大小：

```
$ getconf LEVEL1_DCACHE_LINESIZE
```

#### server_names_hash_max_size

`server_names_hash_max_size`指令指定哈希表的最大大小，该哈希表包含服务器名称。使用`server_names_hash_bucket_size`参数计算得到的哈希表大小不得超过此值。默认值为`512`。

```
http {
  ...
  ...
  server_names_hash_bucket_size 128;
  server_names_hash_max_size 1024;

  server {
  ...
  ...

  }
}
```

#### tcp_nodelay/tcp_nopush

`tcp_nodelay`和`tcp_nopush`指令允许你控制 Linux 中`tcp_nodelay`、`tcp_nopush`或`tcp_nocork`的套接字设置。`tcp_nodelay`对于那些频繁发送小包而不关心响应的服务器非常有用。该指令本质上禁用了 TCP/IP 套接字上的 Nagle 算法。`tcp_nopush`或`tcp_nocork`只有在你使用`sendfile()`内核选项时才会生效。

#### sendfile

`sendfile`指令激活或停用 Linux 内核的`sendfile()`函数。对于需要高效传输文件的应用程序，如 Web 服务器，这提供了显著的性能优势。Web 服务器大部分时间都在将存储在磁盘上的文件传输到连接到客户端的网络连接中，客户端运行着一个 Web 浏览器。通常，这包括`read()`和`write()`调用，这需要上下文切换以及在用户或内核缓冲区之间的数据复制。`sendfile`系统调用允许 Nginx 通过快速通道`sendfile()`将文件从磁盘复制到套接字，整个过程都停留在内核空间。从 Linux 2.6.22 开始，如果你希望使用带有直接 I/O（O_DIRECT）的 Aio，应该关闭`sendfile`。如果 Web 服务器提供大文件（> 4 MB），这种方式可能更高效。在 FreeBSD 5.2.1 之前和 Nginx 0.8.12 之前，必须禁用`sendfile`支持。

#### sendfile_max_chunk

当设置为非零值时，`sendfile_max_chunk`指令限制了单次`sendfile()`调用中可传输的数据量。

#### root

`root`指定了请求的文档根目录，通过将路径附加到请求中。例如，使用以下配置：

```
location  /images/ {
  root  /var/www;
}
```

对`/web/logo.gif`的请求将返回文件`/var/www/images/logo.gif`。

#### resolver/resolver_timeout

这允许你指定 DNS 服务器的地址或名称。你还可以定义名称解析的超时，例如：

```
resolver 192.168.220.1;
resolver_timeout 2s;
```

#### aio

`aio`指令允许 Nginx 在 Linux 中使用 POSIX 的`aio`支持。这种异步 I/O 机制允许多个非阻塞的读写操作。

```
location /audio {
  aio on;
  directio 512;
  output_buffers 1 128k;
}
```

在 Linux 上，这将禁用`sendfile`支持。在 FreeBSD 5.2.1 之前和 Nginx 0.8.12 之前，必须禁用`sendfile`支持。

```
location /audio {
  aio on;
  sendfile off;
}
```

从 FreeBSD 5.2.1 和 Nginx 0.8.12 开始，可以与`sendfile`一起使用。

#### alias

`alias`指令类似于`root`指令，但有一个微妙的区别。当你为一个位置定义别名时，会搜索别名路径，而不是实际位置。这与`root`指令略有不同，`root`指令将根路径附加到位置。例如：

```
location  /img/ {
  alias  /var/www/images/;
}
```

对于`/img/logo.gif`的请求将指示 Nginx 提供文件`/var/www/images/logo.gif`。

别名也可以在通过正则表达式指定的位置中使用。

#### error_page

`error_page`指令允许你根据错误代码显示错误页面。例如：

```
error_page   404          /404.html;
error_page   502 503 504  /50x.html;
```

可以显示不同的错误代码，而不是原始错误代码。还可以指定像 PHP 文件这样的脚本（它生成错误页面的内容）。这使你能够编写一个通用的错误处理程序，根据错误代码和类型生成定制化的页面：

```
error_page 404 =200 /empty.gif;
error_page 500 =errors.php;
```

如果在重定向过程中不需要更改浏览器中的 URL，则可以将错误页面的处理重定向到指定名称的位置：

```
location / (
  error_page 404 @errorhandler;
)
location @ errorhandler (
  proxy_pass http://backend/errors.php;
)
```

#### keepalive_disable、keepalive_timeout 和 keepalive_requests

`keepalive_disable`指令允许你为特定浏览器禁用 HTTP `keepalive`。

`keepalive_timeout`为与客户端的`keepalive`连接分配超时时间。服务器将在此时间后关闭连接。你还可以指定零值来禁用客户端连接的 keepalive。这会将 HTTP 头部 Keep-Alive: timeout=time 添加到响应中。

`keepalive_requests`参数决定通过单个 keepalive 连接服务的客户端请求数量。一旦达到此限制，连接将被关闭，新的 keepalive 会话将会启动。

## 控制访问（HttpAccessModule）

`HttpAccessModule`允许基于 IP 的访问控制。你可以指定 IPv4 和 IPv6 地址。另一种选择是使用 GeoIP 模块。

规则根据声明顺序进行检查。有两个指令叫做`allow`和`deny`，它们控制访问。匹配特定地址或地址集合的第一个规则是被遵守的。

```
location / {
  deny    192.168.1.1;
  allow   192.168.1.0/24;
  allow   10.1.1.0/16;
  allow   2620:100:e000::8001;
  deny    all;
}
```

在这个示例中，访问权限被授予网络 10.1.1.0/16 和 192.168.1.0/24，但地址 192.168.1.1 被拒绝访问，并且所有其他地址都按`deny all`规则被拒绝访问，该规则在该位置块中最后匹配。此外，它还允许一个特定的 IPv6 地址，其他所有地址都将被拒绝。

顺序非常重要。规则会根据顺序解释。所以，如果你将`deny all`移动到列表的顶部，所有请求都将被拒绝，因为这是遇到的第一个规则，因此它优先执行。

## 用户身份验证（HttpBasicAuthModule）

你可以使用`HttpBasicAuthModule`通过基于 HTTP 基本认证的用户名和密码保护你的站点或部分内容。这是最简单的强制访问控制 Web 资源的技术，因为它不需要使用 cookies、会话标识符或登录页面。相反，HTTP 基本认证使用静态的标准 HTTP 头部，这意味着不需要预先进行握手。

以下是一个示例配置：

```
location  /  {
  auth_basic            "Registered Users Only";
  auth_basic_user_file  htpasswd;
}
```

### 解释指令

现在让我们看看这个模块的一些重要指令。

#### auth_basic

该`auth_basic`指令包含通过 HTTP 基本认证来测试用户名和密码。指定的值将作为身份验证区域使用。

#### auth_basic_user_file

`auth_basic_user_file`指令设置身份验证区域的密码文件名。路径相对于 Nginx 配置文件所在的目录。

文件格式如下：

```
user:pass
user2:pass2:comment
user3:pass3
```

密码必须通过 crypt（3）函数进行编码。你可以使用`PLAIN`、`MD5`、`SSHA`和`SHA1`加密方法。如果你的系统上安装了 Apache，你可以使用`htpasswd`工具生成`htpasswd`文件。

该文件应由 Nginx 工作进程读取，这些进程以非特权用户身份运行。

## 负载均衡（HttpUpstreamModule）

`HttpUpstreamModule` 允许基于多种技术（如轮询、权重、IP 地址等）对上游服务器进行简单的负载均衡。

示例：

```
upstream servers  {
  server server1.example.com weight=5;
  server server2.example.com:8080;
  server unix:/tmp/server3;
}
server {
  location / {
    proxy_pass  http://servers;
  }
}
```

### 解释指令

`HttpUpstreamModule` 的一些重要指令如下：

#### ip_hash

`ip_hash` 指令根据客户端的 IP 地址将请求分配到不同的服务器。

哈希的键是客户端的 IP 地址（IPv4 或 IPv6）。该方法保证客户端请求总是转发到同一台服务器。如果服务器不可用，请求将被转发到另一台服务器。

你可以结合使用 `ip_hash` 和 `weight` 基于的方法。如果某个服务器需要下线，你必须将该服务器标记为 `down`。

例如：

```
upstream backend {
  ip_hash;
  server   server1.example.com weight=2;
  server   server2.example.com;
  server   server3.example.com  down;
  server   server4.example.com;
}
```

#### server

`server` 指令用于指定上游服务器的名称。可以使用域名、地址、端口或 UNIX 套接字。如果域名解析为多个地址，则会使用所有这些地址。

此指令接受多个参数，具体如下：

+   `weight`：设置服务器的权重。如果未设置，权重默认为 1。

+   `max_fails`：这是在 `fail_timeout` 时间段内与服务器通信的失败次数，超过该次数后，服务器被视为 `down`。如果没有设置，则只尝试一次。设置为 `0` 会关闭此检查。什么算作失败由 `proxy_next_upstream` 或 `fastcgi_next_upstream` 定义（`http_404` 错误不算在 `max_fails` 之内）。

+   `fail_timeout`：在被视为 `down` 之前，连接上游服务器失败的尝试在此时间段内进行。它也是服务器被视为不可用的时间（在进行下一次尝试之前）。默认值为 10 秒。

+   `down`：该参数将服务器标记为离线。

如果只使用一个上游服务器，Nginx 将忽略 `max_fails` 和 `fail_timeout` 参数。如果上游服务器不可用，这可能导致请求丢失。你可以多次使用相同的服务器名称来模拟重试。

#### upstream

`upstream` 指令描述了一组上游或后端服务器，向这些服务器发送请求。这些服务器可以在 `proxy_pass` 和 `fastcgi_pass` 指令中作为一个整体使用。每个定义的服务器可以位于不同的端口上。你还可以指定监听本地套接字的服务器。

服务器可以分配不同的权重。如果未指定，权重默认为 1。

```
upstream servers {
  server server1.example.com weight=5;
  server 127.0.0.1:8080       max_fails=3  fail_timeout=30s;
  server unix:/tmp/localserver;
}
```

请求会根据服务器的轮询方式以及服务器权重进行分配。

例如，之前给出的每七个请求的分布如下：五个请求会发送到 `server1.example.com`，每个第二和第三个服务器各一个请求。如果连接到服务器时发生错误，请求会发送到下一个服务器。在之前的示例中，如果服务器 2 发生故障，请求会在 30 秒内尝试三次，才会转发到服务器 3。

## 充当代理（HttpProxyModule）

`HttpProxyModule` 允许 Nginx 充当代理并将请求传递给另一个服务器。

```
location / {
  proxy_pass        http://app.localhost:8000;
}
```

注意，在使用 `HttpProxyModule`（或甚至使用 FastCGI）时，整个客户端请求会在 Nginx 中缓冲，之后才会传递给代理服务器。

### 解释指令

`HttpProxyModule` 的一些重要指令如下：

#### proxy_pass

`proxy_pass` 指令设置代理服务器的地址和位置将映射到的 URI。地址可以是主机名或地址和端口，例如：

```
proxy_pass http://localhost:8000/uri/;
```

或者，地址也可以给出为 UNIX 套接字路径：

```
proxy_pass http://unix:/path/to/backend.socket:/uri/;
```

`path` 在单词 `unix` 后面，两个冒号之间给出。

你可以使用 `proxy_pass` 指令将客户端请求的头部转发给代理服务器。

```
proxy_set_header Host $host;
```

在传递请求时，Nginx 会用 `proxy_pass` 指令指定的位置替换 URI 中的原位置。

如果在代理位置内，URI 被 `rewrite` 指令更改，则此配置将用于处理请求。例如：

```
location  /name/ {
  rewrite      /name/([^/] +)  /users?name=$1  break;
  proxy_pass   http://127.0.0.1;
}
```

请求 URI 在规范化后传递给代理服务器，如下所示：

+   双斜杠会被替换为单个斜杠。

+   任何对当前目录的引用，例如 "./"，都会被移除。

+   任何对上级目录的引用，例如 "../"，都会被移除。

如果没有指定 URI（例如在 "`http://example.com/request`" 中，`/request` 是 URI 部分），则请求 URI 会以客户端发送的相同形式传递给服务器。

```
 location /some/path/ {
     proxy_pass http://127.0.0.1;
 }
```

如果你需要代理连接到上游服务器组并使用 SSL，则你的 `proxy_pass` 规则应该使用 `https://`，并且你还必须在上游定义中显式设置 SSL 端口。例如：

```
upstream https-backend {
  server 10.220.129.20:443;
}

server {
  listen 10.220.129.1:443;
  location / {
    proxy_pass https://backend-secure;
  }
}
```

#### proxy_pass_header

`proxy_pass_header` 指令允许传输响应时禁止的头行。

```
For example:
location / {
  proxy_pass_header X-Accel-Redirect;
}
```

#### proxy_connect_timeout

`proxy_connect_timeout` 指令设置与上游服务器的连接超时。你不能将此超时时间设置为超过 75 秒。请记住，这不是响应超时，而仅仅是连接超时。

这是直到服务器返回通过 `proxy_read_timeout` 指令配置的页面之前的时间。如果你的上游服务器已经启动但挂起，此语句将无效，因为与服务器的连接已经建立。

#### proxy_next_upstream

`proxy_next_upstream` 指令决定在哪些情况下请求将被转发到下一个服务器：

+   `error`：在连接到服务器、向其发送请求或读取其响应时发生错误

+   `timeout`：在与服务器连接、传输请求或读取服务器响应时发生了超时

+   `invalid_header`：服务器返回了空的或错误的响应

+   `http_500`：服务器响应代码为 500

+   `http_502`：服务器响应代码为 502

+   `http_503`：服务器响应代码为 503

+   `http_504`：服务器响应代码为 504

+   `http_404`：服务器响应代码为 404

+   `off`：禁用请求转发

仅在向某个服务器发送请求时发生错误时，才能将请求转发到下一个服务器。如果由于错误或其他原因中断了请求发送，转发将无法进行。

#### proxy_redirect

`proxy_redirect`指令允许你通过替换来自上游服务器响应中的文本来操作 HTTP 重定向。具体来说，它替换`Location`和`Refresh`头中的文本。

HTTP `Location`头字段是由代理服务器响应返回的，原因如下：

+   用于指示资源已临时或永久移动。

+   提供有关新创建的资源位置的信息。这可能是 HTTP `PUT`请求的结果。

假设代理服务器返回了以下内容：

```
Location: http://localhost:8080/images/new_folder
```

如果你将`proxy_redirect`指令设置为以下内容：

```
proxy_redirect http://localhost:8080/images/ http://xyz/;
```

`Location`文本将被重写，类似于以下内容：

```
Location: http://xyz/new_folder/.
```

可以在重定向地址中使用某些变量：

```
proxy_redirect http://localhost:8000/ http://$location:8000;
```

你还可以在此指令中使用正则表达式：

```
proxy_redirect ~^(http://[^:]+):\d+(/.+)$ $1$2;
```

`off`值在其级别上禁用所有`proxy_redirect`指令。

```
proxy_redirect off;
```

#### proxy_set_header

`proxy_set_header`指令允许你重新定义并向请求中添加新的 HTTP 头，发送到代理服务器。

你可以将静态文本与变量结合使用，作为`proxy_set_header`指令的值。

默认情况下，以下两个头将被重新定义：

```
proxy_set_header Host $proxy_host;
proxy_set_header Connection Close;
```

你可以通过以下方式将原始`Host`头值转发到服务器：

```
proxy_set_header Host $http_host;
```

然而，如果客户端请求中缺少此头部，则不会转发任何信息。

最好使用变量`$host`；其值等于请求头`Host`，如果客户端请求中缺少该头，则为服务器的基本名称。

```
proxy_set_header Host $host;
```

你可以传递服务器名称以及代理服务器的端口：

```
proxy_set_header Host $host:$proxy_port;
```

如果你将值设置为空字符串，则不会将头传递给上游代理服务器。例如，如果你希望禁用上游的 gzip 压缩，可以执行以下操作：

```
proxy_set_header  Accept-Encoding  "";
```

#### proxy_store

`proxy_store`指令设置上游文件存储的路径，路径与`alias`或`root`指令相对应。`off`指令值禁用本地文件存储。请注意，`proxy_store`与`proxy_cache`不同。它仅仅是将代理文件存储到磁盘的一种方法。它可以用来构建类似缓存的设置（通常涉及基于`error_page`的回退）。此`proxy_store`指令的默认值为`off`。该值可以包含静态字符串和变量的组合。

```
proxy_store   /data/www$uri;
```

文件的修改日期将设置为响应中的`Last-Modified`头的值。响应首先写入`proxy_temp_path`指定的路径中的临时文件，然后重命名。建议将此位置路径与存储文件的路径保持一致，以确保它只是一个重命名操作，而不是创建文件的两个副本。

示例：

```
location /images/ {
  root                 /data/www;
  error_page           404 = @fetch;
}

location /fetch {
  internal;
  proxy_pass           http://backend;
  proxy_store          on;
  proxy_store_access   user:rw  group:rw  all:r;
  proxy_temp_path      /data/temp;
  alias                /data/www;
}
```

在此示例中，`proxy_store_access`定义了创建文件的访问权限。

在发生`404`错误的情况下，获取的内部位置会代理到远程服务器，并将本地副本存储在`/data/temp`文件夹中。

#### proxy_cache

`proxy_cache`指令通过使用`off`值来关闭缓存，或者设置缓存的名称。此名称随后可以在其他地方使用。让我们来看以下示例，在 Nginx 服务器上启用缓存：

```
http {
  proxy_cache_path  /var/www/cache levels=1:2 keys_zone=my-cache:8m max_size=1000m inactive=600m;
  proxy_temp_path /var/www/cache/tmp;

  server {
    location / {
      proxy_pass http://example.net;
      proxy_cache my-cache;
      proxy_cache_valid  200 302  60m;
      proxy_cache_valid  404      1m;
    }
  }
}
```

上一个示例创建了一个名为`my-cache`的缓存。它设置了响应代码`200`和`302`的缓存有效期为`60m`，而`404`的缓存有效期为`1m`。

缓存数据存储在`/var/www/cache`文件夹中。`levels`参数设置缓存中子目录的层数。最多可以定义三层。

`key_zone`的名称后面跟着一个非活动间隔。在`my-cache`中的所有非活动项将在`600m`后被清除。非活动间隔的默认值为 10 分钟。

## 压缩内容（HttpGzipModule）

`HttpGzipModule`允许即时 gzip 压缩。

```
  gzip             on;
  gzip_min_length  1000;
  gzip_proxied     expired no-cache no-store private auth;
  gzip_types       text/plain application/xml;
```

计算出的压缩比作为原始响应大小与压缩后响应大小的比率，可以通过变量`$gzip_ratio`获得。

### 解释指令

`HttpGzipModule`的一些重要指令如下：

#### gzip

`gzip`指令启用或禁用 gzip 压缩。

#### gzip_buffers

`gzip_buffers`指令分配压缩响应将存储的缓冲区的数量和大小。如果未设置，则一个缓冲区的大小等于页面的大小；根据平台的不同，可能是`4K`或`8K`。

#### gzip_comp_level

`gzip_comp_level`指令设置响应的 gzip 压缩级别。压缩级别介于`1`和`9`之间，其中`1`表示压缩最少（最快），`9`表示压缩最多（最慢）。

#### gzip_disable

`gzip_disable`指令会禁用匹配给定正则表达式的浏览器或用户代理的 gzip 压缩。例如，要为 Internet Explorer 6 禁用 gzip 压缩，可以使用：

```
gzip_disable     "msie6";
```

这是一个有用的设置，因为某些浏览器（如 MS Internet Explorer 6）无法正确处理压缩响应。

#### gzip_http_version

`gzip_http_version`指令根据 HTTP 请求版本（1.0 或 1.1）开启或关闭 gzip 压缩。

#### gzip_min_length

`gzip_min_length`指令设置响应的最小长度（以字节为单位），只有超过此字节长度的响应才会被压缩。长度由`Content-Length`头确定。

#### gzip_proxied

`gzip_proxied`指令启用或禁用代理请求的压缩。通过`Via` HTTP 头识别代理请求。此头通知服务器请求是通过哪些代理发送的。根据各种 HTTP 头，我们可以按如下方式启用或禁用代理请求的压缩：

+   `off`：禁用带有`Via`头的请求的压缩。

+   `expired`：如果响应头包含一个禁用缓存的`Expires`字段，则启用压缩。

+   `no-cache`：如果`Cache-Control`头设置为`no-cache`，则启用压缩。

+   `no-store`：如果`Cache-Control`头设置为`no-store`，则启用压缩。

+   `private`：如果`Cache-Control`头设置为`private`，则启用压缩。

+   `no_last_modified`：如果`Last-Modified`未设置，则启用压缩。

+   `no_etag`：如果没有`ETag`头，则启用压缩。

+   `auth`：如果存在`Authorization`头，则启用压缩。

+   `any`：为所有代理请求启用压缩。

#### gzip_types

`gzip_types`指令启用对除了 text 或 html 之外的其他 MIME 类型的压缩。`text/html`始终被压缩。

## 控制日志记录（HttpLogModule）

`HttpLogModule`控制 Nginx 如何记录资源请求，例如：

```
access_log  /var/log/nginx/access.log  gzip  buffer=32k;
```

请注意，这不包括记录错误。

### 解释指令

`HttpLogModule`的一些重要指令如下所示。

#### access_log

`access_log`指令设置访问日志文件的路径、格式和缓冲区大小。使用`off`作为值会在当前级别禁用日志记录。如果未指明格式，则默认为`combined`。缓冲区的大小不得超过写入磁盘文件的原子记录的大小。对于 FreeBSD 3.0-6.0，大小不受限制。如果指定了 gzip，则日志在写入磁盘之前会被压缩。默认的缓冲区大小是 64K，压缩级别为 1。

可以写入的原子大小称为`PIPE_BUF`。管道缓冲区的容量在不同系统之间有所不同。

例如，Mac OS X 默认使用 16,384 字节的容量，但如果向管道写入大量数据，它可以切换到 65,336 字节的容量。或者，如果管道缓冲区已使用太多内核内存，它将切换到单个系统页面的容量（参见 `xnu/bsd/sys/pipe.h` 和 `xnu/bsd/kern/sys_pipe.c`；由于这些来自 FreeBSD，因此这里也可能发生相同的行为）。

根据 Linux pipe(7) 手册页，管道容量自 Linux 2.6.11 以来为 65,536 字节，在此之前为单个系统页面（例如，32 位 x86 系统上为 4096 字节）。每个管道的缓冲区可以通过 fcntl 系统调用更改，最大值为 `/proc/sys/fs/pipe-max-size`。

#### log_format

`log_format` 指令描述了日志条目的格式。你可以在格式中使用通用变量，也可以使用仅在写入日志时存在的变量。以下是 `log_format` 的一个示例：

```
log_format gzip '$msec $request $remote-addr $status $bytes_sent';
```

你可以通过指定要记录的信息来设定日志条目的格式。你可以指定的一些选项如下：

+   `$body_bytes_sent`：这是传输给客户端的字节数，减去响应头。

+   `$bytes_sent`：这是发送给客户端的字节数。

+   `$connection`：这是连接数。

+   `$msec`：这是写入日志条目时的当前时间（微秒精度）。

+   `$pipe`：如果请求是流水线请求，则值为 `p`。

+   `$request_length`：这是请求体的长度。

+   `$request_time`：这是 Nginx 处理请求所用的时间，单位为秒，精确到毫秒。

+   `$status`：这是响应的状态。

+   `$time_iso8601`：这是 ISO 8601 格式的时间，例如，2011-03-21T18:52:25+03。

+   `$time_local`：这是常见日志格式下的本地时间。

## 设置响应头（HttpHeadersModule）

`HttpHeadersModule` 允许设置任意的 HTTP 头。

### 解释指令

`HttpHeadersModule` 的一些重要指令如下：

#### add_header

`add_header` 指令在响应的头部列表中添加一个头部，当响应代码为 200、201、204、206、301、302、303、304 或 307 时。该值可以包含变量，并且可以包含负数或正数的时间值。

请注意，你不应该使用该指令来替代或覆盖某个头的值。通过该指令指定的头部仅仅是附加到头部列表中。

#### expires

`expires` 指令用于设置响应中的 `Expires` 和 `Cache-Control` 头。你可以将值设置为 `off`，以保持这些头部不变。该字段中的时间是当前时间与指令中指定的时间之和。如果使用了修改时间参数，时间是文件修改时间与指令中指定的时间之和。

+   `epoch`：将 `Expires` 头设置为 `1970 年 1 月 1 日 00:00:01 GMT` 的绝对值。

+   `max`：此设置将`Expires`头部设置为`2037 年 12 月 31 日 23:59:59 GMT`，并将`Cache-Control`头部设置为 10 年。

你可以使用`@`指定时间间隔：

```
@5h40m
```

`Cache-Control`头部的内容取决于指定时间的符号。负值时间将其设置为`no-cache`，正值时间将其设置为秒数。

以下是一个示例配置：

```
expires    12h;
expires    modified +14h;
expires    @5h;
expires    0;
expires    -1;
expires    epoch;
add_header X-Name example.org
```

## 重写请求（HttpRewriteModule）

`HttpRewriteModule`用于通过正则表达式更改请求 URI、重定向客户端，并根据条件和变量值选择不同的配置。为了使用此模块，应该在编译 Nginx 时启用 PCRE 支持。

指令的处理从服务器级别开始。之后，搜索与请求匹配的位置块，并执行其中的任何重写指令。如果这个处理导致进一步的重写，会根据改变后的 URI 搜索新的位置块。这个循环会持续进行 10 次，直到服务器抛出`500`错误。

### 解释指令

`HttpRewriteModule`的一些重要指令如下：

#### break

`break`指令停止当前块中任何其他重写块指令的处理。

```
if ($slow) {
  limit_rate  10k;
  break;
}
```

#### if

`if`指令检查一个条件。如果条件为`true`，则执行大括号内指示的代码，并根据以下块中的配置处理请求。`if`块中的配置会继承自上一级。

以下被视为有效条件。

+   变量的名称是一个条件。如果变量包含空字符串`""`或`0`，则条件的值为`false`。

+   使用比较运算符与变量进行比较，将其与另一个变量或字符串进行比较。

+   使用`~`、`*~`或`!~`运算符将变量与正则表达式匹配。`*~`用于不区分大小写的比较，而`!~`是一个不等于运算符。

+   你可以使用`-f`或`!-f`运算符检查文件是否存在（类似于 BASH 测试）。

+   使用`-d`或`!-d`检查目录是否存在。

+   使用`-e`或`!-e`检查文件、目录或符号链接是否存在。

+   使用`-x`或`!-x`检查文件是否可执行。

通过将正则表达式的一部分放入圆括号内，你可以将该部分正则表达式分组。这样，你可以对整个组应用量词，或者将交替限制为正则表达式的部分。这些部分可以通过`$1`到`$9`的变量访问。

示例：

```
if ($http_user_agent ~ MSIE) {
  rewrite ^(.*)$  /msie/$1  break;
}
if ($http_cookie ~* "val=([^;] +)(?:;|$)" ) {
  set $val $1;
}
if ($request_method = GET ) {
  return 405;
}
if ($args ~ post=140){
  rewrite ^ http://acme.com/ permanent
}
```

#### return

`return`指令停止执行并返回状态码。可以使用任何 HTTP 返回码，范围从`0`到`999`。

如果你想终止连接并且不希望在响应中发送任何头部，可以使用返回码`444`。

#### rewrite

`rewrite`指令执行实际的重写操作，并根据正则表达式和替换字符串更改 URI。指令按配置文件中定义的顺序执行。`flag`参数使得可以在当前块中停止重写过程。

如果替换字符串以`http://`开头，客户端将被重定向，且任何后续的重写指令将终止。

`flag`参数的值可以是以下之一：

+   `last`：这将完成当前重写指令的处理，并查找与重写后 URI 匹配的新块。

+   `break`：这将停止当前块中的重写过程。

+   `redirect`：这将返回一个临时重定向，状态码为`302`，如果替换字符串不以`http://`或`https://`开头，则使用此状态。

+   `permanent`：这将返回一个永久重定向，状态码为`301`

请注意，在外部位置块中，`last`和`break`实际上是相同的。

示例：

```
rewrite  ^(/media/.*)/video/(.*)\..*$  $1/mp3/$2.avi last;
rewrite  ^(/media/.*)/audio/(.*)\..*$  $1/mp3/$2.ra break;
return 403;
```

但如果我们将这些指令放在`location`块中，必须将标志`last`替换为`break`，否则 Nginx 将达到 10 次循环限制并返回错误`500`：

```
location /download/ {
  rewrite  ^(/media/.*)/video/(.*)\..*$  $1/mp3/$2.avi  break;
  rewrite  ^(/media/.*)/audio/(.*)\..*$  $1/mp3/$2.ra   break;
  return   403;
}
```

如果替换字符串中有参数，剩余的请求参数将被附加到它们后面。为了避免附加它们，可以在最后一个字符位置放置一个问号：

```
rewrite  ^/pages/(.*)$  /show?page=$1?  last;
```

请注意，对于大括号（{和}），由于它们既用于正则表达式又用于块控制，为了避免冲突，带有大括号的正则表达式必须用双引号（或单引号）括起来。例如，要将 URL `/users/123456` 重写为 `/path/to/users/12/1234/123456.html`，可以使用以下方式（请注意引号）：

```
rewrite  "/users/([0-9]{2})([0-9]{2})([0-9]{2})"/path/to/users/$1/$1$2/$1$2$3.html;
```

如果在重写的末尾指定了`?`，Nginx 将丢弃原始查询字符串。一个好的用例是，当使用`$request_uri`时，应该在重写末尾指定`?`，以避免 Nginx 重复查询字符串。

使用`$request_uri`进行从`www.acme.com`到`acme.com`的重写示例：

```
server {
  server_name www.acme.com;
  rewrite ^ http://acme.com$request_uri? permanent;
}
```

此外，重写仅对路径有效，不对参数有效。要将带有参数的 URL 重写为另一个 URL，请使用以下方式：

```
if ($args ~ post=200){
  rewrite ^ http://acme.com/new-address.html?;
}
```

#### rewrite_log

`rewrite_log`指令启用将重写信息记录到错误日志中的通知级别日志。

#### set

`set`指令为指定的变量设定值。可以使用文本、变量及其组合作为值。

你可以使用 set 来定义一个新变量。请注意，不能设置`$http_xxx`头部变量的值。

#### uninitialized_variable_warn

`uninitialized_variable_warn`指令启用或禁用未初始化变量的警告。

## 与 FastCGI 交互（HttpFastcgiModule）

`HttpFastcgiModule`允许 Nginx 与 FastCGI 进程（即 PHP）交互，并控制将哪些参数传递给该进程。

示例：

```
location / {
  fastcgi_pass   localhost:9090;
  fastcgi_index  index.php;
   fastcgi_param  SCRIPT_FILENAME$document_root/php/$fastcgi_script_name;
  fastcgi_param  QUERY_STRING     $query_string;
  fastcgi_param  REQUEST_METHOD   $request_method;
  fastcgi_param  CONTENT_TYPE     $content_type;
  fastcgi_param  CONTENT_LENGTH   $content_length;
}
```

FastCGI 服务器的名称通过`fastcgi_pass`参数提供。该名称可以是一个 IP 地址，或者带有端口的域名。也可以是一个 UNIX 域套接字。

如果你想向 FastCGI 服务器传递参数，可以使用`fastcgi_param`参数。该参数的值可以是静态值、变量，或者两者的组合。

以下是 PHP 的最小配置：

```
fastcgi_param SCRIPT_FILENAME /php$fastcgi_script_name;
fastcgi_param QUERY_STRING    $query_string;
```

## 简单缓存（HttpMemcachedModule）

你可以使用这个模块通过`memcached`进行简单的缓存。`Memcached`是一个内存中的键值存储，用于存储来自数据库调用、API 调用或页面渲染的任意数据（字符串、对象）的结果。

示例：

```
server {
  location / {
    set $memcached_key $uri$args;
    memcached_pass     http://mem-server:1211
    default_type       text/html;
      error_page         404 502 504 @error;
  }
  location @error {
    proxy_pass http://backend;
  }
}
```

### 解释指令

`HttpMemcachedModule`的一些重要指令如下：

#### memcached_pass

`memcached_pass`指令指定了 memcached 服务器名称，可以是 IP 地址或域名。它也可以包含端口。如果域名对应多个地址，所有地址会以轮询方式进行尝试。

#### memcached_connect_timeout

`memcached_connect_timeout`指令是连接到 memcached 服务器的超时时间。这个超时时间通常最大为`75 秒`。默认值为`60 秒`。

#### memcached_read_timeout

`memcached_read_timeout`指令是从 memcached 服务器读取键的超时时间。此时间是测量两个连续读取之间的时间，如果 memcached 服务器没有响应，则会发生超时。默认值为`60 秒`。

#### memcached_send_timeout

`memcached_send_timeout`指令是向 memcached 服务器发送请求的超时时间。超时仅发生在两个连续写操作之间，而不是整个请求的传输。如果 memcached 服务器在此时间内未收到任何内容，则会关闭连接。

#### memcached_buffer_size

`memcached_buffer_size`指令是接收或发送缓冲区的大小，以字节为单位。它设置用于读取从 memcached 服务器接收到的响应的缓冲区大小。响应会在接收到时同步并立即传递给客户端。默认值为`4K`或`8K`。

#### memcached_next_upstream

哪些故障条件应导致请求转发到另一个 memcached 上游服务器？答案是，只有当`memcached_pass`中的值是一个包含两个或更多服务器的上游块时，才会发生。

## 请求限制（HttpLimitReqModule）

`HttpLimitReqModule`允许按键限制请求处理速率，特别是按地址限制。限制是通过漏桶算法进行的。每当用户发送请求时，都会增加与每个地址相关的计数器，且计数器会周期性地减少。如果计数器在增加时超过阈值，Nginx 会延迟该请求。

以下是一个示例配置：

```
http {
    limit_req_zone  $binary_remote_addr  zone=one:10m   rate=1r/s;

    ...

    server {

        ...

        location /search/ {
            limit_req   zone=one  burst=5;
        }
```

### 解释指令

`HttpLimitReqModule`的一些重要指令如下：

#### limit_req

`limit_req` 指令设置一个共享内存区域和请求的最大突发大小。超出请求会被延迟，直到它们的数量超过最大突发大小，此时请求将以错误 `503`（服务暂时不可用）终止。默认情况下，最大突发大小为零。例如，针对 `limit_req_zone` 指令：

```
  $binary_remote_addr  zone=one:10m   rate=1r/s;
     server {
        location /search/ {
            limit_req   zone=one  burst=5;
        }
```

它允许用户平均每秒最多一次请求，且每秒最多五个请求的突发。

如果在突发中延迟超出请求不是必要的，你应使用 `nodelay` 选项：

```
limit_req zone=one burst=5 nodelay;
```

#### limit_req_log_level

`limit_req_log_level` 指令控制延迟或拒绝请求的日志级别。日志级别可以是 `info`、`notice`、`warn` 或 `error`。拒绝请求的默认日志级别为 `error`。延迟请求将在下一个较低级别记录，例如，当 `limit_req_log_level` 设置为 "error" 时，延迟请求会以 "warn" 级别记录。

#### limit_req_zone

`limit_req_zone` 指令设置了一个共享内存区域的名称和参数，用于存储不同键的状态。该状态特别存储当前超出请求的数量。键是指定变量的任何非空值（空值不被计入）。其用法示例如下：

```
limit_req_zone $binary_remote_addr zone=myzone:20m rate=5r/s;
```

在这种情况下，有一个 20 MB 大小的区域，名为 `myzone`，且该区域的查询平均速度限制为每秒 5 次请求。

在这种情况下，按用户跟踪会话。一个 1 MB 的区域大约可以存储 16,000 个 64 字节的状态。如果区域的存储已满，服务器将对所有进一步的请求返回错误 `503`（服务暂时不可用）。

速度设置为每秒请求数或每分钟请求数。速率必须是整数；因此，如果你需要指定每秒少于一次的请求，例如每两秒请求一次，你应将其指定为 `30r/m`。

## 限制连接数（HttpLimitConnModule）

`HttpLimitConnModule` 使得可以限制每个键（如 IP 地址）的并发连接数。

一个示例配置：

```
http {
    limit_conn_zone $binary_remote_addr zone=addr:10m;

    ...

    server {

        ...

        location /download/ {
            limit_conn addr 1;
        }
```

### 解释指令

`HttpLimitConnModule` 的一些重要指令如下：

#### limit_conn

`limit_conn` 指令的值定义了每个区域的连接限制。当此限制被超过时，服务器将返回 `503` 错误（服务暂时不可用）来响应请求。

可以在同一上下文中使用多个不同区域的限制指令。例如：

```
limit_conn_zone $binary_remote_addr zone=addr:10m;

server {
    location /download/ {
        limit_conn addr 1;
    }
```

这仅允许每个唯一的 IP 地址一次性连接。

#### limit_conn_zone

`limit_conn_zone` 指令设置了一个区域的参数，该区域用于存储不同键的状态。这个状态特别存储当前连接数。键是指定变量的值。例如：

```
limit_conn_zone $binary_remote_addr zone=addr:10m;
```

在这里，客户端的 IP 地址作为键。如果区域的存储空间已用尽，服务器将对所有进一步的请求返回错误 `503`（服务暂时不可用）。

#### limit_conn_log_level

`limit_conn_log_level` 指令设置错误日志级别，当达到连接限制时会使用该级别。默认的日志级别是 `error`。

#### limit_conn_status

`limit_conn_status` 指令定义当达到限制时的响应代码。默认值是 `503`（服务不可用）。

# 概要

在本章中，我们研究了几个标准的 HTTP 模块。这些模块默认提供了非常丰富的功能。你可以在配置时禁用这些模块，若不禁用，它们会默认安装。本章中的模块及其指令并非详尽无遗。Nginx 的在线文档可以为你提供更多详细信息。

在下一章，我们将探讨一些可选的 HTTP 模块。
