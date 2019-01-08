## Redis 

### 一. 概述

​	Redis是一个开源（BSD许可），基于内存存储的数据结构服务器，可用作数据库，高速缓存和消息队列代理。它支持[字符串](http://www.redis.net.cn/tutorial/3508.html)、[哈希表](http://www.redis.net.cn/tutorial/3509.html)、[列表](http://www.redis.net.cn/tutorial/3510.html)、[集合](http://www.redis.net.cn/tutorial/3511.html)、[有序集合](http://www.redis.net.cn/tutorial/3512.html)，[位图](http://www.redis.net.cn/tutorial/3508.html)，[hyperloglogs](http://www.redis.net.cn/tutorial/3513.html)等数据类型。内置复制、[Lua脚本](http://www.redis.net.cn/tutorial/3516.html)、LRU收回、[事务](http://www.redis.net.cn/tutorial/3515.html)以及不同级别磁盘持久化功能，同时通过Redis Sentinel提供高可用，通过Redis Cluster提供自动[分区](http://www.redis.net.cn/tutorial/3524.html)。

### 二. 安装

#### 1. 下载Redis及相关工具

-  http://download.redis.io/releases/redis-3.2.5.tar.gz
-  https://jaist.dl.sourceforge.net/project/tcl/Tcl/8.6.7/tcl8.6.7-src.tar.gz

#### 2. 安装

- Tcl，用于测试redis

```shell
mv tcl8.6.7-src.tar.gz /usr/local/
cd /usr/local/
tar -zxvf tcl8.6.7-src.tar.gz
rm -rf tcl8.6.7-src.tar.gz
cd tcl8.6.7/unix/
sudo ./configure  
sudo make  
sudo make install
```

- Redis

```sh
mv redis-3.2.5.tar.gz /usr/local/
cd /usr/local/
tar -zxvf redis-3.2.5.tar.gz
rm -rf redis-3.2.5.tar.gz
cd redis-3.2.5
make
make test
make install
```

- 启动

```sh
# & 表示 Redis 在后台运行，开发环境
redis-server &

# 指定配置文件启动，生产环境
redis-server /<path>/redis.conf
```

### 三. 配置

#### 1. 详解

```ini
## 载入额外的配置
include /path/to/local.conf

## 绑定redis服务器网卡IP，默认为127.0.0.1,即本地回环地址。这样的话，访问redis服务只能通过本机的客户端连接，而无法通过远程连接。如果bind选项为空的话，那会接受所有来自于可用网络接口的连接。
bind 192.168.1.100 10.0.0.1
bind 127.0.0.1 ::1

## 保护模式，默认是开启状态，只允许本地客户端连接， 可以设置密码或添加bind来连接
protected-mode yes

## 监听端口号，默认为6379，如果设为0，redis将不在socket 上监听任何客户端连接
port 6379

## TCP监听的最大容纳数量，在高并发的环境下，你需要把这个值调高以避免客户端连接缓慢的问题。Linux 内核会把这个值缩小成 /proc/sys/net/core/somaxconn对应的值，要提升并发量需要修改这两个值才能达到目的。/proc/sys/net/core/somaxconn（系统中每一个端口最大的监听队列的长度） 的默认值为128（多小），可调节成 1024。
tcp-backlog 511

## 指定在一个 client 空闲多少秒之后关闭连接（0表示永不关闭）
timeout 0

## 单位是秒，表示将周期性的使用SO_KEEPALIVE检测客户端是否还处于健康状态，避免服务器一直阻塞，官方给出的建议值是300s，如果设置为0，则不会周期性的检测
tcp-keepalive 300

## 默认情况下 redis 不是作为守护进程运行的，如果你想让它在后台运行，你就把它改成 yes。当redis作为守护进程运行的时候，它会写一个 pid 到 /var/run/redis.pid 文件里面
daemonize no

## 可以通过upstart和systemd管理Redis守护进程
## 选项：
##    supervised no - 没有监督互动
##    supervised upstart - 通过将Redis置于SIGSTOP模式来启动信号
##    supervised systemd - signal systemd将READY = 1写入$ NOTIFY_SOCKET
##    supervised auto - 检测upstart或systemd方法基于 UPSTART_JOB或NOTIFY_SOCKET环境变量
supervised no

## 配置PID文件路径，当redis作为守护进程运行的时候，它会把 pid 默认写到 /var/redis/run/redis_6379.pid 文件里面
pidfile /var/run/redis_6379.pid

## 定义日志级别。
##  可以是下面的这些值：
##  debug（记录大量日志信息，适用于开发、测试阶段）
##  verbose（较多日志信息）
##  notice（适量日志信息，使用于生产环境）
##  warning（仅有部分重要、关键信息才会被记录）
loglevel notice

## 日志文件的位置，当指定为空字符串时，为标准输出，如果redis已守护进程模式运行，那么日志将会输出到/dev/null
logfile ""

## 要想把日志记录到系统日志，就把它改成 yes，也可以可选择性的更新其他的syslog 参数以达到你的要求
syslog-enabled no

## 设置系统日志的ID
syslog-ident redis

## 指定系统日志设置，必须是 USER 或者是 LOCAL0-LOCAL7 之间的值
syslog-facility local0

## 设置数据库的数目。默认的数据库是DB 0 ，可以在每个连接上使用select  <dbid> 命令选择一个不同的数据库，dbid是一个介于0到databases - 1 之间的数值。
databases 16

## 数据持久化 ##
## RDB方式的数据持久化，Redis会定期保存数据快照至一个rbd文件中，并在启动时自动加载rdb文件，恢复之前保存的数据
##	1. RDB方式的持久化几乎不损耗Redis本身的性能，在进行RDB持久化时，Redis主进程唯一需要做的事情就是fork出一个子进程，所有持久化工作都由子进程完成
##	2.Redis无论因为什么原因crash掉之后，重启时能够自动恢复到上一次RDB快照中记录的数据。这省去了手工从其他数据源（如DB）同步数据的过程，而且要比其他任何的数据恢复方式都要快
## 快照保存的时机 save [seconds] [changes]
save 900 1			## 900秒数据变更1次
save 300 10			## 300秒数据变更10次
save 60 10000		## 60秒数据变更10000次
save ""			    ## 停用保存功能（或注释所有的 save）

## 如果用户开启了RDB快照功能，那么在redis持久化数据到磁盘时如果出现失败，默认情况下，redis会停止接受所有的写请求
stop-writes-on-bgsave-error yes

## 对于存储到磁盘中的快照，可以设置是否进行压缩存储（LZF算法），会消耗 CPU
rdbcompression yes

## 在存储快照后，我们还可以让redis使用CRC64算法来进行数据校验，但是这样做会增加大约10%的性能消耗，如果希望获取到最大的性能提升，可以关闭此功能。
rdbchecksum yes

## 设置快照的文件名
dbfilename dump.rdb

## 设置快照文件的存放路径，这个配置项一定是个目录，而不能是文件名
dir ./

## 开启Append Only File是另一种持久化方式
appendonly no

## aof文件名
appendfilename "appendonly.aof"

## aof持久化策略的配置
##  no表示不执行fsync，由操作系统保证数据同步到磁盘，速度最快。
##  always表示每次写入都执行fsync，以保证数据同步到磁盘。
##  everysec表示每秒执行一次fsync，可能会导致丢失这1s数据（默认）
# appendfsync always
appendfsync everysec
# appendfsync no

## 在aof重写或者写入rdb文件的时候，会执行大量IO，此时对于everysec和always的aof模式来说，
##   执行fsync会造成阻塞过长时间，no-appendfsync-on-rewrite字段设置为默认设置为no。
##   如果对延迟要求很高的应用，这个字段可以设置为yes，否则还是设置为no，这样对持久化特性来说这是更安全的选择。
##   设置为yes表示rewrite期间对新写操作不fsync,暂时存在内存中,等rewrite完成后再写入，默认为no，建议yes。
##   Linux的默认fsync策略是30秒。可能丢失30秒数据。
no-appendfsync-on-rewrite no

##  aof自动重写配置，当目前aof文件大小超过上一次重写的aof文件大小的百分之多少进行重写，
##  即当aof文件增长到一定大小的时候，Redis能够调用bgrewriteaof对日志文件进行重写。
##  当前AOF文件大小是上次日志重写得到AOF文件大小的二倍（设置为100）时，自动启动新的日志重写过程。
auto-aof-rewrite-percentage 100

## 设置允许重写的最小aof文件大小，避免了达到约定百分比但尺寸仍然很小的情况还要重写
auto-aof-rewrite-min-size 64mb

## aof文件可能在尾部是不完整的，当redis启动的时候，aof文件的数据被载入内存。重启可能发生在redis所在的主机操作系统宕机后，尤其在ext4文件系统没有加上data=ordered选项，出现这种现象
## redis宕机或者异常终止不会造成尾部不完整现象，可以选择让redis退出，或者导入尽可能多的数据。
##  如果选择的是yes，当截断的aof文件被导入的时候，会自动发布一个log给客户端然后load。
##  如果是no，用户必须手动redis-check-aof修复AOF文件才可以。
aof-load-truncated yes

## 主从复制 ##
## 主从复制，使用 slaveof 来让一个 redis 实例成为另一个reids 实例的副本，默认关闭。
## 注意这个只需要在 slave 上配置
slaveof <masterip> <masterport>
## 如果 master 需要密码认证，就在这里设置，默认不设置
masterauth <master-password>

## 当一个 slave 与 master 失去联系，或者复制正在进行的时候，slave 可能会有两种表现：
##  1) 如果为 yes ，slave 仍然会应答客户端请求，但返回的数据可能是过时，或者数据可能是空的在第一次同步的时候 
##  2) 如果为 no ，在你执行除了 info he salveof 之外的其他命令时，slave 都将返回一个 "SYNC with master in progress" 的错误
slave-serve-stale-data yes

## 配置一个 slave 实体是否接受写入操作，默认为只读
slave-read-only yes

## 主从数据复制是否使用无硬盘复制功能。
##  新的从站和重连后不能继续备份的从站，需要做所谓的“完全备份”，即将一个RDB文件从主站传送到从站。
##  这个传送有以下两种方式：
##  1）硬盘备份：redis主站创建一个新的进程，用于把RDB文件写到硬盘上。过一会儿，其父进程递增地将文件传送给从站。
##  2）无硬盘备份：redis主站创建一个新的进程，子进程直接把RDB文件写到从站的套接字，不需要用到硬盘。
##  在硬盘备份的情况下，主站的子进程生成RDB文件。一旦生成，多个从站可以立即排成队列使用主站的RDB文件。
##  在无硬盘备份的情况下，一次RDB传送开始，新的从站到达后，需要等待现在的传送结束，才能开启新的传送。
##  如果使用无硬盘备份，主站会在开始传送之间等待一段时间（可配置，以秒为单位），希望等待多个子站到达后并行传送。
##  在硬盘低速而网络高速（高带宽）情况下，无硬盘备份更好。
repl-diskless-sync no

## 当启用无硬盘备份，服务器等待一段时间后才会通过套接字向从站传送RDB文件，这个等待时间是可配置的。这一点很重要，因为一旦传送开始，就不可能再为一个新到达的从站服务。从站则要排队等待下一次RDB传送。因此服务器等待一段时间以期更多的从站到达。默认为5秒。要关掉这一功能，只需将它设置为0秒，传送会立即启动。
repl-diskless-sync-delay 5

## 从redis会周期性的向主redis发出PING包，你可以通过repl_ping_slave_period指令来控制其周期，默认是10秒。
repl-ping-slave-period 10

## 备份的超时时间
## 1）从从站的角度，同步期间的批量传输的I/O
## 2）从站角度认为的主站超时（数据，ping）
## 3）主站角度认为的从站超时（REPLCONF ACK pings)
## 确认这些值比定义的repl-ping-slave-period要大，否则每次主站和从站之间通信低速时都会被检测为超时。
repl-timeout 60

## yes，redis会使用较少量的TCP包和带宽向从站发送数据。但这会导致在从站增加一点数据的延时。Linux内核默认配置情况下最多40毫秒的延时。
## no，从站的数据延时不会那么多，但备份需要的带宽相对较多。
repl-disable-tcp-nodelay no

## 主站有一段时间没有与从站连接，对应的工作储备就会自动释放。这个选项用于配置释放前等待的秒数，秒数从断开的那一刻开始计算，值为0表示不释放。
repl-backlog-ttl 3600

## 从站优先级是可以从redis的INFO命令输出中查到的一个整数。当主站不能正常工作时，redis sentinel使用它来选择一个从站并将它提升为主站。低优先级的从站被认为更适合于提升，因此如果有三个从站优先级分别是10， 100，25，sentinel会选择优先级为10的从站，因为它的优先级最低。然而优先级值为0的从站不能执行主站的角色，因此优先级为0的从站永远不会被redis sentinel提升。默认优先级是100。
slave-priority 100

## 主站可以停止接受写请求，当与它连接的从站少于N个，滞后少于M秒，N个从站必须是在线状态。
min-slaves-to-write 3
min-slaves-max-lag 10

## 安全性 ##
## 设置redis连接密码
requirepass foobared

## 将命令重命名，为了安全考虑，可以将某些重要的、危险的命令重命名。当你把某个命令重命名成空字符串的时候就等于取消了这个命令。
rename-command CONFIG "
rename-command CONFIG b840fc02d524045429941cc15f59e41cb7be6c52

##  设置客户端最大并发连接数，默认无限制，Redis可以同时打开的客户端连接数为Redis进程可以打开的最大文件描述符数-32（redis server自身会使用一些），如果设置 maxclients为 0。 表示不作限制。当客户端连接数到达限制时，Redis会关闭新的连接并向客户端返回max number of clients reached错误信息
maxclients 10000

## 指定Redis最大内存限制，Redis在启动时会把数据加载到内存中，达到最大内存后，Redis会先尝试清除已到期或即将到期的Key。当此方法处理后，仍然到达最大内存设置，将无法再进行写入操作，但仍然可以进行读取操作。Redis新的vm机制，会把Key存放内存，Value会存放在swap区，格式：maxmemory <bytes>
## 在32位OS中，Redis最大使用3GB的内存，在64位OS中则没有限制。
maxmemory 100mb

## redis内存清楚策略，默认是noeviction，推荐使用的策略是volatile-lru
## volatile-lru：使用LRU算法进行数据淘汰（淘汰上次使用时间最早的，且使用次数最少的key），只淘汰设定了有效期的key
## allkeys-lru：使用LRU算法进行数据淘汰，所有的key都可以被淘汰
## volatile-random：随机淘汰数据，只淘汰设定了有效期的key
## allkeys-random：随机淘汰数据，所有的key都可以被淘汰
## volatile-ttl：淘汰剩余有效期最短的key
## noeviction：不移除任何key，只是返回一个写错误
maxmemory-policy noeviction

## 安全性 ##
## 设置登录密码，连接方式
## 1. redis-cli -h <ip> -p <port> -a <password>
## 2. redis-cli -h <ip> -p <port>，然后 auth <password>
requirepass 123456

## 重命名危险命令
rename-command CONFIG ""
```

### 四. 哨兵模式（Sentinel）

#### 1. 介绍

​	Redis-Sentinel是Redis官方推荐的高可用性(HA)解决方案，当用Redis做Master-slave的高可用方案时，假如master宕机了，Redis本身(包括它的很多客户端)都没有实现自动进行主备切换，而Redis-sentinel本身也是一个独立运行的进程，它能监控多个master-slave集群，发现master宕机后能进行自动切换。

- 不时地监控redis是否按照预期良好地运行;
- 如果发现某个redis节点运行出现状况，能够通知另外一个进程(例如它的客户端);
- 能够进行自动切换。当一个master节点不可用时，能够选举出master的多个slave(如果有超过一个slave的话)中的一个来作为新的master,其它的slave节点会将它所追随的master的地址改为被提升为master的slave的新地址。
- 自支持集群，只使用单个sentinel进程来监控redis集群是不可靠的，当sentinel进程宕掉后(sentinel本身也有单点问题，single-point-of-failure)整个集群系统将无法按照预期的方式运行。

#### 2. 搭建

服务器：10.71.202.121（3个redis数据Node，1个Master，2个Slave）、10.71.202.122（3个sentinel节点）

1. 安装redis，参考章节二。

2. 数据Node，10.71.202.121

   - 复制准备Master和Slave的配置文件

   ```powershell
   cd /opt/
   mkdir redis-conf
   cp /opt/redis-3.2.5/redis.conf master.conf
   cp /opt/redis-3.2.5/redis.conf slave1.conf
   cp /opt/redis-3.2.5/redis.conf slave2.conf
   ```

   - Master，只列修改的配置，其余均默认

   ```powershell
   # bind 127.0.0.1
   protected-mode no
   daemonize yes
   logfile "/opt/redis-conf/redis_6379.log"
   dir "/opt/redis-conf/6379/"
   masterauth "iamdate"
   requirepass "iamdante"
   maxmemory 512mb
   maxmemory-policy volatile-lru
   ```

   - Slave，slave1的端口为6380，slave2的端口为6381

     **Slave1**

   ```powershell
   # bind 127.0.0.1
   protected-mode no
   port 6380
   daemonize yes
   pidfile /var/run/redis_6380.pid
   logfile "/opt/redis-conf/redis_6380.log"
   dir "/opt/redis-conf/6380/"
   slaveof 10.71.202.121 6379
   masterauth "iamdate"
   requirepass "iamdante"
   maxmemory 512mb
   maxmemory-policy volatile-lru
   ```

   ​      **Slave2**

   ```powershell
   # bind 127.0.0.1
   protected-mode no
   port 6380
   daemonize yes
   pidfile /var/run/redis_6381.pid
   logfile "/opt/redis-conf/redis_6381.log"
   dir "/opt/redis-conf/6381/"
   slaveof 10.71.202.121 6379
   masterauth "iamdate"
   requirepass "iamdante"
   maxmemory 512mb
   maxmemory-policy volatile-lru
   ```

3. 启动服务

   ```sh
   redis-server /opt/redis-conf/master.conf 
   redis-server /opt/redis-conf/slave1.conf 
   redis-server /opt/redis-conf/slave2.conf 
   
   ## 开放防火墙端口
   firewall-cmd --zone=public --add-port=6379/tcp --permanent
   firewall-cmd --zone=public --add-port=6380/tcp --permanent
   firewall-cmd --zone=public --add-port=6381/tcp --permanent
   firewall-cmd --reload
   ```

4. 原理

   ① Slave向Master发送sync命令。

   ② Master接收sync命令后，执行BGSAVE命令（保存快照），创建一个RDB文件，在创建RDB文件期间的命令将保存在缓冲区中。

   ③ 当Master执行完BGSAVE时，会向Slave发送RDB文件，而Slave会接收并载入该文件。

   ④ Master将缓冲区的所有写命令发给Slave执行。

   ⑤ 以上处理完之后，之后Master每执行一个写命令，都会将被执行的写命令发送给Slave。

   Redis目前的复制是异步的，只保证最终一致性，而不是强一致性（主从数据库的更新还是分先后，先主后从）。要是一致性要求高的应用，目前还是读写都在主库上去。

5. Sentinel Node

   - 复制准备配置文件

   ```powershell
   cd /opt/
   mkdir redis-conf
   cp /opt/redis-3.2.5/sentinel.conf sentinel_1.conf
   cp /opt/redis-3.2.5/sentinel.conf sentinel_2.conf
   cp /opt/redis-3.2.5/sentinel.conf sentinel_3.conf
   ```

   - 配置，端口26379、26381、26381

   ```powershell
   port 26379
   daemonize yes
   protected-mode no
   
   dir "/opt/redis-conf/26379/"
   logfile "/opt/redis-conf/sentinel_26379.log"
   
   ## sentinel <option_name> <master_name> <option_value>；#该行的意思是：监控的master的名字叫做T1（自定义）,地址为127.0.0.1:10086，行尾最后的一个2代表在sentinel集群中，多少个sentinel认为masters死了，才能真正认为该master不可用了。
   sentinel monitor spiritmaster 10.71.202.121 6379 2
   
   ## sentinel 连接设置了密码的主和从
   sentinel auth-pass spiritmaster iamdante
   
   ## sentinel会向master发送心跳PING来确认master是否存活，如果master在“一定时间范围”内不回应PONG 或者是回复了一个错误消息，那么这个sentinel会主观地(单方面地)认为这个master已经不可用了(subjectively down, 也简称为SDOWN)。而这个down-after-milliseconds就是用来指定这个“一定时间范围”的，单位是毫秒，默认30秒。
   sentinel down-after-milliseconds spiritmaster 10000
   
   ## failover过期时间，当failover开始后，在此时间内仍然没有触发任何failover操作，当前sentinel将会认为此次failoer失败。默认180秒，即3分钟。
   sentinel failover-timeout spiritmaster 120000
   
   ## 在发生failover主备切换时，这个选项指定了最多可以有多少个slave同时对新的master进行同步，这个数字越小，完成failover所需的时间就越长，但是如果这个数字越大，就意味着越多的slave因为replication而不可用。可以通过将这个值设为 1 来保证每次只有一个slave处于不能处理命令请求的状态。
   sentinel parallel-syncs spiritmaster 1
   
   ## 发生切换之后执行的一个自定义脚本：如发邮件、vip切换等
   # sentinel notification-script <master-name> <script-path>     ##不会执行，疑问？
   # sentinel client-reconfig-script <master-name> <script-path>  ##这个会执行
   ```

   - 启动

   ```shell
   redis-sentinel /opt/redis-conf/sentinel_1.conf
   redis-sentinel /opt/redis-conf/sentinel_2.conf
   redis-sentinel /opt/redis-conf/sentinel_3.conf
   
   ## 开放防火墙端口
   firewall-cmd --zone=public --add-port=26379/tcp --permanent
   firewall-cmd --zone=public --add-port=26380/tcp --permanent
   firewall-cmd --zone=public --add-port=26381/tcp --permanent
   firewall-cmd --reload
   ```

   - 原理

   ```markdown
   SDOWN 和 ODOWN.
   SDOWN: subjectively down,直接翻译的为"主观"失效,即当前sentinel实例认为某个redis服务为"不可用"状态.
   ODOWN: objectively down,直接翻译为"客观"失效,即多个sentinel实例都认为master处于"SDOWN"状态,那么此时master将处于ODOWN, ODOWN可以简单理解为master已经被集群确定为"不可用",将会开启failover.
       SDOWN适合于master和slave,但是ODOWN只会使用于master;当slave失效超过"down-after-milliseconds"后,那么所有sentinel实例都会将其标记为"SDOWN".
   
       1) SDOWN与ODOWN转换过程:
   	每个sentinel实例在启动后,都会和已知的slaves/master以及其他sentinels建立TCP连接,并周期性发送PING(默认为1秒)
   	在交互中,如果redis-server无法在"down-after-milliseconds"时间内响应或者响应错误信息,都会被认为此redis-server处于SDOWN状态.
   	2) SDOWN的server为master,那么此时sentinel实例将会向其他sentinel间歇性(一秒)发送"is-master-down-by-addr <ip> <port>"指令并获取响应信息,如果足够多的sentinel实例检测到master处于SDOWN,那么此时当前sentinel实例标记master为ODOWN...其他sentinel实例做同样的交互操作。配置项"sentinel monitor <mastername> <masterip> <masterport> <quorum>",如果检测到master处于SDOWN状态的slave个数达到<quorum>, 那么此时此sentinel实例将会认为master处于ODOWN.
   每个sentinel实例将会间歇性(10秒)向master和slaves发送"INFO"指令,如果master失效且没有新master选出时,每1秒发送一次"INFO";"INFO"的主要目的就是获取并确认当前集群环境中slaves和master的存活情况.
   经过上述过程后,所有的sentinel对master失效达成一致后,开始failover.
       3) Sentinel与slaves"自动发现"机制:
       在sentinel的配置文件中(local-sentinel.conf),都指定了port,此port就是sentinel实例侦听其他sentinel实例建立链接的端口.在集群稳定后,最终会每个sentinel实例之间都会建立一个tcp链接,此链接中发送"PING"以及类似于"is-master-down-by-addr"指令集,可用用来检测其他sentinel实例的有效性以及"ODOWN"和"failover"过程中信息的交互.
       在sentinel之间建立连接之前,sentinel将会尽力和配置文件中指定的master建立连接.sentinel与master的连接中的通信主要是基于pub/sub来发布和接收信息,发布的信息内容包括当前sentinel实例的侦听端口:
   ```

   ​

   - 最后，可通过命令 **info replication** 查看Node信息。

```shell
10.71.202.121:6380> info replication
# Replication
role:slave
master_host:10.71.202.121
master_port:6381
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
slave_repl_offset:557960
slave_priority:100
slave_read_only:1
connected_slaves:0
master_repl_offset:0
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0

10.71.202.121:6381> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=10.71.202.121,port=6380,state=online,offset=562628,lag=1
slave1:ip=10.71.202.121,port=6379,state=online,offset=562628,lag=0
master_repl_offset:562628
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2
repl_backlog_histlen:562627
```

### 五. 集群模式（Cluster）

#### 1. 原理

​	Redis Cluster是一个高性能高可用的分布式系统。由多个Redis实例组成的整体，数据按照Slot存储分布在多个Redis实例上，通过 [Gossip协议](http://www.jianshu.com/p/133560ef28df) 来进行Node之间通信。

 -  背景

    - Redis中存储的数据量大，一台主机的物理内存已经无法容纳。
    - Redis的写请求并发量大，一个Redis实例以无法承载。

-  功能

  - 数据按照Slot存储分布在多个Redis实例上，通过  **key hash tags** 确保特定的 Keys 存储在同一个 Node。

  - 集群只能使用 database 0。

  - Node通信

    - 集群Node通过 **Redis Cluster Bus** (TCP bus、Gossip protocol) 进行通信。


    - Client与Server的通信没有变化，同单机Redis。

  - 当访问的key不在当前分片上时，能够自动将请求转发至正确的分片。

    ```markdown
    在redis的每一个Node上，都有这么两个东西
    1、一个是插槽（slot）可以理解为是一个可以存储两个数值的一个变量，这个变量的取值范围是：0-16383。2、一个就是cluster，是一个集群管理的插件。
    当我们的存取的key到达的时候，redis会根据crc16的算法得出一个结果，然后把结果对 16384 求余数，这样每个 key 都会对应一个编号在 0-16383 之间的Hash Slot，通过这个值，去找到对应的插槽所对应的Node，然后直接自动跳转到这个对应的Node上进行存取操作。
    公式：HASH_SLOT = CRC16(key) mod 16384
    ```

    ![redis_hash_slot](/Users/dante/Documents/Technique/且行且记/Redis/redis_hash_slot.png)

- 请求重定向

  ​	由于每个Node只负责部分slot，以及slot可能从一个Node迁移到另一Node，造成客户端有可能会向错误的Node发起请求。因此需要有一种机制来对其进行发现和修正，这就是请求重定向。

  - MOVED错误
    1. 请求的key对应的Slot不在该Node上，Node将查看自身内部所保存的Hash Slot到Node ID的映射记录，Node回复一个MOVED错误。
    2. 需要客户端进行再次重试。
  - ASK错误
    1. 请求的key对应的Slot目前的状态属于MIGRATING状态，并且当前Node找不到这个key了，Node回复ASK错误。ASK会把对应Slot的IMPORTING节点返回给你，告诉你去IMPORTING的Node尝试找找。	
    2. 客户端进行重试首先发送ASKING命令，Node将为客户端设置一个一次性的标志（flag），使得客户端可以执行一次针对IMPORTING状态的Slot的命令请求，然后再发送真正的命令请求。
    3. 不必更新客户端所记录的Slot至Node的映射。

- 投票机制

  ​	集群中的每个Node都会定期地向集群中的其他Node发送PING消息，以此交换各个Node状态信息，检测各个Node状态：在线状态、疑似下线状态PFAIL、已下线状态FAIL。

  - 有一半以上的节点去ping一个节点的时候没有回应，集群就认为这个节点宕机了，然后去连接它的备用节点。
  - 某个节点和所有Slava Node全部挂掉，集群就进入faill状态。
  - 有一半以上的Master Node宕机，集群就进入faill状态。

![Slave选择Master](/Users/dante/Documents/Technique/且行且记/Redis/Slave选择Master.png)

- 故障转移

  当从节点发现自己的主节点变为已下线(FAIL)状态时，便尝试进Failover，以期成为新的主。执行步骤如下：

  - 1) 从下线主节点的所有从节点中选中一个从节点
  - 2) 被选中的从节点执行SLAVEOF NO NOE命令，成为新的主节点
  - 3) 新的主节点会撤销所有对已下线主节点的槽指派，并将这些槽全部指派给自己
  - 4) 新的主节点对集群进行广播PONG消息，告知其他节点已经成为新的主节点
  - 5) 新的主节点开始接收和处理槽相关的请求

