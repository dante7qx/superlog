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

   ```
   mvn package
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

