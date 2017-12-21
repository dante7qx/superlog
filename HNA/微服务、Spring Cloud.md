## 微服务、SpringCloud、Caas

### 一. 微服务概述

​	微服务是一种架构风格，一个**大型复杂软件应用**由一个或多个微服务组成。系统中的各个微服务可被独立部署，各个微服务之间是松耦合的。每个任务代表着一个小的业务能力，一个独立的项目。

​	为什么要提出微服务？

#### 传统架构的不足

1.   复杂性高

     ​	项目发展到后期，各个模块边界非常模糊，代码质量参差不弃。常常一个很小的功能修改，都会导致一系列的功能受到影响，开发效率直线下降。

2. 坑越来越多

     ​	因为整个项目耦合度非常高，很多代码都知道这样做不好，以后可能会出问题，但没有人敢去修改。

3. 部署困难

     ​	随着代码量的增多，构建和部署的时间也会增加。每次功能变更、缺陷修改都会导致我们需要部署整个项目。影响范围大、风险高，导致部署频率低，而部署频率低又会导致两个部署版本之间有大量的内容变化。加大出错的概率。

4. 扩展性差

     ​	单体应用都是部署在一起的。这样就无法按需进行伸缩。例如，有些业务功能是计算密集型的，就需要CPU强一些；有些业务IO操作密集，硬盘要好；有些业务内存需求大，就要加大内存。单体应用就无法满足。

5. 技术上越来越Low

     ​	原来整个项目用的Struts，现在流行Spring MVC（其他语言框架），一个做了5、6年的系统，想切换，不太可能。

#### 微服务的好处

解决传统架构的不足。

#### 微服务的不足

1. 技术复杂性

   微服务应用都是分布式的，分布式的技术难度，例如：分布式事务

2. 部署复杂性

   分布式部署的复杂性

3. 测试难度变大

   一个微服务的修改导致很多服务都需要修改。

4. 冗余的代码增多

5. 性能比起单体架构有所降低

#### 微服务设计原则

- 单一职责原则 - 高内聚低耦合
- 服务自治原则 - 每个微服务尽量独立
- 轻量级通信原则 - REST API

#### 微服务开发框架

- **Spring Cloud**
- Dubbo + ZK
- RedKale

#### 微服务设计思想

​	DDD - 领域驱动设计思想

### 二. SpringCloud

​	Spring Cloud是一个基于Spring Boot实现的云应用开发工具，它为基于JVM的云应用开发中的配置管理、服务发现、负载均衡、断路器、智能路由、微代理、控制总线、全局锁、决策竞选、分布式会话和集群状态管理等操作提供了一种简单的开发方式。	

![微服务架构](/Users/dante/Documents/Technique/且行且记/HNA/微服务架构.png)

#### 1. Eureka

​	服务注册中心，分为 Eureka Server 和 Eureka Client，通过Eureka Rest API 进行调用。

![Eureka-HA](/Users/dante/Documents/Technique/且行且记/SpringCloud/Eureka-HA.png)

#### 2. Spring Cloud Config

​	在微服务中，由于服务数量巨多，为了方便服务配置文件统一管理，实时更新，所以需要分布式配置中心组件。Spring cloud config 分为config server 和 config client。Server 端将应用配置存储到 git repo中，Client 端向 Server请求抓取配置。

![Spring Cloud Bus](/Users/dante/Documents/Technique/且行且记/SpringCloud/Spring Cloud Bus.png)

#### 3. Ribbon、Feign

​	Ribbon：客户端负载均衡组件。

​	Feign：一个声明式客户端工具，通过Ribbon来实现负载功能。

- **服务器端负载均衡**

  ​	将后端应用服务的地址配置到一个负载均衡服务器（Nginx、HAProxy、F5等），用户的请求先经过Load Balancer，根据指定的负载算法，最终将请求分发到后台应用。

  ![Server Load Balance](/Users/dante/Documents/Technique/且行且记/SpringCloud/Server Load Balance.png)

