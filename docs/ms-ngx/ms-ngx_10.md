# 附录 B. 重写规则指南

本附录旨在介绍 NGINX 的 `rewrite` 模块，并作为创建新规则以及将旧版 Apache 重写规则转换为 NGINX 格式的指南。在本附录中，我们将讨论以下内容：

+   介绍重写模块

+   创建新的重写规则

+   从 Apache 转换

# 介绍重写模块

NGINX 的重写模块是一个简单的正则表达式匹配器，结合了虚拟栈机。任何重写规则的第一部分是一个正则表达式。因此，可以使用括号定义某些部分为“捕获”，以后可以通过位置变量引用。位置变量是其值依赖于正则表达式中捕获顺序的变量。它们按数字标记，因此位置变量 `$1` 引用第一个括号匹配的内容，`$2` 引用第二个括号，依此类推。例如，参考以下正则表达式：

```
^/images/([a-z]{2})/([a-z0-9]{5})/(.*)\.(png|jpg|gif)$
```

第一个位置变量 `$1` 引用紧跟在 URI 开头的 `/images/` 字符串后面的一个两字母字符串。第二个位置变量 `$2` 引用一个由小写字母和数字 0 到 9 组成的五个字符的字符串。第三个位置变量 `$3` 大概是文件名。而从这个正则表达式中提取的最后一个变量 `$4` 是 `png`、`jpg` 或 `gif` 中的一个，出现在 URI 的最后。

`rewrite` 规则的第二部分是请求被重写的 URI。URI 可以包含由第一个参数指定的正则表达式中捕获的任何位置变量，或者任何在此层级的 NGINX 配置中有效的其他变量：

```
/data?file=$3.$4
```

如果该 URI 不匹配 NGINX 配置中的其他位置，则它将作为 `Location` 头部返回给客户端，并附带 301（永久移动）或 302（找到）HTTP 状态码，指示将执行的重定向类型。如果第三个参数为 `permanent` 或 `redirect`，则此状态码可以明确指定。

`rewrite` 规则的第三个参数也可以是 `last` 或 `break`，表示不会再处理任何 `rewrite` 模块的指令。使用 `last` 标志将导致 NGINX 搜索另一个与重写后的 URI 匹配的 `location`。

```
rewrite '^/images/([a-z]{2})/([a-z0-9]{5})/(.*)\.(png|jpg|gif)$' /data?file=$3.$4 last;
```

`break` 参数也可以单独作为指令使用，用来停止 `rewrite` 模块在 `if` 块或其他 `rewrite` 模块处于活动状态时的指令处理。以下代码假设某些外部方法被用来在客户端使用过多带宽时将 `$bwhog` 变量设置为非空且非零的值。然后，`limit_rate` 指令将强制执行较低的传输速率。这里使用 `break` 是因为我们通过 `if` 进入了 `rewrite` 模块，我们不希望继续处理此类指令：

```
if ($bwhog) {

    limit_rate 300k;

    break;

}
```

另一种停止处理`rewrite`模块指令的方式是将控制权`return`到主`http`模块来处理请求。这可能意味着 NGINX 直接将信息返回给客户端，但`return`通常与`error_page`结合使用，要么向客户端展示一个格式化的 HTML 页面，要么激活其他模块来完成请求的处理。`return`指令可以指示状态码、带有一些文本的状态码，或者带有 URI 的状态码。如果仅使用 URI 作为唯一参数，那么状态码默认为 302。当文本位于状态码之后时，这些文本成为响应的主体。如果使用 URI，则该 URI 成为`Location`头的值，客户端将被重定向到该位置。

作为示例，我们希望设置一个短文本作为某个位置的文件未找到错误的输出。我们使用等号（=）指定该位置，以精确匹配此 URI：

```
location = /image404.html {

    return 404 "image not found\n";

}
```

对该 URI 的任何调用将返回 HTTP 代码 404，并显示文本**image not found\n**。因此，我们可以在`try_files`指令的末尾或作为图像文件的错误页面使用`/image404.html`。

除了与重写 URI 相关的指令外，`rewrite`模块还包括`set`指令，用于创建新变量并设置其值。这在多种情况下很有用，从在某些条件满足时创建标志，到将命名参数传递给其他`locations`并记录所做的操作。

以下示例演示了这些概念的一些应用以及相应指令的使用：

