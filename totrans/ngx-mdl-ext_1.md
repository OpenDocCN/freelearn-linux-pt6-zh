# 第一章：从源代码安装 Nginx 核心和模块

本章作为一个快速参考，介绍了使用二进制和源代码分发以及与不同模块和附加组件相关的编译选项来下载和安装 Nginx。

如果您正在阅读这本书，您已经熟悉 Nginx（发音为 engine-x）。因此，我们不会花太多时间讨论基础知识。但是，在进行高级主题之前，您需要有一个可用的 Nginx 副本。

# 安装二进制分发版

大多数 UNIX 和 Linux 发行版的包管理器仓库中都包含 Nginx。您可以使用平台上的包管理器命令来安装它。例如，在 Ubuntu 或 Debian 上使用 **apt-get**，在 Gentoo 上使用 **emerge**。对于 Red Hat、Fedora 或 CentOS，请参阅以下说明。

您可以在 Nginx 安装 Wiki 上找到适用于不同平台（如 Red Hat 和 Ubuntu）的二进制安装说明，地址是 [`wiki.nginx.org/Install`](http://wiki.nginx.org/Install)。不过，我们将在这里简要描述该过程，并引用 Wiki 内容。

## Red Hat、Fedora 和 CentOS

要添加 Nginx yum 仓库，请创建名为 `/etc/yum.repos.d/nginx.repo` 的文件，并粘贴以下配置之一：

+   **对于 CentOS**：

    ```
    [nginx]
    name=nginx repo
    baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
    gpgcheck=0
    enabled=1
    ```

+   **对于 RHEL**：

    ```
    [nginx]
    name=nginx repo
    baseurl=http://nginx.org/packages/rhel/$releasever/$basearch/
    gpgcheck=0
    enabled=1
    ```

CentOS、RHEL 和 Scientific Linux 各自的 `$releasever` 变量设置不同。因此，根据您的操作系统版本，将 `$releasever` 替换为 `5`（对于 5.x）或 `6`（对于 6.x）。因此，6.x 的 `baseurl` 应为 `baseurl=http://nginx.org/packages/rhel/6/$basearch/`。

## 官方 Debian/Ubuntu 包

将以下行追加到 `/etc/apt/sources.list` 文件中，并将代号替换为适合您所使用版本的代号，例如，Ubuntu 13.10 的代号是 `saucy`：

+   **对于 Ubuntu 12.10**：

    ```
    deb http://nginx.org/packages/ubuntu/ saucy nginx
    deb-src http://nginx.org/packages/ubuntu/ saucy nginx
    ```

    请注意，在添加仓库后运行 apt-get update 时，您会遇到无法验证密钥的 **GPG** 错误。如果发生这种情况，且您觉得很难忽略它，请执行以下操作：

    ```
    wget http://nginx.org/packages/keys/nginx_signing.key
    cat nginx_signing.key | sudo apt-key add -
    ```

+   **对于 Debian 6**：

    ```
    deb http://nginx.org/packages/debian/ squeeze nginx
    deb-src http://nginx.org/packages/debian/ squeeze nginx

    ```

+   **对于 Ubuntu PPA**：此 PPA 由志愿者维护，非 [nginx.org](http://nginx.org) 提供。它包含一些额外的编译模块，可能更适合您的环境。您可以从 Launchpad 上的 **Nginx PPA** 获取最新的稳定版本。执行以下命令时，您需要管理员权限。

    +   **对于 Ubuntu 10.04 及更新版本**：

        ```
        sudo -s
        nginx=stable # use nginx=development for latest development version
        add-apt-repository ppa:nginx/$nginx
        apt-get update 
        apt-get install nginx

        ```

        如果遇到关于 add-apt-repository 不存在的错误，您需要安装 python-software-properties。对于其他基于 Debian/Ubuntu 的发行版，您可以尝试使用适用于较旧包集的 PPA 的 lucid 变体。

        ```
        sudo -s
        nginx=stable # use nginx=development for latest developmentversion
        echo "deb http://ppa.launchpad.net/nginx/$nginx/ubuntu lucidmain" > /etc/apt/sources.list.d/nginx-$nginx-lucid.list
        apt-key adv --keyserver keyserver.ubuntu.com --recv-keysC300EE8C
        apt-get update
        apt-get install nginx

        ```

## FreeBSD

使用以下命令更新 BSD 的 ports 树：

```
# portsnap fetch update

```

使用以下命令安装 Web 服务器：

```
# cd /usr/ports/www/nginx
# make install clean

```

输入以下命令来启动 Nginx 服务器：

```
echo 'nginx_enable="YES"' >> /etc/rc.conf

```

启动 Nginx，请输入：

```
# /usr/local/etc/rc.d/nginx start

```

## OpenBSD

从版本 5.1 开始，OpenBSD 将 Nginx 作为基本系统的一部分。这意味着 Nginx 与所有库依赖一起预装。虽然版本不一定是最新的，但这使你可以开始将 Apache 应用程序迁移到 Nginx。未来，预计默认的 httpd 将是 Nginx，而不是 Apache。

## 官方 Win32 二进制文件

从 Nginx 版本 0.8.50 开始，官方 Windows 二进制文件现已提供。

由于当前的构建过程使用 WineTools，因此 Windows 版本目前仅以二进制形式提供。你将无法从源代码编译 Windows 版本。下载 Windows ZIP 文件后，请执行以下步骤：

1.  **安装**：

    ```
    cd c:\
    unzip nginx-1.2.3.zip
    ren nginx-1.2.3 nginx
    cd nginx
    start nginx

    ```

1.  **控制**：

    ```
    nginx -s [ stop | quit | reopen | reload ]

    ```

对于问题，请查看`c:\nginx\logs\error.log`文件或`EventLog`。

# 安装源代码分发版

Nginx 二进制包通常是过时的，并且版本较旧。如果你平台上的二进制文件不是最新的，你可以从[`nginx.org/en/download.html`](http://nginx.org/en/download.html)下载源代码。在写这章时，版本 1.4.3 是稳定的可下载版本。

你也可以检出或克隆最新的源代码。

只读的 Subversion 仓库：

`code: svn://svn.nginx.org/nginx`

只读的 Mercurial 仓库：

`site: http://hg.nginx.org/nginx.org`

下载源代码包后，解压并使用以下标准构建命令来构建标准的二进制文件：

```
./configure
make
sudo make install

```

这会将 Nginx 二进制文件放在`user/local`目录下。然而，你可以通过配置选项覆盖此路径。

## Nginx 库依赖

如果你想从源代码构建 Nginx，至少需要以下库：

+   GCC

+   Autotools（automake 和 autoconf）

+   PCRE（Perl 兼容正则表达式）

+   zlib

+   OpenSSL

你也可以选择禁用 PCRE、zlib 和 OpenSSL 的依赖，方法是禁用重写、gzip 和 ssl 模块的编译。这些模块默认启用。

## 配置选项

编译时选项通过`configure`提供。你还可以在线查找与配置时选项相关的文档，地址为[`wiki.nginx.org/InstallOptions`](http://wiki.nginx.org/InstallOptions)。

`configure`命令定义了系统的各个方面，包括 Nginx 允许用于连接处理的方法。最后，它会创建一个 makefile。你可以使用`./configure --help`查看`configure`命令支持的所有选项。

以下部分摘自 Nginx 在线 wiki。

### 文件和权限

+   `--prefix=path`：默认设置为`/usr/local/nginx`目录。这里指定的路径是存放服务器文件的根文件夹，包括可执行文件、服务器日志文件、配置文件和 HTML 文档。

+   `--sbin-path=path`：默认的 Nginx 可执行文件名是`/sbin/nginx`。你可以通过此配置选项更改名称。

+   `--conf-path=path`：此选项的默认值是 `prefix/conf/nginx.conf`。这是 Nginx 配置文件的默认路径。正如你稍后会学到的，这个文件用于配置 Nginx 服务器的所有内容。该路径还包含一些其他配置文件，如示例 `fastcgi` 配置文件和字符集映射。你可以随时在配置文件中更改该路径。

+   `--pid-path=path`：此选项允许你更改 pid 文件的名称。pid 文件由各种工具（包括启动/停止脚本）使用，用来确定服务器是否在运行。通常，它是一个包含服务器进程 ID 的纯文本文件。默认情况下，文件名为 `prefix/logs/nginx.pid`。

+   `--error-log-path=path`：此选项允许你指定错误日志的位置。默认情况下，该文件名为 `prefix/logs/error.log`。你可以将该值设置为 `stderr`，这将把所有错误信息重定向到系统的标准错误输出。通常，这将是你的控制台或屏幕。

+   `--http-log-path=path`：此选项设置记录所有 HTTP 请求的日志文件名称。默认情况下，该文件名为 `prefix/logs/access.log`。像其他选项一样，你可以通过在配置文件中提供 `access_log` 指令随时更改它。

+   `--user=USER`：此选项设置将用于运行 Nginx 工作进程的用户名。你应该确保这是一个非特权或非 root 用户。默认的用户名是 `nobody`。你可以通过配置文件中的 `user` 指令稍后进行更改。

+   `--group=name`：此选项设置用于运行工作进程的组名。默认的组名是 `nobody`。你可以通过配置文件中的 `user` 指令更改它。

### 事件循环

Nginx 高效和稳定的原因之一是其能够使用基于事件的功能。输入/输出。基于事件的编码通过允许非阻塞操作来确保在单核中实现最佳性能。然而，基于事件的代码需要底层平台的支持，如 `kqueue`（FreeBSD、NetBSD、OpenBSD 和 OSX）、`epoll`（Linux）以及 `/dev/poll`（Solaris、HPUX）。如果这些方法不可用，Nginx 也可以使用更传统的 `select()` 和 `poll()` 方法。以下选项会影响这种行为：

+   `--with-select_module`

+   `--without-select_module`

这些选项启用或禁用构建一个模块，使服务器能够与 `select()` 方法一起工作。如果平台似乎不支持更合适的方法，如 `kqueue`、`epoll`、`rtsig` 或 `/dev/poll`，该模块将自动构建。

+   `--with-poll_module`

+   `--without-poll_module`

这些选项启用或禁用构建一个模块，使服务器能够与 `poll()` 方法一起工作。如果平台似乎不支持更合适的方法，如 `kqueue`、`epoll`、`rtsig` 或 `/dev/poll`，该模块将自动构建。

### 可选模块

可选模块如下：

+   `--without-http_gzip_module`：此选项允许你禁用网络传输压缩。如果你正在通过 HTTP 发送或接收大型文本文件，这个选项非常有用。然而，如果你不想将此压缩功能集成到 Nginx 二进制文件中，或者没有访问 zlib 库（启用此支持所需的库），可以使用此选项将其禁用。

+   `--without-http_rewrite_module`：此选项允许你禁用 HTTP 重写模块。HTTP 重写模块允许你通过修改符合给定模式的 URI 来重定向 HTTP 请求。启用此模块需要 PCRE 库。

+   `--without-http_proxy_module`：禁用 `ngx_http_proxy_module`。代理模块允许你将 HTTP 请求转发到另一个服务器。

+   `--with-http_ssl_module`：启用服务器的 SSL 支持。默认情况下此功能未启用，并且你需要 OpenSSL 来在 Nginx 二进制文件中构建 SSL 支持。

+   `--with-pcre=path`：如果你在本地机器上下载了 PCRE 源代码，可以通过此参数提供其路径。Nginx 会在构建 Nginx 服务器之前自动构建此库。请确保 PCRE 源代码的版本为 4.4 或更高。

+   `--with-pcre-jit`：此选项会构建带有“即时编译”（just-in-time compilation）支持的 PCRE 库。通过将正则表达式转换为机器代码，这显著提高了模式匹配或重写的速度。

+   `--with-zlib=path`：如果你已经下载了 zlib 库的源代码，可以在此处提供其路径。Nginx 会在构建服务器二进制文件之前先构建 zlib 库。请确保源代码的版本为 1.1.3 或更高。

### 编译控制

编译控制如下：

+   `--with-cc-opt=parameters`：CFLAGS 变量的附加选项

+   `--with-ld-opt=parameters`：链接器（`LD_LIBRAY_PATH`）的附加参数，在 FreeBSD 上编译时，你需要提供 `--with-ld-opt="-L /usr/local/lib"`。

### 示例

参数使用示例（所有内容需在一行内输入）：

```
 ./configure
 --sbin-path=/usr/local/nginx/nginx
 --conf-path=/usr/local/nginx/nginx.conf
 --pid-path=/usr/local/nginx/nginx.pid
 --with-http_ssl_module
 --with-pcre=../pcre-4.4
 --with-zlib=../zlib-1.1.3

```

### 提示

在 FreeBSD 上使用系统的 PCRE 库时，应指定以下选项：

+   --with-ld-opt="-L /usr/local/lib" \

+   --with-cc-opt="-I /usr/local/include"

如果需要增加 `select()` 支持的文件数量，可以通过以下方式指定：

+   --with-cc-opt="-D FD_SETSIZE=2048"

### 自定义模块

Nginx 的一个重要优势是它的模块化设计。你可以集成第三方模块或你自己编写的模块。

`--add-module=path` 会将位于 `path` 的模块编译到 Nginx 二进制文件中。你可以在 [`wiki.nginx.org/3rdPartyModules`](http://wiki.nginx.org/3rdPartyModules) 找到可用的第三方 Nginx 模块列表。

### 调试

`--with-debug`启用调试日志记录。此选项在 Windows 二进制文件中已启用。一旦你使用该选项编译 Nginx，你需要通过配置文件中的`error_log`指令来设置调试级别，等等：

```
error_log /path/to/log debug
```

除了调试日志记录，你还可以将调试器附加到正在运行的 Nginx 版本。如果你打算这样做，请在 Nginx 二进制文件中启用调试符号。使用`-g`或`-ggdb`进行编译，并推荐的编译器优化级别为`O0`或`O2`（这会使调试器输出更易于理解）。优化级别`O3`会自动向量化代码，并引入某些其他优化，这会使调试变得更加困难。按如下方式设置 CFLAGS 环境变量并运行`configure`：

```
CFLAGS="-g -O0" ./configure ...

```

## 在其他平台上安装

让我们在以下平台上安装 Nginx：

+   Gentoo：要获取最新版本的 Nginx，请在你的 portage 配置文件`/etc/portage/package.keywords`中添加平台掩码。

    ```
    www-servers/nginx ~x86 (or ~amd64, etc)
    ```

+   X86/64 版本的 Solaris 构建可以在[`joyent.com/blog/ok-nginx-is-cool`](http://joyent.com/blog/ok-nginx-is-cool)找到。

+   MacOSX：安装 Xcode 或 Xcode 命令行工具，以获取所有所需的编译器和库。

+   解决像 PCRE 等依赖项的详细说明可以在 Solaris 10 u5 中找到，详见[`wiki.nginx.org/Installing_on_Solaris_10_u5`](http://wiki.nginx.org/Installing_on_Solaris_10_u5)。

## 验证你的 Nginx 安装

以下是验证 Nginx 是否已安装的步骤：

1.  一旦成功编译并构建了 Nginx，可以通过运行`nginx -V`命令来验证它。

1.  作为 root 用户，使用`prefix/nginx/sbin/nginx`运行 Nginx 服务器。

1.  一旦服务器运行，你将看到`nginx.pid`文件。该文件的位置取决于你在运行配置脚本时提供的选项。在 Ubuntu 上，默认位置是`/var/run/nginx.pid`。

在编辑完`nginx.conf`文件后，你可以重新加载 Nginx 配置。为此，向主进程发送 SIGNUP 信号。该进程的 PID 位于`nginx.pid`文件中。以下命令将在 Ubuntu 上重新加载配置：

```
kill -HUP `cat var/run/nginx.pid

```

# 总结

在本章中，我们学习了如何下载 Nginx 的二进制和源代码版本并安装二进制版本。我们还学习了如何从源代码编译并安装 Nginx，如何使用配置选项覆盖安装路径和其他属性，如何编译带有调试符号的 Nginx，最后，如何验证安装。

在下一章中，我们将学习更多关于核心 Nginx 模块的配置。
