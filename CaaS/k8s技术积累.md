## k8s 技术积累

### 一. Secrets

​	k8s Secret 用来存储少量敏感的数据，例如数据库、Gitlab的用户、密码。这样可以不在 Image 中去保留敏感信息，并且可以在多个 Pod 中共享这些 Secret。因为 k8s 使用 JWT 的方式进行加密，所以需要加密的数据必须先进行 base64 编码。参考 https://kubernetes.io/docs/concepts/configuration/secret

```shell
## 编码
echo -n "root" | base64			# => cm9vdA==
echo -n "123456" | base64		# => MTIzNDU2
## 解码
echo "cm9vdA==" | base64 -D
echo "MTIzNDU2" | base64 -D
```
- ***kubectl create secret.yaml***
```yaml
## secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitlabsecret
  namespace: dante
type: Opaque
data:
  username: cm9vdA==
  password: MTIzNDU2

## data 是 map
## key 字母、数字或 _ - .
## value 必须用 base64 编码
```

- **在 Pod 中使用**
  - 环境变量

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dante-healthcheck
  namespace: dante
spec:
  containers:
  - name: dante-boot-docker
    image: dante/springboot-docker
    imagePullPolicy: Never
    ports:
    - containerPort: 8080
    livenessProbe:
      httpGet:
        path: /docker
        port: 8080
      initialDelaySeconds: 30
      timeoutSeconds: 4
    env:
      - name: hello_msg
        valueFrom:
          secretKeyRef:
            name: gitlabsecret
            key: username
      - name: hello_x_info
        valueFrom:
          secretKeyRef:
            name: gitlabsecret
            key: password
```

### 二. PV / PVC

对于有状态的服务，k8s 提供了 Volume 和 Persistent Volume 来实现服务状态的保存。存储的状态主要有

- 服务的基本配置文件读取、密钥管理；
- 服务的存储状态、数据；
- 不同服务或应用间的共享数据；

![volume](./volume.png)

#### 1. PV 

​	由管理员设置的存储，它是群集的一部分。就像节点是集群中的资源一样，PV 也是集群中的资源。 PV 是 Volume 之类的卷插件，但具有独立于使用 PV 的 Pod 的生命周期。此 API 对象包含存储实现的细节，即 NFS、iSCSI 或特定于云供应商的存储系统。假如有一个独立的存储后端，底层实现可以是NFS、GlusterFS、Cinder、HostPath等等，可以使用PV从中划拨一部分资源用于kubernetes的存储需求，其生命周期不依赖于Pod，是一个独立存在的虚拟存储空间，但是不能直接被Pod的Volume挂载，此时需要用到PVC。

PV 的回收策略有三种

- **Retain**	—	**保留现场**，Kubernetes什么也不做。**（仅支持 nfs 和 hostPath）**
- **Delete**    —    K8s 会自动删除该PV及里面的数据。默认值。
- **Recycle**  —    K8s 会将PV里的数据删除，然后把PV的状态变成Available，又可以被新的PVC绑定使用。

#### 2. PVC

​	Pod使用PV资源是通过PVC来实现的，PVC可以理解为资源使用请求，一个Pod需要先明确使用的资源大小、访问方式，创建PVC申请提交到kubernetes中的PersistentVolume Controller，由其调度合适的PV来与PVC绑定，然后Pod中的Volume就可以通过PVC来使用PV的资源。

#### 3. StorageClass

- **静态 Provision**

  是管理员手动创建一堆PV，组成一个PV池，供PVC来绑定。一个PV创建完后状态会变成Available，等待被PVC绑定。一旦被PVC邦定，PV的状态会变成Bound，就可以被相应的Pod使用。Pod使用完后会释放PV，PV的状态变成Released。

```yaml
---
## pv - spiritdev4xvolume
apiVersion: v1
kind: PersistentVolume
metadata:
  name: spiritdev4xvolume
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: "100"
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: spiritdev4xvolume
    namespace: spiritdev
  mountOptions:
  - vers=3
  - proto=udp
  - nolock
  - noacl
  nfs:
    path: /nfs/spiritdev4xvolume
    server: 10.70.39.214
  persistentVolumeReclaimPolicy: Retain
  
