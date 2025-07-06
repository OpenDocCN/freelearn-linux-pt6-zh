# 第二章：配置核心模块

在本章中，我们将探讨 Nginx 的`Main`和`Events`模块的配置，这些模块也称为核心模块。我们将讨论与模块相关的配置选项和使用这些选项时需要记住的重要事项。

# 了解 Main 模块

Nginx 的`Main`模块包含以下配置指令或命令：

| 名称 | 值 | 默认值 | 示例 |
| --- | --- | --- | --- |
| daemon | on, off | on |   |
| master_process | on, off | on |   |
| timer_resolution | interval | 0 | 100ms |
| pid | file | logs/nginx.pid | /var/log/nginx.pid |
| lock_file | file | logs/nginx.lock | /var/log/nginx.lock |
| worker_processes | number, auto | 1 | 2 |
| debug_points | stop, abort | null | stop |
| user | user [group] | nobody nobody | www users |
| worker_priority | number | 0 | 15 |
| worker_cpu_affinity | cpu mask |   | 0101 1010 |
| worker_rlimit_nofile | number |   | 1000 |
| worker_rlimit_core | size |   | 500M |
| worker_rlimit_sigpending | number |   | 1000 |
| working_directory | directory | 编译时 | /usr/local/nginx |
| env | variable = value |   | PERL5LIB=/data/site/modules |

## 解释指令

接下来，我们将详细讨论前表中提到的`Main`模块的所有指令。

### daemon

`daemon`指令决定了 Nginx 是否会以守护进程模式运行。守护进程（或服务）是设计为在后台运行的进程，几乎不需要用户干预。从 Nginx 版本 1.0.9 开始，关闭此选项是安全的。如果关闭此选项，则无法在不中断 Nginx 的情况下进行二进制升级。

Nginx 的主进程负责不停地进行二进制升级。为此目的使用的信号是`SIGUSR2`。然而，当守护进程模式关闭时，这个信号会被忽略，因为主进程仍会使用旧的二进制文件。这不同于守护进程模式，在这种模式下，新的工作进程的父进程会切换为`Init`进程，因此可以启动一个使用新二进制文件的工作进程。唯一可能希望关闭`master_process`的原因是为了调试。关闭这些选项后，Nginx 将在前台运行，且没有主进程，你可以通过按下*Ctrl* + *R*来终止它。然而，你不应在生产环境中关闭守护进程模式和`master_process`。

### master_process

`master_process`指令决定了 Nginx 是否以主进程模式运行。它在 Nginx 中有许多职责，包括生成工作进程。如果你将`master_process`设为关闭，Nginx 将以单进程模式运行。你应该始终在生产环境中开启`master_process`选项。唯一可能关闭`master_process`的原因是调试，因为在单进程模式下调试代码可能更为简单。

### timer_resolution

设置`timer_resolution`指令将减少`gettimeofday()`系统调用的次数。系统不会对每个内核事件执行该系统调用，而是每个指定的时间间隔仅调用一次`gettimeofday()`。如果在繁忙的服务器上`gettimeofday()`被认为是一个开销，您可能需要关闭此选项。不过在空闲服务器上您不会注意到任何区别。使用`timer_resolution`的一个好理由是可以在日志中获得精确的上游响应时间。

### pid

`pid`指令指定主进程的`PID`文件的文件名。默认值相对于`--prefix`配置选项。

### lock_file

`lock_file`指令指定锁文件的文件名。默认值相对于`--prefix`配置选项。您还可以通过以下配置脚本在编译时指定此选项：

```
./configure --lock-path=/var/log/nginx.lock
```

### worker_processes

`worker_processes`指令定义了最大工作进程数。工作进程是由主进程生成的单线程进程。如果将此选项设置为`auto`，Nginx 将自动尝试确定 CPU/核心的数量，并将此选项设置为该数量。从版本 1.3.8 和 1.2.5 开始支持`auto`参数。如果您希望 Nginx 仅使用服务器上有限数量的核心，明确设置此值是个好主意。

### debug_points

`debug_points`指令对调试 Nginx 非常有用。如果将此指令设置为`abort`，Nginx 将在发生内部错误时生成核心转储文件。您可以使用此核心转储文件，在系统调试器如`gdb`中获取错误追踪。为了获取核心转储文件，您还需要配置`working_rlimit_core`和`working_directory`指令。

如果此指令设置为`stop`，任何内部错误将导致进程停止。

### user

`user`指令用于指定工作进程的用户和组。如果没有指定组名，则使用与用户相同的组名。您可以通过以下`configure`脚本覆盖默认的 nobody nobody，指定用户和组：

```
./configure --user=www --group=users
```

### worker_priority

`worker_priority`指令允许您设置工作进程的优先级。Nginx 将使用`setpriority()`系统调用来设置优先级。将优先级设置为较低的值可以促使工作进程更有利地进行调度。

实际的优先级范围在不同的内核版本之间有所不同。在 1.3.36 之前，Linux 的范围是从–infinity 到 15。自内核 1.3.43 起，Linux 的范围从-20 到 19。在某些系统中，优先级范围是从-20 到 20。

