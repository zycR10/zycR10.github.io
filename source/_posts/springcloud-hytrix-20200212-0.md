---
title: SpringCloud学习笔记——服务容错保护Hystrix
date: 2020-02-12 22:54:18
tags:
categories: 
- SpringCloud
---

# 服务容错保护
在前面的微服务架构学习中我们已经有了自己的注册中心集群，服务提供者集群，以及一个处理客户端请求的负载均衡，但是微服务的世界里首先有一条黄金法则：**永远不要相信第三方服务**。也就是说在微服务中系统被拆分成了很多个模块，随着模块的增多，系统发生故障的概率也是指数倍的增加，常见的异常如网络超时，代码bug等，严重的可能由于某个服务的瘫痪而导致多个系统响应超时，如果事先没有有效的防范措施的话，很有可能会因此产生慢响应甚至宕机等问题，所以微服务断路器模式应运而生。

## 断路器模式
学过电路的人应该都听说过断路器，其实就是在电路上起到保护作用的装置，当电路发生短路导致过载时，断路器能够快速熔断电路，防止因过载导致的发热甚至火灾等更严重的后续问题，这个场景用在微服务中就十分贴切，当整个系统中某个服务发生了异常，很有可能导致与其关联的服务也出现异常，那么异常就会像病毒传播一样迅速影响整个系统，而如果有了熔断机制，那么当请求中存在异常节点时可以直接返回调用方明确的异常信息，从而降低风险

## 快速入门
我们仍然沿用之前搭建的微服务系统，启动我们的注册中心服务eureka-server，端口号1111，再启动两个服务单元hello-service，端口号分别为2221和2222，再启动一个ribbon客户端，端口号3333，此时直接访问我们之前写好的接口地址http://localhost:3333/ribbon-consumer， 反复刷新几次，页面上会轮流出现每次负载均衡后的端口号2221和2222，说明整个微服务系统在正常运转，这时我们只需要关闭端口号为2221的服务单元，当下次请求该server的请求发出后会有短暂的等待时间，然后接口抛出异常信息：
```
{
    "timestamp": "2020-02-12T15:18:55.004+0000",
    "status": 500,
    "error": "Internal Server Error",
    "message": "I/O error on GET request for \" http://hello-service/hello\ ": Connection refused: connect; nested exception is java.net.ConnectException: Connection refused: connect",
    "path": "/ribbon-consumer"
}
```
说明此时我们有个服务已经挂了，我本地使用post进行调试，本次接口耗时2.06s，两秒的时间也许不算长，但是如果这个服务每秒有成千上万次的调用呢？或者当前正处于高峰期时间，一个两秒响应的接口足以造成灾难性的后果了。

所以接下来我们就要引入我们的微服务断路器：***Spring Cloud Hystrix***

在ribbon-consumer项目的build.gradle文件中引入依赖，并在application类上使用注解 **@EnableCircuitBreaker**从而开启熔断功能
```
implementation 'org.springframework.cloud:spring-cloud-starter-netflix-hystrix'
```
同时我们改进一下我们的代码，虽然只是个小demo，但是很多规则还是要遵守的，比如业务代码不要写在controller里，这里我们再新建一个HelloService，把之前的代码挪到service类中的hello方法中，并在方法上增加@HystrixCommand注解，并标明回调方法(fallbackMethod = "helloFallback")：
```
@Service
public class HelloService {
    @Autowired
    RestTemplate restTemplate;

    @HystrixCommand(fallbackMethod = "helloFallback")
    public String hello() {
        return restTemplate.getForEntity("http://hello-service/hello", String.class).getBody();
    }

    private String fallback() {
        return "system error!";
    }
}
```
对应修改controller类中的方法，重新注入HelloService进行调用：
```
@RestController
public class ConsumerController {
    @Autowired
    HelloService helloService;

    @RequestMapping(value = "/ribbon-consumer")
    public String helloConsumer() {
        return helloService.hello();
    }
}
```
此时再次运行ribbon consumer项目，保持端口号为2221的服务单元终止服务状态，这次我们再来看下通过postman调用结果就会变成我们预设的返回值system error!，当然了，在回调函数中可以自定义友好的错误提示信息或者进行额外的业务处理。此外，除了服务宕机导致的不可用之外，可能由于当前系统正在承受非常大的流量请求或其他异常导致接口响应时间过长，hytrix同样可以达到断路的效果，hytrix默认的响应时间为2000毫秒最长，也就是说如果接口响应时间超过了两千毫秒，hytrix也会认为当前服务不再可用，从而像前面模拟服务器宕机一样调用我们预设的回调函数，这里我们需要修改以下hello-service的代码，在/hello接口中我们加入随机睡眠时间来随机性触发响应时间过长的问题，并且在ribbon-consumer调用代码中增加接口响应时长的日志打印，便于我们观察，以下是代码修改：
```
int time = new Random().nextInt(5000);
Thread.sleep(time);
```
```
@Service
public class HelloService {
    private static final Logger LOGGER = LoggerFactory.getLogger(HelloService.class);

    @Autowired
    RestTemplate restTemplate;

    @HystrixCommand(fallbackMethod = "helloFallback")
    public String hello() {
        long start = System.currentTimeMillis();
        String res = restTemplate.getForEntity("http://hello-service/hello", String.class).getBody();
        LOGGER.info("hellor cost : {}ms", System.currentTimeMillis() - start);
        return res;
    }

    private String helloFallback() {
        return "system error!";
    }
}
```
重新启动后继续通过postman调用，通过观察控制条输出日志和接口返回信息不难发现，凡是大于2000ms的请求全部调用了回调函数返回错误信息，其他请求则正常返回，那么在日常使用的过程中，合理的在请求中加入hystrix可以简单有效的防止服务宕机等因素带来的异常问题，同时要记得记录好日志以及回调之后的后续补偿操作。