--- 
## pvc - spiritdev4xvolume
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    pv.kubernetes.io/bind-completed: "yes"
  name: spiritdev4xvolume
  namespace: spiritdev
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: "100"
  volumeName: spiritdev4xvolume
```

- **动态 Provision**

​	通过 **StoraClass** 定义动态PV资源调度，动态 PV 不需要预先创建 PV，而是通过PersistentVolume Controller动态调度，根据PVC的资源请求，寻找 StorageClass 定义的符合要求的底层存储来分配资源。

#### 4. Demo

1) 创建 serviceaccount 和 rbac

```yaml
---
## serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
namespace: youna
metadata:
  name: nfs-client-provisioner
```

```yaml
---
## rbac.yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
--- 
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: youna
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: youna
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

- 将 host mount-anyuid 赋予 sa nfs-client-provisioner

```shell
oc adm policy add-scc-to-user hostmount-anyuid system:serviceaccount:youna:nfs-client-provisioner
```

2) 配置 **NFS-Client provisioner**

前提：NFS 目录已经创建好了

```yaml
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: harbor.poc.com/spirit/nfs-client-provisioner:v1
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes		## 镜像中已被写死，不能修改
          env:
            - name: PROVISIONER_NAME
              value: youna-nfs
            - name: NFS_SERVER
              value: 192.168.0.18
            - name: NFS_PATH
              value: /nfs/youna
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.0.18
            path: /nfs/youna
```

3) 创建 StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: youna-nfs-storage
provisioner: youna-nfs # 匹配 deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false" # When set to "false" your PVs will not be archived
                           # by the provisioner upon deletion of the PVC.
```

4) 测试

```yaml
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Mi
  storageClassName: youna-nfs-storage

---
kind: Pod
apiVersion: v1
metadata:
  name: test-pod
spec:
  containers:
  - name: test-pod
    image: busybox:latest
    command:
      - "/bin/sh"
    args:
      - "-c"
      - "touch /mnt/SUCCESS && exit 0 || exit 1"
    volumeMounts:
      - name: nfs-pvc
        mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: test-pvc
