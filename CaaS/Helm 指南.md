## Helm 指南

### 一. 概念

​	Helm是 k8s 中云应用管理工具，相当于 yum、brew等。可以更加便捷的将应用部署到 k8s 集群。Helm 有三个组成部分 Chart、Repository、Release。安装时需要安装 **Helm Client** 和 **Tiller**（服务端）。

- **Chart**

  ​	k8s 的应用部署包（redis-3.4.1.tgz），包括各种 k8s 对象的配置模板、参数定义、依赖关系、文档说明等。等同于 yum 的 rpm软件包。例如: **stable/redis** 。

- **Repository**

  ​	Chart 包的仓库，相当于 Docker Hub、Maven Repo。

- **Release**

  ​	一个 Chart 应用包部署在 k8s 集群中的 instance，可以指定 instance 的名称。

  ```shell
  helm install stable/redis --name dante-helm-redis
  
  dantedeMacBook-Pro:repository dante$ helm list --all
  NAME            	REVISION	UPDATED                 	STATUS  	CHART      	NAMESPACE
  dante-helm-redis	1       	Wed Aug  1 13:52:04 2018	DEPLOYED	redis-3.4.1	default  
  filled-bison    	1       	Wed Aug  1 10:40:03 2018	DELETED 	redis-3.4.1	default 
  ```

  ```sequence
  Repository->Chart: 从 Repository 搜索 Chart \nhelm search redis
  Chart->Release: Helm 安装 Chart，每个 Chart 运行一个 Release \nhelm install stable/redis
  ```

- **使用目的**

在原来项目中都是基于yaml文件来进行部署发布的，而目前项目大部分微服务化或者模块化，会分成很多个组件来部署，每个组件可能对应一个deployment.yaml、一个service.yaml、一个Ingress.yaml还可能存在各种依赖关系，这样一个项目如果有5个组件，很可能就有15个不同的yaml文件，这些yaml分散存放，如果某天进行项目恢复的话，很难知道部署顺序，依赖关系等，而所有这些包括都可以通过helm来进行解决。

- 基于yaml配置的集中存放
- 基于项目的打包
- 组件间的依赖

### 二. 安装

下载 https://github.com/helm/helm/releases

#### 1. linux

```sh
tar -zxvf helm-{version}-linux-amd64.tgz
mv linux-amd64/helm /usr/local/bin/helm
```

#### 2. MacOS

```bash
brew install kubernetes-helm
```

#### 3. 安装 Tiller

- 未开启 RBAC

```shell
## 安装
helm init

## 检查 helm version，结果如下
Client: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}

## 检查 tiller deployment，结果如下
kubectl get deployment -n kube-system
NAME                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
tiller-deploy          1         1         1            1           47d
```

- 开启 RBAC

```sh
## 创建 sa
kubectl create serviceaccount tiller -n kube-system

## 给 sa 绑定 cluster-admin 规则
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller

## 编辑 kube-system 中的 tiller-deploy
kubectl edit deploy tiller-deploy -n kube-system

...
spec:
  serviceAccount: tiller	## 在此行添加 serviceAccount: tiller
  containers:
  - env:
    - name: TILLER_NAMESPACE
      value: kube-system
    - name: TILLER_HISTORY_MAX
      value: "0"
...
```

#### 4. 卸载

```shell
kubectl delete deployment tiller-deploy --namespace kube-system
## 或者
helm reset
```

### 三. 架构及原理

![Helm通信架构](./Helm通信架构.png)

​	Helm Client通过 gRPC API 向 **Tiller 发出请求，Tiller 通过调用 **k8s 的 **Restful API** 来管理对应的 k8s 资源。

#### 1. Helm Client

主要作用有

- 在本地开发 chart
- 管理 chart 仓库
- 与 Tiller 服务器交互
- 在远程 Kubernetes 集群上安装 chart
- 查看 release 信息
- 升级或卸载已有的 release

#### 2. Tiller

​	对外暴露 gRPC API（`kubectl port-forward --namespace default $POD_NAME 6379:6379`  通过 Kubectl port-forward将tiller的端口映射到本地），供 Helm Client 调用。管理 Release Instance 的生命周期。

### 四. 操作实践



### 五. 参考资料

- https://docs.helm.sh/
- https://hub.kubeapps.com/