- **客户端负载均衡**

  ​	每个应用获取服务提供者的地址列表，然后按照指定的负载策略，分发请求到不同的服务器。

![yRibbon Client Load Balance](/Users/dante/Documents/Technique/且行且记/SpringCloud/Ribbon Client Load Balance.png)

#### 4. Hystrix

​	在分布式环境中，不可避免地有许多服务调用失败。服务调用失败时,接下来的调用都会失败，每一次的失败都会造成很大的延迟，直到服务恢复正常。那么能否在失败时，不再调用失败的服务，而直接返回错误信息，然后服务内部自己去调用服务，如果发现成功了，再直接调用服务，返回结果，而不再返回错误信息。即容错机制。	 		 

​        Hystrix是一个库，可以通过添加延迟容错和容错逻辑来控制这些分布式服务之间的交互。 Hystrix通过隔离服务之间的接入点，阻止它们之间的级联故障，并提供备用选项Fallback，从而提高系统的整体弹性和服务的健壮性。特性如下：

1. 断路器机制

   ​        当Hystrix Command请求后端服务失败数量超过一定比例(默认50%), 断路器会切换到开路状态(Open). 这时所有请求会直接失败而不会发送到后端服务. 断路器保持在开路状态一段时间后(默认5秒), 自动切换到半开路状态(HALF-OPEN). 这时会判断下一次请求的返回情况, 如果请求成功, 断路器切回闭路状态(CLOSED), 否则重新切换到开路状态(OPEN)。

   ![断路器原理](/Users/dante/Documents/Technique/且行且记/SpringCloud/断路器原理.png)

2. Fallback

   ​     Fallback相当于是降级操作。例如：对于查询操作, 我们可以实现一个fallback方法, 当请求后端服务出现异常的时候, 可以使用fallback方法返回的值。fallback方法的返回值一般是设置的默认值或者来自缓存。

3. 资源隔离

   ​    主要通过线程池来实现资源隔离，调用A服务的Command放入A线程池，调用B服务的Command放入B线程池，这样当A服务中存在bug导致线程池耗尽时，其他的服务不会收到影响。

#### 5. Zuul

​	Zuul，服务网关。是微服务不可缺少的组成部分，通过服务网关统一向外系统提供REST API。主要提供动态路由、监控、权限控制、负载均衡、压力测试、静态资源响应处理、弹性伸缩等。

![zuul-场景](/Users/dante/Documents/Technique/且行且记/SpringCloud/zuul-场景.png)

#### 6. 其他组件

**SideCar**：多语言支持。

**Sleuth**：分布式链路调用监控。

#### 9. 参考资料

- http://cloud.spring.io/spring-cloud-static/Dalston.SR1/
- https://github.com/netflix

### 三. Caas平台概述

​	易建科技基于kubernetes开发的一套容器云平台（https://caas.haihangyun.com/）。主要提供应用的自动构建、部署、集群管理等容器服务，支持负载均衡、灰度升级、弹性伸缩、SSL加密、容灾恢复、服务日志、监控报警等功能，使用户无需采购服务器，无需构建发布应用，无需部署服务端环境，无需配置负载均衡，无需手动扩容集群，无需申请IP地址，无需安装SSL证书，提供一站式应用生命周期管理。

### 四. 自动化部署

​	基于Spring Cloud构建的微服务，最终是以 jar、war的形式进行部署的。考虑到每个微服务的高可用性，通常每个微服务都需要多个实例，而每个实例一般会部署到不同的服务器上。当整个系统规模小的时候，可以通过手工部署，但系统规模变大后，手工部署就不合适了。

​	另外，部署到物理机或云主机都需要在每台服务器上安装运行环境，项目大了之后，也是非常耗时，且容易出错的。因此需要利用容器来部署。

​	结合Caas进行自动化部署，主要是通过 Jenkins 来进行，部署流水线如下

![部署流水线](/Users/dante/Documents/Technique/且行且记/HNA/部署流水线.png)

