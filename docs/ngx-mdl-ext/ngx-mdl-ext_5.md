# 第五章 创建自己的模块

Nginx 允许通过用纯 C 语言编写新的模块来扩展功能。本章简要介绍了如何创建自己的模块。它是 Nginx 模块系统的快速参考，并介绍了使扩展成为可能的 Nginx 内部架构。它以高层次的方式介绍了可以创建的不同类别的模块和插件。本章还将简要介绍 NDK，这是 Nginx 中作为其他模块基础的特殊模块。

本章涵盖的主题如下：

+   Nginx 中模块链式调用和委托的概念

+   处理器模块

+   过滤器模块

+   负载均衡器模块

+   **Nginx 开发工具包** (**NDK**)：NDK 是一个 Nginx 模块，旨在以一种可以作为其他 Nginx 模块基础的方式扩展优秀的 Nginx Web 服务器的核心功能。

+   自定义 Nginx 模块的示例源代码

本章结束时，进阶用户将了解 Nginx 内部架构，并了解创建自己第三方模块的基础。读者应能学会如何使用 NDK；源代码将帮助他们看到一个非常简单的自编写模块如何运行。

# Nginx 模块委托

Nginx 拥有非常模块化的架构。Nginx 执行的所有主要操作都是通过模块完成的。所有 Nginx 模块在编译时构建，并且不会动态加载。

模块委托也可以称为 **模块链式调用**。核心基本上完成与设置连接和处理协议相关的基础工作。然后它设置一个模块链来执行，每个模块处理请求处理的某个阶段或步骤。

基于模块的非集中化架构使得高级用户能够开发出符合其需求的模块。

以下是不同类型的 Nginx 模块。

## 处理器

配置文件中为每个定义的位置都有一个处理器。当服务器启动时，处理器会附加或绑定到某个位置。理想情况下，每个位置应该只有一个处理器；如果在配置文件中定义了多个，只有一个会有效（通常是最后一个）。处理器会通过以下三种方式结束：成功（当一切正常时）、失败（当出现错误时），或者它们不会处理请求，而是让默认处理器处理请求。

## 负载均衡器

负载均衡器或上游模块将你的请求转发到配置的多个后端或上游中的一个。默认情况下，Nginx 内置了两个负载均衡模块：**轮询法**和 **IP 哈希** 方法（查看 `ngx_http_upstream_module`）。还有其他第三方模块可以让你基于各种哈希机制进行负载均衡，例如一致性哈希。

## 过滤器

在处理程序生成响应后，过滤器会被执行。过滤器对处理程序的响应进行后处理。举个例子，你可能需要压缩响应，或者添加某些头部信息。多个过滤器可以与每个位置相关联。

### 执行顺序

Nginx 过滤器的执行顺序在编译时确定。你可以在编译代码后查看 `ngx_modules.c` 中的执行顺序。这个文件是通过 `modules` 脚本即时生成的，该脚本位于 `nginx/auto/`。这个脚本确保了模块和过滤器执行顺序的正确性。

内建模块确实需要特定的顺序，例如，gzip 过滤器应该在头部和主体过滤器执行后运行。新的自定义模块通常会在最后执行。

过滤器并不是以完全阻塞的方式执行，而是过滤器的输出通过过滤器链进行流转。默认情况下，一个过滤器处理某些数据并将其传递给下一个模块，依此类推。每次处理的数据量通常是页面大小的倍数。不同的模块，比如 gzip，允许你调整这个值。

# 扩展版 "Hello world" 模块

