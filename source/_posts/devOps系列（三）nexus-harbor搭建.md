# 前言

作者目前打算分享一期关于devOps系列的文章，希望对热爱学习和探索的你有所帮助。

本章为 **devOps系列（三）nexus-harbor搭建**



## 大纲

[devOps系列介绍]()

[devOps系列（一）docker搭建]()

[devOps系列（二）gitlab搭建]()

[devOps系列（三）nexus-harbor搭建]()

[devOps系列（四）jenkins搭建]()

[devOps系列（五）efk系统搭建]()

[devOps系列（六）grafana+prometheus搭建]()

[devOps系列（七）grafana+prometheus监控告警]()

[devOps系列（八）efk+prometheus+grafana日志监控和告警]()



# 正文

## 搭建nexus

#### 创建数据目录
```
mkdir -p /data/nexus/
chmod g+w /data/nexus/
```
#### 部署nexus
```
docker run --detach \
  --publish 8081:8081 \
  --name nexus \
  --restart always \
  --volume /data/nexus:/nexus-data \
  --ulimit nofile=65536:65536 \
  --env TZ=Asia/Shanghai \
  --env "INSTALL4J_ADD_VM_PARAMS=-Xms512m -Xmx1024m -XX:MaxDirectMemorySize=1024m -Djava.util.prefs.userRoot=/nexus-data/javaprefs" \
  sonatype/nexus3:3.38.1
```
#### 访问
浏览器访问 http://ip:8081 ，默认用户名是 admin，初始密码使用命令查看：
```
docker exec nexus cat /nexus-data/admin.password
```

##### 域名解析

建议：可以使用nexus.jafir.top解析到 一个nginx上80端口上，然后再转发到8081上，这样就可以使用http://nexus.jafir.top域名直接访问和管理nexus了。其他的也是如此，比如grafana.jafir.top， jenkins.jafir.top等等都是按照类似方式。







## 搭建harbor

#### 下载离线安装包
下载地址：https://github.com/goharbor/harbor/releases ，文件：harbor-offline-installer-xxx.tgz，下载到/data目录下。
#### 解压
```
cd /data

wget https://github.com/goharbor/harbor/releases/download/v2.5.0/harbor-offline-installer-v2.5.0.tgz

tar zxf harbor-offline-installer-v2.5.0.tgz
cd harbor
cp harbor.yml.tmpl harbor.yml
```
#### 安装docker-compose
```
curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```
#### 创建数据目录
```
mkdir -p /data/harbor/ssl
```
将证书与私钥文件上传到/data/harbor/ssl目录

证书可以去aliyun上申请免费证书，但是目前证书已经不是1年免费，而是3个月。但是可以花点钱申请和长期续费即可。

#### 修改配置文件
```
cd /data/harbor/
vi harbor.yml
```
```
# 访问域名
hostname: harbor.jafir.top

http:
  port: 80

https:
  port: 443
  # 证书与私钥位置（宿主机的位置）
  certificate: /data/harbor/ssl/fullchain.pem
  private_key: /data/harbor/ssl/privkey.pem

# 数据存储位置（宿主机的位置）
data_volume: /data/harbor
```
#### 部署harbor
```
./install.sh
```
#### 开启自启动容器
```
echo "/usr/local/bin/docker-compose -f /data/harbor/docker-compose.yml start" >> /etc/rc.local
chmod +x /etc/rc.local
chmod +x /etc/rc.d/rc.local
```
如果需要手工操作
```
cd /data/harbor/
docker-compose start
docker-compose stop
docker-compose restart
```
#### 访问harbor
随后解析域名harbor.jafir.top到服务器地址，使用浏览器访问 http://harbor.jafir.top ，默认登录帐号密码：admin Harbor12345



#### PS:

这里使用docker-compose进行全套harbor相关的安装，快捷方便，且直接把证书ssl也包含在内，目的主要是：docker默认是不能登录非https的仓库，要么就每个docker都去配置非https，要么直接harbor上https，这里后者简单一些。