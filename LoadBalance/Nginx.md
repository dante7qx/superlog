## Nginx

### 一. 安装 - MAC

```sh
brew install nginx
brew services start nginx
brew uninstall nginx
```

```sh
==> Pouring nginx-1.12.0_1.sierra.bottle.tar.gz
==> Caveats
Docroot is: /usr/local/var/www

The default port has been set in /usr/local/etc/nginx/nginx.conf to 8080 so that
nginx can run without sudo.

nginx will load all files in /usr/local/etc/nginx/servers/.

To have launchd start nginx now and restart at login:
  brew services start nginx
Or, if you don't want/need a background service you can just run:
  nginx
Log store in /usr/local/var/log/nginx or /usr/local/Cellar/nginx/1.12.0_1/logs
```

常用命令

```sh
nginx  #启动nginx
nginx -s quit  #快速停止nginx
nginx -V #查看版本，以及配置文件地址
nginx -v #查看版本
nginx -s reload|reopen|stop|quit   #重新加载配置|重启|快速停止|安全关闭nginx
nginx -h #帮助
```

全局配置（https://www.cnblogs.com/magicsoar/p/5817734.html）

```properties
daemon on | off   默认on

是否以守护进程的方式运行nginx，守护进程是指脱离终端并且在后头运行的进程，关闭守护进程执行的方式可以让我们方便调试nginx。Docker 中使用 nginx -g "daemon off;"

master_process on | of 默认on

是否以master/worker方式进行工作，在实际的环境中 nginx是以一个master进程管理多个worker进程的方式运行的，关闭后 nginx就不会fork出worker子进程来处理请求，

而是用master进程自身来处理请求
```

### 二. 配置

#### 1. 默认配置详解

- http块：可以嵌套多个server，配置代理，缓存，日志定义等绝大多数功能和第三方模块的配置。
- server块：配置虚拟主机的相关参数，一个http中可以有多个server。
- location块：配置请求的路由，以及各种页面的处理情况，一个server可以用多个location。

```nginx
## Nginx用户及组：用户 组。window下不指定。user nginx nginx
#user  nobody;

## 工作进程：数目。根据硬件调整，通常等于CPU数量或者2倍于CPU。
worker_processes  1;

## 错误日志：存放路径。
#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

## 进程标识符：存放路径。
#pid        logs/nginx.pid;


events {
  	## 每个工作进程的最大连接数。根据硬件调整，和前面工作进程配合起来用，尽量大，但是别把cpu跑到100%就行。每个进程允许的最多连接数，理论上每台nginx服务器的最大连接数为。worker_processes*worker_connections
    worker_connections  1024;
}


http {
  	## 文件扩展名与文件类型映射表
    include       mime.types;
  	
  	## 默认文件类型，默认为text/plain
    default_type  application/octet-stream;

  	## 日志格式设置。 若是空值，则用 - 替代
  	## $remote_addr与$http_x_forwarded_for用以记录客户端的ip地址；
  	## $remote_user：用来记录客户端用户名称；
  	## $time_local： 用来记录访问时间与时区；
  	## $request： 用来记录请求的url与http协议；
  	## $status： 用来记录请求状态；成功是200；
  	## $body_bytes_sent：记录发送给客户端文件主体内容大小；
  	## $http_referer：用来记录从那个页面链接访问过来的；
  	## $http_user_agent：记录客户浏览器的相关信息；
  	## 例子：127.0.0.1 - - [04/Jul/2017:10:46:33 +0800] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/59.0.3071.115 Safari/537.36" "-
  
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

  	## 启动高效传输文件的模式。sendfile可以让Nginx在传输文件时直接在磁盘和tcp socket之间传输数据。如果这个参数不开启，会先在用户空间（Nginx进程空间）申请一个buffer，用read函数把数据从磁盘读到cache，再从cache读取到用户空间的buffer，再用write函数把数据从用户空间的buffer写入到内核的buffer，最后到tcp socket。开启这个参数后可以让数据不用经过用户buffer。
    sendfile        on;
  
  	## 一次发送数据的包大小。它不是按时间累计  0.2 秒后发送包，而是当包累计到一定大小后就发送。tcp_nopush 必须和 sendfile 搭配使用。
    #tcp_nopush     on;

    ## 连接超时时间，默认为75s，可以在http，server，location块。
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       8080;
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
    include servers/*;
}
```

#### 2. 反向代理

	反向代理（Reverse Proxy）方式是指以代理服务器来接受internet上的连接请求，然后将请求转发给内部网络上的服务器，并将从服务器上得到的结果返回给internet上请求连接的客户端，此时代理服务器对外就表现为一个反向代理服务器。简单来说就是真实的服务器不能直接被外部网络访问，所以需要一台代理服务器，而代理服务器能被外部网络访问的同时又跟真实服务器在同一个网络环境，当然也可能是同一台服务器，端口不同而已。

