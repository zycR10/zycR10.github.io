---
title: SpringCloud学习笔记——消息总线
date: 2020-04-06 19:07:24
tags:
categories:
- SpringCloud
---

在微服务的架构中，会使用轻量级的消息代理去构建一个消息主题，让微服务中所有的实例都连接上来，主题中生产出的所有消息会被所有实例监听并且消费，这就称之为消息总线，有点儿类似于计算机架构中的消息总线的概念。有了这样一条总线就可以很方便的广播一些需要所有实例知晓的消息，例如上一篇我们刚刚完成的配置变更。

# 消息代理
消息代理（Message Broker）是一种架构模式，它再各个应用之间进行调度通知的作用，并且降低应用间的耦合性，消息代理是一个中间件产品，核心就是一个消息的路由程序，实现消息的接受和分发。目前市面上有很多开源的成熟产品供大家使用，比如ActiveMQ,Kafka,RabbitMQ,RocketMQ等，目前Spring Cloud Bus仅支持RabbitMQ和Kafka，下面我们介绍一下如何使用RabbitMQ来实现消息总线。

# RabbitMQ
RabbitMQ作为一款成熟的开源消息中间件，其基本概念和原理在搜索引擎中成千上万，我这里就不赘述了，有兴趣的同学可以大概了解一下RabbitMQ的各个定义，有利于后续的学习。

这里推荐一篇文章，详细的描述了在windows下如何正确安装RabbitMQ并启动管理平台，我把链接贴出来供大家参考：
```
https://blog.csdn.net/newbie_907486852/article/details/79788471
```

如果你访问`http://localhost:15672/#/users`能够成功显示管理平台登录界面，则说明你的安装成功了，使用guest/guest登录即可。

<!--more-->
# 快速入门
在上一篇中我们实现了配置中心的服务化以及动态更新，但是动态更新其实仍然属于一种伪动态，因为实际上需要我们手动刷新/refresh接口才可以，接下来我们通过rabbitmq和Spring Cloud Bus来实现真正的动态更新。

我们继续使用前面的config-client项目，在项目中添加依赖spring-cloud-starter-bus-amqp：
```
implementation 'org.springframework.cloud:spring-cloud-starter-bus-amqp'
```

在bootstrap.yml配置文件中添加有关rabbitmq的连接配置，这里我使用了我自建的springcloud用户，如果你没有自建的话可以使用默认的guest用户：
```
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: springcloud
    password: 123456
```

然后我们启动两个config-client，端口号分别为7777和7778，此时按照之前的测试方法，当在git中修改from值后，再次调用`http://localhost:7777/getFrom`接口发现并没有返回最新值，此时POST调用`http://localhost:7778/actuator/bus-refresh`之后，再次请求`http://localhost:7777/getFrom`接口，会发现返回值已改变，说明我们的Spring Cloud Bus已经通过mq通知了所有应用刷新from值

最后我们在github的仓库setting中进行webhook的设置，webhook是指在你的git远程仓库发生变化时允许git向你设置的指定路径发送请求，这里要求稍微高一些，需要将我们的服务部署到服务器上并且通过外网可以正常访问，将我们的/bus-refresh路径填写上去就可以了，这里就不详细的介绍这个流程了，如果有机会在公司的项目中用到Spring Cloud Bus的话可以尝试一下webhook功能