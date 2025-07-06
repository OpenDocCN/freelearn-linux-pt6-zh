

# 第六章：NGINX 作为反向代理

过去，Web 相对来说是由简单的网站组成的。然而，近年来这种情况发生了变化。现代 Web 包括与个人博客、新闻网站一样多的复杂 SaaS 应用程序。随着 Web 的演变，用来驱动这些应用程序的技术列表也在不断发展。如今，单单是一个快速的静态文件服务器和 FastCGI 接口已经不够了。现在，我们需要考虑诸如 WebSockets 等技术，以及 Web 应用架构的复杂性和它们对 Web 堆栈前端的需求。

幸运的是，NGINX 最初不仅被构建为一个快速的静态文件服务器，还作为一个反向代理。这意味着，NGINX 从一开始就被设计为位于其他后端服务器前面，将请求分发到内部网络中的不同服务器，并将响应传递给最终用户。本章将介绍如何使用 NGINX 实现这一点的基础知识，以及 NGINX 能够做的一些更高级的功能，帮助我们更轻松地管理。

因此，我们将在本章中覆盖以下主要内容：

+   反向代理机制

+   NGINX 代理模块

+   NGINX 与微服务

# 探索反向代理机制

作为应用服务器运行 NGINX 有点类似于上一章描述的**FastCGI 架构**；我们将运行 NGINX 作为前端服务器，并大部分时间将请求反向代理到我们的后端服务器。

换句话说，它将与外部世界进行直接通信，而我们的后端服务器，无论是 Node.js、Apache 等，都只与 NGINX 交换数据：

![图 6.1：使用 Nginx 作为代理服务器的示例](img/B21787_06_1.jpg)

图 6.1：使用 Nginx 作为代理服务器的示例

现在有两个 Web 服务器在运行并处理请求。

NGINX 作为前端服务器（换句话说，作为反向代理）接收来自外部世界的所有请求。它会对请求进行过滤，要么直接将静态文件服务给客户端，要么将动态内容请求转发给我们的后端服务器。

我们的后端服务器仅与 NGINX 进行通信。它可能与前端托管在同一台计算机上，在这种情况下，监听端口必须进行编辑，以便将端口`80`和`443`留给 NGINX。或者，您也可以在不同的机器上使用多个后端服务器，并在它们之间进行负载均衡。

为了相互通信与交互，两个进程都不会使用 FastCGI。相反，正如名字所示，NGINX 充当一个简单的代理服务器；它接收来自客户端的 HTTP 请求（充当 HTTP 服务器），并将其转发到后端服务器（充当 HTTP 客户端）。因此，没有涉及新的协议或软件。此机制由 NGINX 的代理模块处理，稍后将在本章中详细介绍。

# 探索 NGINX 代理模块

类似于上一章，建立新架构的第一步是发现合适的模块。默认的 NGINX 构建包含了代理模块，允许将 HTTP 请求从客户端转发到后台服务器。我们将配置模块的多个方面：

+   后台服务器的基本地址和端口信息

+   缓存、缓冲和临时文件选项

+   限制、超时和错误行为

+   其他杂项选项

所有这些选项可以通过指令进行配置，我们将在本节中学习如何配置它们。

## 主要指令

第一组指令将允许你建立基本配置，例如后台服务器的位置、要传递的信息以及如何传递：

| **指令** | **描述** |
| --- | --- |

| `proxy_pass`上下文：`location`，`if` | 这指定请求应通过指示其位置转发到后台服务器：

+   对于常规的 HTTP 转发，语法是 `proxy_pass` http://hostname:port;。

+   对于 Unix 域套接字，语法是 `proxy_pass` http://unix:/path/to/file.socket;。

+   你也可以参考上游块 `proxy_pass` http://myblock;。

+   与 `http://` 不同，你可以使用 `https://` 来进行安全流量传输。还允许使用附加的 URI 部分以及变量。

示例：

+   `proxy_pass` http://localhost:8080;

