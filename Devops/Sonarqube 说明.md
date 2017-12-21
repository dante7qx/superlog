## Sonarqube

### 1.Sonarqube是什么 

​	SonarQube能够提供对代码的一整套检查扫描和分析功能，拥有一套服务器端程序，然后再通过客户端或者别的软件的插件的形式完成对各开发环境和软件的支持。	

- 对编程语言的支持非常广泛，包括C、C++、Java、Objective C、Python、JavaScript、PHP、C#、Swift、Erlang、Groovy等众多语言
- 提供了对HTML、CSS、JSON、XML、CSV、SQL、JSP/JSF等类型的文档的支持
- 提供了以FindBugs、PMD、CheckStyle方式执行代码分析和测试检查的功能
- 登录认证方式支持LDAP、Bitbucket、Azure Active Directory（AAD）、Crowd等方式
- 提供了优美的3D视图方式下查看代码分析和测试结果报告

### 2. 安装

- 安装jdk1.8+

- 安装mysql5.x+，数据集必须配置为UTF8字符集，并且操作对象也是区分大小写的，并且仅支持InnoDB存储引擎，不支持MyISAM存储引擎。

  **创建数据库**

  ```sql
  -- 1. 创建SonarQube Server数据库
  CREATE DATABASE sonar CHARACTER SET utf8 COLLATE utf8_general_ci;
  -- 2. 创建SonarQube Server访问数据库的用户
  CREATE USER 'sonar' IDENTIFIED BY 'sonar';
  -- 3. 配置SonarQube Server访问数据库用户的权限
  GRANT ALL ON sonar.* TO 'sonar'@'%' IDENTIFIED BY 'sonar';
  GRANT ALL ON sonar.* TO 'sonar'@'localhost' IDENTIFIED BY 'sonar';
  flush privileges;
  ```

  ​

- 安装sonarqube

  1. 下载sonarqube
     https://www.sonarqube.org/downloads/

  2. 配置数据库
     ​        SonarQube目录下的conf/sonar.properties文件，配置它的数据库连接，启用和配置。

   ```shell
   sonar.jdbc.username=snoar
   sonar.jdbc.password=sonar
   sonar.jdbc.url=jdbc:mysql://localhost:3306/sonar?useUnicode=true&characterEncoding=utf8&rewriteBatchedStatements=true&useConfigs=maxPerformance
   ```

  3. 启动

   进入bin/macosx-universal-64，执行命令启动，访问

   http://localhost:9000（默认：`admin/admin`）

   ```sh
   nohup ./sonar.sh start 
   { console | start | stop | restart | status | dump }
   ```

  4. Sonar配置文件（sonar.properties）

  | 属性             | 说明        |
  | -------------- | --------- |
  | sonar.web.host | 访问域名／IP地址 |
  | sonar.web.port | 访问端口      |
  |                |           |

  5. 安装插件

     汉化插件

     ```
     下载sonar-l10n-zh-plugin-1.15.jar后，
     https://github.com/SonarQubeCommunity/sonar-l10n-zh 

     放到
     SonarQube/sonarqube-6.3.1/extensions/plugins 下，重启Sonar
     ```

     其他插件

     ```
     配置 -> 更新中心 -> Available
     ```


###3. Sonar Scanner

​	SonarQube实际上是一个web系统，展现静态代码扫描的结果（可以自定义），真正进行扫描的是 Sonar Scanner 工具。

1. 安装
   - 下载适合自己系统的工具
     https://docs.sonarqube.org/display/SCAN/Analyzing+with+SonarQube+Scanner

   - 修改对应的SonarQube Server，*<install_directory>*/conf/sonar-scanner.properties*

     ```properties
     #----- Default SonarQube server
     #sonar.host.url=http://localhost:9000
     ```

   - 将 *<install_directory>/bin* 加入到PATH中

2. 使用

   - 在要进行检查的Project的根目录创建文件：==*sonar-project.properties*==，内容如下：

   ```properties
   # must be unique in a given SonarQube instance
   sonar.projectKey=my:project
   # this is the name and version displayed in the SonarQube UI. Was mandatory prior to SonarQube 6.1.
   sonar.projectName=My project
   sonar.projectVersion=1.0
    
   # Path is relative to the sonar-project.properties file. Replace "\" by "/" on Windows.
   # This property is optional if sonar.modules is set. 
   sonar.sources=.
    
   # Encoding of the source code. Default is default system encoding
   sonar.sourceEncoding=UTF-8
   ```

   - 在根目录运行

   ```sh
   sonar-scanner
   ```




### 4. 与Jenkins联合使用 

在Jenkins中安装和使用SonarQube的先决条件

- 安装SonarQube插件 ，插件管理中添加。


-  配置Sonar

  ![Sonar-Jenkins](/Users/dante/Documents/Technique/且行且记/Sonar-Jenkins.png)

- 在pipeline中使用（项目中有sonar-project.properties）

  ```groovy
  node {
    stage('SCM') {
      git 'https://github.com/foo/bar.git'
    }
    stage('SonarQube analysis') {
      // requires SonarQube Scanner 2.8+
      def scannerHome = tool 'SonarQube Scanner 2.8';
      withSonarQubeEnv('My SonarQube Server') {
        sh "${scannerHome}/bin/sonar-scanner"
      }
    }
  }
  ```

- 获取Sonar扫描结果

  - 使用Sonar API https://sonarqube.com/web_api/

    ```shell
    !/bin/bash

    SONAR_KEY="sonar:microservice-spirit"
    STATUS=$(curl -qsSL "http://localhost:9000/api/qualitygates/project_status?projectKey=${SONAR_KEY}" |\
        jq .projectStatus.status | grep -o "\w\+")
    if [ "${STATUS}" = "ERROR" ]; then
        echo "代码质量扫描未通过，请检查 http://localhost:9000/overview?id=${SONAR_KEY} 的扫描报告并修复！"
        exit -1
    fi
    echo "代码质量扫描完成！"
    ```

  - 安装Jq  https://stedolan.github.io/jq/

    ```sh
    brew install https://github.com/stedolan/jq/releases/download/jq-1.5/jq-osx-amd64
    ```

    ​