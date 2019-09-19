## OpenShift

### 一. 开发

#### 1. 本地安装

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

#### 2. 用户认证

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

#### 3. BuildConfig

##### 3.1 概念

BuildConfig 是一个定义从 Input（参数、源码） 到 Output（可运行 Image）的描述对象。

##### 3.2 主要内容

- **Source Clone Secrets** 

  对于需要认证才能访问的 Source，例如：Private Git、自签名的HTTPS等。可以通过下面的命令对构建用户（Builder Service Account）进行授权。

  **前提是需要设置 serviceAccountConfig.limitSecretReferences = true** 。

  ```sh
  oc get secrets
  oc secrets link builder <mysecret>
  ```

- **Image Stream**

  在 Openshift 中，可以通过 is + tag 去引用特定的 Docker Image，相当于 Docker repository。IS 的 Metadata 存储在 etcd 中。

  ```yaml
  spec:
    tags:
      from:
        kind: ImageStreamTag
        name: "20180621214733"
  status:
    dockerImageRepository: docker-registry.default.svc:5000/dante/wt-portal
    tags:
    - items:
    	dockerImageReference: harbor.testos39.com/dante/wt-portal@sha256:4ba00276e67b2c233b6ec98bce04590e60e85e155b6bad8a73354fb2575ccb3b
        generation: 3
        ## Docker image ID
        image: sha256:4ba00276e67b2c233b6ec98bce04590e60e85e155b6bad8a73354fb2575ccb3b
      tag: "20180621214733"
  ```

  - Image Stream Image

  ```powershell
  <image-stream-name>@<image-id>
  ```


##### 3.3 CLI

```shell
## 创建 Java Image Stream
## 确保 is 的 name、tag 和 image 的 name、tag 保持一致
oc import-image java:8 --from=harbor.testos39.com/openshift/java:8 --insecure=true --confirm=true --scheduled=true

oc import-image tomcat:8.0 --from=harbor.testos39.com/openshift/tomcat:8.0 --insecure=true --confirm=true --scheduled=true


## CaaS 中，oc edit is java，加入 annotation（主要是 builder）
- annotations:
    iconClass: icon-java
    tags: builder,java
```

#### 4. S2I

##### 4.1. 安装 

（https://github.com/openshift/source-to-image/blob/master/README.md#installation）

```sh
brew install source-to-image
```

##### 4.2. 构建流程

- 要素

  源代码 (source)、S2I脚本 (s2i scripts)、镜像构造器 (builder image)。

- 基理：

  S2I 必须将 **source** 和 **s2i scripts** 放到 **builder image** 中。首先 S2I 创建一个包含 **source** 和 **s2i scripts** 的 tar 包，并以 file stream 的形式传输到 **builder image** 中。执行 assemble script 前，S2I 会解压 tar 包，并根据 **builder image** 中的  `--destination` flag  或者  label `io.openshift.s2i.destination ` 指定的位置。默认位于 /tmp 下。

![S2I 流程](./S2I 流程.png)

##### 4.3. 工作原理

参考：

- https://github.com/jorgemoralespou/s2i-java
- https://github.com/openshift/source-to-image
- https://github.com/ganrad/openshift-s2i-springboot-java
- https://github.com/openshift-s2i/s2i-wildfly

原理

- assemble

  	将外部代码库下载到本地，并编译打包。通过定义 **save-artifacts**，可以进行增量构建（mvn、npm缓存问题）。

- run 

  	运行 assemble 编译好的应用程序包。

- save-artfacts

  	save-artifacts脚本负责将构建所需要的所有依赖包收集到一个tar文件中。

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

  	使用说明文档。

  ```bash
  #!/bin/bash -e
  cat <<EOF
  开始你的说明......
  EOF
  ```

- test / run

  测试。

##### 4.4. 示例

###### 1)  Java

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

#### 5. Route

用于将一个或多个 Service 以域名的方法暴露给外部 Client。route 可以通过 CLI、YML或 OKD WebConsole 进行创建。

