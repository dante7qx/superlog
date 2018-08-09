# lombok详解

​	Lombok是一个可以通过简单的注解形式来帮助我们简化消除一些必须有但显得很臃肿的Java代码的工具，通过使用对应的注解，可以在编译源码的时候生成对应的方法。

### 1. Eclipse安装

1. 下载 [lombok.jar](https://projectlombok.org/downloads/lombok.jar)

2. 运行，java -jar lombok.jar，按弹出界面安装

3. 确认安装

   在Eclipse安装目录中，多出一个lombok.jar，并且在 eclipse.ini 中加入了一行配置

    `-javaagent:../Eclipse/lombok.jar` 。

4. 重启 Eclipse。

### 2. 使用lombok

在项目中添加依赖

```xml
<dependency>
  <groupId>org.projectlombok</groupId>
  <artifactId>lombok</artifactId>
  <version>1.16.18</version>
  <scope>provided</scope>
</dependency>
```

### 3. 注解详解

#### 1. @NonNull

可以用在成员方法或者构造方法的参数前面，会自动产生一个关于此参数的非空检查，如果参数为空，则抛出一个空指针异常。

```java
//成员方法参数加上@NonNull注解
public String getName(@NonNull Person p){
    return p.getName();
}

// 等同于
public String getName(Person p){
    if(p==null){
        throw new NullPointerException("person");
    }
    return p.getName();
}
```

#### 2. @Cleanup

这个注解用在变量前面，可以保证此变量代表的资源会被自动关闭，默认是调用资源的close()方法，如果该资源有其它关闭方法，可使用@Cleanup(“methodName”)来指定要调用的方法。

```java
public static void main(String[] args) throws IOException {  
  @Cleanup InputStream in = new FileInputStream(args[0]);  
  @Cleanup OutputStream out = new FileOutputStream(args[1]);  
  byte[] b = new byte[10000];  
  while (true) {  
    int r = in.read(b);  
    if (r == -1) break;  
    out.write(b, 0, r);  
  }  
}  

// 等同于自动生成了下面的代码
public static void main(String[] args) throws IOException {  
    InputStream in = new FileInputStream(args[0]);  
    try {  
      OutputStream out = new FileOutputStream(args[1]);  
      try {  
        byte[] b = new byte[10000];  
        while (true) {  
          int r = in.read(b);  
          if (r == -1) break;  
          out.write(b, 0, r);  
        }  
      } finally {  
        if (out != null) {  
          out.close();  
        }  
      }  
    } finally {  
      if (in != null) {  
        in.close();  
      }  
    }  
  }
```

#### 3. @Getter/@Setter

用在成员变量前面，相当于为成员变量生成对应的get和set方法，同时还可以为生成的方法指定访问修饰符。

```java
@Setter(AccessLevel.PROTECTED)
private int age;
```

#### 4. 构造函数注解

**@NoArgsConstructor/@RequiredArgsConstructor /@AllArgsConstructor**

- @NoArgsConstructor，生成一个无参构造方法。当类中有final字段没有被初始化时，编译器会报错，此时可用@NoArgsConstructor(force = true)，然后就会为没有初始化的final字段设置默认值 0 / false / null。
- @AllArgsConstructor，生成一个全参数的构造方法。
- @RequiredArgsConstructor，使用类中所有带有@NonNull注解的或者带有final修饰的成员变量生成对应的构造方法。

#### 5. @Data

​	@Data 包含了@ToString，@EqualsAndHashCode，@Getter / @Setter和@RequiredArgsConstructor的功能。

#### 6. @Log

​	用在类上，可以省去从日志工厂生成日志对象这一步，直接进行日志记录，具体注解根据日志工具的不同而不同，同时，可以在注解中使用topic来指定生成log对象时的类名。（添加具体的Log依赖包）

```java
@CommonsLog
private static final org.apache.commons.logging.Log log = org.apache.commons.logging.LogFactory.getLog(LogExample.class);
@JBossLog
private static final org.jboss.logging.Logger log = org.jboss.logging.Logger.getLogger(LogExample.class);
@Log
private static final java.util.logging.Logger log = java.util.logging.Logger.getLogger(LogExample.class.getName());
@Log4j
private static final org.apache.log4j.Logger log = org.apache.log4j.Logger.getLogger(LogExample.class);
@Log4j2
private static final org.apache.logging.log4j.Logger log = org.apache.logging.log4j.LogManager.getLogger(LogExample.class);
@Slf4j
private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(LogExample.class);
@XSlf4j
private static final org.slf4j.ext.XLogger log = org.slf4j.ext.XLoggerFactory.getXLogger(LogExample.class);
```

### 4. 参考资料

- https://projectlombok.org/features/all

- http://blog.csdn.net/sunsfan/article/details/53542374
- http://www.jianshu.com/p/365ea41b3573