# 第三章：全面记录：日志模块

本章将涵盖：

+   设置错误日志路径和级别

+   像 Apache 一样记录日志

+   禁用在错误日志中记录 404 错误

+   在同一设置中使用不同的日志配置文件

+   启用日志文件缓存

+   为每个虚拟主机使用单独的错误日志

+   设置日志轮换

+   启用使用 syslog-ng 进行远程日志记录

+   设置自定义日志以便轻松解析

# 介绍

本章旨在教授基础和进阶的 Nginx 日志模块配置方法，如日志管理、备份、轮换等。日志记录非常关键，因为它可以帮助你识别和追踪应用程序的各类属性，如性能、用户行为等等。它还帮助你作为系统管理员，主动和被动地识别潜在的安全问题。

# 设置错误日志路径和级别

正确配置日志模块所需的基本设置是确定日志文件的位置并配置日志记录的级别。

Nginx 允许清晰地区分访问日志和错误日志，因此可以轻松跟踪错误列表。

## 如何操作...

你可以使用以下配置来设置日志文件路径，同时设置日志的格式：

```
http {
log_format combined '$remote_addr - $remote_user [$time_local] '
'"$request" $status $body_bytes_sent '
'"$http_referer" "$http_user_agent"';
access_log /var/log/nginx/access.log combined;
error_log /var/log/nginx/error.log crit;
...

```

## 它是如何工作的...

这个简单的配置将允许我们记录在特定环境中托管的所有站点上发生的各种 HTTP 活动。以下是来自访问日志的一个示例日志：

```
204.15.240.18 - - [11/Nov/2010:10:57:50 -0800] "GET /public/images-redux/infobox_join_type.png HTTP/1.1" 200 10212 "http://example.com/public/css/style-redux.css" "Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US; rv:1.9.2.10) Gecko/20100914 Firefox/3.6.10"

```

很容易看到各种变量是如何按顺序输出的，这些顺序是通过组合格式定义的。

错误日志现在记录在 `/var/log/nginx/error.log` 中，将开始记录所有的关键错误，以下是一个错误日志的示例条目：

```
2010/11/11 10:07:28 [error] 3172#0: accept() failed (53: Software caused connection abort)
2010/11/11 10:15:12 [error] 3175#0: *136332 open() "/root/sites/app/public/images/fancybox/blank.gif" failed (2: No such file or directory), client: 122.176.248.115, server: example1.com, request: "GET /public/images/fancybox/blank.gif HTTP/1.1", host: "example1.com", referrer: "http://example1.com/"

```

## 还有更多内容...

你可以根据应用需求更改错误日志级别（error_log）为 `debug,info,notice,warn,error,crit` 或 `alert`。通常最好尝试不同的级别，了解它们具体输出的内容，并判断它们是否能满足当前的调试需求。

| 错误级别 | 含义 |
| --- | --- |
| Alert | 紧急条件 |
| Crit | 严重条件 |
| Error | 错误条件 |
| Warn | 警告条件 |
| Notice | 普通但重要的条件 |
| Info | 信息性消息 |
| Debug | 调试级别消息 |

# 像 Apache 一样记录日志

Apache 或 httpd 是最常用的开源 Web 服务器。它具有非常稳定的代码库和社区，使其在开源企业应用中几乎成了标准。

大多数日志分析器在配置时会考虑 Apache 日志格式。本节的目标是使我们能够在 Nginx 设置中使用与 Apache 配合良好的日志分析工具。

## 如何操作...

如果你希望日志看起来像 Apache 日志，你需要输入以下代码：

```
log_format main '$remote_addr - $remote_user [$time_local] '
'"$request" $status $body_bytes_sent "$http_referer" '
'"$http_user_agent" "$http_x_forwarded_for"';
access_log /var/log/nginx/access.log main;

```

## 它是如何工作的...

在这里，我们基本上创建了一个新的 Apache 兼容格式，可以通过像 AWStats 这样的工具轻松读取。然后我们将前述配置中的标准设置为 Nginx 访问日志格式。该格式包含以下字段：

