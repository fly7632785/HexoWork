# 利用maven和spring来实现多套环境配置

# 配置说明
## 1、创建多个配置文件，dev（本地）、test（测试）、prod（生产）
![1](/uploads/f054a226b3dbb2a32fbfb4c327c014f0/1.png)
说明：bootstrap优先于application被应用，后者可以覆盖前者。application是公共的，application-xx是相应环境的特殊配置
逻辑原理：通过spring的profile的active来动态切换使用哪套配置

dev
```
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 192.168.31.167:8848
        register-enabled: false

```

test
```
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 192.168.31.167:8848
        register-enabled: true

```

prod
```
spring:
  datasource:
    username: srig
    password: asdw@123
#   这里使用了p6spy 如果没有使用 即使用原生mysql
    url: jdbc:p6spy:mysql://rm-bp1u46a5m077774ox.mysql.rds.aliyuncs.com:3306/dcmp_contract?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai
    driver-class-name: com.p6spy.engine.spy.P6SpyDriver
  cloud:
    cloud:
      txc:
        # 不同微服务名字不同        
        txcAppName: contractprovider
        txcServerGroup: trans_test.1585500393834815.HZ
        accessKey: LTAI4GEHNrCLzDLxBHM94xUx
        secretKey: nkiku7fPhpTNtmQcwoJFmlGOdas3BW
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        register-enabled: true


```

## 2、通过maven的profile来创建环境变量参数，用于动态修改spring的profiles.active
![image](/uploads/90d702ef8cf8c46d31e8c7b732e371d3/image.png)

## 3、maven配置多个profile
![image](/uploads/5dc3ee7d559c3b43b2314c0b3b1d8a8d/image.png)

```
<!--  通过maven的profile来 动态修改spring的profile 达到多套配置切换使用的效果  -->
    <profiles>
        <profile>
            <!-- 本地开发环境 -->
            <id>dev</id>
            <properties>
                <profiles.active>dev</profiles.active>
                <skipDocker>true</skipDocker>
            </properties>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
        </profile>
        <profile>
            <!-- 测试环境 -->
            <id>test</id>
            <properties>
                <profiles.active>test</profiles.active>
                <skipDocker>false</skipDocker>
            </properties>
        </profile>
        <profile>
            <!-- 生产环境 -->
            <id>prod</id>
            <properties>
                <profiles.active>prod</profiles.active>
                <skipDocker>true</skipDocker>
            </properties>
        </profile>
    </profiles>
```

在build的docker配置中 设置skipDocker参数
```
<skipDocker>${skipDocker}</skipDocker>
```
![image](/uploads/c120d927a83bb3d594a7e1c66eba09ae/image.png)

## 4、配置发布到测试环境
增加 `-Ptest` 参数 可以给启动器起名 发布到测试环境
![image](/uploads/c7defcc3f4c9acf8db08331f71419d8a/image.png)
![image](/uploads/92d7340ea158fced0c9415964aee7c77/image.png)
![image](/uploads/d9c729ce61ccd61ea6b504cff7bc4285/image.png)


## 5、配置发布到aliyun生产环境
增加 `-Pprod` 参数  可以给启动器起名 发布到生产环境
![image](/uploads/56406d8eff19538c0da2df98d5f7c47e/image.png)

## 6、可以给启动器起个名
![image](/uploads/f762e56cd79862bb80690f971a3edfe0/image.png)