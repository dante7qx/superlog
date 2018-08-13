## Quartz

### 一. 概念

​	Quartz是一个由Java编写的开源作业调度框架。为在Java应用程序中进行作业调度提供了简单却强大的机制，Quartz允许开发人员根据时间间隔来调度作业。它实现了作业和触发器的多对多的关系，还能把多个作业与不同的触发器关联。

​	最小使用依赖 quartz 和 quartz-jobs，slf4j 和 c3p0（存储调度数据到database中）

```xml
 <dependency>
      <groupId>org.quartz-scheduler</groupId>
      <artifactId>quartz</artifactId>
      <version>2.3.0</version>
  </dependency>
  <dependency>
      <groupId>org.quartz-scheduler</groupId>
      <artifactId>quartz-jobs</artifactId>
      <version>2.3.0</version>
  </dependency> 
```

### 二. 核心元素

- Scheduler： 任务调度器，由 SchedulerFactory 创建。
  - StdSchedulerFactory，默认的SchedulerFactory。
  - DirectSchedulerFactory，SchedulerFactory的直接实现，通过它可以直接构建Scheduler、Threadpool 等。
  - StdScheduler， Quartz默认的Scheduler。
  - RemoteScheduler，带有RMI功能的Scheduler。
- Job：任务，即被 **Scheduler** 调度的任务。
- JobDetail：Job实例的具体描述。
- Trigger：触发器，定义由 Scheduler 调度 Job 的时间规则。
  - SimpleTrigger <普通的Trigger> —  SimpleScheduleBuilder，可以定义作业的启动时间、触发器之间的延迟时间以及 repeatInterval(频率)。
  - CronTrigger  <带Cron Like 表达式的Trigger> — CronScheduleBuilder
  - CalendarIntervalTrigger <带日期触发的Trigger> — CalendarIntervalScheduleBuilder
  - DailyTimeIntervalTrigger <按天触发的Trigger> — DailyTimeIntervalScheduleBuilder
- JobBuilder：用于定义/构建 JobDetail 实例。
- TriggerBuilder：用于定义/构建 Trigger 实例。

![Quartz](/Users/dante/Documents/Project/dante-quartz/Quartz.png)

### 三. JDBC存储

​	所有数据通过JDBC存起来，建表SQL在 **org.quartz.impl.jdbcjobstore**包下。事务选择，

- **JobStoreTX**：org.quartz.impl.jdbcjobstore.JobStoreTX。
- **JobStoreJMT**：org.quartz.impl.jdbcjobstore.JobStoreCMT，需要Quartz和其他事务一起工作，应该使用JobStoreCMT，这样Quartz就会将事务的管理权移交给其他的app server容器。

```properties
## JDBC Job Store
org.quartz.jobStore.class: org.quartz.impl.jdbcjobstore.JobStoreTX
org.quartz.jobStore.driverDelegateClass: org.quartz.impl.jdbcjobstore.StdJDBCDelegate
## 所有JobDataMaps中的值都以String的形式存储，这样就可以使用name-value的形式存储，而避免使用复杂的格式存储（例如BLOB）
org.quartz.jobStore.useProperties: true
org.quartz.jobStore.tablePrefix: QRTZ_
org.quartz.jobStore.isClustered: false 

## Datasource, 配置查看 org.quartz.utils.C3p0PoolingConnectionProvider
org.quartz.jobStore.dataSource: xDS
org.quartz.dataSource.xDS.driver: com.mysql.cj.jdbc.Driver
org.quartz.dataSource.xDS.URL: jdbc:mysql://localhost:3306/dante_quartz?useUnicode=true&characterEncoding=utf8 
org.quartz.dataSource.xDS.user: root 
org.quartz.dataSource.xDS.password: iamdante
## 最大连接数，最小值 org.quartz.threadPool.threadCount + 2
org.quartz.dataSource.xDS.maxConnections: 10 
org.quartz.dataSource.xDS.validationQuery: select 1
## 每60秒检查所有连接池中的空闲连接（只有 validationQuery 设置了才会生效，默认是50）
#org.quartz.dataSource.xDS.idleConnectionValidationSeconds: 60
## 每次查询都去连接池中验证当前连接的有效性
#org.quartz.dataSource.xDS.validateOnCheckout: false
## 当连接空闲时间超过指定时间后，丢弃这个连接（0表示永不丢失，默认是0）
#org.quartz.dataSource.xDS.discardIdleConnectionsSeconds: 0

## 自定义DB连接Provider
#org.quartz.dataSource.NAME.connectionProvider.class: 
```