- CLI

```shell
## 所有语法
oc expose --help

## 示例
oc expose svc/x-docker --name x-docker-route --hostname-x.d.com  
```

- YML

```yaml
## 一个 route 绑定两个 service，通过权重 weight 实现灰度发布
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  labels:
    app: x-docker
  name: x-docker-route
  namespace: spiritdev
spec:
  alternateBackends:
  - kind: Service
    name: x-docker-2
    weight: 50
  host: x.d.com
  port:
    targetPort: 8080-http
  to:
    kind: Service
    name: x-docker
    weight: 50
  wildcardPolicy: None
```

- 负载机制

  	Route 的负载采用 session affinity，router 先选择一个 endpoint 处理 Client 的请求，并且会创建一个 cookie 在响应中传回。Client 随后的请求都会携带此 cookie，router 根据cookie 可以将 Client 的请求交给之前的 endpoint（同一个 pod） 进行处理。

    	可修改 cookie 的名称，在 Client 端对 cookie 进行删除，可实现 round-ribbon。

  ```yaml
  ## CLI 操作
  oc annotate route <route_name> router.openshift.io/<cookie_name>="-<cookie_annotation>"
  
  ## YAML 中
  apiVersion: route.openshift.io/v1
  kind: Route
  metadata:
    annotations:
      router.openshift.io/7774d0f24865b81a3860738f3b277fbe: x_docker_cookie
  	...
  ```

- SSL证书配置

  推荐在 WebConsole 进行操作。

```yaml
### CLI 配置证书
oc create route edge --service=x-docker \
    --cert=${MASTER_CONFIG_DIR}/ca.crt \
    --key=${MASTER_CONFIG_DIR}/ca.key \
    --ca-cert=${MASTER_CONFIG_DIR}/ca.crt \
    --hostname=x.d.com

### YAML 中
apiVersion: v1
kind: Route
metadata:
  name: x-docker-route
spec:
  host: x.d.com
  to:
    kind: Service
    name: x-docker
  tls:
    termination: edge
    key: |-
      -----BEGIN PRIVATE KEY-----
      [...]
      -----END PRIVATE KEY-----
    certificate: |-
      -----BEGIN CERTIFICATE-----
      [...]
      -----END CERTIFICATE-----
    caCertificate: |-
      -----BEGIN CERTIFICATE-----
      [...]
      -----END CERTIFICATE-----
```



### 二. 管理

#### 1. 节点管理

- 常用命令

```shell
## 列出集群中的 Node，三种状态
# Ready: 通过 master 的 health check
# NotReady: 未通过 master 的 health check
# SchedulingDisabled: Pod 禁止被调度到这个 Node 上 
oc get node -o wide
oc get nodes --selector='kubernetes.io/hostname=node3'

## 查看具体的 Node
oc describe node node1
## 为 Node 打标签
oc label node node1 pod-in=node1

## Node 上的 Pod 数
oc adm manage-node node3 --list-pods

## 关闭/开启 Node 的调度能力
## 关闭 Node 的调度后，设置了只能调度到此 Node 上的 Pod 无法启动
oc adm manage-node <node1> <node2> --schedulable=false
oc adm manage-node <node1> <node2> --schedulable
```

##### 1.1 撤离Node上的Pod

对于由 **RC、RS、DS、StatefulSet** 管理的 Pod，可以将指定**禁止调度Node**（schedulable=false） 上这些 Pod 迁移到集群的其他 Node 上。

重新启动 Node 时，需要将 Node 上的 Pod 迁移到其他 Node。

执行命令：`oc adm drain --help`

```shell
## node1 和 node2 必须被禁止调度
oc adm drain node1 node2

## Pod 不由 RC、RS、DS、StatefulSet 管理，可强制迁移，通过选项 --force=true
oc adm drain node1 node2 --force=true

## 预设 Pod 准备删除时间，默认是 30 s
oc adm drain node1 node2 --grace-period=10
## 强制删除
oc adm drain node1 node2 --force=true --grace-period=0
```