#### 2. 配置

```ini
## redis默认是单体实例，配置yes则开启集群功能
cluster-enabled yes

## 集群配置文件, 此配置文件不能人工编辑，它是集群Node自动维护的文件，主要用于记录集群中有哪些Node、他们的状态以及一些持久化参数等，方便在重启时恢复这些状态。通常是在收到请求之后这个文件就会被更新。
cluster-config-file nodes-6379.conf

## 集群中的Node失联的最大时间，超过这个时间，该Node就会被认为故障。如果主Node超过这个时间还是不可达，则用它的从Node将启动故障迁移，升级成主Node。注意，任何一个Node在这个时间之内如果还是没有连上大部分的主Node，则此Node将停止接收任何请求。一般设置为15秒即可
cluster-node-timeout 15000

## 设置成０，则无论从Node与主Node失联多久，从Node都会尝试升级成主Node
## 设置成正数, 则此从Node数据有效的最长时间超过 cluster-node-timeout * cluster-slave-validity-factor，从Node不会启动故障迁移。
cluster-slave-validity-factor 10

## 副本迁移，主Node需要的最小从Node数，只有达到这个数，主Node失败时，它从Node才会进行迁移。
cluster-migration-barrier 1

## 在部分key所在的Node不可用时，如果此参数设置为"yes"(默认值), 则整个集群停止接受操作；如果此参数设置为”no”，则集群依然为可达Node上的key提供读操作。
cluster-require-full-coverage yes
```