| 变量 | 它是什么？ |
| --- | --- |
| `$remote_addr` | 这是访问网站的远程地址的 IP。 |
| `$remote_user` | 如果用户通过 HTTP 认证登录，这将是他们的用户名。 |
| `$time_local` | 这是请求发起时服务器的本地时间戳。 |
| `$request` | 在服务器上发出的请求。 |
| `$status` | HTTP 响应代码（200、404、500 等）。 |
| `$body_bytes_sent` | 这是发送到服务器的响应大小。 |
| `$http_referer` | 这是用户从哪个网站访问到此页面/发起此 HTTP 请求。 |
| `$http_user_agent` | 这是发起此 HTTP 请求时使用的浏览器类型。 |
| `$http_x_forwarded_for` | 如果服务器作为反向代理运行，那么它将显示服务器的实际 IP。 |

```
66.249.64.13 - - [18/Sep/2004:11:07:48 +1000] "GET /robots.txt HTTP/1.0" 200 468 "-" "Googlebot/2.1"
66.249.64.13 - - [18/Sep/2004:11:07:48 +1000] "GET / HTTP/1.0" 200 6433 "-" "Googlebot/2.1"

```

访问日志的前几行显示了格式的实际应用。这个特定的格式将与所有的网页日志解析和分析工具（如 webalizer 和 AWStats）兼容，完全无需修改。

# 禁用 404 错误日志的记录

在谷歌的时代，我们可以看到有成千上万的爬虫试图通过读取你所有的页面来从你的站点和内容中获取最大的收益。在许多情况下，当你更新或升级网站时，这些爬虫开始占用系统资源，试图访问以前存在但现在已经不存在的页面。这还会增加日志记录的系统开销，可能成为你网站的瓶颈。本教程将解决这个问题。

## 如何实现...

这段配置被放置在配置文件的 location 上下文中，如下方代码所示：

```
location = /robots.txt {
log_not_found off;
}

```

## 它是如何工作的...

这个简单的配置在服务器上找不到 `/robots.txt` 文件时不会记录日志。这样可以避免不必要地打开错误日志文件并写入 404 条目，指示文件 robots.txt（cit）未找到。

# 在同一配置中使用不同的日志配置文件

正如我们之前所见，Nginx 允许你轻松设置日志格式。在本教程中，我们将探讨如何通过一个配置文件利用多个日志格式。这种精巧的功能可以帮助你在必要时生成针对网站特定部分的自定义日志。

## 如何实现...

这个特定的配置将实现三种日志格式，并有效地将它们应用于记录网站的不同部分：

```
http {
log_format main '$remote_addr - $remote_user [$time_local] '
'"$request" $status $body_bytes_sent "$http_referer" '
'"$http_user_agent" "$http_x_forwarded_for"';
# You do not need HTTP authentication information and Refer information for the static files !
log_format static_main '$remote_addr [$time_local] '
'"$request" $status $body_bytes_sent
'"$http_user_agent";
# You do not need to know about the bytes sent in an error logging case
log_format error_main '$remote_addr - $remote_user [$time_local] '
'"$request" $status "$http_user_agent";
...
server {
listen 80;
server_name example1.com;
error_log var/log/nginx/example1_error.log error_main;
location / {
...
access_log /var/log/nginx/example1_main.log main;
}
location /static/ {
...
access_log /var/log/nginx/example1_static.log static_main;
}
}
...
}

```

## 它是如何工作的...

`main` 日志格式将用于记录正常的动态 PHP 请求，而 `static_main` 日志格式将用于记录到达 Nginx 的静态请求。最后，我们使用 `error_main` 格式来跟踪错误。

## 还有更多内容...

你可以访问以下变量，在 `log_format` 结构中使用它们。这些变量可以有效地帮助你收集并理解 Nginx 在你的栈中可以访问的所有内容：

