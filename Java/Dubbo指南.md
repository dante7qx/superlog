## Dubbo 积累

### 一. 概述

	Apache Dubbo (incubating) |ˈdʌbəʊ| 是一款高性能、轻量级的开源Java RPC框架，它提供了三大核心能力：面向接口的远程方法调用，智能容错和负载均衡，以及服务自动注册和发现。

![Architecture](./dubbo/Architecture.png)

### 五. 容器化

![deploy](./dubbo/deploy.png)

**问题**

1. Pod IP 只在k8s集群内可见，导致无法与集群外的 dubbo 服务交互
2. dubbo 服务路由是基于IP的，但 Pod IP是会经常变的

**解决**

- **集群内外 IP 互通**
  - 方式一，k8s - ovs
  - 方式二，使用 Calico网络

  ![Calico](./dubbo/Calico.png)

  - 方式三，OKD externalIP

- **Provider IP白名单**

  1. Provider 和 Consumer 部署都在容器上，并且集群内网络已打通。

     该场景下，Provider 获取到的 Consumer 的 IP 是Consumer 所在 Pod 的 Pod IP，因为 k8s 中，Pod 重新部署后会将之前的 Pod Kill，重新启动一个新的 Pod，并为这个 Pod 重新分配IP，所以 Provider 会拿到一个新的IP，IP 变化的规律无法人工控制，故无法使用IP白名单的方式进行访问控制。

  2. Provider部署在容器中，Consumer 部署在容器外。

     该场景下，Consumer 的请求都通过容器的负载均衡服务器传递到具体的 Provider Service，在 Provider Service 对应的 Pod 中，使用 RpcContext.getContext().getRemoteHost() 获取到的IP都是 Provider Pod 的网关IP。Provider 无法对集群外的 Consumer 做IP 控制。

  3. Provider 和 Consumer 部署都在容器上，并集群内网络未打通。

     该场景下，Consumer 通过 Provider 暴露的外部 Socket 进行通信，Consumer 的请求过程同第二种情况，Provider 通过 RpcContext.getContext().getRemoteHost() 获取到的IP都是 Provider Pod 的网关IP。Provider 无法对集群内不同租户的 Consumer 做IP 控制。

  4. Provider 部署在集群外，Consumer 部署在容器上

     该场景下，Provider 通过 RpcContext.getContext().getRemoteHost() 获取的 Consumer IP 是 Consumer Pod 所在 Node 上的 IP。Provider无法对容器平台的Consumer做IP控制。

  **解决方案：**

  - 步骤一：容器云平台替换网络插件，使用Calico。

    对于场景2，Provider 可以获取到容器集群外 Consumer 的真实 IP，从而实现 IP 白名单功能。

  - 步骤二：对 Provider 新增一种访问授权方式，例如 OAuth2、OpenID等。

    对于场景1、3、4，OAuth2（或OpenID） 的授权方式都可以满足需求。

### 八. 参考资料

- http://dubbo.apache.org/books/dubbo-user-book/
- http://dubbo.apache.org/books/dubbo-admin-book/
- http://dubbo.apache.org/books/dubbo-dev-book/
- https://github.com/dubbo/dubbo-samples

### 九. 工作经验

1. Provider 需要在一个容器中运行（启动后不能退出），否则会触发 dubboShutdownHook。

