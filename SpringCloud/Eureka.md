## Eureka 说明

### 一. 概念

​	Netflix的Service Discovery Server and Client，基于REST的服务，Spring Cloud将它集成在其子项目spring-cloud-netflix中，以实现服务发现功能。 用于服务发现。非常容易进行**HA**配置，每个集群服务间相互复制注册到Server上的Service。

- 中间层的负载均衡
- 使用 Netflix 的红/黑部署(red/black deployments)，使开发者更加容易实现云部署
- 对于 cassandra deployments，方便于对实例化后的对象维护
- 利用 memcached 提供缓存服务(*Memcached* 是一个高性能的分布式内存对象缓存系统，用于动态Web应用以减轻数据库负载。)。
  - 可配置性/动态配置。使用Eureka，可以即时添加或删除群集节点；Eureka使用archaius(Netflix开源的配置管理类):如果你有一个配置源实现，这些配置可以进行动态调整应用。
- Eureka Client 处理一个或多个Eureka服务器的故障。 由于 Eureka Client 在其中具有注册表缓存信息，因此即使所有 Eureka Servers 都关闭，它们还是可以很好地运行
- Eureka Servers 即使在其他 Eureka 挂了也具有极大的”弹性”。既是是在 Client 与 Server 的网络分裂(network partition)期间，Eureka Server 具有的内部弹性特性也能防止大规模服务中断。

**高可用架构**

![Eureka-HA](/Users/dante/Documents/Technique/且行且记/SpringCloud/Eureka-HA.png)

​	每个区域（Region，本例中只有一个Region）中都有一个 Eureka Cluster，它具有所有Service的注册信息，Eureka Server之间相互复制注册信息，确保总有一个Eureka Server可以处理区域故障。

​	Application Service 向 Eureka Server 注册服务，默认30s/次发送心跳（heartbeats）进行续租（renew），如果 Client 出现几次续租中断，3次心跳后，即90秒后，服务将从Eureka Server中去除。注册表和续租信息被复制到集群中其他Eureka Server中。

​	任何Zone（例：us-east-1）中的Client，30s/次从Eureka Server中获取它们需要的注册服务信息，进行远程调用。

### 二. Client 和 Server 交互方式

​	所有的交互均是通过Eureka REST API进行的。默认 EurekaClient 使用 Jersey 进行 Http 通信。若要使用 RestTemplate 进行交互（禁用Jersey），则需要进行如下配置。这样做的好处如下：

	1. 不用考虑 Jersey 的版本冲突问题，Jersey 1.x与2.x并不兼容。
	2. 减少了项目的依赖。

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
    <exclusions>
        <exclusion>
            <groupId>com.sun.jersey</groupId>
            <artifactId>jersey-client</artifactId>
        </exclusion>
        <exclusion>
            <groupId>com.sun.jersey</groupId>
            <artifactId>jersey-core</artifactId>
        </exclusion>
        <exclusion>
            <groupId>com.sun.jersey.contribs</groupId>
            <artifactId>jersey-apache-client4</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

​	**Register 注册** 

​	**Renew 续租**

```html
	Eureka client needs to renew the lease by sending heartbeats every 30 seconds. The renewal informs the Eureka server that the instance is still alive. If the server hasn't seen a renewal for 90 seconds, it removes the instance out of its registry. It is advisable not to change the renewal interval since the server uses that information to determine if there is a wide spread problem with the client to server communication.
```

​	**Fetch Registry 抓取注册信息**

```html
	Client向Server抓取注册信息并且缓存本地。之后，Client用本地Cache查找服务。该信息通过在最后一个获取周期和当前提取周期之间获取增量来定期更新（每30秒钟）更新一次。 增量信息在Server中保持较长时间（约3分钟），因此增量获取可能会再次返回相同的实例。 Eureka客户端自动处理重复信息。
	增量更新后，与Server比较。若信息不匹配，将会重新获取整个注册信息。Eureka Server以JSON/XML的格式缓存【全注册表、增量更新、每个服务】，Client通过 jersey apache client 来获取信息。
```

​	**Cancel 取消注册**

​	**网络中断**

```md
1、peers之间的心跳复制将会中断，Server检测到这种情况时会进入保护当前状态的自我保护模式。（默认开启自我保护模式）
2、注册可能发生在孤立的服务器中，一些客户端可能会反映新的注册，而其他客户端可能不会。
3、网络连接恢复到稳定状态后，情况会自动更正。 当peers能够正常通信时，注册信息被自动传送到没有它们的服务器。
```

### 三. Peer to Peer 通信

