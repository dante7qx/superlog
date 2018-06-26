## Service Mesh

### 一. 概念

#### 1. 引言

​	一个完整的微服务涉及到的功能有**服务发现**、**智能路由（网关）**、**负载均衡、** **熔断机制、** **流量控制**、**超时重试**等。这些功能是微服务应用程序间的通讯逻辑，一般会进行抽象和归纳，形成一套框架供各个微服务应用使用。这类框架有：SpringCloud、Dubbo等。

这类框架的不足，**业务逻辑的耦合性太强**：

 	1. 不透明，开发人员需要彻底掌握框架的原理后才能正确使用，不能专注于业务逻辑的开发。
	2. 不同的语言有自己的框架，影响技术选择的灵活性。
	3. 框架不同版本的不兼容性，以及大量微服务的升级也是一个很大的难题。

为了解决微服务框架库和业务逻辑的耦合关系，需要将这部分为微服务提供通信的逻辑从应用程序中抽离出来，作为**一个单独的==进程==进行部署**，并将这个代理作为**服务间的通信代理**。如下图的 **Sidecar**。

![通信代理](/Users/dante/Documents/Technique/且行且记/云原生/通信代理.png)	

#### 2. ServiceMesh

##### 2.1 架构

​	服务网格是一个基础设施层，用于处理服务间的通信。在云原生应用服务复杂的网络拓扑中，ServiceMesh 负责实现请求的可靠传递。具体体现，ServiceMesh 以一组**轻量的网络代理**，与应用部署在一起，但对应用透明，负责找到目的服务并负责通讯的可靠性和安全等问题。而这组网络代理称为 SideCar，如下图![跨子](/Users/dante/Documents/Technique/且行且记/云原生/SideCar.png)

应用通信，见下图

![网格-1](/Users/dante/Documents/Technique/且行且记/云原生/网格-1.png)

​	上图中的蓝色方框就是 SideCar，ServiceMesh中数量众多的代理，需要进行集中的控制，因此ServiceMesh之上还需要控制面板组件。

![control-plane](/Users/dante/Documents/Technique/且行且记/云原生/control-plane.png)

##### 2.2 实现框架

**Istio**：Google、IBM、Lyft （Envoy代理）联合开发的Service Mesh框架。

**Conduit**：Buoyant （云原生初创公司），使用 Rust 并结合Linked的生产经验开发的基于k8s环境的新架构。

### 二. Istio

#### 1. 简介

​	Istio 是 Service Mesh 的一套完整的解决方案，通过为整个服务网格提供行为洞察和操作控制来满足微服务应用程序的多样化需求。本文的操作均基于 k8s 环境。

![istio架构](/Users/dante/Documents/Technique/且行且记/云原生/istio架构.png)

#### 2. 原理

##### 2.1 数据面板

由一组智能代理 Envoy 组成。即 SideCar，处理各个服务间的通信。通过数据面板的 api 和控制面板进行交互。

- **Envoy**

  ​	Lyft 的开源项目，兼容数据面板的 api。所有兼容**Data plane API** 的实现都可以成为Istio的SideCar。部署时，和服务部署到同一个pod中。Envoy的具体功能有：

  ​	动态服务发现，负载均衡，TLS终止，HTTP/2&gRPC代理，熔断器，健康检查，基于百分比流量拆分的分段推出，故障注入和丰富指标。

##### 2.2 控制面板

管理和配置Envory，来路由流量、运行时的执行策略、安全。

###### Pilot

1. 平台适配器规定了网格中服务的规范，例如 k8s适配器实现需要去Watch k8s API中Pod的注册信息、**ingress**资源以及用于存储流量管理规则的第三方资源的更改。
2. Pilot 用来管理 Envoy 实例的生命周期。Envoy API 公开了 **服务发现**、**负载均衡**、**路由表**的动态更新的 API。所以Envoy不需要关心底层平台的细节，也就是说兼容Envoy API的所有第三方代理组件都可以取代当前的Envoy。
3. Rules API，可以指定高级流量管理规则，被翻译成 Envoy 理解的规则，然后通过 discovery API 分发到 Envoy 实例。
4. 承上启下
   - 向下适配各个平台，封装平台细节。
   - 向上暴露 Envoy API，管理并且供各个具体的SideCar实现来交互。
   - 具体功能：**• 服务发现 • 负载均衡 • 请求路由 • 故障处理 • 故障注入 • 规则配置** 。  

![Pilot](/Users/dante/Documents/Technique/且行且记/云原生/Pilot.png)

###### Mixer

​	应用微服务和基础设施的中间层。主要作用是将策略决策（白名单、监控、日志、计费、限流、配额等）从应用层中移出，不在和特定的后端集成在一起，而是和 Mixer 进行简单的集成，Mixer 负责和后端进行连接。

![Mixer](/Users/dante/Documents/Technique/且行且记/云原生/Mixer.png)

**实现原理**：

​	Mixer处理不同基础设施后端（白名单、监控、日志、计费、配额等）的功能是通过使用通用插件模型实现的。每个插件成为**适配器**（Prometheus、ZipKin、Fluentd 等），暴露一个**一致的**独立于后端的API。

 1. Envoy Proxy 的每一次请求会将一组属性发送给 Mixer。

	2. 而 Mixer 根据这些属性来决定（运维配置处理逻辑）由哪个适配器来进行工作。

    - **Handler**

      Handler 的唯一标识，`{metadata.name}.{kind}.{metadata.namespace}` 。一个已经配置完毕的适配器实例。

    - **Request Instance**

      将请求中的属性映射成为适配器的输入。

    - **Rule**

      **Request Instance** 和 **Handler** 的选择规则。

	3. Mixer 通过**适配器API**调用后端基础设施。![Mixer流程图](/Users/dante/Documents/Technique/且行且记/云原生/Mixer流程图.png)