```

**结果：**

![pv~pvc](./pv~pvc.png)

**参考：**

- http://blog.51cto.com/forall/2135152
- https://www.kubernetes.org.cn/3462.html
- https://www.jianshu.com/p/7cc54c0626da
- https://cloud.tencent.com/info/a69596dd6dcc4be3488006b100b7dd91.html
- https://jimmysong.io/posts/kubernetes-persistent-volume/
- https://github.com/kubernetes-incubator/external-storage

### 三. PDB

​	PodDisruptionBudget，对于应用来说，有两种情况会导致业务的中断。即，不可预知的中断（Involuntary Disruption）和可预知的中断（Voluntary Disruption）。

- 不可预知的中断
  - 服务器的硬件故障或系统内核崩溃，导致节点宕机。（设置 replicas大于1）
  - 如果容器部署在VM，VM被误删了或者Hyperwisor出问题了。（考虑物理设备的 HA）
  - 集群出现了网络[脑裂](https://blog.csdn.net/varyall/article/details/80427606)。
  - 由于超配因子设置不当，导致计算资源不足。（设置应用容器的 request 值）

- 可预知的中断
  - 删除那些管理Pods的控制器，比如Deployment，RS，RC，StatefulSet。
  - 触发应用的滚动更新。
  - 直接批量删除Pods。
  - kubectl drain一个节点（节点下线、集群缩容）。

PDB 是用来解决 Voluntary Disruption 的情况的，对部署在 k8s 上的应用都可以创建一个 PDB，用来限制Voluntary Disruptions时最大可以down的副本数或者最少应该保持Available的副本数，以此来保证应用的高可用。对于 Deployment、RC、ReplicaSet 和 StatefulSet，PDB 要设置相同的 Selector。要注意同一个namespace下不同的PDB Object不要使用有重叠的Selectors。

```yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: zookeeper
```

**参考**

- https://cloud.tencent.com/developer/article/1096947

### 四. StatefulSets

​	k8s >= 1.9开启。RC、Deployment、DaemonSet都是面向无状态的服务，它们所管理的Pod的IP、名字，启停顺序等都是随机的。StatefulSet 用来管理所有有状态的集群服务，比如MySQL、MongoDB、Redis、Zookeeper集群等。

**StatefulSet 的特点**：

- 稳定的，唯一的网络标识 Headless Service，可以用来发现集群内部的其他成员。比如StatefulSet的名字叫 zk，那么第一个起来的 Pod 叫 zk-0，第二个叫 zk-1，依次类推。

- 启动或关闭时保证有序。Pod 启动顺序从 0 —> N-1，关闭顺序从 N-1 —> 0。Pod 死掉后，重新启动的 Pod 的名字不会改变。 

  ```sh
  $(podname).(headless server name).namespace.svc.cluster.local
  ```

- 每个Pod一个持久化存储，必须由PersistentVolume Provisioner根据请求的存储类**StorageClass**进行配置。

- 删除一个 StatefulSet，先将 StatefulSet 缩小到 0。

**Headless Service**

没有Cluster IP，用于为一个集群内部的每个成员提供一个唯一的DNS名字（通过 Service 定义的 selectors），用于集群内部成员之间通信 。

因为没有ClusterIP，kube-proxy 并不处理此类服务，因为没有load balancing或 proxy 代理设置，在访问服务的时候回返回后端的全部的Pods IP地址，主要用于开发者自己根据pods进行负载均衡器的开发(设置了selector)。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: zk-hs
  labels:
    app: zk
spec:
  ports:
  - port: 2888
    name: server
  - port: 3888
    name: leader-election
  clusterIP: None
  selector:
    app: zk
```

**参考：**

- https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
- http://blog.51cto.com/newfly/2140004

#### 1. Zookeeper

**前提条件**：先创建好 StorageClass，必须使用 PV 确保数据不丢失。

**不足：**在ZooKeeper 3.4.10 中，无法以安全的方式更新集合的成员资格。StatefulSet 本身支持 Scale。