```
http {

    # a special log format referencing variables we'll define later
    log_format imagelog '[$time_local] ' $image_file ' ' $image_type ' ' $body_bytes_sent ' ' $status;

    # we want to enable rewrite-rule debugging to see if our rule does
    #  what we intend
    rewrite_log on;

    server {

        root /home/www;

        location / {

            # we specify which logfile should receive the rewrite-ruledebug
            #  messages
            error_log logs/rewrite.log notice;

            # our rewrite rule, utilizing captures and positional variables
            #  note the quotes around the regular expression - theseare
            #  required because we used {} within the expression itself
            rewrite '^/images/([a-z]{2})/([a-z0-9]{5})/(.*)\.(png|jpg|gif)$' /data?file=$3.$4;

            # note that we didn't use the 'last' parameter above; if we had,
            #  the variables below would not be set because NGINX would
            #  have ended rewrite module processing

            # here we set the variables that are used in the custom log
            #   format 'imagelog'
            set $image_file $3;

            set $image_type $4;

        }

        location /data {

            # we want to log all images to this specially-formatted logfile
            #  to make parsing the type and size easier
            access_log logs/images.log imagelog;

            root /data/images;

            # we could also have used the $image-variables we defined
            #  earlier, but referencing the argument is more readable
            try_files /$arg_file /image404.html;

        }

        location = /image404.html {

            # our special error message for images that don't exist
            return 404 "image not found\n";

        }

}
```

以下表格总结了我们在本节中讨论的`rewrite`模块指令：

## 表格：重写模块指令

| 指令 | 说明 |
| --- | --- |
| `break` | 结束在同一上下文中找到的`rewrite`模块指令的处理。 |

| `if` | 评估一个条件，如果为真，则遵循使用以下格式在上下文中指定的`rewrite`模块指令： |

```
if (condition) { … }
```

条件可以是以下任意之一：

+   变量名：`false`，如果为空或任何以`0`开头的字符串

+   字符串比较：使用`=`和`!=`运算符

+   正则表达式匹配：使用`~`（区分大小写）和`~*`（不区分大小写）正向运算符及其负向运算符`!~`和`!~*`

+   文件存在性：使用`-f`和`! -f`运算符

+   目录存在性：使用`-d`和`! -d`运算符

+   文件、目录或符号链接存在性：使用`-e`和`! -e`运算符

+   文件可执行性：使用`-x`和`! -x`运算符

|

| `return` | 停止处理并将指定的代码返回给客户端。非标准代码 444 会在不发送任何响应头的情况下关闭连接。如果代码后面附带文本，该文本将被放入响应体中。如果在代码后给出了 URL，那么该 URL 将作为 `Location` 头的值。如果没有代码，URL 被视为代码 302。 |
| --- | --- |

| `rewrite` | 将 URI 从第一个参数中匹配的正则表达式所匹配的内容更改为第二个参数中的字符串。如果给定第三个参数，它是以下标志之一：

+   `last`：停止处理 `rewrite` 模块的指令，并搜索与更改后的 URI 匹配的位置

+   `break`：停止处理 `rewrite` 模块的指令

+   `redirect`：返回临时重定向（代码 302），用于 URI 不以方案开头的情况

+   `permanent`：返回永久重定向（代码 301）

|

| `rewrite_log` | 启用 `rewrite` 的 `notice` 级别日志记录，记录到 `error_log` 中。 |
| --- | --- |
| `set` | 将给定的变量设置为特定值。 |
| `unitialized_variable_warn` | 控制是否记录关于未初始化变量的警告。 |

# 创建新的重写规则

在从头创建新规则时，和任何配置块一样，首先要规划好确切需要做什么。你可以问自己以下一些问题：

+   我在 URL 中使用的是什么模式？

+   是否有多种方式可以访问特定页面？

+   我是否想将 URL 的某些部分捕获到变量中？

+   我是否正在重定向到本服务器以外的网站，或者我的规则是否可能再次被看到？

+   我是否想替换查询字符串参数？

在检查你的网站或应用的布局时，应该清楚地了解你在 URL 中使用的模式。如果有多种方式可以访问某个页面，创建一个重写规则，将请求永久重定向回客户端。利用这些知识，你可以构建你网站或应用的规范表示。这不仅使 URL 更加简洁，而且有助于你的网站更容易被发现。

例如，如果你有一个 `home` 控制器来处理默认流量，但也可以通过索引页面访问该控制器，那么用户可以通过以下 URI 访问相同的信息：

```
/
/home
/home/
/home/index
/home/index/
/index
/index.php
/index.php/
```

将包含控制器名称和/或索引页面的请求直接重定向回根目录会更加高效：

```
rewrite ^/(home(/index)?|index(\.php)?)/?$ $scheme://$host/ permanent;
```

我们指定了 `$scheme` 和 `$host` 变量，因为我们正在进行永久重定向（代码 301），并希望 NGINX 使用最初到达此配置行的相同参数来构造 URL。

