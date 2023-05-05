## 容器环境JVM设置最佳实践

### 一. 概述

越来越多的应用项目组使用开阳容器平台对应用进行容器化的部署，其中Java类的项目比重最大。对于Java程序，JVM的设置是一个重要的环节。

### 二. JVM说明

默认情况下，jvm自动分配的heap的大小取决于机器配置，默认可用的内存是通过 MaxRAMFraction 来计算的，默认是4，即25%。例如：机器的内存是64G，那么JVM的最大heap内存就是 64 / 4 = 16G。

JVM实际使用的内存会比heap内存大

```java
JVM内存  = heap 内存 + 线程stack内存 (XSS) * 线程数 + 启动开销（constant overhead）
```

默认的XSS通常在256KB到1MB，也就是说每个线程会分配最少256K额外的内存，constant overhead是JVM分配的其他内存。

对于JVM的设置，用两种方案

方案一：设置 -Xmx

```bash
-Xms32g -Xmx32g -Xmn8g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=128m
```

-Xmn 是Young Generation，一般设置为Xmx的3或4分之一。

方案二：

- Java 8u191之前的版本

  设置 MaxRAMFraction，对应的Heap可用内存比例

```bash
-XX:MaxRAMFraction=2 -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=128m
```

```shell
+----------------+-------------------+
| MaxRAMFraction | % of RAM for heap |
|----------------+-------------------|
|              1 |              100% |
|              2 |               50% |
|              3 |               33% |
|              4 |               25% |
+----------------+-------------------+
```

- Java 8u191之后的版本（包含Java 8u191）

  设置MaxRAMPercentage，对应的Heap可用内存比例，值介于0.0到100.0之间，默认值为25.0。

```bash
-XX:MinRAMPercentage=75.0 -XX:MaxRAMPercentage=75.0 -XX:MetaspaceSize=64m -XX:MaxMetaspaceSize=128m
```

### 三. 容器环境

对于容器环境，在Java 8u191+，Java10 之前 JVM 不能识别容器设置的内存或cpu限制，只能获取到服务器的配置。 这样容易引起不必要的问题，例如：容器设置的内存限制为1G，但是jvm根据服务器配置来分配初始化内存，导致java进程超过容器限制被kill掉。为了解决这样的问题，可以设置-Xmx或者MaxRAMFraction来解决。

在Java 8u191+的版本，JVM会自动的识别cgroup限制，并强制执行内存限制。这样当容器超过内存限制时，会抛出OOM异常，而不是杀死容器。

### 四. 最佳实践

升级Java的版本到8u191+，目前数据中心的主推版本是Java 8u212，符合要求。开阳容器平台也已经提供了基于Java 8u212的 Java基础镜像和Tomcat基础镜像。

将 JAVA_OPTS 或 CATALINA_OPTS 添加到环境变量，并添加到应用程序的启动参数中。 JAVA_OPTS 或 CATALINA_OPTS 的设置用两种推荐方式

方式一：设置内存占用的比例，更加优雅

```bash
## 注意：MaxRAMPercentage 必须是double类型，例如：75.0，不能写成75
JAVA_OPTS="-XX:MinRAMPercentage=75.0 -XX:MaxRAMPercentage=75.0 -XX:MetaspaceSize=64m -XX:MaxMetaspaceSize=128m"
```

方式二：设置 -Xmx，更符合目前开发人员的习惯，不足之处是需要计算具体的内存的值

```bash
## 例如：容器的内存限制2G
JAVA_OPTS="-Xms1536m -Xmx1536m -Xmn384m -XX:MetaspaceSize=64m -XX:MaxMetaspaceSize=128m"
```



### 五. 参考资料

- https://www.cnblogs.com/xiaoqi/p/container-jvm.html
- https://merikan.com/2019/04/jvm-in-a-container/
- https://www.cnblogs.com/leeego-123/p/11572786.html
- https://www.cnblogs.com/zhangfengshi/p/11343102.html