```yaml
---
### headless service
apiVersion: v1
kind: Service
metadata:
  name: zk-svc
  labels:
    app: zk-svc
spec:
  ports:
  - port: 2181                  ## 仅用于同 namespace 下访问, zk-svc:2181
    name: client            
  - port: 2888
    name: server
  - port: 3888
    name: leader-election
  clusterIP: None
  selector:
    app: zk

---
### 配置信息，使用 ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: zk-cm
data:
  jvm.heap: "512M"
  tick: "2000"
  init: "10"
  sync: "5"
  client.cnxns: "60"
  snap.retain: "3"
  purge.interval: "0"
  
---
### 使用 PDB，确保应用最少的可用实例
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
spec:
  selector:
    matchLabels:
      app: zk
  minAvailable: 2
  
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: zk
  name: zk
spec:
  replicas: 3
  selector:
    matchLabels:
      app: zk
  serviceName: zk-svc
  template:
    metadata:
      labels:
        app: zk
    spec:
      affinity:
        podAntiAffinity:
          # requiredDuringSchedulingIgnoredDuringExecution:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 80
              podAffinityTerm: 
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - zk
                topologyKey: kubernetes.io/hostname
      containers:
        - name: spiritzk
          image: 'dante2012/k8s-zookeeper:v3'
          imagePullPolicy: Always
          ports:
            - containerPort: 2181
              name: client
              protocol: TCP
            - containerPort: 2888
              name: server
              protocol: TCP
            - containerPort: 3888
              name: leader-election
              protocol: TCP
          command:
            - sh
            - '-c'
            - zkGenConfig.sh && zkServer.sh start-foreground
          env:
            - name: ZK_REPLICAS
              value: '3'
            - name: ZK_HEAP_SIZE
              valueFrom:
                configMapKeyRef:
                  key: jvm.heap
                  name: zk-cm
            - name: ZK_TICK_TIME
              valueFrom:
                configMapKeyRef:
                  key: tick
                  name: zk-cm
            - name: ZK_INIT_LIMIT
              valueFrom:
                configMapKeyRef:
                  key: init
                  name: zk-cm
            - name: ZK_SYNC_LIMIT
              valueFrom:
                configMapKeyRef:
                  key: tick
                  name: zk-cm
            - name: ZK_MAX_CLIENT_CNXNS
              valueFrom:
                configMapKeyRef:
                  key: client.cnxns
                  name: zk-cm
            - name: ZK_SNAP_RETAIN_COUNT
              valueFrom:
                configMapKeyRef:
                  key: snap.retain
                  name: zk-cm
            - name: ZK_PURGE_INTERVAL
              valueFrom:
                configMapKeyRef:
                  key: purge.interval
                  name: zk-cm
            - name: ZK_CLIENT_PORT
              value: '2181'
            - name: ZK_SERVER_PORT
              value: '2888'
            - name: ZK_ELECTION_PORT
              value: '3888'
          readinessProbe:
            exec:
              command:
                - zkOk.sh
            initialDelaySeconds: 10
            timeoutSeconds: 5
          livenessProbe:
            exec:
              command:
                - zkOk.sh
            initialDelaySeconds: 10
            timeoutSeconds: 5
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
          volumeMounts:
            - mountPath: /var/lib/zookeeper
              name: datadir
      restartPolicy: Always
      securityContext:
        fsGroup: 1000
        runAsUser: 1000
  volumeClaimTemplates:
    - metadata:
        name: datadir
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 2Gi
        storageClassName: youna-nfs-storage		## 先创建好 SotrageClass
        
---
## Headless Service 没有Cluster Ip，因此不能用于外界访问。
## 所以，我们还需要创建一个Service，专用于为 Zookeeper 集群提供访问和负载均：
apiVersion: v1
kind: Service
metadata:
  labels:
    name: zk-access-svc
  name: zk-access-svc
spec:
  ports:
    - nodePort: 30082
      port: 2181
      protocol: TCP
      targetPort: 2181
  selector:
    app: zk
  type: NodePort
```

