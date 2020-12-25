### MAVEN 使用说明

#### 常用命令

1. 创建maven普通java项目

   ```shell
   mvn archetype:create -DgroupId=packageName -DartifactId=projectName
   ```

2. 创建maven的web项目

   ```shell
   mvn archetype:create -DgroupId=packageName -DartifactId=projectName -DarchetypeArtifactId=maven-archetype-webapp
   ```

3. 反向生成maven项目骨架

   ```shell
   mvn archtype:generate

   简写
   mvn archetype:generate -DgroupId=otowa.user.dao -DartifactId=user-dao -Dversion=0.01-SNAPSHOT
   ```

4. 编译源代码

   ```shell
   mvn compile
   ```

5. 编译测试代码

   ```shell
   mvn test-compile
   ```

6. 运行测试

   ```
   mvn test
   ```

7. 产生site

   ```
   mvn site
   ```

8. 打包

   ```shell
   mvn package
   mvn package -Dmaven.repo.local=/Users/dante/Desktop/aa
   ```

9. 在本地安装

   ```shell
   mvn install
   ```

10. 强制更新

 ```sh
 mvn clean install-U
 ```

11. 生成eclipse、idea项目

    ```sh
    1. 生成eclipse项目
    	mvn eclipse:eclipse 
    2. 生成idea项目
    	mvn idea:idea
    ```

12. 只测试而不编译，也不测试编译

    ```shell
    mvn test -skipping compile -skipping test-compile
     ( -skipping 的灵活运用，当然也可以用于其他组合命令)
    ```

13. 将jar上传到远程仓库

    ```shell
    mvn deploy
    ```

14. 多环境打包，**-P 使用指定的Profile配置**

    ```shell
    mvn package -P dev
    ## 参考：http://blog.csdn.net/hjiacheng/article/details/57413933
    ```

    项目开发需要有多个环境，在pom.xml中的配置如下：

    ```xml
    <profiles>
      <profile>
        <id>dev</id>
        <properties>
          <env>ram</env>
        </properties>
        <activation>
          <activeByDefault>true</activeByDefault>
        </activation>
      </profile>
      <profile>
        <id>prod</id>
        <properties>
          <env>jdbc</env>
        </properties>
      </profile>
    </profiles>
    <build>
      <resources>
        <resource>
          <directory>src/main/resources</directory>
          <excludes>
            <exclude>ram/*</exclude>
            <exclude>jdbc/*</exclude>
            <exclude>*.properties</exclude>
          </excludes>
        </resource>
        <resource>
          <directory>src/main/resources/${env}</directory>
          <targetPath>./</targetPath>
        </resource>
      </resources>
      <plugins>
        <plugin>  
          <artifactId>maven-compiler-plugin</artifactId>  
          <extensions>true</extensions>   
          <configuration>  
            <source>1.8</source>  
            <target>1.8</target>  
          </configuration>  
        </plugin>
      </plugins>
    </build> 
    ```



15. man -pl -am -amd

- https://www.cnblogs.com/sandyflower/p/11600108.html

```properties
1. 在dailylog-parent目录运行`mvn clean install -pl org.lxp:dailylog-web -am`，结果

dailylog-common成功安装到本地库
dailylog-parent成功安装到本地库
dailylog-web成功安装到本地库
该命令等价于`mvn clean install -pl ../dailylog-web -am`

2. 在dailylog-parent目录运行`mvn clean install -pl ../dailylog-common -am`，结果

dailylog-common成功安装到本地库
dailylog-parent成功安装到本地库
3. 在dailylog-parent目录运行`mvn clean install -pl ../dailylog-common -amd`，结果

dailylog-common成功安装到本地库
dailylog-web成功安装到本地库
由于dailylog-parent并不依赖dailylog-common模块，故没有被安装

4. 在dailylog-parent目录运行`mvn clean install -pl ../dailylog-common,../dailylog-parent -amd`，结果

dailylog-common成功安装到本地库
dailylog-parent成功安装到本地库
dailylog-web成功安装到本地库
5. 在dailylog-parent目录运行`mvn clean install -N`，结果

dailylog-parent成功安装到本地库
-N表示不递归，那么dailylog-parent管理的子模块不会被同时安装

6. 在dailylog-parent目录运行`mvn clean install -pl ../dailylog-parent -N`，结果

dailylog-parent成功安装到本地库
7. 在dailylog-parent目录运行`mvn clean install -rf ../dailylog-common`，结果

dailylog-common成功安装到本地库
dailylog-web成功安装到本地库
```



