## Hmily 使用心得

### 一. SpringCloud

#### 1. 依赖问题

- 对于单纯的服务提供方（不调用其他服务的），添加如下依赖，可使用 yml 进行 Hmily 的配置

```xml
<dependency>
    <groupId>org.dromara</groupId>
    <artifactId>hmily-spring-boot-starter-springcloud</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
        </exclusion>
        <exclusion>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<!-- 必须添加 Feign -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

```yaml
### application.yml
spring:
  profiles: 
    include: tcc
---
### application-tcc.yml
org:
  dromara:
    hmily:
      serializer: kryo
      recoverDelayTime: 32
      retryMax: 30
      scheduledDelay: 32
      scheduledThreadMax:  10
      repositorySupport: db
      started: false
      hmily-db-config:
        url: jdbc:mysql://localhost/hmily_tcc?characterEncoding=utf8&useSSL=true
        username: root
        password: iamdante
        driver-class-name: com.mysql.cj.jdbc.Driver
```

问题：无法注入 Feign 接口，即如下代码会报错

```java
public class incImpl {
    @Autowired
    private AccountFeignClient accountFeignClient;
}
```

- 通用的依赖和配置

```xml
<dependency>
    <groupId>org.dromara</groupId>
    <artifactId>hmily-springcloud</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
        </exclusion>
        <exclusion>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

```xml
<!-- applicationContext.xml -->
<context:component-scan base-package="org.dromara.hmily.*"/>
<aop:aspectj-autoproxy expose-proxy="true"/>
<bean id="hmilyTransactionBootstrap" class="org.dromara.hmily.core.bootstrap.HmilyTransactionBootstrap">
    <property name="serializer" value="kryo"/>
    <property name="recoverDelayTime" value="15"/>
    <property name="retryMax" value="30"/>
    <property name="scheduledDelay" value="15"/>
    <property name="scheduledThreadMax" value="4"/>
    <property name="repositorySupport" value="db"/>
    <property name="asyncThreads" value="200"/>
    <property name="started" value="false"/>
    <property name="hmilyDbConfig">
        <bean class="org.dromara.hmily.common.config.HmilyDbConfig">
            <property name="url"
                      value="jdbc:mysql://localhost:3306/hmily_tcc?characterEncoding=utf8&amp;useSSL=true"/>
            <property name="driverClassName" value="com.mysql.cj.jdbc.Driver"/>
            <property name="username" value="root"/>
            <property name="password" value="iamdante"/>
        </bean>
    </property>
</bean>
```

#### 2. 使用问题

- 注解@Hmily 使用在接口类上，T、C、C方法都应该进行本地事务控制

```java
@Transactional(readOnly = true)
public interface IMsgService {
    @Hmily
    public void receiveMsg(String msg);
}
public class MsgServiceImpl implements ImsgService {
    @Transactional
    @Hmily(confirmMethod="confirmMsg", cancelMethod="cancelMsg")
    public void receiveMsg(String msg) {
        ...
    } 
}
@Transactional
public void confirmMsg(String msg) {
    ...
}
@Transactional
public void cancelMsg(String msg) {
    ...
}
```

- Try 方法异常，直接本地事务回滚，不会调用 confirm 方法。

#### 3. 配置问题

- scheduledDelay      定期执行每个事务恢复间隔周期
- recoverDelayTime  事务日志表中需要执行的恢复记录（当前时间 - recoverDelayTime），大于RPC超时时间
- started                      消费方设为 true，生产方设为 false

### 二. Dubbo





### 三. 参考文档

- https://dromara.org/website/zh-cn/docs/hmily/index.html