```shell
### zkOK.sh
#!/usr/bin/env bash

ZK_CLIENT_PORT=${ZK_CLIENT_PORT:-2181}
OK=$(echo ruok | nc 127.0.0.1 $ZK_CLIENT_PORT)
if [ "$OK" == "imok" ]; then
	exit 0
else
	exit 1
fi

### zkGenConfig.sh
#!/usr/bin/env bash
ZK_USER=${ZK_USER:-"zookeeper"}
ZK_LOG_LEVEL=${ZK_LOG_LEVEL:-"INFO"}
ZK_DATA_DIR=${ZK_DATA_DIR:-"/var/lib/zookeeper/data"}
ZK_DATA_LOG_DIR=${ZK_DATA_LOG_DIR:-"/var/lib/zookeeper/log"}
ZK_LOG_DIR=${ZK_LOG_DIR:-"var/log/zookeeper"}
ZK_CONF_DIR=${ZK_CONF_DIR:-"/opt/zookeeper/conf"}
ZK_CLIENT_PORT=${ZK_CLIENT_PORT:-2181}
ZK_SERVER_PORT=${ZK_SERVER_PORT:-2888}
ZK_ELECTION_PORT=${ZK_ELECTION_PORT:-3888}
ZK_TICK_TIME=${ZK_TICK_TIME:-2000}
ZK_INIT_LIMIT=${ZK_INIT_LIMIT:-10}
ZK_SYNC_LIMIT=${ZK_SYNC_LIMIT:-5}
ZK_HEAP_SIZE=${ZK_HEAP_SIZE:-2G}
ZK_MAX_CLIENT_CNXNS=${ZK_MAX_CLIENT_CNXNS:-60}
ZK_MIN_SESSION_TIMEOUT=${ZK_MIN_SESSION_TIMEOUT:- $((ZK_TICK_TIME*2))}
ZK_MAX_SESSION_TIMEOUT=${ZK_MAX_SESSION_TIMEOUT:- $((ZK_TICK_TIME*20))}
ZK_SNAP_RETAIN_COUNT=${ZK_SNAP_RETAIN_COUNT:-3}
ZK_PURGE_INTERVAL=${ZK_PURGE_INTERVAL:-0}
ID_FILE="$ZK_DATA_DIR/myid"
ZK_CONFIG_FILE="$ZK_CONF_DIR/zoo.cfg"
LOGGER_PROPS_FILE="$ZK_CONF_DIR/log4j.properties"
JAVA_ENV_FILE="$ZK_CONF_DIR/java.env"
HOST=`hostname -s`
DOMAIN=`hostname -d`

function print_servers() {
    for (( i=1; i<=$ZK_REPLICAS; i++ ))
    do
        echo "server.$i=$NAME-$((i-1)).$DOMAIN:$ZK_SERVER_PORT:$ZK_ELECTION_PORT"
    done
}

function validate_env() {
    echo "Validating environment"

    if [ -z $ZK_REPLICAS ]; then
        echo "ZK_REPLICAS is a mandatory environment variable"
        exit 1
    fi

    if [[ $HOST =~ (.*)-([0-9]+)$ ]]; then
        NAME=${BASH_REMATCH[1]}
        ORD=${BASH_REMATCH[2]}
    else
        echo "Failed to extract ordinal from hostname $HOST"
        exit 1
    fi

    MY_ID=$((ORD+1))
    echo "ZK_REPLICAS=$ZK_REPLICAS"
    echo "MY_ID=$MY_ID"
    echo "ZK_LOG_LEVEL=$ZK_LOG_LEVEL"
    echo "ZK_DATA_DIR=$ZK_DATA_DIR"
    echo "ZK_DATA_LOG_DIR=$ZK_DATA_LOG_DIR"
    echo "ZK_LOG_DIR=$ZK_LOG_DIR"
    echo "ZK_CLIENT_PORT=$ZK_CLIENT_PORT"
    echo "ZK_SERVER_PORT=$ZK_SERVER_PORT"
    echo "ZK_ELECTION_PORT=$ZK_ELECTION_PORT"
    echo "ZK_TICK_TIME=$ZK_TICK_TIME"
    echo "ZK_INIT_LIMIT=$ZK_INIT_LIMIT"
    echo "ZK_SYNC_LIMIT=$ZK_SYNC_LIMIT"
    echo "ZK_MAX_CLIENT_CNXNS=$ZK_MAX_CLIENT_CNXNS"
    echo "ZK_MIN_SESSION_TIMEOUT=$ZK_MIN_SESSION_TIMEOUT"
    echo "ZK_MAX_SESSION_TIMEOUT=$ZK_MAX_SESSION_TIMEOUT"
    echo "ZK_HEAP_SIZE=$ZK_HEAP_SIZE"
    echo "ZK_SNAP_RETAIN_COUNT=$ZK_SNAP_RETAIN_COUNT"
    echo "ZK_PURGE_INTERVAL=$ZK_PURGE_INTERVAL"
    echo "ENSEMBLE"
    print_servers
    echo "Environment validation successful"
}

function create_config() {
    rm -f $ZK_CONFIG_FILE
    echo "Creating ZooKeeper configuration"
    echo "#This file was autogenerated by k8szk DO NOT EDIT" >> $ZK_CONFIG_FILE
    echo "clientPort=$ZK_CLIENT_PORT" >> $ZK_CONFIG_FILE
    echo "dataDir=$ZK_DATA_DIR" >> $ZK_CONFIG_FILE
    echo "dataLogDir=$ZK_DATA_LOG_DIR" >> $ZK_CONFIG_FILE
    echo "tickTime=$ZK_TICK_TIME" >> $ZK_CONFIG_FILE
    echo "initLimit=$ZK_INIT_LIMIT" >> $ZK_CONFIG_FILE
    echo "syncLimit=$ZK_SYNC_LIMIT" >> $ZK_CONFIG_FILE
    echo "maxClientCnxns=$ZK_MAX_CLIENT_CNXNS" >> $ZK_CONFIG_FILE
    echo "minSessionTimeout=$ZK_MIN_SESSION_TIMEOUT" >> $ZK_CONFIG_FILE
    echo "maxSessionTimeout=$ZK_MAX_SESSION_TIMEOUT" >> $ZK_CONFIG_FILE
    echo "autopurge.snapRetainCount=$ZK_SNAP_RETAIN_COUNT" >> $ZK_CONFIG_FILE
    echo "autopurge.purgeInterval=$ZK_PURGE_INTERVAL" >> $ZK_CONFIG_FILE

    if [ $ZK_REPLICAS -gt 1 ]; then
        print_servers >> $ZK_CONFIG_FILE
    fi

    echo "Wrote ZooKeeper configuration file to $ZK_CONFIG_FILE"
}

function create_data_dirs() {
    echo "Creating ZooKeeper data directories and setting permissions"

    if [ ! -d $ZK_DATA_DIR  ]; then
        mkdir -p $ZK_DATA_DIR
        chown -R $ZK_USER:$ZK_USER $ZK_DATA_DIR
    fi

    if [ ! -d $ZK_DATA_LOG_DIR  ]; then
        mkdir -p $ZK_DATA_LOG_DIR
        chown -R $ZK_USER:$ZK_USER $ZK_DATA_LOG_DIR
    fi

    if [ ! -d $ZK_LOG_DIR  ]; then
        mkdir -p $ZK_LOG_DIR
        chown -R $ZK_USER:$ZK_USER $ZK_LOG_DIR
    fi

    if [ ! -f $ID_FILE ]; then
        echo $MY_ID >> $ID_FILE
    fi

    echo "Created ZooKeeper data directories and set permissions in $ZK_DATA_DIR"
}

function create_log_props () {
    rm -f $LOGGER_PROPS_FILE
    echo "Creating ZooKeeper log4j configuration"
    echo "zookeeper.root.logger=CONSOLE" >> $LOGGER_PROPS_FILE
    echo "zookeeper.console.threshold="$ZK_LOG_LEVEL >> $LOGGER_PROPS_FILE
    echo "log4j.rootLogger=\${zookeeper.root.logger}" >> $LOGGER_PROPS_FILE
    echo "log4j.appender.CONSOLE=org.apache.log4j.ConsoleAppender" >> $LOGGER_PROPS_FILE
    echo "log4j.appender.CONSOLE.Threshold=\${zookeeper.console.threshold}" >> $LOGGER_PROPS_FILE
    echo "log4j.appender.CONSOLE.layout=org.apache.log4j.PatternLayout" >> $LOGGER_PROPS_FILE
    echo "log4j.appender.CONSOLE.layout.ConversionPattern=%d{ISO8601} [myid:%X{myid}] - %-5p [%t:%C{1}@%L] - %m%n" >> $LOGGER_PROPS_FILE
    echo "Wrote log4j configuration to $LOGGER_PROPS_FILE"
}

function create_java_env() {
    rm -f $JAVA_ENV_FILE
    echo "Creating JVM configuration file"
    echo "ZOO_LOG_DIR=$ZK_LOG_DIR" >> $JAVA_ENV_FILE
    echo "JVMFLAGS=\"-Xmx$ZK_HEAP_SIZE -Xms$ZK_HEAP_SIZE\"" >> $JAVA_ENV_FILE
    echo "Wrote JVM configuration to $JAVA_ENV_FILE"
}

validate_env && create_config && create_log_props && create_data_dirs && create_java_env
```

