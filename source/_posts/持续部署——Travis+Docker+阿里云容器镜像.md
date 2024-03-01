# 前言

最近了解了一番持续集成、持续交付、持续部署相关。个人对此的相关理解也再说下。

>现在很多大型公司的开发流程较为成熟完善，基本都需要做自动化相关，而docker就能很方便地配合各种自动化框架(jenkins、travis)进行自动化集成、部署、运行等。
>
>关于**持续集成、持续交付、持续部署**等概念，这里个人简单做一下阐述
>
>**持续集成**：以前是多个开发测试完毕后，才把代码往主分支上合并，可能交叉、冲突，合并之后还可能产生新的问题。而持续集成，就是经常提交代码到主分支，一天可能好几次。并且，自动化地进行单元测试并提供测试报告等，这样的话就能拆分细度颗粒，保证产品能够一步一步地安全可靠地迭代。很多以测试驱动开发的公司就是这样做的。
>
>**持续交付**：持续集成的基础上，增加打包构建形成产物。
>
>**持续部署**：持续交付的基础上，增加部署到相应的线上环境。
>
>总而言之，这些持续做的事情，就是为了经常提交代码自动化测试、运行、部署，反馈问题，解决问题，再测试、运行、部署依次循环。持续部署还有个好处，对我们个人网站来说，我们可以直接提交代码，后续一系列过程都是自动化的，就不用管了，它自己自动部署发布。



关于**jenkins**和**travis**，jenkins是我个人比较推荐的，因为它生态好、插件环境强、历史悠久、稳定；而travis，个人主要是练手尝试，他可以直接与github互联，而我的代码也是放在github上的，所以直接试试它。



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

>- 无密ssh登录服务器
>- travis构建
>- 阿里云容器镜像服务
>- docker-compose
>- travis.yml脚本逻辑
>- 持续部署相关





[TOC]



# 正题

### 大体流程：

提交代码->触发github的hooks钩子->travis自动触发->依照.travis.yml构建环境，执行脚本->完成部署

### 脚本大体逻辑：

1、travis openssl加密访问服务器的私钥，生成加密文件xxx.enc，在travis中解密形成其本地密钥

2、项目maven打包

3、docker构建生成镜像

4、发布镜像到阿里云容器镜像

5、ssh访问服务器（利用解密后的本地[指travis运行环境]密钥），执行docker-compose完成部署

### 开发逻辑：

1、绑定travis和github

2、无密登录ssh，本地项目安装travis，加密密钥

3、阿里云开通容器镜像服务

4、添加.travis.yml和Dockerfile文件

5、提交代码



# 一、绑定travis和github

注意，travis现在有两个网站，https://www.travis-ci.org和https://www.travis-ci.com

千万不要搞混了，org的是免费的，对于github的开源项目免费，com是收费的，不过对private的项目有构建次数的限制。（我之前就是没有注意org还是com，结果发现，项目构建，一直出问题，`iv nodefined` 就是因为，add密钥的解密环境变量到org去了，而com下面没有，所以构建失败，如图）

![image-20200716152328630](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200716152328630.png)



1）直接sign in使用github账号登录，并授予权限

![image-20200717110844417](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200717110844418.png)

2）勾选开启你要构建的项目 即可



# 二、无密登录ssh，本地项目安装travis，加密密钥

按道理，一般自己都有了**id_rsa**和**id_rsa.pub**的密钥，路径在`~/.ssh/`下，这里我们再单独新建一个没有密码的密钥

#### 1）生成无密密钥

```shell
cd ~/.ssh
ssh-keygen -t rsa
```

第一次输入的是文件名：nopwd

第二次、第三次都输入密码，我们是无密码，所以直接回车

这样就生成好了密钥对，nopwd(私钥)和nopwd.pub(公钥)

#### 2）上传公钥到服务器

```
sh-copy-id -i ~/.ssh/nopwd.pub root@你的服务器地址
```

#### 3）安装travis

```
换源
gem sources -l
gem sources --add https://gems.ruby-china.org/ --remove 
安装
sudo gem install travis
切换到项目目录下
travis init
```

> 可能出现构建失败，安装依赖ruby-dev`sudo apt install ruby-dev`

#### 4）使用github账号登录travis

```
travis login --auto
```

如果本地有token或者github密钥的话，就能直接登录。不然，就输入账号密码

#### 5）加密密钥

```
travis encrypt-file ~/.ssh/nopwd --add
```

`--add` 会提示让你加入一段解密密钥的脚本到`.travis.yml`文件中，如果有文件就会添加到文件里，没有文件它给你创建或者你自己创建（travis init就会自动创建）。

