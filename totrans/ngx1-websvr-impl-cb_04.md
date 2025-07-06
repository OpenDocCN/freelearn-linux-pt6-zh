# 第四章：减缓他们的速度：访问和速率限制模块

在本章中，我们将涵盖：

+   限制每个会话的请求

+   使用 IP 阻止和允许访问

+   为下载目录设置简单的速率限制

+   限制搜索引擎爬虫的请求

+   使用 MaxMind 国家数据库设置 GeoIP

+   使用 GeoIP 模块设置访问和速率控制

# 介绍

在这个互联网时代，用户对于他们从在线服务中获得的服务质量非常敏感。许多资源有限的小公司通过快速创新能够抓住市场的一部分。然而，这些公司最终不得不进行速率限制，因为他们的流量不可避免地超过了服务器的承载能力。

一些像“被 digged”([`www.digg.com`](http://www.digg.com)) 或“被 slashdotted”([`www.slashdot.org`](http://www.slashdot.org)) 这样简单的事情曾经导致网站崩溃，但 Nginx 通过提供基于 IP 的速率限制和服务器访问，能有效防止这种情况。

# 限制每个会话的请求

由于其事件驱动的特性，Nginx 在全球范围内被广泛采用，尤其是在需要高性能和资源受限的场合。然而，在许多情况下，这还不足够，唯一的办法就是限制请求，以确保你的网站正常运行，服务器不发生宕机。

## 如何实现...

以下配置，当应用于 `server` 指令时，可以限制特定会话的请求：

```
http {
limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;
...
Server {
limit_req zone=one burst=5;
...
}

```

## 它是如何工作的...

`limit_req_zone` 指令基本上允许你定义哪个变量（在这种情况下是`$binary_remote_addr`）作为会话的键，同时为这个“区域”分配 10MB 内存，并将速率限制为每秒一个请求。设置区域的数量没有限制，只要你有足够的内存来处理区域分配。一个区域，假设使用远程地址作为会话的键，可以在分配给它的 1M 会话内存中处理大约 32,000 个会话。

在 `server` 指令中，我们实际上是通过使用 `limit_req` 指令来限制请求速率，该指令基本上使用了区域一，它允许每秒平均最多一个请求，且最大突发速率为五个请求。

超过速率容量的任何请求将收到“服务不可用”503 错误页面。

## 还有更多内容...

你可以使用其他变量作为会话键，但需要注意的是，会话键的变量大小必须较小，以便容纳所有传入的连接（即总连接数 x 会话值大小 < 会话缓存的大小）。

# 使用 IP 阻止和允许访问

网站需要做的一件最重要的事情是黑名单一些恶意的 IP，这些 IP 随着时间的推移会试图探测并对你的网站造成伤害。这可以在多个层面上进行，比如路由器，甚至是在软件防火墙层面，这样也能减少 Nginx 的不必要负载。如果你对自己的技术栈控制不够强，Nginx 是开始屏蔽这些机器人和黑客的最佳地方。

## 如何操作...

这允许你阻止一些 IP 访问你的网站：

```
server {
listen 80;
server_name www.example1.com;
location / {
deny 192.168.1.1;
allow 192.168.1.0/24;
deny all;
}
...
}

```

## 它是如何工作的...

很明显，`deny` 和 `allow` 指令是按顺序执行的，所以它会拒绝 IP 192.168.1.1，同时允许网络 192.168.1.0/24 访问。最后的 `deny all` 指令确保没有其他 IP 可以访问该位置（`http://www.example1.com`）。

所以所有其他 IP 在尝试访问此 HTTP 位置时，将会收到一个 403 禁止访问页面。你可以使用 `error_page` 指令将其重写为 404 页面。

## 还有更多...

重要的是要意识到指令的顺序至关重要。像这样的指令：

```
server {
listen 80;
server_name www.example1.com;
location / {
deny all; # this is not a good idea
deny 192.168.1.1;
allow 192.168.1.0/24;
}
...
}

```

将向所有访问该位置（`http://www.example1.com`）的客户端返回 403 禁止访问页面。

# 为下载目录设置简单的速率限制

我们已经了解了如何限制请求的速率，但有时问题在于某些客户端开始占用带宽，降低其他用户的服务质量。在这种情况下，最好使用基于带宽的速率限制。

这种方法的最佳应用场景是在你网站的静态文件中。它确保没有人因错误原因窃取你的带宽。

## 如何操作...

以下简单的配置将在 server 指令中允许你对整个网站进行速率限制：

```
server {
server_name www.example1.com;
location /downloads/ {
limit_rate 10k;
root /var/www/www.example1.com/downloads/;
}
..
}

```

## 它是如何工作的...

这个简单的配置将把所有用户对 `/downloads` 文件的下载速度限制为 10k。

## 还有更多...

使用这个速率限制，还可以做更多事情；以下配置将在文件已完全以全速传输给客户端 1 兆字节后，将速率限制为 100k：

```
server {
server_name www.example1.com;
location /downloads/ {
limit_rate_after 1m;
limit_rate 100k;
root /var/www/www.example1.com/downloads/;
}
…
}

```

# 速率限制搜索引擎机器人

到目前为止，我们已经学习了阻止、请求限制和带宽限制所有客户端的简便方法。我们可以开始将这些知识应用于生产环境中实际遇到的一些问题。

大多数情况下，对于内容繁重的网站，机器人和搜索引擎开始使用的带宽超过了实际用户。在这种情况下，如果你希望确保实际用户不受影响，同时又能保持 SEO 效果，你将需要对搜索引擎机器人进行速率限制。

## 如何操作...

以下配置，当放置在位置指令中时，将帮助你屏蔽并限制一些机器人的速率：

```
if($http_user_agent ~ "Alexibot|Art-Online|asterias|BackDoorbot|Black.Hole|\
BlackWidow|BlowFish|botALot|BuiltbotTough|Bullseye|BunnySlippers|Cegbfeieh|Cheesebot") {
deny all;
}
if ($http_user_agent ~ Google|Yahoo|MSN|baidu) {
limit_rate 20k;
}

```

## 它是如何工作的...

这个思路相对简单，在确定某个机器人不会给你带来流量，而只是盗取带宽时，最好阻止它们。但并非所有机器人都是坏的。Googlebot、yahoobot 和 msnbot 对你的搜索引擎流量至关重要。在流量较高的站点中，需要在阻止不良机器人和允许有用机器人的平衡中找到细微的差别。

## 还有更多...

你还可以利用这种情况确保你的站点几乎没有垃圾流量。显然，大多数评论机器人可以通过简单地黑名单`HTTP_REFERER`来阻止，具体代码如下：

```
if ($http_referer ~* (\.us$|dating|diamond|forsale|girl|jewelry|organic|poker|poweroversoftware|teen|webcam|zippo) ) {
deny all;
}

```

# 使用 MaxMind 国家数据库设置 GeoIP

MaxMind 是一家专门生成数据库的公司，这些数据库将国家和城市与 IP 地址范围进行映射。它使你能够轻松地定位终端客户的地理位置。该信息可用于显示用户的地理相关数据，或许还可以将其重定向到更靠近的服务器位置，以便更快地为终端客户提供服务。

在本食谱中，我们将在 Nginx 中安装 MaxMind 数据库，并展示如何在 Nginx 配置中使用 GeoIP 变量。你可以在此链接查看他们的演示：[`www.maxmind.com/app/locate_my_ip`](http://www.maxmind.com/app/locate_my_ip)。

![使用 MaxMind 国家数据库设置 GeoIP](img/4965_04_01.jpg)

## 如何操作...

1.  下载 Geo-IP 数据库或安装相关软件包：

    ```
    wget http://geolite.maxmind.com/download/geoip/database/GeoLiteCity.dat.gz

    ```

    +   或者

    ```
    aptitude install geoip-database

    ```

1.  你需要安装一些 GeoIP 库：

    ```
    aptitude install libgeoip-dev

    ```

1.  然后，你也需要为 Nginx 配置安装 GeoIP 模块。这假设你已经下载了 Nginx 的代码库，并且已经安装了编译所需的依赖项。这部分内容已经在前面的食谱中讲解过。

    ```
    ./configure --with-http_geoip_module

    ```

1.  然后我们将在`http`指令下添加以下配置：

    ```
    http {
    geoip_country GeoIP.dat; # the country IP database
    geoip_city GeoLiteCity.dat; # the city IP database
    . . .

    ```

### 它的工作原理...

上述步骤安装了数据库并在 Nginx 中配置了 GeoIP 模块。这使得配置可以访问以下新的变量。这些变量可以帮助你编写特定于地理位置的规则！

| 变量 | 作用 |
| --- | --- |
| `$geoip_city_country_code` | 两字母国家代码，例如“RU”、“US”。 |
| `$geoip_city_country_code3` | 三字母国家代码，例如“RUS”、“USA”。 |
| `$geoip_city_country_name` | 国家名称，例如“俄罗斯联邦”、“美国”等——如果可用的话。 |
| `$geoip_region` | 地区名称（省份、区域、州、联邦土地等），例如“莫斯科市”、“华盛顿特区”等——如果可用的话。 |
| `$geoip_city` | 城市名称，例如“莫斯科”、“华盛顿”、“里斯本”等——如果可用的话。 |
| `$geoip_postal_code` | 邮政编码——如果可用的话。 |
| `$geoip_city_continent_code` | 大洲名称（如果可用的话） |
| `$geoip_latitude` | 纬度——如果可用的话。 |
| `$geoip_longitude` | 经度——如果可用的话。 |

# 使用 GeoIP 模块设置访问和速率控制

现在我们进入 Nginx 的一个有趣部分，在这里我们可以使用 GeoIP 模块来设置访问和速率控制。

例如，你可以根据需要让整个国家无法访问你的网站。Hulu 的视频（[`www.hulu.com`](http://www.hulu.com)）对于美国以外的 IP 是不可用的。当然，这并非完全无懈可击，因为有匿名网络可以让你隐藏真实的 IP，或者让你看起来像是来自美国的客户端。

## 如何实现...

这个简单的配置假设你已经按照前面的教程安装了 GeoIP，它将允许百慕大用户访问某些内容，同时阻止不丹和玻利维亚用户访问该网站：

```
http {
geoip_country GeoIP.dat; # the country IP database
geoip_city GeoLiteCity.dat; # the city IP database
...
server {
server_name www.example1.com;
...
location / {
If($geoip_city_country_code ~ BM) {
rate_limit 20k;
}
If($geoip_city_country_code ~ BT|BO) {
deny all;
}
...
}
...
}

```

## 它是如何工作的...

GeoIP 模块背后的原理很简单。它基本上查看远程客户端 IP，并填充一些变量，让你能够轻松识别客户端的各种地理属性。在这个示例中，我们正在过滤来自百慕大的请求，并将其带宽限制为 20k，同时对于来自不丹的请求，我们发送 403 禁止访问的响应。

你可以扩展这个功能，在同一个网址上为不同国家创建备用站点。这对于语言本地化也非常有用。Nginx 在 GeoIP 映射方面显然是最先进的。
