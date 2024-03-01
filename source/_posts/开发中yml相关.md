# 主配置文件（application.yml）
需要修改以下地方
- server-port
- application-name
- datasource-dynamic-primary
- datasource-dynamic-primary-datasource

``` 
server:
  port: 3001
spring:
  # 环境 dev|test|prod 默认dev
  profiles:
    # 利用maven的profile来配置spring的profile
    active: @profiles.active@
  # jackson时间格式化
  jackson:
    time-zone: GMT+8
    date-format: yyyy-MM-dd HH:mm:ss
  main:
    allow-bean-definition-overriding: true
  application:
    name: app-ranking-project
  datasource:
    dynamic:
      primary: project #设置默认的数据源或者数据源组,和下面的datasource对应
      strict: false #设置严格模式,默认false不启动. 启动后在未匹配到指定数据源时候会抛出异常,不启动则使用默认数据源.
      datasource:    #对应不同数据库，具体参考官方文档
        project:
          username: root
          password: xiaoxi
          url: jdbc:p6spy:mysql://192.168.148.235:3306/project?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai
          driver-class-name: com.p6spy.engine.spy.P6SpyDriver
  cloud:
    sentinel:
      #绑定sentinel
      transport:
        dashboard: http://192.168.148.237:8858
        #通过开启另一端口号进行监听
        port: 8859
#      datasource:
#        #配置sentinel的持久化
#        ds1:
#          nacos:
#            server-addr: 192.168.148.236:8848
#            dataId: ${spring.application.name}
#            groupId: DEFAULT_GROUP
#            data_type: json
#            rule_type: flow


ribbon:
  OkToRetryOnAllOperations: false #对所有操作请求都进行重试,默认false
  ReadTimeout: 30000   #负载均衡超时时间，默认值5000
  ConnectTimeout: 30000 #ribbon请求连接的超时时间，默认值2000
  MaxAutoRetries: 0     #对当前实例的重试次数，默认0
  MaxAutoRetriesNextServer: 1 #对切换实例的重试次数，默认1
knife4j:
  enable: true
  setting:
    enableSwaggerModels: true
mybatis-plus:
  mapper-locations: classpath*:mapper/**/*.xml
  global-config:
    db-config:
      id-type: auto
#      sql日志（太繁杂不想用）
#  configuration:
#    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
oss:
  endpoint: http://oss-cn-hangzhou.aliyuncs.com
  accessKeyId: LTAI4GEHNrCLzDLxBHM94xUx
  accessKeySecret: nkiku7fPhpTNtmQcwoJFmlGOdas3BW
  bucketName: dcmp
  domain: https://dcmp.oss-cn-hangzhou.aliyuncs.com
  prefix: invest
##rabbitmq相关配置
#rabbitmq:
#    onsAddr:      # TCP 接入域名
#    accessKey:    # AccessKey
#    secretKey:    # SecretKey
#    groupId:
#    topic:
#    timeout: 3000         #超时时间
```
# 本地开发环境（application-local）
本地调试无需注册到服务器上，同时分布式锁也没必要使用
```
seata:
  enabled: false
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 192.168.148.236:8848
        register-enabled: false

```
# 开发环境（application-dev）

seata的数据库代理一定是false，因为使用的是多数据源配置，否则事务锁不生效

```
seata:
  enabled: true
  tx-service-group: my_test_tx_group    #此处配置自定义的seata事务分组名称
  enable-auto-data-source-proxy: false    #开启数据库代理
  registry:
    type: nacos
    nacos:
      application: seata-server
      server-addr: 192.168.148.236:8848 #注册中心nacos地址
      group: SEATA_GROUP #分组名 对应服务器端配置
      cluster: default #默认集群名
      namespace: 0fad5534-5f76-47fb-87d1-b28f13cbfcc9
  config:
    type: nacos
    nacos:
      server-addr: 192.168.148.236:8848 #配置中心nacos地址
      group: SEATA_GROUP #分组名 对应服务器端配置
      namespace: 0fad5534-5f76-47fb-87d1-b28f13cbfcc9  #命名空间 对应nacos中配置中心seata
  service:
    vgroup-mapping:
      my_test_tx_group: default #事务分组
  client.rm.lock:
    retryInterval: 20   #校验或占用全局锁重试间隔    默认10，单位毫秒
    retryTimes: 60   #校验或占用全局锁重试次数    默认30
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 192.168.148.236:8848
        register-enabled: true


```

