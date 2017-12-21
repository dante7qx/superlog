## Spring Batch

### 一. 概念

​	SpringSource和Accenture（埃森哲) 合作，基于Java开发的一套用于处理企业级批处理框架。Spring batch为处理大批量数据提供了很多必要的可重用功能，比如日志追踪、事务管理、job执行统计、重启job和资源管理等。同时它也提供了优化和分片技术用于实现高性能的批处理任务。

​	批处理，即在大型企业中，由于业务复杂、数据量大、数据格式不同、数据交互格式繁杂，并非所有的操作都能通过交互界面进行处理。而有一些操作需要定期读取大批量的数据，然后进行一系列的后续处理。

#### 使用场景

- 定期提交批处理任务
- 并发批处理：并行执行任务（Stateless、Stateful）
- 分阶段，企业消息驱动处理
- 高并发批处理任务
- 失败后手动或定时重启
- 按顺序处理任务依赖(使用工作流驱动的批处理插件)
- Partial processing：skip records
- 完整的批处理事务：小数据量的批处理或存储过程/脚本
- 基于Web的管理员接口，Spring batch admin

#### 架构

![总体架构](/Users/dante/Documents/Technique/且行且记/SpringBoot/springbatch/总体架构.png)

- 应用程序：使用SpringBatch编写的job和业务逻辑。
- Batch Core：包括 JobRepository、JobLaunch、Job、Step、Execution、Listener等。
- Batch Infrastructure：包括 ItemReader、ItemProcessor、ItemWriter、Validator、ExecutionContext、

#### JobScope

可以随时为对象注入当前Job实例的上下文信息。只要我们指定Bean的scope为job scope，那么就可以随时使用jobParameters和jobExecutionContext等信息。(StepScope 同 JobScope)

```java
@Component
@JobScope
public class CustomClass {
    @Value("#{jobParameters[jobDate]}")
    private String jobDate;
 
    @Value("#{jobExecutionContext['input.name']}.")
    private String fileName;
}
```

**基本架构**

![简单架构](/Users/dante/Documents/Technique/且行且记/SpringBoot/springbatch/简单架构.png)

### 二. Job

Step的容器，实际要执行的任务。Job ——> * JobInstance（* JobParameters） ——> JobExecution。

配置依赖：1、name；2、JobRespository；3、Step

##### **JobInstance**

每个Job运行时产生的实例，通过 JobParameters 确定唯一性。即 `JobInstance = Job + JobParameters ` 。是

##### JobExecution

存储 Job 运行时发生的信息，并且也存储 Properties 信息。

##### JobParameters

启动一个 Job 时，需要的参数集（例如：启动时间）。

### 三. Step

​	步骤，控制批处理的具体逻辑。**StepExecution**，每次运行 **Step** 时都会创建一个新的**StepExecution**，类似于JobExecution。==（不同点： if a step fails to execute because the step before it fails, there will be no execution persisted for it. ）== **StepExecution** 只有在**Step**实际启动时才会创建。

##### ExecutionContext

​	key/value对的对象，用于保存需要记录的上下文信息。scope是包括StepExecution和JobExecution。可以保存执行过程的信息，用于故障时重新执行和继续执行，甚至状态回滚等等，不过需要用户自行记录。另外：每个JobExectuion都有一个ExecutionContext，每个StepExecution也有一个独立的ExecutionContext。

Step中的context是在每个提交点上被保存，而job的context会在两个Step执行之间被保存。

##### Chunk 机制

​	每次读取一条数据，再处理一条数据，累积到一定数量后再一次性交给writer进行写入操作。这样可以最大化的优化写入效率，整个事务也是基于Chunk来进行。

 - File 和 DB 需设置 Chunk
- WebService 则Chunk=1。这样既可以及时的处理写入，也不会由于整个Chunk中发生异常后，在重试时出现重复调用服务或者重复发送消息的情况。

```java
@Bean
public Step step1() {
    return stepBuilderFactory
      .get("step1")
      .<PersonPO, PersonPO>chunk(10)	// 每次处理的数据 10 条
      .reader(reader())
      .processor(compositePersonItemProcessor())
      .writer(writer())
      .listener(new ChunkNotiListener())
      .listener(new Step1Listener())
      .build();
}
```

##### TaskLetStep

​	对于没有输入输出的场景，例如：存储过程。

```java
public interface Tasklet {
  	/**
  	 * 每个 TastLet 的调用都包含在一个事务中
  	 * 也可实现： TaskletAdapter
  	 **/
  	RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception;
}
```

