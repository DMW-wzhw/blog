# Maven

<br>


## 基础

mvn`的项目标准**目录结构**

* src/main/java		核心代码
* src/main/resources	配置文件部分
* src/test/java		测试代码部分
* src/test/resources	测试配置文件
* src/main/webapp	页面资源，js，css，WEB-INF 等等 
* target 	

<br>

相关的 `mvn` 命令

* mvn clean
* mvn compile
* mvn test
* mvn package
	* 会根据 pom.xml 中配置的 <packaging>war</packaging> 指定的，打成 war 包或者 jar 包
* mvn install
	* 除了打包后，还会将 war 放到本地仓库

<br>

**生命周期**

* 清理生命周期 `clean`   
* 默认声明周期 `compile -> test -> package -> install -> deploy`  在默认生命周期中，**每个命令执行的时候，都会先执行它前面的命令**

<br>

![mvn 的概念模型](https://github.com/DMW-wzhw/blog/resources/images/mvn概念模型.png)

1. 项目自身信息
2. 项目依赖的 jar 包信息
3. 项目运行环境信息，例如 jdk、tomcat

<br>

在 IDEA 中配置没网情况下也能创建 maven 项目的方法：在 `IDEA => Maven => Runner 的 VM Options 配置 ·-DarchetypeCatalog=internal` 可以在不联网的情况下可以正常创建工程

<br>

在 Maven 工程中，添加 `mvn tomcat:run` 命令

![tomcat:run](https://github.com/DMW-wzhw/blog/resources/images/tomcatrun.png)

<br>

在 maven 的 web 项目中，配置 servlet 和 jsp 的依赖的时候，需要添加 <scope>provided</scope>，因为 tomcat 中已经有了这两个包，如果 mvn 打包将这两个包放到 lib 目录下的话，**就会跟 tomcat 本身的冲突**，所以需要添加 scope 作用域。

![scope](https://github.com/DMW-wzhw/blog/resources/images/maven_scope.png)

<br>

Maven 本身集成了 tomcat，不过 tomcat 是 6 版本，不支持 jsp，那么怎么修改 tomcat 的版本呢？
需要在 pom.xml 中的 build => pluginManagement => pluges 中配置

```
<build>
  <finalName>mvn_web</finalName>
  <pluginManagement><!-- lock down plugins versions to avoid using Maven defaults (may be moved to parent pom) -->
    <plugins>
      <plugin>
        <groupId>org.apache.tomcat.maven</groupId>
        <artifactId>tomcat7-maven-plugin</artifactId>
        <version>2.2</version>
        <configuration>
          <port>8888</port>
        </configuration>
      </plugin>
      ...
    <plugins>
  <pluginManagement>
<build>
```
在然使用 `mvn tomcat7:run` 执行

<br>

配置编译的 JDK 版本

```
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <configuration>
    <target>1.8</target>
    <source>1.8</source>
    <encoding>UTF-8</encoding>
  </configuration>
</plugin>
```


