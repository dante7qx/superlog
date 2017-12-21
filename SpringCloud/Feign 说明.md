## Feign 说明

### 一. 概念

​	Feign是一个声明式客户端工具。通过interface和annotation即可完成。interface可支持Feign原生注解、JAX-RS注解，也支持encoders and decoders。在SpringCloud中可以使用Spring MVC注解，并且集成了 Ribbon 和 Eureka用来提供负载均衡。

### 二. 工作机制

​	Feign通过配置注入一个模板化请求进行工作。只需在发送之前关闭它，参数就可以被直接的运用到模板中。然而这也限制了Feign，只支持文本形式的API，它可以在响应请求方面来简化系统。了解了这一点，这也非常容易进行你的单元测试转换。

### 三. 如何使用

1. 在pom.xml中添加依赖

   ```xml
   <dependency>
     	<groupId>org.springframework.cloud</groupId>
     	<artifactId>spring-cloud-starter-feign</artifactId>
   </dependency>
   ```

2. 在启动类添加注解 ***@EnableFeignClients***

   ```java
   @SpringBootApplication
   @EnableEurekaClient
   @EnableFeignClients
   public class FeignApplication {
   	public static void main(String[] args) {
   		SpringApplication.run(FeignApplication.class, args);
   	}
   }
   ```

3. Feign Client Interface

   ```java
   @FeignClient("micro-provider-user")
   public interface UserFeignClient {
   	@RequestMapping(method = RequestMethod.GET, value = "/user/{id}")
       public User getUser(@PathVariable("id") Long id);
   }
   ```

### 四. 集成Hystrix

#### 1. 依赖条件（Dalston中不会默认支持）

- Hystrix依赖包 `spring-cloud-starter-hystrix`
- feign.hystrix.enabled=true，默认为 true（Dalston Release版本默认为false）
- 方法加上注解@HystrixCommand

#### 2. Fallbacks

```java
@FeignClient(name = "micro-provider-user", fallback = HystrixClientFallback.class)
protected interface HystrixClient {
    @RequestMapping(method = RequestMethod.GET, value = "/hello")
    Hello iFailSometimes();
}

@Component
public class HystrixClientFallback implements HystrixClient {
    @Override
    public Hello iFailSometimes() {
        return new Hello("fallback");
    }
}
```

#### 3. FallbackFactory

```java
@FeignClient(name = "micro-provider-user", fallbackFactory = HystrixClientFallbackFactory.class)
protected interface HystrixClient {
	@RequestMapping(method = RequestMethod.GET, value = "/hello")
	Hello iFailSometimes();
}

@Component
public class HystrixClientFallbackFactory implements FallbackFactory<HystrixClient> {
	@Override
	public HystrixClient create(Throwable cause) {
		return new HystrixClientWithFallBackFactory() {
			@Override
			public Hello iFailSometimes() {
				return new Hello("fallback; reason was: " + cause.getMessage());
			}
		};
	}
}
```

#### 4. 禁用单个Feign的Hystrix支持

```java
/**
 * create a vanilla Feign.Builder with the "prototype" scope
 * 因为Feign默认的 Builder 是 HystrixFeign.Builder
 **／
@Configuration
public class DisableHystrixConfiguration {
    @Bean
	@Scope("prototype")
	public Feign.Builder feignBuilder() {
		return Feign.builder();
	}
}
```

#### 5. 注意

==Fallback方法不能返回`com.netflix.hystrix.HystrixCommand` and `rx.Observable`.==

### 五. Feign常见问题

#### 1. 不能使用`@GettingMapping` 之类的组合注解

#### 2. 若使用到 `@PathVariable`，必须指定其value，即@PathVariable("id")

```java
@RequestMapping(method = RequestMethod.GET, value = "/user/{id}")
public User getUser(@PathVariable("id") Long id);
```
#### 3. Get方式不支持接受复杂对象

```java
Feign调用
http://microservice-provider-user/query-by?id=1&username=张三

错误定义
@RequestMapping(value = "/query-by", method = RequestMethod.GET)
public User queryBy(User user);

正确定义1
@RequestMapping(value = "/query-by", method = RequestMethod.GET)
public User queryBy(@RequestParam("id")Long id, @RequestParam("username")String username);

正确定义2
@RequestMapping(value = "/query-by", method = RequestMethod.GET)
public List<User> queryBy(@RequestParam Map<String, Object> param);
```
#### 4. feign自定义Configuration和root 容器有效隔离

- 用@Configuration注解

- 不能在主@ComponentScan (or @SpringBootApplication)范围内，从其包名上分离或使用自定义Springboot不扫描注解（参考[Ribbon 说明.md](Ribbon 说明.md)）

- 注意避免包扫描重叠，最好的方法是明确的指定包名

- 默认Configuration：org.springframework.cloud.netflix.feign.FeignClientsConfiguration

  ```java
  @Configuration
  public class FeignClientConfig {
    	// 使用Feign原生注解
  	@Bean
      public Contract feignContract() {
          return new feign.Contract.Default();
      }
  	
    	// 调用Client具有权限验证
  	@Bean
      public BasicAuthRequestInterceptor basicAuthRequestInterceptor() {
          return new BasicAuthRequestInterceptor("dante", "iamdante");
      }
  	
    	/**
  	 * Feign 日志配置
  	 * 
  	 * 1、FeignClient注解中添加
  	 * 例如：@FeignClient(..., configuration=FeignClientConfig.class)
  	 * 2、在日志配置文件中加入具体Feign Client的配置
  	 * 例如：<logger name="org.dante.springcloud.feignclient.UserFeignConfigClient" level="DEBUG" />
  	 * 说明：level="DEBUG"，只有DEBUG才生效
  	 * 
  	 * @return
  	 */
  	@Bean
      Logger.Level feignLoggerLevel() {
          return Logger.Level.FULL;
      }
  }
  ```

- SpringCloud默认配置

  - `Decoder` feignDecoder: `ResponseEntityDecoder` (which wraps a `SpringDecoder`)
  - `Encoder` feignEncoder: `SpringEncoder`
  - `Logger` feignLogger: `Slf4jLogger`
  - `Contract` feignContract: `SpringMvcContract`
  - `Feign.Builder` feignBuilder: `HystrixFeign.Builder`
  - `Client` feignClient: if Ribbon is enabled it is a `LoadBalancerFeignClient`, otherwise the default feign client is used.

#### 5.使用ThreadLocal

​	如果您需要在RequestInterceptor中使用ThreadLocal绑定变量，则需要将Hystrix的线程隔离策略设置为“SEMAPHORE”，或者在Feign中禁用Hystrix。

```yaml
# To disable Hystrix in Feign
feign:
  hystrix:
    enabled: false

# To set thread isolation to SEMAPHORE
hystrix:
  command:
    default:
      execution:
        isolation:
          strategy: SEMAPHORE
```

#### 6. 配置

```properties
#Hystrix支持，如果为true，hystrix库必须在classpath中
feign.hystrix.enabled=false

#请求和响应GZIP压缩支持
feign.compression.request.enabled=true
feign.compression.response.enabled=true
#支持压缩的mime types
feign.compression.request.enabled=true
feign.compression.request.mime-types=text/xml,application/xml,application/json
feign.compression.request.min-request-size=2048

# 日志支持
logging.level.project.user.UserClient: DEBUG
```



### 六. 参考资料

- https://github.com/OpenFeign/feign
- http://cloud.spring.io/spring-cloud-static/Dalston.SR1/#spring-cloud-feign