**参考**

- https://github.com/kubernetes/contrib/blob/master/statefulsets/zookeeper/README.md
- https://blog.csdn.net/xiaolang85/article/details/13021339
- https://www.kubernetes.org.cn/1130.html

#### 2. Eureka

针对 Spring Cloud框架中的服务发现组件 Eureka，使用 k8s 部署高可用。k8s pod 要在部署的时候就知道其他 pod ip 或者域名，保证在启动pod的时候可以互相寻址。

- **Java**

```yaml
server:
  port: 8761
spring:
  security:
    user:
      name: ${REGISTER_USER:dante}
      password: ${REGISTER_PWD:123456}
  application:
    name: ${EUREKA_APPLICATION_NAME:eureka-server}
  cloud:
    config:
      enabled: false
eureka: 
  server:
    enable-self-preservation: false # 关闭自我保护模式
    peer-node-read-timeout-ms: 1000 # 节点之间读取超时时间（毫秒）
  instance:
    hostname: ${EUREKA_HOST_NAME:localhost} # 服务主机名
    prefer-ip-address: false # 猜测主机名时，优先选择ip。默认为false，使用hostname作为主机名
  client:
    register-with-eureka: ${BOOL_REGISTER:false} # 是否把服务中心本身当做eureka client 注册。默认为true
    fetch-registry: ${BOOL_FETCH:false} # 是否拉取 eureka server 的注册信息。 默认为true
    service-url:
      defaultZone: ${EUREKA_URL_LIST:http://${spring.security.user.name}:${spring.security.user.password}@${eureka.instance.hostname}:${server.port}/eureka/}
  environment: ${RUN_ENV:spirit-dev}
  datacenter: ${RUN_DATA:spirit-cloud}
```

