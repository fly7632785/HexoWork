## 1、idea设置搜索docker，输入   http://192.168.31.168:2375  

![image](/uploads/4a700fb355aed41e3737028c9caabd8d/image.png)

## 2、全局搜索docker，选择docker

![images11](/uploads/1cd8b5e731adf037d19ec42c07a2418e/images11.png)
打开docker
![image1](/uploads/864e600f67c14149c7d228cf8e1385f3/image1.png)

## 3、创建docker连接，地址也是 http://192.168.31.168:2375  

![image2](/uploads/84374e8826570b22fa016f95a50ea27c/image2.png)
![image3](/uploads/cbaf59c8a2a9e54de1fbf1d5bd03ff2b/image3.png)

## 4、代码添加<build> maven插件

```
<!--  maven插件发布到docker中  -->
    <build>
        <!-- 引用我们的项目名字 -->
        <finalName>${project.artifactId}</finalName>

        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>

            <!--使用docker-maven-plugin插件-->
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>1.0.0</version>
                <!--将插件绑定在某个phase执行-->
                <executions>
                    <execution>
                        <id>build-image</id>
                        <!--用户只需执行mvn package ，就会自动执行mvn docker:build-->
                        <phase>package</phase>
                        <goals>
                            <goal>build</goal>
                        </goals>
                    </execution>
                </executions>

                <configuration>
                    <skipDocker>${skipDocker}</skipDocker>
                    <!--指定生成的镜像名,这里是我们的项目名-->
                    <imageName>${project.artifactId}</imageName>

                    <!--指定标签 这里指定的是镜像的版本，我们默认版本是latest-->
                    <imageTags>
                        <imageTag>latest</imageTag>
                    </imageTags>

                    <!--指定基础镜像jdk1.8-->
                    <!--                    <baseImage>java</baseImage>-->
                    <baseImage>adoptopenjdk:8-jdk-openj9</baseImage>
                    <!--
                    镜像制作人本人信息
                    <maintainer>bruceliu@email.com</maintainer>
                    -->
                    <!--切换到ROOT目录-->
                    <workdir>/ROOT</workdir>

                    <!--查看我们的java版本-->
                    <cmd>["java", "-version"]</cmd>

                    <!--${project.build.finalName}.jar是打包后生成的jar包的名字-->
                    <entryPoint>["java", "-jar", "/${project.build.finalName}.jar"]</entryPoint>

                    <!--指定远程 docker api地址-->
                    <dockerHost>http://192.168.31.168:2375</dockerHost>

                    <!-- 这里是复制 jar 包到 docker 容器指定目录配置 -->
                    <resources>
                        <resource>
                            <targetPath>/</targetPath>
                            <!--jar 包所在的路径  此处配置的 即对应 target 目录-->
                            <directory>${project.build.directory}</directory>
                            <!--用于指定需要复制的文件 需要包含的 jar包 ，这里对应的是 Dockerfile中添加的文件名　-->
                            <include>${project.build.finalName}.jar</include>
                        </resource>
                    </resources>
                </configuration>
            </plugin>
        </plugins>

    </build>
```

## 5、package创建带参数的启动器

![image4](/uploads/8017143b92ae18d1968d5b668dcc1c19/image4.png)
如果现在分了多套环境配置，本地、测试、生产，此时docker用于test环境变量
![image9](/uploads/51e23bd12144d8f1a14226d4756bb762/image9.png)
添加 -Ptest 参数
![image](http://192.168.31.168:8100/dcmp/doc/-/wikis/uploads/4a700fb355aed41e3737028c9caabd8d/image.png)
然后启动

## 6、打包好image，创建container

![image5](/uploads/620e3cea6e4ef76caabc4229076ce20b/image5.png)

## 7、设置好container参数

`--net=host -v /etc/localtime:/etc/localtime`
![image66](/uploads/2e533a033ed22a85280931560fb8fe9a/image66.png)


## 8、运行即可