### 三. 集群

​	Quartz的集群功能通过故障转移和负载平衡功能为您的调度程序带来高可用性和可扩展性。目前，群集仅适用于JDBC-Jobstore（JobStoreTX或JobStoreCMT），并且基本上通过使群集的每个节点共享相同的数据库来工作。

​						![Cluster](/Users/dante/Documents/Project/dante-quartz/Cluster.png)

- 负载均衡：自动发生，群集的每个节点都尽可能快地触发jobs。当Triggers的触发时间发生时，获取它的第一个节点（独占锁）是将触发它的节点。每次任务触发，只有一个节点负责执行。

- 故障切换：当其中一个节点在执行一个或多个作业期间失败时发生故障切换，当节点出现故障时，其他节点会检测到该状况并识别数据库中在故障节点内正在进行的作业。任何标记为恢复的作业（在JobDetail上都具有“请求恢复”属性）将被剩余的节点重新执行。没有标记为恢复的作业将在下一次相关的Triggers触发时简单地被释放以执行。

- 适用场景

  ​	适合扩展长时间运行、cpu密集型作业（通过多个节点分配工作负载）。如果需要扩展以支持数千个短期运行（例如1秒）作业，则可以考虑通过使用多个不同的调度程序（包括HA的多个群集调度程序）对作业集进行分区。调度程序使用集群范围的锁，这种模式会在添加更多节点（超过三个节点 - 取决于数据库的功能等）时降低性能。

- 配置

```properties
org.quartz.scheduler.instanceId: AUTO
org.quartz.scheduler.instanceName: DanteClusteredScheduler
org.quartz.threadPool.class: org.quartz.simpl.SimpleThreadPool
org.quartz.threadPool.threadCount: 10
org.quartz.threadPool.threadPriority: 5
org.quartz.jobStore.misfireThreshold: 60000
org.quartz.jobStore.class: org.quartz.impl.jdbcjobstore.JobStoreTX
org.quartz.jobStore.driverDelegateClass: org.quartz.impl.jdbcjobstore.StdJDBCDelegate
org.quartz.jobStore.useProperties: true
org.quartz.jobStore.tablePrefix: QRTZ_
org.quartz.jobStore.isClustered: true
org.quartz.jobStore.clusterCheckinInterval: 20000
org.quartz.jobStore.dataSource: xDS
org.quartz.dataSource.xDS.driver: com.mysql.jdbc.Driver
org.quartz.dataSource.xDS.URL: jdbc:mysql://localhost:3306/dante_quartz_cluster?useUnicode=true&characterEncoding=utf8 
org.quartz.dataSource.xDS.user: root 
org.quartz.dataSource.xDS.password: iamdante
org.quartz.dataSource.xDS.maxConnections: 15
org.quartz.dataSource.xDS.validationQuery: select 1
```

### 四. Springboot动态定时任务

​	Spring集成quartz必须添加 spring-context-support 依赖，里面有Spring对quartz的支持。通过 SchedulerFactoryBean 来配置和创建 org.quartz.Scheduler。具体步骤如下

1. 添加依赖

```xml
<properties>
  <quartz.version>2.3.0</quartz.version>
</properties>
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-context-support</artifactId>
</dependency>
<dependency>
  <groupId>org.quartz-scheduler</groupId>
  <artifactId>quartz</artifactId>
  <version>${quartz.version}</version>
</dependency>
<dependency>
  <groupId>org.quartz-scheduler</groupId>
  <artifactId>quartz-jobs</artifactId>
  <version>${quartz.version}</version>
</dependency>
```

1. 定义 FactoryBean，在Spring的上下文中管理**Quertz Scheduler**的生命周期，可以将 **Scheduler** 作为一个Bean。在运行时，通过注入 SchedulerFactoryBean可以直接操作**Quartz Scheduler**来操作 jobs 和 triggers。

