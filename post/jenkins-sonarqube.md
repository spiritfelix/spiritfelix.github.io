---
date: '2024-12-20T11:22:08+08:00'
draft: false
slug: 'jenkins-sonarqube'
title: '通过docker-compose安装Jenkins和SonarQube并通过飞书通知'
summary: '通过docker-compose安装Jenkins和SonarQube并通过飞书通知'
categories: 
    - DevOps
tags:
    - Jenkins
    - SonarQube
    - Docker
    - Docker Compose
    - 飞书
---
# 通过docker-compose安装Jenkins和SonarQube并通过飞书通知

## 前置知识
1. 需要熟悉docker及使用docker compose方式安装应用
2. 需要熟悉Jerkins的操作及Jenkinsfile的编写
3. 需要熟悉本地IP地址+端口的方式访问web应用
4. 需要熟悉域名解析及反向代理的配置
5. 需要熟悉Jenkins
6. 需要熟悉SonarQube
7. 需要熟悉webhook的调用

## 安装 Jenkins 及相关插件

### 安装Jenkins

首先在本地创建Jenkins目录, 目录中将存放关于Jenkins的相关文件
```shell
mkdir jenkins
```

进入`jenkins`目录, 
```shell
cd jenkins
```

在`jenkins`目录中创建`jenkins_home`目录
```shell
mkdir jenkins_home
```

在`jenkins`目录中创建`docker-compose.yml`文件, 文件内容如下:
```yml
networks:
    jenkinsnet:
        driver: bridge
services:
    jenkins:
        image: jenkins/jenkins:lts
        privileged: true
        user: root
        ports:
            - 8080:8080
            - 50000:50000
        volumes:
            - ./jenkins_home:/var/jenkins_home
            - /var/run/docker.sock:/var/run/docker.sock
            - /usr/local/bin/docker:/usr/local/bin/docker
        networks:
            - jenkinsnet
```

创建好之后, 通过docker compose启动Jenkins服务
```shell
docker compose up -d
```

通过服务器IP+端口的方式访问Jenkins, 或者解析子域名并设置反向代理到Jenkins服务的端口, 文中以 `jenkins.example.com` 为例

浏览器访问Jenkins并进行初始化设置, 然后在Jenkins中安装相关的插件, 如: Blue Ocean、Pipeline、Git、SonarQube Scanner for Jenkins等, 以及按需安装其他插件

## 安装SonarQube

在本地创建sonarqube目录, 目录中将存放关于SonarQube的相关文件
```shell
mkdir sonarqube
```

进入`sonarqube`目录,
```shell
cd sonarqube
```

在`sonarqube`目录中创建`sonarqube_home`和`postgresql`两个目录
```shell
mkdir sonarqube_home
mkdir postgresql
```
在`sonarqube_home`目录中创建4个目录
```shell
mkdir sonarqube_home/extensions
mkdir sonarqube_home/logs
mkdir sonarqube_home/data
mkdir sonarqube_home/conf
```

在`postgresql`目录中创建`data`目录
```shell
mkdir postgresql/data
```

在sonarqube目录中, 创建docker-compose.yml文件, 文件内容如下:
```yml
networks:
    sonarnet:
        driver: bridge
services:
    postgres:
        image: postgres:12
        restart: always
        ports:
            - 5432:5432
        volumes:
            - ./postgresql/data/:/var/lib/postgresql/data
        environment:
            TZ: Asia/Shanghai
            POSTGRES_USER: sonar
            POSTGRES_PASSWORD: sonar123
            POSTGRES_DB: sonar
        networks:
            - sonarnet
    sonar:
        image: sonarqube:community
        restart: always
        volumes:
            - ./sonarqube_home/extensions:/opt/sonarqube/extensions
            - ./sonarqube_home/logs:/opt/sonarqube/logs
            - ./sonarqube_home/data:/opt/sonarqube/data
            - ./sonarqube_home/conf:/opt/sonarqube/conf
        ports:
            - 8090:9000
        environment:
            SONARQUBE_JDBC_USERNAME: sonar
            SONARQUBE_JDBC_PASSWORD: sonar123
            SONARQUBE_JDBC_URL: jdbc:postgresql://postgres:5432/sonar
        links:
            - postgres:postgres
        depends_on:
            - postgres
        networks:
            - sonarnet
```

创建好之后, 通过docker compose启动SonarQube服务
```shell
docker compose up -d
```

通过服务器IP+端口的方式访问SonarQube, 或者解析子域名并设置反向代理到SonarQube服务的端口, 文中以 `sonar.example.com` 为例

## 配置SonarQube
### 创建SonarQube项目

通过浏览器访问SonarQube, 并创建本地项目

