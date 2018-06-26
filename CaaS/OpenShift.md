## OpenShift

### 1. 本地安装

```sh
bogon:~ dante$ oc cluster up --docker-machine='openshift'
Using Docker shared volumes for OpenShift volumes
Using docker-machine IP 192.168.99.100 as the host IP
Using 192.168.99.100 as the server IP
Starting OpenShift using openshift/origin:v3.9.0 ...
OpenShift server started.

The server is accessible via web console at:
    https://192.168.99.100:8443

You are logged in as:
    User:     developer
    Password: <any value>
```

### 2. 用户认证

1. 当前 Session Token

   `oc whoami -t`

2. 认证 Token

   `https://<url>:<port>/oauth/token/request`

   ```ini
   Your API token is
   RS4pGtCc9wY-6loYtxRvwto_E7IGbu7kHWkw7osl0sg
   Log in with this token
   oc login --token=RS4pGtCc9wY-6loYtxRvwto_E7IGbu7kHWkw7osl0sg --server=https://10.70.94.90:8443
   Use this token directly against the API
   curl -H "Authorization: Bearer RS4pGtCc9wY-6loYtxRvwto_E7IGbu7kHWkw7osl0sg" "https://10.70.94.90:8443/oapi/v1/users/~"
   ```

### 3. BuildConfig

#### 3.1 概念

BuildConfig 是一个定义从 Input（参数、源码） 到 Output（可运行 Image）的描述对象。

#### 3.2 主要内容

- **Source Clone Secrets** 

  对于需要认证才能访问的 Source，例如：Private Git、自签名的HTTPS等。可以通过下面的命令对构建用户（Builder Service Account）进行授权。

  **前提是需要设置 serviceAccountConfig.limitSecretReferences = true** 。

  ```sh
  oc get secrets
  oc secrets link builder <mysecret>
  ```

  

### 4. S2I

#### 4.1. 安装 

（https://github.com/openshift/source-to-image/blob/master/README.md#installation）

```sh
brew install source-to-image
```

#### 4.2. 构建流程

- 要素

  源代码 (source)、S2I脚本 (s2i scripts)、镜像构造器 (builder image)。

- 基理：

  S2I 必须将 **source** 和 **s2i scripts** 放到 **builder image** 中。首先 S2I 创建一个包含 **source** 和 **s2i scripts** 的 tar 包，并以 file stream 的形式传输到 **builder image** 中。执行 assemble script 前，S2I 会解压 tar 包，并根据 **builder image** 中的  `--destination` flag  或者  label `io.openshift.s2i.destination ` 指定的位置。默认位于 /tmp 下。

![S2I 流程](/Users/dante/Documents/Technique/且行且记/CaaS/S2I 流程.png)

#### 3. 工作原理

参考：

- https://github.com/jorgemoralespou/s2i-java
- https://github.com/openshift/source-to-image
- https://github.com/ganrad/openshift-s2i-springboot-java
- https://github.com/openshift-s2i/s2i-wildfly

原理

- assemble

  ​	将外部代码库下载到本地，并编译打包。通过定义 **save-artifacts**，可以进行增量构建（mvn、npm缓存问题）。

- run 

  ​	运行 assemble 编译好的应用程序包。

- save-artfacts

  ​	save-artifacts脚本负责将构建所需要的所有依赖包收集到一个tar文件中。

  **增量构建**

  - 必须条件
    1. save-artfacts 脚本必须存在
    2. 之前构建的**Image**必须存在
    3. s2i build 时，必须添加 --incremental=true 
  - 工作流
    1. 先拉取上一次构建后的 Image，基于此 Image 创建一个新的 Container。
    2. 运行  save-artfacts 脚本，将要持久化的内容打包并输出到Stdout。例 `tar cf - ./.m2`。
    3. 构建新的 Image，assemble 脚本中去使用 artifacts 目录中的内容。

- usage

  ​	使用说明文档。

  ```bash
  #!/bin/bash -e
  cat <<EOF
  开始你的说明......
  EOF
  ```

- test / run

  测试。

#### 4. 示例

##### 4.1 Java

**环境变量参数**

- APP_OPTIONS

  java 运行时的参数，例如: `-Xms2048m -Xmx2048m` 。

- JAR_PATH

  可以执行 Jar 打包后的位置，例：`s2i-demo/target` 。

