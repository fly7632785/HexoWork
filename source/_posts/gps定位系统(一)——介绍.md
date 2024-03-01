# 前言

题外话：好久没有写博客了，简书也好久没有看了。最近一直在学习前端、后端的东西，想让自己的知识面更广一些，看到某篇文章讲的。`为了自己而工作（为了学习而工作）；学会自我营销（多端博客，敲门砖）；能把别人讲懂才是真正懂了（写博客）` 很有道理，跟当初的自己想法很契合，前段时间由于工作或者其他一些原因丢掉了写作、输出、分享，是时候该再捡起来了。

前段时间接了一两个小外包，大致是做一个Android端的gps定位APP，功能很简单就是保活、后台上传实时定位。做完了Android端之后，看了他们的后台，然后，我想，为啥我不自己也做一个后台呢，正好练练手，把前端和后端都做了，前后端分离，一举多得。进而开始着手，沉迷于边学习边实践的过程中，收获颇丰，现在差不多基本功能已经完成了，然后决定写一个系列的文章来总结、分享一下。

希望通过这个系列的文章，能够让大家也能一起来学习实践一下，练练手，也能够自己搭建一个多端结合的小系统。加油吧！



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



### 目录

[TOC]

### 收获

学习完整个系列你将收获：

>- 三端联合开发的经验
>- 地图应用、模拟定位、轨迹绘制、覆盖点、信息窗体
>- Android app保活
>- Java springboot+mybatis一套使用
>- 前端vue js css vuex vue-router一套使用
>- 后台admin管理页面



# 一、项目展示

### web端

![登录](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200702143257766.png)

![主页](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200702143344401.png)

![实时定位](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200702143431034.png)

![历史轨迹](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200702143808473.png)

![用户管理](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200702143910659.png)

web端页面大概就是这个样子，使用的是Vue的Iview框架。大致实现了**整体单页面功能**、**实时定位**、**历史轨迹**、**用户管理**、**个人信息**等功能。

并且已经部署到阿里云服务器上去了，如果服务还没有到期的话可以通过**[web网站地址](https://a.keep999.cn)**访问，账号密码kk kk 。

[github项目地址](https://github.com/fly7632785/myadmin/tree/gps)

### Android端

<img src="https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/device-2020-07-02-144812.png" alt="device-2020-07-02-144812" style="zoom:25%;" />

Android端界面也很简单，就是**登录**、**显示地图**，主要是**后台service上传gps定位信**息。

[github项目地址](https://github.com/fly7632785/Gps/tree/gps_mine)

### Java后端

由于前后端分离的，后端没啥页面展示的。

[github项目地址](https://github.com/fly7632785/GpsServer)



# 二、我的开发环境

> 1、mac笔记本、小米6手机
>
> 2、Android studio
>
> 3、Idea
>
> 4、webstorm
>
> 5、postman
>
> 6、Navicat
>
> 7、chrome、safari





# 三、项目架构及其技术选型

### Android

> targetSdkVersion：29
>
> rxjava + retrofit + okhttp
>
> 高德地图sdk
>
> butterknife
>
> rxpermission
>
> hellodaemon

保活最重要的就是使用了[hellodaemon](https://github.com/xingda920813/HelloDaemon)框架，它利用了双进程互拉保活机制、引导用户加入电量优化和白名单、其他一些常用的保活手段等，是一个挺不错的保活框架。

### Web

> Vue + Vuex + Vue-cl + Vue-router
>
> 高德js地图api
>
> iview
>
> es6

### Java

>java8
>
>springboot + mybatis 
>
>jwt
>
>mysql
>
>lombok
>
>mybatis-generator

### Docker

> nginx 
>
> mysql 5.7
>
> oepnjdk9

用了docker之后，很喜欢。很方便，在管理各个服务方面很出色，升级、维护，管理方便。



# 总结

整个系统开发下来，主流的框架都会接触到，各个技术栈也会有涉猎，总而言之就是能够从多端的角度来考虑和设计，从多维的角度来解决问题。

整个开发过程中会遇到许许多多的小问题，比如

- token的全局拦截器验证401、404的问题
- mybatis-generator生成器的问题
- restful的response结构，axios的统一封装
- 前后端跨域问题
- 高德地图显示的问题
- docker挂载映射本地路径的问题
- 上传文件的文件路径问题
- maven打包的问题
- 。。。

整个系列的文章中，三端相关的文章是最重要的，会包含很多小的技术点、代码、问题的解决方案等，希望整个系列能够给大家带来帮助

请移步[GPS定位系统(二)——Android端](https://www.jianshu.com/p/e463b1ea6b7d)





# 关于作者

作者是一个热爱学习、开源、分享，传播正能量，喜欢打篮球、头发还很多的程序员-。-

热烈欢迎大家关注、点赞、评论交流！

简书：https://www.jianshu.com/u/d234d1569eed

github：https://github.com/fly7632785

CSDN：https://blog.csdn.net/fly7632785

掘金：https://juejin.im/user/5efd8d205188252e58582dc7/posts

