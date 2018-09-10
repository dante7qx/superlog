## Edgware 版本

### 一. Maven 依赖

原Artifact ID依然可用，但一旦 **Finchley** 版本发布，老的Artifact ID将会废弃。

|             老 artifact id             |                 新 artifact id                 |
| :------------------------------------: | :--------------------------------------------: |
|      spring-cloud-starter-eureka       |   spring-cloud-starter-netflix-eureka-client   |
|   spring-cloud-starter-eureka-server   |   spring-cloud-starter-netflix-eureka-server   |
|      spring-cloud-starter-ribbon       |      spring-cloud-starter-netflix-ribbon       |
|      spring-cloud-starter-hystrix      |      spring-cloud-starter-netflix-hystrix      |
| spring-cloud-starter-hystrix-dashboard | spring-cloud-starter-netflix-hystrix-dashboard |
|      spring-cloud-starter-turbine      |      spring-cloud-starter-netflix-turbine      |
|  spring-cloud-starter-turbine-stream   |  spring-cloud-starter-netflix-turbine-stream   |
|       spring-cloud-starter-feign       |         spring-cloud-starter-openfeign         |
|       spring-cloud-starter-zuul        |       spring-cloud-starter-netflix-zuul        |

###  二. 配置Zuul的Hystrix线程池

​	默认情况下，Zuul的隔离策略是`SEMAPHORE` 。但一些场景下，我们可能需要将隔离策略改为`THREAD` ，设置`zuul.ribbonIsolationStrategy=THREAD` 即可。

​	当 `zuul.ribbonIsolationStrategy=THREAD`时，Hystrix的线程隔离策略将会作用于所有路由。此时，`HystrixThreadPoolKey` 默认为“RibbonCommand”。这意味着，所有路由的HystrixCommand都会在相同的Hystrix线程池中执行。

```yaml
zuul:
  threadPool:
    useSeparateThreadPools: true
    threadPoolKeyPrefix: zuulgw 
```

**参考：**

- https://github.com/spring-cloud/spring-cloud-netflix/pull/2074


	- https://www.cnblogs.com/happyflyingpig/p/8117309.html
- http://www.itmuch.com/spring-cloud/edgware-new-zuul-hystrix-thread-pool/

### 三. 使用配置文件定义 Feign

​	在 E.x 之前的版本，Feign 只能通过 Java代码进行自定义的配置，现在在 Edgware 版本中，Feign 支持通过配置属性（.yml）进行自定义配置。

```yaml
## 对单个 Feign Client 进行配置
feign:
  client:
    config:
      micro-provider-user:
        connect-timeout: 5000     # 相当于Request.Options
        read-timeout: 5000        # 相当于Request.Options
        loggerLevel: full         # 配置Feign的日志级别，相当于代码配置方式中的Logger
        error-decoder: 	          # Feign的错误解码器，相当于代码配置方式中的ErrorDecoder
        requestInterceptors: 	  # 相当于代码配置方式中的RequestInterceptor
        decode404: false
## 通用配置
feign:
  client:
    config:
      default:
        connect-timeout: 5000     
        read-timeout: 5000        
        loggerLevel: full         
## 整体配置，@EnableFeignClients(defaultConfiguration=DefaultRibbonConfig.class)
```

**优先级**

配置文件 > JavaConfig，可通过 `feign.client.default-to-properties=false`  修改优先级。

**参考：**

- https://github.com/spring-cloud/spring-cloud-netflix/pull/1942

### 四. Zuul routes端点功能增强

​	1. Zuul有一个非常实用的 `/routes` 端点。访问 `$ZUUL_URL/routes` 即可查看当前Zuul的路由规则，从而在很多情况下能够帮助我们定位Zuul的问题——当Zuul没有按照我们的计划去转发请求时， `/routes` 就会很有用，可通过该端点查看Zuul转发的规则。

​	Edgware 版本可以通过 **/routes?format=detail** 看到更多的信息。（没试成功）

```markdown
1. Zuul Server需要有Spring Boot Actuator的依赖，否则访问 /routes 端点将会返回404；。
2. 设置 management.security.enabled=false ，否则将会返回401；也可添加Spring Security的依赖，这样可通过账号、密码访问 routes 端点。
```

	2. 新增 `/filters` 端点，即 $ZUUL_URL/filters 。

### 五. @EnableEurekaClient

​	从Spring Cloud Edgware开始，`@EnableDiscoveryClient` 或`@EnableEurekaClient` **可省略**。只需加上相关依赖，并进行相应配置，即可将微服务注册到服务发现组件上。

​	**如不想将服务注册到Eureka Server**，只需设置`spring.cloud.service-registry.auto-registration.enabled=false` ，或`@EnableDiscoveryClient(autoRegister = false)` 即可。

### 六. Zuul回退的改进

​	通过实现`FallbackProvider` 接口，从而实现回退。

- FallbackProvider是ZuulFallbackProvider的子接口。
- ZuulFallbackProvider已经被标注`Deprecated` ，很可能在未来的版本中被删除。
- FallbackProvider接口比ZuulFallbackProvider多了一个`ClientHttpResponse fallbackResponse(Throwable cause);` 方法，使用该方法，**可获得造成回退的原因。**

```java
@Component
public class SpiritFallbackProvider implements FallbackProvider {

	private static final Logger LOGGER = LoggerFactory.getLogger(SpiritFallbackProvider.class);
	
	/**
	 * 表明是为哪个微服务提供回退，*表示为所有微服务提供回退
	 */
	@Override
	public String getRoute() {
		return "micro-provider-user";
	}

	@Override
	public ClientHttpResponse fallbackResponse(Throwable cause) {
		LOGGER.error("Zuul request error.", cause);
		if (cause instanceof HystrixTimeoutException) {
			return response(HttpStatus.GATEWAY_TIMEOUT);
		} else {
			return this.fallbackResponse();
		}
	}

	@Override
	public ClientHttpResponse fallbackResponse() {
		return this.response(HttpStatus.INTERNAL_SERVER_ERROR);
	}

	private ClientHttpResponse response(final HttpStatus status) {
		return new ClientHttpResponse() {
			@Override
			public HttpStatus getStatusCode() throws IOException {
				return status;
			}

			@Override
			public int getRawStatusCode() throws IOException {
				return status.value();
			}

			@Override
			public String getStatusText() throws IOException {
				return status.getReasonPhrase();
			}

			@Override
			public void close() {
			}

			@Override
			public InputStream getBody() throws IOException {
				return new ByteArrayInputStream("服务不可用，请稍后再试。".getBytes());
			}

			@Override
			public HttpHeaders getHeaders() {
				// headers设定
				HttpHeaders headers = new HttpHeaders();
				MediaType mt = new MediaType("application", "json", Charset.forName("UTF-8"));
				headers.setContentType(mt);
				return headers;
			}
		};
	}
}
```

### 八. 参考文档

- http://itmuch.com/spring-cloud-sum/spring-cloud-edgware-new-features/