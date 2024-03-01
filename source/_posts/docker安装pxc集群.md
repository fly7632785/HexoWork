# 前言

现在mysql自建集群方案有多种，keepalived、MHA、PXC、MYSQL主备等，但是目前根据自身情况和条件，选择使用pxc的放来进行搭建，最大的好处就是，多主多备，即主从一体，没有同步延时问题，方便易用。

本人使用过，直接安装pxc和docker容器方式的安装，个人觉得docker下安装更为方便，也更易维护，所以也推荐大家使用此方式。



## 搭建环境

| 环境                            |
| ------------------------------- |
| centos7                         |
| pxc版本镜像：最新版，目前为8.0+ |

| 主机ip      | 部署         | swarm   |
| ----------- | ------------ | ------- |
| 172.16.9.40 | pxc1         | manager |
| 172.16.9.41 | pxc2         | worker  |
| 172.16.9.42 | pxc3         | worker  |
| 172.16.9.48 | nginx 做负载 |         |



# 前期准备

linux需要关闭防火墙，或者开启某些需要的端口；pxc会自带mysql，版本是对应一致的，所以机子上不需要mysql；最好关闭SELINUX，linux自带的安全增强。

注意这些配置，三台机子上都要操作。

#### 1、开放pxc所需端口

| 端口 | 功能                     |
| ---- | ------------------------ |
| 3306 | mysql数据库              |
| 4567 | pxc cluster 相互通讯端口 |
| 4444 | sst全量传输              |
| 4568 | ist增量传输              |

这里给出一些linux下防火墙的命令相关

```
# 查询防火墙状态
systemctl status firewalld
# 查询防火墙状态
firewall-cmd --state
# 查询8080端口是否开放
firewall-cmd --query-port=8080/tcp
# 开放80端口
firewall-cmd --permanent --add-port=80/tcp
# 移除端口
firewall-cmd --permanent --remove-port=8080/tcp
清理防火墙
iptables -F
```

#### 2、关闭SELINUX、关闭mysql

永久关闭：

```
vi /etc/selinux/config
```

设置SELINUX为disable，然后reboot机子

临时关闭：

```
setenforce 0
```

关闭mysql

```
systemctl status mysql
systemctl stop mysql
```



#### 3、创建docker swarm集群

swarm也需要一些端口的开放，当然如果你是关闭防火墙就无需多言

| 端口 | 功能         |
| ---- | ------------ |
| 2377 | 用于集群通信 |
| 4789 | 容器覆盖网络 |
| 7946 | 容器网络发现 |

我这里是172.16.9.40作为主节点

```
docker swarm init 主节点的初始化

docker swarm join --token xxxx xxxx  其他节点的加入
```

40主节点 init之后，控制台就会出现  `docker swarm join --token xxxx xxxx`

然后41，42机子，就调用对应的命令，即可加入swarm集群

```
docker node ls 
```

可以查看现在的node信息，如下

```
root@srig config]# docker node ls
ID                            HOSTNAME                STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
vk3kzrob1b8jvjq9bxia8lwa7 *   srig.dcmp.database.m1   Ready     Active         Leader           20.10.3
4s0pj57d43hm71wipnnbckfkt     srig.dcmp.database.m2   Ready     Active                          20.10.3
ub1fe2qms2rlhmj9zlap20bsq     srig.dcmp.database.s1   Ready     Active                          20.10.3
```

```
docker node rm -f xxx 强制删除节点
docker swarm leave -f 主节点强制离开swarm集群
docker swarm leave 从节点离开swarm集群
```

### 4、创建虚拟网络

```
docker network create -d overlay --attachable xxxxx
```

其他相关命令

```
docker network inspect xxxx 查看改网络信息
docker network ls 查看所有网络信息
docker network rm xxxx 删除网络
```

这里网络名就叫，`swarm_mysql`，创建好了网络之后，`docker network inspect swarm_mysql` 查看（我这里是节点建立好了之后，就可以看到，有三台机子）

![image-20210224093551159](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20210224093551159.png)

### 5、创建目录及cert证书

如果你是8.0+且没有使用相同的证书，那么你肯定会遇到一个ssl相关的错误

