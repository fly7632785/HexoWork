# 前言

seata是阿里开源的分布式事务，这里我们来做seata的注册及其集群配置。

注册中心呢，我们这里用的也是阿里的nacos，nacos我们也是做了集群的。



# 环境

| 主机ip      | 部署                |
| ----------- | ------------------- |
| 172.16.9.45 | mysql seata1        |
| 172.16.9.46 | seata2              |
| 172.16.9.47 | seata3              |
| 172.16.9.48 | nacos注册、配置中心 |



# 修改上传seata的配置到配置中心

下载[seata](https://github.com/seata/seata/tree/develop/script/config-center)源码，然后进入script/conf-center，修改conf.txt的配置，再进入nacos目录，然后上传到nacos配置中心去

config.txt

```
transport.type=TCP
transport.server=NIO
transport.heartbeat=true
transport.enableClientBatchSendRequest=false
transport.threadFactory.bossThreadPrefix=NettyBoss
transport.threadFactory.workerThreadPrefix=NettyServerNIOWorker
transport.threadFactory.serverExecutorThreadPrefix=NettyServerBizHandler
transport.threadFactory.shareBossWorker=false
transport.threadFactory.clientSelectorThreadPrefix=NettyClientSelector
transport.threadFactory.clientSelectorThreadSize=1
transport.threadFactory.clientWorkerThreadPrefix=NettyClientWorkerThread
transport.threadFactory.bossThreadSize=1
transport.threadFactory.workerThreadSize=default
transport.shutdown.wait=3
# 这里的配置 springcloud代码客户端配置要一致
service.vgroupMapping.my_test_tx_group=default
# 这里仅注册中心为file时使用 此时无用 我们注册中心是nacos
service.default.grouplist=172.16.9.45:8091
service.enableDegrade=false
service.disableGlobalTransaction=false
client.rm.asyncCommitBufferLimit=10000
client.rm.lock.retryInterval=10
client.rm.lock.retryTimes=30
client.rm.lock.retryPolicyBranchRollbackOnConflict=true
client.rm.reportRetryCount=5
client.rm.tableMetaCheckEnable=false
client.rm.tableMetaCheckerInterval=60000
client.rm.sqlParserType=druid
client.rm.reportSuccessEnable=false
client.rm.sagaBranchRegisterEnable=false
client.tm.commitRetryCount=5
client.tm.rollbackRetryCount=5
client.tm.defaultGlobalTransactionTimeout=60000
client.tm.degradeCheck=false
client.tm.degradeCheckAllowTimes=10
client.tm.degradeCheckPeriod=2000
#存储类型为数据库
store.mode=db
store.publicKey=
store.file.dir=file_store/data
store.file.maxBranchSessionSize=16384
store.file.maxGlobalSessionSize=512
store.file.fileWriteBufferCacheSize=16384
store.file.flushDiskMode=async
store.file.sessionReloadReadSize=100
store.db.datasource=druid
#数据库mysql
store.db.dbType=mysql
#数据库驱动 我是8.0 mysql
store.db.driverClassName=com.mysql.cj.jdbc.Driver
#数据库配置
store.db.url=jdbc:mysql://172.16.9.45:3306/seata?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai&useSSL=false&rewriteBatchedStatements=true
store.db.user=root
store.db.password=asdw@123
store.db.minConn=5
store.db.maxConn=30
store.db.globalTable=global_table
store.db.branchTable=branch_table
store.db.queryLimit=100
store.db.lockTable=lock_table
store.db.maxWait=5000
store.redis.mode=single
store.redis.single.host=127.0.0.1
store.redis.single.port=6379
store.redis.maxConn=10
store.redis.minConn=1
store.redis.maxTotal=100
store.redis.database=0
store.redis.password=
store.redis.queryLimit=100
server.recovery.committingRetryPeriod=1000
server.recovery.asynCommittingRetryPeriod=1000
server.recovery.rollbackingRetryPeriod=1000
server.recovery.timeoutRetryPeriod=1000
server.maxCommitRetryTimeout=-1
server.maxRollbackRetryTimeout=-1
server.rollbackRetryTimeoutUnlockEnable=false
client.undo.dataValidation=true
client.undo.logSerialization=jackson
client.undo.onlyCareUpdateColumns=true
server.undo.logSaveDays=7
server.undo.logDeletePeriod=86400000
client.undo.logTable=undo_log
client.undo.compress.enable=true
client.undo.compress.type=zip
client.undo.compress.threshold=64k
log.exceptionRate=100
transport.serialization=seata
transport.compressor=none
metrics.enabled=false
metrics.registryType=compact
metrics.exporterList=prometheus
metrics.exporterPrometheusPort=9898

```

#### 注意：seata默认有2个mysql的驱动 5.0 8.0 ，驱动类不同

> 5.0： driverClassName = "com.mysql.jdbc.Driver"
>
> 8.0： driverClassName = "com.mysql.cj.jdbc.Driver"

### nacos中创建seata独立的命名空间

![image-20210223163325787](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20210223163325787.png)

好了之后，id是自动生成的UUID，后面会用到，包括客户端配置和上传配置中心。

进入 seata-develop/script/config-center/nacos 执行sh，上传配置到nacos配置中心（nacos既是注册中心又是配置中心）

```shell
sh nacos-config.sh -h 172.16.9.48 -p 8848 -g SEATA_GROUP -t f2dd13e4-1886-4b27-b6d5-e8332875d7e5
```

命令解读：

```
sh nacos-config.sh 
-h 172.16.9.48   配置中心的ip
-p 8848  配置中心的端口
-g SEATA_GROUP group名字
-t f2dd13e4-1886-4b27-b6d5-e8332875d7e5 命名空间的id，就是前面创建之后的id
```

上传成功后，nacos配置中心就有了相应配置。

![image-20210223163845924](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20210223163845924.png)



# 创建事务日志数据库seata

创建seata数据库

进入seata源码，script/server/db下面，数据库执行mysql.sql 会生成三张表global_table branch_table lock_table 。其次，所有的业务数据库也要添加undo_log表。

mysql.sql

```
-- -------------------------------- The script used when storeMode is 'db' --------------------------------
-- the table to store GlobalSession data
CREATE TABLE IF NOT EXISTS `global_table`
(
    `xid`                       VARCHAR(128) NOT NULL,
    `transaction_id`            BIGINT,
    `status`                    TINYINT      NOT NULL,
    `application_id`            VARCHAR(32),
    `transaction_service_group` VARCHAR(32),
    `transaction_name`          VARCHAR(128),
    `timeout`                   INT,
    `begin_time`                BIGINT,
    `application_data`          VARCHAR(2000),
    `gmt_create`                DATETIME,
    `gmt_modified`              DATETIME,
    PRIMARY KEY (`xid`),
    KEY `idx_gmt_modified_status` (`gmt_modified`, `status`),
    KEY `idx_transaction_id` (`transaction_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

-- the table to store BranchSession data
CREATE TABLE IF NOT EXISTS `branch_table`
(
    `branch_id`         BIGINT       NOT NULL,
    `xid`               VARCHAR(128) NOT NULL,
    `transaction_id`    BIGINT,
    `resource_group_id` VARCHAR(32),
    `resource_id`       VARCHAR(256),
    `branch_type`       VARCHAR(8),
    `status`            TINYINT,
    `client_id`         VARCHAR(64),
    `application_data`  VARCHAR(2000),
    `gmt_create`        DATETIME(6),
    `gmt_modified`      DATETIME(6),
    PRIMARY KEY (`branch_id`),
    KEY `idx_xid` (`xid`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

-- the table to store lock data
CREATE TABLE IF NOT EXISTS `lock_table`
(
    `row_key`        VARCHAR(128) NOT NULL,
    `xid`            VARCHAR(96),
    `transaction_id` BIGINT,
    `branch_id`      BIGINT       NOT NULL,
    `resource_id`    VARCHAR(256),
    `table_name`     VARCHAR(32),
    `pk`             VARCHAR(36),
    `gmt_create`     DATETIME,
    `gmt_modified`   DATETIME,
    PRIMARY KEY (`row_key`),
    KEY `idx_branch_id` (`branch_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

```

undo_log的sql

```
CREATE TABLE IF NOT EXISTS `undo_log`
(
   `id`            BIGINT(20)   NOT NULL AUTO_INCREMENT COMMENT 'increment id',
   `branch_id`     BIGINT(20)   NOT NULL COMMENT 'branch transaction id',
   `xid`           VARCHAR(100) NOT NULL COMMENT 'global transaction id',
   `context`       VARCHAR(128) NOT NULL COMMENT 'undo_log context,such as serialization',
   `rollback_info` LONGBLOB     NOT NULL COMMENT 'rollback info',
   `log_status`    INT(11)      NOT NULL COMMENT '0:normal status,1:defense status',
   `log_created`   DATETIME     NOT NULL COMMENT 'create datetime',
   `log_modified`  DATETIME     NOT NULL COMMENT 'modify datetime',
   PRIMARY KEY (`id`),
   UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
) ENGINE = InnoDB
 AUTO_INCREMENT = 1
 DEFAULT CHARSET = utf8 COMMENT ='AT transaction mode undo table';

```



# docker部署seata

### 拉取镜像

```
docker pull seataio/seata-server:1.4.0
```

这里使用的是1.4.0，nacos的版本也是1.4+



### seata server使用nacos方式注册，使用db方式存储配置

安装seata需要两个配置文件，file.conf、registry.conf，我是放在了/seata/conf/下面，三台机子都需要相应的配置

#### file.conf

```
## transaction log store, only used in seata-server
store {
  ## store mode: file、db、redis
  mode = "db"

  ## database store property
  db {
    ## the implement of javax.sql.DataSource, such as DruidDataSource(druid)/BasicDataSource(dbcp)/HikariDataSource(hikari) etc.
    datasource = "druid"
    ## mysql/oracle/postgresql/h2/oceanbase etc.
    dbType = "mysql"
    driverClassName = "com.mysql.cj.jdbc.Driver"
    url = "jdbc:mysql://172.16.9.45:3306/seata?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai"
    user = "root"
    password = "asdw@123"
    minConn = 5
    maxConn = 100
    globalTable = "global_table"
    branchTable = "branch_table"
    lockTable = "lock_table"
    queryLimit = 100
    maxWait = 5000
  }
}
```

#### registry.conf

```
registry {
  #seata 的注册中心使用nacos 还可以使用eureka等
  type = "nacos"
  nacos {
    #seata server名字
    application = "seata-server"
    #注册中心地址  现在是nacos
 	serverAddr = "172.16.9.48:8848"
    #分组名
   group = "SEATA_GROUP"
   #集群名称 默认default
   cluster = "default"
  }
}

config {
  #seata 的配置中心使用nacos 还可以使用file本地配置等
  type = "nacos"
  nacos {
    #配置中心地址  现在是nacos
    serverAddr = "172.16.9.48:8848"
    #配置中心命名空间 seata的命名空间id
    namespace = "f2dd13e4-1886-4b27-b6d5-e8332875d7e5"
    #分组名
    group = "SEATA_GROUP"
  }
}

```



## 172.16.9.45安装seata

```
docker run --name seata -d \
-e SEATA_IP=172.16.9.45 \
-e SEATA_PORT=8091 \
-v /root/seata/conf/registry.conf:/seata-server/resources/registry.conf \
-v /root/seata/logs:/root/logs \
--privileged=true \
-p 8091:8091 \
 seataio/seata-server:1.4.0
```

命令解读：

```
-e SEATA_IP seata的被发现地址（也就是自身的ip）
-e SEATA_PORT 端口
-v /seata/conf/registry.conf:/seata-server/resources/registry.conf 映射本地文件到docker容器内
--privileged=true 给予权限
--net=host 使用host模式
```



## 172.16.9.46安装seata

```
docker run --name seata -d \
-e SEATA_IP=172.16.9.46 \
-e SEATA_PORT=8091 \
-v /root/seata/conf/registry.conf:/seata-server/resources/registry.conf \
-v /root/seata/logs:/root/logs \
 --privileged=true \
-p 8091:8091 \
 seataio/seata-server:1.4.0
```



## 172.16.9.47安装seata

```
docker run --name seata -d \
-e SEATA_IP=172.16.9.47 \
-e SEATA_PORT=8091 \
-v /root/seata/conf/registry.conf:/seata-server/resources/registry.conf \
-v /root/seata/logs:/root/logs \
 --privileged=true \
-p 8091:8091 \
 seataio/seata-server:1.4.0
```

启动了之后，不出意外，就可以在nacos上看到3个实例了。

![image-20210223165335764](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20210223165335764.png)





# 客户端部署seata

maven引入依赖

```
<dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-seata</artifactId>
            <version>2.2.0.RELEASE</version>
            <exclusions>
                <exclusion>
                    <groupId>io.seata</groupId>
                    <artifactId>seata-spring-boot-starter</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <dependency>
            <groupId>io.seata</groupId>
            <artifactId>seata-spring-boot-starter</artifactId>
            <version>1.4.0</version>
        </dependency>

```

.yml配置文件

```
seata:
  enabled: true
  application-id: contract #微服务应用名称
  tx-service-group: my_test_tx_group    #此处配置自定义的seata事务分组名称
  enable-auto-data-source-proxy: true    #开启数据库代理
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
      namespace: f2dd13e4-1886-4b27-b6d5-e8332875d7e5  #命名空间 对应nacos中配置中心seata
  service:
    vgroup-mapping:
      my_test_tx_group: default #事务分组

```

启动成功

![image-20210223165652203](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20210223165652203.png)

```
@GlobalTransactional
```

使用注解就OK了





# 关于作者

作者是一个热爱学习、开源、分享，传播正能量，喜欢打篮球、头发还很多的程序员-。-

热烈欢迎大家关注、点赞、评论交流！

简书：https://www.jianshu.com/u/d234d1569eed

github：https://github.com/fly7632785

CSDN：https://blog.csdn.net/fly7632785

掘金：https://juejin.im/user/5efd8d205188252e58582dc7/posts