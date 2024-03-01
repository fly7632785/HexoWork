# 前言

作者目前打算分享一期关于devOps系列的文章，希望对热爱学习和探索的你有所帮助。

文章主要记录一些简洁、高效的运维部署指令，旨在 **记录**和能够**快速地构建系统**。就像运维文档或者手册一样，方便进行系统的重建、改造和优化。每篇文章独立出来，可以单独作为其中一项组件的部署和使用。

本章为 **devOps系列（四）jenkins搭建**



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
mkdir -p /data/jenkins/
chmod o+w /data/jenkins/
```
#### 部署jenkins
```
docker run --detach \
  --publish 8080:8080 \
  --privileged=true \
  --user=0 \
  --name jenkins \
  --restart always \
  --volume /data/jenkins:/var/jenkins_home \
  --volume /usr/bin/docker:/usr/bin/docker \
  --volume /var/run/docker.sock:/var/run/docker.sock \
  --env TZ=Asia/Shanghai \
  --env JAVA_OPTS="-Xms1024m -Xmx4000m -Duser.timezone=GMT+08 -Dfile.encoding=utf-8 -DsessionTimeout=0 -Dhudson.security.csrf.GlobalCrumbIssuerConfiguration.DISABLE_CSRF_PROTECTION=true" \
  jenkins/jenkins:2.414

```
#### 访问
浏览器访问 http://ip:8080 ，初始化密码可以从日志查看：
```
docker logs jenkins
```
也可以直接查看初始化密码文件
```
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

默认安装推荐的插件  

创建管理员用户



建议：jenkins可以做一下域名解析，如 jenkins.jafir.top 到 nginx 转发到ip: 8080，来访问



jenkins装好后，最重要的就是构建任务的创建和测试了。



## pipeline示例

这里给一个java程序的示例：

```
pipeline {
    agent any
    environment {
        GIT_URL = "git@gitlab.jafir.top:example-server.git"
        JAR_FILE = "target/application.jar"
        IMAGE = "${JOB_NAME}"
        RegistryUrl = "harbor.jafir.top"
        GIT_BRANCH = "master"
        IMAGETAG = "prod-${BUILD_ID}"
        APP_PROFILE = "prod"
        JVM_OPTS = "-Dspring.profiles.active=${APP_PROFILE} -DappName=${JOB_NAME}  -Duser.timezone=GMT+08 -Dfile.encoding=utf-8 -Xmn1g -Xms2G -Xmx2G -XX:MetaspaceSize=100m -XX:+PrintCommandLineFlags -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xloggc:gc.log -XX:+PrintHeapAtGC -XX:-UseAdaptiveSizePolicy -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/java.hprof:"
        APP_CONTAINER_NAME = '${JOB_NAME}'
        APP_HOST_PORT = 8080
        APP_CONTAINER_PORT = 8080
    }
    stages {     
        stage('拉取代码') {
            steps {
                sh "echo '拉取代码'"
                git credentialsId: 'gitlab_credential_ssh', url: "$GIT_URL", branch: "$GIT_BRANCH"
            }
        }
        stage('maven构建') {
            agent {
                docker { 
                    image 'maven:3.6.3-jdk-8' 
                    args '-v /data/jenkins_maven_cache:/root/.m2'
                    reuseNode true
                }
            }
            steps {
                sh "echo 'maven构建'"
                    sh """
                        mvn -v
                        mvn clean  package  \
                                -Dmaven.test.skip=true \
                                -Dmaven.compile.fork=true \
                    """        
            }
        }
        stage('docker构建') {
            steps {
                configFileProvider([configFile(fileId: 'java_dockerfile', targetLocation: 'Dockerfile')]) {
                    withCredentials([usernamePassword(
                        credentialsId: 'harbor_credential',
                        usernameVariable: 'DOCKER_HUB_USER',
                        passwordVariable: 'DOCKER_HUB_PASSWORD')]) {
                        echo "构建 Docker 镜像"
                        sh """
                            docker login ${RegistryUrl} -u ${DOCKER_HUB_USER} -p ${DOCKER_HUB_PASSWORD} && \
                            docker build -t ${RegistryUrl}/java/${IMAGE}:${IMAGETAG} \
                            --build-arg JAR_FILE=${JAR_FILE} \
                            --build-arg JVM_OPTS='${JVM_OPTS}' .  && \
                            docker push ${RegistryUrl}/java/${IMAGE}:${IMAGETAG} && \
                            docker logout ${RegistryUrl}
                        """
                    }
                }
            }
        }
        stage('发布') {
            steps {
                echo "发布"
                script {
                    configFileProvider([configFile(fileId: 'deploy.sh', targetLocation: 'deploy.sh')]) {
                        sshPublisher(failOnError: true, publishers: [
                            sshPublisherDesc(
                                configName: 'ssh-server', 
                                transfers: [
                                    sshTransfer(
                                        cleanRemote: false,
                                        excludes: '',
                                        execCommand: "sh /root/data/project/${IMAGE}/deploy.sh '${RegistryUrl}/java/${IMAGE}:${IMAGETAG}' ${APP_CONTAINER_NAME} ${APP_HOST_PORT} ${APP_CONTAINER_PORT}",
                                        execTimeout: 120000,
                                        flatten: false,
                                        makeEmptyDirs: true,
                                        noDefaultExcludes: false,
                                        patternSeparator: '[, ]+',
                                        remoteDirectory: "/data/project/${IMAGE}",
                                        remoteDirectorySDF: false,
                                        removePrefix: '',
                                        sourceFiles: 'deploy.sh'
                                    )
                                ],
                                usePromotionTimestamp: false,
                                useWorkspaceInPromotion: false,
                                verbose: true
                            )
                        ])
                    }
                }
            }
        }
    }
    post {
        failure {
            // 构建失败时发送企业微信通知
            script {
                def message = "构建失败：${currentBuild.fullDisplayName}"
                // 使用HTTP POST请求发送消息到企业微信机器人
                sh """
                curl -X POST -H 'Content-Type: application/json' -d '{
                    "msgtype": "text",
                    "text": {
                        "content": "${message}"
                    }
                }' ${webhookUrl}
                """
            }
        }
    }
}
```

