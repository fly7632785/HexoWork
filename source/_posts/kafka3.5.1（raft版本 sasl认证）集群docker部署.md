
Kafka分布式消息队列集群，kafka的三个节点分别坐落在三台主机上。

###  主机

192.168.59.20 

192.168.59.21  

192.168.59.22  

### 部署kafka集群 kraft模式

目前使用raft模式部署，不需要再依赖zookeeper；**采用sasl认证，传输不加密**。（如果非敏感数据可以这样用，如果敏感数据还是建议  sasl_ssl 传输也加密）

```
mkdir -p /data/kafka_raft
chmod g+w /data/kafka_raft
```

命令说明

```
docker run --detach \
  --net=host \
  --name kafka1 \
  --restart always \
  --volume /data/kafka_raft:/bitnami/kafka \
  --env TZ=Asia/Shanghai \
  --env KAFKA_CFG_PROCESS_ROLES=broker,controller  \ #声明角色 有了这个代表使用raft模式
  --env BITNAMI_DEBUG=true  \ #控制台打印日志
  --env ALLOW_PLAINTEXT_LISTENER=no  \ #生产环境不允许
  --env KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER  \ #控制器名称 对应下面CONTROLLER://:9093
  --env KAFKA_CFG_NUM_PARTITIONS=1 \ #默认分区数
  --env KAFKA_CFG_LISTENERS=INTERNAL://:9094,CLIENT://:9095,CONTROLLER://:9093,EXTERNAL://:9092 \ #监听器的地址和端口
  --env KAFKA_CFG_ADVERTISED_LISTENERS=INTERNAL://192.168.59.20:9094,CLIENT://:9095,EXTERNAL://192.168.59.20:9092  \ #发布监听器的地址和端口
  --env KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=INTERNAL:SASL_PLAINTEXT,CLIENT:SASL_PLAINTEXT,CONTROLLER:PLAINTEXT,EXTERNAL:SASL_PLAINTEXT  \ #监听器的协议 这里sasl_plain表示   仅认证加密 传输不加密
  --env KAFKA_CFG_INTER_BROKER_LISTENER_NAME=INTERNAL \ #内部broker名称
  --env KAFKA_CFG_SASL_MECHANISM_INTER_BROKER_PROTOCOL=PLAIN \  #协议
  --env KAFKA_CFG_SASL_ENABLED_MECHANISMS=PLAIN \ #协议
  --env KAFKA_CLIENT_USERS=vpn \ #加密客户端账号
  --env KAFKA_CLIENT_PASSWORDS=BCfoV77N \ #加密客户端密码
  --env KAFKA_INTER_BROKER_USER=vpn \ #内部broker之间通信账号
  --env KAFKA_INTER_BROKER_PASSWORD=BCfoV77N \ #内部broker之间通信密码
  --env KAFKA_CFG_NODE_ID=1 \ #节点ID 唯一不一样
  --env KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@192.168.59.20:9093,2@192.168.59.21:9093,3@192.168.59.22:9093 \ #投票选举列表
  --env KAFKA_KRAFT_CLUSTER_ID=abcdefghijklmnopqrstuv  \ #集群id 大家都一样
  --env KAFKA_HEAP_OPTS='-Xms512M -Xmx4G' \ #运行内存参数
  bitnami/kafka:3.5.1
```

### 命令

