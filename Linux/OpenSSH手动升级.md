## Centos7 升级OpenSSH

**由于Centos7自带的openssh版本过低且存在漏洞，所以为了安全考虑我们需要升级openssh到最高版本。**

### 一. 关闭防火墙并检查

```shell
## 停止防火墙
systemctl stop firewalld

## 查看系统版本
cat /etc/redhat-release
## 查看软件版本
rpm -qa | grep openssh
ssh -V
```

### 二. 安装工具

```shell
yum install -y tar gcc make
```

### 三. 下载软件

阿里镜像站：https://mirrors.aliyun.com/pub/OpenBSD

```shell
curl -L -O --insecure https://nchc.dl.sourceforge.net/project/libpng/zlib/1.2.11/zlib-1.2.11.tar.gz
curl -L -O --insecure https://mirrors.sonic.net/pub/OpenBSD/OpenSSH/portable/openssh-9.3p1.tar.gz
curl -L -O --insecure https://www.openssl.org/source/openssl-1.1.1t.tar.gz
```

### 四. 解压并安装

```shell
### 解压
tar -zxvf zlib-1.2.11.tar.gz
tar -zxvf openssh-9.3p1.tar.gz
tar -zxvf openssl-1.1.1t.tar.gz

### 安装
cd zlib-1.2.11
./configure --prefix=/usr/local/zlib
make && make install

cd openssl-1.1.1t
./config --prefix=/usr/local/ssl -d shared
make && make install
echo '/usr/local/ssl/lib' >> /etc/ld.so.conf
ldconfig -v

cd openssh-9.3p1
./configure --prefix=/usr/local/openssh --with-zlib=/usr/local/zlib --with-ssl-dir=/usr/local/ssl
make && make install
```

### 五. 删除低版本OpenSSH

```
rpm -e --nodeps `rpm -qa | grep openssh`
```

### 六. 修改配置

`vi /usr/local/openssh/etc/sshd_config`

```ini
PermitRootLogin yes
PubkeyAuthentication yes
PasswordAuthentication yes
```

### 七. 复制文件到相应的系统文件夹

```shell
cd openssh-9.3p1/contrib/redhat/
cp sshd.init /etc/init.d/sshd
chkconfig --add sshd
cp /usr/local/openssh/etc/sshd_config /etc/ssh/sshd_config
cp /usr/local/openssh/sbin/sshd /usr/sbin/sshd
cp /usr/local/openssh/bin/ssh /usr/bin/ssh
cp /usr/local/openssh/bin/ssh-keygen /usr/bin/ssh-keygen
cp /usr/local/openssh/etc/ssh_host_ecdsa_key.pub /etc/ssh/ssh_host_ecdsa_key.pub
```

### 八. 启动sshd服务

```shell
systemctl status sshd
systemctl start sshd
systemctl status sshd

ssh -V

## 开启防火墙
systemctl start firewalld
```

