# 附录 A. 指令参考

本附录列出了全书中使用的配置指令。还有一些指令在书中没有出现，但为了完整性，这里列出。条目已扩展，展示了每个指令可以使用的上下文。如果指令有默认值，也已列出。这些指令适用于 NGINX 版本 1.3.9。最新的指令列表可以在[`nginx.org/en/docs/dirindex.html`](http://nginx.org/en/docs/dirindex.html)找到。

# 表：指令参考

| 指令 | 解释 | 上下文/默认值 |
| --- | --- | --- |
| `accept_mutex` | 在新连接的 `accept()` 方法上进行序列化，以便由工作进程处理。 | 有效上下文：`events` 默认值：`on` |
| `accept_mutex_delay` | 如果另一个工作进程已经在接受新连接，当前工作进程等待接受新连接的最大时间。 | 有效上下文：`events` 默认值：`500ms` |
| `access_log` | 描述了访问日志应该写入的位置和方式。第一个参数是日志存储文件的路径。构建路径时可以使用变量。特殊值 `off` 可禁用访问日志。可选的第二个参数表示写入日志时将使用的 `log_format`。如果未配置第二个参数，则使用预定义的组合格式。可选的第三个参数表示如果使用写入缓冲区来记录日志时，缓冲区的大小。如果使用写入缓冲区，则此大小不能超过该文件系统的原子磁盘写入大小。 | 有效上下文：`http`，`server`，`location`，`if in location`，`limit_except` 默认值：`logs`/`access.log combined` |
| `add_after_body` | 在响应体之后添加处理子请求的结果。 | 有效上下文：`location` 默认值：- |
| `add_before_body` | 在响应体之前添加处理子请求的结果。 | 有效上下文：`location` 默认值：- |
| `add_header` | 向响应中包含 HTTP 代码 200、204、206、301、302、303、304 或 307 的头部添加字段。 | 有效上下文：`http`，`server`，`location` 默认值：- |
| `addition_types` | 列出了除了 `text/html` 之外，响应中会进行附加的 MIME 类型。可以设置为 `*` 以启用所有 MIME 类型。 | 有效上下文：`http`，`server`，`location` 默认值：`text/html` |
| `aio` | 该指令启用异步文件 I/O 的使用。它适用于所有现代版本的 FreeBSD 和 Linux 发行版。在 FreeBSD 上，`aio` 可用于预加载数据以供 `sendfile` 使用。在 Linux 上，需要使用 `directio`，该选项会自动禁用 `sendfile`。 | 有效上下文：`http`，`server`，`location` 默认值：`off` |
| `alias` | 定义 `location` 的另一个名称，如在文件系统中所找到的。如果位置使用正则表达式指定，则别名应引用该正则表达式中定义的捕获组。 | 有效上下文：`location` 默认值：- |
| `allow` | 允许来自此 IP 地址、网络或 `all` 的访问。 | 有效上下文：`http`、`server`、`location`、`limit_except` 默认值：- |
| `ancient_browser` | 指定一个或多个字符串，如果在 `User-Agent` 头部中找到这些字符串，则表明浏览器被认为是过时的，通过将 `$ancient_browser` 变量设置为 `ancient_browser_value` 指令的值。 | 有效上下文：`http`、`server`、`location` 默认值：- |
| `ancient_browser_value` | 设置 `$ancient_browser` 变量的值。 | 有效上下文：`http`、`server`、`location` 默认值：`1`。 |
| `auth_basic` | 启用 HTTP 基本认证。参数字符串用于作为领域名称。如果使用特殊值 `off`，则表示否定父配置级别的 `auth_basic` 值。 | 有效上下文：`http`、`server`、`location`、`limit_except` 默认值：`off`。 |
| `auth_basic_user_file` | 指定用于验证用户的 `username:password:comment` 元组文件的位置。`password` 需要用 crypt 算法加密。评论是可选的。 | 有效上下文：`http`、`server`、`location`、`limit_except` 默认值：- |
| `auth_http` | 此指令指定用于验证 POP3/IMAP 用户的服务器。 | 有效上下文：`mail`、`server` 默认值：- |
| `auth_http_header` | 将额外的头部（第一个参数）设置为指定的值（第二个参数）。 | 有效上下文：`mail`、`server` 默认值：- |
| `auth_http_timeout` | NGINX 与认证服务器通信时的最大等待时间。 | 有效上下文：`mail`、`server` 默认值：`60s` |
| `autoindex` | 启用自动生成目录列表页面。 | 有效上下文：`http`、`server`、`location` 默认值：`off` |
| `autoindex_exact_size` | 指示目录列表页面中的文件大小是否应以字节显示，或四舍五入为千字节、兆字节和千兆字节。 | 有效上下文：`http`、`server`、`location`。 默认值：`on` |
| `autoindex_localtime` | 设置目录列表页中的文件修改时间为本地时间（`on`）或 UTC（`off`）。 | 有效上下文：`http`、`server`、`location` 默认值：`off` |
| `break` | 结束同一上下文内 `rewrite` 模块指令的处理。 | 有效上下文：`server`、`location`、`if` 默认值：- |
| `charset` | 将指定的字符集添加到 `Content-Type` 响应头中。如果与 `source_charset` 指令不同，将执行转换。 | 有效上下文：`http`、`server`、`location`、`if in location` 默认值：`off` |
| `charset_map` | 设置从一种字符集到另一种字符集的转换表。每个字符代码以十六进制表示。文件 `conf/koi-win`、`conf/koi-utf` 和 `conf/win-utf` 包含从 `koi8-r` 到 `windows-1251`、从 `koi8-r` 到 `utf-8` 和从 `windows-1251` 到 `utf-8` 的映射。 | 有效上下文：`http` 默认值：- |
| `charset_types` | 列出除 `text/html` 之外的响应 MIME 类型，在这些类型中将进行字符集转换。可以设置为 `*`，以启用所有 MIME 类型。 | 有效上下文：`http`、`server`、`location` 默认值：`text/html`、`text/xml`、`text/plain`、`text/vnd.wap.wml`、`application/x-javascript`、`application/rss+xml` |
| `chunked_transfer_encoding` | 允许在响应客户端时禁用标准的 HTTP/1.1 分块传输编码。 | 有效上下文：`http`、`server`、`location` 默认值：`on` |
| `client_body_buffer_size` | 用于设置大于默认两页内存的客户端请求体缓冲区大小，以防止临时文件写入磁盘。 | 有效上下文：`http`、`server`、`location` 默认值：`8k&#124;16k`（平台相关） |
| `client_body_in_file_only` | 用于调试或进一步处理客户端请求体，此指令可以设置为 `on`，强制将客户端请求体保存到文件中。`clean` 值将导致请求处理完成后删除文件。 | 有效上下文：`http`、`server`、`location` 默认值：`off` |
| `client_body_in_single_buffer` | 此指令将强制 NGINX 将整个客户端请求体保存在一个单独的缓冲区中，以减少复制操作。 | 有效上下文：`http`、`server`、`location` 默认值：`off` |
| `client_body_temp_path` | 定义保存客户端请求体的目录路径。如果提供第二、第三或第四个参数，这些参数指定了子目录层次结构，并且参数值表示子目录名中的字符数。 | 有效上下文：`http`、`server`、`location` 默认值：`client_body_temp` |
| `client_body_timeout` | 指定客户端请求体的连续读取操作之间的时间长度。如果达到此时间限制，客户端将收到 408 错误信息（`请求超时`）。 | 有效上下文：`http`、`server`、`location` 默认值：`60s` |
| `client_header_buffer_size` | 用于指定客户端请求头的缓冲区大小，当需要大于默认的 1 KB 时使用。 | 有效上下文：`http`、`server` 默认值：`1k` |
| `client_header_timeout` | 指定读取完整客户端请求头的时间长度。如果达到此时间限制，客户端将收到 408 错误信息（`请求超时`）。 | 有效上下文：`http`、`server` 默认值：`60s` |
| `client_max_body_size` | 定义允许的最大客户端请求体大小，超过该大小会返回 413（`请求实体太大`）错误到浏览器。 | 有效上下文：`http`、`server`、`location` 默认值：`1m` |
| `connection_pool_size` | 精细调整每个连接的内存分配。 | 有效上下文: `http`，`server` 默认值: `256` |
| `create_full_put_path` | 在使用 WebDAV 时允许递归创建目录。 | 有效上下文: `http`，`server`，`location` 默认值: `off` |
| `daemon` | 设置是否将 `nginx` 进程后台化。 | 有效上下文: `main` 默认值: `on` |
| `dav_access` | 设置新创建的文件和目录的文件系统访问权限。如果指定了 `group` 或 `all`，则可以省略 `user`。 | 有效上下文: `http`，`server`，`location` 默认值: `user:rw` |
| `dav_methods` | 允许指定的 HTTP 和 WebDAV 方法。当使用 `PUT` 时，首先会创建一个临时文件，然后将其重命名。因此，建议将 `client_body_temp_path` 放在与目标相同的文件系统上。此类文件的修改日期可以在 `Date` 头中指定。 | 有效上下文: `http`，`server`，`location` 默认值: `off` |
| `debug_connection` | 启用调试日志记录，适用于与该指令值匹配的任何客户端。可以多次指定此项。要调试 UNIX 域套接字，请使用 `unix:`。 | 有效上下文: `events` 默认值: - |
| `debug_points` | 在调试时，进程将创建一个核心文件（`abort`）或停止（`stop`），以便可以附加系统调试器。 | 有效上下文: `main` 默认值: - |
| `default_type` | 设置响应的默认 MIME 类型。如果文件的 MIME 类型无法与 `types` 指令中指定的类型匹配，则此项生效。 | 有效上下文: `http`，`server`，`location` 默认值: `text/plain` |
| `deny` | 拒绝来自此 IP 地址、网络或 `all` 的访问。 | 有效上下文: `http`，`server`，`location`，`limit_except` 默认值: - |
| `directio` | 启用操作系统特定的标志或功能，用于处理比给定参数大的文件。使用 Linux 上的 `aio` 时需要此项。 | 有效上下文: `http`，`server`，`location` 默认值: `off` |
| `directio_alignment` | 设置 `directio` 的对齐方式。默认值为 `512`，通常足够，但建议在 Linux 上使用 XFS 时将其增加到 `4K`。 | 有效上下文: `http`，`server`，`location` 默认值: `512` |
| `disable_symlinks` | 请参见第六章中*查找文件*部分的*HTTP 文件路径指令*表格，*NGINX HTTP 服务器*。 | 有效上下文: `http`，`server`，`location` 默认值: `off` |
| `empty_gif` | 为该 `location` 生成一个 1x1 像素的透明 GIF。 | 有效上下文: `location` 默认值: - |

| `env` | 设置用于以下环境变量：

+   现场升级时的继承

+   使其在 `perl` 模块中可用

+   使其可供工作进程使用

单独指定变量将使用 `nginx` 环境中找到的值。设置变量可以采用 `var=value` 的形式。注意：`NGINX` 是一个内部变量，不应由用户设置。 | 有效上下文：`main`默认值：`TZ` |

| `error_log` | `error_log` 文件是记录所有错误的地方。它可以设置为文件或 `stderr`。如果在其他上下文中未提供 `error_log`，则此日志文件将用于记录所有错误，作用于全局。此指令的第二个参数表示错误将以哪种级别（`debug`、`info`、`notice`、`warn`、`error`、`crit`、`alert`、`emerg`）记录到日志中。请注意，只有在编译时使用了 `--with-debug` 配置开关时，`debug` 级别的错误才会生效。 | 有效上下文：`main`、`http`、`server`、`location`默认值：`logs/error.log error` |
| --- | --- | --- |
| `error_page` | 定义遇到错误级响应代码时提供的 URI。添加 `=` 参数允许更改响应代码。如果此参数的值为空，则响应代码将从 URI 中获取，且该 URI 必须由某种上游服务器提供。 | 有效上下文：`http`、`server`、`location`、`if in location`默认值：- |
| `etag` | 禁用自动为静态资源生成 `ETag` 响应头。 | 有效上下文：`http`、`server`、`location`默认值：`on` |
| `events` | 定义一个新上下文，用于指定连接处理指令。 | 有效上下文：`main`默认值：- |
| `expires` | 参见《第七章》中 *文件系统缓存* 部分的 *头部修改指令* 表。 | 有效上下文：`http`、`server`、`location`默认值：`off` |
| `fastcgi_bind` | 指定应使用哪个地址来进行与 FastCGI 服务器的外向连接。 | 有效上下文：`http`、`server`、`location`默认值：- |
| `fastcgi_buffer_size` | 用于从 FastCGI 服务器接收响应的第一部分的缓冲区大小，该部分包含响应头。 | 有效上下文：`http`、`server`、`location`默认值：`4k&#124;8k`（平台相关） |
| `fastcgi_buffers` | 为单个连接使用的 FastCGI 服务器响应的缓冲区数量和大小。 | 有效上下文：`http`、`server`、`location`默认值：`4k`&#124;`8k`（平台相关） |
| `fastcgi_busy_buffers_size` | 分配给发送响应给客户端的缓冲区空间的总大小，同时响应仍从 FastCGI 服务器读取。通常设置为两个 `fastcgi_buffers` 的大小。 | 有效上下文：`http`、`server`、`location`默认值：`4k&#124;8k`（平台相关） |
| `fastcgi_cache` | 定义用于缓存的共享内存区域。 | 有效上下文：`http`、`server`、`location`默认值：`off` |
| `fastcgi_cache_bypass` | 一个或多个字符串变量，当其非空或非零时，将导致响应从 FastCGI 服务器获取，而不是从缓存中获取。 | 有效上下文：`http`，`server`，`location` 默认值：- |
| `fastcgi_cache_key` | 用作存储和检索缓存值的字符串键。 | 有效上下文：`http`，`server`，`location` 默认值：- |
| `fastcgi_cache_lock` | 启用此指令将防止多个请求同时在相同缓存键中创建条目。 | 有效上下文：`http`，`server`，`location` 默认值：`off` |
| `fastcgi_cache_lock_timeout` | 请求等待缓存条目出现或`fastcgi_cache_lock`释放的时间长度。 | 有效上下文：`http`，`server`，`location` 默认值：`5s` |
| `fastcgi_cache_min_uses` | 对于某个键，在缓存响应之前，所需的请求次数。 | 有效上下文：`http`，`server`，`location` 默认值：`1` |
| `fastcgi_cache_path` | 请参考第六章的《*使用 NGINX 与 PHP-FPM*》部分中的*FastCGI 指令*表。 | 有效上下文：`http` 默认值：- |
| `fastcgi_cache_use_stale` | 在访问 FastCGI 服务器时出现错误时，何种情况下可以接受提供过期缓存数据。`updating`参数表示正在加载新数据的情况。 | 有效上下文：`http`，`server`，`location` 默认值：`off` |
| `fastcgi_cache_valid` | 指示缓存响应的有效时间，响应代码为 200、301 或 302 时有效。如果在时间参数前给定可选的响应代码，则该时间仅适用于该响应代码。特殊参数`any`表示任何响应代码应缓存该时长。 | 有效上下文：`http`，`server`，`location` 默认值：- |
| `fastcgi_connect_timeout` | NGINX 在请求 FastCGI 服务器时，等待连接被接受的最大时间。 | 有效上下文：`http`，`server`，`location` 默认值：`60s` |
| `fastcgi_hide_header` | 一些不应传递给客户端的头部字段列表。 | 有效上下文：`http`，`server`，`location` 默认值：- |
| `fastcgi_ignore_client_abort` | 如果设置为`on`，NGINX 将在客户端中止连接时，不中止与 FastCGI 服务器的连接。 | 有效上下文：`http`，`server`，`location` 默认值：`off` |
| `fastcgi_ignore_headers` | 设置处理 FastCGI 服务器响应时可以忽略的头部字段。 | 有效上下文：`http`，`server`，`location` 默认值：- |
| `fastcgi_index` | 设置要附加到`$fastcgi_script_name`的文件名，文件名以斜杠结尾。 | 有效上下文：`http`，`server`，`location` 默认值：- |
| `fastcgi_intercept_errors` | 如果启用，NGINX 将显示配置的 `error_page` 指令，而不是直接从 FastCGI 服务器返回响应。 | 有效上下文：`http`、`server`、`location` 默认值：`off` |
| `fastcgi_keep_conn` | 通过指示服务器不要立即关闭连接，启用与 FastCGI 服务器的 `keepalive` 连接。 | 有效上下文：`http`、`server`、`location` 默认值：`off` |
| `fastcgi_max_temp_file_size` | 当响应不能完全放入内存缓冲区时，溢出文件的最大大小。 | 有效上下文：`http`、`server`、`location` 默认值：`1024m` |
| `fastcgi_next_upstream` | 请参考 *FastCGI 指令* 表格，位于 第六章 *使用 NGINX 与 PHP-FPM* 部分，*NGINX HTTP 服务器*。 | 有效上下文：`http`、`server`、`location` 默认值：错误超时 |
| `fastcgi_no_cache` | 一个或多个字符串变量，当其值非空或非零时，指示 NGINX 不将 FastCGI 服务器的响应缓存。 | 有效上下文：`http`、`server`、`location` 默认值：- |
| `fastcgi_param` | 设置一个参数及其值，传递给 FastCGI 服务器。如果该参数仅在值非空时传递，则应设置额外的 `if_not_empty` 参数。 | 有效上下文：`http`、`server`、`location` 默认值：- |
| `fastcgi_pass` | 指定请求传递给的 FastCGI 服务器，可以是 `address:port` 组合，也可以是 `unix:path` 用于 UNIX 域套接字。 | 有效上下文：`location`、`if in location` 默认值：- |
| `fastcgi_pass_header` | 覆盖 `fastcgi_hide_header` 中禁用的头部，允许它们发送给客户端。 | 有效上下文：`http`、`server`、`location` 默认值：- |
| `fastcgi_read_timeout` | 指定从 FastCGI 服务器进行两次连续读取操作之间需要经过的时间长度，超时后关闭连接。 | 有效上下文：`http`、`server`、`location` 默认值：`60s` |
| `fastcgi_send_lowat` | 这是一个 FreeBSD 指令。当设置为非零时，它会告知 NGINX 在与上游服务器通信时使用 `NOTE_LOWAT` kqueue 方法或指定大小的 `SO_SNDLOWAT` 套接字选项。在 Linux、Solaris 和 Windows 中会被忽略。 | 有效上下文：`http`、`server`、`location` 默认值：`0` |
| `fastcgi_send_timeout` | 在关闭连接之前，两次连续写操作到 FastCGI 服务器之间需要经过的时间长度。 | 有效上下文：`http`、`server`、`location` 默认值：`60s` |
| `fastcgi_split_path_info` | 定义一个包含两个捕获组的正则表达式。第一个捕获组将是 `$fastcgi_script_name` 变量的值。第二个捕获组将成为 `$fastcgi_path_info` 变量的值。 | 有效上下文：`location` 默认值：- |
| `fastcgi_store` | 启用将从 FastCGI 服务器获取的响应作为文件存储在磁盘上。`on` 参数将使用 `alias` 或 `root` 指令作为存储文件的基础路径。也可以给定一个字符串，表示存储文件的替代位置。 | 有效上下文：`http`、`server`、`location` 默认值：`off` |
| `fastcgi_store_access` | 设置新创建的 `fastcgi_store` 文件的访问权限。 | 有效上下文：`http`、`server`、`location` 默认值：`user:rw` |
| `fastcgi_temp_file_write_size` | 限制一次性缓冲到临时文件的数据量，以避免 NGINX 在单个请求上被阻塞过长时间。 | 有效上下文：`http`、`server`、`location` 默认值：`8k&#124;16k`（平台相关） |
| `fastcgi_temp_path` | 一个目录，用于缓冲从 FastCGI 服务器代理过来的临时文件，可能是多级深层。如果给定第二、第三或第四个参数，这些参数指定一个子目录层次，其中子目录名称的字符数由参数值决定。 | 有效上下文：`http`、`server`、`location` 默认值：`fastcgi_temp` |
| `flv` | 为此 `location` 激活 `flv` 模块。 | 有效上下文：`location` 默认值：- |

| `geo` | 定义一个新的上下文，在该上下文中，变量将被设置为基于另一个变量中的 IP 地址的指定值。如果没有指定其他变量，则使用 `$remote_addr` 来确定 IP 地址。上下文定义的格式为：

```
geo [$address-variable] $variable-to-be-set { … }
```

以下参数在上下文中被识别：

+   `delete`：删除指定的网络

+   `default`：如果没有 IP 地址匹配，变量将设置为此值

+   `include`：包含一个地址到值的映射文件

+   `proxy`：定义一个直接连接的地址或网络，从 `X-Forwarded-For` 头部中提取 IP 地址

+   `proxy_recursive`：与 `proxy` 配合使用，指定在多值 `X-Forwarded-For` 头部中最后一个地址将被使用

+   `ranges`：当定义时，表示以下地址被指定为范围

| 有效上下文：`http` 默认值：- |
| --- |

| `geoip_city` | 包含 IP 地址到城市映射的 GeoIP 数据库文件的路径。接下来可用以下变量：

+   `$geoip_city_country_code`：两字母国家代码

+   `$geoip_city_country_code3`：三字母国家代码

+   `$geoip_city_country_name`：国家名称

+   `$geoip_region`：国家区域名称

+   `$geoip_city`：城市名称

+   `$geoip_postal_code`：邮政编码

| 有效上下文：`http` 默认值：- |
| --- |

| `geoip_country` | 包含 IP 地址到国家映射的 GeoIP 数据库文件的路径。接下来可用以下变量：

+   `$geoip_country_code`：两字母国家代码

+   `$geoip_country_code3`：三字母国家代码

+   `$geoip_country_name`：国家名称

| 有效上下文：`http` 默认值：- |
| --- |

| `geoip_org` | 包含 IP 地址到组织映射的 GeoIP 数据库文件的路径。然后，以下变量变得可用： |

+   `$geoip_org`: 组织名称

| 有效上下文：`http`。默认值：- |
| --- |
| `geoip_proxy` | 定义一个直接连接的地址或网络，从中提取 IP 地址，将从`X-Forwarded-For`头中获取。 | 有效上下文：`http`默认值：- |
| `geoip_proxy_recursive` | 与`geoip_proxy`一起工作，指定在多值`X-Forwarded-For`头中最后的地址将被使用。 | 有效上下文：`http`默认值：`off`。 |
| `gunzip` | 启用当客户端不支持 gzip 时解压缩 gzipped 文件。 | 有效上下文：`http`、`server`、`location`默认值：`off` |
| `gunzip buffers` | 指定用于解压缩响应的缓冲区数量和大小。 | 有效上下文：`http`、`server`、`location`默认值：`32 4k&#124;16 8k`（平台依赖） |
| `gzip` | 启用或禁用响应的压缩。 | 有效上下文：`http`、`server`、`location`、`if in location`默认值：`off` |
| `gzip_buffers` | 指定用于压缩响应的缓冲区数量和大小。 | 有效上下文：`http`、`server`、`location`默认值：`32 4k&#124;16 8k`（平台依赖） |
| `gzip_comp_level` | gzip 压缩级别（1-9）。 | 有效上下文：`http`、`server`、`location`默认值：`1` |
| `gzip_disable` | 一个不应接收压缩响应的`User-Agent`的正则表达式。特殊值`msie6`是`MSIE [4-6]\.`的简写，排除`MSIE 6.0; ... SV1`。 | 有效上下文：`http`、`server`、`location`默认值：- |
| `gzip_http_version` | 考虑压缩的请求的最小 HTTP 版本。 | 有效上下文：`http`、`server`、`location`默认值：`1.1` |
| `gzip_min_length` | 压缩考虑之前响应的最小长度，由`Content-Length`头确定。 | 有效上下文：`http`、`server`、`location`默认值：`20` |
| `gzip_proxied` | 请参阅第五章中的*压缩模块指令*表，*反向代理高级话题*。 | 有效上下文：`http`、`server`、`location`默认值：`off` |
| `gzip_static` | 启用检查预压缩文件，直接交付给支持 gzip 压缩的客户端。 | 有效上下文：`http`、`server`、`location`默认值：`off` |
| `gzip_types` | 应该使用 gzip 压缩的 MIME 类型，除了默认的`text/html`之外。它可以是`*`以启用所有 MIME 类型。 | 有效上下文：`http`、`server`、`location`默认值：`text/html` |
| `gzip_vary` | 启用或禁用响应头`Vary: Accept-Encoding`，如果`gzip`或`gzip_static`处于活动状态。 | 有效上下文：`http`、`server`、`location`默认值：`off` |
| `http` | 设置一个配置上下文，用于指定 HTTP 服务器指令。 | 有效上下文：`main` 默认值：- |
| `if` | 请参见附录 B 中的*重写模块指令*表格，位于*重写模块介绍*一节。 | 有效上下文：`server`，`location` 默认值：- |

| `if_modified_since` | 控制响应的修改时间如何与`If-Modified-Since`请求头的值进行比较：

+   `off`：忽略`If-Modified-Since`头部

+   `exact`：进行精确匹配（默认）

+   `before`：响应的修改时间小于或等于`If-Modified-Since`头部的值

| 有效上下文：`http`，`server`，`location` 默认值：`exact` |
| --- |
| `ignore_invalid_headers` | 禁用忽略无效名称的头部。有效名称由 ASCII 字母、数字、连字符和可能的下划线组成（由`underscores_in_headers`指令控制）。 | 有效上下文：`http`，`server` 默认值：`on` |
| `image_filter` | 请参见第七章中的*图像过滤指令*表格，位于*开发者的 NGINX*一节。 | 有效上下文：`location` 默认值：- |
| `image_filter_buffer` | 用于处理图像的缓冲区大小。如果需要更多内存，服务器将返回 415 错误（`不支持的媒体类型`）。 | 有效上下文：`http`，`server`，`location` 默认值：`1M` |
| `image_filter_jpeg_quality` | 处理后的 JPEG 图像质量。建议不要超过 95。 | 有效上下文：`http`，`server`，`location` 默认值：`75` |
| `image_filter_sharpen` | 通过此百分比增加处理后图像的锐度。 | 有效上下文：`http`，`server`，`location` 默认值：`0` |
| `image_filter_transparency` | 禁用对转换后的 GIF 和 PNG 图像的透明度保护。默认值`on`表示保持透明度。 | 有效上下文：`http`，`server`，`location` 默认值：`on` |
| `imap_auth` | 设置支持的客户端认证机制。可以是`login`，`plain`或`cram-md5`之一或多个。 | 有效上下文：`mail`，`server` 默认值：`plain` |
| `imap_capabilities` | 指示后端服务器支持哪些 IMAP4 功能。 | 有效上下文：`mail`，`server` 默认值：`IMAP4 IMAP4rev1 UIDPLUS` |
| `imap_client_buffer` | 设置 IMAP 命令的读取缓冲区大小。 | 有效上下文：`mail`，`server` 默认值：`4k&#124;8k`（依平台而定） |
| `include` | 包含额外配置指令的文件路径。可以指定通配符以包含多个文件。 | 有效上下文：`any` 默认值：- |
| `index` | 定义接收到以`/`结尾的 URI 时，哪个文件将被提供给客户端。它可以是多个值。 | 有效上下文：`http`，`server`，`location` 默认值：`index.html` |
| `internal` | 指定只能用于内部请求的 `location`（在其他指令中定义的重定向、重写请求以及类似的请求处理指令）。 | 有效上下文：`location` 默认值：- |
| `ip_hash` | 通过对 IP 地址进行哈希处理，确保客户端在所有 `server` 之间均匀分布，哈希基于其 C 类网络。 | 有效上下文：`upstream` 默认值：- |
| `keepalive` | 每个工作进程缓存的与上游服务器的连接数。当与 HTTP 连接一起使用时，`proxy_http_version` 应设置为 `1.1`，并将 `proxy_set_header` 设置为 `Connection`。 | 有效上下文：`upstream` 默认值：- |
| `keepalive_disable` | 禁用某些浏览器类型的 keep-alive 请求。 | 有效上下文：`http`、`server`、`location` 默认值：`msie6` |
| `keepalive_requests` | 定义在关闭之前可以通过一个 `keepalive` 连接发出的请求数。 | 有效上下文：`http`、`server`、`location` 默认值：`100` |
| `keepalive_timeout` | 指定 `keep-alive` 连接保持打开的时间。可以提供第二个参数，以在响应中设置 Keep-Alive 头。 | 有效上下文：`http`、`server`、`location` 默认值：`75s` |
| `large_client_header_buffers` | 定义大型客户端请求头的最大 `number` 和 `size`。 | 有效上下文：`http`、`server` 默认值：`4 8k` |
| `least_conn` | 启用负载均衡算法，选择具有最少活动连接的服务器进行下一个新连接。 | 有效上下文：`upstream` 默认值：- |
| `limit_conn` | 指定一个共享内存区域（使用 `limit_conn_zone` 配置）和每个键值允许的最大连接数。 | 有效上下文：`http`、`server`、`location` 默认值：- |
| `limit_conn_log_level` | 当 NGINX 因为 `limit_conn` 指令限制了连接时，该指令指定在哪个日志级别报告该限制。 | 有效上下文：`http`、`server`、`location` 默认值：`error` |
| `limit_conn_zone` | 指定在 `limit_conn` 中作为第一个参数限制的键。第二个参数 `zone` 表示用于存储键及每个键当前连接数的共享内存区域的名称及该区域的大小（`name:size`）。 | 有效上下文：`http` 默认值：- |
| `limit_except` | 将 `location` 限制为指定的 HTTP 动词（`GET` 也包括 `HEAD`）。 | 有效上下文：`location` 默认值：- |
| `limit_rate` | 限制客户端下载内容的速率（以每秒字节数为单位）。速率限制作用于连接级别，这意味着单个客户端可以通过打开多个连接来提高其吞吐量。 | 有效上下文：`http`、`server`、`location`、`if in location` 默认值：`0` |
| `limit_rate_after` | 在传输了指定字节数后，启动 `limit_rate` 限制。 | 有效上下文：`http`、`server`、`location`、`if in location` 默认值：`0` |
| `limit_req` | 为共享内存存储区（通过 `limit_req_zone` 配置）中的特定键设置带有突发能力的请求限制。突发量可以通过第二个参数指定。如果在突发期间不应该有请求之间的延迟，则需要配置第三个参数 `nodelay`。 | 有效上下文：`http`、`server`、`location` 默认值：- |
| `limit_req_log_level` | 当 NGINX 由于 `limit_req` 指令限制请求数量时，此指令指定该限制被记录的日志级别。延迟将以低一个级别记录。 | 有效上下文：`http`、`server`、`location` 默认值：- |
| `limit_req_zone` | 在 `limit_req` 中指定需要限制的键作为第一个参数。第二个参数 `zone` 表示用于存储键及每个键当前请求数的共享内存区域的名称，以及该区域的大小（`name:size`）。第三个参数 `rate` 配置每秒（r/s）或每分钟（r/m）限制前的请求数。 | 有效上下文：`http` 默认值：- |
| `limit_zone` | 已废弃。请改用 `limit_conn_zone`。 | 有效上下文：`http` 默认值：- |
| `lingering_close` | 此指令指定客户端连接将在接收更多数据时保持开启。 | 有效上下文：`http`、`server`、`location` 默认值：`on` |
| `lingering_time` | 与 `lingering_close` 指令一起使用时，此指令将指定客户端连接将在接收更多数据时保持打开的时间。 | 有效上下文：`http`、`server`、`location` 默认值：`30s` |
| `lingering_timeout` | 与 `lingering_close` 一起使用时，此指令表示 NGINX 在关闭客户端连接前等待额外数据的时间。 | 有效上下文：`http`、`server`、`location` 默认值：`5s` |
| `listen` (http) | 请参阅第二章中 *虚拟服务器部分* 的 *listen 参数* 表格，*配置指南*。 | 有效上下文：`server` 默认值：`*:80 &#124; *:8000` |

| `listen` (mail) | `listen` 指令在 NGINX 中唯一标识一个套接字绑定。它接受以下参数：

+   `bind`：为这个 `address:port` 配对进行单独的 `bind()` 调用。

| 有效上下文：`server` 默认值：- |
| --- |
| `location` | 定义一个基于请求 URI 的新上下文。 | 有效上下文：`server`、`location` 默认值：- |
| `lock_file` | 锁文件的前缀名称。根据平台的不同，可能需要锁文件来实现 `accept_mutex` 和共享内存访问的序列化。 | 有效上下文：`main` 默认值：`logs/nginx.lock`。 |
| `log_format` | 指定哪些字段应出现在日志文件中，并设置其格式。 | 有效上下文：`http` 默认值：`combined "$remote_addr - $remote_user [$time_local], "$request" $status $body_bytes_sent, "$http_referer""$http_user_agent"'` |
| `log_not_found` | 禁用在错误日志中报告 404 错误。 | 有效上下文：`http`，`server`，`location` 默认值：`on` |
| `log_subrequest` | 启用访问日志中的子请求日志记录。 | 有效上下文：`http`，`server`，`location` 默认值：`off` |
| `mail` | 设置一个配置上下文，在该上下文中指定邮件服务器指令。 | 有效上下文：`main` 默认值：- |

| `map` | 定义一个新上下文，在该上下文中，变量的值根据源变量的值被设置为指定值。上下文定义的格式如下：

```
map $source-variable $variable-to-be-set { … }
```

这些要映射的字符串也可以是正则表达式。在该上下文中识别以下参数：

+   `default`：如果源变量的值没有匹配任何指定的字符串或正则表达式，则为变量设置一个默认值。

+   `hostnames`：表示源值可能是带有前缀或后缀通配符的主机名。

+   `include`：包含一个文件，其中包含字符串到值的映射。

| 有效上下文：`http` 默认值：- |
| --- |
| `map_hash_bucket_size` | 用于存储 `map` 哈希表的桶大小。 | 有效上下文：`http` 默认值：`32&#124;64&#124;128` |
| `map_hash_max_size` | `map` 哈希表的最大大小。 | 有效上下文：`http` 默认值：`2048` |
| `master_process` | 决定是否启动工作进程。 | 有效上下文：`main` 默认值：`on` |
| `max_ranges` | 设置字节范围请求中允许的最大范围数。指定 `0` 会禁用字节范围支持。 | 有效上下文：`http`，`server`，`location` 默认值：- |
| `memcached_bind` | 指定用于与 memcached 服务器建立连接的地址。 | 有效上下文：`http`，`server`，`location` 默认值：- |
| `memcached_buffer_size` | memcached 响应的缓冲区大小。此响应随后会同步发送到客户端。 | 有效上下文：`http`，`server`，`location` 默认值：`4k&#124;8k` |
| `memcached_connect_timeout` | NGINX 在向 memcached 服务器发起请求时，等待连接被接受的最长时间。 | 有效上下文：`http`，`server`，`location` 默认值：`60s` |
| `memcached_gzip_flag` | 指定一个值，当在 memcached 服务器的响应中找到该值时，会将 `Content-Encoding` 头设置为 `gzip`。 | 有效上下文：`http`，`server`，`location` 默认值：- |
| `memcached_next_upstream` | 请参见第七章中的 *Memcached 模块指令* 表，*开发者的 NGINX*。 | 有效上下文：`http`，`server`，`location` 默认值：错误超时 |
| `memcached_pass` | 指定一个 memcached 服务器的名称或地址及其端口。它也可以是一个在 `upstream` 上下文中声明的 `server` 组。 | 有效上下文：`location`、`if in location` 默认值：- |
| `memcached_read_timeout` | 指定在两个连续的读操作之间，需要经过的时间长度，超过该时间连接会被关闭。 | 有效上下文：`http`、`server`、`location` 默认值：`60s` |
| `memcached_send_timeout` | 在两个连续的写操作之间，需要经过的时间长度，超过该时间连接会被关闭。 | 有效上下文：`http`、`server`、`location` 默认值：`60s` |
| `merge_slashes` | 禁用去除多个斜杠。默认值为 `on`，表示 NGINX 会将两个或更多的 `/` 字符压缩成一个。 | 有效上下文：`http`、`server` 默认值：`on` |
| `min_delete_depth` | 允许 WebDAV `DELETE` 方法在请求路径中至少包含此数量的元素时删除文件。 | 有效上下文：`http`、`server`、`location` 默认值：`0` |
| `modern_browser` | 指定 `browser` 和 `version` 参数，二者一起表示通过将 `$modern_browser` 变量设置为 `modern_browser_value`，浏览器被认为是现代浏览器。`browser` 参数可以取以下值之一：`msie`、`gecko`、`opera`、`safari` 或 `konqueror`。可以指定 `unlisted` 参数，表示任何未在 `ancient_browser` 或 `modern_browser` 中找到的浏览器，或缺少 `User-Agent` 头部的浏览器，都会被认为是现代浏览器。 | 有效上下文：`http`、`server`、`location` 默认值：- |
| `modern_browser_value` | `$modern_browser` 变量将设置为的值。 | 有效上下文：`http`、`server`、`location` 默认值：`1` |
| `mp4` | 为该 `location` 启用 `mp4` 模块。 | 有效上下文：`location` 默认值：- |
| `mp4_buffer_size` | 设置用于传输 MP4 文件的初始缓冲区大小。 | 有效上下文：`http`、`server`、`location` 默认值：`512K` |
| `mp4_max_buffer_size` | 设置用于处理 MP4 元数据的缓冲区的最大大小。 | 有效上下文：`http`、`server`、`location` 默认值：`10M` |
| `msie_padding` | 允许禁用为状态大于 400 的响应添加注释，以便为 MSIE 客户端填充响应大小至 512 字节。 | 有效上下文：`http`、`server`、`location` 默认值：`on` |
| `msie_refresh` | 该指令允许为 MSIE 客户端发送 `refresh` 而非 `redirect`。 | 有效上下文：`http`、`server`、`location` 默认值：`off` |
| `multi_accept` | 指示工作进程一次性接受所有新的连接。如果使用 `kqueue` 事件方法，则会忽略此指令，因为 `kqueue` 会报告等待接受的新连接数。 | 有效上下文：`events` 默认值：`off` |
| `open_file_cache` | 配置一个缓存，可以存储打开的文件描述符、目录查找和文件查找错误。 | 有效的上下文：`http`，`server`，`location` 默认值：`off` |
| `open_file_cache_errors` | 启用`open_file_cache`指令缓存文件查找错误。 | 有效的上下文：`http`，`server`，`location` 默认值：`off` |
| `open_file_cache_min_uses` | 配置文件在`inactive`参数中的最小使用次数，以便该文件描述符在缓存中保持打开状态。 | 有效的上下文：`http`，`server`，`location` 默认值：`1` |
| `open_file_cache_valid` | 指定`open_file_cache`指令中项的有效性检查之间的时间间隔。 | 有效的上下文：`http`，`server`，`location` 默认值：`60s` |
| `open_log_file_cache` | 请参阅*HTTP 日志指令*表格，位于*Logging*部分的第六章中，*NGINX HTTP 服务器*。 | 有效的上下文：`http`，`server`，`location` 默认值：`off` |
| `optimize_server_names` | 该选项已被弃用。请改用`server_name_in_redirect`指令。 | 有效的上下文：`http`，`server` 默认值：`off` |
| `override_charset` | 指示是否应转换从`proxy_pass`或`fastcgi_pass`请求收到的响应中的`Content-Type`头部指定的字符集。如果响应是作为子请求的结果，则总是会转换为主请求的字符集。 | 有效的上下文：`http`，`server`，`location`，`if in location` 默认值：`off` |
| `pcre_jit` | 启用 Perl 兼容正则表达式的即时编译，已知在配置时可用。要利用此加速，必须在 PCRE 库中启用 JIT 支持。 | 有效的上下文：`main` 默认值：`off` |
| `perl` | 为此位置激活 Perl 处理程序。参数是处理程序的名称或描述完整子程序的字符串。 | 有效的上下文：`location`，`limit_except` 默认值：- |
| `perl_modules` | 指定 Perl 模块的额外搜索路径。 | 有效的上下文：`http` 默认值：- |
| `perl_require` | 指示每次 NGINX 重新配置时将加载的 Perl 模块。可以为不同的模块多次指定。 | 有效的上下文：`http` 默认值：- |
| `perl_set` | 安装 Perl 处理程序以设置变量的值。参数是处理程序的名称或描述完整子程序的字符串。 | 有效的上下文：`http` 默认值：- |
| `pid` | 这是将写入主进程进程 ID 的文件，覆盖编译时的默认值。 | 有效的上下文：`main` 默认值：`nginx.pid` |
| `pop3_auth` | 设置支持的客户端认证机制。它可以是`plain`、`apop`或`cram-md5`之一或多个。 | 有效的上下文：`mail`，`server` 默认值：`plain` |
| `pop3_capabilities` | 指示后端服务器支持哪些 POP3 功能。 | 有效上下文：`mail`，`server` 默认值：`TOP USER UIDL` |
| `port_in_redirect` | 确定是否在 NGINX 发出的 `redirect` 方法中指定端口。 | 有效上下文：`http`，`server`，`location` 默认值：`on` |
| `postpone_output` | 指定 NGINX 发送到客户端的数据的最小大小。如果可能，在达到此值之前不会发送任何数据。 | 有效上下文：`http`，`server`，`location` 默认值：`1460` |
| `protocol` | 指示此邮件服务器上下文支持哪种协议。它可能是 `imap`，`pop3` 或 `smtp` 之一。 | 有效上下文：`server` 默认值：- |
| `proxy` | 启用或禁用邮件代理。 | 有效上下文：`server` 默认值：- |
| `proxy_bind` | 指定用于与代理服务器进行外发连接的地址。 | 有效上下文：`http`，`server`，`location` 默认值：- |
| `proxy_buffer` | 允许设置用于邮件代理连接的缓冲区大小，超过默认的单页大小。 | 有效上下文：`mail`，`server` 默认值：`4k&#124;8k`（平台相关） |
| `proxy_buffer_size` | 用于从上游服务器接收响应的第一部分的缓冲区大小，响应头就在其中。 | 有效上下文：`http`，`server`，`location` 默认值：`4k&#124;8k`（平台相关） |
| `proxy_buffering` | 启用代理内容的缓冲；当关闭时，响应将在接收到后同步发送到客户端。 | 有效上下文：`http`，`server`，`location` 默认值：`on` |
| `proxy_buffers` | 用于上游服务器响应的缓冲区的数量和大小。 | 有效上下文：`http`，`server`，`location` 默认值：`8 4k&#124;8k`（平台相关） |
| `proxy_busy_buffers_size` | 分配给发送响应到客户端的缓冲区总大小，同时数据仍在从上游服务器读取。通常设置为两个 `proxy_buffers` 的大小。 | 有效上下文：`http`，`server`，`location` 默认值：`8k&#124;16k`（平台相关） |
| `proxy_cache` | 定义用于缓存的共享内存区域。 | 有效上下文：`http`，`server`，`location` 默认值：`off` |
| `proxy_cache_bypass` | 一个或多个字符串变量，当它们非空或非零时，将导致响应从上游服务器而不是缓存中获取。 | 有效上下文：`http`，`server`，`location` 默认值：- |
| `proxy_cache_key` | 用作存储和检索缓存值的键的字符串。 | 有效上下文：`http`，`server`，`location` 默认值：`$scheme$proxy_host$request_uri` |
| `proxy_cache_lock` | 启用此指令将防止多个请求对同一缓存键进行条目写入。 | 有效上下文：`http`，`server`，`location` 默认值：`off` |
| `proxy_cache_lock_timeout` | 请求等待缓存条目出现或 `proxy_cache_lock` 指令释放的最长时间。 | 有效上下文：`http`，`server`，`location` 默认值：`5s` |
| `proxy_cache_min_uses` | 在缓存响应之前，某个键所需的请求次数。 | 有效上下文：`http`，`server`，`location` 默认值：`1` |
| `proxy_cache_path` | 参考 第五章 *反向代理高级主题* 中的 *代理模块缓存指令* 表。 | 有效上下文：`http` 默认值：- |
| `proxy_cache_use_stale` | 在访问上游服务器时发生错误时，可以提供陈旧的缓存数据的情况。`updating` 参数表示正在加载新数据的情况。 | 有效上下文：`http`，`server`，`location` 默认值：`off` |
| `proxy_cache_valid` | 指示具有响应代码 200、301 或 302 的缓存响应有效的时间长度。如果在时间参数之前给出了可选的响应代码，则该时间仅适用于该响应代码。特殊参数 `any` 表示任何响应代码应缓存该长度的时间。 | 有效上下文：`http`，`server`，`location` 默认值：- |
| `proxy_connect_timeout` | NGINX 在向上游服务器发起请求时，等待连接被接受的最长时间。 | 有效上下文：`http`，`server`，`location` 默认值：`60s` |
| `proxy_cookie_domain` | 替换来自上游服务器的 `Set-Cookie` 头部的域名属性；要替换的域名可以是字符串、正则表达式或引用变量。 | 有效上下文：`http`，`server`，`location` 默认值：`off` |
| `proxy_cookie_path` | 替换来自上游服务器的 `Set-Cookie` 头部的路径属性；要替换的路径可以是字符串、正则表达式或引用变量。 | 有效上下文：`http`，`server`，`location` 默认值：`off` |
| `proxy_header_hash_bucket_size` | 用于存储代理头部名称的桶大小（一个名称不能长于此指令的值）。 | 有效上下文：`http`，`server`，`location`，`if` 默认值：`64` |
| `proxy_header_hash_max_size` | 从上游服务器接收到的头部的总大小。 | 有效上下文：`http`，`server`，`location` 默认值：`512` |
| `proxy_hide_header` | 不应传递给客户端的头部字段列表。 | 有效上下文：`http`，`server`，`location` 默认值：- |
| `proxy_http_version` | 用于与上游服务器通信的 HTTP 协议版本（对于 `keepalive` 连接，使用 `1.1`）。 | 有效上下文：`http`，`server`，`location` 默认值：`1.0` |
| `proxy_ignore_client_abort` | 如果设置为 `on`，则在客户端中止连接时，NGINX 不会中止与上游服务器的连接。 | 有效上下文：`http`、`server`、`location` 默认值：`off` |
| `proxy_ignore_headers` | 设置在处理上游服务器响应时可以忽略哪些头信息。 | 有效上下文：`http`、`server`、`location` 默认值：- |
| `proxy_intercept_errors` | 如果启用，NGINX 将显示配置的 `error_page`，而不是直接从上游服务器响应。 | 有效上下文：`http`、`server`、`location` 默认值：`off` |
| `proxy_max_temp_file_size` | 溢出文件的最大大小，当响应不能完全适应内存缓冲区时，将写入该文件。 | 有效上下文：`http`、`server`、`location` 默认值：`1024m` |

| `proxy_next_upstream` | 指示在何种条件下会选择下一个上游服务器来响应。 如果客户端已经收到响应，则不会使用此设置。条件由以下参数指定：

+   `error`: 与上游服务器通信时发生了错误

+   `timeout`: 与上游服务器通信时发生了超时

+   `invalid_header`: 上游服务器返回了空的或无效的响应

+   `http_500`: 上游服务器返回了 500 错误码

+   `http_503`: 上游服务器返回了 503 错误码

+   `http_504`: 上游服务器返回了 504 错误码

+   `http_404`: 上游服务器返回了 404 错误码

+   `off`: 当发生错误时，禁用将请求传递到下一个上游服务器

| 有效上下文：`http`、`server`、`location` 默认值：`error timeout` |
| --- |
| `proxy_no_cache` | 定义在何种条件下响应不会被保存到缓存中。参数是字符串变量，评估为非空且非零时，不进行缓存。 | 有效上下文：`http`、`server`、`location` 默认值：- |
| `proxy_pass` | 指定请求传递到的上游服务器，采用 URL 形式。 | 有效上下文：`location`、`if in location`、`limit_except` 默认值：- |
| `proxy_pass_error_message` | 在后端身份验证过程中发出有用错误信息并返回给客户端时非常有用。 | 有效上下文：`mail`、`server` 默认值：`off` |
| `proxy_pass_header` | 覆盖 `proxy_hide_header` 中禁用的头信息，允许将其发送给客户端。 | 有效上下文：`http`、`server`、`location` 默认值：- |
| `proxy_pass_request_body` | 如果设置为 `off`，则阻止将请求的主体发送到上游服务器。 | 有效上下文：`http`、`server`、`location` 默认值：`on` |
| `proxy_pass_request_headers` | 如果设置为 `off`，则阻止将请求的头信息发送到上游服务器。 | 有效上下文：`http`、`server`、`location` 默认值：`on` |
| `proxy_read_timeout` | 指定从上游服务器进行两次连续读操作之间所需的时间长度，超过此时间连接将关闭。 | 有效上下文：`http`，`server`，`location` 默认值：`60s` |
| `proxy_redirect` | 重写从上游服务器接收到的`Location`和`Refresh`头部；有助于绕过应用程序框架的假设。 | 有效上下文：`http`，`server`，`location` 默认值：`default` |
| `proxy_send_lowat` | 如果非零，NGINX 将尽量减少对代理服务器的发送操作次数。在 Linux、Solaris 和 Windows 中忽略此设置。 | 有效上下文：`http`，`server`，`location` 默认值：`0` |
| `proxy_send_timeout` | 在关闭连接之前，需要在两次连续写操作之间经过的时间长度。 | 有效上下文：`http`，`server`，`location` 默认值：`60s` |
| `proxy_set_body` | 通过设置此指令，可以更改发送到上游服务器的请求体。 | 有效上下文：`http`，`server`，`location` 默认值：- |
| `proxy_set_header` | 重写发送到上游服务器的头部内容；也可以通过将其值设置为空字符串来不发送某些头部。 | 有效上下文：`http`，`server`，`location` 默认值：`Host $proxy_host`，`Connection close` |
| `proxy_ssl_session_reuse` | 设置是否可以在代理过程中重用 SSL 会话。 | 有效上下文：`http`，`server`，`location` 默认值：`on` |
| `proxy_store` | 启用将从上游服务器获取的响应存储为磁盘上的文件。`on` 参数将使用`alias`或`root`指令作为存储文件的基础路径。可以提供一个字符串，指示用于存储文件的备用位置。 | 有效上下文：`http`，`server`，`location` 默认值：`off` |
| `proxy_store_access` | 设置新创建的`proxy_store`文件的文件访问权限。 | 有效上下文：`http`，`server`，`location` 默认值：`user:rw` |
| `proxy_temp_file_write_size` | 限制一次将数据缓冲到临时文件中的量，以避免 NGINX 在单个请求上阻塞过长时间。 | 有效上下文：`http`，`server`，`location` 默认值：`8k&#124;16k`（平台相关） |
| `proxy_temp_path` | 用于缓冲从上游服务器代理过来的临时文件的目录，可选择多级深度。如果给定第二、第三或第四个参数，这些参数指定了子目录层次结构，参数值表示子目录名称中的字符数。 | 有效上下文：`http`，`server`，`location` 默认值：`proxy_temp` |
| `proxy_timeout` | 如果需要超过默认 24 小时的超时时间，可以使用此指令。 | 有效上下文：`mail`，`server` 默认值：`24h` |
| `random_index` | 启用此选项后，当接收到以`/`结尾的 URI 时，将随机选择一个文件提供给客户端。 | 有效上下文：`location` 默认值：`off` |
| `read_ahead` | 如果可能，内核将预读文件直到`size`参数。当前 FreeBSD 和 Linux 支持此功能（`size`参数在 Linux 上被忽略）。 | 有效上下文：`http`，`server`，`location` 默认值：`0` |
| `real_ip_header` | 设置一个标头，其值在`set_real_ip_from`匹配连接 IP 时用作客户端 IP 地址。 | 有效上下文：`http`，`server`，`location` 默认值：`X-Real-IP` |
| `real_ip_recursive` | 与`set_real_ip_from`一起工作，用于指定多值`real_ip_header`头中的最后一个地址将被使用。 | 有效上下文：`http`，`server`，`location` 默认值：`off` |
| `recursive_error_pages` | 启用后可以使用`error_page`指令进行多次重定向（默认是`off`）。 | 有效上下文：`http`，`server`，`location` 默认值：`off` |
| `referer_hash_bucket_size` | 有效 referers 哈希表的桶大小。 | 有效上下文：`server`，`location` 默认值：`64` |
| `referer_hash_max_size` | 有效 referers 哈希表的最大大小。 | 有效上下文：`server`，`location` 默认值：`2048` |
| `request_pool_size` | 微调每个请求的内存分配。 | 有效上下文：`http`，`server` 默认值：`4k` |
| `reset_timedout_connection` | 启用此指令后，超时的连接将立即重置，释放所有相关内存。默认情况下，套接字将处于`FIN_WAIT1`状态，这对于`keepalive`连接始终是这样。 | 有效上下文：`http`，`server`，`location` 默认值：`off` |
| `resolver` | 配置一个或多个名称服务器，用于将上游服务器名称解析为 IP 地址。可选的`valid`参数可以覆盖域名记录的 TTL。 | 有效上下文：`http`，`server`，`location` 默认值：- |
| `resolver_timeout` | 设置名称解析的超时时间。 | 有效上下文：`http`，`server`，`location` 默认值：`30s` |
| `return` | 停止处理并返回指定的代码给客户端。非标准代码 444 会在不发送任何响应头的情况下关闭连接。如果代码后面附带文本，该文本将放入响应体中。如果代码后面给出一个 URL，则该 URL 将作为`Location`头的值。没有代码的 URL 被视为代码 302。 | 有效上下文：`server`，`location`，`if` 默认值：- |
| `rewrite` | 请参阅附录 B 中 *重写模块指南* 部分的 *重写模块指令* 表。 | 有效上下文：`server`，`location`，`if` 默认值：- |
| `rewrite_log` | 激活将`rewrites`的`notice`级别日志记录到`error_log`中。 | 有效的上下文：`http`、`server`、`if in server`、`location`、`if in location` 默认值：`off` |
| `root` | 设置文档根路径。通过将 URI 附加到该指令的值上，可以找到文件。 | 有效的上下文：`http`、`server`、`location`、`if in location` 默认值：`html` |
| `satisfy` | 如果`access`或`auth_basic`指令中的`all`或`any`允许访问，则允许访问。默认值`all`表示用户必须来自特定的网络地址并输入正确的密码。 | 有效的上下文：`http`、`server`、`location` 默认值：`all` |
| `satisfy_any` | 该指令已被弃用。请使用`satisfy`指令的`any`参数。 | 有效的上下文：`http`、`server`、`location` 默认值：`off` |
| `secure_link_secret` | 用于计算 URI 的 MD5 哈希值的盐值。 | 有效的上下文：`location` 默认值：- |
| `send_lowat` | 如果非零，NGINX 将尽量减少客户端套接字的发送操作次数。在 Linux、Solaris 和 Windows 中被忽略。 | 有效的上下文：`http`、`server`、`location` 默认值：`0` |
| `send_timeout` | 此指令设置客户端接收响应时，连续两次写操作之间的超时时间。 | 有效的上下文：`http`、`server`、`location` 默认值：`60s` |
| `sendfile` | 启用使用`sendfile(2)`直接从一个文件描述符复制数据到另一个文件描述符。 | 有效的上下文：`http`、`server`、`location`、`if in location` 默认值：`off` |
| `sendfile_max_chunk` | 设置每次`sendfile(2)`调用时复制的最大数据大小，以防止工作进程被占用。 | 有效的上下文：`http`、`server`、`location` 默认值：`0` |
| `server` (`http`) | 创建一个新的配置上下文，定义一个虚拟主机。`listen`指令指定 IP 地址和端口；`server_name`指令列出此上下文匹配的`Host`头部值。 | 有效的上下文：`http` 默认值：- |
| `server` (upstream) | 请参阅第四章中*Upstream 模块指令*表格，*NGINX 作为反向代理*部分。 | 有效的上下文：`upstream` 默认值：- |
| `server` (`mail`) | 创建一个新的配置上下文，定义一个邮件服务器。`listen`指令指定 IP 地址和端口；`server_name`指令设置服务器的名称。 | 有效的上下文：`mail` 默认值：- |
| `server_name` (`http`) | 配置虚拟主机可能响应的名称。 | 有效的上下文：`server` 默认值："" |

| `server_name` (`mail`) | 设置服务器的名称，服务器名称可用于以下几种方式：

+   POP3/SMTP 服务器问候语

+   SASL CRAM-MD5 身份验证的盐值

+   使用`xclient`与 SMTP 后台通信时的 EHLO 名称

| 有效的上下文：`mail`、`server` 默认值：`hostname` |
| --- |
| `server_name_in_redirect` | 启用在此上下文中通过 NGINX 发出的任何重定向中使用`server_name`指令的第一个值。 | 有效的上下文：`http`，`server`，`location`默认值：`off` |
| `server_names_hash_bucket_size` | 用于存放`server_name`哈希表的桶大小。 | 有效的上下文：`http`默认值：`32 | 64 | 128`（依赖于处理器） |
| `server_names_hash_max_size` | `server_name`哈希表的最大大小。 | 有效的上下文：`http`默认值：`512` |
| `server_tokens` | 禁用在错误信息和`Server`响应头中发送 NGINX 版本字符串（默认值为`on`）。 | 有效的上下文：`http`，`server`，`location`默认值：`on` |
| `set` | 将给定变量设置为指定值。 | 有效的上下文：`server`，`location`，`if`默认值：- |
| `set_real_ip_from` | 定义从哪些连接地址中提取客户端 IP，这些地址来自`real_ip_header`指令。`unix:`值表示所有来自 UNIX 域套接字的连接都将按此方式处理。 | 有效的上下文：`http`，`server`，`location`默认值：- |
| `smtp_auth` | 设置支持的 SASL 客户端认证机制。可以是`login`、`plain`或`cram-md5`之一或多个。 | 有效的上下文：`mail`，`server`默认值：`login`，`plain` |
| `smtp_capabilities` | 指示后端服务器支持的 SMTP 功能。 | 有效的上下文：`mail`，`server`默认值：- |
| `so_keepalive` | 在与代理服务器的套接字连接上设置 TCP `keepalive` 参数。 | 有效的上下文：`mail`，`server`默认值：`off` |
| `source_charset` | 定义响应的字符集。如果与定义的字符集不同，则会进行转换。 | 有效的上下文：`http`，`server`，`location`，`if in location`默认值：- |
| `split_clients` | 创建一个适合 A/B（或分割）测试的上下文。第一个参数指定的字符串通过`MurmurHash2`进行哈希处理。然后，第二个参数指定的变量将基于字符串在哈希值范围内的位置设置为一个值。匹配可以指定为百分比或`*`，以对值进行加权。 | 有效的上下文：`http`默认值：- |
| `ssi` | 启用 SSI 文件的处理。 | 有效的上下文：`http`，`server`，`location`，`if in location`默认值：`off` |
| `ssi_min_file_chunk` | 设置文件的最小大小，超过此大小的文件应使用`sendfile(2)`发送。 | 有效的上下文：`http`，`server`，`location`默认值：`1k` |
| `ssi_silent_errors` | 抑制在 SSI 处理过程中发生错误时通常输出的错误信息。 | 有效的上下文：`http`，`server`，`location`默认值：`off` |
| `ssi_types` | 在响应中列出除了`text/html`外 SSI 命令处理的 MIME 类型。可以是`*`以启用所有 MIME 类型。 | 有效上下文：`http`, `server`, `location`默认值：`text/html` |
| `ssi_value_length` | 设置用于服务器端包含的参数值的最大长度。 | 有效上下文：`http`, `server`, `location`默认值：`256` |
| `ssl` (`http`) | 启用此虚拟服务器的 HTTPS 协议。 | 有效上下文：`http`, `server`默认值：`off` |
| `ssl` (`mail`) | 指示此上下文是否支持 SSL/TLS 事务。 | 有效上下文：`mail`, `server`默认值：`off` |
| `ssl_certificate` (`http`) | 包含以 PEM 格式存储的`server_name`的 SSL 证书文件路径。如果需要中间证书，需要按顺序添加在与`server_name`指令对应的证书之后，必要时直至根证书。 | 有效上下文：`http`, `server`默认值：- |
| `ssl_certificate` (`mail`) | 这个虚拟服务器的 PEM 编码 SSL 证书的路径。 | 有效上下文：`mail`, `server`默认值：- |
| `ssl_certificate_key` (`http`) | 包含 SSL 证书秘密密钥的文件路径。 | 有效上下文：`http`, `server`默认值：- |
| `ssl_certificate_key` (`mail`) | 这个虚拟服务器的 PEM 编码 SSL 密钥的路径。 | 有效上下文：`mail`, `server`默认值：- |
| `ssl_ciphers` | 在这个虚拟服务器上支持的密码（OpenSSL 格式）。 | 有效上下文：`http`, `server`默认值：`HIGH:!aNULL:!MD5` |
| `ssl_client_certificate` | 包含用于签署客户端证书的证书颁发机构的 PEM 编码公共 CA 证书的文件路径。 | 有效上下文：`http`, `server`默认值：- |
| `ssl_crl` | 包含用于验证客户端证书的 PEM 编码证书吊销列表（**CRL**）的文件路径。 | 有效上下文：`http`, `server`默认值：- |
| `ssl_dhparam` | 包含 DH 参数的文件路径，用于 EDH 密码。 | 有效上下文：`http`, `server`默认值：- |
| `ssl_engine` | 指定硬件 SSL 加速器。 | 有效上下文：`main`默认值：- |
| `ssl_prefer_server_ciphers` (`http`) | 当使用 SSLv3 和 TLS 协议时，指示服务器密码优先于客户端密码。 | 有效上下文：`http`, `server`默认值：`off` |
| `ssl_prefer_server_ciphers` (`mail`) | 指示 SSLv3 和 TLSv1 服务器密码优先于客户端密码。 | 有效上下文：`mail`, `server`默认值：`off` |
| `ssl_protocols` (`http`) | 指示应启用哪些 SSL 协议。 | 有效上下文：`http`, `server`默认值：`SSLv3`, `TLSv1`, `TLSv1.1`, `TLSv1.2` |
| `ssl_protocols` (`mail`) | 指示应启用哪些 SSL 协议。 | 有效上下文：`mail`，`server`默认值：`SSLv3`，`TLSv1`，`TLSv1.1`，`TLSv1.2` |

| `ssl_session_cache` (`http`) | 设置 SSL 缓存的类型和大小，以存储会话参数。缓存可以是以下类型之一：

+   `off`: 客户端被告知根本不会重用会话

+   `none`: 客户端被告知会重用会话，但实际上并没有

+   `builtin`: 只有一个工作进程使用的 OpenSSL 内置缓存，会话大小指定在会话中

+   `shared`: 所有工作进程共享的缓存，指定名称和以兆字节为单位的会话大小

| 有效上下文：`http`，`server`默认值：`none` |
| --- |

| `ssl_session_cache` (`mail`) | 设置 SSL 缓存的类型和大小，以存储会话参数。缓存可以是以下类型之一：

+   `off`: 客户端被告知根本不会重用会话

+   `none`: 客户端被告知会重用会话，但实际上并没有

+   `builtin`: 只有一个工作进程使用的 OpenSSL 内置缓存，会话大小指定在会话中

+   `shared`: 所有工作进程共享的缓存，指定名称和以兆字节为单位的会话大小

| 有效上下文：`mail`，`server`默认值：`none` |
| --- |
| `ssl_session_timeout` (`http`) | 客户端可以使用相同的 SSL 参数的时间长短，只要它们存储在缓存中。 | 有效上下文：`http`，`server`默认值：`5m` |
| `ssl_session_timeout` (`mail`) | 客户端可以使用相同的 SSL 参数的时间长短，只要它们存储在缓存中。 | 有效上下文：`mail`，`server`默认值：`5m` |
| `ssl_stapling` | 启用 OCSP 响应的装订。服务器的颁发者的 CA 证书应包含在由 `ssl_trusted_certificate` 指定的文件中。还应指定一个解析器以能够解析 OCSP 响应器的主机名。 | 有效上下文：`http`，`server`默认值：`off` |
| `ssl_stapling_file` | 包含装订 OCSP 响应的 DER 格式文件的路径。 | 有效上下文：`http`，`server`默认值：- |
| `ssl_stapling_responder` | 指定 OCSP 响应器的 URL。当前仅支持以 `http://` 开头的 URL。 | 有效上下文：`http`，`server`默认值：- |
| `ssl_stapling_verify` | 启用 OCSP 响应的验证。 | 有效上下文：`http`，`server`默认值：- |
| `ssl_trusted_certificate` | 包含 PEM 格式的 SSL 证书的文件路径，用于 CA 的签名客户端证书和启用 `ssl_stapling` 时的 OCSP 响应。 | 有效上下文：`http`，`server`默认值：- |
| `ssl_verify_client` | 启用 SSL 客户端证书的验证。如果指定了 `optional` 参数，将请求客户端证书，并且如果存在则验证。如果指定了 `optional_no_ca` 参数，则请求客户端证书，但不要求其由受信任的 CA 证书签名。 | 有效上下文：`http`，`server`默认值：`off` |
| `ssl_verify_depth` | 设置在声明证书无效之前将检查多少个签发者。 | 有效上下文：`http`，`server` 默认值：`1` |
| `starttls` | 指示是否支持和/或需要使用 STLS/STARTTLS 进行与该服务器的进一步通信。 | 有效上下文：`mail`，`server` 默认值：`off` |
| `sub_filter` | 设置不区分大小写的匹配字符串和要替换为该匹配的字符串。替换字符串可以包含变量。 | 有效上下文：`http`，`server`，`location` 默认值：- |
| `sub_filter_once` | 设置为`off`将导致`sub_filter`中的匹配尽可能多次发生，直到找到所有字符串为止。 | 有效上下文：`http`，`server`，`location` 默认值：`on` |
| `sub_filter_types` | 列出除`text/html`外的响应 MIME 类型，其中将进行替换。可以使用`*`来启用所有 MIME 类型。 | 有效上下文：`http`，`server`，`location` 默认值：`text/html` |
| `tcp_nodelay` | 启用或禁用`keep-alive`连接的`TCP_NODELAY`选项。 | 有效上下文：`http`，`server`，`location` 默认值：`on` |
| `tcp_nopush` | 仅在使用`sendfile`指令时相关。启用 NGINX 尝试将响应头以一个数据包发送，并且将文件以完整数据包发送。 | 有效上下文：`http`，`server`，`location` 默认值：`off` |
| `timeout` | NGINX 在与后台服务器建立连接之前等待的时间。 | 有效上下文：`mail`，`server` 默认值：`60s` |
| `timer_resolution` | 指定每当接收到内核事件时，`gettimeofday()`的调用频率。 | 有效上下文：`main` 默认值：- |
| `try_files` | 测试给定为参数的文件是否存在。如果没有找到任何之前的文件，则使用最后一个条目作为回退，因此请确保该路径或命名的`location`存在。 | 有效上下文：`server`，`location` 默认值：- |

| `types` | 设置 MIME 类型与文件扩展名的映射。NGINX 附带一个`conf/mime.types`文件，其中包含大多数 MIME 类型映射。使用`include`加载此文件应足以满足大多数需求。 | 有效上下文：`http`，`server`，`location` 默认值： |

```
    text/html  html;
    image/gif  gif;
    image/jpeg jpg
```

|

| `types_hash_bucket_size` | 用于存放`types`哈希表的桶大小。 | 有效上下文：`http`，`server`，`location` 默认值：`32&#124;64&#124;128`（处理器依赖） |
| --- | --- | --- |
| `types_hash_max_size` | `types`哈希表的最大大小。 | 有效上下文：`http`，`server`，`location` 默认值：`1024` |
| `underscores_in_headers` | 启用在客户端请求头中使用下划线字符。如果保持默认值`off`，则此类头部的评估将受`ignore_invalid_headers`指令的值影响。 | 有效上下文：`http`，`server` 默认值：`off` |
| `uninitialized_variable_warn` | 控制是否记录未初始化变量的警告。 | 有效上下文：`http`，`server`，`location`，`if` 默认值：`on` |
| `upstream` | 设置一个命名上下文，在该上下文中定义一组服务器。 | 有效上下文：`http` 默认值：- |
| `use` | `use`指令表示应使用哪种连接处理方法。若使用此指令，它将覆盖已编译的默认值，并且必须包含在`events`上下文中。如果使用时，尤其在发现已编译的默认值随着时间的推移产生错误时非常有用。 | 有效上下文：`events` 默认值：- |
| `user` | 配置 worker 进程运行的`user`和`group`。如果省略`group`，则使用与`user`相同的组名。 | 有效上下文：`main` 默认值：`nobody nobody` |

| `userid` | 根据以下参数激活模块：

+   `on`: 设置版本 2 的 cookie 并记录接收到的 cookie

+   `v1`: 设置版本 1 的 cookie 并记录接收到的 cookie

+   `log`: 禁用设置 cookie，但启用记录 cookie

+   `off`: 禁用设置 cookie 和记录 cookie

| 有效上下文：`http`，`server`，`location` 默认值：`off` |
| --- |
| `userid_domain` | 配置要设置在 cookie 中的域名。 | 有效上下文：`http`，`server`，`location` 默认值：`none` |
| `userid_expires` | 设置 cookie 的过期时间。如果使用`max`关键字，则表示`31 Dec 2037 23:55:55 GMT`。 | 有效上下文：`http`，`server`，`location` 默认值：- |
| `userid_mark` | 设置`userid_name` cookie 的 base64 表示形式尾部的第一个字符。 | 有效上下文：`http`，`server`，`location` 默认值：`off` |
| `userid_name` | 设置 cookie 的名称。 | 有效上下文：`http`，`server`，`location` 默认值：`uid` |
| `userid_p3p` | 配置 P3P 头部。 | 有效上下文：`http`，`server`，`location` 默认值：- |
| `userid_path` | 定义 cookie 中设置的路径。 | 有效上下文：`http`，`server`，`location` 默认值：`/` |
| `userid_service` | 设置 cookie 的服务身份。例如，版本 2 的 cookie 默认值是设置 cookie 的服务器的 IP 地址。 | 有效上下文：`http`，`server`，`location` 默认值：服务器的 IP 地址 |

| `valid_referers` | 定义哪些`Referer`头部的值将导致`$invalid_referer`变量被设置为空字符串，否则将设置为`1`。参数可以是以下之一或多个：

+   `none`: 没有`Referer`头部

+   `blocked`: `Referer`头部存在，但为空或缺少方案

+   `server_names`: `Referer`值是`server_names`之一

+   任意字符串：`Referer`头部的值是服务器名称，可以带有或不带有 URI 前缀，并且`*`出现在开头或结尾

+   正则表达式：匹配`Referer`头部值中的方案后的文本

| 有效上下文: `server`, `location` 默认值: - |
| --- |
| `variables_hash_bucket_size` | 用于存储剩余变量的桶大小。 | 有效上下文: `http` 默认值: `64` |
| `variables_hash_max_size` | 存储剩余变量的哈希表的最大大小。 | 有效上下文: `http` 默认值: `512` |
| `worker_aio_requests` | 在使用`aio`和`epoll`时，单个工作进程可以打开的异步 I/O 操作数。 | 有效上下文: `events` 默认值: `32` |
| `worker_connections` | 此指令配置工作进程可以打开的最大同时连接数。包括但不限于客户端连接和与上游服务器的连接。 | 有效上下文: `events` 默认值: `512` |
| `worker_cpu_affinity` | 将工作进程绑定到 CPU 集，按位掩码指定。仅在 FreeBSD 和 Linux 上可用。 | 有效上下文: `main` 默认值: - |
| `worker_priority` | 设置工作进程的调度优先级。其作用类似于 nice 命令，负数表示更高优先级。 | 有效上下文: `main` 默认值: `0` |
| `worker_processes` | 这是将启动的工作进程数量。这些进程将处理所有客户端的连接。选择合适的数量是一个复杂的过程，一个好的经验法则是将其设置为 CPU 核心数。 | 有效上下文: `main` 默认值: `1` |
| `worker_rlimit_core` | 更改运行中进程的核心文件大小限制。 | 有效上下文: `main` 默认值: - |
| `worker_rlimit_nofile` | 更改运行中进程的最大打开文件数限制。 | 有效上下文: `main` 默认值: - |
| `worker_rlimit_sigpending` | 当使用`rtsig`连接处理方法时，更改运行中进程的挂起信号数量限制。 | 有效上下文: `main` 默认值: - |
| `working_directory` | 工作进程的当前工作目录。该目录应可由工作进程写入，以生成核心文件。 | 有效上下文: `main` 默认值: - |
| `xclient` | SMTP 协议允许基于`IP/HELO/LOGIN`参数进行检查，这些参数通过`XCLIENT`命令传递。此指令启用 NGINX 传递这些信息。 | 有效上下文: `mail`, `server` 默认值: `on` |
| `xml_entities` | 处理 XML 中引用的字符实体的 DTD 路径。 | 有效上下文: `http`, `server`, `location` 默认值: - |
| `xslt_param` | 传递给样式表的参数，其值为 XPath 表达式。 | 有效上下文: `http`, `server`, `location` 默认值: - |
| `xslt_string_param` | 传递给样式表的参数，其值为字符串。 | 有效上下文: `http`, `server`, `location` 默认值: - |
| `xslt_stylesheet` | 用于转换 XML 响应的 XSLT 样式表路径。可以作为一系列键/值对传递参数。 | 有效上下文：`location` 默认值：- |
| `xslt_types` | 列出响应的 MIME 类型，除了 `text/xml`，在其中会进行替换。它可以是 `*`，以启用所有 MIME 类型。如果转换结果是 HTML 响应，MIME 类型将更改为 `text/html`。 | 有效上下文：`http`，`server`，`location` 默认值：`text/xml` |