+   `proxy_pass` http://127.0.0.1:8080;

+   `proxy_pass` http://unix:/tmp/nginx.sock;

+   `proxy_pass` https://192.168.0.1;

+   `proxy_pass` http://localhost:8080/uri/;

+   `proxy_pass` http://unix:/tmp/nginx.sock:/uri/;

+   `proxy_pass` http://$server_name:8080;

|

| `proxy_pass`上下文：`location`，`if` | 使用上游块：`upstream` `backend {``server 127.0.0.1:8080;``server 127.0.0.1:8081;``}``location ~* .``php$``{``proxy_pass` http://backend;`}` |
| --- | --- |
| `proxy_method`上下文：http，`server`，`location` | 这允许覆盖要转发到后台服务器的请求的 HTTP 方法。例如，如果指定`POST`，则所有转发到后台服务器的请求都将是`POST`请求。语法：`proxy_method method;` 示例：`proxy_method POST;` |
| `proxy_hide_header`上下文：`http`，`server`，`location` | 默认情况下，当 NGINX 准备将从后台服务器收到的响应转发回客户端时，它会忽略一些头部信息，如 `Date`，`Server`，`X-Pad` 和 `X-Accel-*`。使用此指令，你可以指定要隐藏的额外头部信息。你可以多次插入此指令，每次插入一个头部名称。语法：`proxy_hide_header header_name;` 示例：`proxy_hide_header Cache-Control;` |
| `proxy_pass_header`上下文：`http`，`server`，`location` | 与前一个指令相关，此指令强制将一些被忽略的头部传递给客户端。语法：`proxy_pass_header headername;` 示例：`proxy_pass_header Date;` |
| `proxy_pass_request_body``proxy_pass_request_headers`上下文：`http`，`server`，`location` | 这定义了是否传递请求主体和额外请求头到后端服务器。语法：`on`或`off`默认：`on` |

| `proxy_redirect`上下文：`http`，`server`，`location` | 这允许你重写由后端服务器触发的重定向中出现在 Location HTTP 头中的 URL。语法：`off`，`default`，或者你选择的 URL。`off`：重定向按原样转发。`default`：`proxy_pass`指令的值被用作主机名，并且当前文档路径被附加。注意，`proxy_redirect`指令必须在`proxy_pass`指令之后插入，因为配置是按顺序解析的。URL：用另一个 URL 替换 URL 的一部分。此外，你可以在重写的 URL 中使用变量。示例：

+   `proxy_redirect off;`

+   `proxy_redirect default;`

+   `proxy_redirect` `http://localhost:8080/ http://example.com/;`

+   `proxy_redirect` `http://localhost:8080/wiki/ /w/;`

+   `proxy_redirect` `http://localhost:8080/ http://$host/;`

|

| `proxy_next_upstream`上下文：`http`，`server`，`location` | 当`proxy_pass`连接到上游块时，此指令定义了在何种情况下应放弃请求并将其重发到块的下一个上游服务器。指令接受以下值的组合：

+   `error`：与服务器通信或尝试通信时发生错误

+   `timeout`：传输或连接尝试期间发生超时

+   `invalid_header`：后端服务器返回了空或无效的响应

+   `http_500`，`http_502`，`http_503`，`http_504`，`http_40`：如果发生此类 HTTP 错误，NGINX 将切换到下一个上游

+   `off`：禁止使用下一个上游服务器

示例：

+   `proxy_next_upstream error` `timeout http_504;`

+   `proxy_next_upstream` `timeout invalid_header;`

|

| `proxy_next_upstream_timeout`上下文：`http`，`server`，`location` | 这定义了与`proxy_next_upstream`一起使用的超时时间。将此指令设置为`0`将禁用它。语法：时间值（秒） |
| --- | --- |
| `proxy_next_upstream_tries`上下文：`http`，`server`，`location` | 这定义了在返回错误消息之前尝试的最大上游服务器数量，与`proxy_next_upstream`结合使用。语法：数值（默认：`0`） |