>```
>before_install:
>- openssl aes-256-cbc -K $encrypted_d51746e9182b_key -iv $encrypted_d51746e9182b_iv
>  -in nopwd.enc -out ~/.ssh/nopwd -d
>```
>
>这就是要加入的内容，-out  ~/.ssh/xxx 这里你可以自己去名字，默认是id_rsa，后面ssh登录服务器会用到
>
>其中**$encrypted_d51746e9182b_key**和**$encrypted_d51746e9182b_iv**就是用来解密的

注意，这里travis会上传两个环境变量**$encrypted_d51746e9182b_key**和**$encrypted_d51746e9182b_iv**到服务器后台去

![image-20200717112521822](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200717112521822.png)

当然，**环境变量**的好处就是，你可以把.tavis.yml里面会用到的变量，都到这里来控制。

比如，你的addons添加的ip地址，就可以这里来定义，然后文件里，就$ip来使用。这样，如果ip地址有变动，直接修改一下配置就可以了。



# 三、阿里云开通容器镜像服务

阿里云 产品与服务中 找到【容器镜像服务】然后，添加镜像。当然你也可以使用**dockerhub**来进行镜像的管理，不过速度可能没有国内的阿里云快。

这里我是web前端一个，java后端一个。

每个容器都可以管理不同版本的镜像文件。

![image-20200717112833270](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200717112833270.png)

![image-20200717112936933](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200717112936933.png)

好了，之后就有相关的文档供配置使用。



# 四、添加.travis.yml和Dockerfile文件

## java后端项目配置

