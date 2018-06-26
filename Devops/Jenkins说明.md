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

### 三. 问题积累

#### 1. Jenkins中pipeline后台进程起不来的问题

https://blog.csdn.net/catoop/article/details/79637311

```sh
直接使用以前的方法（修改BUILD_ID），对于pipeline来说是没有效果的，现在你可以使用修改 JENKINS_NODE_COOKIE 的值来解决问题，这样后续结束的时候，后面的sh程序就不会被kill掉了。 
如下示例代码，是我的某个pipeline脚本片段：

......
stage('发布') {
    ansiColor('xterm') {
        sh "mkdir -p ${deploy_package_path}"
        sh "\\cp -Rf ${workspace_package_path} ${deploy_package_path}"
        // 停止Tomcat
        sh "${tomcat_home}/${JOB_NAME}/bin/kill.sh"
        // 启动Tomcat
        sh "JENKINS_NODE_COOKIE=dontKillMe ${tomcat_home}/${JOB_NAME}/bin/startup.sh"
        println("发布并重启Tomcat完成");
    }
}
```

#### 2. Groovy 脚本方法被限制

```groovy
import java.util.*;
import java.text.SimpleDateFormat;
def today()
{
    String str = "";
    SimpleDateFormat sdf = new SimpleDateFormat("yyyyMMdHHmmss");
    Calendar lastDate = Calendar.getInstance();
    lastDate.add(Calendar.MINUTE, 2);
    str = sdf.format(lastDate.getTime());
    return str;
}
def  dateNew = today()
node {
    stage('变量') {
        echo "Hello World - $dateNew - $BUILD_ID - $currentBuild.timeInMillis"
        echo "$currentBuild.durationString"
    }
}
```

https://stackoverflow.com/questions/38276341/jenkins-ci-pipeline-scripts-not-permitted-to-use-method-groovy-lang-groovyobject

- 方式一：http://<JENKINS_URL>/scriptApproval/，逐个授权。

- 方式二：去掉 Use Groovy Sandbox 的 Checkbox。（推荐）

- 方式三：

  1.安装插件： Permissive Script Security

  2.启动参数中禁止 script security，-Dpermissive-script-security.enabled=true。

```xml
<!-- jenkins.xml 中的配置如下 -->
<executable>..bin\java</executable>
<arguments>-Dpermissive-script-security.enabled=true -Xrs -Xmx4096m -Dhudson.lifecycle=hudson.lifecycle.WindowsServiceLifecycle -jar "%BASE%\jenkins.war" --httpPort=80 --webroot="%BASE%\war"</arguments>
```

### 四. Pipeline 案例 



### 五. 坑

参考 https://testerhome.com/topics/10328

### 六. 参考资料

- https://www.w3cschool.cn/jenkins/
- https://www.xncoding.com/2017/03/22/fullstack/jenkins02.html
- https://updates.jenkins-ci.org/download/plugins/（插件下载）
