# 前言

作者目前打算分享一期关于devOps系列的文章，希望对热爱学习和探索的你有所帮助。

文章主要记录一些简洁、高效的运维部署指令，旨在 **记录**和能够**快速地构建系统**。就像运维文档或者手册一样，方便进行系统的重建、改造和优化。每篇文章独立出来，可以单独作为其中一项组件的部署和使用。

本章为 **devOps系列（二）gitlab搭建**



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

#### 创建数据目录
```shell
mkdir -p /data/gitlab/{config,logs,data}
```

#### 启动docker
```
docker run --detach \
  --hostname gitlab.jafir.top \
  --publish 443:443 --publish 80:80 --publish 22:22 \
  --name gitlab \
  --restart always \
  --volume /data/gitlab/config:/etc/gitlab \
  --volume /data/gitlab/logs:/var/log/gitlab \
  --volume /data/gitlab/data:/var/opt/gitlab \
  --shm-size 256m \
  --env GITLAB_ROOT_PASSWORD=VpVV344L \
  --env GITLAB_TIMEZONE=Beijing \
  --env GITLAB_HOST=gitlab.jafir.top \
  --env TZ=Asia/Shanghai \
  gitlab/gitlab-ce:14.9.3-ce.0
```
等待几分钟后，待系统完成初始化。

### 域名

目前我是有一个根域名（jafir.top），对应的也可以替换为公司的根域名。

关于域名：建议搭建的服务的访问地址都采用域名来访问，内网的就解析内网ip。避免使用ip:port传统方式，使用和移植的时候都会很痛苦。

#### 访问gitlab
随后解析域名gitlab.jafir.top到服务器地址，使用浏览器访问 http://gitlab.jafir.top， 默认登录帐号密码：root VpVV344L


#### 修改gitlab中克隆的链接域名
```
vi gitlab.rb
```

修改

```shell
external_url 'http://gitlab.jafir.top'
```
重新应用配置
```shell
gitlab-ctl reconfigure
```