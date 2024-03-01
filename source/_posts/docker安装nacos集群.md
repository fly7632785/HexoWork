# 前言

nacos是一个很不错的注册中心及配置中心，现在用docker来搭建nacos高可用集群。

现在网上的资料不是很多且较为麻烦，github官网也是example的例子，不够全面，我也是踩了很多坑，最终找到了一个简易可行的方案，分享给大家。



# 说明

nacos官方因为nacos mysql的数据库主从模式有问题，1.3.1后已经废弃此方式，依旧只使用单一数据库。可参看[github中文说明](https://github.com/nacos-group/nacos-docker/blob/master/README_ZH.md)，且现在的mysql数据库配置一系列都为MYSQL_SERVICE_HOST，而不是原来的MYSQL_MASTER_XXXX 和 MYSQL_SLAVE_XXX

做nacos集群需要使用外部数据库，而非单机的自带嵌入式数据库，这里使用mysql。如果已经做了数据库的高可用，那么也是同理。



# 环境说明

| 主机ip      | 部署         |
| ----------- | ------------ |
| 172.16.9.45 | mysql nacos1 |
| 172.16.9.46 | nacos2       |
| 172.16.9.47 | nacos3       |
| 172.16.9.48 | nginx 做负载 |

这里mysql就只用了一台，没有高可用，大家可以自行调整。



# 数据库mysql

#### 装一个mysql

自建/mydockerdata/mysql/log   data 等目录，做数据存储的映射路径，路径可自行调整

```
docker run --name mysql -p 3306:3306 -v /mydockerdata/mysql/data:/var/lib/mysql -v /mydockerdata/mysql/log:/var/log/mysql -e MYSQL_ROOT_PASSWORD=asdw@123 -d mysql:5.7.0
```

数据库密码等自行设置，后面安装nacos配置时一致即可。

创建好了之后，创建数据库nacos_config，插入nacos的配置相关的sql，这里给出

```sql
/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_info   */
/******************************************/
CREATE TABLE `config_info` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(255) DEFAULT NULL,
  `content` longtext NOT NULL COMMENT 'content',
  `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
  `gmt_create` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '修改时间',
  `src_user` text COMMENT 'source user',
  `src_ip` varchar(20) DEFAULT NULL COMMENT 'source ip',
  `app_name` varchar(128) DEFAULT NULL,
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  `c_desc` varchar(256) DEFAULT NULL,
  `c_use` varchar(64) DEFAULT NULL,
  `effect` varchar(64) DEFAULT NULL,
  `type` varchar(64) DEFAULT NULL,
  `c_schema` text,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfo_datagrouptenant` (`data_id`,`group_id`,`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_info_aggr   */
/******************************************/
CREATE TABLE `config_info_aggr` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(255) NOT NULL COMMENT 'group_id',
  `datum_id` varchar(255) NOT NULL COMMENT 'datum_id',
  `content` longtext NOT NULL COMMENT '内容',
  `gmt_modified` datetime NOT NULL COMMENT '修改时间',
  `app_name` varchar(128) DEFAULT NULL,
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfoaggr_datagrouptenantdatum` (`data_id`,`group_id`,`tenant_id`,`datum_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='增加租户字段';


/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_info_beta   */
/******************************************/
CREATE TABLE `config_info_beta` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) NOT NULL COMMENT 'group_id',
  `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
  `content` longtext NOT NULL COMMENT 'content',
  `beta_ips` varchar(1024) DEFAULT NULL COMMENT 'betaIps',
  `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
  `gmt_create` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '修改时间',
  `src_user` text COMMENT 'source user',
  `src_ip` varchar(20) DEFAULT NULL COMMENT 'source ip',
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfobeta_datagrouptenant` (`data_id`,`group_id`,`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info_beta';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_info_tag   */
/******************************************/
CREATE TABLE `config_info_tag` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) NOT NULL COMMENT 'group_id',
  `tenant_id` varchar(128) DEFAULT '' COMMENT 'tenant_id',
  `tag_id` varchar(128) NOT NULL COMMENT 'tag_id',
  `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
  `content` longtext NOT NULL COMMENT 'content',
  `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
  `gmt_create` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '修改时间',
  `src_user` text COMMENT 'source user',
  `src_ip` varchar(20) DEFAULT NULL COMMENT 'source ip',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfotag_datagrouptenanttag` (`data_id`,`group_id`,`tenant_id`,`tag_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info_tag';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_tags_relation   */
