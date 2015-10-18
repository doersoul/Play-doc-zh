#设置前端HTTP服务器

你可以简单地通过设置应用程序HTTP端口到80，部署你的应用程序作为单独的服务器:

```shell
$ /path/to/bin/<project-name> -Dhttp.port=80
```

> 注意你可能需要root权限，以绑定进程到这个端口。

然而, 如果你计划在同一个服务器主机运行几个应用程序或为了伸缩性和容错性而提供应用程序的几个负载均衡实例, 你可以使用前端HTTP服务器。

注意使用前端HTTP服务器相对于直接使用Play服务器，不会给你多少性能的提升。但是, HTTP 服务器可以非常好的处理HTTPS, 条件 GET 请求和静态资产, 并且许多服务假定前端HTTP服务器是你的架构的一部分。


##用 lighttpd设置
This example shows you how to configure [lighttpd](http://www.lighttpd.net/) as a front end web server. Note that you can do the same with Apache, but if you only need virtual hosting or load balancing, lighttpd is a very good choice and much easier to configure!

The `/etc/lighttpd/lighttpd.conf` file should define things like this:

```scala
server.modules = (
      "mod_access",
      "mod_proxy",
      "mod_accesslog"
)
…
$HTTP["host"] =~ "www.myapp.com" {
    proxy.balance = "round-robin" proxy.server = ( "/" =>
        ( ( "host" => "127.0.0.1", "port" => 9000 ) ) )
}

$HTTP["host"] =~ "www.loadbalancedapp.com" {
    proxy.balance = "round-robin" proxy.server = ( "/" => (
          ( "host" => "127.0.0.1", "port" => 9001 ),
          ( "host" => "127.0.0.1", "port" => 9002 ) )
    )
}
```


##用 nginx设置
This example shows you how to configure [nginx](http://wiki.nginx.org/Main) as a front end web server. Note that you can do the same with Apache, but if you only need virtual hosting or load balancing, nginx is a very good choice and much easier to configure!

The `/etc/nginx/nginx.conf` file should define things like this:

```scala
worker_processes  1;

events {
    worker_connections  1024;
}

http {
  include       mime.types;
  default_type  application/octet-stream;

  sendfile        on;
  keepalive_timeout  65;

  proxy_buffering    off;
  proxy_set_header   X-Real-IP $remote_addr;
  proxy_set_header   X-Scheme $scheme;
  proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header   Host $http_host;
  proxy_http_version 1.1;

  upstream my-backend {
     server 127.0.0.1:9000;
  }

  server {
    listen       80;
    server_name www.mysite.com;
    location / {
       proxy_pass http://my-backend;
    }
  }

  #server {
  #  listen               443;
  #  ssl                  on;
  #
  #  # http://www.selfsignedcertificate.com/ is useful for development testing
  #  ssl_certificate      /etc/ssl/certs/my_ssl.crt;
  #  ssl_certificate_key  /etc/ssl/private/my_ssl.key;
  #
  #  # From https://bettercrypto.org/static/applied-crypto-hardening.pdf
  #  ssl_prefer_server_ciphers on;
  #  ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # not possible to do exclusive
  #  ssl_ciphers 'EDH+CAMELLIA:EDH+aRSA:EECDH+aRSA+AESGCM:EECDH+aRSA+SHA384:EECDH+aRSA+SHA256:EECDH:+CAMELLIA256:+AES256:+CAMELLIA128:+AES128:+SSLv3:!aNULL:!eNULL:!LOW:!3DES:!MD5:!EXP:!PSK:!DSS:!RC4:!SEED:!ECDSA:CAMELLIA256-SHA:AES256-SHA:CAMELLIA128-SHA:AES128-SHA';
  #  add_header Strict-Transport-Security max-age=15768000; # six months
  #  # use this only if all subdomains support HTTPS!
  #  # add_header Strict-Transport-Security "max-age=15768000; includeSubDomains"
  #
  #  keepalive_timeout    70;
  #  server_name www.mysite.com;
  #  location / {
  #    proxy_pass  http://my-backend;
  #  }
  #}
}
```

> Note Make sure you are using version 1.2 or greater of Nginx otherwise chunked responses won’t work properly.


##用 Apache设置
The example below shows a simple set up with [Apache httpd server](https://httpd.apache.org/) running in front of a standard Play configuration.

```xml
LoadModule proxy_module modules/mod_proxy.so
…
<VirtualHost *:80>
  ProxyPreserveHost On
  ServerName www.loadbalancedapp.com
  ProxyPass  /excluded !
  ProxyPass / http://127.0.0.1:9000/
  ProxyPassReverse / http://127.0.0.1:9000/
</VirtualHost>
```


##高级代理设置
当使用一个HTTP前端服务器, 请求地址将被视为来自HTTP服务器。在通常的设置, 当你的Play应用和代理服务器都同时运行在同一台机器上,  Play应用程序会看请求视为来自127.0.0.1。

代理服务器可以添加一个特殊标头到请求中，以告诉代理的应用程序请求来自哪里。多数web服务器会添加一个`X-Forwarded-For` 标头和远程客户端IP地址作为第一个参数。如果代理服务器是运行在本地并从127.0.0.1连接, Play会信任它的`X-Forwarded-For` 标头。

但是, 主机标头是未改变的, 它会继续代理发出。如果你使用Apache 2.x, 你可以添加这个指令:

```scala
ProxyPreserveHost on
```

主机: 标头会是由客户端发出原始主机的请求标头。通过结合这二种技术, 你的应用会直接暴露。

如果你不希望这个play应用程序占用整个根, 添加排除代理配置的指令:

```
ProxyPass /excluded !
```


##Apache作为前端代理以允许应用程序透明升级
The basic idea is to run two Play instances of your web application and let the front-end proxy load-balance them. In case one is not available, it will forward all the requests to the available one.

Let’s start the same Play application two times: one on port 9999 and one on port 9998.

```shell
$ start -Dhttp.port=9998
$ start -Dhttp.port=9999
```

Now, let’s configure our Apache web server to have a load balancer.

In Apache, I have the following configuration:

```xml
<VirtualHost mysuperwebapp.com:80>
  ServerName mysuperwebapp.com
  <Location /balancer-manager>
    SetHandler balancer-manager
    Order Deny,Allow
    Deny from all
    Allow from .mysuperwebapp.com
  </Location>
  <Proxy balancer://mycluster>
    BalancerMember http://localhost:9999
    BalancerMember http://localhost:9998 status=+H
  </Proxy>
  <Proxy *>
    Order Allow,Deny
    Allow From All
  </Proxy>
  ProxyPreserveHost On
  ProxyPass /balancer-manager !
  ProxyPass / balancer://mycluster/
  ProxyPassReverse / balancer://mycluster/
</VirtualHost>
```

The important part is `balancer://mycluster`. This declares a load balancer. The +H option means that the second Play application is on standby. But you can also instruct it to load balance.

Apache also provides a way to view the status of your cluster. Simply point your browser to `/balancer-manager` to view the current status of your clusters.

Because Play is completely stateless you don’t have to manage sessions between the 2 clusters. You can actually easily scale to more than 2 Play instances.

Note that [Apache does not support Websockets](https://issues.apache.org/bugzilla/show_bug.cgi?id=47485), so you may wish to use another front end proxy (such as [HAProxy](http://www.haproxy.org/) or Nginx) that does implement this functionality.

Note that [ProxyPassReverse might rewrite incorrectly headers](https://issues.apache.org/bugzilla/show_bug.cgi?id=51982) adding an extra / to the URIs, so you may wish to use this workaround:
`ProxyPassReverse / http://localhost:9999 ProxyPassReverse / http://localhost:9998`


##配置受信任的代理
为判断客户端IP地址Play必须知道在你的网络哪个是受信任的代理。

这些可以配置在`play.http.forwarded.trustedProxies`。你可以定义一个代理列表和/或网络掩码，Play 识别为属于您的网络。

默认是`127.0.0.1` 和`::FF`

在HTTP-标头中如何设置代理存在两种可能性:

* 早期方法用 X-Forwarded 标头
* RFC 7239 用 Forwarded 标头

要解析的标头类型是通过`play.http.forwarded.version`设置。有效值是 `x-forwarded` 或0 `rfc7239`。
默认是`x-forwarded`。

要了解更多信息,请参阅[RFC 7239](https://tools.ietf.org/html/rfc7239)。