



[TOC]





# 前言

中小型公司的自动化流水线相关，搭建tapd+jenkins+gitlab+sonarqube+docker+jmeter+ansible这一套流水线的东西。期望的样子就是，从代码提交到测试、扫描、构建、部署等一系列操作都是自动化的，



### 相关介绍

| 框架      | 描述                   |
| --------- | ---------------------- |
| tapd      | 一个项目需求管理平台   |
| jenkins   | 自动化构建调度框架     |
| gitlab    | 代码管理               |
| sonarqube | 代码扫描bug smell框架  |
| jmeter    | 接口测试、压力测试框架 |
| ansible   | 类k8s的轻量级部署框架  |
| docker    | 容器化框架             |





# 一、准备

### 环境

| 需要的环境               | 地址       |
| ------------------------ | ---------- |
| virtualbox+虚拟机centos7 | 10.10.0.44 |

先安装一个linux虚拟机吧，并且设置为静态的ip地址`10.10.0.44`。具体步骤可以参考另一篇

[virtualbox安装centos7]()



# 二、安装docker

安装源工具

```
sudo yum install -y yum-utils
```

设置阿里云源

```
sudo yum-config-manager \
    --add-repo \
    http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

安装

```
sudo yum -y install docker-ce docker-ce-cli containerd.io
启动
sudo systemctl start docker
```

修改daemon配置文件/etc/docker/daemon.json来使用阿里云加速器（地址可以自行进入阿里云官网查找[阿里云镜像加速](https://help.aliyun.com/document_detail/60750.html)）

```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://vc5a3maa.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo systemctl enable docker
```

#### 开放docker 2375端口

```
vim /usr/lib/systemd/system/docker.service
```

添加

```
-H tcp://0.0.0.0:2375
```

![image-20210401132209239](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401132209239.png)

```
systemctl daemon-reload
systemctl restart docker
```







# 三、docker安装gitlab

创建映射目录

```
mkdir -p -m 777 /home/gitlab/config   创建config目录
mkdir -p -m 777 /home/gitlab/logs    创建logs目录
mkdir -p -m 777 /home/gitlab/data    创建data目录
```

docker命令

```
docker run --detach \
    --hostname 10.10.0.44 \
    --publish 7001:443 --publish 7002:80 --publish 7003:22 \
    --name gitlab --restart always \
    --volume /home/gitlab/config:/etc/gitlab \
    --volume /home/gitlab/logs:/var/log/gitlab \
    --volume /home/gitlab/data:/var/opt/gitlab gitlab/gitlab-ce
```

端口说明

| 端口     | 说明     |
| -------- | -------- |
| 7002:80  | 网页http |
| 7001:443 | https    |
| 7003:22  | ssh      |

运行之后等一会儿，就可以访问```10.10.0.44:7002```访问gitlab的网站了，然后就要进行一些密码 url设置等

![image-20210330172727322](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330172727322.png)

默认账号是root

### 修改gitlab.rb配置文件

路径是`/home/gitlab/config/gitlab.rb`

```
external_url 'http://10.10.0.44'
gitlab_rails['gitlab_ssh_host'] = '10.10.0.44'
gitlab_rails['gitlab_shell_ssh_port'] = 7003
```

### 进入容器重启配置

```
docker exec -it gitlab /bin/bash  进去gitlab容器的命令
gitlab-ctl reconfigure  重置gitlab客户端的命令
```

然后要等待一会儿直到页面能够重新显示为止

### 修改http的clone地址加上端口

修改gitlab.yml文件

```
进入容器内部
docker exec -it gitlab /bin/bash
修改文件
cd /opt/gitlab/embedded/service/gitlab-rails/config
vim gitlab.yml
```

```
修改gitlab
             host：10.10.0.44
             port：7002