```md
	Eureka Client尝试与同一个区域的Eureka Server进行通话。 如果在与服务器通话时出现问题，或者服务器不存在于同一个Zone中，则客户机将故障切换到其他Zone中的服务器。
	一旦服务器开始接收流量，在服务器上执行的所有操作将被复制到服务器所知道的所有对等节点。 如果由于某种原因操作失败，则信息将在下一个心跳时，在服务器之间进行复制。
	当Eureka Server启动时，它尝试从相邻节点获取所有Instance注册表信息。如果从节点获取信息时出现问题，Server会放弃之前再进行尝试。如果可以获取所有Instance信息，则根据该信息设置应该接收的更新阈值。任何时间内，续订（renew）将低于为该值配置的百分比（在15分钟内低于85％），Server将不会与该Instance进行通信，并保护当前实例注册表信息。即自我保护模式。
	出现网络分区、eureka在短时间内丢失过多客户端时，会进入自我保护模式，即一个服务长时间没有发送心跳，eureka也不会将其删除。
```

### 四. Eureka Rest API

*APPID* -> `spring.application.name`

*instanceID* -> `服务的唯一表示。可从Eureka 8761 界面中查看`

Content-Type -> `application/json 或 application/xml`

==注意：在Spring Cloud中，要去掉 /V2==

