## Spring Boot Admin

### 一. 前提

​	在生产环境中，需要对系统实际的运行情况（CPU、内存、IO、DB等指标）进行监控运维。Spring Boot 通过 spring-boot-actuator 提供了一个监控和管理生产环境的模块，可以使用http、jmx、ssh、telnet等协议管理和监控应用。审计（Auditing）、 健康（health）、数据采集（metrics gathering）会自动加入到应用里面。

**监控和管理Endpoint**

| HTTP方法 | Endpoint路径   | 描述                                       | 敏感信息  |
| ------ | ------------ | ---------------------------------------- | ----- |
| GET    | /autoconfig  | 当前应用的自动配置的使用情况                           | true  |
| GET    | /configprops | 查看配置属性，包括默认配置, 显示一个所有@ConfigurationProperties的整理列表 | true  |
| GET    | /beans       | bean及其关系列表, 显示一个应用中所有Spring Beans的完整列表   | true  |
| GET    | /dump        | 打印线程栈，线程状态信息                             | true  |
| GET    | /env         | 查看所有环境变量                                 | true  |
| GET    | /health      | 查看应用健康指标, 当使用一个未认证连接访问时显示一个简单的’status’，使用认证连接访问则显示全部信息详情 | false |
| GET    | /info        | 查看应用信息                                   | false |
| GET    | /mappings    | 查看所有url映射, 即所有@RequestMapping路径的整理列表     | true  |
| GET    | /metrics     | 查看应用基本指标                                 | true  |
| GET    | /trace       | 查看基本追踪信息，默认为最新的一些HTTP请求                  | true  |
| POST   | /shutdown    | 关闭应用，允许应用以优雅的方式关闭（默认情况下不启用）              | true  |

依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

配置监控

```yaml
management:
  port: 58888    ## 监控端口
endpoints.metrics.sensitive=false   #actuator的metrics接口是否需要安全保证
endpoints.metrics.enabled=true

endpoints.health.sensitive=false  #actuator的health接口是否需要安全保证
endpoints.health.enabled=true
```

使用

```html
http://localhost:58888/health
```

自定义Endpoint

1. 创建类

```java
/**
1、继承 AbstractEndpoint
2、实现 ApplicationContextAware，可以让当前类访问Spring容器内的资源，例：Bean、IOC
**/
@Component
public class ServerEndpoint extends AbstractEndpoint<String> implements ApplicationContextAware {
	
	private ApplicationContext context;
	
	public ServerEndpoint(String id) {
		super(id);
	}

	@Override
	public String invoke() {
		String appName = context.getEnvironment().getProperty("spring.application.name");
		return "Spirit Endpoint -> " + appName;
	}

	@Override
	public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
		this.context = applicationContext;
	}

}
```

2. 注册Endpoint

```java
@Bean
public Endpoint<String> serverEndpoint() {
    Endpoint<String> serverEndpoint = new ServerEndpoint("spirit");
    return serverEndpoint;
}
```

### 二. 概念

​	用来监控和管理 Springboot 应用的开源包，提供了一款基于Spring Boot Actuator endpoints的UI界面。

### 三. 使用

#### 1. Springboot应用

##### 服务端

添加依赖

```xml
<properties>
  	<admin.version>1.5.5</admin.version>
</properties>
<dependencies>
    <dependency>
        <groupId>de.codecentric</groupId>
        <artifactId>spring-boot-admin-server</artifactId>
        <version>${admin.version}</version>
    </dependency>
    <dependency>
        <groupId>de.codecentric</groupId>
        <artifactId>spring-boot-admin-server-ui</artifactId>
        <version>${admin.version}</version>
    </dependency>
    <dependency>
        <groupId>de.codecentric</groupId>
        <artifactId>spring-boot-admin-server-ui-login</artifactId>
        <version>${admin.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
</dependencies>
```

Java类

1. 启动类上添加注解 `@EnableAdminServer` 。
2. 编写SpringSecurity

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
	@Override
	protected void configure(HttpSecurity http) throws Exception {
		// Page with login form is served as /login.html and does a POST on /login
		http.formLogin().loginPage("/login.html").loginProcessingUrl("/login").permitAll();
		// The UI does a POST on /logout on logout
		http.logout().logoutUrl("/logout");
		// The ui currently doesn't support csrf
		http.csrf().disable();
		// Requests for the login page and the static assets are allowed
		http.authorizeRequests().antMatchers("/login.html", "/**/*.css", "/img/**", "/third-party/**").permitAll();
		// ... and any other request needs to be authorized
		http.authorizeRequests().antMatchers("/**").authenticated();

		// Enable so that the clients can authenticate via HTTP basic for registering
		http.httpBasic();
	}
}
```

配置文件

```yaml
server:
  port: 8900
spring:
  application:
    name: springboot_admin_server
endpoints:
  health:
    sensitive: false
security:
  user:
    name: admin
    password: admin123
```

##### 客户端

添加依赖

```xml
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-client</artifactId>
    <version>1.5.5</version>
</dependency>
```

配置文件

```yaml
server:
  address: 127.0.0.1
spring:
  application:
    name: springboot_admin_client
  boot:
    admin:
      url: http://localhost:8900
      username: admin
      password: admin123
      client:
#        prefer-ip: true
        metadata:	## BasicAuthHttpHeaderProvider使用metadata加入到Authorization Header去验证
          user.name: admin
          user.password: admin123
management:
  port: 58888
  security:
    enabled: false	## 1.5.x版本后，所有的endpoints都是secured，这里需要关闭
```

#### 2. SpringCloud应用



### 八. 参考资料

- https://codecentric.github.io/spring-boot-admin/1.5.5/


- https://github.com/codecentric/spring-boot-admin
- https://medium.com/@brunosimioni/near-real-time-monitoring-charts-with-spring-boot-actuator-jolokia-and-grafana-1ce267c50bcc
- http://www.jianshu.com/p/e20a5f42a395
- http://www.jianshu.com/p/154fbb484614
- http://wiki.jikexueyuan.com/project/spring-boot-cookbook-zh/spring-boot-jmx-monitor.html