#### 3. 搭建

创建6个redis节点，其中三个为主Node，三个为从Node，ip和端口对应关系

```shell
127.0.0.1:7000
127.0.0.1:7001
127.0.0.1:7002
127.0.0.1:7003
127.0.0.1:7004
127.0.0.1:7005
```

- 创建目录

  ```powershell
  mkdir cluster
  cd cluster
  mkdir 7000 7001 7002 7003 7004 7005
  ```

- 修改配置

  ```powershell
  cp /Users/dante/Documents/Technique/Redis/redis-3.2.5/redis.conf ./
  vim redis.conf

  ## 主要配置, 修改各个实例的端口
  port 7000
  daemonize yes
  pidfile /Users/dante/Documents/Technique/Redis/cluster/7000/redis_7000.pid
  logfile "/Users/dante/Documents/Technique/Redis/cluster/7000/7000.log"
  cluster-enabled yes
  cluster-config-file nodes_7000.conf
  cluster-node-timeout 5000
  appendonly yes

  ## 修改后，分别复制到各个端口文件夹中
  cp redis.conf 7000/ 7001/ 7002/ 7003/ 7004/ 7005/
  ```

- 启动

  ```powershell
  redis-server 7000/redis.conf 
  redis-server 7001/redis.conf 
  redis-server 7002/redis.conf 
  redis-server 7003/redis.conf 
  redis-server 7004/redis.conf 
  redis-server 7005/redis.conf

  ## 启动日志，9c51289598cc09025fb00b1f2be22b1bb11c3e0e 称为 Node ID, 唯一不变。
  42713:M 29 Oct 16:39:49.286 * No cluster configuration found, I'm 9c51289598cc09025fb00b1f2be22b1bb11c3e0e
  ```


