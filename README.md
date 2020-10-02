
### Nginx 简介

Nginx ("engine x") 是一个高性能的 HTTP 和 反向代理服务器，也是一个 IMAP/POP3/SMTP 代理服务器。 Nginx 是由 Igor Sysoev 为俄罗斯访问量第二的 
Rambler.ru 站点开发的，第一个公开版本0.1.0发布于2004年10月4日。其将源代码以类BSD许可证的形式发布，因它的稳定性、丰富的功能集、示例[配置](http://www.cnblogs.com/knowledgesea/p/5175711.html)文件和低系统资源的消耗而闻名。

尽管Node.JS的性能不错，但处理静态事务确实不是他的专长，如：gzip编码，静态文件，HTTP缓存，SSL处理，负载平衡和反向代理及多站点代理等，都可以通过nginx 来完成，从而减小node.js的负载，并通过nginx强大的缓存来节省您网站的流量从而提高网站的加载速度。

nginx 配置文件[nginx.conf](nginx_default.conf), 完整例子看 [这里](https://www.nginx.com/resources/wiki/start/topics/examples/full/)， 详细解释请看[这里](http://blog.argteam.com/coding/hardening-node-js-for-production-part-2-using-nginx-to-avoid-node-js-load/),[其他配置](nginx_server)。

### upstream
The upstream directive specifies that these two instances work in tandem as an upstream server for nginx. The keepalive 64; directs nginx to keep a minimum of 64 HTTP/1.1 connections to the proxy server at any given time. This is a true minimum: if there is more traffic then nginx will open more connections to the proxy.


### 静态文件拦截器
  location ~ ^/(images/|img/|javascript/|js/|css/|stylesheets/|flash/|media/|static/|robots.txt|humans.txt|favicon.ico) {
    root /usr/local/silly_face_society/node/public;
    access_log off;
    expires max;
  }


## SELinux 对 nginx 影响
当SELinux开启时，将会禁止访问设置在其他路径下的地址。比如 server 中设置 

      root  /home/www/public

无论你将文件的权限设置为777 还是多少，日志中都会提示  

      ：***  open() "/home/www/centre/public/index.html" failed (13: Permission denied), client:   ***   

只有关闭了SELinux后，才能正常访问。

nginx和selinux冲突解决:

取出selinux中有关于nginx被拒绝的信息，然后通过一些手段将这些文件设置成通过
cat /var/log/audit/audit.log |grep nginx |grep denied| audit2allow -M mynginx
semodule -i mynginx.pp

 
查看 selinux 状态：

/usr/sbin/sestatus -v

临时修改 selinux 状态命令：

setenforce [ Enforcing | Permissive | 1 | 0 ]   // 1 开启， 0 关闭

永久关闭，需要设置文件/etc/sysconfig/selinux 并重启才能生效。

## nginx 安装
1：下载安装包
2：解压 tar -zxvf nginx-1.5.9.tar.gz 
3. 安装依赖包

      aptitude install libpcre3 libpcre3-dev libpcrecpp0 libssl-dev zlib1g-dev
3：设置一下配置信息

      ./configure \
      --prefix=/usr/local/nginx \
      --pid-path=/var/run/nginx/nginx.pid \
      --lock-path=/var/lock/nginx.lock \
      --error-log-path=/var/log/nginx/error.log \
      --http-log-path=/var/log/nginx/access.log \
      --with-http_gzip_static_module \
      --http-client-body-temp-path=/var/temp/nginx/client \
      --http-proxy-temp-path=/var/temp/nginx/proxy \
      --http-fastcgi-temp-path=/var/temp/nginx/fastcgi \
      --http-uwsgi-temp-path=/var/temp/nginx/uwsgi \
      --http-scgi-temp-path=/var/temp/nginx/scgi
4：make 编译      
5：make install 安装     

> ubuntu 依赖库zlib，pcre，openssl安装方法

      解决依赖包openssl安装，命令：sudo apt-get install openssl libssl-dev
      解决依赖包pcre安装，命令：sudo apt-get install libpcre3 libpcre3-dev
      解决依赖包zlib安装，命令：sudo apt-get install zlib1g-dev


## 配置服务
创建脚本 
      
      vim /etc/init.d/nginx   

[ubuntu](ubuntu_nginx)              

[CentOS](centos_nginx)         


添加脚本到系统默认运行级别

      /usr/sbin/update-rc.d -f nginx defaults
 
链接到/etc/下

      ln -s /usr/local/nginx  /etc/nginx
 
可以运行nginx了

      /etc/init.d/nginx start

测试

      lynx http://localhost


## nginx 命令
nginx 重加载

      sudo nginx -s reload

## root 和 alias 区别
nginx 指定文件路径有两种方式root和alias，指令的使用方法和作用域：     
[root]

      语法：root path
      默认值：root html
      配置段：http、server、location、if

[alias]

      语法：alias path
      配置段：location

root与alias主要区别在于nginx如何解释location后面的uri，这会使两者分别以不同的方式将请求映射到服务器文件上。root的处理结果是 root路径＋location路径；alias的处理结果是 使用alias路径替换location路径；alias是一个目录别名的定义，root则是最上层目录的定义。还有一个重要的区别是alias后面必须要用“/”结束，否则会找不到文件的，而root则可有可无。

root实例：

      location ^~ /t/ {
           root /www/root/html/;
      }
如果一个请求的URI是/t/a.html时，web服务器将会返回服务器上的/www/root/html/t/a.html的文件。

alias实例：

      location ^~ /t/ {
       alias /www/root/html/new_t/;
      }
如果一个请求的URI是/t/a.html时，web服务器将会返回服务器上的/www/root/html/new_t/a.html的文件。注意这里是new_t，因为alias会把location后面配置的路径丢弃掉，把当前匹配到的目录指向到指定的目录。

注意：    
1. 使用alias时，目录名后面一定要加"/"。
3. alias在使用正则匹配时，必须捕捉要匹配的内容并在指定的内容处使用。
4. alias只能位于location块中。（root可以不放在location中）


## nginx 正确服务 react-router
react应用在运行时会更改浏览器uri而又不真的希望服务器对这些uri去作响应，如果此时刷新浏览器，服务器当然是收到浏览器发来的uri就去寻找资源，这个uri在服务器上是没有对应资源，结果服务器因找不到资源就发送403错误标志给浏览器。

所以，我们要做的调整是：浏览器在使用这个react应用期间，无论uri更改与否，服务器都发回index.html这个页面就行。

      server {  
        ...
        location / {
          try_files $uri $uri/ /index.html;
        }
      }

try_files $uri $uri/ /index.html是nginx重定向指令，意思是在站点查找有无浏览器发来的uri，如果没有那就发送index.html这个文件给浏览器。


## nginx location 正则写法
语法

      Syntax:	location [ = | ~ | ~* | ^~ ] uri { ... }
                  location @name { ... }
      Default:	—
      Context:	server, location

* 以 = 开头表示精确匹配
* 以 ~ 开头表示区分大小写的正则匹配
* 以 ~* 开头表示不区分大小写的正则匹配
* 以 ^~ 开头表示uri以某个常规字符串开头，不是正则匹配
* 以 / 通用匹配, 如果没有其它匹配,任何请求都会匹配到

顺序优先级：      
(location =) > (location 完整路径) > (location ^~ 路径) > (location ~,~* 正则顺序) > (location 部分起始路径) > (/)    
    
```shell
location ~* \.(gif|jpg|jpeg)$ {
  # 匹配所有以 gif,jpg或jpeg 结尾的请求
  # 然而，所有请求 /images/ 下的图片会被 config D 处理，因为 ^~ 到达不了这一条正则
  [ configuration E ] 
}

location /images/ {
  # 字符匹配到 /images/，继续往下，会发现 ^~ 存在
  [ configuration F ] 
}

location /images/abc {
  # 最长字符匹配到 /images/abc，继续往下，会发现 ^~ 存在
  # F与G的放置顺序是没有关系的
  [ configuration G ] 
}

location ~ /images/abc/ {
  # 只有去掉 config D 才有效：先最长匹配 config G 开头的地址，继续往下搜索，匹配到这一条正则，采用
    [ configuration H ] 
}

location ~* /js/.*/\.js
```

## [Rewrite 规则](https://segmentfault.com/a/1190000002797606)     
rewrite功能就是，使用nginx提供的全局变量或自己设置的变量，结合正则表达式和标志位实现url重写以及重定向。    
rewrite只能放在server{},location{},if{}中，并且只能对域名后边的除去传递的参数外的字符串起作用，例如     
      http://seanlook.com/a/we/index.php?id=1&u=str 
只对/a/we/index.php重写。

语法      

      rewrite regex replacement [flag];

如果相对域名或参数字符串起作用，可以使用全局变量匹配，也可以使用proxy_pass反向代理。

表面看rewrite和location功能有点像，都能实现跳转，主要区别在于rewrite是在同一域名内更改获取资源的路径，而location是对一类路径做控制访问或反向代理，可以proxy_pass到其他机器。很多情况下rewrite也会写在location里，它们的执行顺序是：

* 执行server块的rewrite指令
* 执行location匹配
* 执行选定的location中的rewrite指令
* 如果其中某步URI被重写，则重新循环执行1-3，直到找到真实存在的文件；
* 循环超过10次，则返回500 Internal Server Error错误。


## 代理
在 nginx 中配置`proxy_pass`代理转发时，如果在`proxy_pass`后面的url加/，表示绝对根路径；如果没有/，表示相对路径，把匹配的路径部分也给代理走。

nginx 中有两个模块都有`proxy_pass`指令，都是用来做后端代理的指令：
* `ngx_http_proxy_module` 的 proxy_pass，只能在 `server` 段使用使用, 只需要提供域名或ip地址和端口。可以理解为端口转发，可以是tcp端口，也可以是udp端口。
* `ngx_stream_proxy_module` 的 proxy_pass，需要在 `location` 中的`if`，`limit_except`段中使用，处理需要提供域名或ip地址和端口外，还需要提供协议，如"http"或"https"，还有一个可选的uri可以配置。

示例：
```
server {
    listen      80;
    server_name www.test.com;

    # 情形A
    # 访问 http://www.test.com/testa/aaaa
    # 后端的request_uri为: /testa/aaaa
    location ^~ /testa/ {
        proxy_pass http://127.0.0.1:8801;
    }
    
    # 情形B
    # 访问 http://www.test.com/testb/bbbb
    # 后端的request_uri为: /bbbb
    location ^~ /testb/ {
        proxy_pass http://127.0.0.1:8801/;
    }

    # 情形C
    # 下面这段location是正确的
    location ~ /testc {
        proxy_pass http://127.0.0.1:8801;
    }

    # 情形D
    # 下面这段location是错误的
    #
    # nginx -t 时，会报如下错误: 
    #
    # nginx: [emerg] "proxy_pass" cannot have URI part in location given by regular 
    # expression, or inside named location, or inside "if" statement, or inside 
    # "limit_except" block in /opt/app/nginx/conf/vhost/test.conf:17
    # 
    # 当location为正则表达式时，proxy_pass 不能包含URI部分。本例中包含了"/"
    location ~ /testd {
        proxy_pass http://127.0.0.1:8801/;   # 记住，location为正则表达式时，不能这样写！！！
    }

    # 情形E
    # 访问 http://www.test.com/ccc/bbbb
    # 后端的request_uri为: /aaa/ccc/bbbb
    location /ccc/ {
        proxy_pass http://127.0.0.1:8801/aaa$request_uri;
    }

    # 情形F
    # 访问 http://www.test.com/namea/ddd
    # 后端的request_uri为: /yongfu?namea=ddd
    location /namea/ {
        rewrite    /namea/([^/]+) /yongfu?namea=$1 break;
        proxy_pass http://127.0.0.1:8801;
    }

    # 情形G
    # 访问 http://www.test.com/nameb/eee
    # 后端的request_uri为: /yongfu?nameb=eee
    location /nameb/ {
        rewrite    /nameb/([^/]+) /yongfu?nameb=$1 break;
        proxy_pass http://127.0.0.1:8801/;
    }

    access_log /data/logs/www/www.test.com.log;
}

server {
    listen      8801;
    server_name www.test.com;
    
    root        /data/www/test;
    index       index.php index.html;

    rewrite ^(.*)$ /test.php?u=$1 last;

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_pass unix:/tmp/php-cgi.sock;
        fastcgi_index index.php;
        include fastcgi.conf;
    }

    access_log /data/logs/www/www.test.com.8801.log;
}
```


***
进阶     
[nginx平台初探](http://tengine.taobao.org/book/chapter_02.html)     
[ngx_http_rewrite_module](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#break)        
[nginx之proxy_pass指令完全拆解](https://my.oschina.net/foreverich/blog/1512304)      




