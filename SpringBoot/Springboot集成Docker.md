## 集成 Docker

使用 Spotify 的插件 docker-maven-plugin。

版本使用说明

**0.4.14（老版本）**

https://github.com/spotify/docker-maven-plugin

https://spring.io/guides/gs/spring-boot-docker/

1. 添加插件

```xml
<properties>
   <docker.image.prefix>dante2012</docker.image.prefix>
</properties>
<build>
    <plugins>
        <plugin>
            <groupId>com.spotify</groupId>
            <artifactId>docker-maven-plugin</artifactId>
            <version>0.4.14</version>
            <configuration>
                <!-- 如果要将docker镜像push到DockerHub上去的话，这边的路径要和repo路径一致 -->
                <imageName>${docker.image.prefix}/${project.artifactId}</imageName>
                <dockerDirectory>src/main/docker</dockerDirectory>
                <resources>
                    <resource>
                        <targetPath>/</targetPath>
                        <directory>${project.build.directory}</directory>
                        <include>${project.build.finalName}.jar</include>
                    </resource>
                </resources>
                <!-- 以下两行是为了docker push到DockerHub使用的。 -->
                <!--
                <serverId>docker-hub</serverId>
                <registryUrl>https://index.docker.io/v1/</registryUrl>
				-->
            </configuration>
        </plugin>
    </plugins>
</build>
```

2. 在 src/main/docker 目录下创建 Dockerfile

```dockerfile
# Dockerfile to create Spring boot docker image
# Base image
FROM java:8

LABEL MAINTAINER="Dante <ch.sun@hnair.com>"

ENV JAVA_OPTS="-Xms1g -Xmx1g" \
	TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

VOLUME ["/logs"]

ADD springboot-docker-0.0.1-SNAPSHOT.jar app.jar

# 使 app.jar 具有文件修改时间（默认情况下，Docker创建所有容器文件处于“未修改”状态）
RUN sh -c 'touch /app.jar'

# -Djava.security.egd=file:/dev/./urandom -> 加快 Tomcat 启动速度
# 默认是 file:/dev/random，使用 /dev/random 产生随机数的方式必须要保证熵足够大，才能够产生足够的随机数支持连接，否则系统就会产生等待，直到有足够的随机数再进行连接，这样就有了延时。

EXPOSE 8080

ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar /app.jar" ]
```

3. 执行命令

```shell
mvn clean package -Dmaven.test.skip=true docker:build
```

**1.4.9**

https://github.com/spotify/dockerfile-maven

1. 添加插件

```xml
<build>
    <plugins>
        <plugin>
            <groupId>com.spotify</groupId>
            <artifactId>dockerfile-maven-plugin</artifactId>
            <version>1.4.9</version>
            <executions>
                <execution>
                    <id>default</id>
                    <goals>
                        <goal>build</goal>
                    </goals>
                </execution>
            </executions>
            <configuration>
                <repository>${docker.image.prefix}/${project.artifactId}</repository>
                <tag>${project.version}</tag>
                <buildArgs>
                    <JAR_FILE>target/${project.build.finalName}.jar</JAR_FILE>
                </buildArgs>
            </configuration>
        </plugin>
    </plugins>
</build>
```

2. 在根目录下创建 Dockerfile

```dockerfile
FROM java:8
LABEL MAINTAINER="Dante <ch.sun@hnair.com>"
ENV JAVA_OPTS="-Xms1g -Xmx1g" \
	TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
## Spring Boot 使用的内嵌 Tomcat 容器默认使用/tmp作为工作目录
VOLUME ["/tmp"]
ARG JAR_FILE
ADD ${JAR_FILE} /app/app.jar
WORKDIR /app/
EXPOSE 8801
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","app.jar"]
```

3. 执行命令

```shell
## MacOS下，打开 Docker API
docker pull bobrik/socat
docker run -d --name dante-docker-api -p 2376:2375 bobrik/socat:latest
export DOCKER_HOST=tcp://127.0.0.1:2376

## Maven 命令，https://github.com/spotify/dockerfile-maven/blob/master/docs/usage.md
maven clean package -Dmaven.test.skip=true ## 自动执行构建镜像

## 忽略构建镜像
mvn clean package -Ddockerfile.skip
```

