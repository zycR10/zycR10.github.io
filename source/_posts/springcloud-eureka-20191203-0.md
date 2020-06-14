---
title: SpringCloud学习笔记——搭建注册中心Eureka
date: 2019-12-03 23:46:35
tags:
categories: 
- SpringCloud
---

# 服务治理
服务治理是微服务中最核心和最基础的模块，用于各个微服务的注册和发现功能。在系统发展的初期可能由于模块不多，完全可以通过一些静态配置文件保存各个服务的地址，在各个项目中手工维护一份服务的实例清单。其实技术没有好与坏之分，只有适合与不适合，如果你的公司规模不大，拆分的服务项目一只手就数的过来，那为了顺应微服务的潮流而引入了一套spring cloud完全是加重项目负担，没有任何实际意义。但是相反，如果公司规模越来越大，拆分的服务越来越多，系统功能逐渐复杂，继续采用静态配置的方式就会非常难以维护，而且万一某个服务需要迁移或者变动，会产生牵一发而动全身的效果，相应的调用方都需要修改配置，所以遵从计算机领域真理，没有什么是加一个中间层不能解决的，**注册中心**这样的模块应运而生。
* 服务注册
服务治理框架中通常都会有一个注册中心，各个服务端像注册中心登记自己以及提供的服务，包括主机、端口号、版本号甚至服务协议等信息，由注册中心来维护各个服务实例的实例清单。另外注册中心还会通过心跳的方式检测实例健康程度，在实例不可用后会将实例剔除。
* 服务发现
相应的服务调用的时候肯定不会再是直接进行实例调用，而是会通过服务名称的方式向注册中心发出请求，获取实例清单以进行调用或访问

# 搭建服务注册中心
**Spring Cloud Eureka**，使用Netfilx Eureka来实现服务的注册与发现，Spring Cloud项目是基于Spring Boot项目搭建的，网上有非常多关于搭建SpringBoot项目的相关的而且使用非常简单，这里不再赘述，我使用的项目管理工具的gradle，在build.gradle文件中加入如下代码：
<!--more-->
```
plugins {
	id 'org.springframework.boot' version '2.1.3.RELEASE'
	id 'java'
}

apply plugin: 'io.spring.dependency-management'

group = 'org.zyc.springcloud'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '1.8'


repositories {
	maven{ url 'http://maven.aliyun.com/nexus/content/groups/public/' }
	maven{ url 'http://maven.aliyun.com/nexus/content/repositories/jcenter'}
}

ext {
	set('springCloudVersion', 'Greenwich.RELEASE')
}

dependencies {
	implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-server'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

启动一个服务注册中心页非常简单，可以通过@EnableEurekaServer注解启动，在SpringBoot启动类中加入这个注解即可
```
@EnableEurekaServer
@SpringBootApplication
public class EurekaServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaServerApplication.class, args);
	}

}
```

在默认设置中，服务注册中心会把自身作为客户端来尝试注册自己，可以通过配置项进行配置，同时为了区分后续可能启动的其他服务注册中心，这里通过设置端口号的方式进行区分，在项目的application.properties中设置即可，为了便于区分不同环境的配置，我们这里新建一个application-peer1.properties，作为第一个配置中心的配置文件
```
spring:
  application:
    name: eureka-server
server:
  port: 1111
eureka:
  instance:
    hostname: peer1
```
配置完成后启动SpringBoot项目即可，注意启动时加入启动应用参数--spring.profiles.active=peer1，用于区分使用配置文件，此时http://localhost:1111 即可访问服务注册中心


## 高可用注册中心
每当引入了新的模块或架构之后，首先我们需要考虑的就是随之带来的运维问题，在故障状态下我们需要保证各个服务节点的高可用性，注册中心也是如此，上面我们搭建的都是单节点的，我们下面再来构建一个高可用的服务注册中心。
Eureka设计之初就考虑到了高可用性，方式非常简单，Eureka Server的高可用实际上就是将自己作为服务向另一个注册中心注册自己，这样可以形成一组相互注册的注册中心，从而达到高可用的效果。
在之前的例子中我们使用了配置文件application-peer1，现在我们就可以复制一份application-peer2来作为第二个注册中心的配置文件，这里我们只需要修改一下端口号即可，当然为了实现互相注册的功能，我们需要加入一个serviceUrl参数，更新后的配置文件如下：
peer1增加serviceUrl指向peer2
```
spring:
  application:
    name: eureka-server
server:
  port: 1111
eureka:
  instance:
    hostname: peer1
  client:
    service-url:
      defaultZone: http://peer2:1112/eureka/
```
peer2修改端口号为1112，hostname peer并且serviceUrl指向peer1
```
spring:
  application:
    name: eureka-server
server:
  port: 1112
eureka:
  instance:
    hostname: peer2
  client:
    service-url:
      defaultZone: http://peer1:1111/eureka/
```
注意在启动时加入参数spring.profiles.active=peer2，此时访问http://localhost:1111 和http://localshot:1112 应该可以分别看到注册中心相互注册，实现了高可用