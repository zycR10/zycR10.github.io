---
title: SpringCloud学习笔记——分布式配置中心
date: 2020-03-17 22:05:58
tags:
categories:
- SpringCloud
---

这一篇要学习的是SpringCloud中的新项目————配置中心，这一部分相对于前面来说属于相对比较独立的部分，在日常开发中配置文件几乎存在于各个应用之中，常见的是数据库的配置文件，项目框架的配置文件以及应用中使用的动态参数组成的配置文件，这种传统的配置文件有两个问题

* 1.对于某些配置项来说，各个应用间相差无几甚至完全一样，尤其是在微服务的体系下，应用数量多，如果要更改其中一项配置会十分痛苦
* 2.如果将配置写在了xxx.properties文件中，那么应用可能需要重启才会读取到配置项的变化，对于某些服务来说影响比较大

今天要学习的配置中心就是为了解决这样的痛点，Spingcloud Config的服务端又称为分布式配置中心，它是基于git存储的微服务应用，用于连接配置仓库和客户端，为其他应用提供配置信息，并且该项目不仅仅局限于java语言，它可以适用于其他任何系统，并且由于基于git，所以天生具备版本管理等优势。

# 快速入门

## 构建配置中心服务端
SpringCloud的组件入门可以说都是非常简单易操作，只不过这次的config-server需要加一些git操作。首先我们还是创建一个新的springboot工程config-server，并且添加config相关依赖：
```
implementation 'org.springframework.cloud:spring-cloud-config-server'
```

创建Spring Boot程序主类，并添加@EnableConfigServer注解
```
@EnableConfigServer
@SpringBootApplication
public class ConfigServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConfigServerApplication.class, args);
	}
}
```

接下来在application.yml中需要额外配置git相关信息：
```
server:
  port: 6666
spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/zycR10/SpringCloud-Learning
          search-paths: /config
          username: username
          password: password

```
* uri代表了配置的Git仓库位置
* search-paths代表了路径下的相对搜索位置
* username、password是访问Git仓库的用户名和密码
配置完成后首先启动我们的服务，确保在6666端口上正常启动 

<!--more-->


## 配置Git仓库
接下来我们需要配置Git仓库，在上面配置好的Git仓库位置新建四个properties文件
* springcloud-learn.properties
* springcloud-learn-dev.properties
* springcloud-learn-test.properties
* springcloud-learn-prod.properties

在文件中我们设置一个from属性，不同文件中存放不同的值
* from=git-default-1.0
* from=git-dev-1.0
* from=git-test-1.0
* from=git-prod-1.0

我们将文件内容完成后push到Git仓库，接下来就可以使用配置中心了，我们可以通过浏览器，postman，curl等多种方式访问配置资源，访问URL如下：
* /{application}/{profile}/{label}
* /{application}-{profile}.properties
* /{label}/{application}-{profile}.properties
上面的url中{application}-{profile}对应的配置文件，{label}表示Git分支，如不填则默认master，比如我们现在就可以通过GET请求访问`http://localhost:6666/springcloud-learn/dev`，得到如下返回：
```
{
    "name": "springcloud-learn",
    "profiles": [
        "dev"
    ],
    "label": null,
    "version": "924e60691ab93a960732209c9d7ca5dbc8ad3697",
    "state": null,
    "propertySources": [
        {
            "name": "https://github.com/zycR10/SpringCloud-Learning/config/springcloud-learn-dev.properties",
            "source": {
                "from": "git-dev-1.0"
            }
        },
        {
            "name": "https://github.com/zycR10/SpringCloud-Learning/config/springcloud-learn.properties",
            "source": {
                "from": "git-default-1.0"
            }
        }
    ]
}
```
由于是基于Git的版本控制，所以修改配置可以说异常方便，比如我们现在修改dev文件中的from值为git-dev-1.1，再次访问上面的url会发现dev文件中的值已经变了，这样就解决了我们开篇说的两个痛点：多应用重复配置文件和修改配置文件需要重启服务。另外观察日志你会发现在请求的过程中已经将配置文件做了缓存：
```
2020-03-18 23:02:15.550  INFO 9256 --- [nio-6666-exec-3] o.s.c.c.s.e.NativeEnvironmentRepository  : Adding property source: file:/C:/Users/27193/AppData/Local/Temp/config-repo-2774293861501680965/config/springcloud-learn-dev.properties
2020-03-18 23:02:15.551  INFO 9256 --- [nio-6666-exec-3] o.s.c.c.s.e.NativeEnvironmentRepository  : Adding property source: file:/C:/Users/27193/AppData/Local/Temp/config-repo-2774293861501680965/config/springcloud-learn.properties
```
这是为了防止Git仓库出现故障导致了无法加载配置信息，这也是我们之前提到过的，在微服务中，永远不要相信其他服务，要认为它一定会出问题，做好对应的备用计划。现在我们把网断掉使得无法连接到Git仓库，我们再次请求上面的接口控制台会抛出IO异常的错误，但是实际接口也返回了刚才的配置信息。