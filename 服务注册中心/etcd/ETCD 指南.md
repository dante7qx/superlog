## ETCD

### 一. 概述

etcd 是一个分布式键值存储数据库，CoreOS 团队用 Go 编写。通过分布式锁、leader election、Write Barriers 来实现可靠的分布式协作。

### 二. 安装

1. 下载安装包 https://github.com/etcd-io/etcd/releases
2. 配置

```shell
## 设置环境变量
export ETCDCTL_API=3
## 将 etcd 和 etcdctl 加入 PATH
cd /usr/local/bin
ln -s etcd-v3.3.13/etcd etcd
ln -s etcd-v3.3.13/etcdctl etcdctl
```

### 三. 集群

**参数详解**

```yaml
--name: 节点名称，集群唯一，可以使用 hostname
--data-dir: 服务运行数据保存的路径，默认为 ${name}.etcd
--heartbeat-interval: leader向followers发送心跳的周期，默认是 100ms
--eletion-timeout: 重新投票的超时时间，若followers在该间隔未收到心跳包，会触发重新投票。默认1000ms
--listen-peer-urls: 与对等节点通信的url，例如：http://ip:2380
--listen-client-urls: etcd服务器自己监听用的，即监听那个IP和Port
--advertise-client-urls: 与 Client（etcdctl/curl）进行交互时请求的url
--initial-cluster: 集群中所有节点的信息（需要实现知道集群节点数）
--initial-advertise-peer-urls: 对等节点通信url，该 url 其他对等节点会被告知
--initial-cluster-stat: 初始集群状态 ('new' or 'existing')
--initial-cluster-token: 创建集群的token，这个值每个集群保持唯一
--enable-pprof: 开启Http运行时分析，client-url + "/debug/pprof/"。默认 false，不开启
## 参考：https://www.jianshu.com/p/7bbef1ca9733
```

#### 1. 单机集群

```shell
## 单节点集群，etcd member 监听 endpoint localhost:2379
./etcd

## 多节点集群
## 1. 安装 goreman
go get github.com/mattn/goreman
## 2. 创建 Procfile 文件，参考 Technique/Etcd/soft/Procfile/Procfile 
## 3. 使用 goreman 创建，在 Procfile 所在目录下执行
##    监听 endpoint localhost:2379, localhost:22379, and localhost:32379
goreman start (或 goreman -f Procfile start)
```

```shell
## 查看集群
etcdctl --write-out=table --endpoints=localhost:2379 member list

etcdctl --endpoints 127.0.0.1:2379,127.0.0.1:22379,127.0.0.1:32379 endpoint status  --write-out="table"
```

#### 2. 多机集群 

参考：https://github.com/etcd-io/etcd/blob/master/Documentation/demo.md

- **固定节点式**

  参考: Technique/Etcd/conf/cluster.sh

- **注册中心式**

  固定节点的方式搭建集群，既要知道集群大小，又要知道集群中所有节点的 IP。但很多时候，我们只知道集群的大小，但不知道集群各个节点的 IP 信息，从而无法使用 `--initial-cluster`。此时，就需要通过 discovery 的方式来搭建 etcd 集群。

  > etcd discovery 和 DNS discovery

  ​	etcd discovery 的方式，是依赖另外一个ETCD集群，在该集群中创建一个目录，并在该目录中创建一个_config的子目录，并且在该子目录中增加一个size节点，指定集群的节点数目。

  - 公共etcd discovery服务

    通过CoreOS提供的公共discovery服务申请token

  ```shell
  ## 1. 获取集群标识，size 表示集群大小，Token 是 72213befc76d994187083064b4e531d5
  $ curl -w "\n" 'https://discovery.etcd.io/new?size=3'
  https://discovery.etcd.io/72213befc76d994187083064b4e531d5
  
  ## 2. 去掉 --initial-cluster、--initial-cluster-token、--initial-cluster-state
  ## 每个节点上执行
  NAME_1=public-discovery-etcd1
  PEER_1=http://127.0.0.1:12380
  CLIENT_URL=http://127.0.0.1:2379
  DISCOVERY=https://discovery.etcd.io/324bce8918939c46255ed9dcbc1810f5
  
  etcd --name ${NAME_1} \
  --data-dir ./public-discovery-cluster/data/${NAME_1}.etcd \
  --listen-client-urls ${CLIENT_URL} \
  --advertise-client-urls ${CLIENT_URL} \
  --listen-peer-urls ${PEER_1} \
  --initial-advertise-peer-urls ${PEER_1} \
  --discovery ${DISCOVERY} \
  --enable-pprof \
  --log-output 'stderr'
  ```

  - 自定义etcd discovery服务

    利用一个已有的`etcd`集群来提供discovery服务，从而搭建一个新的`etcd`集群。

  ```shell
  ## 1. 已创建集群 127.0.0.1 2379，向已有集群申请创建一个特殊的 Key
  ## 7654321 用来做discovery的token
  $ curl http://127.0.0.1:2379/v2/keys/discovery/7654321/_config/size -d value=3
  {"action":"create","node":{"key":"/discovery/7654321/_config/size/00000000000000000008","value":"3","modifiedIndex":8,"createdIndex":8}}
  
  ## 2. 同公共 etcd discovery 服务
  ```

  **说明： **若实际启动的etcd节点个数大于discovery token创建时指定的size，多余的节点会自动变为proxy节点。

  **etcd proxy模式简介：**

  作为反向代理把客户的请求转发给可用的etcd集群，新节点加入集群如果核心节点数已满足要求，则自动转化为proxy模式，此项不会在节点不足时逆向转化为实际节点。

