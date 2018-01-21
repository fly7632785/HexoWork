---
title: hello
date: 2017-01-18 17:22:10
tags: 
---
### 前言
很久以前就说弄一个自己的github.io博客，但是一直没有下决心花时间去弄，现在好了，弄了一个，
期间遇到了很多问题呢，觉得学到了的很多东西，当然也踩了很多坑。不过最终还是选定了一个material的主题。

### 踩坑
遇到了很多问题，我是mac系统
- 首先装node.js
- 安装git
- npm install hexo (最好加一个sudo因为是要安装到系统local下面)
- 到你创建的对应的文件夹 cd /xxxx （比如cd /user/jafir/hexo）
- hexo init
- hexo generate(简写 hexo g)
- hexo server (开启服务器测试本地 http://localhost:4000)
- 如果成功，那么就创建github repository(遇到一个严重坑 xx.github.io  xx必须要跟你的username一致)
- 如果你之前已经有过上传的经验（比如已经在本地上传过项目到github），那么你就不需要再配置ssh了
![username 和 repository一致](http://upload-images.jianshu.io/upload_images/1945618-7e4b5b66b4e0c708.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 不然，你就要获取ssh（/user/yourname/.ssh），找到id_rsa.pub 文件打开，然后复制内容
- 到github settings -> SSH and GPG keys,添加一个ssh,title 随便填，内容粘贴 保存 ，ok 
![打开id_rsa.pub,复制内容](http://upload-images.jianshu.io/upload_images/1945618-946df6deb330f92f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![没有就要添加](http://upload-images.jianshu.io/upload_images/1945618-d17ea418f9b3df8f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![粘贴](http://upload-images.jianshu.io/upload_images/1945618-70b8c4f108f217f3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 配置_config.yml文件（里面有包括什么网页的标题和一些信息）
- 主要就修改这几个地方
![SunShineIos替换为你的username](http://upload-images.jianshu.io/upload_images/1945618-c2a79bc6c2aac85b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 这里的deploy地址就是你repository的ssh地址（可以是ssh 也可以是 https）
![主题可以换有很大，百度搜"hexo好看主题"](http://upload-images.jianshu.io/upload_images/1945618-ad136a1c2a51c199.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![修改language等，注意:后面都有一个空格](http://upload-images.jianshu.io/upload_images/1945618-9c243ac2ca7859f0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 最后就hexo deploy(简写 hexo d, 如果没有反应那么你要执行 npm install hexo-deployer-git --save)
- 不出意外就成功了，地址是 http://username.github.io
- 如果你还想弄个域名，可以，直接去阿里云搜（我买了一个jafir.top,感觉.top有点高级，但还是因为头年4块）
- 买好了域名，在域名->解析里面，添加
- 1、添加 记录类型：A（表示address），主机记录：@ ，记录值：192.30.252.153 ,保存
- 2、添加 记录类型：A（表示address），主机记录：@ ，记录值：192.30.252.154 ,保存（这俩都是github的主机地址）
- 3、添加 记录类型：www（表示www），主机记录：CNAME ，记录值：username.github.io ,保存
- 域名弄好了，然后在hexo的项目目录下的 source 里面加入 一个文件（没有后缀），里面写入 jafir.top(你域名，不要www)
- 然后重新发布（可以 hexo d -g 一起）
- 如果说你没有本地项目没有.git文件，你可以执行 git.init
- 注意一点：生效是有一定时间的，如果你在过程中出现了一些问题，导致你加载username.github.io直接跳到你的域名，但是又还没有生效，你过一会它还是那样的，你就需要清理一下你的浏览器缓存
- 如果不出意外，搞定，开心!
- 如果还不行这里有两篇比较好的文章，综合一下应该没有问题
- http://www.jianshu.com/p/fdb5d7978cdb
- http://www.jianshu.com/p/8ec1b1e5d3b2

### 总结
还是比较喜欢总结的，因为古有温故而知新，我有总结而知新，总结是优秀的学习方法中很重要的一环，多多总结，经验才会显得牢靠，而不是泛泛而谈，没有达到脚踏实地的效果，
 自己做的一些有意义的事情，最后总结一下，能够加深自己的理解和印象，让你烙下深深的印记，perfect！

 