##### 1.2 Node 资源

通过计算节点的配置文件进行配置 **/etc/origin/node/node-config.yaml**，配置 **kubeletArguments**。

```yaml
kubeletArguments:
  image-gc-high-threshold:	## 镜像垃圾回收的最高磁盘使用百分比。默认 90%
  - '90'
  image-gc-low-threshold:	## 镜像垃圾回收的最低磁盘使用百分比。默认 80%
  - '80'
  kube-reserved:
  - cpu=200m,memory=512Mi	## DNS 容器工作所需资源
  max-pods:
  - '320'
  node-labels:
  - region=node
  pods-per-core:			## 单核可运行的 Pod 数
  - '10'
  system-reserved:
  - cpu=500m,memory=1024Mi
```

##### 1.3 IP 管理

可编辑 **/etc/origin/master/master-config.yaml**，重启才能生效 

`systemctl restart origin-master-api origin-master-controllers`

```yaml
networkConfig:
  clusterNetworkCIDR: 10.128.0.0/18
  clusterNetworks:
  - cidr: 10.128.0.0/18
    hostSubnetLength: 9
  externalIPNetworkCIDRs:
  - 0.0.0.0/0
  hostSubnetLength: 9
  networkPluginName: redhat/openshift-ovs-multitenant
  serviceNetworkCIDR: 172.30.0.0/16
```

#### 2. 权限机制

![OKD RBAC](./OKD RBAC.png)

​	Openshift具有很优秀的权限管理机制，提供非常细粒度的权限管理。访问Openshift平台服务的方式包括GUI、CLI、API、POD运行的权限，而可能的访问实体包括自然用户、外部应用程序、内部模块、POD内程序。这些不同的访问方式和访问实体需要使用灵活的机制去进行权限赋予和权限认证。

**Cluster Role**：访问Openshift平台不同服务的能力。

**SCC**：一组POD内Linux系统的权限的集合。

**User**：分为user id或者 service account。

```shell
## 用户 user、认证 identity、角色 clusterrole、角色绑定 clusterrolebinding
oc create user dante --full-name="Michale Dante"
oc create identity acme_ldap:dante
oc create clusterrole pod-reader --verb=get,list,watch --resource=pods
oc create clusterrolebinding cluster-admin --clusterrole=cluster-admin --user=dante --user=snake [--serviceaccount=namespace:serviceaccountname]
```

**Service Account**

Service Account是用来控制非自然用户在访问Openshift平台进行服务部署的时候所具有的能力范围的，它是承载SCC和cluster role的实体。

```shell
## 命名规则
system:serviceaccount:<project>:<saname>
oc create sa xbot

## 权限管理 Cli
oc policy --help
oc add-role-to-user view system:serviceaccount:spiritdev:xbot

```

	每个 Project 都需要 Sa 去运行 build、deploy和其他的 Pod，默认的 Sa 设置在 **/etc/origin/master/master-config.yml**。

```yaml
serviceAccountConfig:
  limitSecretReferences: false
  managedNames:
  - default
  - builder
  - deployer
  masterCA: ca-bundle.crt
  privateKeyFile: serviceaccounts.private.key
  publicKeyFiles:
  - serviceaccounts.public.key
```

**SCC（Security Context Constains）**

用户操作 Pod 内Linux系统操作权限的集合。

```shell
## scc 操作
oc get / describe / export / edit scc

## 赋予 user / sa 的 scc权限
oc adm policy add-scc-to-user <scc_name> <user/sa>
oc adm policy add-scc-to-user privileged spiritprd
oc adm policy add-scc-to-user privileged system:serviceaccount:spiritdev:xbot
oc adm policy add-scc-to-user privileged system:serviceaccount:spiritdev:default
```

创建自定义 SCC

