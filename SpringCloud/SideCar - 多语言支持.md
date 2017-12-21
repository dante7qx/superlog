## SideCar - 多语言支持

### 一. 概念

​	非 JVM 语言提供的 ==**Restful API**== ，如何使用 Eureka、Ribbon、ConfigServer。*Spring Cloud Netflix Sidecar* 借鉴 *Netflix Prana* 的思路。

1. 通过一个 http api 获取给定服务的所有实例（即主机和端口）。
2. 通过 Zuul Proxy 从 Eureka 获取路由信息。
3. 非 JVM 应用需要进行健康检查，以便 Sidecar 可以在应用程序启动或关闭时向eureka报告。
4. 非 JVM 应用必须和 Sidecar 应用在同一个 host 启动，若不在，则需要在 Sidecar 服务中配置 eureka.instance.hostname 。

### 二. 如何使用

1. 在pom.xml中添加依赖

   ```xml
   <dependency>
     	<groupId>org.springframework.cloud</groupId>
     	<artifactId>spring-cloud-netflix-sidecar</artifactId>
   </dependency>
   ```

2. 在启动类添加注解 @EnableSidecar

   ```java
   @SpringBootApplication
   @EnableSidecar
   public class SidecarApplication {
   	public static void main(String[] args) {
   		SpringApplication.run(SidecarApplication.class, args);
   	}
   }
   ```

   说明：@EnableSidecar 是组合注解，即添加此注解，就拥有了 CircuitBreaker、DiscoveryClient 和 Zuul 的功能。

   ```java
   @EnableCircuitBreaker
   @EnableDiscoveryClient
   @EnableZuulProxy
   @Target({java.lang.annotation.ElementType.TYPE})
   @Retention(RetentionPolicy.RUNTIME)
   @Documented
   @Import({SidecarConfiguration.class})
   public @interface EnableSidecar {
   }
   ```

3. 配置

   ```yaml
   server:
     port: 6789
   spring:
     application:
       name: micro-sidecar
       
   sidecar:
     port: 3000 # 非JVM微服务的端口
     health-uri: http://localhost:3000/health.json # 非JVM微服务的健康检查地址

   eureka: 
     instance:
       prefer-ip-address: true
     client:
       service-url:
         defaultZone: http://dante:iamdante@${eureka.host:localhost}:${eureka.port:8761}/eureka/
   ```

   注意：

   - 非JVM服务必须指定 Content-Type : application/json

   - Sidecar 服务本身不能直接访问非 JVM 应用服务

   - http://localhost:6789/

     [ping](http://localhost:6789/ping)
     [health](http://localhost:6789/health)
     [hosts/micro-sidecar](http://localhost:6789/hosts/micro-sidecar)


4. 访问

   启动一个 Zuul Proxy 微服务，如下

   ```json
   // Zuul 地址
   http://localhost:8882/routes
   {
   	"/zuul_file/**": "micro-provider-user3",
   	"/micro-sidecar/**": "micro-sidecar"
   }

   http://localhost:8882/micro-sidecar/
   {
   	"index": "欢迎使用 Sidecar 访问首页！"
   }
   ```

五. 参考资料

- http://cloud.spring.io/spring-cloud-static/Dalston.SR1/#_polyglot_support_with_sidecar


- https://github.com/Netflix/Prana
- https://github.com/spring-cloud/spring-cloud-netflix/issues/1505