#### 常用插件

##### ***maven-compiler-plugin*** 

```xml
<plugin>  
    <artifactId>maven-compiler-plugin</artifactId>  
    <extensions>true</extensions>   
    <configuration>  
        <source>1.8</source>  
        <target>1.8</target>  
    </configuration>  
</plugin>
```

##### **maven-dependency-plugin **

把依赖的jar包拷到指定目录下

```xml
<plugin>  
    <groupId>org.apache.maven.plugins</groupId>  
    <artifactId>maven-dependency-plugin</artifactId>  
    <executions>  
        <execution>  
            <id>copy-dependencies</id>  
            <phase>process-resources</phase>  
            <goals>  
                <goal>copy-dependencies</goal>  
            </goals>  
            <configuration>  
                <excludeScope>provided</excludeScope>  
                <excludeArtifactIds>  
                    module1,module2  
                </excludeArtifactIds>  
                <outputDirectory>${project.build.directory}/lib</outputDirectory>  
            </configuration>  
        </execution>  
        <execution>  
            <id>copy-modules</id>  
            <phase>process-resources</phase>  
            <goals>  
                <goal>copy-dependencies</goal>  
            </goals>  
            <configuration>  
                <includeArtifactIds>  
                    module1,module2  
                </includeArtifactIds>  
                <outputDirectory>${project.build.directory}/lib/modules</outputDirectory>  
            </configuration>  
        </execution>  
    </executions>  
</plugin>  
```

##### maven-resources-plugin

***把依赖的资源拷到指定目录下***

```xml
<plugin>  
    <groupId>org.apache.maven.plugins</groupId>  
    <artifactId>maven-resources-plugin</artifactId>  
    <version>3.0.2</version>  
    <executions>  
        <execution>  
            <id>copy-resources</id>  
            <!-- here the phase you need -->  
            <phase>validate</phase>  
            <goals>  
                <goal>copy-resources</goal>  
            </goals>  
            <configuration>  
                <outputDirectory>${basedir}/target/test-classes</outputDirectory>  
                <resources>  
                    <resource>  
                        <directory>${basedir}/src/main/webapp/WEB-INF/config</directory>  
                        <filtering>true</filtering>  
                    </resource>  
                </resources>  
            </configuration>  
        </execution>  
    </executions>  
</plugin> 
```

##### **maven-jar-plugin**

打jar包配置

```xml
<plugin>  
    <groupId>org.apache.maven.plugins</groupId>  
    <artifactId>maven-jar-plugin</artifactId>  
    <configuration>  
        <excludes>  
            <exclude>**/config/*</exclude>  
        </excludes>
      	<archive>  
            <manifest>  
                <addClasspath>true</addClasspath>  
                <classpathPrefix>lib/</classpathPrefix>  
                <mainClass>com.xxx.uploadFile</mainClass>
                <!-- 打包时 MANIFEST.MF文件不记录的时间戳版本 -->
                <useUniqueVersions>false</useUniqueVersions>
            </manifest>  
      	</archive>  
    </configuration>  
</plugin>
```

##### maven-assembly-plugin

**包含依赖** https://www.cnblogs.com/dzblog/p/6913809.html

```xml
<plugin>
    <artifactId>maven-assembly-plugin</artifactId>
    <configuration>
        <appendAssemblyId>false</appendAssemblyId>
        <descriptorRefs>
          	<descriptorRef>jar-with-dependencies</descriptorRef>
        </descriptorRefs>
        <archive>
            <manifest>
                <!-- 此处指定main方法入口的class -->
                <mainClass>com.xxx.uploadFile</mainClass>
            </manifest>
        </archive>
    </configuration>
    <executions>
        <execution>
            <id>make-assembly</id>
            <phase>package</phase>
            <goals>
              	<goal>assembly</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

##### findbugs-maven-plugin

`mvn compile site`

```xml
<reporting>
    <plugins>
        <plugin>
            <groupId>org.codehaus.mojo</groupId>
            <artifactId>findbugs-maven-plugin</artifactId>
            <version>3.0.5</version>
            <configuration>
                <effort>Max</effort>
                <threshold>Low</threshold>
            </configuration>
        </plugin>
    </plugins>