```
docker run --detach \
  --net=host \
  --name kafka1 \
  --restart always \
  --volume /data/kafka_raft:/bitnami/kafka \
  --env TZ=Asia/Shanghai \
  --env KAFKA_CFG_PROCESS_ROLES=broker,controller  \
  --env BITNAMI_DEBUG=true  \
  --env ALLOW_PLAINTEXT_LISTENER=no  \
  --env KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER  \
  --env KAFKA_CFG_NUM_PARTITIONS=1 \
  --env KAFKA_CFG_LISTENERS=INTERNAL://:9094,CLIENT://:9095,CONTROLLER://:9093,EXTERNAL://:9092 \
  --env KAFKA_CFG_ADVERTISED_LISTENERS=INTERNAL://192.168.59.20:9094,CLIENT://:9095,EXTERNAL://192.168.59.20:9092  \
  --env KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=INTERNAL:SASL_PLAINTEXT,CLIENT:SASL_PLAINTEXT,CONTROLLER:PLAINTEXT,EXTERNAL:SASL_PLAINTEXT  \
  --env KAFKA_CFG_INTER_BROKER_LISTENER_NAME=INTERNAL \
  --env KAFKA_CFG_SASL_MECHANISM_INTER_BROKER_PROTOCOL=PLAIN \
  --env KAFKA_CFG_SASL_ENABLED_MECHANISMS=PLAIN \
  --env KAFKA_CLIENT_USERS=vpn \
  --env KAFKA_CLIENT_PASSWORDS=BCfoV77N \
  --env KAFKA_INTER_BROKER_USER=vpn \
  --env KAFKA_INTER_BROKER_PASSWORD=BCfoV77N  \
  --env KAFKA_CFG_NODE_ID=1 \
  --env KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@192.168.59.20:9093,2@192.168.59.21:9093,3@192.168.59.22:9093 \
  --env KAFKA_KRAFT_CLUSTER_ID=abcdefghijklmnopqrstuv  \
  --env KAFKA_HEAP_OPTS='-Xms512M -Xmx4G' \
  bitnami/kafka:3.5.1
 docker run --detach \
   --net=host \
   --name kafka2 \
   --restart always \
   --volume /data/kafka_raft:/bitnami/kafka \
   --env TZ=Asia/Shanghai \
   --env KAFKA_CFG_PROCESS_ROLES=broker,controller  \
   --env BITNAMI_DEBUG=true  \
   --env ALLOW_PLAINTEXT_LISTENER=no  \
   --env KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER  \
   --env KAFKA_CFG_NUM_PARTITIONS=1 \
   --env KAFKA_CFG_LISTENERS=INTERNAL://:9094,CLIENT://:9095,CONTROLLER://:9093,EXTERNAL://:9092 \
   --env KAFKA_CFG_ADVERTISED_LISTENERS=INTERNAL://192.168.59.21:9094,CLIENT://:9095,EXTERNAL://192.168.59.21:9092  \
   --env KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=INTERNAL:SASL_PLAINTEXT,CLIENT:SASL_PLAINTEXT,CONTROLLER:PLAINTEXT,EXTERNAL:SASL_PLAINTEXT  \
   --env KAFKA_CFG_INTER_BROKER_LISTENER_NAME=INTERNAL \
   --env KAFKA_CFG_SASL_MECHANISM_INTER_BROKER_PROTOCOL=PLAIN \
   --env KAFKA_CFG_SASL_ENABLED_MECHANISMS=PLAIN \
   --env KAFKA_CLIENT_USERS=vpn \
   --env KAFKA_CLIENT_PASSWORDS=BCfoV77N \
   --env KAFKA_INTER_BROKER_USER=vpn \
   --env KAFKA_INTER_BROKER_PASSWORD=BCfoV77N \
   --env KAFKA_CFG_NODE_ID=2 \
   --env KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@192.168.59.20:9093,2@192.168.59.21:9093,3@192.168.59.22:9093 \
   --env KAFKA_KRAFT_CLUSTER_ID=abcdefghijklmnopqrstuv  \
   --env KAFKA_HEAP_OPTS='-Xms512M -Xmx4G' \
   bitnami/kafka:3.5.1
   docker run --detach \
     --net=host \
     --name kafka3 \
     --restart always \
     --volume /data/kafka_raft:/bitnami/kafka \
     --env TZ=Asia/Shanghai \
     --env KAFKA_CFG_PROCESS_ROLES=broker,controller  \
     --env BITNAMI_DEBUG=true  \
     --env ALLOW_PLAINTEXT_LISTENER=no  \
     --env KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER  \
     --env KAFKA_CFG_NUM_PARTITIONS=1 \
     --env KAFKA_CFG_LISTENERS=INTERNAL://:9094,CLIENT://:9095,CONTROLLER://:9093,EXTERNAL://:9092 \
     --env KAFKA_CFG_ADVERTISED_LISTENERS=INTERNAL://192.168.59.22:9094,CLIENT://:9095,EXTERNAL://192.168.59.22:9092  \
     --env KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=INTERNAL:SASL_PLAINTEXT,CLIENT:SASL_PLAINTEXT,CONTROLLER:PLAINTEXT,EXTERNAL:SASL_PLAINTEXT  \
     --env KAFKA_CFG_INTER_BROKER_LISTENER_NAME=INTERNAL \
     --env KAFKA_CFG_SASL_MECHANISM_INTER_BROKER_PROTOCOL=PLAIN \
     --env KAFKA_CFG_SASL_ENABLED_MECHANISMS=PLAIN \
     --env KAFKA_CLIENT_USERS=vpn \
     --env KAFKA_CLIENT_PASSWORDS=BCfoV77N \
     --env KAFKA_INTER_BROKER_USER=vpn \
     --env KAFKA_INTER_BROKER_PASSWORD=BCfoV77N \
     --env KAFKA_CFG_NODE_ID=3 \
     --env KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@192.168.59.20:9093,2@192.168.59.21:9093,3@192.168.59.22:9093 \
     --env KAFKA_KRAFT_CLUSTER_ID=abcdefghijklmnopqrstuv  \
     --env KAFKA_HEAP_OPTS='-Xms512M -Xmx4G' \
     bitnami/kafka:3.5.1
```

