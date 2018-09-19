## JMX - Prometheus

1. 添加 Java 启动参数

JMX 的 host 是 127.0.0.1，port 是 5555

```sh
JMX_OPTS="-Djava.rmi.server.hostname=127.0.0.1 \
		  -Dcom.sun.management.jmxremote \
		  -Dcom.sun.management.jmxremote.port=5555 \
		  -Dcom.sun.management.jmxremote.authenticate=false \
		  -Dcom.sun.management.jmxremote.ssl=false"
JAVA_OPTS="-Xms1g -Xmx1g"
ENV_OPTS="-Dspring.profiles.active=dev"

java -jar $JAVA_OPTS $JAVA_OPTS $ENV_OPTS -jar xx.jar
## 或者
nohup java -jar $JAVA_OPTS $JAVA_OPTS $ENV_OPTS -jar xx.jar >nohup 2>&1 &
```

**JMX_exporter**

	使用 Prometheus 收集指标本地 JVM Metrics（作为代理运行），或者作为独立 HTTP Server 抓取远程 JMX 目标，参考：https://github.com/prometheus/jmx_exporter 。

1. Java 代理（推荐）

使用 [jmx_prometheus_javaagent-0.3.1.jar](JMX-Prometheus/jmx_prometheus_javaagent-0.3.1.jar) ，按如下方式启动应用 Jar。

```shell
## 8000 - 应用暴露指标的端口
## 5555 - JMX 端口
java -javaagent:./jmx_prometheus_javaagent-0.3.1.jar=8000:config.yml \
-Djava.rmi.server.hostname=127.0.0.1 \
-Dcom.sun.management.jmxremote \
-Dcom.sun.management.jmxremote.port=5555 \
-Dcom.sun.management.jmxremote.authenticate=false \
-Dcom.sun.management.jmxremote.ssl=false \
-Dspring.profiles.active=node2 \
-jar springboot-docker-0.0.1-SNAPSHOT.jar
```

- config.yaml 如下

```yaml
---
hostPort: localhost:5555
username: 
password: 
ssl: false
lowercaseOutputName: true
rules:
- pattern: ".*"
```

2. HTTP Server

- Java 应用先暴露 JMX Port（参照 1）
- 获取 jmx_prometheus_httpserver 可执行 Jar

```shell
git clone https://github.com/prometheus/jmx_exporter.git
cd jmx_exporter
mvn package
```

- 运行服务

```shell
java -jar jmx_prometheus_httpserver-0.3.2-SNAPSHOT-jar-with-dependencies.jar 8001 config.yml
```