[java后端github地址](https://github.com/fly7632785/GpsServer)

### Dockerfile

在项目根目录下，创建Dockerfile（以前在服务器端创建过，这里直接放到项目中来了）

```
# FROM adoptopenjdk/openjdk8:ubi
# openj9内存会小很多
FROM adoptopenjdk:8-jdk-openj9

VOLUME /tmp
COPY target/demo-0.0.1-SNAPSHOT.jar app.jar
RUN bash -c 'touch /app.jar'
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

逻辑大致为：拉取jdk镜像，拷贝项目生成的jar包到容器，运行jar包

### .travis.yml

```yaml
language: java
services:
- docker
sudo: required
branches:
  only:
  - master
#使用ssh登陆的时候会确认主机信息，travis-ci自动化运行无法进行交互操作，所以在.travis.yml中添加以下内容跳过确认
addons:
  ssh_known_hosts: $server_ip
before_install:
- openssl aes-256-cbc -K $encrypted_d51746e9182b_key -iv $encrypted_d51746e9182b_iv
  -in nopwd.enc -out ~/.ssh/nopwd -d
script:
- mvn install -DskipTests=true -Dmaven.javadoc.skip=true -B -V
- docker build -t registry.cn-hangzhou.aliyuncs.com/jafir_docker_images/jafir-images:latest .
after_success:
- docker login --username=jafir@1866908499405822 -p=$aliyun_pwd registry.cn-hangzhou.aliyuncs.com
- docker push registry.cn-hangzhou.aliyuncs.com/jafir_docker_images/jafir-images:latest
- chmod 600 ~/.ssh/nopwd
- ssh -o "StrictHostKeyChecking no" -i ~/.ssh/nopwd root@$server_ip "cd /mydockerdata;docker-compose -f docker-compose.yml pull;docker-compose -f docker-compose.yml up -d;exit"
```

**使用ssh登陆的时候会确认主机信息，travis-ci自动化运行无法进行交互操作，所以在.travis.yml中添加**，

这里我们使用**环境变量**来控制，避免写死。环境变量，在travis的后台settings里面添加

```
addons:
  ssh_known_hosts: $server_ip
```

`-t` 后面则是你阿里云容器的镜像地址，可以设置版本，这里直接使用**latest**最新版

```
- docker build -t registry.cn-hangzhou.aliyuncs.com/jafir_docker_images/jafir-images:latest .
```

**登陆阿里云容器镜像，并推送镜像**

```
- docker login --username=jafir@1866908499405822 -p=$aliyun_pwd registry.cn-hangzhou.aliyuncs.com
- docker push registry.cn-hangzhou.aliyuncs.com/jafir_docker_images/jafir-images:latest
```

这里跟文档上有点不同，你可以直接`-p`加入你自己的密码，就不用输入了，密码也是**环境变量**来配置aliyun_pwd

![image-20200717142422260](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200717142422260.png)



密钥文件可能没有权限，给予权限

```
chmod 600 ~/.ssh/nopwd
```

**ssh登录服务器，执行docker-compose拉取镜像，开启服务**

```
ssh -o "StrictHostKeyChecking no" -i ~/.ssh/nopwd root@47.108.82.91 "cd /mydockerdata;docker-compose -f docker-compose.yml pull;docker-compose -f docker-compose.yml up -d;exit"
```

`-i ~/.ssh/nopwd`使用travsi系统中的密钥登录

**docker-compose.yml**

```
version: '2'
services:
  mysql:
    container_name: mysql1
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: 数据库密码
    ports:
      - "3306:3306"
    volumes:
      - /usr/local/docker/mysql/data:/var/lib/mysql
      - /usr/local/docker/mysql/conf:/etc/mysql
      - /usr/local/docker/mysql/logs:/var/log/mysql
  web:
    container_name: jafir_nginx1
    image: registry.cn-hangzhou.aliyuncs.com/jafir_docker_images/web:latest
    ports:
      - 8080:80
    restart: always
    depends_on:
      - java
    volumes:
      - /mydockerdata/nginx/etc/nginx.conf:/etc/nginx/nginx.conf
      - /mydockerdata/nginx/log/:/var/log/nginx
  java:
    container_name: jafir_gps1
    image: registry.cn-hangzhou.aliyuncs.com/jafir_docker_images/jafir-images:latest
    restart: always
    depends_on:
      - mysql
    ports:
      - 9090:9090
    volumes:
      - /mydockerdata/java/gps/upload:/Users/jafir/Downloads/upload
      - /mydockerdata/arme/out/*.nofoo.cn/*.nofoo.cn.pfx:/Users/jafir/Downloads/upload/cert/*.nofoo.cn.pfx
```

我们这里直接使用**docker-compose**来启动3个服务，方便快捷。当然，你也可以单独启动java一个容器。

逻辑也比较清晰

完成后提交代码即可，不出意外的话就触发travis的构建，服务也能够启动起来啦。

![image-20200717115550739](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200717115550739.png)





## vue前端项目配置

[web前端github项目地址](https://github.com/fly7632785/myadmin/tree/gps)

### Dockerfile

在项目根目录下，创建Dockerfile

```
FROM nginx
COPY dist/ /usr/share/nginx/html/
```

逻辑大致为：拉取镜像，拷贝项目静态文件到nginx容器内默认的目录

### .travis.yml

```yaml
language: js
services:
- docker
sudo: required
branches:
  only:
  - gps
addons:
  ssh_known_hosts: $server_ip
before_install:
- openssl aes-256-cbc -K $encrypted_d51746e9182b_key -iv $encrypted_d51746e9182b_iv
  -in nopwd.enc -out ~/.ssh/nopwd -d
script:
- npm run build
- docker build -t registry.cn-hangzhou.aliyuncs.com/jafir_docker_images/web:latest .
after_success:
- docker login --username=jafir@1866908499405822 -p=$aliyun_pwd registry.cn-hangzhou.aliyuncs.com
- docker push registry.cn-hangzhou.aliyuncs.com/jafir_docker_images/jafir-web:latest
- chmod 600 ~/.ssh/nopwd
- ssh -o "StrictHostKeyChecking no" -i ~/.ssh/nopwd root@$server_ip "cd /mydockerdata;docker-compose -f docker-compose.yml pull;docker-compose -f docker-compose.yml up -d;exit"


```

主要逻辑与上面的几乎一致。

**不过要注意一点：**
以往我们都是通过上传到服务器的dist目录，再映射到 `/usr/share/nginx/html` 去的。而现在是，直接把代码**打包到镜像里面去**，所以，不需要再映射了，不然的话就会映射一个空目录进去了。

```yaml
 web:
    container_name: jafir_nginx1
    image: registry.cn-hangzhou.aliyuncs.com/jafir_docker_images/web:latest
    ports:
      - 8080:80
    restart: always
    depends_on:
      - java
    volumes:
      - /mydockerdata/nginx/etc/nginx.conf:/etc/nginx/nginx.conf
      - /mydockerdata/nginx/log/:/var/log/nginx
```

volumes是只有配置文件的映射的，**不需要项目目录的映射**。



# 五、提交代码，成功

随意提交一下代码，就能看到服务器的docker-compose的项目启动起来啦！

travis的控制台也能看到构建成功啦。

![image-20200717125015731](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200717125015731.png)









# 关于作者

作者是一个热爱学习、开源、分享，传播正能量，喜欢打篮球、头发还很多的程序员-。-

热烈欢迎大家关注、点赞、评论交流！

简书：https://www.jianshu.com/u/d234d1569eed

github：https://github.com/fly7632785

CSDN：https://blog.csdn.net/fly7632785

掘金：https://juejin.im/user/5efd8d205188252e58582dc7/posts





