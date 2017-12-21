## Circuit Breaker - Hystrix

![hystrix-logo](/Users/dante/Documents/Technique/且行且记/SpringCloud/hystrix-logo.png)

### 一. 概念

​	断路器模式。断路器是一个电路中的元器件，作用是当电路中出现了短路或者瞬间电流过大等问题，断路器直接跳闸，当修复电路中的问题后，必须手动拨动断路器的开关到联通状态，整个电路才恢复正常通电。

​	在云环境中，通常都是微服务的架构，各个服务往往运行在不同的进程中,而且很可能是不同的服务器,甚至是不同的数据中心，服务调用通常会有延迟,如果某个环节的延迟不断传递下去，讲导致整个系统的崩溃（雪崩效应）。

​	云环境的服务调用模式。

服务调用失败时,接下来的调用都会失败，每一次的失败都会造成很大的延迟，直到服务恢复正常。那么能否在失败时，不再调用失败的服务，而直接返回错误信息，然后服务内部自己去调用服务，如果发现成功了，再直接调用服务，返回结果，而不再返回错误信息。即容错机制。

![断路器模式](/Users/dante/Documents/Technique/且行且记/SpringCloud/断路器模式.png)

### 二. 原理

​	断路器模式能阻止应用重复调用曾经调用失败的服务,断路器直接返回错误信息，与此同时，断路器内部能够侦测服务是否恢复可用，如果可用，应用将再次直接调用正常的服务。

​	断路器：当Hystrix Command请求后端服务失败数量超过一定比例(默认50%)，断路器会切换到开路状态(Open). 这时所有请求会直接失败而不会发送到后端服务. 断路器保持在开路状态一段时间后(默认5秒), 自动切换到半开路状态(HALF-OPEN). 这时会判断下一次请求的返回情况, 如果请求成功, 断路器切回闭路状态(CLOSED), 否则重新切换到开路状态(OPEN)。![断路器原理](/Users/dante/Documents/Technique/且行且记/SpringCloud/断路器原理.png)

### 三. Hystrix介绍

​	在分布式环境中，不可避免地有许多服务调用失败。 Hystrix是一个库，可以通过添加延迟容错和容错逻辑来帮助您控制这些分布式服务之间的交互。 Hystrix通过隔离服务之间的接入点，阻止它们之间的级联故障，并提供备用选项Fallback，从而提高系统的整体弹性。

#### **1. 作用**

- 控制被依赖服务的延时和失败
- 防止在复杂系统中的级联失败
- 可以进行快速失败（不需要等待）和 **快速恢复**（当依赖服务失效后又恢复正常，其对应的线程池会被清理干净，即剩下的都是未使用的线程，相对于整个 Tomcat 容器的线程池被占满需要耗费更长时间以恢复可用来说，此时系统可以快速恢复）
- getFallback（失败时指定的操作）和优雅降级
- 实现近实时的检测、报警、运维

#### **2. 注意**

- 为每一个依赖服务维护一个线程池（或者信号量），当线程池占满+queueSizeRejectionThreshold占满，该依赖服务将会立即拒绝服务而不是排队等待
- 引入『熔断器』机制，在依赖服务失效比例超过阈值时，手动或者自动地切断服务一段时间
- 当请求依赖服务时出现拒绝服务、超时或者短路（多个依赖服务顺序请求，前面的依赖服务请求失败，则后面的请求不会发出）时，执行该依赖服务的失败回退逻辑

#### **3. 线程模型**

![Hystrix线程模型](/Users/dante/Documents/Technique/且行且记/SpringCloud/Hystrix线程模型.png)

#### **4. 工作机理**

![Hystrix工作机理](/Users/dante/Documents/Technique/且行且记/SpringCloud/Hystrix工作机理.png)

#### 5. 线程池与信号量

​	使用场景：对于那些本来延迟就比较小的请求（例如访问本地缓存成功率很高的请求）来说，线程池带来的开销是非常高的，这时，你可以考虑采用其他方法，例如非阻塞信号量（不支持超时），来实现依赖服务的隔离，使用信号量的开销很小。但**绝大多数情况下，Netflix 更偏向于使用线程池来隔离依赖服务**，因为其带来的额外开销可以接受，并且能支持包括超时在内的所有功能。

​	如果你对客户端库有足够的信任（延迟不会过高），并且你只需要控制系统负载，那么你可以使用信号量。

### 四. 如何使用

#### 1. 在pom.xml中添加依赖

```xml
<dependency>
  	<groupId>org.springframework.cloud</groupId>
  	<artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>
```

#### 2. 在启动类添加注解 @EnableCircuitBreaker

```java
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients
@EnableCircuitBreaker
public class FeignHystrixApplication {
	public static void main(String[] args) {
		SpringApplication.run(FeignHystrixApplication.class, args);
	}
}
```

#### 3. 在方法上添加注解 @HystrixCommand

SpringCloud 自动在一个Proxy中封装有@HystrixCommand注解的 Spring bean，Proxy会接入 Hystrix circuit breaker。 

#### 4. 配置@HystrixCommand

 `com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand`，参考

https://github.com/Netflix/Hystrix/wiki/Configuration

