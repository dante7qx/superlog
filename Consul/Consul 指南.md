## Consul 指南

### 一. 概念

​	Consul 是 Service Mesh 的一整套解决方案，采用 **Go** 语言编写。包含服务发现、配置共享、segmentation functionality 等。Consul 默认提供一个 Proxy（agent），也可以集成第三方Proxy，例如 Envoy。内置了服务注册与发现框架、分布一致性协议实现（Raft算法）、健康检查、Key/Value存储、TSL、服务通信限制、多数据中心方案，不再需要依赖其他工具（比如ZooKeeper等）。

**Gossip 协议**

​	常用于 P2P 的通信协议。现有一个 seed node，seed node 每秒都会随机向其他节点发送自己所拥有的节点列表，以及需要传播的消息。任何新加入的节点，就在这种传播方式下很快地被全网所知道。随着时间的增长，在最终的某一时刻，全网会得到相同的信息。即，达到最终一致性。

​	可参考：

		- http://bijian1013.iteye.com/blog/2355336
- https://hyperledgercn.github.io/hyperledgerDocs/gossip_zh/

**Consensus 协议**

​	Consul 使用 **Raft** 协议来实现 **CAP**。一致性指的是多个服务器的状态达成一致**（共享的存储保持一致）**，但在分布式环境，难免出现各种意外，这个就无法和其他服务保持一致。为了在意外发生时，系统任然能够正常的对外提供服务，就需要一种容错性的协议，即最终一致性协议。Raft 协议中，每一个Node在三中角色切换Leader、Follower、Candidate。可参考 https://zhuanlan.zhihu.com/p/27207160。

### 二. 架构

**术语**

#### 1. agent

Consul 的核心概念，每一个Consul Node 都需要启动一个 agent（Server 和 Client）。每个 agent 上运行有 DNS、HTTP接口，负责运行检查并保持服务同步。可以使用其他**Proxy（Envoy）**替代。

#### 2. Server

Server 负责扩展 Raft quorum、维持集群状态、相应**RPC**请求、通过 WAN gossip 和其他 DataCenter 通信。

#### 3. Client

Client 把所有的RPCs转发到server端，是相对无状态的。唯一在后台运行时的client端执行了LAN gossip pool，只消耗极少的资源和网络带宽。

#### 4. LAN Gossip 

局域网 Gossip Pool，其中包含位于同一局域网或数据中心的节点。

#### 5. WAN Gossip

WAN Gossip Pool，包含不同数据中心的服务器，通常通过互联网或广域网进行通信。

#### 6. RPC

Consul Client 向 Server 发出 RPC 请求。

### 三. 安装

1. 下载 https://www.consul.io/downloads.html
2. 解压压缩包，将解压后的二进制文件 consul 移到 /usr/local/bin 下。

```sh
mv consul /usr/local/bin
```

3. 启动

- 开发模式

```shell
consul agent -dev -node=consul-server -ui
```

- 集群模式

本例环境如下

- Mac 本机 - 10.71.226.130 
- Centos7 -   10.71.202.121
- Centos7 -   10.71.204.22 

```shell
## Mac 10.71.226.130 
consul agent -server -bootstrap -ui -data-dir=/Users/dante/Documents/Technique/Consul/data/demo -node=dante-mac-server -datacenter=dante-consul

## Centos 10.71.204.22 
consul agent -server -bootstrap-expect=3 -join=10.71.226.130 -ui -data-dir=/opt/data/consul -bind=10.71.204.22 -node=dante-linux-22 -datacenter=dante-consul

## Centos 10.71.202.121 
consul agent -server -bootstrap-expect=3 -join=10.71.226.130 -ui -data-dir=/opt/data/consul -bind=10.71.202.121 -node=dante-linux-121 -datacenter=dante-consul

## 测试
consul operator raft list-peers
consul members
```

### 四. 配置详解

```properties
-bootstrap：用来控制一个server是否在bootstrap模式，在一个datacenter中只能有一个server处于bootstrap模式，当一个server处于bootstrap模式时，可以自己选举为raft leader。

-bootstrap-expect：在一个datacenter中期望提供的server节点数目，当该值提供的时候，consul一直等到达到指定sever数目的时候才会引导整个集群，该标记不能和 bootstrap 公用。
```

### 五. SpringCloud集成

1. 添加依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
<!-- 默认健康检查Path: /actuator/health -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

2. 添加注解

```java
@EnableDiscoveryClient
@SpringBootApplication(exclude = { EurekaClientAutoConfiguration.class })
public class ConsulServiceApplication {
	public static void main(String[] args) {
		SpringApplication.run(ConsulServiceApplication.class, args);
	}
}
```

3. 配置

```yaml
server:
  port: 8100
spring:
  application:
    name: micro-consul-service
  cloud:
    consul:
      host: localhost
      port: 8500
      scheme: http
      discovery:
        instanceId: ${spring.application.name}:${vcap.application.instance_id:${spring.application.instance_id:${random.value}}}
        prefer-ip-address: true
        health-check-path: /health
        health-check-timeout: 10s
        health-check-interval: 8s
        tags: x-man, TaiTan
```

4. Consul Web 管理

   访问： http://localhost:8500/ui

### 六. 参考资料

- https://www.consul.io/docs
- http://chenjumin.iteye.com/blog/2293498