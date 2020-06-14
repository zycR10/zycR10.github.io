---
title: SpringCloud学习笔记——客户端负载均衡Ribbon
date: 2019-12-30 23:22:22
tags:
categories:
- SpringCloud
---

Ribbon是一个基于HTTP和TCP客户端的负载均衡工具，spring cloud在netflix ribbon的基础上对其进行了封装，我们可以简单的将一个REST请求转换成负载均衡的服务调用，Ribbon作为一个框架工具，不同于我们前面所操作的注册中心，服务提供者等模块，它并不需要独立的机器部署，而是只需要在代码中使用即可。

# 客户端负载均衡
负载均衡在系统框架中属于不算复杂的模块，但绝对是非常重要而且必须要实施的内容，因为负载均衡是解决高可用，缓解峰值压力和快速扩容的基本手段之一，一般情况下我们所说的负载均衡都是指服务端负载均衡，其中又分为硬件和软件均衡，其中F5是比较常见的硬件手段之一，而软件负载一般就是在服务端部署具有负载均衡功能的软件来支持分发工作，常见的如nginx等。而我们今天要讨论的ribbon负载均衡其实属于是客户端负载均衡，客户端可以通过注册中心获取到服务端的注册实例列表，并且通过心跳来维护服务端清单的健康性，有了服务清单后使用ribbon负载均衡非常简单：
* 服务提供者启动多个实例注册到注册中心
* 消费者直接调用被@LoadBalanced修饰的RestTemplate实现接口调用即可

# RestTemplate
之前我们使用了一个非常有用的对象RestTemplate，该类可以简单的实现针对不同请求类型和参数的调用，我们日常开发中最常见的可能是GET和POST方法，所以这里简单的介绍一下RestTemplate的使用：

## GET请求
getForEntity是调用GET请求的方法，该方法返回值ResponseEntity是Spring框架对于http相应的封装，其中储存了http状态码，头信息，请求体对象等信息，常见的使用方法：
```
RestTemplate restTemplate = new RestTemplate();
        ResponseEntity<String> responseEntity = restTemplate.getForEntity(
                "http://host:port/path?param={1}",
                String.class,
                "value");
        String bode = responseEntity.getBody();
```
如果接口的返回值比较复杂，那我们可能还需要在代码中定义好返回值的DTO供后续使用
```
RestTemplate restTemplate = new RestTemplate();
        ResponseEntity<XXXDTO> responseEntity = restTemplate.getForEntity(
                "http://host:port/path?param={1}",
                XXXDTO.class,
                "value");
        XXXDTO bode = responseEntity.getBody();
```
getForEntity还有几个重载方法，基本上都是对于参数取值的调整，还有类似于getForObject方法是只关心返回值body而不在乎其他信息，使用方法比较简单，这里不再过多介绍。

## POST请求
了解完GET请求后再看POST请求你会发现使用上基本大同小异，基本上GET和POST方法就可以涵盖我们日常开发的任务，如果更细分的话还有PUT请求，DELETE请求等，实际使用起来基本都是差不多了，这里也就不再占用篇幅了，有兴趣的可以自己尝试一下

## 源码浅析
（未完待续。。。）