```java
// 隔离策略 - SEMAPHORE，默认为 THREAD
@HystrixCommand(commandProperties = {@HystrixProperty(name = "execution.isolation.strategy", value = "SEMAPHORE") })

// 禁用 HystrixCommand 的超时控制，默认为 True 启用
@HystrixCommand(commandProperties = {@HystrixProperty(name = "execution.timeout.enabled", value = "false") })

// HystrixCommand 超时时间，默认为 1000ms
@HystrixCommand(commandProperties = {@HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "5000") })
```

#### 5.监控Hystrix

- */hystrix.stream*
- */health*

### 五. Hystrix Dashborad

​	以仪表盘的方式的界面显示每一个HystrixCommand的 hystrix.stream信息。

#### 1. 在pom.xml中添加依赖

```xml
<dependency>
  	<groupId>org.springframework.cloud</groupId>
  	<artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>
<dependency>
  	<groupId>org.springframework.cloud</groupId>
  	<artifactId>spring-cloud-starter-hystrix-dashboard</artifactId>
</dependency>
```

#### 2. 在启动类添加注解

```java
@SpringBootApplication
@EnableHystrixDashboard
public class MicroHystrixDashboardApplication {
	public static void main(String[] args) {
		SpringApplication.run(MicroHystrixDashboardApplication.class, args);
	}
}
```

#### 3. 使用方式（访问过HystrixCommand后才生效）

​	访问 http://localhost:8666/hystrix，然后输入 http://localhost:7061/hystrix.stream

### 六. Turbine

#### 1. 概念

​	Turbine是将服务器发送事件（SSE）JSON数据流聚合成一个流的工具。 目标用例是针对仪表板聚合的SOA中的实例的度量流。	在SpringCloud中，将所有`/hystrix.stream` endpoints 的信息聚合到一个 /turbine.stream。可通过Hystrix Dashboard查看。

#### 2. 使用

	1. 在pom.xml中添加依赖

```xml
<dependency>
  	<groupId>org.springframework.cloud</groupId>
  	<artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>
<dependency>
  	<groupId>org.springframework.cloud</groupId>
  	<artifactId>spring-cloud-starter-turbine</artifactId>
</dependency>
```

2. 在启动类添加注解

```java
// 注册到 Eureka Server，可以监控服务注册表中的所有服务
@SpringBootApplication
@EnableEurekaClient
@EnableTurbine
public class MicroHystrixTurbineApplication {

	public static void main(String[] args) {
		SpringApplication.run(MicroHystrixTurbineApplication.class, args);
	}
}
```

3. 配置

   访问方式

 http://my.turbine.sever:8080/turbine.stream?cluster=<CLUSTERNAME>
 http://localhost:9001/turbine.stream?cluster=MICRO-CONSUMER-USER-FEIGN-HYSTRIX

```yaml
# 单服务配置
turbine:
  aggregator:
    clusterConfig: MICRO-CONSUMER-USER-FEIGN-HYSTRIX   # 必须大写
  appConfig: micro-consumer-user-feign-hystrix
```

```yaml
# 集群服务配置，可在dashborad中直接访问 http://my.turbine.sever:8080/turbine.stream
turbine:
  aggregator:
    clusterConfig: default # 指定聚合哪些集群，多个用","分割，默认为default
  appConfig: micro-consumer-user-feign-hystrix,micro-consumer-user-feign-hystrix2 # 配置Eureka中serverId列表，表明监控哪些服务
  cluster-name-expression: "'default'"
# 1. clusterNameExpression指定集群名称，默认表达式appName。clusterConfig需要配置想要监控的应用名称
# 2. 当clusterNameExpression: default时。clusterConfig 可以不配置
# 3. 当clusterNameExpression: metadata['cluster']时。假设要监控的应用配置了 eureka.instance.metadata-map.cluster: AMP，则需要配置 clusterConfig: AMP
```

```yaml
1. 当应用服务配置 server.context-path: /project 后，也需要配置 eureka.instance.home-page-url: /project
2. turbine服务需要配置 turbine.instanceUrlSuffix.<appName>: project/hystrix.stream, 即
   turbine.instanceUrlSuffix.MICRO-CONSUMER-USER-FEIGN-HYSTRIX: project/hystrix.stream
```

```yaml
# 配置管理端口。一个应用可以有两个端口
--- 
# 应用
management:
  port: 10000
eureka:
  instance:
    metadata-map:
      management.port: ${management.port:7065}
      cluster: USER
--- 
# Turbine 配置
turbine:
  aggregator:
    clusterConfig: USER
  appConfig: micro-consumer-user-feign-hystrix4
  cluster-name-expression: metadata['cluster']
```

### 七. 参考资料

- https://martinfowler.com/bliki/CircuitBreaker.html
- https://github.com/Netflix/Hystrix/wiki
- https://github.com/Netflix/Hystrix/tree/master/hystrix-contrib/hystrix-javanica
- http://www.cnblogs.com/java-zhao/p/5521233.html
- https://github.com/Netflix/Hystrix/wiki/Dashboard
- https://github.com/Netflix/Turbine
- http://blog.csdn.net/liaokailin/article/details/51344281
- http://blog.csdn.net/forezp/article/details/70233227