```nginx
server {
	listen       8080;
	server_name  localhost;
  	client_max_body_size 200M;	# 客户端上传限制
	access_log  logs/reverse-proxy.access.log  main;

	location / {
  		proxy_pass http://localhost:8100;
		proxy_set_header Host $host:$server_port;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_redirect off; # 关闭重定向  
	}
}
```

**X-Forwarded-For**

简称XFF头，它代表客户端，也就是HTTP的请求端真实的IP，只有在通过了HTTP 代理或者负载均衡服务器时才会添加该项。标准格式：`X-Forwarded-For: client1, proxy1, proxy2`

- ***proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;***

  $proxy_add_x_forwarded_for变量包含客户端请求头中的"X-Forwarded-For"，与$remote_addr用逗号分开，如果没有"X-Forwarded-For" 请求头，则$proxy_add_x_forwarded_for等于$remote_addr。$remote_addr变量的值是客户端的IP。

- 被代理的后台服务通过 ***X-Forwarded-For Header*** 来获取客户端真实的IP。

#### 3. 负载均衡

	Nginx作为一个非常有效的HTTP负载平衡器，将流量分配给多个应用服务器，并通过nginx提高Web应用程序的性能，可扩展性和可靠性。Nginx 本身支持3种负载策略：

- **round-robin**（轮询）：默认的策略。每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器down掉，能自动剔除。

```nginx
upstream test {
	server localhost:8100;
	server localhost:8101;
}

server {
	listen       8081;
	client_max_body_size 200M;
	access_log  logs/upstream.access.log  main;
	location / {
		proxy_pass http://test;
		
		proxy_set_header Host $host:$server_port;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_redirect off;
	}
}
```

==注意：`proxy_pass http://test ` 的意思是，nginx 将请求按指定策略传递到 `upstream test` 块中，并不会将请求发送到反向代理的Url上。proxy_pass 和 upstream 要一一对应。==

- **least-connected** （最少连接）：nginx将不会在负载很忙的服务器上增加请求，而是分发新的请求到不忙的服务器上面。

```nginx
upstream test {
    least_conn;
    server localhost:8100;
    server localhost:8101;
    server localhost:8102;
}
```

- **ip-hash**：将客户端的IP作为一个 hashing key，绑定到负载池中的一台服务器。每个访客（IP固定）就会固定的访问一个后段的服务（除非后台服务器宕机），可以解决session等有状态的问题。

```nginx
upstream test {
    ip_hash;
    server localhost:8100;
    server localhost:8101;
    server localhost:8102;
}
```

- **weighted**（权重）：指定轮询几率，weight和访问比率成正比，用于后端服务器性能不均的情况。 如果后端服务器down掉，能自动剔除。 ==least_conn、ip_hash 策略也可指定 weight==。


```nginx
upstream test {
    server localhost:8100 weight=5;
    server localhost:8101 weight=3;
    server localhost:8102 weight=2;
}
```

- 第三方（fair、url_hash）

  后续...

#### 4. Rewrite

### 三. 非Root启动

1. apache的80端口为系统保留端口，如果通过其他非root用户启动，会报错。普通用户只能用1024以上的端口，1024以内的端口只能由root用户使用。
2. Nignx 默认配置下，会去读写需要 ROOT 权限的文件和目录，需要提供自定义的配置文件。
3. 注释配置文件的第一行，`# user nginx;`
4. Dockerfile 中设置

```ini
RUN chown -R 1001:0 $HOME /usr/share/nginx /var /run && \
    chown 1001:0 /usr/sbin/nginx && \
    chmod 755 /usr/sbin/nginx && chmod u+s /usr/sbin/nginx && \
    chmod -R 755 $STI_SCRIPTS_PATH && \
    chmod -R g+rw /opt/s2i/destination

USER 1001
```



### 八. 参考

- http://blog.csdn.net/happydream_c/article/details/54943802
- http://blog.jobbole.com/110400/
- http://wiki.nginx.org/NginxHttpUpstreamModule
- https://github.com/Homebrew/homebrew-nginx
- https://segmentfault.com/a/1190000002797606
- http://www.cnblogs.com/pycode/p/6588896.html
- https://zhongwuzw.github.io/2016/09/24/%E8%B0%83%E6%95%B4Nginx%E7%9A%84%E7%BD%91%E7%AB%99%E6%A0%B9%E7%9B%AE%E5%BD%95/
- https://www.cnblogs.com/SummerinShire/p/7410557.html (配置优化)