- 创建集群

  ```powershell
  ## 通过  redis-trib 工具创建集群, redis-trib 在 redis/src 下
  ## Install redis gem to be able to run redis-trib.
  gem install redis

  ## 创建命令
  cd /Users/dante/Documents/Technique/Redis/redis-3.2.5/src
  ./redis-trib.rb create --replicas 1 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005

  ## 日志如下
  >>> Creating cluster
  >>> Performing hash slots allocation on 6 nodes...
  Using 3 masters:
  127.0.0.1:7000
  127.0.0.1:7001
  127.0.0.1:7002
  Adding replica 127.0.0.1:7003 to 127.0.0.1:7000
  Adding replica 127.0.0.1:7004 to 127.0.0.1:7001
  Adding replica 127.0.0.1:7005 to 127.0.0.1:7002
  M: 9c51289598cc09025fb00b1f2be22b1bb11c3e0e 127.0.0.1:7000
     slots:0-5460 (5461 slots) master
  M: 1bcd5d934db936ba0757ce2df19aa2c9d1fe73be 127.0.0.1:7001
     slots:5461-10922 (5462 slots) master
  M: 54696794d478d5c9e29f723e0800eb0656f50c15 127.0.0.1:7002
     slots:10923-16383 (5461 slots) master
  S: 2a91650db59f38f921dac36067d438a657909a9a 127.0.0.1:7003
     replicates 9c51289598cc09025fb00b1f2be22b1bb11c3e0e
  S: 8c37e225f252be748fb1055ab54f73d76e5e8194 127.0.0.1:7004
     replicates 1bcd5d934db936ba0757ce2df19aa2c9d1fe73be
  S: 660ac3879f3288826d9b8b33d1eb6fd045720cd2 127.0.0.1:7005
     replicates 54696794d478d5c9e29f723e0800eb0656f50c15
  Can I set the above configuration? (type 'yes' to accept): yes
  >>> Nodes configuration updated
  >>> Assign a different config epoch to each node
  >>> Sending CLUSTER MEET messages to join the cluster
  Waiting for the cluster to join...
  >>> Performing Cluster Check (using node 127.0.0.1:7000)
  M: 9c51289598cc09025fb00b1f2be22b1bb11c3e0e 127.0.0.1:7000
     slots:0-5460 (5461 slots) master
     1 additional replica(s)
  M: 54696794d478d5c9e29f723e0800eb0656f50c15 127.0.0.1:7002
     slots:10923-16383 (5461 slots) master
     1 additional replica(s)
  M: 1bcd5d934db936ba0757ce2df19aa2c9d1fe73be 127.0.0.1:7001
     slots:5461-10922 (5462 slots) master
     1 additional replica(s)
  S: 660ac3879f3288826d9b8b33d1eb6fd045720cd2 127.0.0.1:7005
     slots: (0 slots) slave
     replicates 54696794d478d5c9e29f723e0800eb0656f50c15
  S: 2a91650db59f38f921dac36067d438a657909a9a 127.0.0.1:7003
     slots: (0 slots) slave
     replicates 9c51289598cc09025fb00b1f2be22b1bb11c3e0e
  S: 8c37e225f252be748fb1055ab54f73d76e5e8194 127.0.0.1:7004
     slots: (0 slots) slave
     replicates 1bcd5d934db936ba0757ce2df19aa2c9d1fe73be
  [OK] All nodes agree about slots configuration.
  >>> Check for open slots...
  >>> Check slots coverage...
  [OK] All 16384 slots covered.
  ```

