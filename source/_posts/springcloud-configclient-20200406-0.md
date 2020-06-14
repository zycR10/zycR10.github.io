---
title: SpringCloud学习笔记——分布式配置中心客户端
date: 2020-03-31 22:48:00
tags:
categories:
- SpringCloud
---

# 客户端详解
在学习了如何配置Spring Cloud Config配置中心后，现在我们需要一个客户端来使用配置中心。

# 服务化配置中心
在微服务体系中，任何模块都应该服务化，任何需要服务化的东西都只需要向注册中心进行注册，就可以简单快捷的实现其服务化机制，从而让微服务中的其他服务更能够发现并使用，所以无疑配置中心需要接入eureka注册中心，过程我们就不赘述了，前面几乎每个模块都进行了类似的操作，只需要在config-server项目中引入eureka并在配置文件中配置注册中心地址即可，启动成功后可以在注册中心管理页面中看到我们的config-server服务已经成功注册。

# 客户端

## 直接配置
我们先来看一下如果没有接入注册中心的情况下，客户端如何获取到配置中心的值，下面我们启动一个新的springboot项目config-client作为配置中心客户端，在配置文件引入如下依赖：
```
implementation 'org.springframework.cloud:spring-cloud-starter-config'
implementation 'org.springframework.boot:spring-boot-starter-web'
```

然后在bootstrap.yml中配置我们的配置中心地址即可：
```
spring:
  application:
    name: springcloud-learn
  cloud:
    config:
      profile: dev
      label: master
      uri: http://localhost:6666/

```
两个注意点：
* bootstrap文件时优先于应用jar包内的配置文件加载的，所以配置在此文件中可以保证通过配置中心获取外部配置信息
* spring.application.name应该是配置中心中的{application}部分的名字，如果配置错误会导致启动失败

<!--more-->
配置完成后我们可以简单的实现一个RESTful接口获取配置：
```
@RestController
public class ConfigController {

    @Value("${from}")
    private String from;

    @RequestMapping("/getFrom")
    public String from () {
        return this.from;
    }
}
```

启动客户端后通过调用`http://localhost:7777/getFrom`可以得到当前配置中心的值

## 客户端服务化
上面的方法虽然也实现了客户端功能，但是由于在配置文件中存在类似于硬编码ip和port的行为，显然不符合微服务的要求，解决方法相信大家也很明白了，就是万能的注册中心，只要将客户端也接入注册中心，就可以轻松实现服务化。

在依赖中增加eureka依赖并且在配置文件中添加如下配置：
```
spring:
  application:
    name: springcloud-learn
  cloud:
    config:
      discovery:
        enabled: true
        service-id: config-server
      profile: dev
#  cloud:
#    config:
#      profile: dev
#      label: master
#      uri: http://localhost:6666/
eureka:
  client:
    service-url:
      defaultZone: http://peer1:1111/eureka/,http://peer2:1112/eureka/

```

## 快速失败与重试
正如我一直强调的，在微服务中任何时候都不能相信其他服务，要认为故障一定会出现，那么这种情况下就要做好故障处理措施，毕竟配置项一般属于比较重要的信息，如果配置项丢失可能导致业务流程错误或者甚至程序崩溃。

* 快速失败
只需要在配置文件中加入spring.cloud.config.failFast=true就可以实现快速失败功能，暂时关闭config-server然后重新启动config-client，在启动日志中我们可以看到：
```
java.lang.IllegalStateException: Could not locate PropertySource and the fail fast property is set, failing
```
由于配置中没有启动，我们的客户端也直接快速失败了

* 重试
由于配置中心异常导致其他项目启动失败应该并不是最理想的结果，毕竟可能我们希望有个更友好的处理方式，或者由于网络波动导致的异常的话完全可以通过重新解决，接下来我们就要引入spring-retry和spring-aop依赖：
```
implementation 'org.springframework.retry:spring-retry'
implementation 'org.springframework.boot:spring-boot-starter-aop'
```
不需要任何其他配置，再次重启项目，会看到进行了六次重试之后才报了错误，若对于默认重新次数或时间间隔有要求，可以通过配置参数配置：
spring.cloud.config.retry.multiplier 初始重试间隔时间
spring.cloud.config.retry.initial-interval 下一间隔重试时间
spring.cloud.config.retry.max-interval 最大间隔时间
spring.cloud.config.retry.max-attempts 最大重试次数

## 动态更新
对于配置文件来说有一个很重要的功能是动态更新，如果每次更改都需要重启项目的话就麻烦了，现在如果修改了git中from值后会发现通过config-client调用的返回值并没有改变，也就是说没有动态刷新，实现这个功能需要我们在config-client中新增spring-boot-starter-actuator模块，其中的/refresh实现了应用配置信息的重新获取和刷新，添加了依赖后还有两个需要更改的地方

* 在bootstrap.yml中加入actuator暴露的接口，默认只有health和info，缺少这个配置会报404：
```
management:
  endpoints:
    web:
      exposure:
        include: refresh,health,info
```

* 在config-client的ConfigController类上添加@RefreshScope用于标志需要刷新的范围

此时我们重启config-client，重新修改git中的from值，再次调用`http://localhost:7777/getFrom`发现配置没有改变，此时通过POST请求调用`http://localhost:7777/actuator/refresh`，config-client控制台中会输出重新拉取配置中心的日志，再次调用getFrom接口发现值已经改变了，说明内容已经被更新了

但是不难发现，虽然实现了更新，但是距离动态还是差了一些，总不能每次有了配置变更我们都要手动调用refresh接口进行更新，需要一个机制帮我们来实现这个需求，显然spring cloud早就帮我们实现好了，也就是下一篇中要讲的Spring Cloud Bus