```
“error:0407008A:rsa routines:RSA_padding_check_PKCS1_type_1:invalid padding”
```

这是因为8.0后，是ssl来连接，三台机子，就必须保持密钥的一致性才可以通信。

这是[官方的解决方案](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/install/docker.html)，生成证书，大家使用同一套。

##### 创建目录

一般情况下我们最好看一下系统磁盘的分区情况，然后把mysql的数据要放到大的磁盘上

```
df -h
```

![image-20210224130631346](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20210224130631346.png)

我这里`/home`下面最大，所以我的数据都是在`/home`下面

### ！注意这里的目录在三台机子上都要做相同的操作创建

```
cd /home
mkdir -m 777 pxc_cert       证书
mkdir -m 777 pxc_config     mysql自定义配置文件
mkdir -m 777 pxc_data      数据
```

注意：这里需要给予权限，不然很多地方会报错

##### 创建custom.cnf

```
cd /home/pxc_config
vi custom.cnf
```

输入内容 这里我们

```
[mysqld]
lower_case_table_names=1
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
ssl-ca = /cert/ca.pem
ssl-cert = /cert/server-cert.pem
ssl-key = /cert/server-key.pem

[client]
ssl-ca = /cert/ca.pem
ssl-cert = /cert/client-cert.pem
ssl-key = /cert/client-key.pem

[sst]
encrypt = 4
ssl-ca = /cert/ca.pem
ssl-cert = /cert/server-cert.pem
ssl-key = /cert/server-key.pem
```

我这里需要设置数据库不区分大小写 和 8.0以后 可以使用group by

##### 创建cert证书

```
docker run --name pxc-cert --rm -v /home/pxc_cert:/cert \
percona/percona-xtradb-cluster:8.0 mysql_ssl_rsa_setup -d /cert
```

就能在`/home/pxc_cert`目录下创建证书文件

#### ！注意这里的证书创建好，需要拷贝到其他两台机子上的相应目录去

```
scp -r root@172.16.9.40:/home/pxc_cert /Users/jafir/downloads/pxc_cert
```

下载到本地

```
scp -r /Users/jafir/Downloads/pxc_cert root@172.16.9.41:/home/
scp -r /Users/jafir/Downloads/pxc_cert root@172.16.9.42:/home/
```

上传到41 42其他两台机子

#### ！注意：三台机子都需要给你证书文件权限

```
cd /home/pxc_cert
chmod 777 *
```





# 搭建pxc集群

#### 1、安装镜像

```
docker pull percona/percona-xtradb-cluster
```

名字有点长，可以tag重命名

```
docker tag percona/percona-xtradb-cluster pxc
```

删除原来的

```
docker rmi percona/percona-xtradb-cluster
```

#### 2、多台机子创建容器

我这里是40主节点，其他是丛节点，所以40先开始创建

##### 172.9.16.40主节点

```
docker run -d -p 3306:3306 --net=swarm_mysql  \
-e MYSQL_ROOT_PASSWORD=asdw@123  \
-e CLUSTER_NAME=pxc_cluster \
-e XTRABACKUP_PASSWORD=asdw@123  \
-v /home/pxc_data:/var/lib/mysql  \
-v /home/pxc_cert:/cert \
-v /home/pxc_config/:/etc/percona-xtradb-cluster.conf.d  \
--privileged --name=pxc1  pxc
```

命令解读:

```
docker run -d 
-p 3306:3306  3306端口映射
--net=swarm_mysql  虚拟网络名字
-e MYSQL_ROOT_PASSWORD=asdw@123  数据库初始密码
-e CLUSTER_NAME=pxc_cluster 集群名字
-e XTRABACKUP_PASSWORD=asdw@123  备份密码
-v /home/pxc_cert:/cert 证书路径映射
-v /home/pxc:/var/lib/mysql  pxc路径映射 
-v /home/pxc/config/:/etc/percona-xtradb-cluster.conf.d  mysql配置文件路径映射
--privileged 给予权限
--name=pxc1  pxc
```

可以`docker logs pxc1`看看日志是否报错等

如果成功，你可以用Navicat连接看看是否成功启动了mysql，启动了之后再安装从节点。

##### 172.9.16.41节点