###### Istio-Auth

​	提高微服务及其通信的安全性，而不需要修改服务代码。

#### 3. 安装

- 前提条件

  1. **一个 Pod 只能属于一个 Service。不支持一个 Pod 同时属于多个 Service。**
  2. **Service 的端口必须命名，命名规则 <protocol>[-<suffix>]，例如：http-shop、mongo-db。目的是使用Istio的路由功能，否则流量均作为 TCP 流量来处理。**
  3. **Deployment 必须为 Pod 设置一个明确的Label （app），即 app = login，并且每个 Deployment 中的 app label 要唯一。目的是 app label 用于在 Zipkin 中添加上下文。**
  4. **每个 pod 里都有一个 Sidecar。分手动注入、自动注入。**

- 安装步骤

  1. 下载 Istio。 https://github.com/istio/istio/releases

  2. 设置 Istio 的环境变量 

  3. 安装 istio （Master 节点）

     - 默认自动注入 Sidecar（k8s 版本必须是 1.9 以上），可在 istio-demo.yaml 中设置为手工注入。

       1）Sidecar 之间通信不进行 TLS 验证，可以和非 Istio 的k8s Service 进行通信。

       `kubectl apply -f install/kubernetes/istio-demo.yaml`

       2) Sidecar 之间通信部进行 TLS 验证，在新的 k8s 集群中使用此安装。

       `kubectl apply -f install/kubernetes/istio-demo-auth.yaml`

     - 部署应用

       1）自动注入

       ```shell
       kubectl label namespace <namespace> istio-injection=enabled
       kubectl create -n <namespace> -f <your-app-spec>.yaml
       ```

       2）手动注入

       `kubectl create -f <(istioctl kube-inject -f <your-app-spec>.yaml)`

  4. 卸载

     `kubectl delete -f install/kubernetes/istio-demo.yaml`

#### 4. 功能详解

##### 4.1 术语

|     **Service**      | 一个应用用于服务发现的惟一标识，例如：k8s 中的 service，SpringCloud 中的 spring.application.name。 |
| :------------------: | ------------------------------------------------------------ |
| **Service versions** | 服务Instance的 Tag，可以是版本 v1, v2, v3...，也可以是项目阶段 dev、uat、prod。应用本身不需要知道具体的要通信的 Service Version，由 Envoy 代理拦截并转发客户端和服务器之间的所有请求/响应到指定的 Service Version。 |
|      **Source**      | 调用服务的客户端。                                           |
|       **Host**       | Source 请求连接服务的访问地址。建议写 **svcname.namespace.svc.cluster.local**。 |
|   **Destination**    | Source host 的处理服务（网络可寻址服务），destination.host 指定 k8s service name（实际交互的 host 是 **svcname.namespace.svc.cluster.local**），而 destination.subset 指定 service version，即 labels[0].version。 |
|     **Gateway**      | 统一配置Service Mesh中的各个HTTP/TCP负载均衡器。Gateway可以是处理入口流量的Ingress Gateway，负责Service Mesh内部各个服务间通信的Sidecar Proxy，也可以是负责出口流量的Egress Gateway。 只负责配置**L4～L6**的功能，例如暴露的端口，TLS设置。<br/>1. 浏览器只支持访问（80或443）的 load balancer ingress gateways。<br/>2. NodePort，部分浏览器只支持 VirtualService 中 host 是 "*"。 |
| **DestinationRule**  | 定义路由触发后，Service流量的策略。包括负载策略、服务**endpoints**策略(subset) |
|  **VirtualService**  | 可以绑定Gateway，将一个Service的所有 Route Rule 一起配置，负责 L7 层的负载配置。其中 route. destination.subset 通过  DestinationRule 进行配置。 |
|       待续...        |                                                              |

##### 4.2 路由控制

- 请求路由
- 故障转移（注入故障、设置请求超时时间）
- 入/出站流量控制
- 熔断器
- 流量投影/镜像
- Prometheus 和 Service Graph 实验不成功（0.8.0）

![流量路由](/Users/dante/Documents/Technique/且行且记/云原生/流量路由.png)



# 八. 参考资料	

- http://dockone.io/article/4825
- http://dockone.io/article/2982
- http://istio.doczh.cn/
- https://conduit.io
- http://www.servicemesh.cn/
- https://www.jianshu.com/p/b72c1e06b140
- https://github.com/redhat-developer-demos/istio-tutorial （OpenShift）

API 参考

- https://www.2cto.com/kf/201804/740234.html

- http://localhost:8001/apis/authentication.istio.io/v1alpha1
- http://localhost:8001/apis/networking.istio.io/v1alpha3

- http://localhost:8001/apis/config.istio.io/v1alpha2
- https://github.com/snowdrop/istio-java-api

Java实战

- https://github.com/saturnism/istio-by-example-java
- https://github.com/snowdrop

版本更新

- https://blog.csdn.net/zhaohuabing/article/details/80566619 （0.8 版本更新内容）



