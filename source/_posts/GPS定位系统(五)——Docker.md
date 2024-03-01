# 前言

前面已经把Android、Java、web端都弄得差不多了，现在，需要打包并进行服务器部署了，这样，我们的网站也能够在公网上给大家进行访问。接下来，想要学习一下docker来进行服务器环境的搭建和部署，因为docker的部署有诸多好处，所以也学习实践一番。



### GPS定位系统系列

>[GPS定位系统(一)——介绍](https://www.jianshu.com/p/1aa130ffbeaf)
>
>[GPS定位系统(二)——Android端](https://www.jianshu.com/p/e463b1ea6b7d)
>
>[GPS定位系统(三)——Java后端](https://www.jianshu.com/p/2e51318c56e0)
>
>[GPS定位系统(四)——Vue前端](https://www.jianshu.com/p/9df016cd65fa)
>
>[GPS定位系统(五)——Docker](https://www.jianshu.com/p/75ba5bc90988)
>
>- [Docker nginx 二级域名无端口访问多个web项目](https://www.jianshu.com/p/3378d2eacb3d)
>- [Docker nginx https二级域名无端口访问多个web项目](https://www.jianshu.com/p/3e25b24d7247)
>- [持续部署——Travis+Docker+阿里云容器镜像](https://www.jianshu.com/p/ce648e120727)



### 收获

学习完这篇文章你将收获：

>- docker的好处及其相关
>- docker部署mysql 构建数据库
>- docker部署nginx 构建静态web网站
>- docker部署java 构建后端服务提供接口
>- docker+nginx部署多个web网站应用
>- docker+nginx部署多个https的web网站应用



[TOC]



# 一、docker介绍

docker有很多好处，可能光看定义理论会有些理解不了，但是通过使用实践以及一些案例的探究，就会发现docker真的是实实在在的，优化配置效率，节约时间、人力成本。以下是个人总结docker的一些好处（可能有交叉）

- 1、一次构建，随处运行，多端跨平台

- 2、可移植性，方便运维部署

- 3、轻量高效，优于虚拟机

- 4、隔离性、松耦合

- 5、方便持续集成、持续交付及其自动化
- 6、管理方便



## 镜像与容器

关于docker，需要先介绍一下，关于容器和镜像。

docker其实顾其名和logo思义，就能发现，最明显的特征就是集装箱（容器）。

要知道，在全球的物流运输中，**集装箱**的标准化可谓是立了大功，这样的标准化使得，集装箱能够在船舶、汽车、火车、飞机等等交通工具中**任意装载和运输**，跟docker的**构建一次，随处运行**非常地契合。集装箱就类似于docker中的容器的概念，而镜像就类似于容器中具体装载什么物料的清单（个人理解）。

docker运行一个服务的流程，简单来说就是，**构建镜像，生成容器，运行容器**。用刚才那个例子类比，则为**拟一张货料清单，装成一个集装箱，搬上交通工具。**



## 好处及例子

### 1、一次构建，随处运行，多端跨平台

docker首先win、mac、linux等都兼容跨平台，部署服务器不再怕各种不同的操作系统。无论一个系统部署地有多么的复杂，配置了多少的模块或者服务，都可以一次性生成镜像，复用在其他各处。

例如：

公司需要配置一个服务器环境，mysql、nginx、tomcat、redis、java、python、go等多个环境，配置个系统环境，花了一天半天，然后，某天，领导有让你去帮忙给客户部署N多台相同环境的服务器，你怎么办，那不是要又去一台一台的配置N次吗？有了docker，你可以生成一个镜像，上传镜像仓库，然后，其他N台服务器就安装docker，拉镜像，运行就OK了。节约了配置时间，而且隔离了不同的服务器或者操作系统，是不是很方便呀。

### 2、可移植性，方便运维部署

一般开发和运维的关系是这样的，开发完成打包后，丢给运维，运维进行部署更新等，对于你所需的运行环境，运维都必须了如指掌，且最好让线上环境的配置跟你本地的测试环境的配置一样，保证其环境一样，排除其他影响条件。但是，这样的话，沟通成本，环境部署成本，都很高，还不能避免一些人为操作失误。

有了docker，你可以直接把打包成镜像，上传仓库，运维人员，只需要几条命令，拉镜像，运行。而且，如果线上版本出现了问题，运维还可以直接回滚上一个版本的镜像就OK了。

### 3、轻量高效，优于虚拟机

一般的虚拟机分配的都比较大，docker容器则很轻量，可以到M级别，构建部署运行几乎都是秒级别的，操作效率时分得高。

### 4、隔离性、松耦合

每个容器之间是隔离的，相互没有影响。像以前的大型项目，如果都合在一起，如果其中某个模块或者功能出现了问题，直接就会影响到其他的功能的使用，耦合性十分的大。但是，有了docker，再完美配合微服务、SOA等架构体系的实施，就能保证各个模块之间的独立自治。也可以使得开发人员各司其职，”各自为政“。

### 5、方便持续集成、持续交付及其自动化

现在很多大型公司的开发流程较为成熟完善，基本都需要做自动化相关，而docker就能很方便地配合各种自动化框架(jenkins、travis)进行自动化集成、部署、运行等。

关于**持续集成、持续交付、持续部署**等概念，这里个人简单做一下阐述

**持续集成**：以前是多个开发测试完毕后，才把代码往主分支上合并，可能交叉、冲突，合并之后还可能产生新的问题。而持续集成，就是经常提交代码到主分支，一天可能好几次。并且，自动化地进行单元测试并提供测试报告等，这样的话就能拆分细度颗粒，保证产品能够一步一步地安全可靠地迭代。很多以测试驱动开发的公司就是这样做的。

**持续交付**：持续集成的基础上，增加打包构建形成产物。

**持续部署**：持续交付的基础上，增加部署到相应的线上环境。

总而言之，这些持续做的事情，就是为了经常提交代码自动化测试、运行、部署，反馈问题，解决问题，再测试、运行、部署依次循环。不过持续部署也是有好处的，我们可以直接提交代码，后续一系列过程都是自动化的，就不用管了，只需要看一下是否出错就行了。

### 6、管理方便

各个容器管理方便，可以轻易地安装、卸载、升级、启动、停止等等。

如果换做直接在系统上部署环境，各个配置文件之间可能存在交叉或者污染系统本身的一些环境变量的配置，想想就难受。

总之，docker的好处是很多的，可能以上的不是最全面的，但是是我个人能够理解和体会的。



# 二、docker安装

[官网安装](https://docs.docker.com/engine/install/)

安装可选win、mac、linux，win和mac都提供了desktop的桌面应用管理，linux的主要是应用于服务器。

照着文档安装即可。

这里还要说一下【kitematic】和【kubernetes】

**kitematic**是docker以前的版本官方推荐的一个轻量的桌面管理docker容器、配置的软件。**kubernetes**是最新版本推荐的，比较重一些，功能也比较多一些，可以通过docker软件间接下载安装使用。

其实如果熟悉命令一点的同学，完全用不着它们-。- 直接使用命令来查看container就行了。



### 国内镜像加速

一般docker去拉镜像的话，可能有些镜像会很慢，这里可以使用aliyun的镜像加速，[官网方法](https://help.aliyun.com/knowledge_detail/60750.html)

1、登录aliyun，进入[容器镜像服务控制台](https://cr.console.aliyun.com/?spm=a2c4g.11186623.2.14.4da340bdhnLmjr) 左侧点击 【镜像加速器】，就会生成一个你自己独有的加速器地址，类似【xxxxxx.mirror.aliyuncs.com】

2、在docker的settings中，json里面加入

```json
{
    "registry-mirrors": ["<your accelerate address>"]
}
```

![image-20200715100928020](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200715100928020.png)

### 常用docker命令

>docker ps 查看现在运行的容器
>
>docker ps -a 查看所有容器，包括已经退出的
>
>docker rm xxx  删除容器  -f 强制删除
>
>docker rmi xxx 删除镜像 
>
>docker start xxx 启动容器
>
>docker stop xxx 停止容器
>
>docker restart xxx 重启容器
>
>docker exec -it xxxx  /bin/bash  进入容器内部的系统 可查看文件等
>
>docker  images 查看镜像





# 三、docker部署mysql

## 1、使用命令创建

```shell
docker run --name mysql -p 3306:3306 -v /mydockerdata/mysql/data:/var/lib/mysql -v /mydockerdata/mysql/mysql.cnf:/etc/mysql/my.cnf -v /mydockerdata/mysql/log:/var/log/mysql -e MYSQL_ROOT_PASSWORD=7632785 -d mysql:5.7.0
```

## 2、命令解释

```shell
docker 
run 运行一个容器
--name mysql 给container容器起名字
-p 3306:3306 端口映射 把容器内的3306端口映射到本机的3306   本机端口：容器端口
-v /mydockerdata/mysql/data:/var/lib/mysql  映射mysql的数据库文件目录到本机
-v /mydockerdata/mysql/mysql.cnf:/etc/mysql/my.cnf 映射mysql的配置到本机
-v /mydockerdata/mysql/log:/var/log/mysql 映射mysql的log目录到本机
-e MYSQL_ROOT_PASSWORD=7632785 设置初始化的root_password环境变量
-d 后台运行
mysql:5.7.0  镜像及其版本
```

### 端口映射

```
-p 3309:3307
```

容器内的端口，我们把容器看做一个封闭的沙盒，就像一个只有mysql的"操作系统"一样。

比如你的数据库的端口是**3307**，那么按道理，你就应该把"操作系统”的**3307**给占用，对于容器而言，则为**3307**容器端口需要使用，则你就可以把容器的**3307**进行映射，比如映射到本机的**3309**端口，这样你进行访问则为：

- **外部访问3309 ->等同于访问本机的3309-> 进而访问docker容器的3307->访问到mysql的3307数据库**

映射命令则为：`-p 3309:3307`

### 数据库文件映射

```
-v /mydockerdata/mysql/data:/var/lib/mysql
```

要知道，我们使用docker管理mysql，很有可能会多次rm掉mysql的container然后重装，这时候，如果你的数据文件就会丢失了，如何做到，**无论mysql怎么安装卸载，数据库数据都在**呢？那就需要数据库目录的映射了。

我们在本机单独建一个目录来管理mysql的一些log、my.cnf、data，我这里是`/mydockerdata/mysql`。

然后映射本机的`mydockerdata/mysql/data`到容器内部的`/var/lib/mysql`，本机的路径你可以自定，但是容器内部的路径，没有特殊操作，就是固定的`/var/lib/mysql`

### 数据库log映射

```
-v /mydockerdata/mysql/log:/var/log/mysql
```

同上

### 数据库配置文件映射

```
-v /mydockerdata/mysql/mysql.cnf:/etc/mysql/my.cnf
```

mysql有各个不同的版本，目前比较稳定的是5.6、5.7等，不过最新的已经8.0了，新版本的msyql增加了很多功能，如果需要兼容旧版本的mysql的使用方式，就需要更改一些配置，而在容器内，是没有办法直接通过`vim`命令来修改配置文件的，所以最好把配置文件映射出来。

自己先新建一个mysql.cnf空文件，或者先新建一个容器，并把其mysql.cnf文件配置拷贝出来，然后再进行映射，总之，自己先要有一个mysql.cnf文件，不管是不是空的。

### 数据库初始化密码

```
-e MYSQL_ROOT_PASSWORD=7632785
```

mysql初次创建，需要设置root密码

### log

如果创建出错，可以通过`docker logs mysql`(你容器的命名) 来查看log，或者进入映射的log目录来查看



# 四、docker部署nginx

#### 1、先随便创建一个nginx，目的是为了去把默认的nginx的配置拷贝出来，因为映射配置的时候，是需要有默认的配置文件的。

```
docker run nginx
```

```
docker ps 查看其container id等信息
```

#### 2、拷贝配置文件

```
nginx docker cp 【CONTAINER ID】 :/etc/nginx/ /mydockerdata/nginx/etc/
```

`/mydockerdata/nginx/etc/`为本机自建的管理目录

拷贝完了之后，可以删掉原来那个nginx

`docker rm nginx -f `

#### 3、重新创建nginx，并映射

```shell
docker run -p 8080:80 --name mynginx -v /mydockerdata/nginx/log/:/var/log/nginx -v /mydockerdata/nginx/etc/nginx/nginx.conf:/etc/nginx/nginx.conf -v /mydockerdata/nginx/dist/:/usr/share/nginx/html -d nginx
```

#### 命令解释

```
docker 
run 
-p 8080:80  这样就可以通过keep999.cn:8080访问网站了
--name mynginx 
-v /mydockerdata/nginx/log/:/var/log/nginx  映射log目录
-v /mydockerdata/nginx/etc/nginx/nginx.conf:/etc/nginx/nginx.conf 映射配置文件
-v /mydockerdata/nginx/dist/:/usr/share/nginx/html 映射静态web网页文件目录
-d 
nginx
```

这里我们把`/mydockerdata/nginx/dist`作为网站的静态文件的存放目录。

当我们打包好web前端网页，然后可以里面`scp`命令把文件上传到服务器的`/mydockerdata/nginx/dist`下，再容器nginx就可以实现reload新网页了。

```
scp -r  /Users/jafir/Documents/myadmin/dist/  root@keep999.cn:/mydockerdata/nginx/
```

`-r`为目录的递归文件

`/Users/jafir/Documents/myadmin/dist/`前端网页打包生成的dist文件目录

这样就可以通过keep999.cn:8080访问网站了，然后如果你的80端口没有被占用，你可以直接

```
-p 80:80 就可以使用keep999.cn直接访问网站了
```



**注意：**这里因为比较简单，所以就直接使用命令进行创建了，如果遇到复杂的需求，其实更应该使用dockerfile来进行配置，并且dockerfile后期对于使用[docker-compose](https://docs.docker.com/compose/reference/rm/)的一键发布启动有很大的铺垫意义，所以，大家也去了解一下dockerfile的配置和docker-compose。



#### nginx代理的静态单页面网站刷新，会报404错误

**问题：**不过对于单页面应用，会到一个问题：**nginx代理的静态单页面网站刷新，会报404错误**，那是因为nginx代理的应用入口只有一个，单页面的网页也是**内部进行的路由**，一个子页面的url，nginx代理后，浏览器是么有办法访问的，就报了404错误。解决办法就是配置一下nginx.conf文件

加入，使得，所有的url最终走向。index.html，由网页自己内部来进行路由。

```
 location / {
        try_files $uri $uri/ /index.html;
    }
```

nginx.conf中的配置

```
server {
    listen       80;         #监听的端口 注意这里是容器里面的端口 不是docker转发出来的端口
    server_name  keep999.cn;    #监听的URL 二级域名
    root  /usr/share/nginx/html;           #项目路径
    index index.html;
    # Any route that doesn't have a file extension (e.g. /devices)
    location / {
        try_files $uri $uri/ /index.html;
    }
```





# 五、docker部署java

部署java环境的话我们使用dockerfile来进行配置

#### 1、新建一个java 后台的管理目录 `/mydocker/java/gps`，并scp上传打包的jar到此目录

```
scp /Users/jafir/Downloads/SpringbootDemo/target/demo-0.0.1-SNAPSHOT.jar root@keep999.cn:/mydockerdata/java/gps/app.jar
```

#### 2、cd进入其目录`/mydocker/java/gps`，生成一个dockerfile文件

```
# FROM adoptopenjdk/openjdk8:ubi
# openj9内存会小很多
FROM adoptopenjdk:8-jdk-openj9    

VOLUME /tmp
COPY app.jar app.jar
RUN bash -c 'touch /app.jar'
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

这里可以使用`FROM adoptopenjdk:8-jdk-openj9`这个镜像，内存会少很多

参看https://v2ex.com/t/686196#reply12

#### 3、在dockerfile的目录下，创建镜像

```
docker build -t gps:1.0 .
```

这里我们每次发新包，可以**创建新版本的镜像**，如果出问题了，也可以**回滚**

#### 4、创建容器

```shell
docker run --name gps -p 9090:9090 -d -v /mydockerdata/java/gps/upload:/Users/jafir/Downloads/upload    -v /mydockerdata/arme/out/*.keep999.cn/*.keep999.cn.pfx:/Users/jafir/Downloads/upload/cert/*.keep999.cn.pfx      gps:1.0 
```

#### 命令解释

```
-v /mydockerdata/java/gps/upload:/Users/jafir/Downloads/upload
```

此映射为，**上传文件目录的映射**，项目里面是把`/Users/jafir/Downloads/upload`作为了上传文件的目录，这里最好把目录映射到本机，方便查看管理（如果没有可以不用映射）



```
-v /mydockerdata/arme/out/*.keep999.cn/*.keep999.cn.pfx:/Users/jafir/Downloads/upload/cert/*.keep999.cn.pfx
```

此映射为，**证书目录的映射**，由于使用了ssl的功能，所以需要映射一下本地的证书目录（如果没有可以不用映射）

相关可参看[Docker nginx https二级域名无端口访问多个web项目](https://www.jianshu.com/p/3e25b24d7247)

启动之后，便可以通过`keep999.cn:9090`来访问接口了，也可以用postman来调试一下。



# 六、Docker nginx 二级域名无端口访问多个web项目

具体可参看[Docker nginx 二级域名无端口访问多个web项目](https://www.jianshu.com/p/3378d2eacb3d)



# 七、Docker nginx https二级域名无端口访问多个web项目

具体可参看[Docker nginx https二级域名无端口访问多个web项目](https://www.jianshu.com/p/3e25b24d7247)



# 总结

前端、后端，我们都在服务器上使用docker部署好了，也可以访问使用了。docker相关的使用，nginx的配置，处理一些小问题等。总之，整个大体的框架及其步骤，我们都实践了一番，还是收获了很多的。就连docker的命令，以及其启动不同的容器，都已经滚瓜烂熟了-。-  linux服务器的操作命令也掌握了很多，很有成就感，希望大家也能够加油💪，每天都在进步。





# 关于作者

作者是一个热爱学习、开源、分享，传播正能量，喜欢打篮球、头发还很多的程序员-。-

热烈欢迎大家关注、点赞、评论交流！

简书：https://www.jianshu.com/u/d234d1569eed

github：https://github.com/fly7632785

CSDN：https://blog.csdn.net/fly7632785

掘金：https://juejin.im/user/5efd8d205188252e58582dc7/posts

