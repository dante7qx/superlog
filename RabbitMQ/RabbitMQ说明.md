## RabbitMQ 说明

### 一. 概念

​	MQ，即Message Queue，一种应用程序对应用程序的通信方法。JMS（Java Message Service）Java消息中间件服务的标准和API定义，比较脆弱。AMQP（Advanced Message Queuing Protocol）高级消息队列协议。**RabbitMQ**是一个在 AMQP 基础上使用 Erlang 语言完成的，可复用的企业消息系统。

### 二. 安装

以下以 Centos7 为例

#### 1. 单例

- 安装Erlang（http://erlang.org/download/）

  - 下载：http://erlang.org/download/otp_src_20.2.tar.gz

  - 安装

    ```shell
    cd otp_src_20.2

    ./configure --prefix=/usr/local/erlang --with-ssl --enable-threads --enable-smmp-support --enable-kernel-poll --enable-hipe --without-javac

    ## 参数说明
    # --prefix 指定安装目录
    # --with-ssl  支持加密通信ssl
    # --enable-threads  启用异步线程支持
    # --enable-smmp-support 启用对称多处理支持（Symmetric Multi-Processing对称多处理结构的简称）
    # --enable-kernel-poll   启用Linux内核poll
    # --enable-hipe   启用高性能Erlang
    # --without-javac

    ## 编译、安装
    make && make install

    ## 添加环境变量
    vim /etc/profile
    ERLANG_HOME=/usr/local/erlang
    PATH=$ERLANG_HOME/bin:$PATH
    export ERLANG_HOME
    export PATH

    source /etc/profile
    ## 测试
    erl
    ```

- 安装 RabbitMQ Server（https://github.com/rabbitmq/rabbitmq-server/releases）

  - 下载：https://www.rabbitmq.com/releases/rabbitmq-server/v3.6.14/rabbitmq-server-generic-unix-3.6.14.tar.xz

  - 安装

    ```shell
    tar -xvf rabbitmq-server-generic-unix-3.6.14.tar.xz 
    cd rabbitmq_server-3.6.14/

    ## 添加环境变量
    vim /etc/profile
    RABBITMQ_HOME=/opt/rabbitmq_server-3.6.14
    PATH=$RABBITMQ_HOME/sbin:$PATH
    export PATH

    ## Web管理插件
    rabbitmq-plugins enable rabbitmq_management
    ## 启动
    rabbitmq-server -detached
    ## 关闭
    rabbitmqctl stop

    ## 管理用户
    rabbitmqctl delete_user guest
    rabbitmqctl add_user admin admin123
    rabbitmqctl set_user_tags  admin administrator 
    ## 访问 http://localhost:15672   admin/admin123

    ## 开启防火墙端口 5672、15672
    firewall-cmd --zone=public --add-port=5672/tcp --permanent
    firewall-cmd --zone=public --add-port=15672/tcp --permanent
    firewall-cmd --reload
    ```

#### 2. 普通集群

​	参考：https://www.rabbitmq.com/clustering.html

1. 配置 

   默认在 `$RABBITMQ_HOME/etc/` 下，修改 `$RABBITMQ_HOME/sbin/rabbitmq-defaults`

   ```ini
   SYS_PREFIX=${RABBITMQ_HOME}
   ## 修改为，则配置文件在 /etc/rabbitmq/ 目录下
   SYS_PREFIX=
   ```

   - rabbitmq-env.conf 环境变量

     ​	在外部环境变量的，并且是以`RABBITMQ_`开头的环境变量名，在这个文件里就对应为去掉这个前缀的环境变量名。例如，在命令行里，如果有个外部环境变量，名为`RABBITMQ_NODENAME=xxx`，就对应这个文件的变量名`NODENAME=xxx`。优先级高于 rabbitmq.config 。

   - rabbitmq.config 属性配置文件，例：

     ```erlang
     [
     {rabbit, [{tcp_listeners, [5673]}]},
     {rabbitmq_management, [{listener, [{port, 15673}]}]}
     ].
     ```

2. 设置Erlang Cookie

   ​	RabbitMQ的Node之间是通过Erlang Cookie（具有相同的cookie）来互相通信的。Erlang VM在RabbitMQ服务启动后会自动创建，位于 **$HOME/.erlang.cookie**。

3. 创建集群

   - 配置式 https://www.rabbitmq.com/configure.html

   ​

   - 命令行 rabbitmqctl

   ```shell
   ## 启动三个节点
   ./rabbitmq1/sbin/rabbitmq-server -detached
   ./rabbitmq2/sbin/rabbitmq-server -detached
   ./rabbitmq3/sbin/rabbitmq-server -detached

   ## 通过 cluster_status 验证
   ./rabbitmq1/sbin/rabbitmqctl cluster_status
   ./rabbitmq2/sbin/rabbitmqctl cluster_status
   ./rabbitmq3/sbin/rabbitmqctl cluster_status

   ## 创建集群
   ## 停止rabbit2@localhost应用，加入rabbit1@localhost集群，重启rabbit2 application
   ./rabbitmq2/sbin/rabbitmqctl stop_app
   ## ==> Stopping rabbit application on node rabbit2@localhost
   ./rabbitmq2/sbin/rabbitmqctl join_cluster rabbit1@localhost
   ## ==> Clustering node rabbit2@localhost with rabbit1@localhost
   ./rabbitmq2/sbin/rabbitmqctl start_app
   ## ==> Starting node rabbit2@localhost

   ## 将rabbit3@localhost，加入rabbit2@localhost集群
   ./rabbitmq3/sbin/rabbitmqctl stop_app
   ./rabbitmq3/sbin/rabbitmqctl join_cluster rabbit2@localhost
   ./rabbitmq3/sbin/rabbitmqctl start_app

   ## 查看rabbit1的集群状态
   ./rabbitmq1/sbin/rabbitmqctl cluster_status
   ## ==> Cluster status of node rabbit1@localhost
   ## ==> [{nodes,[{disc,[rabbit1@localhost,rabbit2@localhost,rabbit3@localhost]}]},
   ## ==>  {running_nodes,[rabbit3@localhost,rabbit2@localhost,rabbit1@localhost]},
   ## ==>  {cluster_name,<<"rabbit1@localhost">>},
   ## ==>  {partitions,[]},
   ## ==>  {alarms,[{rabbit3@localhost,[]},
   ## ==>           {rabbit2@localhost,[]},
   ## ==>           {rabbit1@localhost,[]}]}]
   ```

   **配置Nginx负载**

   ​	负载到Web管理控制台 15673、15674、15675，参照Nginx说明。

   **集群运维**

   ​	移除一个集群节点，例如 rabbit3

   ```
   ./rabbitmq3/sbin/rabbitmqctl stop_app
   ./rabbitmq3/sbin/rabbitmqctl reset
   ```

   ​

### 八. 参考资料

- http://www.jianshu.com/p/79ca08116d57
- http://www.codeweblog.com/rabbitmq-%E5%AE%89%E8%A3%85%E5%90%8E%E7%9A%84%E9%80%9A%E7%94%A8%E9%85%8D%E7%BD%AE%E4%B8%8E%E6%93%8D%E4%BD%9C/