如果你希望能够单独记录 URL 的各个部分，可以在正则表达式中对 URI 使用捕获。然后，将位置变量赋值给命名变量，这些命名变量将成为`log_format`定义的一部分。我们在前一节中看到过一个例子。各个组件基本如下：

```
log_format imagelog '[$time_local] ' $image_file ' ' $image_type ' ' $body_bytes_sent ' ' $status;

rewrite '^/images/([a-z]{2})/([a-z0-9]{5})/(.*)\.(png|jpg|gif)$' /data?file=$3.$4;

set $image_file $3;

set $image_type $4;

access_log logs/images.log imagelog;
```

当你的重写规则导致内部重定向或指示客户端调用包含该规则的定位时，必须特别小心以避免重写循环。例如，规则可能在服务器上下文中定义，并使用 `last` 标志，但在它引用的位置中定义时必须使用 `break` 标志。

```
server {

    rewrite ^(/images)/(.*)\.(png|jpg|gif)$ $1/$3/$2.$3 last;
    location /images/ {

        rewrite ^(/images)/(.*)\.(png|jpg|gif)$ $1/$3/$2.$3 break;

    }

}
```

将新的查询字符串参数作为重写规则的一部分传递是使用重写规则的目标之一。然而，当初始的查询字符串参数应被丢弃，且仅应使用规则中定义的参数时，必须在新参数列表的末尾添加一个 `?` 字符。

```
rewrite ^/images/(.*)_(\d+)x(\d+)\.(png|jpg|gif)$ /resizer/$1.$4?width=$2&height=$3? last;
```

# 从 Apache 转换

Apache 的强大 `mod_rewrite` 模块有着悠久的重写规则编写历史，互联网上的大多数资源都集中在这些内容上。当遇到 Apache 格式的重写规则时，可以通过遵循一些简单规则将其转换为 NGINX 可解析的形式。

## 规则 #1：用 try_files 替代目录和文件存在性检查

当遇到如下形式的 Apache 重写规则时：

```
RewriteCond %{REQUEST_FILENAME} !-f

RewriteCond %{REQUEST_FILENAME} !-d

RewriteRule ^(.*)$ index.php?q=$1 [L]
```

这可以最好地转换为如下 NGINX 配置：

```
try_files $uri $uri/ /index.php?q=$uri;
```

这些规则规定，当 URI 中指定的文件名既不是文件也不是磁盘上的目录时，应该将请求传递给当前上下文根目录中的 `index.php` 文件，并将参数 `q` 设置为与原始 URI 匹配的值。

在 NGINX 拥有 `try_files` 指令之前，不得不使用 `if` 来测试 URI 的存在性：

```
if (!-e $request_filename) {

    rewrite ^/(.*)$ /index.php?q=$1 last;

}
```

不要这样做。你可能会在互联网上看到建议你这样做的配置，但它们已经过时，或者是过时配置的复制。虽然严格来说这不是重写规则，因为 `try_files` 属于核心的 `http` 模块，但 `try_files` 指令在执行此任务时更加高效，这正是它的设计目的。

## 规则 #2：用位置替换对 REQUEST_URI 的匹配

许多 Apache 重写规则是为了放入 `.htaccess` 文件中，因为历史上用户最有可能自己访问这些文件。典型的共享主机提供商不会允许用户直接访问负责其网站的虚拟主机配置上下文，而是提供将几乎任何类型的配置放入 `.htaccess` 文件的能力。这导致了我们今天的情况，出现了大量针对 `.htaccess` 文件的重写规则。

虽然 Apache 也有 `Location` 指令，但它很少用来解决与 URI 匹配的问题，因为它只能在主服务器配置或虚拟主机的配置中使用。因此，我们会看到大量匹配 `REQUEST_URI` 的重写规则。

```
RewriteCond %{REQUEST_URI} ^/niceurl

RewriteRule ^(.*)$ /index.php?q=$1 [L]
```

最好通过在 NGINX 中使用 `location` 来处理此问题：

```
location /niceurl {

    include fastcgi_params;

    fastcgi_index index.php;

    fastcgi_pass 127.0.0.1:9000;

}
```

当然，`location`上下文内的内容取决于你的配置，但原则是相同的；与 URI 的匹配最好通过`location`来提供。

这个原则同样适用于有隐式`REQUEST_URI`的`RewriteRules`。这些通常是裸露的`RewriteRules`，它们将 URI 从旧格式转换为新格式。在以下示例中，我们看到`show.do`不再是必要的：

