## Nacos

本文档基于 Nacos v1.1.4。

### 一. 概述

Nacos 是阿里开源的一个以服务（微服务、云原生）为中心，提供动态服务发现、服务配置、服务元数据及流量管理的服务基础设施。目前 Nacos 支持的服务类型有 Kubernates Service、gRPC、Dubbo RPC、Spring Cloud Restful Service。

#### 1. 服务发现

Nacos 支持基于 DNS 和 RPC 的服务发现。服务提供者使用 SDK（目前官方提供Java版本）、OpenAPI（https://nacos.io/zh-cn/docs/open-api.html）进行注册，服务消费者通过 OpenAPI 去查找和发现所需要消费的服务。

#### 2. 健康检查

检查方式有 agent 上报模式和服务端主动检测2种健康检查模式。

- 传输层 L4 的（PING、TCP）健康检查
- 应用层 L7 的（HTTP、MYSQL？、用户自定义）健康检查
- 对于复杂的云环境和网络拓扑环境中（如 VPC、边缘网络等）服务的健康检查 ？

#### 3. 动态配置服务

以中心化、外部化和动态化的方式管理**所有环境**的**应用配置**和**服务配置**。提供包括配置版本跟踪、金丝雀发布、一键回滚配置以及客户端配置更新状态跟踪在内的一系列开箱即用的配置管理特性。

#### 4. 动态 DNS 服务

动态 DNS 服务支持权重路由，让您更容易地实现中间层负载均衡、更灵活的路由策略、流量控制以及数据中心内网的简单DNS解析服务。？

#### 5. 服务元数据

Nacos数据（如配置和服务）描述信息，如服务版本、权重、容灾策略、负载均衡策略、鉴权配置、监控 Metrics、各种自定义标签 (label)，从作用范围来看，分为服务级别的元信息、集群的元信息及实例的元信息。

### 二. 架构

![nacos架构](./images/nacos架构.jpg)



### 三. 使用

#### 1. 安装 Nacos

 - 二进制单机

   1. 下载 https://github.com/alibaba/nacos/releases，初始化数据库
   2. 启动

   ```bash
   sh startup.sh -m standalone
   （或）
   bash startup.sh -m standalone
   ```

 - 二进制单机集群

   1. 下载 https://github.com/alibaba/nacos/releases

   2. 复制 3 个 nacos-server，修改 application.properties、cluster.conf

      application.properties — 修改端口

      cluster.conf

      ```ini
      127.0.0.1:8848
      127.0.0.1:8849
      127.0.0.1:8850
      ```

      startup.sh 

      添加 `JAVA_OPT="${JAVA_OPT} -Dnacos.server.ip=127.0.0.1"` ，因为 Nacos 使用的

      InetAddress.*getLocalHost*().getHostAddress() 获取本机 IP，要确保和cluster.conf里配置的IP是一致的。

   3. 配置 nginx

      ```nginx
      upstream nacos-server {
          server 127.0.0.1:8848;
          server 127.0.0.1:8849;
          server 127.0.0.1:8850;
      }
      
      server {
      	listen 8888;
      	server_tokens on;
      	server_name x.dante.com;
      	
      	location /nacos/ {
          	proxy_pass http://nacos-server/nacos/;
        }
      }
      ```

      

### 八. 参考资料

- https://nacos.io/zh-cn/docs/what-is-nacos.html