```

然后在容器内执行`gitlab-ctl restart` （注意 这里如果docker restart gitlab了，设置会被重新覆盖，也就丢失了，因为restart会重新执行`gitlab-ctl reconfigure`，目前没有什么好的方式，只有尽量少启动gitlab）

![image-20210330174815638](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330174815638.png)

这下地址都是带端口的了

其他上传ssh公钥，就自行操作了

#### 新建一个springboot的test项目

然后自己测试一下clone push等

![image-20210401133121425](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401133121425.png)





# 四、docker安装jenkins

创建映射目录

```
mkdir -p -m 777 /home/jenkins
```

```
docker run -d --name jenkins  --restart always -p 8040:8080 -p 50000:50000 -v /home/jenkins:/var/jenkins_home jenkins/jenkins:2.259
```

注意：这里使用的版本是2.259，如果你要使用jmeter且要使用`Performance`插件的话，不要用高版本的！！

这里我已经踩过坑了，高版本的会有一个问题：一旦在任务配置里面添加了Performance相关的设置，再次打开配置就会出现无法应用、保存，而且明显格式是混乱的，且不能删除掉。所以，在没有解决掉这个bug之前暂且不要使用。这里多次尝试，2.259为较高版本的可以用版本。此时的LATEST是2.275。

![image-20210330180225203](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330180225203.png)

我们已经映射了路径

```
cd /home/jenkins/secrets/
cat initialAdminPassword
```

获取到密码，然后继续，安装推荐插件，等待完成

设置用户密码即可

![image-20210330181158588](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330181158588.png)







# 五、docker安装nexus

创建映射目录

```
mkdir -p -m 777 /home/nexus
```

```
docker run -d --name nexus --restart always  --net=host --privileged -v /home/nexus:/nexus-data sonatype/nexus3
```

默认端口为8081

10.10.0.44:8081即可访问

![image-20210330182018364](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330182018364.png)

因为已经映射了路径

```
cat /home/nexus/admin.password
```

然后就输入密码，重新设置密码即可



# 六、docker安装sonarqube

创建映射目录

```
mkdir -p -m 777 /home/sonarqube/conf
mkdir -p -m 777 /home/sonarqube/data
mkdir -p -m 777 /home/sonarqube/extensions
mkdir -p -m 777 /home/sonarqube/bundled-plugins
```

注意这里我们要使用7.6一下的版本，不要使用最新版本，不然和jenkins不兼容

### 建立数据库mysql

创建映射目录

```
mkdir -p -m 777 /home/mysql/data
mkdir -p -m 777 /home/mysql/log
```

新建一个my.cnf配置文件，方便以后修改

```
touch my.cnf
```

运行mysql

```
docker run --name mysql  --restart always -p 3306:3306 -v /home/mysql/data:/var/lib/mysql  -v /home/mysql/my.cnf:/etc/mysql/my.cnf -v  /home/mysql/log:/var/log/mysql -e MYSQL_ROOT_PASSWORD=xiaoxi -d mysql:5.7
```

然后连接数据库，新建一个sonar的数据库



### 创建sonarqube

```
docker run -d --name sonarqube \
    --restart always \
    -p 9000:9000 \
    -v /home/sonarqube/conf:/opt/sonarqube/conf \
    -e SONARQUBE_JDBC_URL="jdbc:mysql://10.10.0.44:3306/sonar?useUnicode=true&characterEncoding=utf8&rewriteBatchedStatements=true&useConfigs=maxPerformance&useSSL=false"  \
    -e SONARQUBE_JDBC_USERNAME=root \
    -e SONARQUBE_JDBC_PASSWORD=xiaoxi \
    sonarqube:7.4-community