### worker_cpu_affinity

`worker_cpu_affinity` 指令允许你设置工作进程的亲和性掩码。亲和性掩码是位掩码，可以将进程绑定到特定的 CPU。这在一些系统上可以提高性能，比如 Windows 系统，那里有多个系统进程被限制在第一个 CPU/核心上。因此，排除第一个 CPU/核心可能会带来更好的性能。我们可以使用以下代码片段将每个工作进程绑定到不同的 CPU：

```
worker_processes    4;
worker_cpu_affinity 0001 0010 0100 1000;
```

你可以将 CPU 亲和性值设置为最多 64 个 CPU/核心。

### worker_rlimit_nofile

`worker_rlimit_nofile` 指令设置了工作进程可以使用的最大打开文件数限制（`RLIMIT_NOFILE`）。当设置此指令时，`setrlimit()` 系统调用会被用来设置资源限制。任何超出此限制的尝试将导致 `EFILE` 错误（表示打开的文件过多）。

### worker_rlimit_core

`worker_rlimit_core` 指令设置了工作进程创建的核心转储文件的最大大小。当设置此选项时，`setrlimit()` 系统调用会用来设置文件大小的资源限制。当此选项为 `0` 时，将不会创建核心转储文件。

### worker_rlimit_sigpending

`worker_rlimit_sigpending` 指令指定了调用进程的真实用户 ID 可以排队的信号数量限制。如果你使用的是版本大于 2.6.6 的 Linux 内核，它将支持实时信号（`rtsig`），并且每个进程将拥有自己的信号队列，而不是早期实现中的系统范围队列。当设置此选项时，`setrlimit()` 系统调用会用来设置资源限制 `RLIMIT_SIGPENDING`。为了执行此限制，标准信号和实时信号都会被计数。

### working_directory

`working_directory` 指令指定了核心转储文件创建的路径。此路径应为绝对路径。你必须确保通过 `user` 选项指定的用户或组在此文件夹中具有写入权限。

### env

`env` 指令允许你设置一些环境变量，这些变量将被工作进程继承。如果你在 shell 中设置了一些环境变量，它们将被 Nginx 清除，除了 `TZ`。此指令允许你保留一些继承的变量，修改它们的值，或创建新的环境变量。环境变量 `NGINX` 是服务器内部使用的，不应由用户设置。

如果你只指定了环境变量的名称，而没有指定其值，Nginx 将保留来自父进程的值，即来自 shell 的值。但如果你也指定了一个值，则服务器将使用这个新值。

```
env MALLOC_OPTIONS;
env PERL5LIB=/data/site/modules;
env OPENSSL_ALLOW_PROXY_CERTS=1;
```

在上面的代码片段中，环境变量 `MALLOC_OPTIONS` 的值将从父进程保留，而其他两个环境变量则会被定义或重新定义，父进程设置的任何值将被清除。

# 理解事件模块

`Events` 模块处理 `epoll`、`kqueue`、`select`、`poll` 等的配置。此模块包含以下指令：

| 名称 | 值 | 默认值 | 示例 |
| --- | --- | --- | --- |
| accept_mutex | 开启，关闭 | 开启 | 关闭 |
| accept_mutex_delay | 间隔（毫秒） | 500 毫秒 | 500 毫秒 |
| debug_connection | ip, cidr | 无 | 192.168.1.1 |
| devpoll_changes | 数字 | 32 | 64 |
| devpoll_events | 数字 | 32 | 64 |
| kqueue_changes | 数字 | 512 | 512 |
| kqueue_events | 数字 | 512 | 512 |
| epoll_events | 数字 | 512 | 512 |
| multi_accept | 开启，关闭 | 关闭 | 开启 |
| rtsig_signo | 信号编号 | SIGRTMIN+10 |   |
| rtsig_overflow_events | 数字 | 16 | 24 |
| rtsig_overflow_test | 数字 | 32 | 40 |
| rtsig_overflow_threshold | 数字 | 10 | 3 |
| use | Kqueue, rtsig, epoll, /dev/poll, select | 根据配置脚本决定 | rtsig |
| worker_connections | 数字 | 512 | 200 |

## 解释指令

现在我们将详细讨论前面表格中总结的 `Events` 模块的指令。

### accept_mutex

`accept_mutex` 指令尝试防止工作进程在 `accept()` 时竞争用于监听套接字的资源（内核中的套接字）。换句话说，如果没有 `accept_mutex`，工作进程可能会同时检查套接字上的新事件，这可能会导致 CPU 使用率略有上升。根据操作系统和事件通知机制的不同，结果可能会有所不同。如果你使用 `rtsig` 作为信号处理机制，必须启用此选项。

### accept_mutex_delay

启用 `accept_mutex` 会使工作进程的 `accept()` 调用串行化，也就是说，只有一个工作进程每次处理一个 `accept`。`accept_mutex_delay` 指令决定了当另一个进程正在进行 `accept` 时，其他工作进程等待多长时间才准备接受新连接。

### debug_connection

