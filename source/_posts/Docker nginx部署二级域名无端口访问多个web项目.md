# 前言

情况是这样的，我借了朋友的阿里云服务器 **用docker** 部署一下自己的网站（方便管理）。他的服务器自身也用nginx挂了一个网站，端口也用的是默认的80端口。服务器有域名，[keep999.cn](keep999.cn)

我的是docker里面添加的nginx容器，代理的静态网页。docker映射的是8080端口出来，所以，访问的话，需要

`keep999.cn:8080`来访问。现在的话，我不想加端口访问了，就想用域名（懒得写端口），于是乎，就要开始想办法解决这个问题了。



[TOC]



### 收获

学习完这篇文章你将收获：

>- docker创建nginx容器
>- nginx映射配置文件 网页目录 log目录
>- docker部署多个web网站
>- 二级域名转发到不同web网站
>- 阿里云设置云解析





# 正题

具体做法：

一、使用二级域名+转发来访问我的网站

二、使用域名解析



## 一、使用二级域名+转发来访问我的网站

大致方法：

>docker容器3个nginx容器
>
>1、mynginx 我的网站部署的nginx 端口8080
>
>2、web_nginx 朋友的网站部署的nginx 端口8081
>
>3、proxy_nginx 用于代理转发的nginx 端口80



先说这种方法，是比较理想，比较优质的，统一使用nginx来做入口的代理和转发。**需要注意的是，proxy_nginx的端口，必须是80，不是其他的**。因为你浏览器访问域名，默认不写端口，就是80端口。

### 1、首先要阿里云配置一个二级域名。

![image-20200707111501684](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200707111501684.png)

配置安全组，我这里直接开放了80【proxy_nginx】、8080-8100【部署多个网站】多个端口

![image-20200707111415458](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200707111415458.png)



### 2、docker创建容器，并且映射配置文件等

#### 1）docker创建mynginx  部署我的网站

新建一个nginx去把配置文件拷出来供映射的时候使用

```
docker run nginx
```

自己新建一个目录管理nginx，比如我这里是/mydockerdata/nginx 用这个目录来管理，并创建dist log目录     

```
mkdir log
mkdir dist 

docker ps 查看container 的ID
```

拷贝配置文件到自己的管理目录下面 

```
nginx docker cp 【CONTAINER ID】 :/etc/nginx/nginx.conf /mydockerdata/nginx/
```

拷贝完成之后，可以把新建的nginx删了，重新创建自己的

```
docker rm nginx -f 
```

上传静态网站的到/mydockerdata/nginx/dist下面（这个目录就是存放网页的）

```
scp -r  /Users/jafir/Documents/myadmin/dist/  root@keep999.cn:/mydockerdata/nginx/
```

创建nginx容器

```
docker run 
-p 8080:80 映射端口本机8080到容器80
--name mynginx container的名字mynginx
-v /mydockerdata/nginx/log/:/var/log/nginx 映射log文件目录
-v /mydockerdata/nginx/nginx.conf:/etc/nginx/nginx.conf 映射配置文件
-v /mydockerdata/nginx/dist/:/usr/share/nginx/html 映射网页存放目录
-d 后台运行
nginx 镜像
```

**注意**这里本机的ip是8080，容器内是80（多个容器内的80端口是不会相互冲突影响的，因为docker的容器就是隔离的），外界访问是通过8080

修改本机nginx.conf的配置即可修改容器里面的nginx.conf配置（因为做了映射）

```
vim /mydockerdata/nginx/nginx.conf
```

http节点下添加 server

```java
server {
        listen       80;         #监听的端口
        server_name  keep999.cn;    #监听的URL
        root  /usr/share/nginx/html;           #项目路径
        index index.html;
        # Any route that doesn't have a file extension (e.g. /devices)
        location / {
            try_files $uri $uri/ /index.html;
        }
    
    }
```

按esc，:wq    保存成功  重启mynginx就可以了

```
docker restart mynginx
```

这样就成功了，访问http://47.108.59.73:8080 或 http://keep999.cn:8080就能访问网站了



#### 2） 按照跟1）一样的方法，新建/mydockerdata/web_nginx目录来管理朋友的网站

由于朋友的网站路径是在**/var/www/html**下，所以创建容器的时候网站路径映射要改一下

```
docker run -p 8081:80 --name web_nginx -v /mydockerdata/web_nginx/log/:/var/log/nginx -v /mydockerdata/web_nginx/nginx.conf:/etc/nginx/nginx.conf -v /var/www/html:/usr/share/nginx/html -d nginx
```