```
docker run -d -p 3306:3306 --net=swarm_mysql  \
-e MYSQL_ROOT_PASSWORD=asdw@123  \
-e CLUSTER_NAME=pxc_cluster \
-e XTRABACKUP_PASSWORD=asdw@123  \
-v /home/pxc_data:/var/lib/mysql  \
-v /home/pxc_cert:/cert \
-v /home/pxc_config/:/etc/percona-xtradb-cluster.conf.d  \
-e CLUSTER_JOIN=pxc1 \
--privileged --name=pxc2  pxc
```

这里跟上面比起来，多了一句 `-e CLUSTER_JOIN=pxc1` ，表示加入pxc1。
为第2台机子可以知道pxc1呢？就是因为swarm集群的建立，让彼此可以相互通信。

##### 172.9.16.42节点

```
docker run -d -p 3306:3306 --net=swarm_mysql  \
-e MYSQL_ROOT_PASSWORD=asdw@123  \
-e CLUSTER_NAME=pxc_cluster \
-e XTRABACKUP_PASSWORD=asdw@123  \
-v /home/pxc_data:/var/lib/mysql  \
-v /home/pxc_cert:/cert \
-v /home/pxc_config/:/etc/percona-xtradb-cluster.conf.d  \
-e CLUSTER_JOIN=pxc1 \
--privileged --name=pxc3  pxc
```

### 注意：如果你是8.0+那么你肯定会遇到一个ssl相关的错误

```
“error:0407008A:rsa routines:RSA_padding_check_PKCS1_type_1:invalid padding”
```

这是因为8.0后，是ssl来连接，三台机子，就必须保持密钥的一致性才可以通信。

这是[官方的解决方案](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/install/docker.html)，生成证书，大家使用同一套。我这边，就简单点，直接把主机点那台的文件考出来，然后传输给其他两台机子，覆盖之后，重启即可。

server-key.pem、server-cert.pem、client-key.pem、client-cert.pem、ca.pem

```
scp -r root@172.16.9.40:/home/pxc /Users/jafir/Downloads/pxc
```

从40节点，把数据拷贝下来，然后删除里面除了那5个文件的其他文件

```
scp -r /Users/jafir/Downloads/pxc root@172.16.9.41:/home
```

再上传到41、42上面去覆盖，然后重启即可

### 成功

如果三台都成功了，再确认一下。

主节点进入容器，再进入mysql查看

```
docker exec -it pxc1 sh
```

```
mysql -uroot -p 
```

```
show status like 'wsrep%';  
```

![image-20210224095442347](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20210224095442347.png)

不出意外，这里cluster size就是3台

```
docker network inspect xxx
```

![image-20210224093551159](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20210224093551159.png)

网络也是3个

### 校验

你可以在其中一台上Navicat 创建一个数据库，一张表等，就可以看到3台都同步了！



# nginx负载

nginx我是放在172.16.9.48上面的

如果没有nginx.conf配置文件，可以先随便建一个，然后把配置考出来

自建/nginx/log  /nginx/etc/nginx.conf等

```
docker run -d -name nginx nginx
docker cp nginx:/etc/nginx/nginx.conf  拷贝出来
docker rm -f nginx
```

##### nginx.conf的配置修改

在最后一行添加，也就是和http同级

```
stream {
    upstream pxc {
        server 172.16.9.40:3306;
        server 172.16.9.41:3306;
        server 172.16.9.42:3306;
    }
    server {
        listen 3306;
        proxy_pass pxc;
    }
}
```

```shell
docker run --net=host  --name nginx -v /nginx/log/:/var/log/nginx -v /nginx/etc/nginx.conf:/etc/nginx/nginx.conf -d nginx
```

然后Navicat连接http://172.16.9.48:3306也可以连上数据库啦





# 关于作者

作者是一个热爱学习、开源、分享，传播正能量，喜欢打篮球、头发还很多的程序员-。-

热烈欢迎大家关注、点赞、评论交流！

简书：https://www.jianshu.com/u/d234d1569eed

github：https://github.com/fly7632785

CSDN：https://blog.csdn.net/fly7632785

掘金：https://juejin.im/user/5efd8d205188252e58582dc7/posts