为了使用 `debug_connection` 指令，Nginx 需要在运行配置脚本时以 `--debug-enabled` 选项构建。你可以指定启用调试日志的客户端的 IPv4 或 IPv6 地址，也可以提供域名或 unix:（用于 Unix 域套接字连接）。其余客户端的日志级别通过 `error_log` 选项确定。调试日志将写入在 `error_log` 选项中指定的日志文件，具体代码如下：

```
error_log /var/log/nginx-error.log;
events {
    debug_connection 127.0.0.1;
    debug_connection localhost;
    debug_connection 192.0.2.0/24;	
    debug_connection ::1;
    debug_connection 2001:0db8::/32;
    debug_connection unix:;
    ...
}
```

### devpoll_changes 和 devpoll_events

`/dev/poll` 指令是一种高效的方法，用于 Solaris 7 11/99+、HP/UX 11.22+（eventport）、IRIX 6.5.15+ 和 Tru64 UNIX 5.1A+。`devpoll_changes` 和 `devpoll_events` 参数定义了可以从内核传入和传出的变更和事件数量。在处理事件时，Nginx 每次只会将 `devpoll_changes` 或 `devpoll_events` 数量的事件和变更写入 `/dev/poll`。

### kqueue_changes 和 kqueue_events

`kqueue`指令是一个高效的事件通知接口，支持 FreeBSD 4.1+、OpenBSD 2.9+、NetBSD 2.0 和 Mac OS X。它提供了一个高效的输入输出事件管道，在`kernel`和`user`进程之间进行数据交换。使用单个系统调用`kevent()`即可接收挂起的事件。这与较旧的传统轮询系统调用（如`poll`和`select`）形成对比，后者在轮询大量文件描述符上的事件时效率较低。其他先进的事件机制，如`/dev/poll`和`epoll`，也可以实现类似的功能。

这些参数控制在一次调用`kevent()`系统调用时传递的更改和事件的数量。

### epoll_events

`epoll_events`指令是 Linux 2.6+中使用的一种高效的事件通知方法。也有补丁可以将此功能移植到旧版本的内核中。此参数确定在事件处理之前，一次返回的事件的缓冲区大小。

### multi_accept

当启用`multi_accept`指令时，工作进程会尽可能接受所有或尽可能多的传入连接。关闭此选项将导致工作进程一次只接受一个连接。如果使用`kqueue_events`通知方法，此选项将被忽略。如果使用`rtsig`方法，此选项会自动启用。默认情况下，此选项是关闭的。它可以是一个高性能的调整选项；然而，`multi_accept`的缺点是，如果你有一个恒定且高频率的传入连接流，它可能会在没有任何机会处理已接受连接的情况下，溢出`worker_connections`指令。

### rtsig_signo

Linux 支持 32 个实时信号，编号从 32（`SIGRTMIN`）到 63（`SIGRTMAX`）。程序应始终使用`SIGRTMIN+n`的表示法来引用实时信号，因为实时信号编号的范围在*nix 平台上是不同的。Nginx 在使用`rtsig`方法时会使用两个信号。该指令指定第一个信号编号。第二个信号编号比第一个信号编号大 1。默认情况下，`rtsig_signo`是 SIGRTMIN+10（42），在这种情况下，第二个信号编号为 43。

### rtsig_overflow_events、rtsig_overflow_test 和 rtsig_overflow_threshold

`rtsig_overflow_events`、`rtsig_overflow_test`和`rtsig_overflow_threshold`指令定义了在使用实时信号时如何处理队列溢出。当溢出发生时，Nginx 会刷新`rtsig`队列，然后处理`poll()`和`rtsig poll()`之间事件的切换，并连续处理所有未处理的事件，同时`rtsig`会定期清空队列，以防止新的溢出。当溢出完全处理完毕后，Nginx 会重新切换回`rtsig`方法。

`rtsig_overflow_events`指令指定通过`poll()`传递的事件数量。

`rtsig_overflow_test` 指令指定了 `poll()` 处理的事件数量，超过这个数量后，Nginx 将清空 `rtsig` 队列。

`rtsig_overflow_threshold` 指令仅在 Linux 2.4.x 中有效。在清空 `rtsig` 队列之前，Nginx 会检查队列的填充程度。默认值为 1/10。`rtsig_overflow_threshold 3` 意味着值为 1/3。

### use

`use` 指令指定应使用的事件通知方法（`kqueue`、epoll 等）。对于有多个选项的平台，这非常有用。在配置时，系统会自动选择最佳的事件通知方法，因此在大多数情况下，你不需要设置此指令。

### worker_connections

`worker_connections` 指令指定了工作进程可以打开的最大同时连接数。

需要注意的是，这个数字包括所有连接，而不仅仅是客户端连接。另一个考虑因素是，实际的同时连接数不应超过当前的最大打开文件数限制，该限制可以通过 `worker_rlimit_nofile` 进行调整。

# 总结

本章我们介绍了 Nginx 两个核心模块的配置细节，即 `Main` 模块和 `Events` 模块。这些模块是不可选择的，你无法禁用它们。因此，我们逐一讲解了每个选项及其适用的场景。

在下一章，我们将讨论如何安装和配置 HTTP 模块以及与这些模块相关的配置选项。
