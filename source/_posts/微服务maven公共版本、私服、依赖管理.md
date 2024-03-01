# 前言

关于微服务的公共依赖模块的抽取、统一版本管理、统一私服配置等，尝试了多种方案，整理优化了多次，分享一下。



### 期望达到的效果：

1、统一的三方库版本管理

2、统一的私服配置（上传、下载、下载插件）

3、springcloud   springalibaba  springboot 版本依赖管理统一配置



# 正文

单独建一个common的maven工程，内部生成一个core module。

核心就两块，一块是最外层的 common-parent（pom artifacId的名字），一块是内层的common



### 关系图

![image-20210423091353364](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423091353364.png)



## common工程

##### common的功能：

- 公共类、公共工具类
- 公共三方依赖
- 注意：common的packaging是jar



##### common-parent的功能：

- 版本的统一管理
- springboot   springcloud springalibaba 版本管理
- 私服的配置
- 注意：common-parent的packaging是pom





##### 外层的pom.xml

![image-20210423093343989](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423093343989.png)

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.8.RELEASE</version>
    </parent>

    <groupId>com.xiaoxi</groupId>
    <artifactId>common-parent</artifactId>
    <version>1.0-SNAPSHOT</version>
    <description>common-parent</description>
    <packaging>pom</packaging>

    <modules><module>core</module></modules>

    <!--  统一版本管理  -->
    <properties>
        <mybatis-plus.version>3.4.0</mybatis-plus.version>
        <dynamic-datasource.version>3.3.2</dynamic-datasource.version>
        <p6sy.version>3.8.7</p6sy.version>
        <mysql-connector.version>8.0.13</mysql-connector.version>
        <knife4j.version>2.0.7</knife4j.version>
        <lombok.version>1.18.12</lombok.version>
        <hutool.version>5.3.8</hutool.version>
        <java.version>1.8</java.version>
        <seata.version>1.4.0</seata.version>
        <nacos.version>2.2.1.RELEASE</nacos.version>


        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
        <spring-cloud.version>Hoxton.SR6</spring-cloud.version>
        <spring-cloud-alibaba.version>2.2.1.RELEASE</spring-cloud-alibaba.version>
    </properties>


    <!--  springcloud springAlibaba 依赖版本管理  -->
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <type>pom</type>
                <version>${spring-cloud.version}</version>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>${spring-cloud-alibaba.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>io.seata</groupId>
                <artifactId>seata-spring-boot-starter</artifactId>
                <version>${seata.version}</version>
            </dependency>
            <dependency>
                <groupId>mysql</groupId>
                <artifactId>mysql-connector-java</artifactId>
                <version>${mysql-connector.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <!--拉取仓库-->
    <repositories>
        <repository>
            <id>maven_xiaoxi</id>
            <url>http://192.168.148.237:8081/repository/maven-public/</url>
            <releases>
                <enabled>true</enabled>
                <updatePolicy>always</updatePolicy>
            </releases>
            <snapshots>
                <updatePolicy>always</updatePolicy>
                <enabled>true</enabled>
            </snapshots>
        </repository>
    </repositories>
    <!--插件拉取仓库-->
    <pluginRepositories>
        <pluginRepository>
            <id>maven_xiaoxi</id>
            <url>http://192.168.148.237:8081/repository/maven-public/</url>
            <releases>
                <updatePolicy>always</updatePolicy>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <updatePolicy>always</updatePolicy>
                <enabled>true</enabled>
            </snapshots>

        </pluginRepository>
    </pluginRepositories>
    <!--发布仓库-->
    <distributionManagement>
        <repository>
            <id>maven_xiaoxi</id>
            <url>http://192.168.148.237:8081/repository/maven-releases/</url>
        </repository>
        <snapshotRepository>
            <id>maven_xiaoxi</id>
            <url>http://192.168.148.237:8081/repository/maven-snapshots/</url>
        </snapshotRepository>
    </distributionManagement>

</project>
```







##### 内层core的pom.xml

![image-20210423093126870](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423093126870.png)



```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>common-parent</artifactId>
        <groupId>com.xiaoxi</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>

    <modelVersion>4.0.0</modelVersion>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>
    <artifactId>common</artifactId>


    <!--  公共依赖  -->
    <dependencies>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>${mysql-connector.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <version>2.2.8.RELEASE</version>
        </dependency>

        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>${mybatis-plus.version}</version>
        </dependency>

        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
            <version>${dynamic-datasource.version}</version>
        </dependency>

        <dependency>
            <groupId>p6spy</groupId>
            <artifactId>p6spy</artifactId>
            <version>${p6sy.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
            <version>2.2.3.RELEASE</version>
        </dependency>

        <dependency>
            <groupId>com.github.xiaoymin</groupId>
            <artifactId>knife4j-spring-boot-starter</artifactId>
            <version>${knife4j.version}</version>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
        </dependency>

        <dependency>
            <groupId>cn.hutool</groupId>
            <artifactId>hutool-all</artifactId>
            <version>${hutool.version}</version>
        </dependency>

        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
            <exclusions>
                <exclusion>
                    <artifactId>httpclient</artifactId>
                    <groupId>org.apache.httpcomponents</groupId>
                </exclusion>
            </exclusions>
            <version>${nacos.version}</version>
        </dependency>

        <!--    seata    -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-seata</artifactId>
            <version>2.2.0.RELEASE</version>
            <exclusions>
                <exclusion>
                    <groupId>io.seata</groupId>
                    <artifactId>seata-spring-boot-starter</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <dependency>
            <groupId>io.seata</groupId>
            <artifactId>seata-spring-boot-starter</artifactId>
            <version>${seata.version}</version>
        </dependency>

        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
        </dependency>

        <dependency>
            <groupId>com.auth0</groupId>
            <artifactId>java-jwt</artifactId>
            <version>3.15.0</version>
        </dependency>

        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
        </dependency>

    </dependencies>


</project>
```



### 发布

每次发布，最好先clean，然后在最外层maven deploy即可。（因为他们是父子级关系）







## 微服务

![image-20210418160500541](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210418160500541.png)



## xxxx-interface

![image-20210418160710114](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210418160710114.png)

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <parent>
        <groupId>com.xiaoxi</groupId>
        <artifactId>common-parent</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <groupId>com.example</groupId>
    <version>1.0-SNAPSHOT</version>

    <modelVersion>4.0.0</modelVersion>

    <artifactId>test1-interface</artifactId>

    <dependencies>
        <dependency>
            <groupId>com.xiaoxi</groupId>
            <artifactId>common</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    </dependencies>


    <repositories>
        <repository>
            <id>maven_xiaoxi</id>
            <url>http://192.168.148.237:8081/repository/maven-public/</url>
            <releases>
                <enabled>true</enabled>
                <updatePolicy>always</updatePolicy>
            </releases>
            <snapshots>
                <updatePolicy>always</updatePolicy>
                <enabled>true</enabled>
            </snapshots>
        </repository>
    </repositories>

</project>
```



可以看到，这里interface，是继承了common-parent，因为它是在私服上面的，所以无论如何你也需要配置一个私服的拉取仓库。但是好处是，你继承了common-parent，等于里面的 下载、插件下载、发布的配置都有了，就不必再配置发布了。





## xxxx-service

![image-20210418160853622](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210418160853622.png)

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
        <parent>
            <groupId>com.xiaoxi</groupId>
            <artifactId>common-parent</artifactId>
            <version>1.0-SNAPSHOT</version>
        </parent>

    <groupId>com.example</groupId>
    <artifactId>test1-service</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>test1-service</name>
    <description>Demo project for Spring Boot</description>


    <dependencies>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>com.example</groupId>
            <artifactId>test1-interface</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    </dependencies>

    <!--  通过maven的profile来 动态修改spring的profile 达到多套配置切换使用的效果  -->
    <profiles>
        <profile>
            <!-- 本地环境 -->
            <id>local</id>
            <properties>
                <profiles.active>local</profiles.active>
                <skipDocker>true</skipDocker>
                <dockerHost></dockerHost>
            </properties>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
        </profile>
        <profile>
            <!-- 开发环境 -->
            <id>dev</id>
            <properties>
                <profiles.active>dev</profiles.active>
                <skipDocker>true</skipDocker>
                <dockerHost>http://192.168.148.236:2375</dockerHost>
            </properties>
        </profile>
        <profile>
            <!-- 测试环境 -->
            <id>test</id>
            <properties>
                <profiles.active>test</profiles.active>
                <skipDocker>false</skipDocker>
                <dockerHost>http://192.168.148.235:2375</dockerHost>
            </properties>
        </profile>
        <profile>
            <!-- 生产环境 -->
            <id>prod</id>
            <properties>
                <profiles.active>prod</profiles.active>
                <skipDocker>false</skipDocker>
                <dockerHost>http://xxxx:2375</dockerHost>
            </properties>
        </profile>
    </profiles>

    <!--  maven插件发布到docker中  -->
    <build>
        <!-- 引用我们的项目名字 -->
        <finalName>${project.artifactId}</finalName>

        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>

        </plugins>

    </build>

    <repositories>
        <repository>
            <id>maven_xiaoxi</id>
            <url>http://192.168.148.237:8081/repository/maven-public/</url>
            <releases>
                <enabled>true</enabled>
                <updatePolicy>always</updatePolicy>
            </releases>
            <snapshots>
                <updatePolicy>always</updatePolicy>
                <enabled>true</enabled>
            </snapshots>
        </repository>
    </repositories>

</project>
```



service，现在就清爽很多了。只需要继承common-parent，加上私服拉取，就可以节省很多配置了。依赖的话，只需要微服务模块间的 和 自己的 interface的依赖即可。





## 依赖关系为：
xxxx-service -> xxxx-interface -> common



## 继承关系为

xxx-interface ->  common-parent

xxx-service ->  common-parent



# 关于service想要自定义版本的实现

在整理优化的过程中发现一个问题，就是：

我们公共依赖里面配置的seata版本是1.4.0，而最终在service里面的版本是1.1.0。

这是为什么呢？这就跟maven的依赖加载顺序有关系了。

[有一篇文章讲得不错，分享一下](https://blog.csdn.net/lizz861109/article/details/111594969)



#### 这里也简述一下：



我们大致把依赖分为这几个级别：

- 本级

<dependencies> 自身的工程的直接依赖

- **上级**

<parent> 自身工程的parent继承

- **下级**

依赖的库的依赖 自身工程依赖的三方库  的 依赖

- **版本管理**

<dependencyManagement>  自身工程的版本管理



### 优先级顺序为：

**本级  > 本级版本管理 > 上级  > 上级版本管理 > 下级  （无下级版本管理）**



而对照我们的公共依赖库的话：

我们parent继承的common-parent的版本优先级是高于 依赖的第三方common的依赖的版本的

也就是说，common里面是1.4.0，而parent里面是1.1.0（springalibaba），此时最终就会以1.1.0为最优先。



下面是图

![image-20210423095204507](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423095204507.png)



#### 那么这种情况怎么办呢？

我们知道maven的依赖如果是有多个相同的，那么后面的会覆盖前面的配置。



在我们common-parent的dependencyManagement里面已经定义了 springalibaba，那么我们就可以在下面重新指定一下 seata的版本为1.4.0，这样对于common-parent而言最终其实seata版本就是1.4.0了。

![image-20210423095706068](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423095706068.png)



**对照加载顺序就是**：

- 原来：

本级(无)  > 本级版本管理 (无) > 上级(无)  > 上级版本管理 （common-parent  1.1.0）> 下级 (common 1.4.0)  （无下级版本管理）

- 现在：

本级(无)  > 本级版本管理 (无) > 上级(无)  > 上级版本管理 （common-parent  1.4.0）> 下级 (common 1.4.0)  







# 关于作者

作者是一个热爱学习、开源、分享，传播正能量，喜欢打篮球、头发还很多的程序员-。-

热烈欢迎大家关注、点赞、评论交流！

简书：https://www.jianshu.com/u/d234d1569eed

github：https://github.com/fly7632785

CSDN：https://blog.csdn.net/fly7632785

掘金：https://juejin.im/user/5efd8d205188252e58582dc7/posts