```
server {
        listen       80;         #监听的端口
        server_name  keep999.cn;    #监听的URL
        root  /usr/share/nginx/html;           #项目路径
        index index.html;
        # Any route that doesn't have a file extension (e.g. /devices)
        location / {
            try_files $uri $uri/ /index.html;
        }
   
    }
```

配置文件也是一样的，两个项目的配置文件最好分开

创建好了之后，就能通过http://47.108.59.73:8081来访问他的网站了



#### 3）创建proxy_nginx来代理和二级域名转发

创建nginx的管理目录也是一样的，`/mydockerdata/proxy_nginx`目录来管理

不过80原来是被朋友的nginx给占用了，可以先把它给杀掉，kill -9 pid，再创建我们的容器

```
docker run -p 80:80 --name proxy_nginx -v /mydockerdata/proxy_nginx/log/:/var/log/nginx -v /mydockerdata/proxy_nginx/nginx.conf:/etc/nginx/nginx.conf -v /mydockerdata/proxy_nginx/dist/:/usr/share/nginx/html -d nginx
```

这里的配置文件就比较重要了

```
    server {
        listen       80;         #监听的端口
        server_name  a.keep999.cn;    #监听的URL
        location / {
           proxy_redirect off;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://47.108.59.73:8080;
        }
    }
    server {
        listen       80;         #监听的端口
        server_name  keep999.cn;    #监听的URL
        location / {
           proxy_redirect off;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://47.108.59.73:8081;
        }
    }
```

把 [a.keep999.cn](a.keep999.cn) 转发到我的项目，我的项目访问是http://47.108.59.73:8080

把   [keep999.cn](keep999.cn)  转发给他的项目，他的项目访问是http://47.108.59.73:8081

修改了之后，docker restart proxy_nginx就可以咯

### 整个目录树结构/mydockerdata/下的

```
│─── mynginx  我的项目
|   ├──- dist
|   ├── log
|   │   ├── access.log
|   │   └── error.log
|   └── nginx.conf
├── proxy_nginx  代理
│   ├── dist
│   ├── log
│   │   ├── access.log
│   │   └── error.log
│   └── nginx.conf
└── web_nginx 朋友的项目
    ├── dist
    ├── log
    │   ├── access.log
    │   └── error.log
    └── nginx.conf
```

![image-20200707115009010](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200707115009010.png)

![image-20200707115721719](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200707115721719.png)



## 二、使用域名解析

第二种的话就很简单了，直接使用aliyun的云解析。

首先还是跟(一)的一些步骤一样，生成我的项目的nginx容器，并使得[keep999.cn:8080]()可以访问到我的项目

然后就添加云解析，【隐性url】

![image-20200707123700268](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200707123700268.png)

这样的话，就可以通过[b.keep999.cn](b.keep999.cn)来访问的的项目了。

但是呢，为啥没有推荐这种写法呢？因为本身朋友的服务器  nginx和docker俩同级的，按道理应该全部交给docker来创建容器，方便管理，也不占用80端口，可操作性更强。也是能够更好的去了解和掌握docker与容器的关系，docker与本机的关系。去学习docker及其微服务方面的更多的知识。





# 总结

nginx还是很强大的，功能远不止我们用到的这些。

其三大功能：反向代理、负载均衡、静态资源服务器，我们这里用到了反向代理、静态网站，还没有用到负载均衡。负载均衡，简单点说，就是当有多台服务器的时候，会根据设置的策略分散请求到不同的服务器上，分担各个服务器的压力。有兴趣的可以自己去尝试一下。

接下来，我们的接口也通了，觉得还可以进一步提升一下安全性，所以，下一篇，我们将继续，给网站访问加上https，学会申请免费的证书和使用nginx配置ssl，请参看[Docker nginx https二级域名无端口访问多个web项目](https://www.jianshu.com/p/3378d2eacb3d)





# 关于作者

作者是一个热爱学习、开源、分享，传播正能量，喜欢打篮球、头发还很多的程序员-。-

热烈欢迎大家关注、点赞、评论交流！

简书：https://www.jianshu.com/u/d234d1569eed

github：https://github.com/fly7632785

CSDN：https://blog.csdn.net/fly7632785

掘金：https://juejin.im/user/5efd8d205188252e58582dc7/posts