- Jenkins（http://jenkins.haihangyun.com）流水线脚本

  ```shell
  withEnv(['PATH=/usr/local/bin:$PATH']) {
      node {
              stage('代码更新') {
                  checkout scm: [$class: 'GitSCM', branches: [[name: '*/master']],
                               userRemoteConfigs: [[url: 'http://gitbj.haihangyun.com/ch.sun/caas.git']]]
              }
                         
              stage('代码检查') {
                  dir('caas-provider/') {
                      sh "sonar-scanner -Dsonar.login='jenkins' -Dsonar.password='jenkins' -Dsonar.host.url=http://sonar.haihangyun.com"
                  }
                  dir('caas-provider/deploy/') {
                      sh "bash ./checkSonarStatus.sh"
                  }
              }
              
              stage('单元测试') {
                  dir('caas-provider/') {
                      sh "mvn test"
                  }   
              }
              
              stage('上传运行Jar到远程仓库') {
                  dir('caas-provider/') {
                      sh "mvn deploy -Dmaven.test.skip=true"
                  }
              }
          }
  		
          stage('部署确认') {
              timeout(time:1, unit:'HOURS') {
                  milestone()
                  input "现在执行部署？"
              }
          } 
          
          node {
              lock(resource: 'CAAS_PROVIDER', inversePrecedence: true) {
      			stage('部署服务') {
                      dir('caas-provider/deploy/') {
                          sh "bash ./buildImage.sh"
                          sh "bash ./makesureImageReady.sh"
                          sh "bash ./deployImage.sh"
                          sh "bash ./waitDeployment.sh"
                      }
      				
      			}
                  
                  stage('接口测试') {
                      echo '接口测试'
                  }
                      
                  milestone()
              }
  		}
      
  }
  ```

