---
title: nginx multi-domain config
date: 2020-09-13 15:13:15
tags: nginx
---

nginx配置多域名转发方法，同一台机器上，配置不同域名转发到不同的后端。

## 快速开始

### win环境配置

- nginx安装路径: `d:\setup\nginx\nginx-1.19.2`
- 配置文件路径: `d:\setup\nginx\nginx-1.19.2\conf`

#### 原理
- 通过修改`nginx.conf`文件，如下:
```
#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   html;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

}
```

- 默认的配置文件内容解析如下：
    - `work_processes`: 即启动的工作进程数量。
    - `events`: socket的events配置，主要是连接数等
        - `worker_connections`: 即每个工作进程运行的最大连接数。
    - `http`: http的具体配置
        - `include`: 有点类似于c语言的`include`，将包含的文件内容替换到该处。
        - `default_type`: http请求的默认文件类型。
        - `sendfile`: 是否开启0拷贝的方式，通常win/linux都有相对应的api，能够实现高效的文件传输。
        - `keepalive_timeout`: 保活超时。
        - `server`: http服务器配置，可与允许有多段，同时也可以用`include <file>`的方式引入其他文件，内容只有`server`段，注意这里引用文件名称允许的方式：
            - 1. 相对于当前conf文件的路径，如: `vhost\*.conf`，补全的完整路径即为: `d:\setup\nginx\nginx-1.19.2\conf\vhost\*.conf`
            - 2. 绝对路径。
        - 多`server`配置: 即为多域名配置，当多个服务器，分别通过不同的域名分发流量的时候，则通常采用这种方式。如果将多个`server`段都配置到默认配置文件中，可能会造成文件很长，通常建议用`include`的方式，将不同的域名包含进来。
    
- 通过上述的配置文件的不同段落的作用，假定我们有两个域名: `www.abc.com/www.ABC.com`
    - 思考: 
    - 答案: 可以采用在默认配置文件的`http`字段里面添加`server`字段的方式，也可以通过`include`的方式引入。
    - 域名`www.abc.com`的段配置如下:
        ```
        server {
            listen       80;
            server_name  www.abc.com;

            # 其他路由配置。
            ...
        }
        ```
    - 域名`www.ABC.com`的段配置如下:
        ```
        server {
            listen       80;
            server_name  www.ABC.com;

            # 其他路由配置。
            ...
        }
        ```
    - 方案一: 将上面的两个段，追加到默认配置文件的http段的最后。略。
    - 方案二: 
        - 将上面的两个文件，分别保存为文件： `www.abc.com.conf`/`www.ABC.com.conf`
        - 在默认配置文件目录创建新目录: `vhost`
        - 将上述的两个文件拷贝到`vhost`目录。
        - 在默认配置文件的`http`段末尾追加一行: `include vhost\*.conf`
    - 如上的二选一方案配置完成后，需要重启或者重新加载nginx
    
- 至此，配置完成。