/******************************************/
CREATE TABLE `config_tags_relation` (
  `id` bigint(20) NOT NULL COMMENT 'id',
  `tag_name` varchar(128) NOT NULL COMMENT 'tag_name',
  `tag_type` varchar(64) DEFAULT NULL COMMENT 'tag_type',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) NOT NULL COMMENT 'group_id',
  `tenant_id` varchar(128) DEFAULT '' COMMENT 'tenant_id',
  `nid` bigint(20) NOT NULL AUTO_INCREMENT,
  PRIMARY KEY (`nid`),
  UNIQUE KEY `uk_configtagrelation_configidtag` (`id`,`tag_name`,`tag_type`),
  KEY `idx_tenant_id` (`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_tag_relation';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = group_capacity   */
/******************************************/
CREATE TABLE `group_capacity` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `group_id` varchar(128) NOT NULL DEFAULT '' COMMENT 'Group ID，空字符表示整个集群',
  `quota` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '配额，0表示使用默认值',
  `usage` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '使用量',
  `max_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个配置大小上限，单位为字节，0表示使用默认值',
  `max_aggr_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '聚合子配置最大个数，，0表示使用默认值',
  `max_aggr_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个聚合数据的子配置大小上限，单位为字节，0表示使用默认值',
  `max_history_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '最大变更历史数量',
  `gmt_create` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '修改时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_group_id` (`group_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='集群、各Group容量信息表';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = his_config_info   */
/******************************************/
CREATE TABLE `his_config_info` (
  `id` bigint(64) unsigned NOT NULL,
  `nid` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `data_id` varchar(255) NOT NULL,
  `group_id` varchar(128) NOT NULL,
  `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
  `content` longtext NOT NULL,
  `md5` varchar(32) DEFAULT NULL,
  `gmt_create` datetime NOT NULL DEFAULT '2010-05-05 00:00:00',
  `gmt_modified` datetime NOT NULL DEFAULT '2010-05-05 00:00:00',
  `src_user` text,
  `src_ip` varchar(20) DEFAULT NULL,
  `op_type` char(10) DEFAULT NULL,
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  PRIMARY KEY (`nid`),
  KEY `idx_gmt_create` (`gmt_create`),
  KEY `idx_gmt_modified` (`gmt_modified`),
  KEY `idx_did` (`data_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='多租户改造';


/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = tenant_capacity   */
/******************************************/
CREATE TABLE `tenant_capacity` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `tenant_id` varchar(128) NOT NULL DEFAULT '' COMMENT 'Tenant ID',
  `quota` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '配额，0表示使用默认值',
  `usage` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '使用量',
  `max_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个配置大小上限，单位为字节，0表示使用默认值',
  `max_aggr_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '聚合子配置最大个数',
  `max_aggr_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个聚合数据的子配置大小上限，单位为字节，0表示使用默认值',
  `max_history_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '最大变更历史数量',
  `gmt_create` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '修改时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tenant_id` (`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='租户容量信息表';


CREATE TABLE `tenant_info` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `kp` varchar(128) NOT NULL COMMENT 'kp',
  `tenant_id` varchar(128) default '' COMMENT 'tenant_id',
  `tenant_name` varchar(128) default '' COMMENT 'tenant_name',
  `tenant_desc` varchar(256) DEFAULT NULL COMMENT 'tenant_desc',
  `create_source` varchar(32) DEFAULT NULL COMMENT 'create_source',
  `gmt_create` bigint(20) NOT NULL COMMENT '创建时间',
  `gmt_modified` bigint(20) NOT NULL COMMENT '修改时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tenant_info_kptenantid` (`kp`,`tenant_id`),
  KEY `idx_tenant_id` (`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='tenant_info';

CREATE TABLE users (
	username varchar(50) NOT NULL PRIMARY KEY,
	password varchar(500) NOT NULL,
	enabled boolean NOT NULL
);

CREATE TABLE roles (
	username varchar(50) NOT NULL,
	role varchar(50) NOT NULL
);
CREATE TABLE `permissions` (
    `role` varchar(50) NOT NULL,
    `resource` varchar(255) NOT NULL,
    `action` varchar(8) NOT NULL,
    UNIQUE INDEX `uk_role_permission` (`role`,`resource`,`action`) USING BTREE
);
INSERT INTO users (username, password, enabled) VALUES ('nacos', '$2a$10$EuWPZHzz32dJN7jexM34MOeYirDdFAZm2kuWj7VEOJhhZkDrxfvUu', TRUE);

INSERT INTO roles (username, role) VALUES ('nacos', 'ROLE_ADMIN');
```



# nacos集群

自建/root/nacos/config logs 等目录，做路径映射

/root/nacos/config下增加一个custom.properties文件

```
vi /root/nacos/config/custom.properties
```

内容为

```
#spring.security.enabled=false
#management.security=false
#security.basic.enabled=false
#nacos.security.ignore.urls=/**
#management.metrics.export.elastic.host=http://localhost:9200
# metrics for prometheus
management.endpoints.web.exposure.include=*

# metrics for elastic search
#management.metrics.export.elastic.enabled=false
#management.metrics.export.elastic.host=http://localhost:9200

# metrics for influx
#management.metrics.export.influx.enabled=false
#management.metrics.export.influx.db=springboot
#management.metrics.export.influx.uri=http://localhost:8086
#management.metrics.export.influx.auto-create-db=true
#management.metrics.export.influx.consistency=one
#management.metrics.export.influx.compressed=true
```

其他都注释掉了，包括一些监控等，以后需要可以自行修改配置

## 各个主机配置nacos

#### 172.16.9.45这台nacos1

```shell
docker run -d --name nacos1 --hostname nacos1 --net=host \
--add-host nacos1:172.16.9.45 \
--add-host nacos2:172.16.9.46 \
--add-host nacos3:172.16.9.47 \
-e PREFER_HOST_MODE=hostname \
-e MYSQL_SERVICE_HOST=172.16.9.45 \
-e MYSQL_SERVICE_DB_NAME=nacos_config \
-e MYSQL_SERVICE_USER=root \
-e MYSQL_SERVICE_PASSWORD=asdw@123 \
-e MYSQL_SERVICE_PORT=3306 \
-e NACOS_SERVERS="nacos1:8848 nacos2:8848 nacos3:8848" \
-v /root/nacos/config/custom.properties:/home/nacos/init.d/custom.properties \
-v /root/nacos/logs:/home/nacos/logs \
nacos/nacos-server:1.4.1
```

命令解读：

这里使用host的模式

```shell
docker run -d --name nacos1 --hostname nacos1 \
--net=host \  这里很重要，我也是踩了坑，网上很多都是直接-p 8848:8848，而这里为--net=host主机模式
--add-host nacos1:172.16.9.45 \ 添加host
--add-host nacos2:172.16.9.46 \
--add-host nacos3:172.16.9.47 \
-e PREFER_HOST_MODE=hostname \ 模式为host
-e MYSQL_SERVICE_HOST=172.16.9.45 \ 数据库的ip
-e MYSQL_SERVICE_DB_NAME=nacos_config \ 数据库名
-e MYSQL_SERVICE_USER=root \
-e MYSQL_SERVICE_PASSWORD=asdw@123 \ 数据库密码
-e MYSQL_SERVICE_PORT=3306 \ 数据库端口
-e NACOS_SERVERS="nacos1:8848 nacos2:8848 nacos3:8848" \  集群nacos的host及其端口
-v /root/nacos/config/custom.properties:/home/nacos/init.d/custom.properties \ 配置文件映射
-v /root/nacos/logs:/home/nacos/logs \ 日志映射
nacos/nacos-server:1.4.1
```

这里重点解释一下 --net=host，这里是采用docker的主机模式，也就是无网桥，直接占用本机的端口。网上很多人的例子都是在一台机子下面做多个不同端口的nacos，来模拟伪集群，所以很多人都是-p 8848:8848就OK了。而在不同机子上时，nacos其实不仅仅使用到了8848端口，还用了其他一些端口，比如7848等。所以，要么你就要暴露所有的端口，要么简单点直接使用--net=host即可

#### 172.16.9.45这台nacos2

```shell
docker run -d --name nacos2 --hostname nacos2 --net=host \
--add-host nacos1:172.16.9.45 \
--add-host nacos2:172.16.9.46 \
--add-host nacos3:172.16.9.47 \
-e MYSQL_SERVICE_HOST=172.16.9.45 \
-e MYSQL_SERVICE_DB_NAME=nacos_config \
-e MYSQL_SERVICE_USER=root \
-e MYSQL_SERVICE_PASSWORD=asdw@123 \
-e MYSQL_SERVICE_PORT=3306 \
-e PREFER_HOST_MODE=hostname \
-e NACOS_SERVERS="nacos1:8848 nacos2:8848 nacos3:8848" \
-v /root/nacos/config/custom.properties:/home/nacos/init.d/custom.properties \
-v /root/nacos/logs:/home/nacos/logs \
nacos/nacos-server:1.4.1
```



#### 172.16.9.45这台nacos3

```shell
docker run -d --name nacos3 --hostname nacos3 --net=host \
--add-host nacos1:172.16.9.45 \
--add-host nacos2:172.16.9.46 \
--add-host nacos3:172.16.9.47 \
-e MYSQL_SERVICE_HOST=172.16.9.45 \
-e MYSQL_SERVICE_DB_NAME=nacos_config \
-e MYSQL_SERVICE_USER=root \
-e MYSQL_SERVICE_PASSWORD=asdw@123 \
-e MYSQL_SERVICE_PORT=3306 \
-e PREFER_HOST_MODE=hostname \
-e PREFER_HOST_MODE=hostname \
-e NACOS_SERVERS="nacos1:8848 nacos2:8848 nacos3:8848" \
-v /root/nacos/config/custom.properties:/home/nacos/init.d/custom.properties \
-v /root/nacos/logs:/home/nacos/logs \
nacos/nacos-server:1.4.1
```

#### 完成

ok! 不出意外集群就搭建好了。

访问：http://172.16.9.45:8488/nacos  http://172.16.9.46:8488/nacos  http://172.16.9.47:8488/nacos 任一都可以看到集群节点

默认用户密码：nacos/nacos

![image-20210223152702258](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20210223152702258.png)



# Nginx做高可用负载

nginx我是放在172.16.9.48上面的

如果没有nginx.conf配置文件，可以先随便建一个，然后把配置考出来

自建/nginx/log  /nginx/etc/nginx.conf等

```
docker run -d -name nginx nginx
docker cp nginx:/etc/nginx/nginx.conf  拷贝出来
docker rm -f nginx
```

##### nginx.conf的配置修改

在最后一行添加，也就是和http同级

```
stream {
    upstream nacos {
        server 172.16.9.45:8848;
        server 172.16.9.46:8848;
        server 172.16.9.47:8848;
    }
    server {
        listen 8848;
        proxy_pass nacos;
    }
}
```

```shell
docker run --net=host  --name nginx -v /nginx/log/:/var/log/nginx -v /nginx/etc/nginx.conf:/etc/nginx/nginx.conf -d nginx
```

然后访问http://172.16.9.48:8848/nacos就不出意外的可以进入到相应的控制台界面了。



### ps

遇到问题也不要害怕，网上可以搜，如果搜不到，就需要自行排查分析。

/nacos/logs下面有很多log

![image-20210223153835864](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20210223153835864.png)

最主要分析几个，alipay-jraft.log  naming-raft.log   naming-server.log   nacos.log  start.out



 



# 关于作者

作者是一个热爱学习、开源、分享，传播正能量，喜欢打篮球、头发还很多的程序员-。-

热烈欢迎大家关注、点赞、评论交流！

简书：https://www.jianshu.com/u/d234d1569eed

github：https://github.com/fly7632785

CSDN：https://blog.csdn.net/fly7632785

掘金：https://juejin.im/user/5efd8d205188252e58582dc7/posts