```
RewriteRule ^/controller/show.do$ http://example.com/controller [L,R=301]
```

这可以转化为如下的 NGINX 配置：

```
location = /controller/show.do {

    rewrite ^ http://example.com/controller permanent;

}
```

在我们看到`RewriteRule`时，不要过于激动地创建新的位置，我们应记住正则表达式是直接转换的。

## 规则#3：用服务器替换与 HTTP_HOST 匹配的规则

与*规则#2*密切相关，该规则考虑了试图去除或添加`www`到域名上的配置。这类重写规则通常出现在`.htaccess`文件中或在有重载`ServerAliases`的虚拟主机中：

```
RewriteCond %{HTTP_HOST} !^www

RewriteRule ^(.*)$ http://www.example.com/$1 [L,R=301]
```

这里，我们将没有`www`的 URL 中的`Host`部分转换为带有`www`的变体：

```
server {

    server_name example.com;

    rewrite ^ http://www.example.com$request_uri permanent;

}
```

在不需要`www`的情况下，我们输入以下规则：

```
RewriteCond %{HTTP_HOST} ^www

RewriteRule ^(.*)$ http://example.com/$1 [L,R=301]
```

这可以转化为如下的 NGINX 配置：

```
server {

    server_name www.example.com;

    rewrite ^ http://example.com$request_uri permanent;

}
```

未显示的是已经被重定向的变体的`server`上下文。之所以没有展示，是因为它与重写本身无关。

这个相同的原则不仅适用于匹配`www`或缺失`www`的情况，它还可以用于处理任何使用`%{HTTP_HOST}`的`RewriteCond`。这些重写最好在 NGINX 中通过使用多个`server`上下文来完成，每个上下文匹配一个所需的条件。

例如，我们有以下的 Apache 多站点配置：

```
RewriteCond %{HTTP_HOST} ^site1

RewriteRule ^(.*)$ /site1/$1 [L]

RewriteCond %{HTTP_HOST} ^site2

RewriteRule ^(.*)$ /site2/$1 [L]

RewriteCond %{HTTP_HOST} ^site3

RewriteRule ^(.*)$ /site3/$1 [L]
```

这基本上转化为一个根据主机名进行匹配，并且每个主机有不同`root`配置的配置。

```
server {

    server_name site1.example.com;
    root /home/www/site1;

}

server {

    server_name site2.example.com;

    root /home/www/site2;

}

server {

    server_name site3.example.com;

    root /home/www/site3;

}
```

这些基本上是不同的虚拟主机，因此最好在配置中也将它们视为不同的虚拟主机。

## 规则#4：用`if`替代`RewriteCond`进行变量检查

该规则仅在应用了规则 1 到规则 3 之后生效。如果仍有未被这些规则覆盖的条件，那么可以使用`if`来测试变量的值。任何 HTTP 变量都可以通过将变量名的小写形式加上`$http_`前缀来使用。如果变量名中有连字符（-），它们会被转换成下划线（_）。

以下示例（摘自 Apache 关于`mod_rewrite`模块的文档，网址为[`httpd.apache.org/docs/2.2/mod/mod_rewrite.html`](http://httpd.apache.org/docs/2.2/mod/mod_rewrite.html)）用于根据`User-Agent`头决定应该向客户端提供哪个页面：

```
RewriteCond  %{HTTP_USER_AGENT}  ^Mozilla 

RewriteRule  ^/$                 /homepage.max.html  [L] 

RewriteCond  %{HTTP_USER_AGENT}  ^Lynx 
RewriteRule  ^/$                 /homepage.min.html  [L] 

RewriteRule  ^/$                 /homepage.std.html  [L]
```

这可以转化为如下的 NGINX 配置：

```
if ($http_user_agent ~* ^Mozilla) { 

    rewrite ^/$ /homepage.max.html break; 

}

if ($http_user_agent ~* ^Lynx) { 

    rewrite ^/$ /homepage.min.html break; 

}

index homepage.std.html;
```

如果有些特殊变量只在 Apache 的`mod_rewrite`中可用，那么这些在 NGINX 中当然无法检查。

# 总结

我们在本附录中探讨了 NGINX 的`rewrite`模块。与该模块相关的指令并不多，但这些指令可以用来创建一些复杂的配置。希望通过逐步创建新的重写规则的过程，演示了如何轻松制作重写规则。在创建任何复杂规则之前，需要理解正则表达式的构建和阅读方法。我们最后讨论了如何将 Apache 风格的重写规则转换为 NGINX 可解析的配置。通过这样做，我们发现在 NGINX 中可以以不同的方式解决相当多的 Apache 重写规则场景。
