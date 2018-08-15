## HAproxy

### 一. 概念

​	HAProxy是一款提供高可用性、负载均衡以及基于TCP（L4）和HTTP（L7）应用的代理软件。Nginx （免费版）只能支持 L7 层的负载。HAProxy 的主要功能如下：

- **路由**

  根据静态分配的cookie路由HTTP请求。

 - **LoadBalance**

   L4和L7两种模式，支持**RR / 静态RR / LC / IP Hash / URI Hash / URL_PARAM Hash / HTTP_HEADER Hash**等丰富的负载均衡算法。

- **健康检查**

  支持TCP和HTTP两种健康检查模式。

- **会话保持**

  对于未实现会话共享的应用集群，可通过**Insert Cookie/Rewrite Cookie/Prefix Cookie**，以及上述的多种Hash方式实现会话保持。

- **SSL**

  HAProxy可以解析HTTPS协议，并能够将请求解密为HTTP后向后端传输。

- **HTTP Request Rewrite 和 Redirect**

- **监控与统计**

  HAProxy提供了基于Web的统计信息页面，展现健康状态和流量数据。基于此功能，使用者可以开发监控程序来监控HAProxy的状态。

### 二. 安装

#### 1. MAC 安装

```shell
## 安装
brew install haproxy
## 启动方式 1，配置文件默认在/usr/local/etc/haproxy.cfg，日志默认在 /usr/local/var/log
brew services start haproxy
## 启动方式 2
haproxy -f /usr/local/etc/haproxy/haproxy.cfg

## 停止
haproxys=$(ps aux | grep haproxy | grep -v grep | awk '{print $2}' | xargs)
for v in ${haproxys}
do
  kill -9 $v
done
```

```shell
------------------------------------ 安装结果 ------------------------------------
==> haproxy
To have launchd start haproxy now and restart at login:
  brew services start haproxy
Or, if you don't want/need a background service you can just run:
  haproxy -f /usr/local/etc/haproxy/haproxy.cfg
```

#### 2. Centos 安装

```shell
wget https://www.haproxy.org/download/1.8/src/haproxy-1.8.13.tar.gz
tar -zxvf haproxy-1.8.13.tar.gz
## 查询操作系统内核版本
uname -r 
## TARGET 规则
# - linux22     for Linux 2.2
# - linux24     for Linux 2.4 and above (default)
# - linux24e    for Linux 2.4 with support for a working epoll (> 0.21)
# - linux26     for Linux 2.6 and above
# - linux2628   for Linux 2.6.28, 3.x, and above (enables splice and tproxy)
make PREFIX=/opt/server/haproxy TARGET=linux2628
make install PREFIX=/opt/server/haproxy
## 创建配置文件 /opt/server/haproxy/conf/haproxy.cfg
mkdir -p /opt/server/haproxy/conf
## 注册系统服务，/etc/init.d目录下添加 HAProxy 启动脚本
vi /etc/init.d/haproxy
chmod +x /etc/init.d/haproxy

## 配置日志
vi /etc/rsyslog.d/haproxy.conf
## 内容
$ModLoad imudp
$UDPServerRun 514
$FileCreateMode 0644  #日志文件的权限
$FileOwner ha  #日志文件的owner
local0.*     /var/log/haproxy.log  #local0接口对应的日志输出文件
local1.*     /var/log/haproxy_warn.log  #local1接口对应的日志输出文件

## 修改rsyslog的启动参数
vi /etc/sysconfig/rsyslog
## 内容
SYSLOGD_OPTIONS="-c 2 -r -m 0"

## 重启rsyslog和HAProxy
service rsyslog restart
service haproxy restart
```

```bash
## haproxy 启动脚本
set -e

PATH=/sbin:/bin:/usr/sbin:/usr/bin:/opt/server/haproxy/sbin
PROGDIR=/opt/server/haproxy
PROGNAME=haproxy
DAEMON=$PROGDIR/sbin/$PROGNAME
CONFIG=$PROGDIR/conf/$PROGNAME.cfg
PIDFILE=$PROGDIR/conf/$PROGNAME.pid
DESC="HAProxy daemon"
SCRIPTNAME=/etc/init.d/$PROGNAME

# Gracefully exit if the package has been removed.
test -x $DAEMON || exit 0

start()
{
       echo -e "Starting $DESC: $PROGNAME\n"
       $DAEMON -f $CONFIG
       echo "."
}

stop()
{
       echo -e "Stopping $DESC: $PROGNAME\n"
       haproxy_pid="$(cat $PIDFILE)"
       kill $haproxy_pid
       echo "."
}

restart()
{
       echo -e "Restarting $DESC: $PROGNAME\n"
       $DAEMON -f $CONFIG -p $PIDFILE -sf $(cat $PIDFILE)
       echo "."
}

case "$1" in
 start)
       start
       ;;
 stop)
       stop
       ;;
 restart)
       restart
       ;;
 *)
       echo "Usage: $SCRIPTNAME {start|stop|restart}" >&2
       exit 1
       ;;
esac
exit 0
```