- 集群脚本

  在Redis的 utils/create-cluster 目录下，可以通过脚本操作Redis Cluster，修改脚本 create-cluster，默认脚本如下，操作命令

  - **./create-cluster start**
  - **./create-cluster create**
  - **./create-cluster stop**

  ```sh
  #!/bin/bash

  # Settings
  PORT=30000
  TIMEOUT=2000
  NODES=6
  REPLICAS=1

  # You may want to put the above config parameters into config.sh in order to
  # override the defaults without modifying this script.

  if [ -a config.sh ]
  then
      source "config.sh"
  fi

  # Computed vars
  ENDPORT=$((PORT+NODES))

  if [ "$1" == "start" ]
  then
      while [ $((PORT < ENDPORT)) != "0" ]; do
          PORT=$((PORT+1))
          echo "Starting $PORT"
          ../../src/redis-server --port $PORT --cluster-enabled yes --cluster-config-file nodes-${PORT}.conf --cluster-node-timeout $TIMEOUT --appendonly yes --appendfilename appendonly-${PORT}.aof --dbfilename dump-${PORT}.rdb --logfile ${PORT}.log --daemonize yes
      done
      exit 0
  fi

  if [ "$1" == "create" ]
  then
      HOSTS=""
      while [ $((PORT < ENDPORT)) != "0" ]; do
          PORT=$((PORT+1))
          HOSTS="$HOSTS 127.0.0.1:$PORT"
      done
      ../../src/redis-trib.rb create --replicas $REPLICAS $HOSTS
      exit 0
  fi

  if [ "$1" == "stop" ]
  then
      while [ $((PORT < ENDPORT)) != "0" ]; do
          PORT=$((PORT+1))
          echo "Stopping $PORT"
          ../../src/redis-cli -p $PORT shutdown nosave
      done
      exit 0
  fi

  if [ "$1" == "watch" ]
  then
      PORT=$((PORT+1))
      while [ 1 ]; do
          clear
          date
          ../../src/redis-cli -p $PORT cluster nodes | head -30
          sleep 1
      done
      exit 0
  fi

  if [ "$1" == "tail" ]
  then
      INSTANCE=$2
      PORT=$((PORT+INSTANCE))
      tail -f ${PORT}.log
      exit 0
  fi

  if [ "$1" == "call" ]
  then
      while [ $((PORT < ENDPORT)) != "0" ]; do
          PORT=$((PORT+1))
          ../../src/redis-cli -p $PORT $2 $3 $4 $5 $6 $7 $8 $9
      done
      exit 0
  fi

  if [ "$1" == "clean" ]
  then
      rm -rf *.log
      rm -rf appendonly*.aof
      rm -rf dump*.rdb
      rm -rf nodes*.conf
      exit 0
  fi

  echo "Usage: $0 [start|create|stop|watch|tail|clean]"
  echo "start       -- Launch Redis Cluster instances."
  echo "create      -- Create a cluster using redis-trib create."
  echo "stop        -- Stop Redis Cluster instances."
  echo "watch       -- Show CLUSTER NODES output (first 30 lines) of first node."
  echo "tail <id>   -- Run tail -f of instance at base port + ID."
  echo "clean       -- Remove all instances data, logs, configs."
  ```

  ​