```

**注意里面的JDBC相关的url uesrname password要改为跟数据库一致的**

![image-20210330184552052](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330184552052.png)



添加新项目

![image-20210401103633039](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401103633039.png)

![image-20210401103710271](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401103710271.png)

保存好token后面会用到

![image-20210401103736859](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401103736859.png)

可以在maven项目中，使用一下命令即可实现代码的扫描

![image-20210401104103844](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401104103844.png)

这个project key后面也会用到

#### 另外汉化

如果需要汉化可以这样，登录之后下载汉化插件

如果你是8.7的可以直接market搜索下载，我的是7.4的只有去下载第三方的

![image-20210401145345298](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401145345298.png)

[github汉化插件下载](https://github.com/xuhuisheng/sonar-l10n-zh)

![image-20210401145734329](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401145734329.png)

下载对应版本的，7.4=1.24

下载好了，之后上传到目标服务器中利用scp命令

```
scp C:\\Users\\Administrator\\Downloads\\sonar-l10n-zh-plugin-1.24.jar root@10.10.0.44:/home/sonarqube
```

scp 本地文件  roo@xxxx:/文件目录

服务器中拷贝到容器里面去（当然你也可以把插件路径/opt/sonarqube/extensions/plugins/映射出来）

```
docker cp /home/sonarqube/sonar-l10n-zh-plugin-1.24.jar sonarqube:/opt/sonarqube/extensions/plugins/
```

```
docker restart sonarqube 
```

然后就可以了

![image-20210401150851190](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401150851190.png)







# 八、nexus添加仓库

nexus作为一个制品库的私服，也可以作为maven依赖的一个私服。

## maven私服仓库

#### 增加aliyun代理仓库

![image-20210401093429989](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401093429989.png)

![image-20210401094943725](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401094943725.png)

url:http://maven.aliyun.com/nexus/content/groups/public/

修改maven-public的一个顺序位置

![image-20210401095111246](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401095111246.png)

有了阿里云代理，从maven-public拉取依赖就可以走阿里云先拉，这样速度就会很快

![image-20210401100313472](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401100313472.png)

另外，maven-release默认是不可以多次deploy的，为了方便，这里也可以修改配置为Allow redeploy

#### maven配置相关

修改maven的setting.xml文件，如果你使用Idea那么先看一下自己的使用路径及其配置

![image-20210401095641744](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401095641744.png)

我这里是自己下载的maven，也自定义了配置文件和本地仓库的路径，修改的话就要修改对应路径的settings.xml文件（默认情况是idea的bundled的maven，settings.xml在C盘下面）

```
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <localRepository/>
  <interactiveMode/>
  <offline/>
  <servers>
    <server>
      <!--根据id来取用账号密码，这里的账号密码就是可以登录nexus的-->
      <id>maven_xiaoxi</id>
      <username>admin</username>
      <password>xiaoxi</password>
    </server>
  </servers>
  <!--阿里云镜像-->
  <mirrors>
    <mirror>
      <id>alimaven</id>
      <name>aliyun maven</name>
      <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
      <mirrorOf>central</mirrorOf>
    </mirror>
  </mirrors>
  <proxies/>
  <profiles>
  </profiles>
  <activeProfiles/>
</settings>

```

设置server，添加nexus的登录账号密码

#### pom.xml

添加拉取仓库地址、插件地址和上传仓库地址

```
<!--拉取仓库-->
    <repositories>
        <repository>
            <id>maven_xiaoxi</id>
            <url>http://10.10.0.44:8081/repository/maven-public/</url>
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
            <url>http://10.10.0.44:8081/repository/maven-public/</url>
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
            <url>http://10.10.0.44:8081/repository/maven-releases/</url>
        </repository>
        <snapshotRepository>
            <id>maven_xiaoxi</id>
            <url>http://10.10.0.44:8081/repository/maven-snapshots/</url>
        </snapshotRepository>
    </distributionManagement>
