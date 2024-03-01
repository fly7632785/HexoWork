## 相关链接：
https://blog.csdn.net/qq853632587/article/details/111644295

## 相关释义
- 1、nacos可以有账号密码登录 也可以没有
- 2、命名空间：默认是public   配置了就会有相应的在nacos控制台展示
- 3、seata的注册中心和配置中心，有多种形式，默认是file，可以改成nocos；配置中心如果是nacos，需要上传config.txt上的配置到nacos上（注意 -t为命名空间的id不是名字）
- 4、seata默认有2个mysql的驱动 5.0  8.0 如果要使用8.0 
> 5.0： driverClassName = "com.mysql.jdbc.Driver"
>
> 8.0： driverClassName = "com.mysql.cj.jdbc.Driver" 
- 5、现在的情况是：seata的走nacos的配置中心，所以有seata的配置都走seata的命名空间下，其他照旧



# 1、修改上传seata的配置到配置中心

### 下载[seata](https://github.com/seata/seata/tree/develop/script/config-center)源码，然后进入script/conf-center，修改conf.txt的配置，再进入nacos目录，然后上传到nacos配置中心去

#### config.txt
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
service.vgroupMapping.my_test_tx_group=default
service.default.grouplist=192.168.31.168:8091
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
store.mode=db
store.publicKey=
store.file.dir=file_store/data
store.file.maxBranchSessionSize=16384
store.file.maxGlobalSessionSize=512
store.file.fileWriteBufferCacheSize=16384
store.file.flushDiskMode=async
store.file.sessionReloadReadSize=100
store.db.datasource=druid
store.db.dbType=mysql
store.db.driverClassName=com.mysql.cj.jdbc.Driver
store.db.url=jdbc:mysql://192.168.31.167:3306/seata?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai&useSSL=false
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

```
sh nacos-config.sh -h 192.168.31.167 -p 8848 -g SEATA_GROUP -t f2dd13e4-1886-4b27-b6d5-e8332875d7e5
```

> -h  配置中心url
> -p 配置中心端口
> -g 分组名
> -t 命名空间的id(不是名字是id)

### 创建事务日志数据库seata
进入seata源码，script/server/db下面，执行mysql.sql
有三张表global_table  branch_table lock_table
其次，所有的业务数据库也要添加undo_log表
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


# 2、docker部署seata

### 拉取镜像
```
docker pull seataio/seata-server:1.4.0
```




## seata server使用file的方式 (暂时不使用这种方式，忽略)
### file.conf默认配置
```
## transaction log store, only used in seata-server
store {
  ## store mode: file、db、redis
  mode = "db"

  ## file store property
  file {
    ## store location dir
    dir = "sessionStore"
    # branch session size , if exceeded first try compress lockkey, still exceeded throws exceptions
    maxBranchSessionSize = 16384
    # globe session size , if exceeded throws exceptions
    maxGlobalSessionSize = 512
    # file buffer size , if exceeded allocate new buffer
    fileWriteBufferCacheSize = 16384
    # when recover batch read size
    sessionReloadReadSize = 100
    # async, sync
    flushDiskMode = async
  }

  ## database store property
  db {
    ## the implement of javax.sql.DataSource, such as DruidDataSource(druid)/BasicDataSource(dbcp)/HikariDataSource(hikari) etc.
    datasource = "druid"
    ## mysql/oracle/postgresql/h2/oceanbase etc.
    dbType = "mysql"
    driverClassName = "com.mysql.cj.jdbc.Driver"
    url = "jdbc:mysql://192.168.31.167:3306/seata?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai"
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

  ## redis store property
  redis {
    host = "127.0.0.1"
    port = "6379"
    password = ""
    database = "0"
    minConn = 1
    maxConn = 10
    maxTotal = 100
    queryLimit = 100
  }

}
```


### registry.conf 默认配置
```
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "nacos" # 这里使用nacos注册中心，所以只修改nacos配置即可，下面的file配置也可以删掉

  nacos {
     application = "seata-server"
 	 serverAddr = "192.168.31.167:8848"
     group = "SEATA_GROUP"
 	 namespace = ""
    cluster = "default"
  }
}

config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "file" # 这里使用的file 配置，也就是本地file文件配置，nacos 也可以删掉
  file {
    name = "file.conf"
  }
}
```

```
docker run --name seata-server -d \
        -p 8091:8091 \
		-e SEATA_IP=192.168.31.168 \
		-e SEATA_PORT=8091 \
        -v /seata/conf/registry.conf:/seata-server/resources/registry.conf \
        -v /seata/conf/file.conf:/seata-server/resources/file.conf \
		-v /seata/logs:/root/logs \
      --privileged=true \
        seataio/seata-server:1.4.0
```
只能映射两个文件 registry.conf 和 file.conf 
不要使用 文件夹映射 -v /seata/conf:/seata-server/resources


### seata server使用nacos方式 （推荐使用）
```
registry {
  #seata 的注册中心使用nacos 还可以使用eureka等
  type = "nacos"
  nacos {
    #seata server名字
    application = "seata-server"
    #注册中心地址  现在是nacos
 	serverAddr = "192.168.31.167:8848"
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
    serverAddr = "192.168.31.167:8848"
    #配置中心命名空间 seata的命名空间id
    namespace = "f2dd13e4-1886-4b27-b6d5-e8332875d7e5"
    #分组名
    group = "SEATA_GROUP"
  }
}
```

```
docker run --name seata-server -d \
		-e SEATA_IP=192.168.31.168 \
		-e SEATA_PORT=8091 \
-v /seata/conf/registry.conf:/seata-server/resources/registry.conf \
		-v /seata/logs:/root/logs \
      --privileged=true \
      --net=host\
        seataio/seata-server:1.4.0
```

> -e SEATA_IP seata的被发现地址（也就是自身的ip）
> -e SEATA_PORT 端口
> -v /seata/conf/registry.conf:/seata-server/resources/registry.conf 映射本地文件到docker容器内
> --privileged=true 给予权限
> --net=host 使用host模式




# 3、客户端部署seata

#### pom.xml
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
#### .yaml
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
     server-addr: 192.168.31.167:8848 #注册中心nacos地址
     group: SEATA_GROUP #分组名 对应服务器端配置
     cluster: default #默认集群名
  config:
    type: nacos
    nacos:
      server-addr: 192.168.31.167:8848 #配置中心nacos地址
      group: SEATA_GROUP #分组名 对应服务器端配置
      namespace: f2dd13e4-1886-4b27-b6d5-e8332875d7e5  #命名空间 对应nacos中配置中心seata
  service:
    vgroup-mapping:
      my_test_tx_group: default #事务分组

```