#### 3. 热备高可用

​	利用 Keepalived 实现 HAProxy 的热备。即两台主机上的两个HAProxy实例同时在线，其中权重较高的实例为MASTER，MASTER出现问题时，另一台实例自动接管所有流量。

 	具体原理，在两台 HAProxy 的主机上分别运行着一个 Keepalived 实例，两个 Keepalived 同时争抢同一个 VIP，两个 HAProxy 也尝试去绑定这同一个 VIP上的端口。Keepalived内部维护一个权重值，权重值最高的Keepalived实例能够抢到 VIP，成为 MASTER。同时Keepalived会定期check本主机上的HAProxy状态，状态OK时权重值增加。

```bash
## 主/备上安装 Keepalived
wget http://www.keepalived.org/software/keepalived-2.0.6.tar.gz
tar -zxvf keepalived-2.0.6.tar.gz
## 安装依赖
yum  -y  install  e2fsprogs-devel keyutils-libs-devel  libsepol-devel libselinux-devel   krb5-devel zlib-devel openssl-devel   popt-devel

./configure --prefix=/usr/local/keepalived
make && make install

## 注册成系统服务
mkdir /etc/keepalived
cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf
cp /usr/local/keepalived/sbin/keepalived  /usr/bin/keepalived
cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/keepalived

## 启动
systemctl start keepalived
```

```nginx
## 配置 /etc/keepalived/keepalived.conf
global_defs {
    router_id LVS_DEVEL  #虚拟路由名称
}

#HAProxy健康检查配置
vrrp_script chk_haproxy {
    script "killall -0 haproxy"  #使用killall -0检查haproxy实例是否存在，性能高于ps命令
    interval 2   #脚本运行周期
    weight 2   #每次检查的加权权重值
}

#虚拟路由配置
vrrp_instance VI_1 {
    state MASTER           # 本机实例状态，MASTER/BACKUP，备机配置文件中请写BACKUP
    interface eno1      #本机网卡名称，使用 ip a 命令查看
    virtual_router_id 51   # 虚拟路由编号，主备机保持一致
    priority 101           # 本机初始权重，备机请填写小于主机的值（例如100）
    advert_int 1           # 争抢虚地址的周期，秒
    virtual_ipaddress {
        10.71.204.99      # 虚地址IP，主备机保持一致（保持和 HAProxy 同一网段）
    }
    track_script {
        chk_haproxy        # 对应的健康检查配置
    }
}
```

### 三. 配置

#### 1. 配置格式

- 命令行方式（优先级最高）

- 全局配置

​        global，设定全局配置参数。

```nginx
global # 全局属性
  node dante_haproxy_1  # 节点名称
  daemon  # 以daemon方式在后台运行
  maxconn 256  # 最大同时256连接
  pidfile /usr/local/etc/haproxy/haproxy.pid  # 指定保存HAProxy进程号的文件
  description "HAProxy 使用说明......"
```

- 代理配置

  - defaults，为其他的 Proxy Config 提供默认的参数，目的提供可读性，可选配置。

  - frontend，描述接受 Client 的 一组 Socket 连接。

  - backend，描述一组后端 Server，代理将 frontend 接受的连接转发到后端 Server。
  - listen，一个包含了 frontend 和 backend 的组合。通常仅对 TCP 流量有用。

  **命名规则**

  所有代理名称必须由大写和小写字母，数字组成，' - '（短划线），'_'（下划线），'.' （点）和':'（冒号）。

#### 2. 时间格式

```nginx
- us : microseconds. ## 微妙
- ms : milliseconds. ## 毫秒
- s  : seconds.      ## 秒
- m  : minutes. 	 ## 分钟
- h  : hours.        ## 小时
- d  : days.         ## 天
```

#### 3. 主要配置说明

- **mode tcp**

  L4 代理，HAProxy 只是转发双方之间的双向流量。