```

这样默认就会从私服拉取了，然后发布的话mvn deploy就可以发布到私服



## maven jar制品仓库

此仓库，可以存打包好的jar包，可以作为制品的备份存储

老方法创建一个maven-repo仓库，类型为Mixed，Allow redeploy

![image-20210401140306978](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401140306978.png)





## docker镜像私服仓库

该仓库用于，制品docker镜像的存储及备份

docker私服镜像，我们也需要添加类似maven的central\public等

#### 新建docker-central仓库阿里云代理

![image-20210401100701786](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401100701786.png)

![image-20210401100816712](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401100816712.png)

设置aliyun镜像地址，这个地址具体参考[阿里云镜像加速](https://help.aliyun.com/document_detail/60750.html)



#### 新建docker-repo仓库

![image-20210401095340218](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401095340218.png)

新建一个docker(hosted)

![image-20210401101113483](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401101113483.png)

添加一个端口8082，这样就可以使用10.10.0.44:8082来进行镜像的上传或者拉取了，这样方便管理端口



#### 新建docker-public组

![image-20210401101237853](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401101237853.png)

![image-20210401101435633](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401101435633.png)

设置端口8083，增加和调整仓库顺序，这样拉取就可以走10.10.0.44:8083进行拉取，而它会优先走阿里云镜像加速器，所以拉取就会很快

| 地址            | 描述                  |
| --------------- | --------------------- |
| 10.10.0.44:8082 | maven-repo 上传镜像   |
| 10.10.0.44:8083 | maven-public 拉取镜像 |

设置docker的deamon.json文件

```
vi /etc/docker/daemon.json
```

添加

```
"insecure-registries":["10.10.0.44:8082","10.10.0.44:8083"
```

```
systemctl restart docker
```



![image-20210401102814105](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401102814105.png)

重启docker即可

```
docker login -u admin -p xiaoxi 10.10.0.44:8082
docker login -u admin -p xiaoxi 10.10.0.44:8083
```

![image-20210401103216044](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401103216044.png)

docker登录之后，就可以进行相应的私服的拉取和上传操作了



# 九、jenkins配置相关

## jenkins插件下载

![image-20210401104839721](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401104839721.png)

![image-20210401104857289](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401104857289.png)

#### 自行下载这些插件：

gitlab，gitlab hook plugin ,，gitlab plugin，jquery ， sonar，Nexus Platform

sonar scanner， post build task，Publish Over SSH，Maven Integration，docker，nodejs

![image-20210401105606158](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401105606158.png)

添加完成插件之后，要在**Global Tool Configuration**中设置自动安装，及其对应版本

![image-20210406172912355](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406172912355.png)





#### 设置sonarqube scanner

![image-20210401110823446](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401110823446.png)

进入系统配置中

![image-20210401111036891](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401111036891.png)

地址就是http://10.10.0.44:9000

要添加凭据

![image-20210401111226212](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401111226212.png)

如果没有jenkins下拉的选项，就可以去其他地方添加一下，或者去设置-凭据管理

![image-20210401111646212](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401111646212.png)

完成后选择该凭据即可

![image-20210401111725969](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401111725969.png)

#### 添加nexus

![image-20210401112004130](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401112004130.png)

纠错（图中ip地址应该是10.10.0.44:8081），也相应添加凭据

![image-20210401112215244](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401112215244.png)

Test Connection测试一下是否连接成功



#### 设置ssh

![image-20210401142534645](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401142534645.png)

设置一下ssh服务器，也就是你后面会部署的机子，包括一个远程目录等



## 全局configureTool设置

设置maven版本

![image-20210401113830736](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401113830736.png)

设置sonarqube

![image-20210401113856634](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401113856634.png)







### 新建maven项目的任务

![image-20210401112900359](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401112900359.png)

设置git源，也需要添加gitlab的相应凭据，不再赘述

![image-20210401112928012](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401112928012.png)

设置gitlab构建触发器，这里是使用gitlab的webhook方式

![image-20210401113047030](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401113047030.png)

高级里面，生成token

![image-20210401113119743](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401113119743.png)

这个token自己可以记录下来

#### gitlab设置webhook

![image-20210401113330346](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401113330346.png)

默认情况会有报错，因为gitlab默认地址需要时外网地址，而我们现在在局域网内。这里可以设置一下安全选项。

![image-20210401113554151](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401113554151.png)

保存一下，再次重新添加webhook即可

![image-20210401113718305](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401113718305.png)

这样就好了。



#### 设置sonarqube

![image-20210401114524062](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401114524062.png)

![image-20210401114620523](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401114620523.png)

```
sonar.projectKey=${POM_GROUPID}:${POM_ARTIFACTID}
# this is the name and version displayed in the SonarQube UI. Was mandatory prior to SonarQube 6.1.
sonar.projectName=${POM_ARTIFACTID}-${POM_VERSION}
sonar.projectVersion=${POM_VERSION}
sonar.login=admin
sonar.password=xiaoxi
# Path is relative to the sonar-project.properties file. Replace "\" by "/" on Windows.
# This property is optional if sonar.modules is set. 
sonar.sources=.
sonar.scm.provider=git
sonar.java.binaries=.
```



### 关于maven项目jenkins中的环境变量

| 变量名             | 描述             |
| ------------------ | ---------------- |
| ${POM_GROUPID}     | pom的groupid     |
| ${POM_PACKAGING}   | pom的packaging   |
| ${POM_DISPLAYNAME} | pom的projectName |
| ${POM_ARTIFACTID}  | pom的artifactid  |
| ${POM_VERSION}     | pom的version     |

你可以在pre-post添加一个shell打印看看

![image-20210401132757578](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401132757578.png)

![image-20210401132810390](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401132810390.png)

```
echo "pom groupId: ${POM_GROUPID}"
echo "pom packaging: ${POM_PACKAGING}"
echo "pom displayName: ${POM_DISPLAYNAME}"
echo "pom artifactId: ${POM_ARTIFACTID}"
echo "pom version: ${POM_VERSION}"
```



### 设置maven参数

![image-20210401115028055](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401115028055.png)

这里我们暂时跳过了单元测试，后面会开启

```
clean package -Dmaven.test.failure.ignore=false
```

`-Dmaven.test.failure.ignore=false`可以让maven构建单元测试失败的时候，导致构建失败（默认情况下，会继续往下进行）



## 阶段性测试

OK到目前为止，我们已经做好了，gitlab的代码提交触发，sonarqube的代码扫描，和maven的打包。接下来我们来测试一下。

你可以修改一下代码，然后提交，这里就会有新的构建触发。

![image-20210401115154589](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401115154589.png)

不出意外的话就会成功的构建，工作区是有代码的，且target目录下是有打好的jar包的

![image-20210401115334705](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401115334705.png)



## 接下来还要做的

接下来，我们就要maven打包好了之后，把jar传到nexus私服，然后生成镜像文件，然后镜像上传nexus私服，然后在目标机器上运行docker容器实例。



### docker打包发布

这里我们利用maven的docker插件来进行发布镜像

修改pom.xml

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
                    <skipDocker>false</skipDocker>
                    <!--指定生成的镜像名,这里是我们的项目名-->
                    <imageName>${project.artifactId}</imageName>
                    <!-- 覆盖之前相同的tag-->
                    <forceTags>true</forceTags>
                    <!--指定标签 这里指定的是镜像的版本，我们默认版本是latest-->
                    <imageTags>
<!--                        <imageTag>latest</imageTag>-->
                        <imageTag>${project.version}</imageTag>
                    </imageTags>

                    <serverId>maven_xiaoxi</serverId>
                    <registryUrl>http://10.10.0.44:8081/repository/docker-repo/</registryUrl>

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
                    <dockerHost>http://10.10.0.44:2375</dockerHost>

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



#### 修改任务配置 maven 参数

![image-20210401133526509](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401133526509.png)

```
clean package  docker:build -DskipTests
```



### 保存jar到nexus私服

![image-20210401133818192](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401133818192.png)

post step中添加Nexus Repository Manager Publisher

![image-20210401134101382](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401134101382.png)

![image-20210401134151264](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401134151264.png)

```
$POM_GROUPID
$POM_ARTIFACTID
$POM_VERSION
jar
target/${POM_ARTIFACTID}.jar
jar
```







### 保存然后测试一下

按道理修改了代码提交之后，就会触发构建，构建成功之后，nexus maven-repo仓库会有jar，然后目标主机上也会有相应的docker镜像

#### pom.xml版本

![image-20210401140714782](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401140714782.png)



![image-20210401140632106](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401140632106.png)

![image-20210401140940639](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401140940639.png)

![image-20210401140535452](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401140535452.png)

![image-20210401140603703](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401140603703.png)



可以看出，我们的不管是jar包，还是镜像tag，还是sonarqube的版本号，都是跟pom的version一致的，这里就是利用maven项目在jenkins中的环境变量来保证的。如果其他非maven方式的，还有其他的解决办法。类似于用groovy脚本解析pom.xml，读取其中的属性；或者其他相应的插件也可以。



## 接下来做镜像的运行及其发布nexus镜像私服

#### 增加ssh上传并执行脚本shell

后面就是涉及容器实例的运行相关，我们可以使用ssh上传一个shell脚本，然后执行，从而来完成容器的运行等

#### 编写shell

我们可以把shell新建在项目的根目录下，这样shell脚本也能加入git的代码管理当中

![image-20210401141705777](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401141705777.png)

```
#!/bin/bash -l
POM_VERSION=$1
POM_ARTIFACTID=$2
REGISTRY_URL=$3
echo $POM_ARTIFACTID
echo $POM_VERSION
echo $REGISTRY_URL
#登录远端nexus registry
docker login -u admin -p xiaoxi $REGISTRY_URL
#打要上传nexus的tag
docker tag  $POM_ARTIFACTID:$POM_VERSION $REGISTRY_URL/$POM_ARTIFACTID:$POM_VERSION
#推送到远端
docker push $REGISTRY_URL/$POM_ARTIFACTID:$POM_VERSION
#删除打到远端的tag
docker rmi -f $REGISTRY_URL/$POM_ARTIFACTID:$POM_VERSION
#停止实例
if docker ps | grep $POM_ARTIFACTID;then
    docker stop $POM_ARTIFACTID
fi
#删除实例
if docker ps -a | grep $POM_ARTIFACTID;then
    docker rm $POM_ARTIFACTID
fi
#重新运行新的实例
docker run -d --name $POM_ARTIFACTID -p 8080:8080 $POM_ARTIFACTID:$POM_VERSION
#删除对应的image
#docker images|awk '{print $1":"$2}'|grep $POM_ARTIFACTID|xargs -t docker rmi
#删除<none>的被覆盖的实例
docker images|awk '{print  $1"\t"$3}'|grep "<none"|awk '{print $2}'| xargs -t docker rmi
exit 0
```

脚本大概做的事情就是，打私服ip对应的tag，然后push到私服镜像库去，停止掉并删除原来的容器实例，重新运行新的实例。删除一些覆盖而冗余的<none>的镜像等。



#### jenkins任务配置

![image-20210401141934903](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401141934903.png)

构建后操作增加 Send build artifactc over SSH

![image-20210401142128820](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401142128820.png)

执行脚本

```
cd /home/jar
sh startdocker.sh ${POM_VERSION} ${POM_ARTIFACTID} 10.10.0.44:8082
```



## 保存并测试

![image-20210401142716710](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401142716710.png)

目标服务器上  镜像也有了，实例也运行起来了

![image-20210401142757052](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401142757052.png)

nexus镜像docker-repo也有镜像了

![image-20210401142828276](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401142828276.png)

服务也启动起来了，接口也能正常返回啦！！！！恭喜！！！！



## 测试不同版本及其代码改动

我这边pom现在是3.0-SNAPSHOT，测试修改为4.0-SNAPSHOT，然后提交代码触发构建。

![image-20210401143850544](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401143850544.png)

![image-20210401144015233](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401144015233.png)



### 结果

![image-20210401144211387](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401144211387.png)

![image-20210401144239709](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401144239709.png)

4.0的镜像制品也有了

![image-20210401144314605](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401144314605.png)

镜像也有了，且还保留有3.0的版本

容器也启动了

![image-20210401144348084](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210401144348084.png)

接口也更新了！



# 十、关于多分支多环境多任务配置

### 分支管理

![image-20210406173147635](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406173147635.png)

我们是基于多分支的开发模式，这里就涉及多任务多环境多分支的部署。



features -> develop（测试环境） -> release（预发布环境） -> master（生产环境）



这里我们是利用maven-docker的插件，直接在目标机器上生成了image镜像，利用ssh命令执行镜像上传nexus、重启应用等操作。而对于不同的环境而言，这样做就比较局限了，所以，这里我们要做一些改进。



理想的逻辑：jenkins 构建打包生成image镜像，然后上传到nexus；其他环境的主机，利用ssh执行命令，先从nexus私服拉取镜像，再进行部署或者重启。



这样的话，就可以做到多个环境的主机去nexus拉镜像即可。

![image-20210406173905596](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406173905596.png)



#### 暂不使用maven的docker插件

![image-20210406173939068](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406173939068.png)



#### 添加dockerfile

![image-20210406174042550](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406174042550.png)



```
FROM adoptopenjdk:8-jdk-openj9
WORKDIR /ROOT
ADD /target/test1.jar //
ENTRYPOINT ["java", "-jar", "/test1.jar"]
```

这里使用pom的finnalName修改了jar包的名字，所以这里是test1.jar而不是test1-1.0-SNAPSHOT.jar，这样的好处是，dockerfile不必关注jar的版本

![image-20210406174219510](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406174219510.png)



#### jenkins任务配置中增加docker配置

![image-20210406174355024](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406174355024.png)

```
${WORKSPACE}
dockerfile的目录，这里默认是jenkins工作目录的根目录

${nexusurl}/${POM_ARTIFACTID}:${POM_VERSION}
生成镜像的名字，类似于最终为  10.10.0.44/test1:1.0-SNAPSHOT

```



### 修改startdocker.sh

```
#!/bin/bash -l
POM_VERSION=$1
POM_ARTIFACTID=$2
REGISTRY_URL=$3
echo $POM_ARTIFACTID
echo $POM_VERSION
echo $REGISTRY_URL
#登录远端nexus registry
docker login -u admin -p xiaoxi $REGISTRY_URL
#打要上传nexus的tag (取消)
#docker tag  $POM_ARTIFACTID:$POM_VERSION $REGISTRY_URL/$POM_ARTIFACTID:$POM_VERSION
#推送到远端(取消)
#docker push $REGISTRY_URL/$POM_ARTIFACTID:$POM_VERSION
#删除打到远端的tag(取消)
#docker rmi -f $REGISTRY_URL/$POM_ARTIFACTID:$POM_VERSION
#停止实例
if docker ps | grep $POM_ARTIFACTID;then
    docker stop $POM_ARTIFACTID
fi

#删除实例
if docker ps -a | grep $POM_ARTIFACTID;then
    docker rm $POM_ARTIFACTID
fi
#重新运行新的实例
docker run -d --name $POM_ARTIFACTID --net=host $REGISTRY_URL/$POM_ARTIFACTID:$POM_VERSION
#删除对应的image
#docker images|awk '{print $1":"$2}'|grep $POM_ARTIFACTID|xargs -t docker rmi
#删除<none>的被覆盖的实例
docker images|awk '{print  $1"\t"$3}'|grep "<none"|awk '{print $2}'| xargs -t docker rmi
exit 0
```

这里就注释取消了从主机上上传镜像的过程，只保留从nexus拉取和重新部署的脚本





### 不同的环境不同的ssh即可

在jenkins系统配置中添加多个环境的ssh，比如我添加了 测试环境ssh\生产环境ssh

![image-20210406174729923](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406174729923.png)

在多个任务中只需要使用不同的ssh即可让不同环境的主机部署容器实例



### 环境变量优化

![image-20210406174900670](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406174900670.png)

这里根据一些可能会变动的配置变量生产了参数变量，这样可以在手动构建的时候修改参数配置，增加了配置灵活性

![image-20210406175009669](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406175009669.png)



## 最终jenkins配置

![image-20210406175033633](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406175033633.png)

#### jenkins tool config 配置截图

![](https://gitee.com/jafir/blogs/raw/master/2021/images/10.10.0.44_8040_configureTools_.png)



#### jenkins系统配置截图

![](https://gitee.com/jafir/blogs/raw/master/2021/images/10.10.0.44_8040_configure.png)

#### jenkins测试环境任务配置截图

![](https://gitee.com/jafir/blogs/raw/master/2021/images/10.10.0.44_8040_job_test1-test.png)

#### jenkins生产环境任务配置截图

![](https://gitee.com/jafir/blogs/raw/master/2021/images/10.10.0.44_8040_job_test1-prod.png)





# 十一、关联tapd流水线

[tapd关联jenkins等流水线文档](https://www.tapd.cn/help/show#1120003271001000200)

### 关联gitlab

![image-20210406171703917](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406171703917.png)

webhook参数添加到gitlab中

![image-20210406171731731](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406171731731.png)

关联gitlab源码

![image-20210406172544405](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406172544405.png)

你可以在提交代码的commit里面，添加复制的信息，tapd就可以统计对应的git提交次数及其对应需求、缺陷、迭代等

![image-20210406172644212](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406172644212.png)



### 关联jenkins

![image-20210406171822657](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406171822657.png)

这里需要先下载tapd插件，然后jenkins本地上传

![image-20210406171923156](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406171923156.png)

然后设置jenkinsApiToken webhook参数等

![image-20210406171857422](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406171857422.png)



### nexus sonarqube jmeter ansible

其他的jenkins已经依赖了，这里就直接保存即可

![image-20210406172043493](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406172043493.png)



### 拷贝对应id到jenkins的任务配置中

![image-20210406172137982](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406172137982.png)

![image-20210406172155427](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406172155427.png)



### 添加jmeter tapd报告

![image-20210406172306067](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406172306067.png)

路径还是跟Publish Performance 插件的jtl文件路径一致（也就是你jmeter报告生成的位置）



## 完成关联

![image-20210406172412003](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406172412003.png)

![image-20210406172427134](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210406172427134.png)





### 一键多任务执行

如果说你的项目是微服务项目，你想要一键把所有的微服务都进行构建；或者你想一键发布测试、生产环境，就要用到多任务执行。

下载`multijob` 插件

#### 修改node最大执行数

![image-20210407160500109](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210407160500109.png)



![image-20210407160514431](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210407160514431.png)



默认是2，这里并行执行的话需要改多一点。

![image-20210407160608741](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210407160608741.png)

新建multijob ，构建里面添加 `multijob phase`

![image-20210407160651176](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210407160651176.png)

设置多个任务，然后选择` Running phase jobs in parallel`，保存即可

当然你可以还可以设置一些其他的condition条件，自行摸索

![image-20210407160828132](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210407160828132.png)

立即构建，就可以看到多个构建任务并行的在执行了。





# 最后

如果以后结合微服务的话，可能还会有一些改动。



### 整理

| url                    | 描述                    |
| ---------------------- | ----------------------- |
| http://10.10.0.44:7002 | gitlab                  |
| http://10.10.0.44:8040 | jenkins                 |
| http://10.10.0.44:8081 | nexus                   |
| http://10.10.0.44:8082 | nexus docker镜像库 上传 |
| http://10.10.0.44:8083 | nexus docker镜像库 下载 |
| http://10.10.0.44:9000 | sonarqube               |
|                        |                         |



### 账号密码

| 服务      | 账号   | 密码      |
| --------- | ------ | --------- |
| gitlab    | root   | xiaoxi123 |
| jenkins   | xiaoxi | xiaoxi    |
| nexus     | admin  | xiaoxi    |
| sonarqube | admin  | xiaoxi    |
| 44虚拟机  | root   | xiaoxi    |







# 关于作者

作者是一个热爱学习、开源、分享，传播正能量，喜欢打篮球、头发还很多的程序员-。-

热烈欢迎大家关注、点赞、评论交流！

简书：https://www.jianshu.com/u/d234d1569eed

github：https://github.com/fly7632785

CSDN：https://blog.csdn.net/fly7632785

掘金：https://juejin.im/user/5efd8d205188252e58582dc7/posts