### 六. 数据持久化

#### 1. RDB

​	RDB方式（SNAPSHOTTING）的数据持久化，Redis会定期保存数据快照至一个rbd文件中，并在启动时自动加载rdb文件，恢复之前保存的数据。

- **RDB的优点**

  - 对性能影响最小。如前文所述，Redis在保存RDB快照时会fork出子进程进行，几乎不影响Redis处理客户端请求的效率。

  - 每次快照会生成一个完整的数据快照文件，所以可以辅以其他手段保存多个时间点的快照（例如把每天0点的快照备份至其他存储媒介中），作为非常可靠的灾难恢复手段。

    使用RDB文件进行数据恢复比使用AOF要快很多。

- **RDB的缺点**

  - 快照是定期生成的，所以在Redis crash时或多或少会丢失一部分数据。
  - 如果数据集非常大且CPU不够强（比如单核CPU），Redis在fork子进程时可能会消耗相对较长的时间（长至1秒），影响这期间的客户端请求。

#### 2. AOF

​	采用AOF（APPEND ONLY MODE）持久方式时，Redis会把每一个写请求都记录在一个日志文件里。在Redis重启时，会把AOF文件中记录的所有写操作顺序执行一遍，确保数据恢复到最新。AOF默认是关闭的，如要开启

```ini
appendonly yes
```

