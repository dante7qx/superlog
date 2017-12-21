## Jenkins说明

### 一. 启动

**直接启动**

*java  -jar jenkins.war*

**修改端口**

*java -jar jenkins.war --ajp13Port=-1 --httpPort=8089*	

**后台启动**	

*nohup java -jar jenkins.war &*	

### 二. 通知构建

**Jenkins**

构建触发器 -> 触发远程构建，设置Token，例如Token=123456，记住下面的地址

`JENKINS_URL/job/git-test/build?token=123456`

**Github、Gitlab 下**

repository -> Settings -> webhooks -> Add Webhook

```markdown
[Payload URL]

http://<jenkinsuser>:<jenkinsuser-api-token>@JENKINS_URL/job/git-test/build?token=123456
```

**备注**

Jenkins错误：No valid crumb was included in the request

分析：

```
jenkins在http请求头部中放置了一个名为.crumb的token。在使用了反向代理，并且在jenkins设置中勾选了“防止跨站点请求伪造（Prevent Cross Site Request Forgery exploits）”之后此token会被转发服务器apache/nginx认为是不合法头部而去掉。导致跳转失败。
```

处理：

```xml
1.在apache/nginx中设置ignore_invalid_headers。
2.在jenkins全局安全设置中取消勾选“防止跨站点请求伪造（Prevent Cross Site Request Forgery exploits）”。
```