| 变量 | 描述 |
| --- | --- |
| `$arg_PARAMETER` | 这个变量包含 GET 请求参数 PARAMETER 在查询字符串中的值（如果存在）。 |
| `$args` | 这个变量是请求行中的 GET 参数，例如 `foo=123&bar=blahblah`。 |
| `$binary_remote_addr` | 客户端的二进制地址。 |
| `$body_bytes_sent` | 发送的正文字节数。 |
| `$content_length` | 这个变量等于请求头中的 Content-Length 行。 |
| `$content_type` | 这个变量等于请求头中的 Content-Type 行。 |
| `$document_root` | 这个变量等于当前请求的指令 root 的值。 |
| `$document_uri` | 与 `$uri` 相同。 |
| `$host` | 这个变量等于请求头中的 Host 行，或者当 Host 头部不可用时，等于处理请求的服务器的名称。当 Host 输入头缺失或为空时，这个变量的值可能与 `$http_host` 不同。 |
| `$http_HEADER` | 转换为小写并将 HTTP 头部中的 HEADER 的“短横线”转换为“下划线”后的值，例如，`$http_user_agent, $http_referer`。 |
| `$is_args` | 如果 `$args` 被设置，返回“?”，否则返回空字符串 ""。 |
| `$request_uri` | 这个变量等于从客户端接收到的*原始*请求 URI，包括参数。它无法被修改。查看 `$uri` 获取重写或修改后的 URI。不会包含主机名。示例：`"/foo/bar.php?arg=baz"`。 |
| `$scheme` | HTTP 协议（即 http, https）。仅在需要时计算，例如：`rewrite ^(.+)$ $scheme://example.com$1 redirect`; |
| `$server_addr` | 等于服务器地址。通常，为了获取此变量的值会进行一次系统调用。为了避免系统调用，需要在指令中指定地址，监听，并使用 bind 参数。 |
| `$server_name` | 服务器的名称。 |
| `$server_port` | 这个变量等于服务器的端口号，接收到请求的端口。 |
| `$server_protocol` | 这个变量等于请求的协议，通常是 HTTP/1.0 或 HTTP/1.1。 |
| `$uri` | 这个变量等于当前请求的 URI（不包含参数，参数在 `$args` 中）。它可能与 `$request_uri` 不同，后者是浏览器发送的 URI。修改的例子有内部重定向，或使用索引。不会包含主机名。示例：`"/foo/bar.html"`。 |

# 启用日志文件缓存

日志记录主要是基于磁盘的活动，在繁忙的服务器上，这需要作为审计要求进行日志记录。确保在 Nginx 中启用文件描述符缓存至关重要。这是一个性能提升方案，也将延长服务器硬盘的使用寿命。

## 如何实现...

你可以将此配置放入配置的`http`部分：

```
http {
...
open_log_file_cache max=1000 inactive=20s min_uses=2 valid=1m;
..

```

## 它是如何工作的...

这个简单的配置设置了以下标志，具体描述如下表：

| 标志 | 功能 |
| --- | --- |
| 最大 | 缓存中描述符的最大数量，溢出时最近最少使用的描述符将被移除（LRU） |
| 不活跃 | 设置没有命中事件的描述符被移除的时间；默认值为 10 秒 |
| min_uses | 设置在参数 inactive 指定的时间内文件的最小使用次数，超过该次数后，文件描述符将被放入缓存；默认值为 1 |
| 有效 | 设置检查文件是否仍以相同名称存在的时间；默认值为 60 秒 |
| 关闭 | 禁用缓存 |

这些设置可以通过一段时间的测试进行优化，从而为你提供 Nginx 在日志记录方面的最佳性能，并减少相应的性能开销。

# 为每个虚拟主机使用单独的错误日志

我们之前看过，在 Nginx 中设置虚拟主机并清晰地管理它们在各自的文件中是多么简单。在这个方案中，我们将看看如何为每个虚拟主机创建不同的访问日志和错误日志。

## 如何实现...