AOF提供了三种fsync配置，always/everysec/no，通过配置项[appendfsync]指定：

- appendfsync no：不进行fsync，将flush文件的时机交给OS决定，速度最快
- appendfsync always：每写入一条日志就进行一次fsync操作，数据安全性最高，但速度最慢
- appendfsync everysec：折中的做法，交由后台线程每秒fsync一次

随着AOF不断地记录写操作日志，必定会出现一些无用的日志，例如某个时间点执行了命令**SET key1 "abc"**，在之后某个时间点又执行了**SET key1 "bcd"**，那么第一条命令很显然是没有用的。大量的无用日志会让AOF文件过大，也会让数据恢复的时间过长。
所以Redis提供了AOF rewrite功能，可以重写AOF文件，只保留能够把数据恢复到最新状态的最小写操作集。
AOF rewrite可以通过**BGREWRITEAOF**命令触发，也可以配置Redis定期自动进行：

```
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

上面两行配置的含义是，Redis在每次AOF rewrite时，会记录完成rewrite后的AOF日志大小，当AOF日志大小在该基础上增长了100%后，自动进行AOF rewrite。同时如果增长的大小没有达到64mb，则不会进行rewrite。

- **AOF的优点**
  - 最安全，在启用appendfsync always时，任何已写入的数据都不会丢失，使用在启用appendfsync everysec也至多只会丢失1秒的数据。
  - AOF文件在发生断电等问题时也不会损坏，即使出现了某条日志只写入了一半的情况，也可以使用redis-check-aof工具轻松修复。
  - AOF文件易读，可修改，在进行了某些错误的数据清除操作后，只要AOF文件没有rewrite，就可以把AOF文件备份出来，把错误的命令删除，然后恢复数据。
- **AOF的缺点**
  - AOF文件通常比RDB文件更大
  - 性能消耗比RDB高
  - 数据恢复速度比RDB慢

### 七. Springboot集成

​	具体源码实现见 ` /Users/dante/Documents/Project/spring/springboot-data-redis`。

```yaml
---
## 单机模式
spring:
  redis:
    host: 10.71.202.121
    port: 6379
    password: iamdante
    pool:
      min-idle: 10
      max-idle: 50
      max-active: 200
      max-wait: 3000 
      
