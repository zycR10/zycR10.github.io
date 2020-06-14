---
title: SpringCloud学习笔记——API网关服务gateway
date: 2020-03-09 22:53:38
tags:
categories: 
- SpringCloud
---

在前面的学习中我们已经搭建了Eureka注册中心，服务提供者hello-service并且继承了Hystrix熔断器，Ribbon负载均衡消费者，并且学习了如何通过Feign进行整合，毫不夸张地说如果是面对一个小企业的项目，目前的框架再加上数据库，缓存和web页面之后可能已经可以上线使用了，但是如果项目规模逐渐发展，业务逐渐增多，就会暴露出当前架构一系列很致命的问题

# 当前问题

* 从运维角度来说，当前项目架构并不复杂，服务器应该也不多，如果引入了Nginx，F5等设施进行路由和负载均衡的话，需要运维人员配置转发的服务实例。如果服务的IP地址变了，还需要手动修改，再规模不大的时候还好，但是随着系统的壮大，运维人员一定会痛不欲生，而且出错概率大大增加
* 从开发角度来说，当我们的项目开始发展，可能业务越来越多样，一些对外的服务一定会涉及限流，鉴权，安全登录等问题，由于我们将应用拆分成了多个微服务，那么这些接口的校验要在每一份系统中都复制一份，虽然说复制粘贴是开发人员的常规操作，但是假如发现某个环节有bug，需要修复的就是所有项目，全部都需要重新上线，显然也是费时费力的重复劳动

# 解决办法
其实不难想到，在计算机领域，没有什么事情是加一层中间件不能解决的，所以API网关的想法应运而生，这类似于设计模式中的Facade模式，网关相当于系统的门面，所有接入方都是与门面打交道，在网关服务中实现了请求路由，负载均衡，校验过滤等功能，上面我们说的散落在各个微服务中的逻辑，其实与各个服务并没有太大关系，完全可以放入API网关应用中，spring cloud中的gateway就是为此而生

## 快速入门
搭建一个gateway项目可以说非常简单，新建一个springboot项目，依赖中加入spring-cloud-gateway的相关配置即可
```
implementation 'org.springframework.cloud:spring-cloud-starter-gateway'
```

网关最简单的功能其实就是集成后端接口并转发，有点类似于一个应用版的nginx，在项目中加入下面的代码即可实现转发功能：
```
    @Bean
	public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
		return builder
				.routes()
				.route("path_route", r->r.path("/feign-consumer*")
						.uri("http://localhost:4444")
				)
				.build();
	}
```
代码很简单应该一看就明白，网关接受路径为/feign-consumer的接口，转发到<http://localhost:4444>上，按照路径测试一下，我们访问<http://localhost:5555/feign-consumer>和<http://localhost:5555/feign-consumer2>应该分别可以看到之前我们搭建的接口返回，网关最基本的功能可以说就完成了，当然具体有很多匹配规则或者过滤条件，例如根据header，cookies，接口入参等，就不一一列举了

## 注册中心
上面的简单例子已经实现了网关功能，但是可能你也发现了，其实这样的网关在实际应用中没有任何意义，转发的接口信息还是自己维护的，跟我们文章开头说的运维痛点是一样的，甚至还需要多维护一个gateway应用，所以其实我们还是希望能够尽量做到自动化，在微服务中这种要求可以说很简单，为什么呢？因为我们所有服务都在注册中心啊！那把网关也放到注册中心让他去读取注册中心的服务不就可以了。
要实现这个功能很简单，首先我们要引入eureka的依赖，并且修改配置文件
```
implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
```
```
spring.cloud.gateway.discovery.locator.enabled=true
eureka.client.service-url.defaultZone = http://peer1:1111/eureka/,http://peer2:1112/eureka/
```
**这里要注意我们引入的eureka包和之前是不同的，这个包不会引入spring-boot-starter-web，该包会与gateway冲突，因为gateway使用的是webflux启动的**

启动后我们试验一下配置结果，注册到注册中心之后网管可以代理注册中心中所有的服务了，访问规则是这样的：
`http://gateway host:port/serviceId/url`
也就是说如果现在我们再访问前面的feign-consumer接口，则可以直接访问<http://localhost:5555/FEIGN-CONSUMER/feign-consumer>，这样就无需我们手动在项目中配置所有接口了

## 高级功能

* 限流
在高并发场景中限流是比较常见的手段，springcloud gateway中提供了基于redis的限流策略，首先要添加redis依赖spring-boot-starter-data-redis-reactive，然后在配置文件中进行配置：
```
eureka:
  client:
    service-url:
      defaultZone: http://peer1:1111/eureka/,http://peer2:1112/eureka/
server:
  port: 5555
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
      routes:
        - id: requestratelimiter_route
          uri: http://localhost:4444/feign-consumer
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 5
                redis-rate-limiter.burstCapacity: 10
                key-resolver: "#{@hostKeyResolver}"
          predicates:
            - Method=GET
  redis:
    host: localhost
    password:
    port: 6379

```