| **Operation**                            | **HTTP action**                          | **Description**                          |
| ---------------------------------------- | ---------------------------------------- | ---------------------------------------- |
| Register new application instance        | POST /eureka/v2/apps/**appID**           | Input: JSON/XML payload HTTP Code: 204 on success |
| De-register application instance         | DELETE /eureka/v2/apps/**appID**/**instanceID** | HTTP Code: 200 on success                |
| Send application instance heartbeat      | PUT /eureka/v2/apps/**appID**/**instanceID** | HTTP Code:* 200 on success* 404 if **instanceID** doesn’t exist |
| Query for all instances                  | GET /eureka/v2/apps                      | HTTP Code: 200 on success Output: JSON/XML |
| Query for all **appID** instances        | GET /eureka/v2/apps/**appID**            | HTTP Code: 200 on success Output: JSON/XML |
| Query for a specific **appID**/**instanceID** | GET /eureka/v2/apps/**appID**/**instanceID** | HTTP Code: 200 on success Output: JSON/XML |
| Query for a specific **instanceID**      | GET /eureka/v2/instances/**instanceID**  | HTTP Code: 200 on success Output: JSON/XML |
| Take instance out of service             | PUT /eureka/v2/apps/**appID**/**instanceID**/status?value=OUT_OF_SERVICE | HTTP Code:* 200 on success* 500 on failure |
| Put instance back into service (remove override) | DELETE /eureka/v2/apps/**appID**/**instanceID**/status?value=UP (The value=UP is optional, it is used as a suggestion for the fallback status due to removal of the override) | HTTP Code:* 200 on success* 500 on failure |
| Update metadata                          | PUT /eureka/v2/apps/**appID**/**instanceID**/metadata?key=value | HTTP Code:* 200 on success* 500 on failure |
| Query for all instances under a particular **vip address** | GET /eureka/v2/vips/**vipAddress**       | * HTTP Code: 200 on success Output: JSON/XML * 404 if the **vipAddress** does not exist. |
| Query for all instances under a particular **secure vip address** | GET /eureka/v2/svips/**svipAddress**     | * HTTP Code: 200 on success Output: JSON/XML * 404 if the **svipAddress** does not exist. |

### 五. 配置Eureka Server 和 Eureka Client

- **公共配置**

  ```xml
  <!-- Spring boot 和 cloud -->
  <parent>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-parent</artifactId>
      <version>1.4.5.RELEASE</version>
  </parent>
  <dependencyManagement>
      <dependencies>
          <dependency>
              <groupId>org.springframework.cloud</groupId>
              <artifactId>spring-cloud-dependencies</artifactId>
              <version>Camden.SR7</version>
              <type>pom</type>
              <scope>import</scope>
          </dependency>
      </dependencies>
  </dependencyManagement>
  ```


#### Eureka Server

1. pom.xml 加入依赖

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```

2、在启动类上添加注解

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaStandaloneApplication {
	public static void main(String[] args) {
		SpringApplication.run(EurekaStandaloneApplication.class, args);
	}
}
```

3、配置

单例模式

```yaml
server:
  port: 8761
eureka:
  server:
    enable-self-preservation: false # 关闭自我保护模式
  instance:
    hostname: localhost
    prefer-ip-address: true # 猜测主机名时，优先选择ip。默认为false，使用hostname作为主机名
  client:
    registerWithEureka: false	# 不将自己注册到Eureka Server
    fetchRegistry: false 		# 不从Eureka Server抓取注册表信息，默认为true
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

集群模式（`eureka.instance.hostname`不是必须的）

```yaml
spring:
  application:
    name: amp-eureka-server-cluster
server:
  port: 8761

--- 
spring:
  profiles: cluster1
eureka:
  server:
    enable-self-preservation: false
  instance:
    prefer-ip-address: true
  client:
    serviceUrl:
      defaultZone: http://${amp.eureka.node2}:${server.port}/eureka/,http://${amp.eureka.node3}:${server.port}/eureka/

--- 
spring:
  profiles: cluster2
eureka:
  server:
    enable-self-preservation: false
  instance:
    prefer-ip-address: true
  client:
    serviceUrl:
      defaultZone: http://${amp.eureka.node1}:${server.port}/eureka/,http://${amp.eureka.node3}:${server.port}/eureka/
      
--- 
spring:
  profiles: cluster3
eureka:
  server:
    enable-self-preservation: false
  instance:
    hostname: localhost
    prefer-ip-address: true
  client:
    serviceUrl:
      defaultZone: http://${amp.eureka.node1}:${server.port}/eureka/,http://${amp.eureka.node2}:${server.port}/eureka/
```
#### Eureka Client

1. pom.xml 添加依赖

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

2. 在启动类上添加注解

```java
@SpringBootApplication
@EnableEurekaClient
public class Application {
	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
}
```

3. 配置 bootstrap.yml

```yaml
spring:
  application:
    name: micro-provider-user
eureka:
  instance:
    prefer-ip-address: true
  client:
    service-url:
      defaultZone: http://${eureka.host:localhost}:${eureka.port:8761}/eureka/
```

### 六. 常用配置

#### 1. Environment

`eureka.environment`

#### 2. Data center

`eureka.datacenter`

#### 3. Eureka的Instance ID

- 默认值

```
机器主机名:应用名称:应用端口

${spring.cloud.client.hostname}:${spring.application.name}:${spring.application.instance_id:${server.port}}
```

- 自定义

```yaml
eureka:
  instance:
    preferIpAddress: true
    instance-id: ${spring.application.name}:${spring.cloud.client.ipAddress}:${server.port}
```

#### 4. 原生应用名

Get the name of the application to be registered with eureka.

`eureka.instance.appname`

#### 5. 解决Eureka Server不踢出已关停的节点的问题（测试环境）

Server 端:

```yaml
eureka:
  server:
    enable-self-preservation: false	# 设为false，关闭自我保护
    eviction-interval-timer-in-ms: 4000 # 清理间隔（单位毫秒，默认是60*1000）
```

Client 端：

```yaml
eureka:
  client:
    healthcheck:
      enabled: true #开启健康检查（需要spring-boot-starter-actuator依赖）
  instance:
    lease-expiration-duration-in-seconds: 30 # 续约到期时间（默认90秒）
    lease-renewal-interval-in-seconds: 10 # 续约更新时间间隔（默认30秒)
```

==注意：`eureka.client.healthcheck.enabled=true` 只应该在application.yml中设置。如果设置在bootstrap.yml中将会导致一些不良的副作用，例如在Eureka中注册的应用名称是UNKNOWN等。==

### 七. 参考资料

-  https://github.com/Netflix/eureka/wiki
-  http://cloud.spring.io/spring-cloud-static/Dalston.SR1/#_spring_cloud_netflix
-  http://cloud.spring.io/spring-cloud-static/Dalston.SR1/#_appendix_compendium_of_configuration_properties
-  https://github.com/spring-cloud/spring-cloud-netflix/issues/203
-  https://mp.weixin.qq.com/s?__biz=MzI4ODQ3NjE2OA==&mid=2247483802&idx=1&sn=a3c10280d3345b0aa6ae9030bd744da0&chksm=ec3c9cfddb4b15eb3d213cfd3e060b08e81abbef112bdcb1c9c567eaa86f4558a7525b87a717&scene=0&key=dde1e3347992b369768d2b2d1deb375789428e10ac0f88e4629c83ebce894940d655dca10ef6e643300e6e28cbc7c6e80f0eef5952b3c8cb83898718079d9a802eda802d5f76722376b82e288809533d&ascene=0&uin=MTY1MzQxMzYxNg%3D%3D&devicetype=iMac+MacBookPro12%2C1+OSX+OSX+10.13.1+build(17B48)&version=12020010&nettype=WIFI&fontScale=100&pass_ticket=zlmRG2xqtSMMkJfOmIRopAZZZfIjfZWP8HA4vjsZ%2FTFnnkYNJh0kiAIw0yJ3YvTI