这是一个配置，涉及三个虚拟主机（www.example1.com，[www.example2.com](http://www.example2.com) 和 [www.example3.com](http://www.example3.com)），它们具有不同的访问和错误日志：

```
http {
...
server {
listen 80;
server_name www.example1.com ;
access_log /var/log/nginx/example1.access.log;
error_log /var/log/nginx/example1.error.log;
...
}
server {
listen 80;
server_name www.example2.com ;
access_log /var/log/nginx/example2.access.log;
error_log /var/log/nginx/example2.error.log;
...
}
server {
listen 80;
server_name www.example3.com ;
access_log /var/log/nginx/example3.access.log;
error_log /var/log/nginx/example3.error.log;
...
}
}

```

## 它是如何工作的...

如你所见，我们可以将`access_log`和`error_log`指令单独放入每个虚拟主机的配置中。这样，我们可以为每个站点创建不同的文件。

## 还有更多...

你可以将前面关于不同日志格式的配置与此方案结合，为每个站点创建不同的访问日志和错误日志。这清楚地展示了 Nginx 日志模块为系统管理员带来的强大功能。

```
http {
...
server {
listen 80;
server_name www.example1.com ;
access_log /var/log/nginx/example1.access.log main;
error_log /var/log/nginx/example1.error.log error_main;
...
}

```

上面的示例展示了如何使用不同的格式记录访问日志和错误日志，分别是`main`和`error_main`。在某些情况下，可能需要记录较少的变量来进行访问日志，并在错误日志中跟踪更多晦涩的变量。

# 设置日志轮换

在已经运行了一段时间的生产站点中，日志归档变得非常必要。为了进行正确的日志归档，你需要设置一个合适的日志轮换系统。每个网站请求都会生成多个日志条目（因为静态文件也会记录日志），因此日志会迅速膨胀。此方法可以帮助你处理 Nginx 的日志轮换设置，确保你正确归档日志。

这取决于可用的 logrotate 脚本，例如 Fedora 和 Debian 发行版上都有。

## 如何操作...

你需要向 logrotate 配置文件中添加一个配置：

```
/var/log/nginx/*.log {
daily
missingok
rotate 52
compress
delaycompress
notifempty
create 640 root adm
sharedscripts
postrotate
[ ! -f /var/run/nginx.pid ] || kill -USR1 `cat
/var/run/nginx.pid`
endscript
}

```

这假设 Nginx 配置文件位于 `/var/log/nginx` 目录，并且 Nginx PID 文件位于 `/var/run/nginx.pid`。

## 它是如何工作的...

这是一个简单的 logrotate 配置，能有效执行以下步骤：

1.  将现有日志文件移动并重命名为新文件名，并进行压缩。

1.  向 Nginx 主进程发送 USR1 信号，它将释放刚刚移动的日志并开始写入正常的日志文件。

logrotate 脚本提供了非常有趣的配置选项，允许你在日志需要轮换时进行精细控制，选择需要的压缩方式，以及设置归档文件的权限。

# 启用通过 syslog-ng 进行远程日志记录

想象一下运行一个分布在不同地理位置的服务器集群。在这种情况下，你可能需要在一组冗余的中央日志服务器上执行远程日志记录。这样可以更轻松地进行日志解析和系统管理任务。

在本方法中，我们将查看 syslog-ng 和 Nginx 如何在网络环境中协同工作。这将涉及一些有趣的内容，比如修改 Nginx 的代码库。

## 如何操作...

如果你希望让 Nginx 安装与 syslog-ng 进行交互，你需要小心执行以下步骤。本方法假设你已经在系统中安装了 syslog-ng：

1.  你需要从以下网址下载 Nginx：([`nginx.org/download/nginx-0.7.67.tar.gz`](http://nginx.org/download/nginx-0.7.67.tar.gz))

    ```
    wget http://nginx.org/download/nginx-0.7.67.tar.gz

    ```

1.  然后解压下载的文件：

    ```
    tar xvzf nginx-0.7.67.tar.gz

    ```

1.  下载补丁：([`bugs.gentoo.org/attachment.cgi?id=197180`](http://bugs.gentoo.org/attachment.cgi?id=197180))

    ```
    wget "http://bugs.gentoo.org/attachment.cgi?id=197180" -O syslog.patch

    ```

1.  应用补丁：

    ```
    patch -p0 < syslog.patch

    ```

1.  配置 Nginx：

    ```
    ./configure --with-syslog

    ```

1.  构建并安装 Nginx：

    ```
    make && make install

    ```

1.  你现在需要配置 syslog 客户端，将以下配置添加到 `/etc/syslog-ng/syslog-ng.conf` 中，并重启 syslog-ng 服务：

    ```
    filter f_local5 { facility(local5); };
    destination d_loghost {tcp("nginx_log" port(514));};
    log { source(s_all); filter(f_local5); destination(d_loghost); };

    ```

1.  你现在需要配置远程日志服务器 `nginx_log`，将以下配置添加到 `/etc/syslog-ng/syslog-ng.conf` 中，并重启 syslog-ng 服务：

    ```
    source s_remote { tcp(); };
    destination d_clients { file("/var/log/HOSTS/nginx.$HOST"); };
    log { source(s_remote); destination(d_clients); };

    ```

1.  你可以通过运行以下命令来测试此配置：

```
logger -p local5.info HelloWorld

```

## 它是如何工作的...

默认情况下，Nginx 不支持 syslog-ng，并且需要一些补丁才能正常工作。所以在第一步中，我们实际上是安装了带有补丁的 Nginx，然后继续配置 syslog-ng。

syslog-ng 配置分为两个部分。在第一部分中，我们实际上配置了运行 Nginx 的客户端，并使本地 5 级日志设施（在我们的案例中是 Nginx 日志）指向运行在 `nginx_log` 服务器上的 syslog-ng 服务器。第二部分涉及配置 syslog-ng 服务器以接受来自远程客户端的日志请求，并将其存储在硬盘的特定位置。

有一个小工具叫做 "logger"，它允许你在不启动 Nginx 的情况下测试你机器上的日志。它非常实用，可以让你轻松调试 syslog-ng 设置。

# 设置自定义日志以便于解析

日志记录的目的不仅是为了发现设置中的错误，还可以进行各种分析，分析运行在服务器上的站点的使用情况。有各种工具可以用来分析你的网页日志。一些开源且免费提供的工具包括 AWstats、webalizer 等。我们将看看如何为 AWstats 进行设置。

以下截图是 AWstats 生成的 HTML 报告示例：

![设置自定义日志以便于解析](img/4965_03_01.jpg)

## 如何操作……

我们将首先看看如何安装 AWstats，然后配置它以围绕 Nginx 日志生成持续报告。

1.  将以下内容添加到 Nginx 配置文件中：

    ```
    log_format new_log
    $remote_addr - $remote_user [$time_local] $request '
    '"$status" $body_bytes_sent "$http_referer" '
    '"$http_user_agent" "$http_x_forwarded_for"';
    access_log logs/access.log new_log;

    ```

1.  现在我们需要安装 AWstats 包，下载最新版本并运行配置向导 `awstats_configure.pl` 来创建一个新的统计档案：

    ```
    -----> Check for web server install
    Enter full config file path of your Web server.
    Example: /etc/httpd/httpd.conf
    Example: /usr/local/apache2/conf/httpd.conf
    Example: c:Program filesapache groupapacheconfhttpd.conf
    Config file path ('none' to skip web server setup):
    #> none
    Enter
    Your web server config file(s) could not be found.
    You will need to setup your web server manually to declare AWStats
    script as a CGI, if you want to build reports dynamically.
    See AWStats setup documentation (file docs/index.html)
    -----> Update model config file '/usr/local/awstats/wwwroot/cgi-bin/awstats.model.conf'
    File awstats.model.conf updated.
    -----> Need to create a new config file ?
    Do you want me to build a new AWStats config/profile
    file (required if first install) [y/N] ?
    #> y
    Enter
    -----> Define config file name to create
    What is the name of your web site or profile analysis ?
    Example: www.mysite.com
    Example: demo
    Your web site, virtual server or profile name:
    #> www.example1.com
    www.example1.com
    Enter
    -----> Define config file path
    In which directory do you plan to store your config file(s) ?
    Default: /etc/awstats
    Directory path to store config file(s) (Enter for default):
    #>
    ----> Add update process inside a scheduler
    Sorry, configure.pl does not support automatic add to cron yet.
    You can do it manually by adding the following command to your cron:
    /usr/local/awstats/wwwroot/cgi-bin/awstats.pl -update -config=www.moabc.net
    Or if you have several config files and prefer having only one command:
    /usr/local/awstats/tools/awstats_updateall.pl now
    A SIMPLE config file has been created: /etc/awstats/awstats.www.example1.com.conf
    You should have a look inside to check and change manually main parameters.
    You can then manually update your statistics for 'www.example1.com' with command:
    > perl awstats.pl -update -config=www.example1.com
    You can also build static report pages for 'www.example1.com' with command:
    > perl awstats.pl -output=pagetype -config=www.example1.com
    Press ENTER to finish...

    ```

1.  你现在可以打开生成的文件 `/etc/awstats/awstats.www.example1.com.conf`，并将 LogFile 变量更新为 Nginx 日志文件的路径（假设它们正在进行日志轮换）。

    ```
    LogFile="/usr/local/nginx/logs/access_%YYYY-0%MM-0%DD-0.log"

    ```

1.  现在你可以使用以下命令测试新的日志分析。这也取决于你安装 AWstats 包的位置：

    ```
    /usr/local/awstats/wwwroot/cgi-bin/awstats.pl -update -config=www.example1.com

    ```

1.  现在你需要生成 HTML 格式的报告，因此你需要创建一个目录，然后运行 HTML 生成脚本：

    ```
    # mkdir /data/webroot/awstats
    # /usr/local/awstats/tools/awstats_buildstaticpages.pl -update
    -config=www.example1.com -lang=en -dir=/data/admin_web/awstats
    -awstatsprog=/usr/local/awstats/wwwroot/cgi-bin/awstats.pl

    ```

1.  你现在可以向 Nginx 添加一些配置，以便在你自己的域名上公开此 HTML 分析：

    ```
    location ~ ^/awstats/ {
    root /data/webroot/awstats;
    index index.html;
    access_log off;
    error_log off;
    charset utf-8;
    }

    ```

1.  现在你可以访问 [`example1.com/awstats/awstats.www.example1.com.html`](http://example1.com/awstats/awstats.www.example1.com.html) 查看生成的 HTML。

## 它是如何工作的……

这是一个相对简单的设置，我们首先在 Nginx 上设置日志格式，以便能够充分利用 AWstats 为我们生成的所有内容。然后我们继续安装 AWstats，这是一个 Perl 脚本集。我们为我们的域 `www.example1.com` 生成一个配置文件，并开始分析日志。除了基本分析外，我们还可以生成非常易于使用的 HTML 报告。

## 还有更多内容……

像 AWstats 这样的工具可以帮助你追踪以下内容：

+   访问量（独立访客数量）

+   访问时间和最后一次访问

+   用户认证（上次使用站点凭据登录的时间）

+   每周高峰时段（每小时的页面数量、点击率和每周流量）

+   主机/国家访客（页面数、点击率、字节数，检测到 269 个域名/国家，GeoIP 检测）

+   最近访问的主机列表和未解析的 IP 地址列表

+   阅读量最多的入口页和出口页

+   文件类型

+   站点压缩表（mod_gzip 或 mod_deflate）

+   操作系统（每种操作系统的页面数量、点击率、字节数，检测到 35 个操作系统）

+   使用的浏览器类型

+   机器人访问（检测到 319 个机器人）

+   蠕虫攻击（5 个蠕虫家族）

+   搜索引擎统计数据，关于哪些关键词将用户引导到你的网站

+   HTTP 协议错误（最近的检查未找到页面）

+   基于个性化 URL 和链接参数的其他报告，涉及综合营销的目的

+   通过添加“收藏书签”视图来优化你的网站

+   屏幕大小（需要在首页添加一些 HTML 标签）

+   浏览器支持的比例：Java、Flash、RealG2 阅读器、Quicktime 阅读器、WMA 阅读器、PDF 阅读器

+   负载均衡服务器集群报告的比例

在你第一次运行脚本后，之前的设置并没有完全自动化。我们可以更进一步，将这一切放入一个 cron 脚本中，这样我们就可以将其作为 cron 任务来运行。

你需要在你的 cron 中添加以下内容（crontab `e`）：

```
00 1 * * * /usr/local/awstats/tools/awstats_buildstaticpages.pl
-update -config=www.example1.com -lang=cn -dir=/data/admin_web/awstats
-awstatsprog=/usr/local/awstats/wwwroot/cgi-bin/awstats.pl

```

上面的 cron 任务基本上是在每天的 1:00AM 启动一个脚本。脚本的任务是解析并生成为其配置的站点的报告。