--- 
## Sentinel 模式
spring:
  redis:
    password: iamdante
    pool:
      min-idle: 10
      max-idle: 50
      max-active: 200
      max-wait: 3000  
    sentinel:
      master: spiritmaster
      nodes: 10.71.202.122:26379,10.71.202.122:26380,10.71.202.122:26381 ## 哨兵的配置列表
      
---
## Cluster 模式
spring:
  redis:
    host: 127.0.0.1
    pool:
      min-idle: 10
      max-idle: 50
      max-active: 200
      max-wait: 3000
    cluster:
      nodes:
      - 127.0.0.1:7000
      - 127.0.0.1:7001
      - 127.0.0.1:7002
      - 127.0.0.1:7003
      - 127.0.0.1:7004
      - 127.0.0.1:7005
      max-redirects: 3 # Node调转次数（实际可以看做是重试次数）
```

### 八. 参考资料

- http://redis.io/
- https://redis.io/topics/sentinel
- https://redis.io/topics/cluster-tutorial


- http://www.jianshu.com/p/2f14bc570563?from=jiantop.com
- http://blog.csdn.net/donggang1992/article/details/50981954
- http://blog.csdn.net/a491857321/article/details/51954783
- http://www.cnblogs.com/pqchao/p/6558688.html
- http://www.cnblogs.com/zhoujinyi/p/5570024.html
- http://diannaowa.blog.51cto.com/3219919/1557617
- http://www.redis.cn/topics/cluster-tutorial.html
- http://blog.csdn.net/cuiy6642/article/details/50924722
- http://www.jianshu.com/p/0232236688c1

