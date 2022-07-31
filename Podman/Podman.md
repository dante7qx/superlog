## Podman 说明

### 一. 概述

### 二. 安装说明

### 三. 使用说明

### 四. 常见问题

#### 1. 容器挂载数据卷无法访问

​	通过Podman或[Docker](https://so.csdn.net/so/search?q=Docker&spm=1001.2101.3001.7020)运行容器后，在容器内访问数据卷被拒绝	

```bash
$ mkdir -p /tmp/test
$ podman run -itd --name test -v /tmp/test/:/test:rw fedora:28 /bin/bash
$ podman exec test ls /test
ls: cannot open directory '/test': Permission denied
Error: exit status 2
```

原因及解决方案
首先这是个正常现象，因为容器运行在系统中，还会受到SELinux的安全限制。
经过我的尝试，这个现象不仅在Podman中存在，在较新版本的Docker中也是存在的。

遇到这个问题你可以暂时关闭SELinux：

sudo setenforce 0

当然这样做不够安全，或者你还可以采用更好的方法，为路径设定SELinux权限：

chcon -Rt svirt_sandbox_file_t /tmp/test/

### 五. 参考资料
- https://ywnz.com/linuxjc/7948.html
