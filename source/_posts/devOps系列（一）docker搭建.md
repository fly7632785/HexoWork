# 前言

作者目前打算分享一期关于devOps系列的文章，希望对热爱学习和探索的你有所帮助。

文章主要记录一些简洁、高效的运维部署指令，旨在 **记录**和能够**快速地构建系统**。就像运维文档或者手册一样，方便进行系统的重建、改造和优化。每篇文章独立出来，可以单独作为其中一项组件的部署和使用。

本章为 **devOps系列（一）docker搭建**



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

## Docker社区版
适用于单机docker部署环境。
#### 添加yum源
```
cat <<EOF > /etc/yum.repos.d/docker-ce.repo
[docker-ce-stable]
name=Docker CE Stable - \$basearch
baseurl=https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/\$releasever/\$basearch/stable
enabled=1
gpgcheck=1
gpgkey=https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/gpg
EOF

yum clean all;yum makecache
```
#### 安装软件包
```
yum install docker-ce-20.10.14 -y
```
#### 添加配置

目前数据盘都是挂载在/data目录，所以需要把docker的数据也都放在/data下最好，做到系统盘和数据盘分离，方便后期扩容。

这里也指定了docker日志系统的大小和数量，避免日志文件占用过大的磁盘空间。

镜像仓库可以指定，这里是我自己阿里云上免费的。

```
mkdir -p /etc/docker
mkdir -p /data/docker
tee /etc/docker/daemon.json <<-'EOF'
{
  "data-root": "/data/docker",
  "log-opts": {"max-size": "50m", "max-file": "5"},
  "registry-mirrors": ["https://8e3pea7v.mirror.aliyuncs.com"]
}
EOF
```
#### 启动服务
```
systemctl enable docker.service
systemctl start docker.service
```







