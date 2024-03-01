# 前言

一个能把知识给别人讲清楚的人，一定是理解清楚透彻的人。

很多人都愿意本着分享、开源的精神，将自己的经验、见解无私地分享给生态、大众，我很敬佩这样的人，也一直向他们学习。



# 切换gitee码云图片仓库

由于github太不稳定了，所以自己有找了[gitee](https://gitee.com)仓库来做图片的存储

进入码云，然后创建一下私人token

![image-20200717123503931](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200717123503931.png)



![image-20200717123522626](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200717123522626.png)

生成的token自己记住。

新建一个**public**的仓库，专门管理md文件和图片。如果你原来有github，可以直接导入过来。

#### picgo增加插件

![image-20200717123705602](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200717123705602.png)

安装好，重新配置一下

![image-20200717123747512](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200717123747512.png)

主要是customUrl:

> 格式为https://gitee.com/jafir/blogs/raw/master

https://gitee.com/用户名/仓库名/raw/master

完成后就可以上传图片啦，很快。

![image-20200717123928374](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200717123928374.png)





# 正题

现在写博客，然后要发布到很多写作平台，例如，简书、csdn、掘金等，可能面临着有些图片防盗链或者失效的情况，我推荐的策略体系是:

> markdown+Typora + picGo + jsdelivr + github仓库 + bloghelper

#### Typora

[Typora](https://www.typora.io)无论是在win还是在mac都可以用，md的标准，适用于大多数主流的博客平台，小巧灵动，很多人都喜欢。

![typora](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200702174443491.png)

#### PicGo

[PicGo](https://github.com/Molunerfinn/PicGo) 快速上传图片并获取url连接的工具，win,mac都可以用。而且支持很多图床：七牛、腾讯云、又拍云、github、阿里云等等，有桌面状态栏直接拖拽上传，很方便。最终要的是可以和typora结合使用

![pickgo](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200702174517761.png)



#### jsDelivr

[jsDelivr](https://www.jsdelivr.com) 免费的开源仓库的cdn，支持npm、github、WordPress等

![jsDelivr](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200702174915492.png)

## 方案详情

#### 一、github生成token，生成仓库用于保存md和图片

在【Settings】-【Developer settings】-【Personal access tokens】-【Generate new token】

![](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200702180849939.png)

![](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200702181423199.png)

保存好token，后面会用到

创建一个github新仓库，用于存放md文件和图片



#### 二、pickGo设置github图床和jsDelivr的域名url

![](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/QQ20200702-181217.png)

> 仓库名：【用户名】/【你的仓库名】

> 分支：master

> token：就是刚才github的token

> 存储路径：指的是github的仓库的目录，这里推荐按年份来存放md和图片，明年你就改一下2021就行了

> 自定义域名：这个就需要用免费的jsDelivr来做github的cdn中转，这样访问会快很多
>
> https://cdn.jsdelivr.net/gh +【账户名fly7632785】+ 【仓库名blogs】@latest
>
> 例如我的：https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest





PS：有时候会出现上传图片失败的问题，不是picGo的问题，多数是因为github，你懂的。如果你有自己的oss或者其他云储存，可以把地址改为其他picGo支持的图床云储存。

### 三、一键发布到多个写作平台

我写博客是发布到简书、掘金、csdn上的，当然不想一个一个复制啦。所以，网上找了一圈儿工具，有些说openwrite、有些说bloghelper还有一些其他大型媒体工具比如易媒、简媒等，但是多数还是商业化的，说白了，就是要收费。openwrite：每月10次发布。发布底部还加了类似水印的文字。。。。不喜欢

github上的[bloghelper](https://github.com/ystcode/BlogHelper)，这个很不错，推荐，跨操作系统的。大致原理就是，登录拿cookie、token等信息，自己做请求进行发布。所以你需要先登录你要发布的平台。

![bloghelper](https://cdn.jsdelivr.net/gh/fly7632785/blogs@latest/2020/images/image-20200703102017853.png)

你们可以自行下载。

PS:如果下载了不能用的话（我的mac10.15.4 就不能用），可以自行clone项目，npm方式自运行。还有一点就是，作者使用mac的话状态栏是黑色的，所以它做的图标是白色透明的，而我的mac是白色的状态栏，就看不见图标，一开始我还以为程序报错了，clone下来之后运行，最后才发现是图标问题，改成黑色，就能完美看见了。

### BlogHelper修改版本
需要自行clone ; npm install ; npm start
[自行编译版本](https://github.com/fly7632785/BlogHelper)
- 兼容mac的浅色外观模式 图标可见（原来是透明的白色 不可见）
- 增加**一键存稿、发布** 博客到 已登录的写作平台



# 关于作者

作者是一个热爱学习、开源、分享，传播正能量，喜欢打篮球、头发还很多的程序员-。-

热烈欢迎大家关注、点赞、评论交流！

简书：https://www.jianshu.com/u/d234d1569eed

github：https://github.com/fly7632785

CSDN：https://blog.csdn.net/fly7632785

掘金：https://juejin.im/user/5efd8d205188252e58582dc7/posts