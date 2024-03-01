# 前言

作者目前打算分享一期关于devOps系列的文章，希望对热爱学习和探索的你有所帮助。

文章主要记录一些简洁、高效的运维部署指令，旨在 **记录**和能够**快速地构建系统**。就像运维文档或者手册一样，方便进行系统的重建、改造和优化。每篇文章独立出来，可以单独作为其中一项组件的部署和使用。

本章为 **devOps系列（七）grafana+prometheus监控告警**



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



## 使用 prometheus + blackbox-exporter + alertmanager 做http的接口监控和告警

![image-20240221133725314](https://jafir-blog-oss.oss-cn-hangzhou.aliyuncs.com/2024/images/image-20240221133725314.png)

### 安装blackbox-exporter
```
docker run  --restart=always  -d  --name blackbox-exporter -p 9115:9115  prom/blackbox-exporter
```

好了之后 http://localhost:9115可以访问查看

### 修改prometheus.yml
```
rule_files:
   - "blackbox_rules.yml"

scrape_configs:
  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    file_sd_configs:
    - files: ['http_check.yml']
      refresh_interval: 10s
    relabel_configs:
     - source_labels: [__address__]
       target_label: __param_target
     - source_labels: [__param_target]
       target_label: instance
     - target_label: __address__
       replacement: 192.168.20.2:9115  #blackbox-exporter 所在的机器和端口
```
rule_files 下面添加blackbox_rules.yml

scrape_configs下面添加job 

http_check.yml中添加检查接口
```
vi blackbox_rules.yml
```
```
groups:
- name: 服务探测
  rules:
  - alert: BlackboxProbeFailed
    expr: probe_success == 0
    for: 0m
    labels:
      severity: critical
      team: node
    annotations:      
        summary: Blackbox probe failed (instance {{ $labels.instance }})
        description: "服务在线检查失败\n当前值= {{ $value }}\nIp = {{ $labels.ip }}\nDomain= {{ $labels.domain }}\nEnv= {{ $labels.env }}\n服务名= {{ $labels.service }}"

  - alert: BlackboxProbeHttpFailure
    expr: probe_http_status_code <= 199 OR probe_http_status_code >= 400                            
    for: 0m
    labels:
      severity: critical
      team: node                                                            
    annotations:                                                               
        summary: Blackbox probe HTTP failure (instance {{ $labels.instance }})
        description: "HTTP状态码不在200-399\n当前值= {{ $value }}\nIp = {{ $labels.ip }}\nDomain= {{ $labels.domain }}\nEnv= {{ $labels.env }}\n服务名= {{ $labels.service }}"

  - alert: BlackboxSslCertificateWillExpireSoon
    expr: probe_ssl_earliest_cert_expiry - time() < 86400 * 30
    for: 0m
    labels:
      severity: warning
    annotations:
        summary: Blackbox SSL certificate will expire soon (instance {{ $labels.instance }})
        description: "SSL certificate expires in 30 days\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
```
添加接口检查配置
```
vi http_check.yml
```

```
- targets: [ 'https://www.jafir.top' ]
  labels:
    name: web
    env: prod
    service: api 
```

重启prometheus，测试即可



如上示例：可以对 www.jafir.top进行检测，如果失败的话会进行相应的告警信息提示，最终通过prometheus和alertmanager来触发告警。也可以对一些重要的网站、接口等进行检测。



# 短信和电话 prometheus-alert全家桶里面支持

目前我们使用的aliyun的语音和短信模板 不支持code (code只提供给验证码用)  所以这边只有把prometheus-alert源码下载下来改一下即可

可以找一套linux服务器，clone源码  当前是4.9版本
```
git@github.com:feiyu563/PrometheusAlert.git
```
安装go 
```
yum install go
```

目前修改地方为
```
PrometheusAlert/controllers/aliyun.go
```
添加了
```
        TemplateKey := beego.AppConfig.DefaultString("ALY_DX_Template_Key","code")
        TemplateKey := beego.AppConfig.DefaultString("ALY_DH_Template_Key","code")

```
全文件如下：
```
package controllers

import (
        "PrometheusAlert/models"
        "github.com/aliyun/alibaba-cloud-sdk-go/sdk/requests"
        "github.com/aliyun/alibaba-cloud-sdk-go/services/dysmsapi"
        "github.com/aliyun/alibaba-cloud-sdk-go/services/dyvmsapi"
        "github.com/astaxie/beego"
        "github.com/astaxie/beego/logs"
        "strings"
)

func PostALYmessage(Messages, PhoneNumbers, logsign string) string {
        open := beego.AppConfig.String("open-alydx")
        if open != "1" {
                logs.Info(logsign, "[alymessage]", "阿里云短信接口未配置未开启状态,请先配置open-alydx为1")
                return "阿里云短信接口未配置未开启状态,请先配置open-alydx为1"
        }
        AccessKeyId := beego.AppConfig.String("ALY_DX_AccessKeyId")
        AccessSecret := beego.AppConfig.String("ALY_DX_AccessSecret")
        SignName := beego.AppConfig.String("ALY_DX_SignName")
        Template := beego.AppConfig.String("ALY_DX_Template")
        TemplateKey := beego.AppConfig.DefaultString("ALY_DX_Template_Key","code")
        client, err := dysmsapi.NewClientWithAccessKey("cn-hangzhou", AccessKeyId, AccessSecret)

        request := dysmsapi.CreateSendSmsRequest()
        request.Scheme = "https"
        request.PhoneNumbers = PhoneNumbers
        request.SignName = SignName
        request.TemplateCode = Template
        request.TemplateParam = `{"`+TemplateKey+`":"` + Messages + `"}`
        response, err := client.SendSms(request)

        if err != nil {
                logs.Error(logsign, "[alymessage]", err.Error())
        }
        logs.Info(logsign, "[alymessage]", response)
        models.AlertToCounter.WithLabelValues("alydx").Add(1)
        ChartsJson.Alydx += 1
        return response.Message
}
func PostALYphonecall(Messages string, PhoneNumbers, logsign string) string {
        open := beego.AppConfig.String("open-alydh")
        if open != "1" {
                logs.Info(logsign, "[alyphonecall]", "阿里云电话接口未配置未开启状态,请先配置open-alydh为1")
                return "阿里云电话接口未配置未开启状态,请先配置open-alydh为1"
        }
        AccessKeyId := beego.AppConfig.String("ALY_DH_AccessKeyId")
        AccessSecret := beego.AppConfig.String("ALY_DH_AccessSecret")
        CalledShowNumber := beego.AppConfig.String("ALY_DX_CalledShowNumber")
        TtsCode := beego.AppConfig.String("ALY_DH_TtsCode")
        TemplateKey := beego.AppConfig.DefaultString("ALY_DH_Template_Key","code")
        mobiles := strings.Split(PhoneNumbers, ",")
        for _, m := range mobiles {
                client, err := dyvmsapi.NewClientWithAccessKey("cn-hangzhou", AccessKeyId, AccessSecret)
                request := dyvmsapi.CreateSingleCallByTtsRequest()
                request.Scheme = "https"
                request.CalledShowNumber = CalledShowNumber
                request.CalledNumber = m
                request.TtsCode = TtsCode
                request.TtsParam = `{"`+TemplateKey+`":"` + Messages + `"}`
                request.PlayTimes = requests.NewInteger(2)

                response, err := client.SingleCallByTts(request)
                if err != nil {
                        logs.Error(logsign, "[alyphonecall]", err.Error())
                }
                logs.Info(logsign, "[alyphonecall]", response)
        }
        models.AlertToCounter.WithLabelValues("alydh").Add(1)
        ChartsJson.Alydh += 1
        return PhoneNumbers + "Called Over."
}
```
后面可以在conf文件中添加使用
```
#阿里云短信模板key
ALY_DX_Template_Key=desc

#阿里云电话模板key
ALY_DH_Template_Key=desc
```
对应变量可以自定义，默认是code

比如这里是msg，我的aliyun的模板里面 就可以创建一个desc的
![img](https://jafir-blog-oss.oss-cn-hangzhou.aliyuncs.com/2024/images/1782e0dc87c4209a.png)

这里还遇到一个问题，可能是文件系统的问题，这里把Dockerfile的文件系统做了改动  FROM alpine:3.18 改为了centos7
```
FROM centos:7

LABEL maintainer="jikun.zhang"

RUN yum -y install epel-release && \
    yum -y install tzdata && \
    ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone && \
    yum -y remove epel-release && \
        mkdir -p /app/logs && \
    yum -y install sqlite curl && \
    yum clean all

HEALTHCHECK --start-period=10s --interval=20s --timeout=3s --retries=3 \
    CMD curl -fs http://localhost:8080/health || exit 1
```

修改好源码之后重新编译
```
make docker
```
就会在本地打包一个镜像出来，如果需要上传私服可以再打个tag上传（需要docker login先）

最终alertmanager就支持aliyun的短信和语音了
```
docker tag feiyu563/prometheus-alert:latest harbor.jafir.top/java/feiyu563/prometheus-alert:latest
docker push harbor.jafir.top/java/feiyu563/prometheus-alert:latest
```
![img](https://jafir-blog-oss.oss-cn-hangzhou.aliyuncs.com/2024/images/1782e112cadb8608.png)

### prometheus-laert配置
部署好了之后，修改app.conf文件 aliyun的accesskey secret等 （建议把数据映射到主机目录）

添加短信自定义模板
![img](https://jafir-blog-oss.oss-cn-hangzhou.aliyuncs.com/2024/images/1782e14638bf316d.png)

```
{{ range $k,$v:=.alerts }}{{if eq $v.status "resolved"}}
[Prometheus恢复信息]
{{$v.annotations.description}}
{{else}}
[Prometheus告警信息]
{{$v.annotations.description}}
{{end}}
{{ end }}
```

模板测试内容：
```
{"receiver":"prometheus-dx-ali","status":"firing","alerts":[{"status":"firing","labels":{"alertname":"BlackboxProbeHttpFailure","env":"prod","instance":"https://www.jafir.top","job":"blackbox-http","name":"http-web","service":"web","severity":"critical","team":"node"},"annotations":{"description":"HTTP状态码不在200-399\n当前值= 502\nName= http-web\nEnv= prod\n服务名= web","summary":"Blackbox probe HTTP failure (instance https://www.jafir.top)"},"startsAt":"2023-09-08T09:13:24.324Z","endsAt":"0001-01-01T00:00:00Z","generatorURL":"http://prometheus:9090/graph?g0.expr=probe_http_status_code+%3C%3D+199+or+probe_http_status_code+%3E%3D+400\u0026g0.tab=1","fingerprint":"af6387fb6deb5043"}],"groupLabels":{"alertname":"BlackboxProbeHttpFailure","instance":"https://www.jafir.top"},"commonLabels":{"alertname":"BlackboxProbeHttpFailure","env":"prod","instance":"https://www.jafir.top","job":"blackbox-http","name":"http-web","service":"web","severity":"critical","team":"node"},"commonAnnotations":{"description":"HTTP状态码不在200-399\n当前值= 502\nName= http-web\nEnv= prod\n服务名= web","summary":"Blackbox probe HTTP failure (instance https://www.jafir.top)"},"externalURL":"http://alertmanager:9093","version":"4","groupKey":"{}/{alertname=\"BlackboxProbeHttpFailure\"}:{alertname=\"BlackboxProbeHttpFailure\", instance=\"https://www.jafir.top\"}","truncatedAlerts":0}
```


添加电话自定义模板
![img](https://jafir-blog-oss.oss-cn-hangzhou.aliyuncs.com/2024/images/1782e1509c3880be.png)
```
{{ range $k,$v:=.alerts }}{{if eq $v.status "resolved"}}恢复信息{{$v.annotations.description}}{{else}}告警信息{{$v.annotations.description}}{{end}}{{ end }}
```
模板测试内容
```
 {"receiver":"prometheus-dh-ali","status":"firing","alerts":[{"status":"firing","labels":{"alertname":"BlackboxProbeFailed","env":"prod","instance":"https://www.jafir.top","job":"blackbox-http","name":"http-web","service":"web","severity":"critical","team":"node"},"annotations":{"description":"服务在线检查失败\n当前值= 0\nName= http-web\nEnv= prod\n服务名= web","summary":"Blackbox probe failed (instance https://www.jafir.top)"},"startsAt":"2023-09-08T09:13:24.324Z","endsAt":"0001-01-01T00:00:00Z","generatorURL":"http://prometheus:9090/graph?g0.expr=probe_success+%3D%3D+0\u0026g0.tab=1","fingerprint":"9d044445256d0302"}],"groupLabels":{"alertname":"BlackboxProbeFailed","instance":"https://www.jafir.top"},"commonLabels":{"alertname":"BlackboxProbeFailed","env":"prod","instance":"https://www.jafir.top","job":"blackbox-http","name":"http-web","service":"web","severity":"critical","team":"node"},"commonAnnotations":{"description":"服务在线检查失败\n当前值= 0\nName= http-web\nEnv= prod\n服务名= web","summary":"Blackbox probe failed (instance https://www.jafir.top)"},"externalURL":"http://alertmanager:9093","version":"4","groupKey":"{}/{alertname=\"BlackboxProbeFailed\"}:{alertname=\"BlackboxProbeFailed\", instance=\"https://www.jafir.top\"}","truncatedAlerts":0}
```

测试：
修改app.conf 的defaultPhone
![img](https://jafir-blog-oss.oss-cn-hangzhou.aliyuncs.com/2024/images/1782e183ce0face6.png)


#### alertmanager配置
添加路由和receiver

路由规则自己定义，这里是把blackbox的监控检查配置了进来，发送语音和打电话
```
routes:
  - receiver: 'web.hook.grafanaalert'  # 路由到名为 "web.hook.grafanaalert" 的接收器
    match:
      __alert_rule_namespace_uid__: 'IrqNMj34z'  # 匹配 alertname 为 "grafana" 的告警
  - receiver: 'prometheus-dh-ali'
    match:
      alertname: "BlackboxProbeFailed"
  - receiver: 'prometheus-dx-ali'
    match:
      alertname: "BlackboxProbeHttpFailure"
```


```
- name: 'prometheus-dx-ali'
  webhook_configs:
  - url: 'http://prometheus-alert-new:8080/prometheusalert?type=alydx&tpl=prometheus-dx-ali&phone=139xxxx'
    send_resolved: false
- name: 'prometheus-dh-ali'
  webhook_configs:
  - url: 'http://prometheus-alert-new:8080/prometheusalert?type=alydh&tpl=prometheus-dh-ali&phone=139xxxx'
    send_resolved: false
```
全文件：（这里又添加了两个receiver，发短信和wx \ 打电话和wx）
```
global:
  resolve_timeout: 15s
route:
  group_by: ['alertname','instance']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 2m
  receiver: 'web.hook.prometheusalert'

  routes:
  - receiver: 'web.hook.grafanaalert'  # 路由到名为 "web.hook.grafanaalert" 的接收器
    match:
      __alert_rule_namespace_uid__: 'IrqNMj34z'  # 匹配 alertname 为 "grafana" 的告警  
  - match:
      alertname: "BlackboxProbeFailed"
    receiver: 'prometheus-dh-ali-all'
  - match:
      alertname: "BlackboxProbeHttpFailure"
    receiver: 'prometheus-dx-ali-all'

receivers:
- name: 'web.hook.prometheusalert'
  webhook_configs:
  - url: 'http://prometheus-alert:8080/prometheusalert?type=wx&tpl=prometheus-wx&wxurl=你的企业微信webhookurl'
- name: 'web.hook.grafanaalert'
  webhook_configs:
  - url: 'http://prometheus-alert:8080/prometheusalert?type=wx&tpl=grafana-wx&wxurl=你的企业微信webhookurl'
- name: 'prometheus-dx-ali'
  webhook_configs:
  - url: 'http://prometheus-alert-new:8080/prometheusalert?type=alydx&tpl=prometheus-dx-ali&phone=139xxxx'
    send_resolved: false
- name: 'prometheus-dh-ali'
  webhook_configs:
  - url: 'http://prometheus-alert-new:8080/prometheusalert?type=alydh&tpl=prometheus-dh-ali&phone=139xxxx'
    send_resolved: false
- name: 'prometheus-dx-ali-all'
  webhook_configs:
  - url: 'http://prometheus-alert-new:8080/prometheusalert?type=alydx&tpl=prometheus-dx-ali&phone=139xxxx'
  - url: 'http://prometheus-alert:8080/prometheusalert?type=wx&tpl=prometheus-wx&wxurl=你的企业微信webhookurl'
    send_resolved: false
- name: 'prometheus-dh-ali-all'
  webhook_configs:
  - url: 'http://prometheus-alert-new:8080/prometheusalert?type=alydh&tpl=prometheus-dh-ali&phone=139xxxx'
  - url: 'http://prometheus-alert:8080/prometheusalert?type=wx&tpl=prometheus-wx&wxurl=你的企业微信webhookurl'
    send_resolved: false
```

重启alertmanager
```
docker restart alertmanager
```

目前上述配置文件可以参考使用，作用主要是配置电话+短信+企业微信的通知。