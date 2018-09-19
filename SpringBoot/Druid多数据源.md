## Springboot Druid Jpa 多数据源配置

### 1. 引入Druid

在 Spring Boot 项目中加入 `druid-spring-boot-starter` 依赖

```xml
<dependency>
   <groupId>com.alibaba</groupId>
   <artifactId>druid-spring-boot-starter</artifactId>
   <version>1.1.5</version>
</dependency>
```

### 2. 配置属性

####  JDBC 配置

```properties
spring.datasource.druid.url= # 或spring.datasource.url= 
spring.datasource.druid.username= # 或spring.datasource.username=
spring.datasource.druid.password= # 或spring.datasource.password=
spring.datasource.druid.driver-class-name= #或 spring.datasource.driver-class-name=
```

#### 多数据源配置

Druid 数据源配置，继承spring.datasource.* 配置，相同则覆盖

```yaml
spring:
  datasource:
    druid: 
      driver-class-name: com.mysql.jdbc.Driver
      initial-size: 10
      min-idle: 10
      max-active: 30
      max-wait: 60000
      min-evictable-idle-time-millis: 300000
      pool-prepared-statements: false
      max-pool-prepared-statement-per-connection-size: 20
      validation-query: select 1
      test-on-borrow: false
      test-on-return: false
      test-while-idle: true
      filter:
            stat:
              db-type: mysql
              log-slow-sql: true
              slow-sql-millis: 2000
            wall:
              enabled: true
              db-type: mysql
              config:
                multi-statement-allow: true
      web-stat-filter:
        exclusions: '*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*'
      stat-view-servlet:
        login-username: druid
        login-password: 123456
      springboot: ## springboot数据源，继承 spring.datasource.druid.* 配置，相同则覆盖
        url: jdbc:mysql://localhost/springboot
        username: root
        password: iamdante
      shiro: ## shiro数据源，继承 spring.datasource.druid.* 配置，相同则覆盖
        url: jdbc:mysql://localhost/shiro
        username: root
        password: iamdante
mybatis:
  config-location: classpath:mybatis-config.xml ## Mybatis 配置
```

#### JavaConfig

==注意：从数据源Mybatis的事务不起作用。==

##### 总数据源配置

```java
@Configuration
public class DruidDatasourceConfig {
    // @Primary表示主数据源
	@Primary
	@Bean
	@ConfigurationProperties("spring.datasource.druid.springboot")
	public DataSource springbootDataSource(){
	    return DruidDataSourceBuilder.create().build();
	}
	@Bean
	@ConfigurationProperties("spring.datasource.druid.shiro")
	public DataSource shiroDataSource(){
	    return DruidDataSourceBuilder.create().build();
	}
}
```

##### springboot数据源

​	LocalContainerEntityManagerFactoryBean 和 EntityManagerFactory 方法其中一个注解@Primary即可，不然启动会报错。

```java
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(entityManagerFactoryRef = "springbootEntityManagerFactory", transactionManagerRef = "springbootTransactionManager", basePackages = {SpringbootDataSourceConfig.JPA_DAO_PKG}) // 设置Repository所在位置
@MapperScan(basePackages = SpringbootDataSourceConfig.MYBATIS_MAPPER_PKG, sqlSessionFactoryRef = "springbootSqlSessionFactory")
public class SpringbootDataSourceConfig {
	protected static final String JPA_DAO_PKG = "org.dante.springboot.dao.springboot";
	protected static final String JPA_PO_PKG = "org.dante.springboot.po.springboot";
	protected static final String MYBATIS_BO_PKG = "org.dante.springboot.bo.springboot";
	protected static final String MYBATIS_MAPPER_PKG = "org.dante.springboot.mapper.springboot";
	protected static final String MYBATIS_XML_PKG = "classpath:mapper/springboot/*.xml";
    protected static final String MYBATIS_CONFIG = "classpath:mybatis-config.xml";
	
	@Autowired
	private JpaProperties jpaProperties;
	@Autowired
	@Qualifier("springbootDataSource")
	private DataSource springbootDataSource;
	
	/**
	 * 通过LocalContainerEntityManagerFactoryBean来获取EntityManagerFactory实例
	 * 
	 * @return
	 */
	@Bean(name = "springbootEntityManagerFactoryBean")
	public LocalContainerEntityManagerFactoryBean springbootEntityManagerFactoryBean(
			EntityManagerFactoryBuilder builder) {
		return builder.dataSource(springbootDataSource).properties(getVendorProperties(springbootDataSource))
				.packages(SpringbootDataSourceConfig.JPA_PO_PKG) // 设置实体类所在位置
				.persistenceUnit("springbootPersistenceUnit").build();
		// .getObject();//不要在这里直接获取EntityManagerFactory
	}

	private Map<String, String> getVendorProperties(DataSource dataSource) {
		return jpaProperties.getHibernateProperties(dataSource);
	}

	/**
	 * EntityManagerFactory类似于Hibernate的SessionFactory,mybatis的SqlSessionFactory
	 * 总之,在执行操作之前,我们总要获取一个EntityManager,这就类似于Hibernate的Session,
	 * mybatis的sqlSession.
	 * 
	 * @param builder
	 * @return
	 */
	@Bean(name = "springbootEntityManagerFactory")
	@Primary
	public EntityManagerFactory springbootEntityManagerFactory(EntityManagerFactoryBuilder builder) {
		return this.springbootEntityManagerFactoryBean(builder).getObject();
	}
  
	/**
	 * 配置事物管理器
	 * 
	 * @return
	 */
	@Bean(name = "springbootTransactionManager")
	@Primary
	public PlatformTransactionManager writeTransactionManager(EntityManagerFactoryBuilder builder) {
		return new JpaTransactionManager(springbootEntityManagerFactory(builder));
	}

	/**
	 * Mybatis 配置
	 * 
	 */
	@Bean(name = "springbootSqlSessionFactory")
    @Primary
    public SqlSessionFactory springSqlSessionFactory()
            throws Exception {
        final SqlSessionFactoryBean sessionFactory = new SqlSessionFactoryBean();
        sessionFactory.setDataSource(springbootDataSource);
        sessionFactory.setVfs(SpringBootVFS.class); 
      	PathMatchingResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();        		sessionFactory.setMapperLocations(resolver.getResources(SpringbootDataSourceConfig.MYBATIS_XML_PKG));
        sessionFactory.setConfigLocation(resolver.getResource(SpringbootDataSourceConfig.MYBATIS_CONFIG));
        sessionFactory.setTypeAliasesPackage(SpringbootDataSourceConfig.MYBATIS_BO_PKG);
        return sessionFactory.getObject();
    }
}
```

