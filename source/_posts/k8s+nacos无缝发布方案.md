

# 前言

目前k8s 对微服务发版是可能出现问题的，即便是**滚动发布模式+健康检查** 也会出现发布期间的请求错误。

原因是：k8s新的实例启动起来，旧的就会立即被杀掉，而nacos和其他服务没法很快感知到 该服务已经下线（心跳机制），期间就会出现流量还会转到旧的节点上（实际上已经被杀掉了 就会报错）

所以方案的原理也是：在k8s杀掉旧的实例之前，让实例主动在nacos下线，且给予一定的缓冲时间（其他服务的列表刷新），然后额外再调用一下优雅下线接口（友好地让原来的任务执行完毕）



#### 服务侧：

yml配置：

```
server:
# tomcat开启优雅下线
  shutdown: graceful
 
management:
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
# spring容器优雅下线 暴露 shutdown 接口
    shutdown:
      enabled: true
    prometheus:
      enabled: true
    metrics:
      enabled: true
 
 
spring:
  lifecycle:
    # 优雅下线超时时间
    timeout-per-shutdown-phase: 5m
```





#### k8s侧：

1、添加健康检查接口   /health/detect

2、配置滚动发布更新 

3、jenkins的dockerfile配置 curl的安装下载 (保证docker中是可以直接执行curl命令的)

4、deployment.yaml配置 健康检查+preStop钩子

```
terminationGracePeriodSeconds: 300
 
      lifecycle:
        preStop:
          exec:
            command:
              - sh
              - '-c'
              - >-
                curl http://127.0.0.1:10105/actuator/deregister;sleep
                30;curl -X POST http://127.0.0.1:10105/actuator/shutdown;sleep 5;
       readinessProbe:
        failureThreshold: 3
        httpGet:
          httpHeaders:
            - name: Connection
              value: keep-alive
            - name: Content-Type
              value: application/json
            - name: User-Agent
              value: Kubernetes-readinessProbe
          path: /health/detect
          port: 10105
          scheme: HTTP
        initialDelaySeconds: 30
        periodSeconds: 5
        successThreshold: 1
        timeoutSeconds: 60
       livenessProbe:
        failureThreshold: 3
        httpGet:
          httpHeaders:
            - name: Connection
              value: keep-alive
            - name: Content-Type
              value: application/json
            - name: User-Agent
              value: Kubernetes-readinessProbe
          path: /health/detect
          port: 10105
          scheme: HTTP
        initialDelaySeconds: 30
        periodSeconds: 5
        successThreshold: 1
        timeoutSeconds: 60

```



核心：

先调用应用程序的nacos主动下线  缓冲一段时间，再优雅下线接口，再缓冲一段时间，最终才让k8s滚动发布

```
curl http://127.0.0.1:10105/actuator/deregister;
sleep 30;
curl -X POST http://127.0.0.1:10105/actuator/shutdown;
sleep 5
```