```java
@Component("spiritJobFactory")
public class JobFactory extends SpringBeanJobFactory implements ApplicationContextAware {
	private transient AutowireCapableBeanFactory beanFactory;
	@Override
	public void setApplicationContext(ApplicationContext context) throws BeansException {
		beanFactory = context.getAutowireCapableBeanFactory();
	}
	@Override
	protected Object createJobInstance(final TriggerFiredBundle bundle) throws Exception {
		final Object job = super.createJobInstance(bundle);
		// 将 job 加入 Spring 容器，Spring的Bean都可以在Job中直接注入
		beanFactory.autowireBean(job);
		return job;
	}
}
```

1. 配置

```java
@Configuration
public class SchedulerConfig {
	@Autowired
	@Qualifier("spiritJobFactory")
    private JobFactory jobFactory;
	
	@Bean(name="spiritSchedulerFactory")
    public SchedulerFactoryBean schedulerFactoryBean() throws IOException {
        SchedulerFactoryBean factory = new SchedulerFactoryBean();
        factory.setQuartzProperties(quartzProperties());
        factory.setOverwriteExistingJobs(true);
        factory.setJobFactory(jobFactory);
        return factory;
    }
	@Bean
    public Properties quartzProperties() throws IOException {
        PropertiesFactoryBean propertiesFactory = new PropertiesFactoryBean();
        propertiesFactory.setLocation(new ClassPathResource("/quartz.properties"));
        // 手动初始化 Properties，不加下面这句quartz.properties配置无效
        propertiesFactory.afterPropertiesSet();
        return propertiesFactory.getObject();
    }
}
```

1. 具体Job管理类

```java
@Service
public class SpiritSchedulerService {
  	// Job 所在包
	public static final String JOB_PACKAGE = "org.dante.springboot.quartz.job";
	private static final String JOB_GROUP = "SPIRIT_JOB_GROUP";
	private static final String TRIGGER_GROUP = "SPIRIT_TRIGGER_GROUP";
	
	@Autowired
	@Qualifier("spiritSchedulerFactory")
	private SchedulerFactoryBean schedulerFactory;
	@Autowired
	private SpiritJobListener spiritJobListener;

	/**
	 * 新增定时任务（默认启动）
	 * 
	 * @param jobId
	 * @param jobClazz
	 * @param cron
	 * @param startTime
	 */
	public void addJob(String jobId, String jobClazz, String cron, Date startTime) {
		addJob(jobId, jobClazz, cron, startTime, true);
	}
	
	/**
	 * 新增定时任务
	 * 
	 * @param jobId
	 * @param jobClazz 必须实现 org.quartz.Job 接口
	 * @param cron
	 * @param startTime
	 * @param startJob
	 */
	@SuppressWarnings({ "rawtypes", "unchecked" })
	public void addJob(String jobId, String jobClazz, String cron, Date startTime, boolean startJob) {
		try {
			Scheduler scheduler = schedulerFactory.getScheduler();
			Class jobClass = Class.forName(jobClazz);
			JobDetail jobDetail = JobBuilder.newJob(jobClass).withIdentity(jobId, JOB_GROUP).build();// 设置Job的名字和组
			TriggerBuilder<Trigger> triggerBuilder = TriggerBuilder.newTrigger();
			triggerBuilder.withIdentity(jobId, TRIGGER_GROUP);
			if(startTime == null || new Date().compareTo(startTime) > 0) {
				triggerBuilder.startNow();
			} else {
				triggerBuilder.startAt(startTime);
			}
			triggerBuilder.withSchedule(CronScheduleBuilder.cronSchedule(cron));
			CronTrigger trigger = (CronTrigger) triggerBuilder.build();
			scheduler.scheduleJob(jobDetail, trigger);
			scheduler.getListenerManager().addJobListener(spiritJobListener);
			if(!startJob) {
				pauseJob(jobId);
			}
		} catch (Exception e) {
			throw new RuntimeException(e);
		}
	}

	/**
	 * 修改一个任务的触发时间
	 * 
	 * @param jobId
	 * @param cron
	 * @param startTime
	 */
	public void updateJobCron(String jobId, String cron, Date startTime) {
		try {
			Scheduler scheduler = schedulerFactory.getScheduler();
			TriggerKey triggerKey = TriggerKey.triggerKey(jobId, TRIGGER_GROUP);
			CronTrigger trigger = (CronTrigger) scheduler.getTrigger(triggerKey);
			if (trigger == null) {
				return;
			}
			String oldTime = trigger.getCronExpression();
			if (!oldTime.equalsIgnoreCase(cron)) {
				TriggerBuilder<Trigger> triggerBuilder = TriggerBuilder.newTrigger();
				triggerBuilder.withIdentity(jobId, TRIGGER_GROUP);
				if(startTime == null || new Date().compareTo(startTime) > 0) {
					triggerBuilder.startNow();
				} else {
					triggerBuilder.startAt(startTime);
				}
				triggerBuilder.withSchedule(CronScheduleBuilder.cronSchedule(cron));
				trigger = (CronTrigger) triggerBuilder.build();
				scheduler.rescheduleJob(triggerKey, trigger);
			}
		} catch (Exception e) {
			throw new RuntimeException(e);
		}
	}
	
	/**
	 * 移除指定jobId的定时任务
	 * 
	 * @param jobId
	 */
	public void removeJob(String jobId) {
		try {
			Scheduler scheduler = schedulerFactory.getScheduler();
			TriggerKey triggerKey = TriggerKey.triggerKey(jobId, TRIGGER_GROUP);
			scheduler.pauseTrigger(triggerKey);
			scheduler.unscheduleJob(triggerKey);
			scheduler.deleteJob(JobKey.jobKey(jobId, JOB_GROUP));
		} catch (Exception e) {
			throw new RuntimeException(e);
		}
	}
	
	/**
	 * 暂停指定jobId的定时任务
	 * 
	 * @param jobId
	 */
	public void pauseJob(String jobId) {
		try {
			Scheduler scheduler = schedulerFactory.getScheduler();
			JobKey jobKey = JobKey.jobKey(jobId, JOB_GROUP);  
			scheduler.pauseJob(jobKey);
		} catch (Exception e) {
			throw new RuntimeException(e);
		}
	}
	
	/**
	 * 恢复指定jobId的定时任务
	 * 
	 * @param jobId
	 */
	public void resumeJob(String jobId) {
		try {
			Scheduler scheduler = schedulerFactory.getScheduler();
			JobKey jobKey = JobKey.jobKey(jobId, JOB_GROUP);  
			scheduler.resumeJob(jobKey);
		} catch (Exception e) {
			throw new RuntimeException(e);
		}
	}
	
	/**
	 * 启动所有定时任务
	 */
	public void startJobs() {
		try {
			Scheduler sched = schedulerFactory.getScheduler();
			sched.start();
		} catch (Exception e) {
			throw new RuntimeException(e);
		}
	}

	/**
	 * 关闭所有定时任务
	 */
	public void shutdownJobs() {
		try {
			Scheduler sched = schedulerFactory.getScheduler();
			if (!sched.isShutdown()) {
				sched.shutdown();
			}
		} catch (Exception e) {
			throw new RuntimeException(e);
		}
	}
	
}
```