表 6.1：代理模块的主要指令

## 缓存、缓冲和临时文件

理想情况下，尽可能减少转发到后端服务器的请求数量。以下指令将帮助您构建缓存系统，并控制缓冲选项以及 NGINX 处理临时文件的方式：

| **指令** | **描述** |
| --- | --- |
| `proxy_buffer_size`上下文：`http`，`server`，`location` | 此指令设置用于读取后端服务器响应开头的缓冲区大小，通常包含简单的头部数据。默认值对应于前一个指令(`proxy_buffers`)定义的`1`个缓冲区大小。语法：数值（大小） 示例：`proxy_buffer_size 4k;` |
| `proxy_buffering, proxy_request_buffering`上下文：`http`，`server`，`location` | 该指令定义是否应缓冲来自后端服务器的响应数据（或者在`proxy_request_buffering`的情况下，缓冲客户端请求）。如果设置为`on`，NGINX 将使用缓冲区提供的内存空间存储响应数据。如果缓冲区已满，响应数据将作为临时文件存储。如果该指令设置为`off`，则响应会直接转发到客户端。语法：`on` 或 `off` 默认值：`on` |
| `proxy_buffers`上下文：`http`，`server`，`location` | 该指令设置用于从后端服务器读取响应数据的缓冲区数量和大小。语法：`proxy_buffers` `数量 大小;` 默认值：8 个缓冲区，每个缓冲区`4k`或`8k`，具体取决于平台示例：`fastcgi_buffers` `8 4k;` |
| `proxy_busy_buffers_size`上下文：`http`，`server`，`location` | 当后端接收的数据积累在缓冲区中并超过指定值时，缓冲区会被刷新，数据会发送到客户端。语法：数值（大小） 默认值：`2 *` `proxy_buffer_size` |
| `proxy_cache`上下文：`http`，`server`，`location` | 此指令定义一个缓存区域。给该区域指定的标识符将在后续的指令中重用。语法：`proxy_cache 区域名称;` 示例：`proxy_cache cache1;` |
| `proxy_cache_key`上下文：`http`，`server`，`location` | 此指令定义缓存键；换句话说，它区分了不同的缓存条目。如果缓存键设置为`$uri`，则所有带有`$uri`的请求将作为一个单独的缓存条目。但对于大多数动态网站来说，这还不够。您还需要将查询字符串参数包含在缓存键中，这样`/index.php`和`/index.php?page=contact`就不会指向同一个缓存条目。语法：`proxy_cache_key 键;` 示例：`proxy_cache_key "$scheme$host$request_uri $cookie_user";` |

| `proxy_cache_path`上下文：`http` | 此指令指定存储缓存文件的目录以及其他参数。语法：`proxy_cache_path 路径 [use_temp_path=on&#124;off] [levels=数字 keys_zone=名称:大小] [inactive=时间 max_size=大小];` 额外参数如下：

+   `use_temp_path`：如果您想使用通过`proxy_temp_path`指令定义的路径，请将此标志设置为`on`。

+   `levels`：此指令表示子目录的深度级别（通常*1:2*就足够了）。

+   `keys_zone`：此指令允许您使用之前通过`proxy_cache`指令声明的区域，并指示要占用的内存大小。

+   `inactive`：如果缓存的响应在指定时间内未被使用，则将其从缓存中移除。

+   `max_size`：该指令定义了整个缓存的最大大小。

示例：`proxy_cache_path /tmp/nginx_cache levels=1:2 zone=zone1:10m` `inactive=10m max_size=200M;` |

| `proxy_cache_methods`上下文：`http`，`server`，`location` | 该指令定义了符合缓存条件的 HTTP 方法。默认情况下，`GET`和`HEAD`方法包含在内，并且不能禁用。语法：`proxy_cache_methods METHOD;` 示例：`proxy_cache_methods OPTIONS;` |
| --- | --- |
| `proxy_cache_min_uses`上下文：`http`，`server`，`location` | 该指令定义了请求符合缓存条件的最小访问次数。默认情况下，请求的响应在第一次访问后会被缓存（后续具有相同缓存键的请求将返回缓存响应）。语法：数字值 示例：`proxy_cache_min_uses 1;` |

