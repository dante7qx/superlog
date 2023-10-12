## Linux 命令

### 一. firewalld 和 iptables

#### 1. firewalld

- https://www.cnblogs.com/huchong/p/9669737.html
- https://wangchujiang.com/linux-command/c/firewall-cmd.html	

#### 2. 字符串

- https://blog.csdn.net/universe_hao/article/details/52640321

#### 3. 文件权限

**使用chown命令可以修改文件或目录所属的用户：**

​	`chown 用户 目录或文件名`

**使用chgrp命令可以修改文件或目录所属的组：**

​	`chgrp 组 目录或文件名`

**文件读、写、执行权限**

​	`chmod u+x file`

#### 4. Watch命令

周期性地运行其他命令并实时显示输出结果

```bash
## Monitor files in the current directory:
    watch ls

## Monitor disk space and highlight the changes:
    watch -d df

## Monitor "node" processes, refreshing every 3 seconds:
    watch -n 3 "ps aux | grep node"
```



### 二. 参考

- https://mp.weixin.qq.com/s/85aHzjm01YD5ZCTVUt8PsA