1. 延伸思考 AutowireCapableBeanFactory

   AutowireCapableBeanFactory，作用：让不受spring管理的类具有spring自动注入的特性。

   ```java
   public class MyServletContextListener implements ServletContextListener {
    
       @Autowired
       private HelloService helloService; // 无法注入，使用 AutowireCapableBeanFactory
    
       @Override
       public void contextDestroyed(ServletContextEvent event) {
    
       }
    
       @Override
       public void contextInitialized(ServletContextEvent event) {
           AutowireCapableBeanFactory autowireCapableBeanFactory = WebApplicationContextUtils.getRequiredWebApplicationContext(event.getServletContext()).getAutowireCapableBeanFactory();
           autowireCapableBeanFactory.autowireBean(this);
       }
   }
   
   ```

### 五. 参考资料

- http://www.quartz-scheduler.org/
- https://www.w3cschool.cn/quartz_doc/quartz_doc-d8pn2do9.html
- http://www.cnblogs.com/fanshuyao/p/7484843.html
- http://blog.csdn.net/liuchuanhong1/article/details/78543574
- http://blog.csdn.net/liuchuanhong1/article/details/60873295
- https://articles.microservices.com/designing-a-cron-scheduler-microservice-18a52471d13f
- http://blog.csdn.net/beliefer/article/details/51578546
- https://wuxinshui.github.io/spring%20boot/2017/08/28/Spring-Boot%E9%9B%86%E6%88%90Quartz-%E5%8A%A8%E6%80%81%E4%BB%BB%E5%8A%A1%E7%AE%A1%E7%90%86.html