#### 3. TLS 集群

ectd 支持 TLS 对 client - server 和 server - server 的通信身份认证。要启动认证，首先要为集群中的每一个成员创建和签名新的 Key Pair。

##### 1. 安装 cfssl

```shell
$ git clone https://github.com/cloudflare/cfssl.git $GOPATH/src/github.com/cloudflare/cfssl
$ cd $GOPATH/src/github.com/cloudflare/cfssl
$ make
```

##### 2. 生产证书

```markdown
** client  certificate **
  Server 对 Client 的身份进行认证，例如 etcdctl、ectd proxy、docker client。
** server certificate **
  Server 使用，Client 验证 Server 的身份。
** peer certificate **
  etcd member 互信通信。
```

- 1) 生成 CA 证书和私钥

  - 编写 ca-csr.json

  ```json
  {
      "CN": "Spirit CA",
      "key": {
          "algo": "rsa",
          "size": 4096
      },
      "names": [{
          "C": "CN",
          "ST": "BeiJing",
          "L": "BeiJing",
          "O": "etcd server",
          "OU": "spirit cloud"
      }]
  }
  ```

  - 生成证书

  ```shell
  $ cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
  
  ## 生成结果 ca-key.pem、ca.csr、ca.pem
  ## ca-key.pem 和 ca.pem 很重要，用于 Spirit CA 下创建任何类型的证书
  ```

  - 创建 ca-config.json

    可定义多个 profile，分别设置 expiry 和 usages。43800h = 5年，usages 中

    - **signing** 表示此CA证书可以用于签名其他证书，ca.pem中的`CA=TRUE`
    - **server auth** 表示TLS Server Authentication
    - **client auth **表示TLS Client Authentication

  ```json
  {
      "signing": {
          "default": {
              "expiry": "43800h"
          },
          "profiles": {
              "server": {
                  "expiry": "43800h",
                  "usages": [
                      "signing",
                      "key encipherment",
                      "server auth"
                  ]
              },
              "client": {
                  "expiry": "43800h",
                  "usages": [
                      "signing",
                      "key encipherment",
                      "client auth"
                  ]
              },
              "peer": {
                  "expiry": "43800h",
                  "usages": [
                      "signing",
                      "key encipherment",
                      "server auth",
                      "client auth"
                  ]
              }
          }
      }
  }
  ```

- 2）生成 etcd 证书和私钥

  - 创建 etcd csr 配置 etcd-csr.json

  ```json
  {
      "CN": "etcd",
      "hosts": [
          "127.0.0.1",
          "127.0.0.1",
          "127.0.0.1",
          "peer1",
          "peer2",
          "peer3"
      ],
      "key": {
          "algo": "rsa",
          "size": 4096
      },
      "names": [{
          "C": "CN",
          "ST": "BeiJing",
          "L": "BeiJing",
          "O": "etcd server",
          "OU": "spirit cloud"
      }]
  }
  ```

  - 生成证书

  ```shell
  cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer etcd-csr.json | cfssljson -bare etcd
  ## 生成结果 etcd-key.pem、etcd.pem、etcd.csr
  ```

##### 3. 配置 etcd

配置TLS etcd 集群，需要 ca.pem（CA证书）、etcd-key.pem（etcd 私钥）、etcd.pem（etcd证书）。

```shell
## 相关配置参数
--cert-file=etcd.pem \
--key-file=etcd-key.pem \
--peer-cert-file=etcd.pem \
--peer-key-file=etcd-key.pem \
--trusted-ca-file=ca.pem \
--peer-trusted-ca-file=ca.pem \
-- url 均使用 https
```

验证 

```shell
etcdctl --write-out=table --cert=./ssl/etcd.pem --key=./ssl/etcd-key.pem --cacert=./ssl/ca.pem --endpoints=https://127.0.0.1:2379,https://127.0.0.1:22379,https://127.0.0.1:32379 member list
```

### 四. 原理

参考：https://draveness.me/etcd-introduction

### 五. 实战



### 六. 参考文档

- https://github.com/etcd-io/etcd

- https://www.hi-linux.com/posts/19457.html

- https://www.hi-linux.com/posts/49138.html

- https://www.cnblogs.com/lishijia/p/5412872.html

- TLS

  - https://github.com/coreos/docs/blob/master/os/generate-self-signed-certificates.md

  - https://www.jianshu.com/p/d7e53895338f （openssl）
  - https://blog.frognew.com/2017/04/install-etcd-cluster.html
  - [http://www.voidcn.com/article/p-duwtseyn-bsb.html](http://www.voidcn.com/article/p-duwtseyn-bsb.html)

- **Write Barriers：** https://yq.aliyun.com/articles/18103