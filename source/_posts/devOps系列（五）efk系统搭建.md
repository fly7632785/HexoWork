# 前言

作者目前打算分享一期关于devOps系列的文章，希望对热爱学习和探索的你有所帮助。

文章主要记录一些简洁、高效的运维部署指令，旨在 **记录**和能够**快速地构建系统**。就像运维文档或者手册一样，方便进行系统的重建、改造和优化。每篇文章独立出来，可以单独作为其中一项组件的部署和使用。

本章为 **devOps系列（五）efk系统搭建**



## 大纲

[devOps系列介绍]()

[devOps系列（一）docker搭建]()

[devOps系列（二）gitlab搭建]()

[devOps系列（三）nexus-harbor搭建]()

[devOps系列（四）jenkins搭建]()

[devOps系列（五）efk系统搭建]()

[devOps系列（六）grafana+prometheus搭建]()

[devOps系列（七）grafana+prometheus监控告警]()

[devOps系列（八）efk+prometheus+grafana日志监控和告警]()



# 正文

#### 创建数据目录
```
VOLUME_DIR=/data
mkdir -p ${VOLUME_DIR}/elasticsearch/{data,plugins}
mkdir -p ${VOLUME_DIR}/fluentd/{etc,log}
mkdir -p ${VOLUME_DIR}/kibana/data
mkdir -p /data/filebeat/{config,data}
chmod g+w ${VOLUME_DIR}/elasticsearch/{data,plugins}
chmod o+w ${VOLUME_DIR}/logstash/data
chmod +777 -R ${VOLUME_DIR}/kibana/data
chmod g+w ${VOLUME_DIR}/filebeat/data
chmod +777 -R ${VOLUME_DIR}/fluentd/log
```

#### 安装docker-compose
```
curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```
#### 创建docker-compose.yml
```
cat > /data/docker-compose.yml << EOF
version: '3.7'

networks:
  elk:
    driver: bridge

services:
  elasticsearch:
    image: elasticsearch:7.17.2
    container_name: elasticsearch
    hostname: elasticsearch
    restart: always
    volumes:
      - /data/elasticsearch/data:/usr/share/elasticsearch/data
      - /data/elasticsearch/plugins:/usr/share/elasticsearch/plugins
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      - "TZ=Asia/Shanghai"
      - cluster.name=elk
      - ES_JAVA_OPTS=-Xms4g -Xmx4g
      - discovery.type=single-node
      - bootstrap.memory_lock=true
	  - xpack.security.enabled=false
      - http.cors.enabled=true
      - "http.cors.allow-origin=*"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9200"]
      start_period: 20s
    networks:
      - elk
 
  fluentd:
    build: /data/fluentd
    container_name: fluentd
    hostname: fluentd
    restart: always
    volumes:
      - /data/fluentd/etc/fluent.conf:/fluentd/etc/fluent.conf
      - /data/fluentd/log:/fluentd/log
    ports:
      - "24224:24224"
      - "24224:24224/udp"
      - "514:514/udp"
    environment:
      - "TZ=Asia/Shanghai"
      - "FLUENTD_CONF=fluent.conf"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:24224"]
      start_period: 30s
    depends_on:
      - elasticsearch
    links:
      - elasticsearch:elasticsearch
    networks:
      - elk

  kibana:
    image: kibana:7.17.2
    container_name: kibana
    hostname: kibana
    restart: always
    volumes:
      - /data/kibana/data:/usr/share/kibana/data
    ports:
      - "5601:5601"
    environment:
      - "TZ=Asia/Shanghai"
      - "ELASTICSEARCH_HOSTS=http://elasticsearch:9200"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5601"]
      start_period: 20s
    depends_on:
      - elasticsearch
    links:
      - elasticsearch:elasticsearch
    networks:
      - elk
EOF
```



#### 创建fluentd配置文件

```
vi /data/fluentd/etc/fluent.conf 
```
```
<system>
  log_level error # debug模式利于排查问题
</system>

<source>
  @type  forward
  port  24224
</source>

<source>
  @type  syslog
  @id    syslog
  tag    syslog
  port   514
  bind   0.0.0.0
  <parse>
    @type syslog
  </parse>
</source>

# syslog传输端口
<source>
@type syslog
port 8888
#bind 0.0.0.0
tag nginx_access
</source>

<source>
@type http
@id input_http
port 8881
</source>

<filter nginx>
  @type parser
  format json
  key_name log
</filter>

<filter access*>
  @type parser
  key_name msg
  reserve_data true # 保留除temp外的其他字段
  remove_key_name_field false # 解析成功删除temp字段，如果要保留temp字段则关闭
  <parse>
    @type json
    json_parser json # 重点，必须加这句才能把json解析成字段
  </parse>
</filter>



<match **>
  @type elasticsearch
  host elasticsearch
  port 9200
  flush_interval 2s
  logstash_format true  #设置以后index为logstash-日期，代替index_name的值，并且索引添加@timestamp字段记录日志读取时间
  logstash_prefix ${tag}  #设置以后索引会以tag名称-日期进行命名
  index_name ${tag}-%Y.%m.%d
  type_name  ${tag}-%Y.%m.%d
  include_tag_key true  #把tag当做字段写入ES
  tag_key @log_name
  <buffer tag, time>  #让index_name ${tag}-%Y.%m.%d 时间格式生效
    timekey 15s # 在指定的时间刷新块
    timekey_wait 5s # 延迟5秒后写入块
    timekey_use_utc false # 是否使用utc时间
    timekey_zone: Asia/Shanghai # 默认本地时区，可以使用例如+0800或Asia/Shanghai

    @type file # 缓冲类型，可以设置file或者memory
    path /fluentd/log/elastic-buffer
    flush_thread_count 16
    flush_interval 1s
    chunk_limit_size 16M # 每个chunk块的大小，默认8MB
    queue_limit_length 512 # chunk块队列的最大长度，默认256。也就是缓存内最多可以存储 queue_limit * chunk_limit 的数据
    flush_mode interval
    retry_max_interval 30
    retry_forever true
  </buffer>
</match>


```

创建构建fluentd镜像的Dockerfile   
```shell
cat > /data/fluentd/Dockerfile << EOF
FROM fluentd:v1.14-1

USER root

RUN echo "source 'https://mirrors.tuna.tsinghua.edu.cn/rubygems/'" > Gemfile && \
  gem install 'elasticsearch:<=7.14' fluent-plugin-elasticsearch fluent-plugin-tail-ex fluent-plugin-tail-multiline

USER fluent

EXPOSE 24224 24224/udp
EOF
```



#### 启动
```
cd /data
docker-compose up -d
```
#### 访问
elasticsearch：http://ip:9200

fluentd：ip:24224

kibana：http://ip:5601

建议：kibana可以做一下域名解析，如 kibana.jafir.top 到 nginx 转发到ip: 5601，来访问日志；

fluentd也可以做一下域名解析，如 fluentd.jafir.top到ip（可能有多端口，所以解析到ip即可），而后使用fluentd收集地址为： fluentd.jafir.top:24224