# 测试环境（application-test）

seata的数据库代理一定是false，因为使用的是多数据源配置，否则事务锁不生效

```
seata:
  enabled: true
  tx-service-group: my_test_tx_group    #此处配置自定义的seata事务分组名称
  enable-auto-data-source-proxy: true    #开启数据库代理
  registry:
    type: nacos
    nacos:
      application: seata-server
      server-addr: 192.168.148.236:8848 #注册中心nacos地址
      group: SEATA_GROUP #分组名 对应服务器端配置
      cluster: default #默认集群名
      namespace: 0fad5534-5f76-47fb-87d1-b28f13cbfcc9
  config:
    type: nacos
    nacos:
      server-addr: 192.168.148.236:8848 #配置中心nacos地址
      group: SEATA_GROUP #分组名 对应服务器端配置
      namespace: 0fad5534-5f76-47fb-87d1-b28f13cbfcc9  #命名空间 对应nacos中配置中心seata
  service:
    vgroup-mapping:
      my_test_tx_group: default #事务分组
  client.rm.lock:
    retryInterval: 20   #校验或占用全局锁重试间隔    默认10，单位毫秒
    retryTimes: 60   #校验或占用全局锁重试次数    默认30
spring:
  datasource:
    dynamic:
      primary: project #设置默认的数据源或者数据源组,默认值即为master
      strict: false #设置严格模式,默认false不启动. 启动后在未匹配到指定数据源时候会抛出异常,不启动则使用默认数据源.
      datasource:
        project:
          username: root
          password: xiaoxi
          url: jdbc:p6spy:mysql://192.168.148.235:3307/project?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai
          driver-class-name: com.p6spy.engine.spy.P6SpyDriver
  cloud:
    nacos:
      discovery:
        server-addr: 192.168.148.236:8848
        register-enabled: true
        group: TEST_GROUP
        namespace: 374d1cad-c024-41b9-9885-9a7fd2585c33

```
# 生产环境（application-prod）

seata的数据库代理一定是false，因为使用的是多数据源配置，否则事务锁不生效

```
seata:
  enabled: true
  tx-service-group: my_test_tx_group    #此处配置自定义的seata事务分组名称
  enable-auto-data-source-proxy: false    #开启数据库代理
  registry:
    type: nacos
    nacos:
      application: seata-server
      server-addr: 172.16.9.48:8848 #注册中心nacos地址
      group: SEATA_GROUP #分组名 对应服务器端配置
      cluster: default #默认集群名
  config:
    type: nacos
    nacos:
      server-addr: 172.16.9.48:8848 #配置中心nacos地址
      group: SEATA_GROUP #分组名 对应服务器端配置
      namespace: 0fad5534-5f76-47fb-87d1-b28f13cbfcc9  #命名空间 对应nacos中配置中心seata
  service:
    vgroup-mapping:
      my_test_tx_group: default #事务分组
  client.rm.lock:
    retryInterval: 20   #校验或占用全局锁重试间隔    默认10，单位毫秒
    retryTimes: 60   #校验或占用全局锁重试次数    默认30
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 172.16.9.48:8848
        register-enabled: true
  datasource:
    username: root
    password: xiaoxi
    url: jdbc:p6spy:mysql://172.16.9.48:3306/dcmp_contract?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai
    driver-class-name: com.p6spy.engine.spy.P6SpyDriver

```