- MAVEN_ARGS_APPEND

  `mvn package -Dmaven.test.skip=true ${MAVEN_ARGS_APPEND}`

**Dockerfile**

```dockerfile
# dante/s2i-java:v1

FROM openshift/base-centos7

LABEL MAINTAINER="dante <ch.sun@haihangyun.com>" \
			io.k8s.description="Platform for building Springboot app" \
      io.k8s.display-name="Spring Boot builder v1" \
      io.openshift.expose-services="8080:http" \
			io.openshift.tags="Java,Springboot,builder"	\
	    io.openshift.s2i.destination="/opt/s2i/destination"

ENV BUILDER_VERSION=v1 \
	MAVEN_VERSION=3.3.9 \
	JAVA_HOME=/opt/jdk1.8.0_131 \
	M2_HOME=/opt/apache-maven-3.3.9 \
	TZ=Asia/Shanghai

# Set PATH
ENV PATH $JAVA_HOME/bin:$M2_HOME/bin:$PATH
# Set the default build type to 'Maven'
ENV BUILD_TYPE=Maven \
		JAR_PATH="$HOME/target"

## Install Java Maven
ADD ./jdk-8u131-linux-x64.tar.gz /opt
ADD ./apache-maven-$MAVEN_VERSION-bin.tar.gz /opt

RUN rm -rf /opt/jdk1.8.0_131/src.zip && rm -rf /opt/jdk1.8.0_131/javafx-src.zip && \
	ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone && \
	ln -sf /opt/apache-maven-$MAVEN_VERSION/bin/mvn /usr/local/bin/mvn && \
	mkdir -p $HOME/.m2 && mkdir -p $STI_SCRIPTS_PATH && mkdir -p /opt/s2i/destination && \
	mkdir -p /opt/appserver 

COPY ./settings.xml /opt/apache-maven-$MAVEN_VERSION/conf/settings.xml
COPY ./settings.xml $HOME/.m2/settings.xml
COPY ./s2i/bin/ $STI_SCRIPTS_PATH

RUN chown -R 1001:0 $HOME /opt/appserver && \
    chmod -R 755 $STI_SCRIPTS_PATH && \
		chmod -R 777 /opt/appserver && \
    chmod -R g+rw /opt/s2i/destination

USER 1001

EXPOSE 8080

CMD $STI_SCRIPTS_PATH/usage
```

**assemble**

```shell
#!/bin/bash -e

. $(dirname $0)/functions

echo "--> S2I:assemble step start ..."
echo "--> Executing script as user=" + `id`

if [ "$1" = "-h" ]; then
  exec /usr/libexec/s2i/usage
fi

STI_DESTINATION=/opt/s2i/destination
DEPLOY_DIR=/opt/appserver

manage_incremental_build

echo "---> Starting Java web application build process ..."
echo "---> Application source directory is set to $HOME ..."
echo "---> Set target directory to $DEPLOY_DIR ..."

cp -Rf $STI_DESTINATION/src/. ./
echo "---> Copied application source to $HOME ..."
ls -la $HOME

echo "---> S2I:assemble Build type=$BUILD_TYPE ..."
if [ $BUILD_TYPE = "Maven" ] && [ -f "$HOME/pom.xml" ]; then
  execute_maven_build
else
  # Copy the fat jar to the deployment directory
  cp -v $HOME/*.jar $DEPLOY_DIR 2> /dev/null
fi

echo "---> Rename *.jar to app.jar"
if [ $(ls $DEPLOY_DIR/*.jar | wc -l) -eq 1 ]; then
  mv $DEPLOY_DIR/*.jar $DEPLOY_DIR/app.jar
  [ ! -f $DEPLOY_DIR/app.jar ] && echo "Application could not be properly built." && exit 1 
  echo "---> Application deployed successfully.  jar file is located in $DEPLOY_DIR/app.jar"
else
  exit 1
fi
```

**run**

```shell
#!/bin/bash -e
exec java -Djava.security.egd=file:/dev/./urandom -jar /opt/appserver/app.jar $APP_OPTIONS
```

**save-artifacts**

```sh
#!/bin/sh -e

pushd ${HOME} >/dev/null
tar cf - ./.m2
popd >/dev/null
```

##### 4.2 Tomcat（8.0.x 和 8.5.x）



##### 4.3 Node