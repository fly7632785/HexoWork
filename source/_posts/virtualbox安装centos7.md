# 前言

我们需要一个linux虚拟机，且保证它在局域网内是一个独立ip的存在，保证其他局域网里面的虚拟机（包括宿主机）能够ssh访问它



# 开始

这里用的是virtualbox，自行下载，centos7的镜像也自行下载

### 新建一个虚拟机，默认linux，RedHat即可

![image-20210330144224558](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330144224558.png)

自行设置内存，推荐8G以上

![image-20210330144407622](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330144407622.png)

![image-20210330144416259](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330144416259.png)

默认VDI

![image-20210330144437436](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330144437436.png)

动态分配

![image-20210330144454866](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330144454866.png)

设置存放位置，最好不要设在C盘

![image-20210330144621625](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330144621625.png)

创建好了之后，设置盘片

![image-20210330144717797](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330144717797.png)

选择一个iso镜像，centos7的，自行官网下载。

![image-20210330144759110](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330144759110.png)

![image-20210330144820738](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330144820738.png)

## 设置网卡为桥接网卡

网络模式有多种nat\host only\桥接等，这里桥接最适合我们，所以要选桥接，简单点说就是可以独占一个ip

![image-20210330151720670](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330151720670.png)

调整一下启动顺序

![image-20210330152912082](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330152912082.png)



### 启动

![image-20210330145133950](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330145133950.png)

选择安装位置

![image-20210330145249132](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330145249132.png)

## 选择带UI的

![image-20210330153126294](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330153126294.png)

### 打开以太网

![image-20210330145430955](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330145430955.png)

这里10.10.0.17就是默认生成的ip地址，开始安装，回头再重新设置网卡的静态ip

### 开始安装

![image-20210330145624438](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330145624438.png)

### 设置root密码 和 用户

![image-20210330145651274](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330145651274.png)

### 开启许可证

![image-20210330154811935](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330154811935.png)

### 打开网络连接

![image-20210330155044308](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330155044308.png)



### 开启ssh 22端口，或者直接关闭防火墙

开启22

```
firewall-cmd --zone=public --permanent --add-port=22/tcp  
```

关闭防火墙

```
systemctl stop  firewalld    #关闭防火墙
systemctl disable  firewalld   #关闭自启
```



### 设置静态ip

![image-20210330155335710](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330155335710.png)

```
ip addr
```

就可以查看自己的ip了，现在默认是动态的

此时其他机子已经可以ssh访问了

现在我们设置静态固定ip

进入网桥设置目录

```
cd /etc/sysconfig/network-scripts/
```

```
ls
```

![image-20210330150508737](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210330150508737.png)

修改ifcfg-enp0s3，这个每个人的可能名字不同

```
sudo vi ifcfg-enp0s3
```

添加或修改设置

这些设置可以先查看一下宿主机的配置，包括子网掩码，网关，dns等都一样

```
BOOTPROTO=static
ONBOOT=yes #自启
IPADDR=10.10.0.44      #需要的静态ip
NETMASK=255.255.240.0  #你宿主机的网络掩码 
GATEWAY=10.10.10.1  #网关一般是x.x.x.1
DNS1=61.139.2.69 #宿主机的dns
NM_CONTROLLED=no 
```

重启网卡

```
systemctl restart network.service
```



## 最后

```
ip addr
```

查看网络地址

实现了我们想要的效果，`10.10.0.44`为静态ip

然后你可以ping一下`baidu.com`，测试一下，外网应该也是没有问题的

其他局域网内的其他机子，包括其他的虚拟机，宿主机都可以ssh访问了。





# 关于作者

作者是一个热爱学习、开源、分享，传播正能量，喜欢打篮球、头发还很多的程序员-。-

热烈欢迎大家关注、点赞、评论交流！

简书：https://www.jianshu.com/u/d234d1569eed

github：https://github.com/fly7632785

CSDN：https://blog.csdn.net/fly7632785

掘金：https://juejin.im/user/5efd8d205188252e58582dc7/posts