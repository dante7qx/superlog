## Sleuth 说明

### 一. 概念

​	Spring Cloud Sleuth 是 Spring Cloud 实现的一套分布式跟踪系统的解决方案。设计理念基于[Dapper](http://bigbully.github.io/Dapper-translation/)。

#### 1. 分布式跟踪系统

​	为了支撑日益增长的庞大业务量，我们会把服务进行整合、拆分，使我们的服务不仅能通过集群部署抵挡流量的冲击，又能根据业务在其上进行灵活的扩展。一次请求少则经过三四次服务调用完成，多则跨越几十个甚至是上百个服务节点。如何动态展示服务的链路？如何分析服务链路的瓶颈并对其进行调优？如何快速进行服务链路的故障发现？这就是服务跟踪系统存在的目的和意义。

**设计目标**：无所不在的部署，持续的监控

1. 低消耗、高稳定：跟踪系统对在线服务的影响应该做到足够小。在一些高度优化过的服务，即使一点点损耗也会很容易察觉到，而且有可能迫使在线服务的部署团队不得不将跟踪系统关停。

2. ==**应用级的透明**==：对于应用的程序员来说，是不需要知道有跟踪系统这回事的。如果一个跟踪系统想生效，就必须需要依赖应用的开发者主动配合，那么这个跟踪系统也太脆弱了，往往由于跟踪系统在应用中植入代码的bug或疏忽导致应用出问题，这样才是无法满足对跟踪系统“无所不在的部署”这个需求。

   ​	最具挑战的设计目标，把核心跟踪代码做的很轻巧，然后把它植入到那些无所不在的公共组件中，比如线程调用、控制流以及RPC库。使用自适应的**采样率**可以使跟踪系统变得可伸缩，并降低性能损耗。

3. 延展性：随着接入的分布式系统的增多，压力也将不断增长，分布式跟踪系统是否能动态的扩展来支撑不断接入的业务系统。

#### 2. ZipKin

![zipkin](/Users/dante/Documents/Technique/且行且记/SpringCloud/zipkin.png)

​	ZipKin是基于Dapper论文理念的一套开源实现。主要流程如下：

![zipkin-流程图](/Users/dante/Documents/Technique/且行且记/SpringCloud/zipkin-流程图.png)

- collector：Zipkin Collector。
- storage：默认存储在 Cassandra，支持 ElasticSearch 和 MySQL。
- search：Zipkin Query Service。
- web UI：UI没有权限功能。

#### 3. 核心构成

- **Span**：跟踪系统中的基本数据单元，通常是很小的。可以简单的认为 一个请求就是一个 span。每个span通过一个64位 UUID 来保证唯一性。Span 必须是一个闭环，即**创建了一个 Span，必须在未来某一个时刻去关闭它**。Span 的数据结构体如下：
  - **TraceId**：全局跟踪ID（128-bit），标记一次完整的服务调用，一系列 **Span** 的集合，这些 Span 共享一个 RootSpan（服务的起始点，无 ParentId），并且共享一个 TraceId，然后通过 SpanId（64-bit） 和 ParentId 构成一个跟踪树的结构。
  - **SpanId**
  - **ParentId**
  - **name：**Span的名称，一般是接口方法名，用于分析服务。
  - **timestamp：**Span创建时的时间戳，用来记录采集的时刻。此记录是有偏差的，主机之间的时钟偏差。
  - **duration：**持续时间，实际分析服务调用的值，即span的创建到span完成最终的采集所经历的时间，除去span自己逻辑处理的时间，该时间段可以理解成对于该跟踪埋点来说服务调用的总耗时。
  - **Annotation**：基本标注列表，一个标注可以理解成span生命周期中重要时刻的数据快照。
    - timestamp：发生时刻
    - value：事件类型
      - cs - client send：客户端/消费者发起请求，一个 Span 的开始。
      - sr - server received：服务端/生产者接收到请求。sr - cs = 网络延迟时间
      - ss - server  send：服务端/生产者发送响应。ss - sr = 服务端处理时间
      - cr - client received：客户端/消费者接受响应，一个 Span 的结束。cr - cs = 整个服务调用的耗时
    - endpoint：端点信息
      - serviceName：Spring cloud 中的 spring.application.name
      - ipv4
      - port
  - **BinaryAnnotations：**业务标注列表，如果某些跟踪埋点需要带上部分业务数据（比如url地址、返回码和异常信息等），可以将需要的数据以键值对的形式放入到这个字段中。

![Zipkin flow](/Users/dante/Documents/Technique/且行且记/SpringCloud/Zipkin flow.png)

![zipkin span flow](/Users/dante/Documents/Technique/且行且记/SpringCloud/zipkin span flow.png)

- **Sampled:** 采样率，保证低消耗遇到大量请求时只记录其中的一小部分。自适应采样率。

### 二. 如何使用

#### 1. 服务端（也可直接使用官方jar）

- 在pom.xml添加依赖

  ```xml
  <dependency>
    	<groupId>io.zipkin.java</groupId>
    	<artifactId>zipkin-server</artifactId>
  </dependency>
  <dependency>
      <groupId>io.zipkin.java</groupId>
      <artifactId>zipkin-autoconfigure-ui</artifactId>
  </dependency>
  ```

- 在启动类上添加注解

  ```java
  @SpringBootApplication
  @EnableZipkinServer
  public class MicroSleuthServerApplication {
  	public static void main(String[] args) {
  		SpringApplication.run(MicroSleuthServerApplication.class, args);
  	}
  }
  ```

- 结合MySQL，参考 https://github.com/openzipkin/zipkin/tree/master/zipkin-storage/

  1. 在MySQL中创建数据库，[脚本](https://github.com/openzipkin/zipkin/blob/master/zipkin-storage/mysql/src/main/resources/mysql.sql)

  2. 添加依赖

     ```xml
     <!-- zipkin-storage-mysql 的版本要和 zipkin server 的版本保持一致 -->
     <dependency>
         <groupId>io.zipkin.java</groupId>
         <artifactId>zipkin-storage-mysql</artifactId>
       	<version>${zipkin-storage-mysql.version}</version>
     </dependency>
     <dependency>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-starter-jdbc</artifactId>
     </dependency>
     <dependency>
         <groupId>mysql</groupId>
         <artifactId>mysql-connector-java</artifactId>
     </dependency>
     ```

  3. 配置文件

     ```yaml
     spring:
       datasource:
         driver-class-name: com.mysql.jdbc.Driver
         url: jdbc:mysql://localhost/zipkin
         username: root
         password: iamdante
         continue-on-error: true
     zipkin:
       storage:
         type: mysql
     server:
       port: 9411
     ```

  4. 配置类

     ```java
     @Configuration
     public class ZipkinConfig {
     	@Bean
         public MySQLStorage mySQLStorage(DataSource datasource) {
             return MySQLStorage.builder().datasource(datasource).executor(Runnable::run).build();
         }
     }
     ```

#### 2.客户端

- 在 pom.xml 中添加依赖

  ```xml
  <dependency>
    	<groupId>org.springframework.cloud</groupId>
    	<artifactId>spring-cloud-starter-zipkin</artifactId>
  </dependency>
  ```

- 配置文件

  ```yaml
  # 只需在网关处设置
  spring:
    sleuth:
      sampler:
        percentage: 1.0  # 采样率，全量采集通信数据
    zipkin:
      base-url: http://localhost:9411	# zipkin server 的地址
  ```

  说明：采样率也可通过 javaconfig 进行配置

  ```java
  @Bean
  public Sampler sampler() {
    	return new AlwaysSampler();
  }
  ```

- 服务发现

  ```yaml
  spring.zipkin.locator.discovery.enabled: true  # 必须设置
  ```

  ​

### 三.  Zipkin 和 MQ

待续...

### 四. 参考资料

- http://bigbully.github.io/Dapper-translation/
- http://cloud.spring.io/spring-cloud-static/Dalston.SR1/#_spring_cloud_sleuth
- http://zipkin.io/
- https://github.com/openzipkin/
- http://opentracing.io/