主要任务内容为：

从gitlab拉不同分支的代码，然后进行构建，构建好打包jar为docker img镜像，然后上传到harbor。再通过ssh 目标部署服务器执行 部署脚本，部署失败发送企业微信通知。部署脚本内容大致为： 从harbor拉取镜像，然后重启实例。



### 脚本文件及变量：

##### gitlab_credential_ssh: gitlab的ssh(证书 或者 账密)



##### java_dockerfile：java程序构建的通用dockerfile

```
FROM harbor.jafi.top/java/java-base:latest
ARG PROJECT_PATH=/app
ARG JAR_FILE
ARG JVM_OPTS
WORKDIR ${PROJECT_PATH}
COPY <jar> ${PROJECT_PATH}/app.jar
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

这里引用的baseJava镜像，自己打包的，主要是把一些常用命令和arthas打包进去，免得每次都要下载。

```
FROM openjdk:8u292-jdk-slim
RUN sed -i "s:deb.debian.org:mirrors.tuna.tsinghua.edu.cn:g" /etc/apt/sources.list \
  && sed -i "s:security.debian.org:mirrors.tuna.tsinghua.edu.cn:g" /etc/apt/sources.list \
  && apt-get update \
  && apt-get install -y libfreetype6 fontconfig wget curl vim telnet inetutils-ping \
  && apt-get clean \
  && apt-get autoclean \
  && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
COPY arthas /arthas-bin
```

```
docker build -t harbor.jafir.top/java/java-base:latest .
```

```
docker login harbor.jafir.top -u admin -p Harbor12345
docker push harbor.jafir.top/java/java-base:latest
```



##### ssh-server: 部署的目标机器配置，在jenkins的系统配置里面ssh添加server



##### RegistryUrl：harbor的地址 如  https://harbor.jafir.top



##### DOCKER_HUB_USER: harbor的账号



##### DOCKER_HUB_PASSWORD：harbor的密码



##### deploy.sh：

示例如下：

```
#!/bin/bash

# 提取参数
IMAGE="$1"
APP_CONTAINER_NAME="$2"
APP_HOST_PORT="$3"
APP_CONTAINER_PORT="$4"

# 登录 Harbor
docker login -u harborAccount -p harborPwd "https://barbor.jafir.top"

# 停止并删除已运行的容器
docker stop "${APP_CONTAINER_NAME}" >/dev/null 2>&1
docker rm "${APP_CONTAINER_NAME}" >/dev/null 2>&1

# 运行 Docker 容器
docker run -d --restart=always --name "${APP_CONTAINER_NAME}" -p "${APP_HOST_PORT}:${APP_CONTAINER_PORT}" "${IMAGE}"
```



正常情况下，jenkins的自动化构建就已经建立起来了，jenkins也可以通过配置定时或者触发来进行任务的自动化构建。最终发布到部署的目标服务器，进行部署更新。