- 获取代码更新 [Gitlab](http://gitbj.haihangyun.com/)

- 代码静态扫描  [Sonnar](http://sonar.haihangyun.com/)

- 单元测试 `mvn test`

- 编译上传运行jar `mvn clean compile deploy`，[Nexus](http://maven.haihangyun.com/)

- 部署，调用 Caas 平台接口

  ```json
  1. 从指定仓库读取Dockerfile构建镜像，并返回构建出的镜像完整名称
  请求方法：
  POST
  请求URL：
  https://caas.haihangyun.com/rest/buildImage
  请求参数：
  json形式，例：{
  	"namespace":"yournamespace ",
  	"gitlabBranch":"master",
  	"gitlabPassword":"pwd",
  	"gitlabUser":"user",
  	"imageName":"testproject",
  	"gitlabRepo":"http://127.0.0.1/repo/ testproject.git",
  	"dockerPath":"/"
  }
  返回值：
  {"statusCode":0,"status":"Success","imageNameAndTag":"hkdhub.paas.haihangyun.com/yournamespace/testproject: 20161125115213"}
  {"statusCode":-3,"status":"Can’t find XXX"}
  {"statusCode":-4,"status":" Build error：XXXX"}
     
  2. 获得指定镜像构建状态
  请求方法：
  GET
  请求URL：
  https://caas.haihangyun.com/rest /queryImage/{namespace}/{imageName}/{tag}
  请求参数：
  namespace：命名空间
  imageName：镜像名称
  tag：镜像的版本
  返回值：
  {"statusCode":0,"status":"Success","imageBuildStatus":"Complete/Failed/CREATING"}
  说明：Complete:构建成功   Failed:构建失败   CREATING:构建中
  {"statusCode":-3,"status":"Can’t find XXX"}   
     
  3. 对指定服务进行版本升级（不论该服务处于运行/停止/出错状态）
  请求方法：
  POST
  请求URL：
  https://caas.haihangyun.com/rest/deploy/{namespace}/{servicename}/{targetTag}/upgrade
  请求参数：
  namespace：命名空间
  servicename：服务名称
  targetTag：镜像的版本
  返回值：
  {"statusCode":0,"status":"upgrading"}
  {"statusCode":-1,"status":"check failed"}
  {"statusCode":-3,"status":"can’t find user"}
  {"statusCode":-8,"status":"xxx service is not exist"}
  {"statusCode":-9,"status":"upgrade failed"}
     
  4. 获得指定服务运行状态（至少能区分：运行中/部署中/运行出错）
  请求方法：
  GET
  请求URL：
  https://caas.haihangyun.com/rest/deploy/{location}{namespace}/{servicename}/status
  请求参数：
  namespace：命名空间
  servicename：服务名称
  返回值：
  {"statusCode":0,"status":"Success","serviceRuntimeStatus":"Running/Pending/Failed"}
  说明：Running:运行中   Pending:部署中   Failed:失败
  {"statusCode":-1,"status":"check failed"}
  {"statusCode":-3,"status":"can’t find user"}
  {"statusCode":-8,"status":"xxx service is not exist"}
  {"statusCode":-10,"status":"get service status failed"}
   

  5. 查询指定镜像的所有Tag列表
  请求方法：
  GET
  请求URL：
  https://caas.haihangyun.com/rest/v1/namespaces/{namespace}/images/{name}/tags

  请求参数：
  namespace：命名空间
  name：镜像名称
  返回值：
  {"statusCode":0,"status":"Success"," imageName":" hkdhub.paas.haihangyun.com/yournamespace/testproject","imageTag":["20161227142451","20161227155042"]} 
  {"statusCode":-1,"status":"check failed"}
  {"statusCode":-3,"status":"can’t find xxx"}

  6. 删除指定的任意个镜像Tag
  请求方法：
  DELETE
  请求URL：
  https://caas.haihangyun.com/rest/v1/namespaces/{namespace}/images/{name}/tags

  请求参数：
  namespace：命名空间
  name：镜像名称
  请求体参数：
  {"tags":[]}，例：{"tags":["20161227142451","20161227155042"]}
  返回值：
  {"statusCode":0,"status":"Success"} 
  {"statusCode":-1,"status":"check failed"}
  {"statusCode":-3,"status":"can’t find xxx"}
   

  7. 取消构建
  请求方法：
  PUT
  请求URL：
  https://caas.haihangyun.com/rest/v1/namespaces/{namespace}/images/{name}/cancel
  请求参数：
  namespace：命名空间
  name：镜像名称
  返回值：
  {"statusCode":0,"status":"Success"} 
  {"statusCode":-1,"status":"check failed"}
  {"statusCode":-3,"status":"can’t find xxx"}

  8. 创建服务
  请求方法：
  POST
  请求URL：
  https://caas.haihangyun.com/rest/deploy/{namespace}/services/{servicename}/create
  请求参数：
  namespace：命名空间
  servicename：服务名
  请求体参数示例：
  {
   "imageName":"worthy",   镜像名称
   "tagName":"20170307144924",    镜像版本
   "appName":"test",   应用名（服务所属应用）
   "replicas":"1",   容器个数（默认为1）
   "location":"BJ",   地区（默认北京，海口为HK）
   "containerConfig":{"cpu":"1","memory":"512"},   （容器的配置，memory以M为单位）
   "portsJson":[{"containerPort":"8080","protocol":"HTTP"}],   （端口的配置，协议为HTTP/TCP）
   "envsJson":{"testenv":"test"},   （环境变量）
   "forceUpdate":true,   （如果服务存在是否进行服务升级，默认false）
   "volumes":[{"volName":"test7","volSize":"20","volPath":"/var"}]   （挂卷，卷大小以M为单位）
  }
  返回值：
  {"statusCode":0,"status":"Creating"} 
  {"statusCode":-1,"status":"check failed"}
  {"statusCode":-3,"status":"can’t find xxx"}
  {"statusCode":-8,"status":"xx service is already exist"}
  {"statusCode":-9,"status":"Create failed"}
  {"statusCode":-11,"status":"Authorization failed"}
  ```

### 五. Demo 演示

	1. Eureka Server 高可用自动部署。3个Eureka集群相互发现注册。
	2. 服务提供者 caas-provider 自动部署2个实例。
	3. 服务消费者 caas-consumer 自动部署2个实例。

代码地址

​	http://gitbj.haihangyun.com/ch.sun/caas.git









