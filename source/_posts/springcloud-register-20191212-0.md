---
title: SpringCloud学习笔记——服务注册
date: 2019-12-12 23:50:08
tags:
categories: 
- SpringCloud
---

## 注册服务提供者
在[上一篇文章](https://zycR10.github.io/2019/12/03/springcloud-eureka-20191203-0/)中我们搭建了一个高可用的服务注册中心，既然服务注册中心搭建完成，那么空有一个注册中心毫无意义，重要的是提供服务，所以我们现在可以尝试像注册中心注册一个服务的提供者，依旧类似于之前的方式生成一个SpringBoot项目，我们新建一个hello-service项目当作服务的提供者，build.gradle文件中稍作修改，加入如下依赖
```
implementation 'org.springframework.boot:spring-boot-starter-web'
implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
```
为了测试服务提供效果，我们简单的写一个HelloController，通过接口返回一个hello字符串即可
```
@RestController
public class HelloController {
    private final Logger logger = LoggerFactory.getLogger(getClass());

    @Autowired
    private DiscoveryClient client;


    @RequestMapping(value = "/hello", method = RequestMethod.GET)
    public String hello() {
        return "Hello World";
    }
}
```
最后在启动类加上注解@EnableDiscoveryClient，这样代码我们就完成了，但是到这里还不够，既然是要将服务提供者注册到服务注册中心，那么首先肯定要让提供者知道像哪里注册，所以还需要在application.properties中配置一下注册中心的url
```
server.port=2222
spring.application.name=hello-service
eureka.client.service-url.defaultZone = http://peer1:1111/eureka/,http://peer2:1112/eureka/
```
这里我们指定了端口号为2222，服务名字为hello-service以及刚刚搭建的eureka，这里完成后我们可以启动一下hello-service项目，当然在启动之前，别忘了先将我们之前搭建好的服务注册中心先启动，在启动日志中应该可以看到如下信息，说明我们的服务已经注册成功了，同时在注册中心管理页面http://localhost:1111 和 http://localhost:1112 中**Instances currently registered with Eureka**一栏中应该可以看到我们hello-service的注册信息了
```
DiscoveryClient_HELLO-SERVICE/{#localIP}:hello-service:2222: registering service...
DiscoveryClient_HELLO-SERVICE/{#localIP}:hello-service:2222 - registration status: 204
Tomcat started on port(s): 2222 (http) with context path ''
Updating port to 2222
Started HelloServiceApplication in 6.352 seconds (JVM running for 16.223)
```
那么至此，我们就拥有了一个高可用的服务注册中心，并将一个服务注册成功，可以说，一个最简单微服务系统已经搭建了