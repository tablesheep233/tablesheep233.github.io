---
layout:     note
title:      "nginx memo"
subtitle:   " \"nginx memo\""
author:     "tablesheep"
catalog: true
hide-in-nav: true
header-style: text
tags:
- note
- nginx
---


> nginx 备忘


# Installation

依赖

```sh
yum -y install make zlib zlib-devel gcc-c++ libtool  openssl openssl-devel #编译工具及库文件
```



编译、安装

```sh
./configure --prefix=/安装目录 --conf-path=/配置目录 \
 --with-http_ssl_module \ #添加ssl module
 --with-openssl=/openssl安装目录，非自定义安装可省略 \
 --with-stream \ #添加stream module
 --with-http_gzip_static_module \ #
 --add-module=/module-path #添加第三方module

 
 make & make install
```

[编译参数](http://nginx.org/en/docs/configure.html)



**问题：zlib、openssl升级导致nginx编译找不到路径**

```conf
以openssl为例，编辑/nginx-xxx/auto/lib/openssl/conf文件
将
 CORE_INCS="$CORE_INCS $OPENSSL/.openssl/include"
 CORE_DEPS="$CORE_DEPS $OPENSSL/.openssl/include/openssl/ssl.h"
 CORE_LIBS="$CORE_LIBS $OPENSSL/.openssl/lib/libssl.a"
 CORE_LIBS="$CORE_LIBS $OPENSSL/.openssl/lib/libcrypto.a"

改为
  CORE_INCS="$CORE_INCS $OPENSSL/include"
  CORE_DEPS="$CORE_DEPS $OPENSSL/include/openssl/ssl.h"
  CORE_LIBS="$CORE_LIBS $OPENSSL/lib/libssl.a"
  CORE_LIBS="$CORE_LIBS $OPENSSL/lib/libcrypto.a"
  
 即将目录改正确
```



```sh
nginx -V 查看版本以及编译配置
```

> 重新编译后替换sbin目录即可



**nginx 第三方module**

[NGINX 3rd Party Modules](https://www.nginx.com/resources/wiki/modules/)

- [headers-more-nginx-module](https://github.com/openresty/headers-more-nginx-module) header 修改、删除
- [nginx-rtmp-module](https://github.com/arut/nginx-rtmp-module) rtmp流代理
- [lua-nginx-module](https://github.com/openresty/lua-nginx-module) nginx lua
- [nginx-upstream-fair](https://github.com/gnosek/nginx-upstream-fair) upstream fair策略
- ......



# nginx.conf

## 全局配置

```nginx
#nignx worker进程数量 auto让nginx自动检测
worker_processes auto;
```

## events

```nginx
events {
    #设置网络连接序列化，防止惊群现象，默认为on
    accept_mutex on;
    #设置worker进程是否一次性的接收监听队列里的所有请求，为off（默认）时要一个个接收
    multi_accept on; 
    #worker进程最大连接数
    worker_connections  1000;
    #I/O模型，select|poll|kqueue|epoll
    #use epoll; 
}
```



## http

```nginx
http {
    include       mime.types;
    default_type  application/octet-stream;
    
    #指定日志名称、格式
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for" "$request_time" "$upstream_response_time" "$upstream_addr" "$upstream_status"';
    #指定日志位置，以及输出格式
    #access_log  logs/access.log  main;

    log_format  json  '{"@timestamp":"$time_iso8601",'
                       '"server_addr":"$server_addr",'
                       '"hostname":"$hostname",'
                       '"remote_add":"$remote_addr",'
                       '"request_method":"$request_method",'
                       '"scheme":"$scheme",'
                       '"server_name":"$server_name",'
                       '"http_referer":"$http_referer",'
                       '"request_uri":"$request_uri",'
                       '"args":"$args",'
                       '"body_bytes_sent":$body_bytes_sent,'
                       '"status": $status,'
                       '"request_time":$request_time,'
                       '"upstream_response_time":"$upstream_response_time",'
                       '"upstream_addr":"$upstream_addr",'
                       '"http_user_agent":"$http_user_agent",'
                       '"https":"$https"'
                       '}'; 
    
    access_log  logs/access.log  json;
    error_log logs/error.log  error;
    
    #使用sendfile函数优化性能（零拷贝），tcp_nopush 等数据包累积到一定大小才发送，tcp_nodelay 是要尽快发送，两个都配置on表示先填满包，再尽快发送。
    sendfile           on;
    tcp_nopush         on;
    tcp_nodelay        on;
    
    keepalive_timeout  60;
    
    #gizp 配置
    gzip               on;
    #vary 响应头，与缓存有关
    gzip_vary          on;
    #压缩等级，1-9，越高压缩效果越好，相应的时间越长
    gzip_comp_level    6;
    #压缩所需要的缓冲区大小
    gzip_buffers       32 4k;
    #使用压缩的最小文件大小
    gzip_min_length    1k;
    #反向代理的时候启用any表示压缩所有结果数据
    gzip_proxied       any;
    #通过表达式，与UserAgent匹配，满足时不使用gzip压缩，msie6 等价于 MSIE [4-6]\.，但性能更好一些
    gzip_disable       "msie6";
    #gzip HTTP 最低版本
    gzip_http_version  1.0;
    #压缩类型
    gzip_types         text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript application/javascript image/svg+xml;
    
    #加载 /home/nginx_conf目录下的配置文件
    #include            /home/nginx_conf/*.conf;
    
    ########upstream 负载均衡#####
    upstream api-cluster {
        server 192.168.1.1:2333;
        #max_fails表示允许失败次数，fail_timeouts表示max_fails后暂停时间
        server 192.168.1.2:2333 max_fails=2 fail_timeouts=30s; 
        server 192.168.1.3:2333 down; #down表示下线，不参与负载
        server 192.168.1.4:2333 backup; #表示备用节点
    }
    
    ##权重配置##
    upstream api-cluster-weight {
        server 192.168.1.1:2333 weight=2;
        server 192.168.1.2:2333 weight=1;
    }
    
    ##ip_hash,将同IP的请求固定在同一服务器##
    upstream api-cluster-ip {
        ip_hash;
        server 192.168.1.1:2333;
        server 192.168.1.2:2333;
    }
    
    ##按响应时间分配，需要第三方module##
    upstream api-cluster-fair {
        server 192.168.1.1:2333;
        server 192.168.1.2:2333;
        fair;
    }
    
    server {
        listen       80;
        server_name  localhost;
        #隐藏Server响应头版本，也可通过改源码或者移除响应头的方法更彻底解决
        server_tokens off;
        #nginx默认忽略下划线的header,可以开启不忽略，但是建议对于自定义的header还是使用-
        underscores_in_headers on;
        
        #请求方法限制
        if ($request_method !~ ^(POST|GET|DELETE|PUT|OPTIONS)$ ) {
          return 501;
        }
        
        ##############静态资源###########
        
        #匹配文件，在html目录下存放着前端目录front
        #使用root配置 访问 127.0.0.1/front/index.html, 实际目录/html/front/index.html
        #root 会拼接目录
        location ^~ /front/ {
            root html;
        }
        
        #使用alias配置 访问 127.0.0.1/front/index.html, 实际目录/html/front/index.html
        #alias 不会拼接
        #注意当location 的路径配置了/, alias的目录结尾也要带上/
        location ^~ /front/ {
            alias html/front/;
        }
        
        #后缀匹配
        location ~* \.(gif|jpg|jpeg|png|ico)$ {
            root html/static/;
        }
        
        #######后端服务代理######
        location /backend/ {
            proxy_pass http://api-cluster/;
            #proxy_set_header 重新设置或添加请求头给服务端
            proxy_set_header Host $host:$server_port;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Real-PORT $remote_port;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            
            #请求重定向、刷新 url重写 proxy_redirect [ default|off|redirect replacement ]
            proxy_redirect default;
            
            #允许192.168.0.0/24网段和127.0.0.1的请求，其它全部拒绝
            allow 192.168.0.0/24;
            allow 127.0.0.1;
            deny all;
        }
        
        #rewrite regex replacement [flag];
        #URL重写 regex用于匹配URI的正则 replacement 重写的URI
        #flag 常用的有redirect（302 临时重定向） | permanent（301 永久重定向）
        location /search/ {
          rewrite (?<=/search/)(.*) https://www.baidu.com/s?wd=$1 permanent;
        }
        
        #状态查看
        location /nginx_status{
            stub_status on;
            access_log   on;
            allow 192.168.0.0/24;
            allow 127.0.0.1;
            deny all;
        }
    }

    #https配置 http://nginx.org/en/docs/http/ngx_http_ssl_module.html
    #https://docshome.gitbook.io/nginx-docs/he-xin-gong-neng/http/ngx_http_ssl_module
    server {
        listen 443 default ssl;
        server_name 域名;
        # 中间证书 + 站点证书
    	ssl_certificate /home/.../domain.pem;
    	# 创建 CSR 文件时用的密钥
    	ssl_certificate_key /home/.../domain.key;
        #ssl协议
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        #配置ssl加密算法，多个算法用:分隔，ALL表示全部算法，!表示不启用该算法，+表示将该算法排到最后面去。 使用openssl ciphers查看完整列表
        ssl_ciphers EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5;
        #是否由服务器决定采用哪种加密算法
        ssl_prefer_server_ciphers on;
        ssl_session_cache         shared:SSL:10m;
        ssl_session_timeout       30m;
        ssl_session_tickets      on;
        
        #安全配置
        location / {
            #要求浏览器收到请求后的max-age秒的时间内凡是访问这个域名下的请求都使用HTTPS请求。includeSubDomains代表子域名也适用
            add_header  Strict-Transport-Security  "max-age=31536000; includeSubDomains";
            #是否允许页面被嵌套 DENY：不允许被任何页面嵌入 | SAMEORIGIN：不允许被本域以外的页面嵌入
			add_header  X-Frame-Options  deny;
            #禁用浏览器的类型猜测 通常浏览器根据Content-Type来判断类型，但如果Content-Type是错的或者未定义，某些浏览器会自己猜测类型、解析内容
			add_header  X-Content-Type-Options  nosniff;
            #防范XSS 0：禁用XSS保护 | 1：启用XSS保护 | 1; mode=block：启用XSS保护，并在检查到XSS攻击时，停止渲染页面
            add_header X-XSS-Protection "1; mode=block";
            
            #此外还有 X-Content-Security-Policy 定义页面可以加载哪些资源
        }
    }
}
```



## stream

```nginx
stream {
    upstream mysql {
        server ip:post;
    }


    server {
        listen post;
        proxy_connect_timeout 2s;
        proxy_timeout 180s;
        proxy_pass mysql;
    }
}
```



## auth_basic

> ```shell
> openssl passwd 密码 #生成密码
> ```

将生成的密码放在文件中，格式

> username:password

```nginx
    location /api/doc.html {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Real-Port $remote_port;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://backend/doc.html;
        auth_basic "secret";
        auth_basic_user_file /app/nginx/security/passwd.db; #鉴权文件
    }
```



## auth_request

auth_request 鉴权模块

> --with-http_auth_request_module 编译时加入

```nginx
        location /internal-auth {
            internal;
            proxy_pass http://127.0.0.1:10909/auth;
            proxy_pass_request_body off;
            proxy_set_header Content-Length "";
        }

        location /nacos/ {
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Real-Port $remote_port;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://nacos;
            auth_request /internal-auth;
        }
```

