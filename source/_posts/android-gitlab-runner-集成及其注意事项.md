---
title: android-gitlab-runner-集成及其注意事项
date: 2018-02-26 17:05:49
tags: android 
---
### 前言
如果你的公司使用的是gitlab，你完全可以做一个[gitlab runner](https://docs.gitlab.com/runner/)来节省很多自己打包给测试人员的时间。
它是基于gitlab ci的，关于[gitlab ci](https://about.gitlab.com/features/gitlab-ci-cd/)可以自行了解一下。
我现在使用的gitlab ci，说白了其实就是一个运行在电脑上的脚本，可以在每次提交代码的时候自动打包成apk。

### 效果
每次有提交，就会自动生成好apk供下载使用
![你可以直接点击下载apk了](http://upload-images.jianshu.io/upload_images/1311457-22debb91bf92d4ce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 正文
[官方安装手册](https://docs.gitlab.com/runner/install/osx.html)
我这里主要介绍mac的安装与集成，以及集成过程中遇到的问题和需要注意的地方。
- 1、下载gitlab runner 
```
sudo curl --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-darwin-amd64
```
- 2、修改运行权限
```
sudo chmod +x /usr/local/bin/gitlab-runner
```
注意：这里需要使用sudo来执行命令
- 3、注册runner
![注册runner](http://upload-images.jianshu.io/upload_images/1311457-774e49bb7623d2c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  - 1） 输出注册命令
  ```
  gitlab-runner register
  ```
  注意：这里不要看错了，mac和linux不一样，没有sudo！因为这里涉及用户问题。（这里需要使用普通用户，不要用root用户）
  - 2）输出url
  ```
  xxxxxx(你的url)
  ```
  - 3）输入token
  ```
  xxxxx(你的token)
  ```

  - 4）输入你的runner描述(后面可以修改的)
  ```
  Jafir's runner
  ```
  - 5） 输入你的tag名字
  ```
  Jafir
  ```
  - 6） 是否可以运行没有标记的jobs(最好选true)
   - 7） runner是否仅仅用于这个项目（因为有些runner是可以分享共用的）
  ```
  true
  ```
   - 8） 选择shell（这里我们选择shell，你也可以选择docker或者其他方式）
  ```
  shell
  ```
- 4、新建一个文件夹gitlabci（用于管理gitlab ci所需要运行的project，其实就是从gitlab clone下来的）
![gitlab runner管理的文件夹](http://upload-images.jianshu.io/upload_images/1311457-55f47ad3f3592055.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
cd gitlabci(管理文件夹)
```
```
gitlab-runner install
```
```
gitlab-runner start
```
- 5、生成.gitlab-ci.yml文件（脚本文件）
![可以直接点击这里](http://upload-images.jianshu.io/upload_images/1311457-6f9a8eee04327f30.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![.gitlab-ci.yml文件介绍](http://upload-images.jianshu.io/upload_images/1311457-690fb58cf6ec8951.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
before_script:
  - chmod +x ./gradlew

stages:
  - build

apk:
  stage: build
  only:
    - branches@bandai/p-bandai-Android
  script:
    - ./gradlew assembleRelease
    - ./gradlew assembleStaging
    - mv app/build/outputs/apk/*.apk .
  artifacts:
    name: "$CI_PIPELINE_ID APK"
    paths:
      - ./*.apk

```
这里你可以直接copy然后修改使用，如果还有其他的需求，也可以定制。这里是[定义规则](https://docs.gitlab.com/ce/ci/yaml/README.html)

注意：这里直接提交的话，是在主分支master的根目录上面。只针对master的分支有提交才会触发，如果其他分支也需要，那么也需要在其他分支上copy一份这个文件。

如果不出意外的话就ok了，可以从pipelines里面看到一些日志信息
![](http://upload-images.jianshu.io/upload_images/1311457-30db8f5909171e97.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 意外
1、出现 Fetching changes...然后就报错不动了
> 打开~/.rvmrc 或者 /etc/rvmrc
然后添加一句
rvm_project_rvmrc=0

2、如果你的bash有问题（有时候不排除版本问题，需要更新一下）
> 可以参照[这里](https://apple.stackexchange.com/questions/224511/how-to-use-bash-as-default-shell/232983#232983)，新安装一个bash来使用

3、出现New runner. Has not connected yet
>可以参照[这里](https://codereviewvideos.com/blog/how-i-solved-new-runner-has-not-connected-yet-in-gitlab-ci/)（很大程度上是跟是否使用sudo相关的，因为用户的不同会导致一些权限问题）

### 常用命令介绍
1、gitlab-runner --debug run，如果你遇到一些错误，可以使用这个命令来在前端（控制台运行），查看log
2、gitlab-runner run --user jafir(普通用户)，如果需要切换用户可以使用这个
3、sudo chmod -x xxx，修改用户权限
4、gitlab-runner uninstall，如果想从头再来
5、gitlab-runner status，查看状态
6、sudo gitlab-runner verify，查看runner是否在运行后
7、sudo gitlab-runner verify --delete，删除注册的用户，如果想要从头再来
8、删除 ~/.gitlab-runner/config.toml(注册的用户的配置文件)，和/etc/gitlab-runner/config.toml，如果想要从头再来