</reporting>
```

##### exec-maven-plugin

- https://blog.csdn.net/bluishglc/article/details/7622286#

对于那些打包成jar包形式的本地java应用来说，通过java命令启动将会是一件极为繁琐的事情，原因很简单，太多的依赖让参数-classpath变得异常的恐怖。

1. 通过工具将工程及其所有依赖的jar包打包成一个独立的jar包（在maven里有两个插件assemly和shade是用来完成这种工作的）。
2. 编写一个run.bat文件，文件包含一个启动应用的java命令，很显然，这个命令的classpath必须包含全部依赖的jar包。

```xml
<plugin>  
    <groupId>org.codehaus.mojo</groupId>  
    <artifactId>exec-maven-plugin</artifactId>  
    <version>1.2.1</version>  
    <executions>  
        <execution>  
            <goals>  
                <goal>java</goal>  
            </goals>  
        </execution>  
    </executions>  
    <configuration>  
        <mainClass>com.yourcompany.app.Main</mainClass>  
    </configuration>  
</plugin>  
```

​	java -DsystemProperty1=value1 -DsystemProperty2=value2 -XX:MaxPermSize=256m -classpath .... com.yourcompany.app.Main arg1 arg2

```xml
<plugin>  
    <groupId>org.codehaus.mojo</groupId>  
    <artifactId>exec-maven-plugin</artifactId>  
    <version>1.2.1</version>  
    <configuration>  
        <executable>java</executable> <!-- executable指的是要执行什么样的命令 -->  
        <arguments>  
            <argument>-DsystemProperty1=value1</argument> <!-- 这是一个系统属性参数 -->  
            <argument>-DsystemProperty2=value2</argument> <!-- 这是一个系统属性参数 -->  
            <argument>-XX:MaxPermSize=256m</argument> <!-- 这是一个JVM参数 -->  
            <!--automatically creates the classpath using all project dependencies,   
                also adding the project build directory -->  
            <argument>-classpath</argument> <!-- 这是classpath属性，其值就是下面的<classpath/> -->  
            <classpath/> <!-- 这是exec插件最有价值的地方，关于工程的classpath并不需要手动指定，它将由exec自动计算得出 -->  
            <argument>com.yourcompany.app.Main</argument> <!-- 程序入口，主类名称 -->  
            <argument>arg1</argument> <!-- 程序的第一个命令行参数 -->  
            <argument>arg2</argument> <!-- 程序的第二个命令行参数 -->  
        </arguments>  
    </configuration>  
</plugin>  
<!--
	执行 mvn exec:exec
-->
```



#### 生产组合

下面的命令经常在生产环境中使用

- 导出项目依赖jar包

  ```shell
  1. 将jar包导出到targed/dependency
  mvn dependency:copy-dependencies 

  2. 指定导出目录
  mvn dependency:copy-dependencies -DoutputDirectory=/Users/dante/Desktop/lib

  3. 只导出编译范围的jar
  mvn dependency:copy-dependencies -DincludeScope=compile
  ```

- 忽略测试

  ```shell
  1. -DskipTests，不执行测试用例，但编译测试用例类生成相应的class文件到target/test-classes下
  	mvn clean package -DskipTests=true
  2. -Dmaven.test.skip=true，不执行测试用例，也不编译测试用例类
  	mvn clean package -Dmaven.test.skip=true
  ```

- 将第三方jar变为maven jar（本地）

  ```shell
  mvn install:install-file -Dfile=<第三方jar> -DgroupId=org.dante.spring -DartifactId=micro-sms -Dversion=1.2.2 -Dpackaging=jar -Dgenerate=true
  ```

- 删除.lastUpdated

  find ./ -name "*.lastUpdated" | xargs rm -rf