```yaml
kind: SecurityContextConstraints
apiVersion: v1
metadata:
  name: scc-admin
allowPrivilegedContainer: true
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: RunAsAny
fsGroup:
  type: RunAsAny
supplementalGroups:
  type: RunAsAny
users:
- my-admin-user
groups:
- my-admin-group
```

#### 3. 调度

Pod 如何在合适的 Node 上启动。Openshift 提供了三种调度手段 Default、Advance、Custom。调度的实际使用目的包括亲和性和反亲和性。在 OKD 中，可以将 Node 通过 Label 分类，例如 Region — Zone — 机架。

**亲和性**：可以将同一服务的 Pod 都调度到相同的 Region — Zone 上，确保不会因地理位置而导致的延迟问题。

**反亲和性**：可以将统一服务的 Pod 均匀的分布在可用的 Node 上，确保高可用 HA。

```yaml
### 配置：/etc/origin/master/scheduler.json
### region - zone 亲和，rack 反亲和
### 生效：systemctl restart origin-master-api origin-master-controller 
kind: "Policy"
version: "v1"
predicates:
...
  - name: "RegionZoneAffinity"
    argument:
      serviceAffinity:
        labels:
          - "region"
          - "zone"
priorities:
...
  - name: "RackSpread"
    weight: 1
    argument:
      serviceAntiAffinity:
        label: "rack"
```

##### 3.1 默认调度

通过 predicate 和 priority 选择合适的 Node。

##### 3.2 自定义调度

参照 k8s 源码，编写自己的调度器 https://github.com/kubernetes/kubernetes/blob/master/pkg/scheduler/scheduler.go 。

**scheduler.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-scheduler
  labels:
    name: multischeduler-example
spec:
  schedulerName: custom-scheduler 
  containers:
  - name: pod-with-second-annotation-container
    image: docker.io/ocpqe/hello-pod
```

##### 3.3 高级调度

前期准备：`oc label node node2 supergod=dante`

- **Node Affinity**

  采用 label 标记 Node，创建 Pod 时在 affinity 字段标明匹配规则，支持两种调度方式：

  - requiredDuringSchedulingIgnoredDuringExecution

    一定要存在满足条件 Y 的节点，如果不存在，则 pod 创建失败， hard 模式。

  - preferredDuringSchedulingIgnoredDuringExecution

    优先选择满足条件 Y 的节点，如果不存在，则在其它节点中择优创建 pod， soft 模式。需要指定权重 weight（1 - 100），权重从高到底。

  ```yaml
  --- 
  ### 设置 requiredDuringSchedulingIgnoredDuringExecution
  spec:
    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: supergod
              operator: Exists
              values:
              - dante
              
  ---    
  ### 设置 preferredDuringSchedulingIgnoredDuringExecution
  	nodeAffinity:
  	  preferredDuringSchedulingIgnoredDuringExecution:
  	  - weight: 20
  	    preference:
  	      matchExpressions:
  	      - key: supergod
              operator: IN
              values:
              - dante
              - snake 
  ```

- **Pod Affinity / Anti-Affinity**

  ​	Node 亲和性不方便满足反亲和性的需求，例如：为了保证 redis 高可用，将 redis 的3个实例分布在不同的节点。故引入了 Pod Affinity / Anti-Affinity，用来控制 Pod 相对其他 Pod 的分布，实际用于 Service 层，控制不同 Service 的亲和和反亲和。采用 label 标记 Pod 实现。

  - 亲和性

  ```yaml
  --- 
  ### 网络要求高的 Pod 分布在相同的 Node
  spec:
    template:
      metadata:
        labels:
          network: mustfast
      spec:
        affinity:
          podAffinity: 
            requiredDuringSchedulingIgnoredDuringExecution: 
            - labelSelector:
                matchExpressions:
                - key: network 
                  operator: In 
                  values:
                  - mustfast 
              topologyKey: failure-domain.beta.kubernetes.io/zone
  ```

  - 反亲和性

  ```yaml
  ---
  ## 3个 redis 实例分布在不同的 Node 
  spec:
    selector:
      matchLabels:
        app: redis
    replicas: 3
    template:
      metadata:
        labels:
          app: redis
      spec:
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - redis
              topologyKey: kubernetes.io/hostname
              
          podAntiAffinity:
            preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                  - key: security
                    operator: In
                    values:
                    - S2
                topologyKey: kubernetes.io/hostname
  ```

- **Node Selector**（**不建议）**

  ​	最简单的一种调度策略（Affinity 可以完全满足，会被废弃），不建议使用。**pod.spec.nodeSelector** 通过kubernetes的label-selector机制选择节点，由调度器调度策略匹配label，而后调度pod到目标节点，该匹配规则属于强制约束。

```yaml
## 为 Pod 设置 NodeSelector
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
  image: nginx
  imagePullPolicy: IfNotPresent
  nodeSelector:
    supergod: dante

