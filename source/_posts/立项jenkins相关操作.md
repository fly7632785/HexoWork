# 一、阅读wiki上的文档

doc项目的wiki，所有的文档都应浏览阅读



# 二、gitlab建仓库，分支

#### 由仓库管理人员建库，命名为微服务的业务名字，比如project



### 建分支，包括develop  test  release  master 



### 设置分支保护，只有develop为开发者developer，其他都是maintainer

![image-20210423105343896](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423105343896.png)



# 三、拷贝base框架做修改

### 1、clone Base项目，clone 微服务空仓库，拷贝过去

两个pom.xml右键设置为maven project



### 2、修改module名字

![image-20210423164724597](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423164724597.png)

命名为xxx-interface   xxxx-service



### 3、修改包名

### 4、修改interface模块的pom.xml

修改artifactId

![image-20210423164931329](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423164931329.png)

发布一下依赖，发布前IDE配置maven 依赖 setting.xml的私服server等

![image-20210423164946497](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423164946497.png)





### 5、修改service模块的pom.xml

修改artifactId  修改依赖xxxx-interface



##### pom.xml设置为maven project



### 6、修改启动类名

### 7、修改swagger配置类 title等

### 8、修改application.yml

端口号

application-name

数据库连接url

### 9、修改dockerfile

![image-20210423165313339](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423165313339.png)



### 10、clone mybatisplus-generator到根目录

![image-20210423170122476](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423170122476.png)



修改启动类配置

测试是否能生成代码模板（在数据库有数据的前提下）



生成了之后，xxxDTO，和xxxVO是没有的，此时可以直接复制 Person 为PersonDTO 和 PersonVO 到相应包下面，即可运行测试。







# 四、tapd建项目，配置流水线

### 配置gitlab 的webhook

把url token在所有微服务项目（包括网关，只要有git仓库的）都配置webhook

![image-20210423104952722](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423104952722.png)

![image-20210423105023123](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423105023123.png)

![image-20210423105035932](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423105035932.png)



### 配置jenkins

![image-20210423105231118](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423105231118.png)





# 五、jenkins建任务

### 构建多环境的微服务构建任务 和 回滚任务

在jenkins中**新建任务**，先切换到对应的视图（前端、后端、测试等），再新建任务

![image-20210423105503688](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423105503688.png)



#### 1、新建复制任务

命名为   微服务模块名-环境，例如：project-测试环境

![image-20210423105706706](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423105706706.png)

可以复制现在已有的任务。



#### 2、进入任务设置，关联tapd的id

![image-20210423105816213](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423105816213.png)

![image-20210423105837361](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423105837361.png)



#### 3、修改仓库地址

![image-20210423105913686](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423105913686.png)





#### 4、修改微服务名字

![image-20210423105937387](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423105937387.png)

注意这个名字要跟你项目工程里面的路径一致：如project-service



#### 5、设置gitlab的webhook

拷贝url 和 token（在高级选项里面），例如http://192.168.148.236:8040/project/project-%E5%BC%80%E5%8F%91%E7%8E%AF%E5%A2%83

![image-20210423110038472](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423110038472.png)



![image-20210423110105923](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423110105923.png)

**注意这里的一个filter branches by name 就是区分 触发的分支的**



![image-20210423110144658](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423110144658.png)



![image-20210423111501778](https://gitee.com/jafir/blogs/raw/master/2021/images/image-20210423111501778.png)

大概有这几个webhook



### 回滚任务

回滚任务几乎也是一样，拷贝project-回滚，然后修改微服务名



### 构建多环境的前端构建任务 和 回滚任务

拷贝web-back-开发环境任务，然后修改webhook、项目名、git地址等



### 构建多环境网关构建任务

拷贝web-back-开发环境任务，然后修改webhook、项目名、git地址等



### 构建jmeter接口测试任务

拷贝jmeter接口测试任务，然后修改git地址等



### 测试git提交develop分支触发构建

