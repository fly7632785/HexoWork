# 前言

[Docker nginx部署二级域名无端口访问多个web项目](https://www.jianshu.com/p/3378d2eacb3d) 在这篇文章中，我们已经实现了在docker容器中利用二级域名无端口地去访问不同的项目。

继续进阶一下，搞一下https访问多个项目，提升一下安全性。

另外，再提一下，我们现在的框架结构为 docker 下的多个不同的nginx容器来管理项目的，没有跟传统的服务器下直接配置nginx管理多个项目一样，而是多了一层docker的装载。

虽然可能大多数情况下，直接服务器下部署nginx会简单方便很多，但是，为了学习多样化的、虚拟化的docker+nginx，了解更多端口转发、重定向、host网络等相关的东西。

当然，你也可以使用**docker+单nginx容器**的方式来实现，只需要把配置文件组合在一起就OK了，实际上更简单一些。但是如果我们想要**隔离来管理**不同的服务和网站web，更深入的**实践和了解**内部原理。目前依旧是**docker+多nginx服务**的方式来实现的。

一般nginx部署多个项目有3种方法：

>1、利用二级域名配置不同的项目
>
>2、利用不同的端口配置不同的项目
>
>3、利用不同的url路径来配置不同的项目

具体详情可以参见[使用nginx部署多个前端项目](https://www.cnblogs.com/zhaoxxnbsp/p/12691398.html)





[TOC]



### 收获

学习完这篇文章你将收获：

>- docker下申请免费自动续签的证书
>- 二级域名访问不同的项目
>- nginx配置https



### 期望

keep999.cn是一级域名，通过它访问到我朋友的项目，并且走https协议，http请求转发为https请求

a.keep999.cn是二级域名，通过它访问到我的项目，并且走https协议，http请求转发为https请求

![image-20200708172123463](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200708172123463.png)



# 正题

## 一、docker 使用arme.sh申请免费证书

[arme.sh](https://github.com/acmesh-official/acme.sh) 可以免费申请证书，我们将用它来申请证书。

arme.sh主要做的事情就是**申请证书**，并通过一个**cronjob脚本**来定期检查和申请更新证书

但是，情况比较特殊的是，我们需要**在docker中来跑申请和续约的脚本**。

github上就有docker如果使用arme.sh的方法[Run acme.sh in docker](https://github.com/acmesh-official/acme.sh/wiki/Run-acme.sh-in-docker)

> 申请证书的话有2种方式：
>
> 1、http
>
> 2、dns验证

http方式的话，需要80端口是空闲的才行，而我的80端口已经被占了（nginx服务），所以只能使用dns方式

我是阿里云，如果你其他的云，可以参看这里的用法：[How to use DNS API](https://github.com/acmesh-official/acme.sh/wiki/dnsapi)



### 1、申请阿里云accesskey

阿里云服务器dns验证的话，就是去申请一个accessKey和accesSecret

![image-20200708173552369](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200708173552369.png)

### 2、docker运行arme.sh

```shell
docker run --rm  -itd  \
  -v "$(pwd)/out":/acme.sh  \
  -e Ali_Key=你的accessKey \
  -e Ali_Secret=你的accessSecret \
  --net=host \
  --name=acme.sh \
  neilpang/acme.sh daemon
```

注意修改-e 的环境变量参数 为你自己的，生成的证书会在你pwd(当前目录)的out目录下

运行之后就可以使用 arme.sh 的命令来申请证书了

### 3、申请证书

自己生成一个目录来管理证书

```
mkdir /mydockerdata/arme
```

进入管理证书的目录

```
cd /mydockerdata/arme
```

申请证书

```shell
docker exec acme.sh --issue --dns dns_ali -d *.keep999.cn -d keep999.cn
```

```
docker exec acme.sh --issue --dns dns_ali -d *.keep999.cn -d keep999.cn  --toPkcs --password 7632785
# 转为pfx的证书 
```

**注意：我这里是给自己的域名及其二级域名都申请证书**

**并且一般证书是前端后端都用同一个的话，我的后端是java springboot构建的，tomcat只支持pkcs12和jks，还好arme.sh可以支持，转为pkcs（pfx）类型的证书。**



```shell
root@keep999:/mydockerdata# cd arme/
root@keep999:/mydockerdata/arme# ls
out
root@keep999:/mydockerdata/arme# cd out/
root@keep999:/mydockerdata/arme/out# ls
account.conf  ca  http.header  *.keep999.cn
root@keep999:/mydockerdata/arme/out# cd \*.keep999.cn/
root@keep999:/mydockerdata/arme/out/*.keep999.cn# ls
backup  ca.cer  fullchain.cer  *.keep999.cn.cer  *.keep999.cn.conf  *.keep999.cn.csr  *.keep999.cn.csr.conf  *.keep999.cn.key
root@keep999:/mydockerdata/arme/out/*.keep999.cn#
```

你的**accessKey**和**accessSecret**就在**account.conf**里面

好啦，证书就在*.keep999.cn目录下

如果使用nginx的话，就需要使用**fullchain.cer**和***.keep999.cn.key**



## 二、docker+多nginx来管理

先介绍一下具体架构

![image-20200708180645691](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200708180645691.png)





### 1、配置我的项目nginx

```shell
docker run -p 8080:443 --name mynginx -v /mydockerdata/nginx/log/:/var/log/nginx \
-v /mydockerdata/nginx/etc/nginx/nginx.conf:/etc/nginx/nginx.conf \
-v /mydockerdata/nginx/dist/:/usr/share/nginx/html  \
-v /mydockerdata/arme/out/*.keep999.cn/:/etc/nginx/cert \
-d nginx
```

**注意**：`-v /mydockerdata/arme/out/*.keep999.cn/:/etc/nginx/cert \`就是映射证书目录到nginx容器内部的证书目录



配置文件加入server

```
server {
        listen 443 ssl;
        server_name a.keep999.cn;
        root  /usr/share/nginx/html;           #项目路径
        index index.html index.htm;
        ssl_certificate   cert/fullchain.cer;
        ssl_certificate_key  cert/*.keep999.cn.key;
        location / {
            try_files $uri $uri/ /index.html;
        }
}
```

```
ssl_certificate   cert/fullchain.cer;
ssl_certificate_key  cert/*.keep999.cn.key;
```

`cert/fullchain.cer`就是**相对nginx.conf**的路径下的证书目录下的文件



具体怎么生成nginx和映射配置文件，请参看[Docker nginx部署二级域名无端口访问多个web项目]()

### 2、配置他的项目nginx

```shell
docker run -p 8081:443 --name web_nginx -v /mydockerdata/web_nginx/log/:/var/log/nginx \
-v /mydockerdata/web_nginx/nginx.conf:/etc/nginx/nginx.conf \
-v /var/www/html:/usr/share/nginx/html  \
-v /mydockerdata/arme/out/*.keep999.cn/:/etc/nginx/cert \
-d nginx
```

配置文件加入server

```
server {
    listen 443 ssl;
    server_name keep999.cn;
    root  /usr/share/nginx/html;           #项目路径
    index index.html index.htm;
    ssl_certificate   cert/fullchain.cer;
    ssl_certificate_key  cert/*.keep999.cn.key;
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```



### 3、配置代理nginx

```
docker run -p 80:80  -p 443:443 --name proxy_nginx \
-v /mydockerdata/proxy_nginx/log/:/var/log/nginx \
-v /mydockerdata/proxy_nginx/nginx.conf:/etc/nginx/nginx.conf \
-v /mydockerdata/arme/out/*.keep999.cn/:/etc/nginx/cert \
-d nginx
```

配置文件加入多个server

```
server {
    listen 80;
    server_name  a.keep999.cn;    #监听的URL
    rewrite ^(.*)$ https://${server_name}$1 permanent;
}

server {
     listen 443 ssl;
     server_name  a.keep999.cn;    #监听的URL
    ssl_certificate   cert/fullchain.cer;
    ssl_certificate_key  cert/*.keep999.cn.key;
    location / {
         proxy_redirect off;
         proxy_set_header Host $host;
         proxy_set_header X-Real-IP $remote_addr;
         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
         proxy_pass https://a.keep999.cn:8080;
    }

}

server {
    listen 80;
    server_name  keep999.cn;    #监听的URL
    rewrite ^(.*)$ https://${server_name}$1 permanent;

}
server {
    listen       443 ssl ;
    server_name  keep999.cn;    #监听的URL
    ssl_certificate   cert/fullchain.cer;
    ssl_certificate_key  cert/*.keep999.cn.key;
    location / {
           proxy_redirect off;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass https://keep999.cn:8081;
    }
}
```



**完成！**

![image-20200708181201853](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200708181201853.png)



这里呢是直接把最终方案写了出来，但其实在探索的过程中，尝试了很多种方式，有很多种方式都行不通，因为包裹了docker这层容器的关系，端口映射出来，很多端口都会被占据，比如80、443端口，一个容器占据了，其他容器就不能使用了。

所以，只有使用**多域名不同端口的转发**的方式来实现。



## 三、docker+单nginx来管理

其实，我们也可以在docker中用单nginx来管理多个项目，这样的话，配置会简单一些，实现起来也简单一些，只是管理不同的服务，没有那么灵活。这里，采用**基于二级域名**的方式来实现的。

```shell
docker run -p 80:80  -p 443:443 --name all_nginx \
-v /mydockerdata/all_nginx/log/:/var/log/nginx \
-v /mydockerdata/all_nginx/nginx.conf:/etc/nginx/nginx.conf \
-v /mydockerdata/nginx/dist/:/usr/share/nginx/html1  \
-v /var/www/html:/usr/share/nginx/html2  \
-v /mydockerdata/arme/out/*.keep999.cn/:/etc/nginx/cert \
-d nginx
```

不过映射的话要注意一下，`mydockerdata/nginx/dist/:/usr/share/nginx/html1` 把我的项目映射到**html1**,

`/var/www/html:/usr/share/nginx/html2`另一个项目映射到**html2**

完整的配置文件：

```
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;
    
    server {
        listen 80;
        server_name  a.keep999.cn;    #监听的URL
        rewrite ^(.*)$ https://${server_name}$1 permanent;
    }
    
    
    server {
        listen 80;
        server_name  keep999.cn;    #监听的URL
        rewrite ^(.*)$ https://${server_name}$1 permanent;
    }
    
    
    server {
            listen 443 ssl;
            server_name a.keep999.cn;
            root  /usr/share/nginx/html1;           #项目路径
            index index.html index.htm;
            ssl_certificate   cert/fullchain.cer;
            ssl_certificate_key  cert/*.keep999.cn.key;
            location / {
                try_files $uri $uri/ /index.html;
            }
    }
    
    server {
        listen 443 ssl;
        server_name keep999.cn;
        root  /usr/share/nginx/html2;           #项目路径
        index index.html index.htm;
        ssl_certificate   cert/fullchain.cer;
        ssl_certificate_key  cert/*.keep999.cn.key;
        location / {
            try_files $uri $uri/ /index.html;
        }
    }

    include /etc/nginx/conf.d/*.conf;
}
```





# 总结

已经把网站从开发->docker、nginx部署->http->https这一套流程都走完了，docker和其他微服务的联合应用，docker与主机的端口、映射路径、容器内部隔离等关系都理解清楚。

相信大家也收获了很多，**生命不息，学习不止**，继续加油吧💪



另外还有一点要注意：我们的网站支持了https，所以里面的接口请求也要支持https才行，也就是说我们的后台也需要支持https。后台呢，我是用的springboot，支持https的话请参见[GPS定位系统(三)——Java后端]()





# 关于作者

作者是一个热爱学习、开源、分享，传播正能量，喜欢打篮球、头发还很多的程序员-。-

热烈欢迎大家关注、点赞、评论交流！

简书：https://www.jianshu.com/u/d234d1569eed

github：https://github.com/fly7632785

CSDN：https://blog.csdn.net/fly7632785

掘金：https://juejin.im/user/5efd8d205188252e58582dc7/posts