- **Docker**

```dockerfile
FROM java:8

LABEL author="dante"
LABEL email="sunchao.0129@163.com"

ENV JAVA_OPTS="-Xms1g -Xmx1g"
ENV TZ=Asia/Shanghai
ENV APPS_HOME=/AppServer

RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone \
	&& mkdir -p $APPS_HOME/ \
	&& rm /bin/sh && ln -s /bin/bash /bin/sh 

WORKDIR $APPS_HOME/	 
	 
ADD micro-eureka-k8s-0.0.1-SNAPSHOT.jar app.jar	 
ADD init.sh init.sh

RUN sh -c 'touch app.jar'

EXPOSE 8761

ENTRYPOINT ["/bin/bash","-c","source init.sh"] 
```

- **init.sh**

```bash
#!/usr/bin/env bash

POSTFIX="svc.cluster.local"
EUREKA_HOST_NAME="$MY_POD_NAME.$MY_INNER_SERVICE.$MY_POD_NAMESPACE.$POSTFIX"
export EUREKA_HOST_NAME=$EUREKA_HOST_NAME
REGISTER_USER="$EUREKA_USER"
REGISTER_PWD="$EUREKA_PWD"
BOOL_REGISTER="false"
BOOL_FETCH="false"

if [ $EUREKA_REPLICAS == 1 ]; then
	EUREKA_URL_LIST="http://$EUREKA_HOST_NAME:8761/eureka/,"
else 
	BOOL_REGISTER="true"
	BOOL_FETCH="true"
	for ((i=0; i<$EUREKA_REPLICAS; i ++))
    do
    	if [ $MY_POD_NAME != "$EUREKA_APPLICATION_NAME-$i" ]; then
	        temp="http://$REGISTER_USER:$REGISTER_PWD@$EUREKA_APPLICATION_NAME-$i.$MY_INNER_SERVICE.$MY_POD_NAMESPACE.$POSTFIX:8761/eureka/,"
	        EUREKA_URL_LIST="$EUREKA_URL_LIST$temp"
        fi
    done
fi

## 设置 eureka 的注册 url
EUREKA_URL_LIST=${EUREKA_URL_LIST%?}
echo "EUREKA_URL_LIST is $EUREKA_URL_LIST"
export EUREKA_URL_LIST=$EUREKA_URL_LIST
export BOOL_FETCH=$BOOL_FETCH
export BOOL_REGISTER=$BOOL_REGISTER
export REGISTER_USER=$REGISTER_USER
export REGISTER_PWD=$REGISTER_PWD

java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar app.jar
```