###  测试kafka消息队列

创建一个新的topic

```
docker exec kafka1 /opt/bitnami/kafka/bin/kafka-topics.sh --bootstrap-server 192.168.59.20:9092 --topic test1 --create --partitions 1 --replication-factor 1 --command-config /opt/bitnami/kafka/config/producer.properties
```

列出所有的topic

```
docker exec kafka2 /opt/bitnami/kafka/bin/kafka-topics.sh --bootstrap-server 192.168.59.21:9092 --list --command-config /opt/bitnami/kafka/config/producer.properties
```

打开一个消费者会话

```
docker exec -it kafka1 /opt/bitnami/kafka/bin/kafka-console-consumer.sh --bootstrap-server 192.168.59.20:9092 --topic test1 --from-beginning --consumer.config /opt/bitnami/kafka/config/consumer.properties
```

打开一个生产者会话

```
docker exec -it kafka3 /opt/bitnami/kafka/bin/kafka-console-producer.sh --broker-list 192.168.59.22:9092 --topic test1 --producer.config /opt/bitnami/kafka/config/producer.properties
```

在生产者会话窗口输入信息，随后在消费者会话窗口查看成功收到信息，kafka集群部署成功。

###  部署kafka-ui

集群监控系统，可视化图标展示。

```
  docker run --detach \
  -p 8081:8080 \
  --name kafka-ui \
  --restart always \
  --env KAFKA_CLUSTERS_0_NAME=local \
  --env DYNAMIC_CONFIG_ENABLED=true \
  --env AUTH_TYPE=LOGIN_FORM \
  --env SPRING_SECURITY_USER_NAME=admin \
  --env SPRING_SECURITY_USER_PASSWORD=admin \
  provectuslabs/kafka-ui:master
```

浏览器访问：[http://ip:8081](http://ip:8081/) ，默认帐号密码：admin admin

进入之后添加kafka配置信息，sasl_plaintext   plain  账号 密码

![img](https://jafir-blog-oss.oss-cn-hangzhou.aliyuncs.com/2024/images/v2-ed9a75f96c5af9756e6bc4e21abcbe8a_720w.png)





添加图片注释，不超过 140 字（可选）

![img](https://jafir-blog-oss.oss-cn-hangzhou.aliyuncs.com/2024/images/v2-2a2b4489ab66dbe5521c7c09942a36e1_720w.png)





添加图片注释，不超过 140 字（可选）