```

- **Taints and Tolerations（污点 和 容忍）**

  ​	Taints（Node） 和 Tolerations（Pod），Pod 无法被调度到被标记上 Taint 的 Node 上，除非 Pod 被标记了 Toleration。即，只有那些容忍污点Node的Pod，才会被调度到污点Node上。

  使用场景，当我们想保留一些具有特定硬件功能的 Node，只让特定的 Pod 可以调度到这些Node上。

  Taint 和 Toletation 的组成：**key**（**253字符**）、**value**（**63字符**）、**effect**

  | effect               | 说明                                                         |
  | -------------------- | ------------------------------------------------------------ |
  | **NoSchedule**       | 1、新的非容忍Pod不会调度到污点Node上<br>2、已存在的Pod不受影响 |
  | **PreferNoSchedule** | 新的非容忍Pod可能会调度到污点Node上，但调度器不会尝试将该Pod调度到污点Node上 |
  | **NoExecute**        | 1、新的非容忍Pod不会调度到污点Node上<br/>2、已存在的Pod不容忍的Pod会被从污点Node上删除 |

  - **Node** 

  ```shell
  oc adm taint nodes node1 key1=value1:NoSchedule
  oc adm taint nodes node2 key2=value2:PreferNoSchedule
  oc adm taint nodes node1 key1=value1:NoExecute
  ```

  - **Pod**

  ```yaml
  tolerations:
  - key: "key2"
    operator: "Equal"
    value: "value2"
    effect: "NoExecute"
    tolerationSeconds: 3600	## Pod不匹配污点Node时，从Node上删除的等待时间，单位：秒
  ```

  ​	默认的 Toleration Seconds 是5分钟，可在 **/etc/origin/master/master-config.yaml** 配置，需要重启才能生效：`systemctl restart origin-master-api origin-master-controllers` 。

  ```yaml
  admissionConfig:
    pluginConfig:
      DefaultTolerationSeconds:
        configuration:
          kind: DefaultAdmissionConfig
          apiVersion: v1
          disable: false
  ```

### 三. 网络

#### 1. OVS

- 东西走向

  - **Pod 在同一个 Node**

  ![Pod1Node](./Network/Pod1Node.jpg)

  Pod1要和pod2进行通讯（在ovs的br0中转发是依赖于Openflow的流表规则），通讯流量：

  ```shell
  pod1的eth0 → veth1 → br0 → veth2 → pod2的eth0
  ```

  - **Pod不在同一个Node**

  ![Pod2Node](./Network/Pod2Node.jpg)

  1. Pod1 访问 Pod3

  ```shell
  Pod1的eth0 → veth1 → br0 → vxlan0 → OCP nodeA的eth0网卡 → OCP NodeB的eth0网卡 → vxlan0 → br0 → veth3 → Pod3的eth0
  ```

  2. Pod1 访问外部网络

  ```shell
  Pod1的eth0 → veth1 → br0 → tun0 → 通过iptables实现NAT → OCP nodeA的eth0(physical device) → Internet
  ```

#### 2. Calico







### 八. 参考资料

- https://cloud.tencent.com/developer/article/1070380