现在我们将开始创建一个简单的 Nginx 模块。这个模块将在你输入特定位置时，在浏览器中显示一个可配置的文本。这是一个非常简单的模块，目的是介绍如何创建 Nginx 模块的核心概念。这个模块是基于并扩展了简单的 Hello world 模块，原文可以在 [`blog.zhuzhaoyuan.com/2009/08/creating-a-hello-world-nginx-module/`](http://blog.zhuzhaoyuan.com/2009/08/creating-a-hello-world-nginx-module/) 找到。这个模块是一个处理程序模块的示例。

## 编写和编译模块

你首先要做的显然是为你的新模块创建一个文件夹。将其创建在 Nginx 源代码树以外的地方。你应首先创建以下两个文件：

+   `config`

+   `ngx_http_hello_module.c`

`config` 文件的内容将取决于你编写的模块类型。

对于这个简单的模块，它的代码大致如下：

```
ngx_addon_name=ngx_http_hello_module
HTTP_MODULES="$HTTP_MODULES ngx_http_hello_module"
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_hello_module.c"
```

该文件非常直观易懂。在第二行中，我们将模块添加到 HTTP 模块的列表中。根据你编写的模块类型，你需要将其添加到不同的列表中。你可以在 `nginx/auto/` 中找到的 `modules` 脚本中查看完整列表。

在编译之前，需要使用 `configure` 脚本显式指定模块，像以下代码所示。`add-module` 列表应该包含你希望在编译中包含的所有第三方模块。

```
./configure --add-module=path/to/your/new/module/directory
```

这之后需要像往常一样执行 `make` 和 `make install`。

### "Hello world" 源代码

以下代码来自 `ngx_http_hello_module.c`：

```
#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_http.h>

static char *ngx_http_hello(ngx_conf_t *cf, void *post, void*data);

static ngx_conf_post_handler_pt ngx_http_hello_p = ngx_http_hello;

/*
 * The structure will hold the value of the
 * module directive hello
 */
typedef struct {
  ngx_str_t   name;
} ngx_http_hello_loc_conf_t;

/* The function which initializes memory for the module configuration structure       
 */
static void *
ngx_http_hello_create_loc_conf(ngx_conf_t *cf)
{
  ngx_http_hello_loc_conf_t  *conf;

  conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_hello_loc_conf_t));
  if (conf == NULL) {
    return NULL;
  }

  return conf;
}
/*
 * The command array or array, which holds one subarray for each module
 * directive along with a function which validates the value of the
 * directive and also initializes the main handler of this module
 */
static ngx_command_t ngx_http_hello_commands[] = {
  { ngx_string("hello"),NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,ngx_conf_set_str_slot,NGX_HTTP_LOC_CONF_OFFSET,offsetof(ngx_http_hello_loc_conf_t, name),&ngx_http_hello_p },

  ngx_null_command
};

static ngx_str_t hello_string;

/*
 * The module context has hooks , here we have a hook for creating
 * location configuration
 */
static ngx_http_module_t ngx_http_hello_module_ctx = {
  NULL,                          /* preconfiguration */
  NULL,                          /* postconfiguration */

  NULL,                          /* create main configuration */
  NULL,                          /* init main configuration */

  NULL,                          /* create server configuration */
  NULL,                          /* merge server configuration */

  ngx_http_hello_create_loc_conf, /* create location configuration */
  NULL                           /* merge location configuration */
};

/*
 * The module which binds the context and commands
 *
 */
ngx_module_t ngx_http_hello_module = {NGX_MODULE_V1,
  &ngx_http_hello_module_ctx,    /* module context */ngx_http_hello_commands,       /* module directives */NGX_HTTP_MODULE,               /* module type */NULL,                          /* init master */NULL,                          /* init module */NULL,                          /* init process */NULL,                          /* init thread */NULL,                          /* exit thread */NULL,                          /* exit process */NULL,                          /* exit master */NGX_MODULE_V1_PADDING
};

/*
 * Main handler function of the module.
 */
static ngx_int_t
ngx_http_hello_handler(ngx_http_request_t *r)
{
  ngx_int_t    rc;
  ngx_buf_t   *b;
  ngx_chain_t  out;

  /* we response to 'GET' and 'HEAD' requests only */
  if (!(r->method & (NGX_HTTP_GET|NGX_HTTP_HEAD))) {
    return NGX_HTTP_NOT_ALLOWED;
  }

  /* discard request body, since we don't need it here */
  rc = ngx_http_discard_request_body(r);

  if (rc != NGX_OK) {
    return rc;
  }

  /* set the 'Content-type' header */
  r->headers_out.content_type_len = sizeof("text/html") - 1;
  r->headers_out.content_type.len = sizeof("text/html") - 1;
  r->headers_out.content_type.data = (u_char *) "text/html";

  /* send the header only, if the request type is http 'HEAD' */
  if (r->method == NGX_HTTP_HEAD) {
    r->headers_out.status = NGX_HTTP_OK;
    r->headers_out.content_length_n = hello_string.len;
    return ngx_http_send_header(r);
  }

  /* allocate a buffer for your response body */
  b = ngx_pcalloc(r->pool, sizeof(ngx_buf_t));
  if (b == NULL) {
    return NGX_HTTP_INTERNAL_SERVER_ERROR;
  }

  /* attach this buffer to the buffer chain */
  out.buf = b;
  out.next = NULL;

  /* adjust the pointers of the buffer */
  b->pos = hello_string.data;
  b->last = hello_string.data + hello_string.len;
  b->memory = 1;    /* this buffer is in memory */
  b->last_buf = 1;  /* this is the last buffer in the buffer chain*/

  /* set the status line */
  r->headers_out.status = NGX_HTTP_OK;
  r->headers_out.content_length_n = hello_string.len;

  /* send the headers of your response */
  rc = ngx_http_send_header(r);

  if (rc == NGX_ERROR || rc > NGX_OK || r->header_only) {
    return rc;
  }

  /* send the buffer chain of your response */
  return ngx_http_output_filter(r, &out);
}

/*
 * Function for the directive hello , it validates its value
 * and copies it to a static variable to be printed later
 */
static char *
ngx_http_hello(ngx_conf_t *cf, void *post, void *data)
{
  ngx_http_core_loc_conf_t *clcf;
  clcf = ngx_http_conf_get_module_loc_conf(cf,ngx_http_core_module);
  clcf->handler = ngx_http_hello_handler;

  ngx_str_t  *name = data; // i.e., first field ofngx_http_hello_loc_conf_t

  if (ngx_strcmp(name->data, "") == 0) {
    return NGX_CONF_ERROR;
  }
  hello_string.data = name->data;
  hello_string.len = ngx_strlen(hello_string.data);

  return NGX_CONF_OK;
}
```

这个扩展版 `hello world` 模块的示例配置可能如下所示：

```
server {
listen 8080;
server_name localhost;

location / {
hello 'Hello World';
  }
}
```

## Nginx 模块的组件

根据模块类型的不同，Nginx 模块有许多组件。我们现在将讨论那些几乎所有模块都通用的部分。目的是以一种易于理解的方式为您提供参考，使您能够准备好编写自己的模块。

### 模块配置结构

模块可以为每个配置文件的配置上下文定义一个配置——主上下文、服务器上下文和位置上下文都有各自的独立结构。大多数模块只需要一个位置结构是可以的。这些结构应该命名为`convention ngx_http_<module name>_(main|srv|loc)_conf_t`。以下是来自示例模块的代码片段：

```
typedef struct {
  ngx_str_t   name;
} ngx_http_hello_loc_conf_t;
```

这个结构的成员应该使用 Nginx 特有的数据类型（`ngx_uint_t`，`ngx_flag_t`，和 `ngx_str_t`），这些数据类型只是基本类型的别名。您可以在源代码树中的`core/nginx_config.h`文件里查看这些数据类型的定义。

这个结构的成员应该与模块指令的数量相匹配。在前面的示例中，我们的模块只有一个指令，所以我们可以预见该模块将在位置级别支持一个指令/选项，并将填充该结构的`name`成员。

如今这已经显而易见，配置结构中的元素是由配置文件中定义的模块指令填充的。

### 模块指令

在定义了模块指令值存储位置之后，就可以定义模块指令的名称及其接受的参数类型。模块的指令是通过`ngx_command_t type`结构的静态数组来定义的。回顾我们之前写的示例代码，以下是指令结构的样子：

```
static ngx_command_t ngx_http_hello_commands[] = {{ ngx_string("hello"),NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,ngx_conf_set_str_slot,NGX_HTTP_LOC_CONF_OFFSET,offsetof(ngx_http_hello_loc_conf_t, name),&ngx_http_hello_p },

  ngx_null_command
};
```

前面的结构可能看起来有些复杂。不过，我们现在将逐一查看这些元素，以便更好地理解它们。

第一个参数定义了指令的名称。这个参数是`ngx_str`类型，并且实例化为指令名称，例如`ngx_str("hello")`。`ngx_str_t`数据类型是一个结构体类型，包含数据和长度元素。Nginx 使用这个数据结构来处理所有字符串。

第二个参数定义了指令的类型以及它接受什么样的参数。这些参数的接受值应按位排列。可能的值如下：

```
NGX_HTTP_MAIN_CONF: directive should be used in main section
NGX_HTTP_SRV_CONF : directive should be used in the server section
NGX_HTTP_LOC_CONF : directive should be used in the locationsection
NGX_HTTP_UPS_CONF : directive should be used in the upstreamsection
NGX_CONF_NOARGS   : directive will take no arguments
NGX_CONF_TAKE1    : directive will take 1 argument
NGX_CONF_TAKE2    : directive will take 2 arguments
…
NGX_CONF_TAKE7    : directive will take 7 arguments
NGX_CONF_TAKE12   : directive will take 1 or 2 arguments
NGX_CONF_TAKE13   : directive will take 1 or 3 arguments   
NGX_CONF_TAKE23   : directive will take 2 or 3 arguments
NGX_CONF_TAKE123  : directive will take 1, 2 or 3 arguments
NGX_CONF_TAKE1234 : directive will take 1, 2 , 3 or 4 arguments

NGX_CONF_FLAG     : directive accepts a boolean value from "on" or"off"
NGX_CONF_1MORE    : directive requires at least one argument
NGX_CONF_2MORE    : directive requires at least at least twoarguments
```

请查看`core`文件夹中的`ngx_conf_file.h`，了解更多细节。

指令可以接受的最大参数数为八个（0-7），如`core/ngx_conf_file.h`中所定义，代码如下所示：

```
#define NGX_CONF_MAX_ARGS   8
```

在前面的示例中，我们只使用了数组中的一个元素，因为我们为单个`ngx_command_t`结构提供了值。

第三个参数是一个函数指针。这是一个设置函数，它获取配置文件中提供的指令值，并将其存储在结构的相应元素中。这个函数可以接受以下三个参数：

+   指向 `ngx_conf_t`（`main`、`srv` 或 `loc`）结构的指针，其中包含配置文件中的指令值

+   指向目标 `ngx_command_t` 结构的指针，值将被存储在此处

+   指向模块自定义配置结构的指针（可以是 `NULL`）

Nginx 提供了许多函数，可以用来设置内置数据类型的值。这些函数包括：

+   `ngx_conf_set_flag_slot`

+   `ngx_conf_set_str_slot`

+   `ngx_conf_set_str_array_slot`

+   `ngx_conf_set_keyval_slot`

+   `ngx_conf_set_num_slot`

+   `ngx_conf_set_size_slot`

+   `ngx_conf_set_off_slot`

+   `ngx_conf_set_msec_slot`

+   `ngx_conf_set_sec_slot`

+   `ngx_conf_set_bufs_slot`

+   `ngx_conf_set_enum_slot`

+   `ngx_conf_set_bitmask_slot`

其中一些如下所述：

+   `ngx_conf_set_flag_slot`：此函数将 on 或 off 转换为 1 或 0

+   `ngx_conf_set_str_slot`：将一个字符串保存为 `ngx_str_t`

+   `ngx_conf_set_num_slot`：此函数解析一个数字并将其保存到整数中

+   `ngx_conf_set_size_slot`：此函数解析数据大小（5k、2m 等）并将其保存到 `size_t`

如果内置函数无法满足模块作者的需求，例如，如果字符串需要以某种特定方式解释，而不仅仅是按原样存储，模块作者也可以在此传递指向自定义函数的指针。

为了指定这些内置（或自定义）函数将在哪里存储指令值，你必须将 `conf` 和 `offset` 作为下两个参数进行指定。`conf` 指定存储值的结构类型（`main`、`srv`、`loc`），而 `offset` 指定存储在此配置结构的哪一部分。以下是该元素在结构中的 `offset`，即 `offsetof(ngx_http_hello_loc_conf_t, name)`。

最后一个元素通常是 `NULL`，目前我们可以选择忽略它。

命令数组的最后一个元素是 `ngx_null_command`，它表示终止。

### 模块上下文

Nginx 模块中需要定义的第三个结构是一个静态的 `ngx_http_module_t` 结构，它包含用于创建 `main`、`srv` 和 `loc` 配置的函数指针，并将它们合并在一起。它的名字是 `ngx_http_<module name>_module_ctx`。你可以提供的函数引用如下：

+   配置前

+   配置后

+   创建主 `conf`

+   初始化主 `conf`

+   创建服务器 `conf`

+   与主 `conf` 合并

+   创建位置 `conf`

+   与服务器 `conf` 合并

这些函数根据其功能接受不同的参数。以下是结构定义，摘自 `http/ngx_http_config.h`，你可以看到回调函数的不同签名：

```
typedef struct {
  ngx_int_t   (*preconfiguration)(ngx_conf_t *cf);
  ngx_int_t   (*postconfiguration)(ngx_conf_t *cf);

  void       *(*create_main_conf)(ngx_conf_t *cf);
  char       *(*init_main_conf)(ngx_conf_t *cf, void *conf);

  void       *(*create_srv_conf)(ngx_conf_t *cf);
  char       *(*merge_srv_conf)(ngx_conf_t *cf, void *prev, void*conf);

  void       *(*create_loc_conf)(ngx_conf_t *cf);
  char       *(*merge_loc_conf)(ngx_conf_t *cf, void *prev, void*conf);
} ngx_http_module_t;
```

你可以将不需要的函数设置为`NULL`，Nginx 会接受它并做出正确的处理。

创建函数，如创建主`conf`、创建服务器`conf`和创建位置`conf`，通常只是为结构体分配内存（例如`malloc()`）并将元素初始化为默认值。像初始化主`conf`、与主`conf`合并等函数则提供了覆盖默认值的机会。

在合并过程中，模块作者可以查找元素的重复定义，如果配置文件中配置作者提供的指令有问题，则抛出错误。

大多数模块作者只是使用最后两个元素：一个函数用于为`ngx_loc_conf`（`main`、`srv`或`loc`）配置分配内存，另一个函数用于设置默认值并将此配置合并到合并后的位置配置中（称为`ngx_http_<module name >_merge_loc_conf`）。

以下是一个示例模块上下文结构：

```
/*
 * The module context has hooks , here we have a hook for creating
 * location configuration
 */
static ngx_http_module_t ngx_http_hello_module_ctx = {NULL,                          /* preconfiguration */NULL,                          /* postconfiguration */NULL,                          /* create main configuration */NULL,                          /* init main configuration */NULL,                          /* create server configuration */NULL,                          /* merge server configuration */ngx_http_hello_create_loc_conf, /* create location configuration*/NULL                           /* merge location configuration*/
};
```

现在我们可以更仔细地看这些函数，它们根据配置设置位置。

#### create_loc_conf

以下是一个基本的`create_loc_conf`函数的样子。它接受一个指令结构体（`ngx_conf_t`）并返回一个新分配的模块配置结构体。

```
/* The function which initializes memory for the module configuration structure       
 */
static void *
ngx_http_hello_create_loc_conf(ngx_conf_t *cf)
{
  ngx_http_hello_loc_conf_t  *conf;

  conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_hello_loc_conf_t));
  if (conf == NULL) {
    return NULL;
  }

  return conf;
}
```

如果你使用内置的`ngx_palloc`（一个 malloc 包装器）或`ngx_pcalloc`（一个 calloc 包装器），Nginx 内存分配会负责释放内存。

#### merge_loc_conf

我们创建的示例模块不包含合并位置`conf`函数。然而，我们可以看看以下示例代码，只是为了说明一些基本概念。如果一个指令可以多次定义，你通常需要一个合并函数。你的任务是定义一个合并函数，能够在该指令多次定义或在多个位置定义的情况下设置适当的值。

```
static char *ngx_http_example_merge_loc_conf(ngx_conf_t *cf, void *parent,void *child)
{
  ngx_http_example_loc_conf_t *prev = parent;
  ngx_http_example_loc_conf_t *conf = child;

  ngx_conf_merge_uint_value(conf->val1, prev->val1, 10);
  ngx_conf_merge_uint_value(conf->val2, prev->val2, 20);

  if (conf->val1 < 1) {
    ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,"value 1 must be equal or more than 1");
    return NGX_CONF_ERROR;
  }
  if (conf->val2 < conf->val1) {
    ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,"val2 must be equal or more than val1");
    return NGX_CONF_ERROR;
  }

  return NGX_CONF_OK;
}
```

Nginx 为各种数据类型提供了非常有用的合并内置函数（`ngx_conf_merge_<data type>_value`）。这些函数接受以下参数：

+   位置的值（可以引用结构体中的元素）

+   如果未设置第一个值，则使用此值

+   如果未设置第一个和第二个值，则使用默认值

第一个参数存储结果。有关可用的合并函数的完整列表，请参阅`core/ngx_conf_file.h`。以下是文件中的摘录：

```
#define ngx_conf_merge_value(conf, prev, default) \
  if (conf == NGX_CONF_UNSET) { \
  conf = (prev == NGX_CONF_UNSET) ? default : prev; \
  }

#define ngx_conf_merge_ptr_value(conf, prev, default) \
  if (conf == NGX_CONF_UNSET_PTR) { \
    conf = (prev == NGX_CONF_UNSET_PTR) ? default : prev; \
  }

#define ngx_conf_merge_uint_value(conf, prev, default) \
  if (conf == NGX_CONF_UNSET_UINT) { \
    conf = (prev == NGX_CONF_UNSET_UINT) ? default : prev; \
  }

#define ngx_conf_merge_msec_value(conf, prev, default) \
  if (conf == NGX_CONF_UNSET_MSEC) { \
    conf = (prev == NGX_CONF_UNSET_MSEC) ? default : prev; \
  }

#define ngx_conf_merge_sec_value(conf, prev, default) \
  if (conf == NGX_CONF_UNSET) { \
    conf = (prev == NGX_CONF_UNSET) ? default : prev; \
  }

#define ngx_conf_merge_size_value(conf, prev, default) \
  if (conf == NGX_CONF_UNSET_SIZE) { \
    conf = (prev == NGX_CONF_UNSET_SIZE) ? default : prev; \
  }

#define ngx_conf_merge_off_value(conf, prev, default) \
  if (conf == NGX_CONF_UNSET) { \
    conf = (prev == NGX_CONF_UNSET) ? default : prev; \
  }

#define ngx_conf_merge_str_value(conf, prev, default) \
  if (conf.data == NULL) { \
  if (prev.data) { \
    conf.len = prev.len; \
     conf.data = prev.data; \
    } else { \
    conf.len = sizeof(default) - 1; \
    conf.data = (u_char *) default; \
    } \
  }

#define ngx_conf_merge_bufs_value(conf, prev, default_num,default_size)     \
  if (conf.num == 0) { \
    if (prev.num) { \
      conf.num = prev.num; \
      conf.size = prev.size; \
      } else { \
      conf.num = default_num; \
      conf.size = default_size; \
      } \
  }
```

如你所见，这些函数被定义为宏，并且在编译时展开并内联到代码中。

另一个要学习的内容是如何记录错误。该函数使用`ngx_conf_log_error`函数将信息输出到日志文件——你可以指定日志级别——并返回`NGX_CONF_ERROR`，这将停止服务器启动。

在 Nginx 中定义了几个日志级别，这些级别在`ngx_log.h`中定义。以下是代码中的摘录：

```
#define NGX_LOG_STDERR            0
#define NGX_LOG_EMERG             1
#define NGX_LOG_ALERT             2
#define NGX_LOG_CRIT              3
#define NGX_LOG_ERR               4
#define NGX_LOG_WARN              5
#define NGX_LOG_NOTICE            6
#define NGX_LOG_INFO              7
#define NGX_LOG_DEBUG             8
```

### 模块定义

下一个结构是新模块应该定义的模块定义结构或`ngx_module_t`结构。变量称为`ngx_http_<module name>_module`。这个结构将前面定义的上下文和指令结构以及剩余的回调（退出线程、退出进程等）绑定在一起。模块定义可以像一个键一样查找与特定模块关联的数据。我们自定义模块的模块定义如下：

```
#define NGX_MODULE_V1          0, 0, 0, 0, 0, 0, 1
#define NGX_MODULE_V1_PADDING  0, 0, 0, 0, 0, 0, 0, 0

struct ngx_module_s {
  ngx_uint_t            ctx_index;
  ngx_uint_t            index;

  ngx_uint_t            spare0;
  ngx_uint_t            spare1;
  ngx_uint_t            spare2;
  ngx_uint_t            spare3;
  ngx_uint_t            version;

  void                 *ctx;
  ngx_command_t        *commands;
  ngx_uint_t            type;

  ngx_int_t           (*init_master)(ngx_log_t *log);

  ngx_int_t           (*init_module)(ngx_cycle_t *cycle);

  ngx_int_t           (*init_process)(ngx_cycle_t *cycle);
  ngx_int_t           (*init_thread)(ngx_cycle_t *cycle);
  void                (*exit_thread)(ngx_cycle_t *cycle);
  void                (*exit_process)(ngx_cycle_t *cycle);

  void                (*exit_master)(ngx_cycle_t *cycle);

  uintptr_t             spare_hook0;
  uintptr_t             spare_hook1;
  uintptr_t             spare_hook2;
  uintptr_t             spare_hook3;
  uintptr_t             spare_hook4;
  uintptr_t             spare_hook5;
  uintptr_t             spare_hook6;
  uintptr_t             spare_hook7;
};
```

你可以看到宏`NGX_MODULE_V1`和`NGX_MODULE_V1_PADDING`在上述代码突出显示的部分之前和之后为结构元素提供了值。这是一个我们目前不需要深入了解的细节。现在，看看以下示例如何使用它们：

```
/*
 * The module which binds the context and commands
 *
 */
ngx_module_t ngx_http_hello_module = {NGX_MODULE_V1,&ngx_http_hello_module_ctx,    /* module context */ngx_http_hello_commands,       /* module directives */NGX_HTTP_MODULE,               /* module type */NULL,                          /* init master */NULL,                          /* init module */NULL,                          /* init process */NULL,                          /* init thread */NULL,                          /* exit thread */
  NULL,                          /* exit process */NULL,                          /* exit master */NGX_MODULE_V1_PADDING
};
```

你可以从前面代码的注释中看到每个参数的含义。第一个和最后一个元素是隐藏额外结构元素的掩码，主要是因为我们不需要它们，并且它们是未来的占位符。我们还提供了一个模块类型，本例中是 HTTP。大多数用户定义的自定义模块将属于这种类型。你可以定义其他类型，如 CORE、MAIL、EVENT 等；然而，它们大多数不用作附加模块类型。

### 处理函数

在所有的准备工作和配置结构之后，最后一块拼图是实际的处理函数，它完成所有的工作。我们示例模块的处理函数如下：

```
/*
 * Main handler function of the module.
 */
static ngx_int_tngx_http_hello_handler(ngx_http_request_t *r)
{
  ngx_int_t    rc;
  ngx_buf_t   *b;
  ngx_chain_t  out;

  /* we response to 'GET' and 'HEAD' requests only */
  if (!(r->method & (NGX_HTTP_GET|NGX_HTTP_HEAD))) {
    return NGX_HTTP_NOT_ALLOWED;
  }

  /* discard request body, since we don't need it here */
  rc = ngx_http_discard_request_body(r);

  if (rc != NGX_OK) {
    return rc;
  }

  /* set the 'Content-type' header */
  r->headers_out.content_type_len = sizeof("text/html") - 1;
  r->headers_out.content_type.data = (u_char *) "text/html";
  /* send the header only, if the request type is http 'HEAD' */
  if (r->method == NGX_HTTP_HEAD) {
    r->headers_out.status = NGX_HTTP_OK;
    r->headers_out.content_length_n = hello_string.len;

  return ngx_http_send_header(r);
  }

  /* allocate a buffer for your response body */
  b = ngx_pcalloc(r->pool, sizeof(ngx_buf_t));
  if (b == NULL) {
    return NGX_HTTP_INTERNAL_SERVER_ERROR;
  }

  /* attach this buffer to the buffer chain */
  out.buf = b;
  out.next = NULL;

  /* adjust the pointers of the buffer */
  b->pos = hello_string.data;
  b->last = hello_string.data + hello_string.len;
  b->memory = 1;    /* this buffer is in memory */
  b->last_buf = 1;  /* this is the last buffer in the buffer chain*/

  /* set the status line */
  r->headers_out.status = NGX_HTTP_OK;
  r->headers_out.content_length_n = hello_string.len;

  /* send the headers of your response */
  rc = ngx_http_send_header(r);

  if (rc == NGX_ERROR || rc > NGX_OK || r->header_only) {
    return rc;
  }

  /* send the buffer chain of your response */
  return ngx_http_output_filter(r, &out);
}
```

在代码中有几件事情要学习。正如前面解释的那样，这个模块基本上会打印出你在配置中提供的任何内容。例如，根据以下配置，当你打开`http://localhost:8080`时，这个模块将确保打印**Hello World**：

```
server {
listen 8080;
server_name localhost;

location / {
hello 'Hello World';
  }
}
```

这个方法接收 HTTP 请求作为参数。如果你的模块只响应特定类型的 HTTP 请求，你可以通过查看 HTTP 请求结构来检查。例如，我们的模块只响应 HTTP `GET` 和 `HEAD` 请求，如这段代码所检查的；否则返回"错误代码 405（不允许）"。

所有的 HTTP 错误代码都在`ngx_http_request.h`中定义如下：

```
  /* we response to 'GET' and 'HEAD' requests only */
  if (!(r->method & (NGX_HTTP_GET|NGX_HTTP_HEAD))) {
    return NGX_HTTP_NOT_ALLOWED;
  }
```

然后，我们丢弃请求体，因为在这个模块中我们不需要它。在一些模块中，将写一个对请求体很重要的正文；然而，现在我们不关心它。通过丢弃请求体，Nginx 将不会完全读取请求体以进行处理，并且不会在内部为其分配内存。

接下来，在响应中设置一些 HTTP 头部。你可以通过 HTTP 请求结构的`headers_out`成员访问所有可以设置在响应中的头部。`headers_out`结构允许你设置多个输出头部。从`ngx_http_request.h`中提取的部分如下：

```
typedef struct {
  ngx_list_t                        headers;

  ngx_uint_t                        status;
  ngx_str_t                         status_line;

  ngx_table_elt_t                  *server;
  ngx_table_elt_t                  *date;
  ngx_table_elt_t                  *content_length;
  ngx_table_elt_t                  *content_encoding;
  ngx_table_elt_t                  *location;
  ngx_table_elt_t                  *refresh;
  ngx_table_elt_t                  *last_modified;
  ngx_table_elt_t                  *content_range;   ngx_table_elt_t                  *accept_ranges;
  ngx_table_elt_t                  *www_authenticate;
  ngx_table_elt_t                  *expires;
  ngx_table_elt_t                  *etag;

  ngx_str_t                        *override_charset;

  size_t                            content_type_len;
  ngx_str_t                         content_type;
  ngx_str_t                         charset;
  u_char                           *content_type_lowcase;
  ngx_uint_t                        content_type_hash;

  ngx_array_t                       cache_control;

  off_t                             content_length_n;
  time_t                            date_time;
  time_t                            last_modified_time;
} ngx_http_headers_out_t;
```

我们模块中的下一个重要步骤是为响应缓冲区分配内存。应该使用 Nginx 自有的 API 分配这块内存，如前面章节所述（因为它也会自动处理释放内存）。之所以可以这样做，是因为内存是从本地内存池中分配的，所以所有内存分配都受到追踪。

响应是在一个链接列表或*链*的缓冲区中创建的，每个缓冲区的大小为`ngx_buf_s`。这使得 Nginx 可以并行处理响应。如果有其他处理程序或过滤器需要对响应进行后处理，它们可以在链中的第一个缓冲区准备好时开始工作，而你则在填充第二个缓冲区。这使得 Nginx 可以继续并行操作，而无需等待任何模块完全处理完毕。

当你在最后一个缓冲区中创建完响应后，应设置`b->last_buf = 1`。正如其名称所示，这将告诉 Nginx，这是来自你的模块的最后一个响应缓冲区。

如果响应处理成功，你会希望将响应头的状态设置为`HTTP_OK`。可以通过`r->headers_out.status = NGX_HTTP_OK`来完成。

然后，你需要通过调用`ngx_http_send_header`来初始化头部过滤器链。这将告诉 Nginx，输出头部的处理已完成，现在 Nginx 可以将它们传递给一个过滤器链，这些过滤器可能希望对头部进行进一步的后处理。

最后一步是通过调用`ngx_http_output_filter`从函数中返回。这将启动 HTTP 主体过滤链的处理过程。也就是说，Nginx 或可能已经安装的自定义过滤模块将对你刚刚在缓冲区中创建的 HTTP 响应主体进行后处理。

创建 Nginx 自定义模块的总结如下：

1.  创建一个模块配置，它可以根据位置、主配置或服务器结构化；每种结构都有特定的命名约定（见`ngx_http_hello_loc_conf_t`）。

    模块的允许指令在`typengx_command_t`的静态数组中（见`ngx_http_hello_commands`）。这将包含指向函数的指针，函数中包含了验证每个指令值以及初始化处理程序的代码。

1.  创建一个模块上下文结构体，如`ngx_http_<module name>_module_ctx`，它是`ngx_http_module_t`类型，包含许多用于设置配置的钩子。你可以在这里设置后配置钩子，例如，设置模块的主处理程序（见`ngx_http_hello_module_ctx`）。

1.  然后，我们定义模块，它也是一个`ngx_module_t`类型的结构体，包含对你在前面步骤中创建的模块上下文和模块命令的引用（见`ngx_http_hello_module`）。

1.  创建处理 HTTP 请求的主模块处理函数。该函数还会以一系列固定大小的缓冲区输出响应头和主体。

# Nginx 开发工具包 (NDK)

NDK 是一个 Nginx 模块，它使得模块开发者更容易开发 Nginx 模块。正如你在本章中所看到的，开发模块时会有一些重复的通用任务。NDK 提供了一些内置的宏和函数，可以减少你在开发模块时需要编写的代码量。

为了使用 NDK，你需要像其他模块一样将其作为一个模块进行添加。如果你希望使用该模块提供的宏和函数，你还需要在你的模块源代码中包含 `ndk.h` 文件。

NDK 提供了一些有用的工具，如用于处理路径和正则表达式等复杂类型的 `conf` 设置函数，NULL 检查、返回值处理以及将数据设置为零的工具方法。

NDK 还包含了一个 **Auto Lib Core**，它允许开发者和用户以一致的、跨平台的方式将外部库包含到 Nginx 中。

你可以在 [`github.com/simpl/ngx_devel_kit`](https://github.com/simpl/ngx_devel_kit) 上查看更多详细信息和文档。

# 总结

在本章中，我们学习了创建一个简单的 Nginx 处理器模块的过程。我们还了解了一个新模块应该定义的基本结构，以及如何将它们相互关联。最后，我们看了一个执行基本任务的小型处理器函数，它为你编写更复杂的模块提供了基础。

如果你是一个 Nginx 模块开发者，你必须广泛浏览其他模块和 Nginx 源代码，这将帮助你学习如何在代码中完成不同的任务以及通常使用哪个 API。

你还可以在 [`github.com/simpl/ngx_devel_kit`](https://github.com/simpl/ngx_devel_kit) 上找到 Nginx 开发工具包。这将为你提供额外的 `conf_set` 函数，用于处理正则表达式、复杂/脚本值、路径和宏，从而简化任务，比如在进行 `ngx_array_push` 时检查 NULL 值等等，这将使你在编写自定义 Nginx 模块时更加轻松。