例子：

```java
public class FileDeletingTasklet implements Tasklet {
	private Resource directory;
	public FileDeletingTasklet(String filePath) {
		this.directory = new FileSystemResource(filePath);
	}
	@Override
	public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
		File dir = directory.getFile();
        File[] files = dir.listFiles();
        if(files == null) {
			return RepeatStatus.FINISHED;
		}
        for (int i = 0; i < files.length; i++) {
            boolean deleted = files[i].delete();
            if (!deleted) {
                throw new UnexpectedJobExecutionException("Could not delete file " + files[i].getPath());
            }
        }
        dir.deleteOnExit();
        return RepeatStatus.FINISHED;
	}
}
```

##### Step Flow

- 顺序执行

  ```java
  @Bean
  public Job reportJob() {
      return jobBuilderFactory
        .get("reportJob")
        .incrementer(new RunIdIncrementer())
        .listener(listener)
        .flow(step1())
        .next(step2())
        .end()
        .build();
  }
  ```

- 条件流程

  ```java
  // "*" will zero or more characters
  // "?" will match exactly one character
  @Bean
  public Job reportJob() {
      return jobBuilderFactory
        .get("reportJob")
        .incrementer(new RunIdIncrementer())
        .listener(listener)
        .flow(step1())
        .on("*").to(step2())
        .on("FAILED").to(step3())
        .end()
      .build();
  }
  ```

-  Programmatic Flow Decisions

   ```java
   public class ReportDecider implements JobExecutionDecider {
       @Override
       public FlowExecutionStatus decide(JobExecution jobExecution, StepExecution stepExecution) {
           if (report.isExist()) {
               return new FlowExecutionStatus(“SEND");
           }
           return new FlowExecutionStatus(“SKIP");
       }
   }

   @Bean
   public Job job() {
       return new JobBuilder("petstore")
           .start(orderProcess())
           .next(reportDecider)
           .on("SEND").to(sendReportStep)
           .on("SKIP").end().build()
           .build();
   }
   ```

   ​

### 四. JobRepository

注册Job的容器，持久化整个 Job 的生命周期。

```java
@Bean
public JobRepository jobRepository(DataSource dataSource, PlatformTransactionManager transactionManager) throws Exception {
    JobRepositoryFactoryBean jobRepositoryFactory = new JobRepositoryFactoryBean();
    jobRepositoryFactory.setDataSource(dataSource);
    jobRepositoryFactory.setTransactionManager(transactionManager);
    return jobRepositoryFactory.getObject();
}
```

### 五. JobLauncher

JobLauncher是用于用指定的JobParameters加载Job的接口。

```java
@Bean
public SimpleJobLauncher jobLauncher(DataSource dataSource, PlatformTransactionManager transactionManager) throws Exception {
    SimpleJobLauncher jobLauncher = new SimpleJobLauncher();
    jobLauncher.setJobRepository(jobRepository(dataSource, transactionManager));
    return jobLauncher;
}
```

### 六. Item Reader

批处理从来源（文件、XML、DB等）获取输入Input。

### 七. Item Writer

将Input的数据按照指定的方式输出Output。

### 八. Item Processor

从**ItemReader**获取Input的数据，经过业务处理**ItemProcessor**后，返回给**ItemWriter**输出。

### 九. JobParametersIncrementer

Job的参数，区分不同的 JobInstance。

```java
public class SampleIncrementer implements JobParametersIncrementer {
    public JobParameters getNext(JobParameters parameters) {
        if (parameters==null || parameters.isEmpty()) {
            return new JobParametersBuilder().addLong("run.id", 1L).toJobParameters();
        }
        long id = parameters.getLong("run.id",1L) + 1;
        return new JobParametersBuilder().addLong("run.id", id).toJobParameters();
    }
}
```

### 十. Java Config

在配置类上添加 `@EnableBatchProcessing` ，下面的 Bean 是可以自动装载的

- `JobRepository` - bean name "jobRepository"
- `JobLauncher` - bean name "jobLauncher"
- `JobRegistry` - bean name "jobRegistry"
- `PlatformTransactionManager` - bean name "transactionManager"
- `JobBuilderFactory` - bean name "jobBuilders"
- `StepBuilderFactory` - bean name "stepBuilders"

### 十一. 高级应用

### 二十. 参考资料

- https://docs.spring.io/spring-batch/reference/htmlsingle


- http://blog.jobbole.com/109857/


- https://kimmking.gitbooks.io/springbatchreference