- **mode http**

  L7 代理，HAProxy 分析协议，可通过 frontend 和 backend 的 options 于 Http 请求/响应进行交互。

- **balance**

  后端的负载算法，主要有如下

  - **roundrobin** --> 动态、加权轮询，实时生效，不用重启服务，但是连接数受限，最多支持4128

    ```properties
    server a weight=1;  
    server b weight=2;  
    server c weight=4; 
    
    每收到7个客户端的请求，会把其中的1个转发给后端a，把其中的2个转发给后端b，把其中的4个转发给后端c
    ```

  - **source** --> 基于hash表的算法，类似于nginx中的 iphash

  - **leastconn** --> 最少活动连接数，最好在长连接会话中使用。如sql、ldap

  - **hdr(Host)** --> 基于用户请求的主机名进行调度

  - **url_params** --> 根据url的参数来调度，用于将同一个用户的信息都发送到同一个后端server

### 四. 实际案例

#### 1. HTTP

本机分别启动三个 Java 服务，localhost:8101、localhost:8102、localhost:8103，配置 HAProxy localhost:8100

```nginx
global # 全局属性
  node dante_haproxy_1  # 节点名称
  daemon  # 以daemon方式在后台运行
  nbproc 1 # HAProxy启动时作为守护运行可创建的进程数，配合daemon参数使用，默认只启动一个进程，该值应小于cpu核数。
  maxconn 256  # 最大同时256连接
  pidfile /usr/local/etc/haproxy/haproxy.pid  # 指定保存HAProxy进程号的文件
  log 127.0.0.1 local0 info
  log 127.0.0.1 local1 warning
  description "HAProxy 使用说明......"

defaults
  # mode  tcp   
  # retries 3   # 超时重试次数
  # maxconn global
  # timeout connect 5s  # haproxy将客户端请求转发至后端服务器，连接server端超时5s
  # timeout client 30s  # 客户端响应超时30s
  # timeout server 30s  # server端响应超时30s
  # timeout check 3s   # 健康检查的超时时间,即心跳3s
  # log global
  # option tcplog

  mode http
  retries 2
  maxconn 64
  timeout http-request  3s  # 在客户端建立连接但不请求数据时，关闭客户端连接
  timeout connect 5s  # haproxy将客户端请求转发至后端服务器，连接server端超时5s
  timeout client 30s  # 客户端响应超时30s
  timeout server 30s  # server端响应超时30s
  timeout check 3s   # 健康检查的超时时间,即心跳3s
  option http-keep-alive
  log global
  option httplog

## springboot-docker 前端 VIP
frontend  spt-vip
  mode  http
  bind  0.0.0.0:8100  # 监听端口 8100
  default_backend spt-server  # 可使用 use_backend 指定
  option forwardfor except 127.0.0.1  # 记录客户端IP在X-Forwarded-For头域中

## springboot-docker 服务集群
backend spt-server
  balance roundrobin          # 负载策略
  option httpchk GET /healthz # 健康检查 URI
  http-send-name-header spt-server-name      # 将 server 的名字添加到请求头
  server spt-node1 127.0.0.1:8101 maxconn 40 check inter 10s weight 5 rise 2 fall 3 # 健康检查间隔和超时时间为 5s，两次成功视为节点UP，三次失败视为节点DOWN
  server spt-node2 127.0.0.1:8102 maxconn 12 check inter 30s weight 5
  server spt-node3 127.0.0.1:8103 maxconn 12 check inter 30s weight 1

## 统计web页面配置
listen  admin_stats                           # 统计web页面配置
        bind 0.0.0.0:8888                     
        mode http 
        maxconn 10  
        stats enable                          # 开启统计
        stats refresh 30s                     # 监控页面自动刷新时间
        stats uri /haproxy
        stats realm welcome login\ Haproxy Statistics    # 统计页面密码框提示文本
        stats auth  admin:admin               # 监控页面的用户和密码:admin, 可设置多个用户名
```

#### 2. TCP

...

### 八. 参考资料

**HaProxy**

- https://cbonte.github.io/haproxy-dconv/1.8/configuration.html
- http://www.ttlsa.com/linux/haproxy-study-tutorial
- https://blog.csdn.net/anningzhu/article/details/77725354

- https://www.jianshu.com/p/c9f6d55288c0
- https://www.jianshu.com/p/5d3cd36fe7c3

**高可用**

- https://www.zhihu.com/question/34822368
- https://www.jianshu.com/p/95cc6e875456
- https://www.yyblogs.net/article/78.html