| `proxy_cache_valid`上下文：`http`，`server`，`location` | 该指令允许您为不同类型的响应代码自定义缓存时间。您可以将与`404`错误代码相关的响应缓存`1`分钟，反之将`200 OK`响应缓存`10`分钟或更长时间。此指令可以多次插入：

+   `proxy_cache_valid` `404 1m;`

+   `proxy_cache_valid 500` `502` `504 5m;`

+   `proxy_cache_valid` `200 10;`

语法：`proxy_cache_valid code1 [code2...] time;` |

| `proxy_cache_use_stale`上下文：`http`，`server`，`location` | 该指令定义了在某些情况下（关于网关）NGINX 是否应该提供过时的缓存数据。如果使用了`proxy_cache_use_stale timeout`，且网关超时，则 NGINX 将提供缓存的数据。语法：`proxy_cache_use_stale [updating] [error] [timeout] [`invalid_header] [http_500];` 示例：`proxy_cache_use_stale` `error timeout;` |
| --- | --- |
| `proxy_max_temp_file_size`上下文：`http`，`server`，`location` | 设置此指令为`0`以禁用代理转发请求使用临时文件，或者指定最大文件大小。语法：大小值 默认值：1 GB 示例：`proxy_max_temp_file_size 5m;` |
| `proxy_temp_file_write_size`上下文：`http`，`server`，`location` | 该指令设置了将临时文件写入存储设备时的缓冲区大小。语法：大小值 默认值：`2 *` `proxy_buffer_size` |

| `proxy_temp_path`上下文：`http`，`server`，`location` | 该指令设置临时和缓存文件的存储路径。语法：`proxy_temp_path path [level1 [level2...]]` 示例：

+   `proxy_temp_path /tmp/nginx_proxy;`

+   `proxy_temp_path /tmp/cache` `1 2;`

|

表 6.2：代理模块的缓存、缓冲和临时文件指令

## 限制、超时和错误

以下指令将帮助您定义超时行为以及与后端服务器通信时的各种限制：

| **指令** | **描述** |
| --- | --- |
| `proxy_connect_timeout`上下文: `http`, `server`, `location` | 该指令定义了后端服务器的连接超时。与读取/发送超时不同。如果 NGINX 已经与后端服务器连接，则`proxy_connect_timeout`不适用。语法: `时间值`（单位：秒）示例: `proxy_connect_timeout 15;` |
| `proxy_read_timeout`上下文: `http`, `server`, `location` | 这是从后端服务器读取数据的超时。此超时不会应用于整个响应延迟，而是应用于两个读取操作之间。语法: `时间值`（单位：秒） 默认值: `60` 示例: `proxy_read_timeout 60;` |
| `proxy_send_timeout`上下文: `http`, `server`, `location` | 此超时用于向后端服务器发送数据。超时不会应用于整个响应延迟，而是应用于两个写操作之间。语法: `时间值`（单位：秒） 默认值: `60` 示例: `proxy_send_timeout 60;` |
| `proxy_ignore_client_abort`上下文: `http`, `server`, `location` | 如果设置为`on`，即使客户端中止请求，NGINX 也将继续处理代理请求。在另一种情况下（`off`），当客户端中止请求时，NGINX 也会中止对后端服务器的请求。默认值: `off` |
| `proxy_intercept_errors`上下文: `http`, `server`, `location` | 默认情况下，NGINX 将所有后端服务器发送的错误页面（HTTP 状态码`400`及以上）直接返回给客户端。如果将此指令设置为`on`，错误代码将被解析，并可与`error_page`指令中指定的值进行匹配。默认值: `off` |
| `proxy_send_lowat`上下文: `http`, `server`, `location` | 此选项允许你在仅支持 BSD 操作系统的 TCP 套接字上使用`SO_SNDLOWAT`标志。此值定义了输出操作的缓冲区中的最小字节数。语法: 数值（大小） 默认值: `0` |
| `proxy_limit_rate`上下文: `http`, `server`, `location` | 此指令允许你限制 NGINX 从后端代理下载响应的速率。语法: 数值（字节/秒） |

表 6.3：与限制、超时和错误相关的指令

## 其他指令

最后，代理模块中最后一组指令是未分类的，内容如下：

| **指令** | **描述** |
| --- | --- |
| `proxy_headers_hash_max_size`上下文: `http`, `server`, `location` | 该指令设置代理头哈希表的最大大小。语法: 数值 默认值: `512` |
| `proxy_headers_hash_bucket_size`上下文: `http`, `server`, `location` | 该指令设置代理头哈希表的桶大小。语法: 数值 默认值: `64` |
| `proxy_force_ranges`上下文: `http`, `server`, `location` | 当设置为`on`时，NGINX 将启用字节范围支持，允许从后端代理返回的响应支持字节范围。语法: `on` 或 `off` 默认值: `off` |
| `proxy_ignore_headers`上下文：`http`、`server`、`location` | 该指令阻止 NGINX 处理后端服务器响应中的以下四个头部之一：`X-Accel-Redirect`、`X-Accel-Expires`、`Expires` 和 `Cache-Control`。语法：`proxy_ignore_headers` `header1 [header2...];` |
| `proxy_set_body`上下文：`http`、`server`、`location` | 此指令允许你为调试目的设置静态请求体。可以在指令值中使用变量。语法：字符串值（任何值）示例：`proxy_set_body test;` |
| `proxy_set_header`上下文：`http`、`server`、`location` | 此指令允许你重新定义传递给后端服务器的头部值。可以多次声明。语法：`proxy_set_header` `Header Value;` 示例：`proxy_set_header` `Host $host;` |

| `proxy_store`上下文：`http`、`server`、`location` | 此指令指定是否将后端服务器的响应存储为文件。存储的响应文件可以在为其他请求提供服务时重用。可能的值为`on`、`off`，或相对于文档根目录（或别名）的路径。你也可以将此指令设置为`on`并定义`proxy_temp_path`指令。示例：

+   `proxy_store on;`

+   `proxy_temp_path /temp/store;`

|

| `proxy_store_access`上下文：`http`、`server`、`location` | 此指令定义存储的响应文件的文件访问权限。语法：`proxy_store_access [user:[r&#124;w&#124;rw]][group:[r&#124;w&#124;rw]][all:[r&#124;w&#124;rw]];` 示例：`proxy_store_access user:rw` `group:rw all:r;` |
| --- | --- |
| `proxy_http_version`上下文：`http`、`server`、`location` | 此指令设置与代理后端通信时使用的 HTTP 版本。HTTP `1.0` 是默认值，但如果你想启用长连接，可能希望将此指令设置为 `1.1`。语法：`proxy_http_version 1.0 &#124;` `1.1;` |

| `proxy_cookie_domain``proxy_cookie_path`上下文：`http`、`server`、`location` | 这会对 cookie 的域名或路径属性进行动态修改（不区分大小写）。语法：

+   `proxy_cookie_domain off &#124;` `domain replacement;`

+   `proxy_cookie_path off &#124; domain` `replacement ;`

|

表 6.4：代理模块的未分类指令

## 变量

代理模块提供了多个变量，可以插入到各种位置，例如在 `proxy_set_header` 指令中，或日志相关的指令（如 `log_format`）中。可用的变量如下：

+   `$proxy_host`：此变量包含用于当前请求的后端服务器的主机名。

+   `$proxy_port`：此变量包含用于当前请求的后端服务器端口。

+   `$proxy_add_x_forwarded_for`：此变量包含 `X-Forwarded-For` 请求头的值，后面跟着客户端的远程地址。两者之间用逗号分隔。如果 `X-Forwarded-For` 请求头不可用，则此变量只包含客户端的远程地址。

+   `$proxy_internal_body_length`：请求体的长度（通过`proxy_set_body`指令设置，或者设置为`0`）。

尽管目前提到的指令和变量仅供参考，但我们将在接下来的章节中积极应用它们，探索它们在负载均衡等场景中的实际用途。

# 查看 NGINX 与微服务的关系

现在我们已经深入探讨了代理模块，是时候看看现代 Web 应用程序架构可能是什么样的了。这个主题有专门的书籍介绍，但我们只需要了解 NGINX 如何支持各种设置，并且 NGINX 部分在不同设置之间并没有太大差异。

对于任何我们需要应用程序完成的任务，我们有两个选择：可以选择将请求代理到像 Node.js 这样的后端服务器，让它处理工作，或者直接在 NGINX 中实现。你选择哪种方案取决于许多因素，但主要考虑的两个因素是速度和复杂性。

代理到一个复杂的后端服务器会有额外的开销，但通常可以实现代码复用，并且可以使用像 Packagist 和 NPM 这样的包管理器。相反，在 NGINX 中实现一个功能则使我们更接近用户，从而减少了开销，但开发本身也变得更加困难。

大多数设置会选择将请求代理到后端，以简化操作。在 NGINX 实现的一个功能示例中，可以看到 Cloudflare 及其代理/CDN 服务。由于他们处理着大量请求且响应时间对他们至关重要，所以他们直接在 NGINX 中实现了自己的安全过滤（Web 应用防火墙），通过一个模块来为 NGINX 配置文件添加 Lua 支持。

Cloudflare 拥有数百名开发人员，其中包括曾参与 NGINX 核心代码开发的人，因此不要指望能够完全达到他们的水平，但也有一些较简单的场景，NGINX 可以实现部分应用程序逻辑。

在 NGINX 中实现应用程序逻辑的一个简单示例是将我们的缓存从后端服务器迁移到 NGINX 本身。在以下示例中，我们首先检查 Memcached 中是否有页面的缓存版本，只有当我们没有找到缓存时，才会代理到应用程序后端：

```
# Check cache and use PHP as a fallback.
location ~* \.php$ {
default_type text/html;
charset utf-8;
if ($request_method = GET) {
set $memcached_key $request_uri;
memcached_pass memcached;
error_page 404 502 = @nocache;
}
if ($request_method != GET) {
fastcgi_pass backend;
}
}
location @nocache {
fastcgi_pass backend;
}
```

当我们进入更复杂的逻辑时，NGINX 的配置会变得有些复杂。在下一章中，我们将更详细地探讨几种代理配置场景。

# 总结

在本章中，我们了解了反向代理的工作原理以及 NGINX 如何适应现代微服务和复杂 Web 应用程序的架构，既包括使微服务架构得以实现，也包括将应用程序逻辑直接构建到 NGINX 中的方式。

本章应该让你了解了 NGINX 作为应用服务器所提供的可能性，并且希望阐明了在 NGINX 中实现逻辑时所涉及的复杂性与速度之间的权衡。

现在我们已经了解了 NGINX 所提供的各种可能性，接下来我们将把所学的知识付诸实践。在下一章中，我们将讨论一个具体的案例：使用 NGINX 代理与 Docker 的结合。

# 第三部分：NGINX 实战

在本书的最后部分，您将探讨如何将 NGINX 集成到更大的 IT 基础架构中，包括先进的部署策略、云环境以及自动化管理。本部分聚焦于 NGINX 在复杂系统中的实际应用案例，包括负载均衡、云部署以及如何保持高可用性和安全性。

本部分包含以下章节：

+   *第七章**，负载均衡与优化简介*

+   *第八章**，NGINX 在云基础架构中的应用*

+   *第九章**，使用 Ansible 完全部署、管理和自动更新 NGINX*

+   *第十章**，案例研究*

+   *第十一章**，故障排除*