![](https://assets-spiritfelix-cn.oss-cn-chengdu.aliyuncs.com/20241223142725965.png)

项目名称和key根据实际项目自定义, 文中以`test`为例
在第二步选择使用全局设置即可

![](https://assets-spiritfelix-cn.oss-cn-chengdu.aliyuncs.com/20241223143158216.png)

在Analysis Method中选择Locally, 进入该界面

![](https://assets-spiritfelix-cn.oss-cn-chengdu.aliyuncs.com/20241223143226697.png)

设置Token名字并设置Token过期时间, 此处可根据实际设置, 为了方便此处以永不过期为例(生产环境建议设置Token过期时间并定期更新各处系统中使用的token), 生成token后, 根据实际情况选择项目类型和操作系统.

### 验证SonarQube可用

为了验证SonarQube的安装, 我们先在本地运行 scanner
在本地命令行中, 进入对应项目的目录, 并在命令行中执行界面中提示的命令

![](https://assets-spiritfelix-cn.oss-cn-chengdu.aliyuncs.com/20241223143255979.png)


等待命令执行之后, SonarQube项目页面将会变成如图所示

![](https://assets-spiritfelix-cn.oss-cn-chengdu.aliyuncs.com/20241223143311661.png)

即可在此界面中查看代码质量, 以及根据实际情况修复issue来提高质量评分

## 在 Jenkins 流水线中使用SonarQube

### 为项目增加sonar属性文件

在开始将SonarQube加入进Jenkins流水线中, 我们需要给项目增加一些文件, 以简化后续Jenkins流水线相关配置
在项目根目录创建 `sonar-project.properties` 文件, 其内容如下(内容中的参数根据实际情况修改, test仅为示例):
```properties
# key given when creating the project sonarqube
sonar.projectKey=test

# project name in sonarqube
sonar.projectName=test
sonar.projectVersion=1.0

# I want to scan everything inside ./src folder
sonar.sources=.

# Add the files you don't want to scan. Eg : Third parties libraries.
sonar.exclusions=**/*.doc,**/*.docx,**/*.ipch,/node_modules/,

#url to sonar instance
sonar.host.url=http://sonar.example.com/
```

### 在Jenkins中配置SonarQube Scanner

此前我们通过本地命令执行了scanner, 现在我们需要让scanner可以在Jenkins环境中执行

#### 准备SonarQube Token

首先我们需要准备一个给Jenkins使用的SonarQube Token
在SonarQube中, 登陆Administrator账户, 并在My Account中进入Security界面
在生成token的表单中为token取名(如 jenkins), Type选择Global Analysis Token, 并设置过期时间, 然后生成Token, 并将token保存好留用

![](https://assets-spiritfelix-cn.oss-cn-chengdu.aliyuncs.com/20241223143329094.png)


#### 安装SonarQube Scanner

进入Jenkins的系统管理 》全局工具配置, 找到`SonarQube Scanner 安装` (注意, 不是 for MSBuild), 然后新增 `SonarQube Scanner`, 并在表单中按如下内容填写并保存

![](https://assets-spiritfelix-cn.oss-cn-chengdu.aliyuncs.com/20241223143344198.png)

#### 配置SonarQube Server

进入Jenkins的系统管理 》 系统配置, 找到`SonarQube servers`, 并在表单中填写相关信息

![](https://assets-spiritfelix-cn.oss-cn-chengdu.aliyuncs.com/20241223143359481.png)

这里的token, 默认Jenkins中没有, 需要添加一个, 可以添加一个全局的 Secret text

![](https://assets-spiritfelix-cn.oss-cn-chengdu.aliyuncs.com/20241223143413282.png)

表单中的Secret, 填写此前在SonarQube中生成的Global Analysis Token
其他字段按需填写即可.
回到Jenkins的系统配置中选择SonarQube的Token, 并保存应用设置, 即可在Jenkins的pipeline中加入sonarqube代码检查

#### 在Jenkins流水线中加入代码检查

在项目的Jenkinsfile中, 添加SonarQube相关的stage(该Stage需要在代码检出之后)
```groovy
......
stage('SonarQube Analysis') {
    when {
        expression {
            currentBuild.result == null || currentBuild.result == 'SUCCESS'
        }
    }
    steps {
        script {
            echo 'SonarQube Analysis Stage'
            def scannerHome = tool name: 'SonarQube', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
            withSonarQubeEnv(installationName: 'SonarQube') {
                sh "${scannerHome}/bin/sonar-scanner"
            }
        }
        timeout(time: 3, unit: 'MINUTES') {
            script {
                sleep(120)
                def qg = waitForQualityGate()
                if (qg.status != 'OK') {
                    error "Pipeline aborted due to quality gate failure: ${qg.status}"
                    currentBuild.result = 'FAILURE'
                }
                currentBuild.result = 'SUCCESS'
            }
        }
    }
}
......
```

相关代码解释:
1. `def scannerHome`一行, 定义了scanner的工作目录, 根据名字`SonarQube`找到Jenkins的SonarRunnerInstallation插件
2. `withSonarQubeEnv`代码块约定内容在需要SonarQube环境
3. `sh "${scannerHome}/bin/sonar-scanner"` 只需执行sonar-scanner, 不同于本地执行, 需要的参数已经通过此前项目中创建的`sonar-project.properties`文件提供
4. `timeout(time: 3, unit: 'MINUTES') `定义了代码块执行的超时时间为3分钟
5. `sleep(120)`的目的是等待SonarQube分析结束, 否则`waitForQualityGate`获取不到结果
6. `def qg = waitForQualityGate()`将SonarQube的结果, 返回给qg, 并可以在后续代码中使用,  如果SonarQube检查通过, qg中的status属性为OK
7. `currentBuild.result` 为Jenkins当前构建的状态标签, stage根据前置步骤的状态决定当前步骤是否执行, 并影响最终构建成功与否

## 配置飞书通知机器人

通过此前步骤, 我们已经可以让项目通过Jenkins的流水线进行构建及发布, 由于整个过程需要等待流程执行完成, 因此我们希望当流程执行完成的时候, 通过飞书通知我们

### 创建飞书通知机器人

首先我们选择或者创建一个群聊天, 用来接受Jenkins的通知, 并在群聊中创建Jenkins通知机器人
在飞书群设置 》 群机器人 中, 添加机器人

![](https://assets-spiritfelix-cn.oss-cn-chengdu.aliyuncs.com/20241223143437263.png)

选择自定义机器人, 并取名(如 JenkinsBot)

![](https://assets-spiritfelix-cn.oss-cn-chengdu.aliyuncs.com/20241223143450933.png)

复制webhook地址留用

### 在Jenkins流水线中增加飞书通知

#### 增加环境变量
首先, 在pipeline的定义中增加飞书webhook的环境变量, 相关的Jenkinsfile修改如下:
```groovy
pipeline {
    ......
    environment{
        ......
        FeiShuWebHook = 'https://open.feishu.cn/open-apis/bot/v2/hook/xxxxxxxxxx'
        ......
    }
    ......
}
```
其中 地址换成上一步中获取到的webhook地址


#### 增加消息推送

在流水线结束阶段, 增加消息推送相关内容, Jenkinsfile相关内容如下:
```groovy
stage('Finishing'){
    steps {
        script{
            currentTime = sh(returnStdout: true, script: 'date +"%Y-%m-%d %H:%M:%S"').trim()
            if (currentBuild.result == null || currentBuild.result == 'SUCCESS' ) {
                status_cn = '成功'
            } else {
                status_cn = '失败'
            }
            build_link = env.BUILD_URL
              
            msg_json = """
                {
                    "msg_type": "post",
                    "content":
                    {
                        "post":
                        {
                            "zh_cn":
                            {
                                "title": "项目构建 ${status_cn} 通知",
                                "content": [[
                                    {
                                        "tag": "text",
                                        "text": "构建任务: ${JOB_NAME} - ${BUILD_DISPLAY_NAME}, "
                                    },
                                    {
                                        "tag": "text",
                                        "text": "构建分支: ${git_branch}, "
                                    },
                                    {
                                        "tag": "a",
                                        "text": "构建详情, ",
                                        "href": "${BUILD_URL}"
                                    },
                                    {
                                        "tag": "text",
                                        "text": "完成时间: ${currentTime}"
                                    }
                                ]]
                            }
                        }
                    }
                }
            """
            sh """
                curl -X POST -H "Content-Type: application/json" -d '${msg_json}' '${FeiShuWebHook}'
            """
        }
    }
}
```

相关代码解释:
1. `currentTime` 获取当前时间, 如果通知中不需要相关消息, 可以去掉相关部分
2. `currentBuild.result` 获取当前构建状态, 用来判断当前构建是否成功, 并设置状态文案
3. `build_link = env.BUILD_URL` 获取当前构建的连接, 可以通过消息中的连接直接访问对应构建的Jenkins地址
4. `msg_json` 按照飞书webhook通知所需的JSON结构, 组装消息结构,包含构建相关信息
5. `curl -X POST` 此行内容通过`curl`的方式, 向飞书的webhook地址发送构建好的JSON内容
至此, 项目构建结束的时候, 飞书将会收到对应的消息

![](https://assets-spiritfelix-cn.oss-cn-chengdu.aliyuncs.com/20241223143514195.png)