##### shiro数据源

​	注意 @Primary 的使用，shiro不是主数据源，EntityManagerFactory、PlatformTransactionManager不要配置@Primary 。

```java
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(entityManagerFactoryRef = "shiroEntityManagerFactory", transactionManagerRef = "shiroTransactionManager", basePackages = { ShiroDataSourceConfig.JPA_DAO_PKG }) 
@MapperScan(basePackages = ShiroDataSourceConfig.MYBATIS_MAPPER_PKG, sqlSessionFactoryRef = "shiroSqlSessionFactory")
public class ShiroDataSourceConfig {
	protected static final String JPA_DAO_PKG = "org.dante.springboot.dao.shiro";
	protected static final String JPA_PO_PKG = "org.dante.springboot.po.shiro";
	protected static final String MYBATIS_BO_PKG = "org.dante.springboot.bo.shiro";
	protected static final String MYBATIS_MAPPER_PKG = "org.dante.springboot.mapper.shiro";
	protected static final String MYBATIS_XML_PKG = "classpath:mapper/shiro/*.xml";
	
	@Autowired
	private JpaProperties jpaProperties;

	@Autowired
	@Qualifier("shiroDataSource")
	private DataSource shiroDataSource;

	/**
	 * 通过LocalContainerEntityManagerFactoryBean来获取EntityManagerFactory实例
	 * 
	 * @return
	 */
	@Bean(name = "shiroEntityManagerFactoryBean")
	// @Primary
	public LocalContainerEntityManagerFactoryBean shiroEntityManagerFactoryBean(EntityManagerFactoryBuilder builder) {
		return builder.dataSource(shiroDataSource).properties(getVendorProperties(shiroDataSource))
				.packages(ShiroDataSourceConfig.JPA_PO_PKG) // 设置实体类所在位置
				.persistenceUnit("shiroPersistenceUnit").build();
		// .getObject();//不要在这里直接获取EntityManagerFactory
	}

	private Map<String, String> getVendorProperties(DataSource dataSource) {
		return jpaProperties.getHibernateProperties(dataSource);
	}

	/**
	 * EntityManagerFactory类似于Hibernate的SessionFactory,mybatis的SqlSessionFactory
	 * 总之,在执行操作之前,我们总要获取一个EntityManager,这就类似于Hibernate的Session,
	 * mybatis的sqlSession.
	 * 
	 * @param builder
	 * @return
	 */
	@Bean(name = "shiroEntityManagerFactory")
	public EntityManagerFactory shiroEntityManagerFactory(EntityManagerFactoryBuilder builder) {
		return this.shiroEntityManagerFactoryBean(builder).getObject();
	}

	/**
	 * 配置事物管理器
	 * 
	 * @return
	 */
	@Bean(name = "shiroTransactionManager")
	public PlatformTransactionManager writeTransactionManager(EntityManagerFactoryBuilder builder) {
		return new JpaTransactionManager(shiroEntityManagerFactory(builder));
	}

	/**
	 * Mybatis 配置
	 * 
	 */
	@Bean(name = "shiroSqlSessionFactory")
    public SqlSessionFactory springSqlSessionFactory()
            throws Exception {
        final SqlSessionFactoryBean sessionFactory = new SqlSessionFactoryBean();
        sessionFactory.setDataSource(shiroDataSource);
        sessionFactory.setVfs(SpringBootVFS.class);
        PathMatchingResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
        sessionFactory.setMapperLocations(resolver.getResources(ShiroDataSourceConfig.MYBATIS_XML_PKG));
        sessionFactory.setConfigLocation(resolver.getResource(ShiroDataSourceConfig.MYBATIS_CONFIG));
        sessionFactory.setTypeAliasesPackage(ShiroDataSourceConfig.MYBATIS_BO_PKG);
        return sessionFactory.getObject();
    }
}
```

##### mybatis-config.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <properties>
        <property name="dialect" value="mysql" />
    </properties>
	<settings>
        <!-- 开启驼峰匹配 -->
        <setting name="mapUnderscoreToCamelCase" value="true"/>
    </settings>
    <!-- 分页助手 -->
    <plugins>
        <plugin interceptor="com.github.pagehelper.PageHelper">
            <property name="dialect" value="h2" />
            <property name="offsetAsPageNum" value="true" />
            <property name="rowBoundsWithCount" value="true" />
            <property name="pageSizeZero" value="true" />
            <property name="reasonable" value="true" />
        </plugin>
        <plugin interceptor="org.dante.springboot.plugin.SqlCostInterceptor"></plugin>
    </plugins>
</configuration> 
```

### 3. 参考资料

- https://github.com/alibaba/druid/tree/master/druid-spring-boot-starter
- http://www.jianshu.com/p/9f812e651319
- https://www.cnblogs.com/Alandre/p/6611813.html
- https://www.bysocket.com/?p=1712