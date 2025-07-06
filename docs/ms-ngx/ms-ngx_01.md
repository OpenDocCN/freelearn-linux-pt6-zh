# 第一章 安装 NGINX 和第三方模块

NGINX 最初被设计为一个 HTTP 服务器。它的诞生是为了解决 Daniel Kegel 在[`www.kegel.com/c10k.html`](http://www.kegel.com/c10k.html)中描述的 C10K 问题，即设计一个能够处理 10,000 个并发连接的 Web 服务器。NGINX 能够通过其基于事件的连接处理机制来实现这一点，并将使用适合操作系统的事件机制来实现这一目标。

在我们开始探讨如何配置 NGINX 之前，我们首先需要安装它。本章详细介绍了如何安装 NGINX 及如何正确安装和配置所需的模块。NGINX 是模块化设计的，且有一个活跃的第三方模块开发者社区，他们通过创建可以编译到服务器中并与之一起安装的模块，扩展了 NGINX 核心服务器的功能。

本章将涵盖：

+   使用包管理器安装 NGINX

+   从源代码安装 NGINX

+   配置 Web 或邮件服务

+   启用各种模块

+   查找并安装第三方模块

+   整合所有内容

# 使用包管理器安装 NGINX

很可能你的操作系统已经提供了`nginx`作为包管理的一部分。安装它就像使用包管理器的命令一样简单：

+   Linux（基于 deb 的系统）

    ```
    sudo apt-get install nginx

    ```

+   Linux（基于 rpm 的系统）

    ```
    sudo yum install nginx

    ```

+   FreeBSD

    ```
    sudo pkg_install -r nginx

    ```

    ### 注意

    `sudo`命令代表你在操作系统上需要执行的命令，以获得超级用户（'root'）权限。如果你的操作系统支持**RBAC**（**基于角色的访问控制**），则你需要使用其他命令，如'pfexec'，来达到相同的目的。

这些命令将把 NGINX 安装到操作系统特定的标准位置。如果你需要使用操作系统的包，这是首选的安装方法。

NGINX 核心团队还提供了稳定版本的二进制文件，可以从[`nginx.org/en/download.html`](http://nginx.org/en/download.html)下载。没有`nginx`包的发行版（如 CentOS）用户，可以使用以下说明安装经过预先测试和编译的二进制文件。

## CentOS

通过创建以下文件，将 NGINX 仓库添加到你的 yum 配置中：

```
sudo vi /etc/yum.repos.d/nginx.repo
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/6/$basearch/
gpgcheck=0
enabled=1

```

然后执行以下命令安装`nginx`：

```
sudo yum install nginx

```

有关安装`nginx-release`包的替代说明，请参阅上述 URL。

## Debian

通过从[`nginx.org/keys/nginx_signing.key`](http://nginx.org/keys/nginx_signing.key)下载 NGINX 签名密钥，并将其添加到 apt 密钥环中来安装 NGINX 签名密钥：

```
sudo apt-key add nginx_signing.key

```

将`nginx.org`仓库添加到`/etc/apt/sources.list`的末尾：

```
vi /etc/apt/sources.list
deb http://nginx.org/packages/debian/ squeeze nginx
deb-src http://nginx.org/packages/debian/ squeeze nginx

```

然后执行以下命令安装`nginx`：

```
sudo apt-get update
sudo apt-get install nginx

```

如果你的操作系统没有在其可用软件包列表中包含 `nginx`，或者列表中的版本太旧，不满足你的需求，或者 [nginx.org](http://nginx.org) 提供的包不能满足你的需求，或者你希望使用 NGINX 的“开发版”，那么从源代码编译 NGINX 将是唯一的选择。

# 从源代码安装 NGINX

NGINX 提供了两个独立版本的下载——稳定版和开发版。开发版是正在进行活跃开发的版本，新功能会在此分支中实现，并在集成之后转移到稳定版中。当发布“开发版”时，它已经经过了与稳定版相同的质量保证和类似的功能测试，因此两个分支都可以在生产系统中使用。两者之间的主要区别在于对第三方模块的支持。开发版中的内部 API 可能会发生变化，而稳定版中的 API 保持不变，因此，只有稳定版才支持第三方模块的向后兼容性。

## 准备构建环境

为了从源代码编译 NGINX，你的系统需要满足某些要求。除了编译器外，如果你希望启用 SSL 支持并能够使用 `rewrite` 模块，你还需要 OpenSSL 和 **PCRE**（**Perl 兼容正则表达式**）库以及开发头文件。根据你的系统，这些要求可能已经在默认安装中满足。如果没有，你需要找到相应的包并安装，或者下载源代码并解压，然后将 NGINX 的配置脚本指向该位置。

如果你在配置时包含 `–with-<library>=<path>` 选项，NGINX 会尝试静态构建依赖的库。如果你希望确保 NGINX 不依赖系统的其他部分，或希望从 `nginx` 二进制文件中挤出额外的性能，你可能需要这么做。如果你使用的是外部库的某些功能，而这些功能只有在某个特定版本及之后的版本中才可用（例如，OpenSSL 1.0.1 版本及以后的 Next Protocol Negotiation TLS 扩展），那么你必须指定该版本的源代码路径。

还有一些其他可选的软件包，你可以根据需要提供支持。这些包括 MD5 和 SHA-1 哈希算法支持、zlib 压缩和 libatomic 库支持。哈希算法在 NGINX 的许多地方使用，例如，用来计算 URI 的哈希值，以确定缓存键。zlib 压缩库用于传输压缩过的内容。如果 `atomic_ops` 库可用，NGINX 将使用它的原子内存更新操作来实现高性能的内存锁定代码。

## 从源代码编译

NGINX 可以从 [`nginx.org/en/download.html`](http://nginx.org/en/download.html) 下载。在这里，你将找到 `.tar.gz` 或 `.zip` 格式的源代码。将归档文件解压到临时目录，操作如下：

```
$ mkdir $HOME/build
$ cd $HOME/build && tar xzf nginx-<version-number>.tar.gz

```

使用以下命令进行配置：

```
$ cd $HOME/build/nginx-<version-number> && ./configure

```

然后按如下方式编译：

```
$ make && sudo make install

```

在编译自己的 `nginx` 二进制文件时，你可以更自由地只包含需要的部分。你是否已经确定了 NGINX 应该以哪个用户身份运行？你是否想指定默认的日志文件位置，这样就不需要在配置中显式设置？以下配置选项表将帮助你设计自己的二进制文件。这些选项在 NGINX 中有效，与启用的模块无关。

### 表格：常见配置选项

| 选项 | 说明 |
| --- | --- |
| `--prefix=<path>` | 安装的根路径，所有其他安装路径都相对于此路径。 |
| `--sbin-path=<path>` | `nginx` 二进制文件的路径。如果未指定，将相对于前缀路径。 |
| `--conf-path=<path>` | 指定 `nginx` 查找其配置文件的路径，如果命令行中未指定该路径。 |
| `--error-log-path=<path>` | `nginx` 将在此路径下写入错误日志文件，除非另行配置。 |
| `--pid-path=<path>` | 指定 `nginx` 将在此路径下写入主进程的 PID 文件，通常位于 `/var/run` 下。 |
| `--lock-path=<path>` | 共享内存互斥锁文件的路径。 |
| `--user=<user>` | 指定工作进程应该运行的用户。 |
| `--group=<group>` | 指定工作进程应该运行的用户组。 |
| `--with-file-aio` | 启用 FreeBSD 4.3+ 和 Linux 2.6.22+ 的异步 I/O。 |
| `--with-debug` | 启用调试日志记录的选项。建议不要在生产环境中使用。 |

你还可以使用某些优化选项，这些选项在打包安装时可能无法获得。以下选项在这种情况下尤其有用：

### 表格：优化配置选项

| 选项 | 说明 |
| --- | --- |
| `--with-cc=<path>` | 如果你想设置一个不在默认 PATH 中的 C 编译器，可以使用此选项。 |
| `--with-cpp=<path>` | 指定 C 预处理器的路径。 |
| `--with-cc-opt=<options>` | 在此处可以指定必要的 `include` 文件路径（`-I<path>`），以及优化选项（`-O4`）和指定 64 位构建。 |
| `--with-ld-opt=<options>` | 链接器的选项，包括库路径（`-L<path>`）和运行路径（`-R<path>`）。 |
| `--with-cpu-opt=<cpu>` | 可以通过此选项指定特定 CPU 家族的构建。 |

# 配置 Web 或邮件服务

NGINX 在高性能 Web 服务器中独树一帜，因为它还被设计为邮件代理服务器。根据你构建 NGINX 的目标，可以将其配置为 Web 加速、Web 服务器、邮件代理或三者兼顾。通过配置可以使得安装在基础架构中任何服务器的单一软件包发挥作用，或许将 NGINX 的角色通过配置进行设置会更有益，或者在高性能环境下，如果每多出的 KB 都很重要，使用精简的二进制文件也许更符合你的需求。

## 邮件代理的配置选项

下表列出了与邮件模块相关的独特配置选项：

### 表格：邮件配置选项

| 选项 | 说明 |
| --- | --- |
| `--with-mail` | 启用`mail`模块，该模块默认未启用。 |
| `--with-mail_ssl_module` | 为了代理任何使用 SSL/TLS 的`mail`事务，需要启用此模块。 |
| `--without-mail_pop3_module` | 启用`mail`模块时，可以单独禁用 POP3 模块。 |
| `--without-mail_imap_module` | 启用`mail`模块时，可以单独禁用 IMAP 模块。 |
| `--without-mail_smtp_module` | 启用`mail`模块时，可以单独禁用 SMTP 模块。 |
| `--without-http` | 该选项将完全禁用`http`模块；如果只想编译`mail`支持，可以使用此选项。 |

对于典型的邮件代理，我建议按如下方式配置 NGINX：

```
$ ./configure --with-mail --with-mail_ssl_module --with-openssl=${BUILD_DIR}/openssl-1.0.1c
```

目前几乎所有邮件安装都需要 SSL/TLS，如果在邮件代理上未启用 SSL/TLS，会剥夺用户预期的功能。我推荐将 OpenSSL 静态编译，以避免依赖操作系统的 OpenSSL 库。前面命令中提到的`BUILD_DIR`变量当然需要预先设置。

## 配置选项以指定路径

下表展示了`http`模块的可用配置选项，从激活 Perl 模块到指定临时目录的位置：

### 表格：HTTP 配置选项

| 选项 | 说明 |
| --- | --- |
| `--without-http-cache` | 使用上游模块时，NGINX 可以配置为本地缓存内容。此选项禁用该缓存。 |
| `--with-http_perl_module` | 可以通过使用 Perl 代码扩展 NGINX 配置。此选项激活该模块。（不过，使用此模块会降低性能。） |
| `--with-perl_modules_path=<path>` | 该选项指定了用于嵌入式 Perl 解释器所需的附加 Perl 模块的路径。也可以作为配置选项指定。 |
| `--with-perl=<path>` | Perl 的路径（版本 5.6.1 及以上），如果在默认路径中没有找到。 |
| `--http-log-path=<path>` | HTTP 访问日志的默认路径。 |
| `--http-client-body-temp-path=<path>` | 在接收客户端请求时，这是用于存储该请求主体的临时目录。如果启用了 WebDAV 模块，建议将此路径设置为与最终目的地位于同一文件系统中。 |
| `--http-proxy-temp-path=<path>` | 在代理时，这是用来存储临时文件的目录。 |
| `--http-fastcgi-temp-path=<path>` | FastCGI 临时文件的存储位置。 |
| `--http-uwsgi-temp-path=<path>` | uWSGI 临时文件的存储位置。 |
| `--http-scgi-temp-path=<path>` | SCGI 临时文件的存储位置。 |

# 启用各种模块

除了 `http` 和 `mail` 模块，NGINX 发行版中还包含了许多其他模块。这些模块默认情况下是未激活的，但可以通过设置适当的配置选项 `--with-<module-name>_module` 来启用它们。

## 表格：HTTP 模块配置选项

| 选项 | 说明 |
| --- | --- |
| `--with-http_ssl_module` | 如果需要加密 Web 流量，则需要启用此选项，以便能够使用以`https`开头的 URL。（需要 OpenSSL 库。） |
| `--with-http_realip_module` | 如果你的 NGINX 位于 L7 负载均衡器或其他将客户端 IP 地址传递到 HTTP 头中的设备之后，则需要启用此模块。用于多个客户端似乎来自同一 IP 地址的情况。 |
| `--with-http_addition_module` | 该模块作为输出过滤器工作，允许你在指定位置之前或之后添加来自其他位置的内容。 |
| `--with-http_xslt_module` | 该模块将处理基于一个或多个 XSLT 样式表的 XML 响应转换。（需要 libxml2 和 libxslt 库。） |
| `--with-http_image_filter_module` | 该模块能够对图像进行过滤，在将其交给客户端之前进行处理。（需要 libgd 库。） |
| `--with-http_geoip_module` | 通过该模块，你可以设置多个变量，在配置块中根据客户端 IP 地址的地理位置做出决策。（需要 MaxMind GeoIP 库和相应的预编译数据库文件。） |
| `--with-http_sub_module` | 该模块实现了一个替换过滤器，将响应中的一个字符串替换为另一个字符串。 |
| `--with-http_dav_module` | 启用此模块将激活用于使用 WebDAV 的配置指令。请注意，仅在需要使用时启用此模块，因为如果配置不当，可能会带来安全问题。 |
| `--with-http_flv_module` | 如果你需要能够流式传输 Flash 视频文件，启用此模块将提供伪流式传输功能。 |
| `--with-http_mp4_module` | 该模块支持 H.264/AAC 文件的伪流式传输。 |
| `--with-http_gzip_static_module` | 如果您希望在调用资源时无需 `.gz` 后缀就能发送预先压缩的静态文件版本，请使用此模块。 |
| `--with-http_gunzip_module` | 该模块将为不支持 gzip 编码的客户端解压预先压缩的内容。 |
| `--with-http_random_index_module` | 如果您希望从目录中的文件中随机选择一个索引文件进行服务，则需要启用此模块。 |
| `--with-http_secure_link_module` | 该模块提供了一种机制，用于将 URL 链接进行哈希处理，使得只有拥有正确密码的人才能计算该链接。 |
| `--with-http_stub_status_module` | 启用此模块将帮助您从 NGINX 本身收集统计信息。可以使用 RRDtool 或类似工具对输出进行图表化处理。 |

如您所见，这些都是在 `http` 模块基础上构建的模块，提供了额外的功能。在编译时启用这些模块应该不会影响运行时性能。后续在配置中使用这些模块时，才可能会影响性能。

因此，我建议针对 Web 加速器/代理使用以下 `configure` 选项：

```
$ ./configure --with-http_ssl_module --with-http_realip_module --with-http_geoip_module --with-http_stub_status_module --with-openssl=${BUILD_DIR}/openssl-1.0.1c

```

以及以下针对 Web 服务器的配置：

```
$ ./configure --with-http_stub_status_module

```

区别在于 NGINX 面向客户端的角色。Web 加速角色会处理 SSL 请求终止，处理代理客户端，并根据客户端来源做出决策。而 Web 服务器角色只需要提供默认的文件服务能力。

我建议始终启用 `stub_status` 模块，因为它提供了一种收集有关 NGINX 性能指标的手段。

## 禁用未使用的模块

还有一些通常启用的 `http` 模块，但可以通过设置相应的配置选项 `--without-<module-name>_module` 来禁用它们。如果您在配置中没有使用这些模块，您可以安全地禁用它们。

### 表格：禁用配置选项

| 选项 | 解释 |
| --- | --- |
| `--without-http_charset_module` | charset 模块负责设置 `Content-Type` 响应头，并进行字符集之间的转换。 |
| `--without-http_gzip_module` | `gzip` 模块作为输出过滤器工作，在将内容交付给客户端时进行压缩。 |
| `--without-http_ssi_module` | 该模块是一个处理服务器端包含（SSI）的过滤器。如果启用了 Perl 模块，还可以使用额外的 SSI 命令（`perl`）。 |
| `--without-http_userid_module` | `userid` 模块使 NGINX 能够设置可以用于客户端识别的 cookies。然后，可以记录变量 `$uid_set` 和 `$uid_got` 进行用户跟踪。 |
| `--without-http_access_module` | `access` 模块控制基于 IP 地址访问某个位置。 |
| `--without-http_auth_basic_module` | 该模块通过 HTTP 基本认证限制访问。 |
| `--without-http_autoindex_module` | `autoindex` 模块使 NGINX 能够为没有索引文件的目录生成目录列表。 |
| `--without-http_geo_module` | 此模块使您能够根据客户端的 IP 地址设置配置变量，然后根据这些变量的值采取相应的操作。 |
| `--without-http_map_module` | `map` 模块使您能够将一个变量映射到另一个变量。 |
| `--without-http_split_clients_module` | 此模块创建可用于 A/B 测试的变量。 |
| `--without-http_referer_module` | 该模块使 NGINX 能够根据 Referer HTTP 头部阻止请求。 |
| `--without-http_rewrite_module` | `rewrite` 模块允许您根据各种条件更改 URI。 |
| `--without-http_proxy_module` | `proxy` 模块允许 NGINX 将请求转发到另一个服务器或服务器组。 |
| `--without-http_fastcgi_module` | FastCGI 模块使 NGINX 能够将请求转发到 FastCGI 服务器。 |
| `--without-http_uwsgi_module` | 此模块使 NGINX 能够将请求转发到 uWSGI 服务器。 |
| `--without-http_scgi_module` | SCGI 模块使 NGINX 能够将请求转发到 SCGI 服务器。 |
| `--without-http_memcached_module` | 此模块使 NGINX 能够与 memcached 服务器交互，将查询响应存储在变量中。 |
| `--without-http_limit_conn_module` | 此模块使 NGINX 能够基于某些键（通常是 IP 地址）设置连接限制。 |
| `--without-http_limit_req_module` | 使用此模块，NGINX 可以按键限制请求速率。 |
| `--without-http_empty_gif_module` | 空 GIF 模块生成一个 1 x 1 像素的内存透明 GIF。 |
| `--without-http_browser_module` | 浏览器模块允许根据 `User-Agent` HTTP 请求头部进行配置。变量根据此头部中的版本进行设置。 |
| `--without-http_upstream_ip_hash_module` | 该模块定义了一组服务器，可以与各种代理模块一起使用。 |

# 寻找和安装第三方模块

与许多开源项目一样，NGINX 周围有一个活跃的开发者社区。得益于 NGINX 的模块化特性，该社区能够开发并发布模块，提供额外的功能。它们涵盖了广泛的应用，因此在开发自己的模块之前，查看可用的模块是值得的。

安装第三方模块的过程相当直接：

1.  查找您想使用的模块（可以在 [`github.com`](https://github.com) 搜索，或者查看 [`wiki.nginx.org/3rdPartyModules`](http://wiki.nginx.org/3rdPartyModules)）。 

1.  下载模块。

1.  解压源代码。

1.  阅读 README 文件（如果有）。查看是否有需要安装的依赖项。

1.  按如下方式配置 NGINX 使用该模块：`/configure –add-module=<path>`。

这个过程将为你提供一个带有额外模块功能的`nginx`二进制文件。

请记住，许多第三方模块都是实验性质的。在将某个模块投入生产系统之前，先进行测试。还要记住，NGINX 的开发版本可能会有 API 变动，导致第三方模块出现问题。

这里特别提到`ngx_lua`第三方模块。`ngx_lua`模块用于在配置时启用 Lua，而不是 Perl 作为嵌入式脚本语言。这个模块相较于`perl`模块的巨大优势在于其非阻塞特性以及与其他第三方模块的紧密集成。安装说明在[`wiki.nginx.org/HttpLuaModule#Installation`](http://wiki.nginx.org/HttpLuaModule#Installation)中有详细描述。接下来我们将以这个模块为例，介绍如何安装第三方模块。

# 整合全部内容

现在你已经初步了解了各种配置选项的作用，你可以设计一个完全符合你需求的二进制文件。以下示例指定了前缀、用户、组、某些路径，禁用了部分模块，启用了其他模块，并包含了几个第三方模块：

```
$ export BUILD_DIR=`pwd`
$ export NGINX_INSTALLDIR=/opt/nginx
$ export VAR_DIR=/home/www/tmp
$ export LUAJIT_LIB=/opt/luajit/lib
$ export LUAJIT_INC=/opt/luajit/include/luajit-2.0

$ ./configure \
        --prefix=${NGINX_INSTALLDIR} \
        --user=www \
        --group=www \
        --http-client-body-temp-path=${VAR_DIR}/client_body_temp \
        --http-proxy-temp-path=${VAR_DIR}/proxy_temp \
        --http-fastcgi-temp-path=${VAR_DIR}/fastcgi_temp \
        --without-http_uwsgi_module \
        --without-http_scgi_module \
        --without-http_browser_module \
        --with-openssl=${BUILD_DIR}/../openssl-1.0.1c \
        --with-pcre=${BUILD_DIR}/../pcre-8.32 \
        --with-http_ssl_module \
        --with-http_realip_module \
        --with-http_sub_module \
        --with-http_flv_module \
        --with-http_gzip_static_module \
        --with-http_gunzip_module \
        --with-http_secure_link_module \
        --with-http_stub_status_module \
        --add-module=${BUILD_DIR}/ngx_devel_kit-0.2.17 \
        --add-module=${BUILD_DIR}/ngx_lua-0.7.9
```

在显示了`configure`在你的系统上找到的所有信息后，接下来会输出一个总结：

```
Configuration summary
  + using PCRE library: /home/builder/build/pcre-8.32
  + using OpenSSL library: /home/builder/build/openssl-1.0.1c
  + md5: using OpenSSL library
  + sha1: using OpenSSL library
  + using system zlib library

  nginx path prefix: "/opt/nginx"
  nginx binary file: "/opt/nginx/sbin/nginx"
  nginx configuration prefix: "/opt/nginx/conf"
  nginx configuration file: "/opt/nginx/conf/nginx.conf"
  nginx pid file: "/opt/nginx/logs/nginx.pid"
  nginx error log file: "/opt/nginx/logs/error.log"
  nginx http access log file: "/opt/nginx/logs/access.log"
  nginx http client request body temporary files: "/home/www/tmp/client_body_temp"
  nginx http proxy temporary files: "/home/www/tmp/proxy_temp"
  nginx http fastcgi temporary files: "/home/www/tmp/fastcgi_temp"
```

如你所见，`configure`找到了我们需要的所有项目，并确认了我们对于某些路径的偏好。现在，你可以按照本章开头所述构建并安装你的`nginx`。

# 总结

本章介绍了 NGINX 可用的各种模块。通过编译自己的二进制文件，你可以定制`nginx`提供的功能。构建和安装软件对你来说应该不陌生，因此在创建构建环境或确保所有依赖项到位的部分上并未花费太多时间。NGINX 的安装应该符合你的需求，因此可以根据需要随意启用或禁用模块。

接下来我们将介绍 NGINX 基础配置的概述，以便让你了解如何进行一般的 NGINX 配置。
