### Spring Boot Cache 使用说明

*官方文档*

http://docs.spring.io/spring/docs/current/spring-framework-reference/html/cache.html

#### Redis 缓存

- 1 安装Redis
- 2 添加Redis的配置

```    yaml
spring:
  redis:
    host: localhost
    password: null
    port: 6379
    pool:
      min-idle: 0
      max-idle: 8
      max-active: 8
      max-wait: 1
```
RedisCacheConfig.java

```java
package org.dante.springboot.cache.config;

import java.lang.reflect.Method;

import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.CachingConfigurerSupport;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.interceptor.CacheErrorHandler;
import org.springframework.cache.interceptor.CacheResolver;
import org.springframework.cache.interceptor.KeyGenerator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.ObjectMapper;

@Configuration
@EnableCaching
public class RedisCacheConfig extends CachingConfigurerSupport {

	/**
	 * 缓存管理器
	 */
	@Bean
	public CacheManager cacheManager(RedisTemplate<?, ?> redisTemplate) {
		CacheManager cacheManager = new RedisCacheManager(redisTemplate);
		return cacheManager;
	}
	@Bean
	public RedisTemplate<String, String> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
		RedisTemplate<String, String> template = new RedisTemplate<String, String>();
		template.setConnectionFactory(redisConnectionFactory);
		Jackson2JsonRedisSerializer<?> jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer<>(Object.class);  
        ObjectMapper om = new ObjectMapper();  
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);  
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);  
        jackson2JsonRedisSerializer.setObjectMapper(om);  
        template.setValueSerializer(jackson2JsonRedisSerializer);  
        template.setKeySerializer(new StringRedisSerializer());
        template.afterPropertiesSet();  
		return template;
	}
	@Override
	public KeyGenerator keyGenerator() {
		return new KeyGenerator() {
	           @Override
	           public Object generate(Object o, Method method, Object... objects) {
	              StringBuilder sb = new StringBuilder();
	              sb.append(o.getClass().getName());
	              sb.append(method.getName());
	              for (Object obj : objects) {
	                  sb.append(obj.toString());
	              }
	              return sb.toString();
	           }
	       };
	}
	@Override
	public CacheResolver cacheResolver() {
		// TODO Auto-generated method stub
		return super.cacheResolver();
	}
	@Override
	public CacheErrorHandler errorHandler() {
		// TODO Auto-generated method stub
		return super.errorHandler();
	}

}

```
- 3 启用注解

```xml
  1) 添加依赖包
  <dependency>
  	<groupId>org.springframework.boot</groupId>
  	<artifactId>spring-boot-starter-data-redis</artifactId>
  </dependency>
  <dependency>  
      <groupId>org.springframework.boot</groupId>  
      <artifactId>spring-boot-starter-cache</artifactId>  
  </dependency>
```

```xml
  2) 开启声明式缓存支持
  在配置类上添加注解  @EnableCaching
```

  

- 4 声明式注解

  ***@CacheConfig***

  ​	主要用于配置该类中会用到的一些共用的缓存配置。

  ```java
  @CacheConfig(cacheNames=RedisCacheConsts.FIND_USER_CACHE)
  说明：该类返回内容都存储到 RedisCacheConsts.FIND_USER_CACHE 的缓存对象中
  ```

  ***@Cacheable***

  ​	在方法执行前，Spring先查看缓存中是否有数据。有，则直接返回缓存数据；无，则调用方法将方法返回值放入缓存。

  ```java
  @Cacheable(value=RedisCacheConsts.FIND_USER_CACHE, key="'"+RedisCacheConsts.FIND_USER_CACHE+"'")
  public List<UserPO> findUsers() {
      logger.info("没有从缓存中读取所有用户。。。。。。。。。。。。。");
      return userDAO.findAll(new Sort(Sort.Direction.DESC, "updateDate"));
  }

  @Cacheable(value=RedisCacheConsts.FIND_USER_CACHE, key="\""+RedisCacheConsts.FIND_USER_CACHE+"\".concat('_').concat(#id)")
  public UserPO findUser(Long id) {
      logger.info("没有从缓存中读取指定 [{}] 的用户。。。。。。。。。。。。。", id);
      return userDAO.findOne(id);
  }
  ```

  ***@CachePut***

  ​	无论怎样，都会将方法的返回值放入缓存中。

  ```java
  @CachePut(value=RedisCacheConsts.FIND_USER_CACHE, key="\""+RedisCacheConsts.FIND_USER_CACHE+"\".concat('_').concat(#userPO.getId())")
  public UserPO insert(UserPO userPO) {
      UserPO u = userDAO.save(userPO);
      logger.info("添加缓存：{}", u);
      return u;
  }
  ```

  ***@CacheEvict***

  ​	从缓存中删除相应 Key 的数据。下例是删除所有RedisCacheConsts.FIND_USER_CACHE缓存对象。

  ```java
  @CacheEvict(value=RedisCacheConsts.FIND_USER_CACHE, allEntries=true)
  public void delete(Long id) {
      logger.info("删除缓存：{}", id);
      userDAO.delete(id);
  }
  ```

  ***@Caching***

  ​	组合多个缓存策略到一个方法上。

  ```java
  @Caching(evict={
  		@CacheEvict(key="\""+RedisCacheConsts.FIND_USER_CACHE+"\".concat('_').concat(#id)"),
  		@CacheEvict(key="'"+RedisCacheConsts.FIND_USER_CACHE+"'")
  })
  public void delete(Long id) {
      logger.info("删除缓存：{}", id);
      userDAO.delete(id);
  }
  ```
  ​

- 5 缓存的 Key

  ​	缓存的可以支持使用**SPEL**表达式

  | Name            | Location           | Description                              | Example                                  |
  | --------------- | ------------------ | ---------------------------------------- | ---------------------------------------- |
  | methodName      | root object        | The name of the method being invoked     | `#root.methodName`                       |
  | method          | root object        | The method being invoked                 | `#root.method.name`                      |
  | target          | root object        | The target object being invoked          | `#root.target`                           |
  | targetClass     | root object        | The class of the target being invoked    | `#root.targetClass`                      |
  | args            | root object        | The arguments (as array) used for invoking the target | `#root.args[0]`                          |
  | caches          | root object        | Collection of caches against which the current method is executed | `#root.caches[0].name`                   |
  | *argument name* | evaluation context | Name of any of the method arguments. If for some reason the names are not available (e.g. no debug information), the argument names are also available under the `#a<#arg>` where *#arg* stands for the argument index (starting from 0). | `#iban` or `#a0` (one can also use `#p0` or `#p<#arg>` notation as an alias). |
  | result          | evaluation context | The result of the method call (the value to be cached). Only available in `unless` expressions, `cache put` expressions (to compute the `key`), or `cache evict` expressions (when `beforeInvocation` is `false`). For supported wrappers such as `Optional`, `#result` refers to the actual object, not the wrapper. | `#result`                                |