- **k8s**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: eureka-svc-internal
  labels:
    app: eureka-svc-internal
  namespace: spiritdev
spec:
  clusterIP: None
  ports:
  - port: 8761
    name: server
  selector:
    app: eureka
      
---
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: eureka-pdb
  namespace: spiritdev
spec:
  selector:
    matchLabels:
      app: eureka
  minAvailable: 1

---
apiVersion: apps/v1
kind: StatefulSet
metadata: 
  name: eureka
  namespace: spiritdev
spec: 
  selector: 
    matchLabels:
      app: eureka
  serviceName: eureka-svc-internal
  replicas: "3"
  template:
    metadata:
      labels:
        app: eureka
    spec:
      terminationGracePeriodSeconds: 10
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 80
              podAffinityTerm: 
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - eureka
                topologyKey: kubernetes.io/hostname
      containers:
        - name: eureka-cluster
          image: harbor.xiaohehe.com/spiritdev/eureka-cluster:v1
          imagePullPolicy: IfNotPresent
          imagePullSecrets:
            - name: spiritdev-pushsecret-harbor-xiaohehe-com    ## 设置推送权限（可选）或者 oc secrets link default spiritdev-pushsecret-harbor-xiaohehe-com --for=pull
          ports: 
            - containerPort: 8761
              name: server
              protocol: TCP
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: 500m
              memory: 1Gi
          env: 
            - name: MY_NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: MY_POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: MY_POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: MY_POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: MY_INNER_SERVICE
              value: eureka-svc-internal
            - name: EUREKA_APPLICATION_NAME
              value: eureka
            - name: EUREKA_REPLICAS
              value: "3"
            - name: EUREKA_USER
              value: dante
            - name: EUREKA_PWD
              value: "123456"
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8761
            initialDelaySeconds: 10
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 8761
            initialDelaySeconds: 10
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  labels:
    name: eureka-external-svc
  name: eureka-external-svc
spec:
  ports:
    - name: http-8761
      port: 8761
      protocol: TCP
      targetPort: 8761
  selector:
    app: eureka
  type: ClusterIP  
```

**参考**

- https://blog.csdn.net/gujian2517/article/details/82491101

#### 3. Redis

**参考**

- https://www.jianshu.com/p/65c4baadf5d9

### 五. 探针

Kubernetes提供了两种探针（Probe，支持exec、tcp和httpGet方式）来探测容器的状态。

#### 1. liveness

用来确定是否重启容器，当程序处于运行状态但无法执行进一步操作时，需要有一个状态去通知 k8s 重启应用。

```yaml
spec:
  containers:
  - name: liveness
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
          - name: X-Custom-Header
            value: x-dante
      initialDelaySeconds: 3
      periodSeconds: 3
```

#### 2. readiness

用来确定 Pod 是否已经就绪，可以接受流量。只有当Pod中的容器都处于就绪状态时kubelet才会认定该Pod处于就绪状态。只有就绪状态的 Pod 才会做为 Service 后端负载的一部分。

```yaml
readinessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

| 配置项              |                                                              |
| ------------------- | ------------------------------------------------------------ |
| initialDelaySeconds | 容器启动后第一次执行探测是需要等待多少秒（应用启动时间）     |
| periodSeconds       | 执行探测的频率。默认是10秒，最小1秒。                        |
| timeoutSeconds      | 探测超时时间。默认1秒，最小1秒。                             |
| successThreshold    | 探测失败后，最少连续探测成功多少次才被认定为成功。默认是1。对于liveness必须是1。最小值是1。 |
| failureThreshold    | 探测成功后，最少连续探测失败多少次才被认定为失败。默认是3。最小值是1。 |
| **针对 httpGet**    |                                                              |
| host                | 连接的主机名。默认是 pod 的 IP。                             |
| scheme              | 连接的schema，默认HTTP。                                     |
| path                | 访问的 HTTP server的path。                                   |
| port                | 访问的容器的端口名字或者端口号。端口号必须介于1和65525之间。 |
| httpHeaders         | 自定义请求的header。HTTP运行重复的header。                   |

