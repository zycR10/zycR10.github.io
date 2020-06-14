---
title: SpringCloud学习笔记——声明式服务调用Feign
date: 2020-02-18 23:12:20
tags:
categories:
- SpringCloud
---

# Feign简介
在前面我们已经搭建了Spring Cloud中的Ribbon和Hytrix，从而实现了微服务架构中客户端的负载均衡以及断路器机制保护服务，这两者的使用非常广泛并且经常一起出现，SpringCloud Feign其实就是一个整合工具，对两者进行了整合，除了原有的功能外，还提供了声明式的Web服务客户端定义方式。

# 快速搭建
* 新创建一个springboot项目feign-consumer，添加feign和eureka依赖：
```
dependencies {
	implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-server'
	implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
}
```
* 在启动类上添加注解开启Feign支持功能
```
@EnableDiscoveryClient
@EnableFeignClients
@SpringBootApplication
public class FeignConsumerApplication {

	public static void main(String[] args) {
		SpringApplication.run(FeignConsumerApplication.class, args);
	}

}
```
* 定义HelloService接口，添加@FeignClient注解指定已注册到注册中心的服务名，并且创建对应的REST接口
```
@FeignClient("hello-service")
public interface HelloService {

    @RequestMapping("/hello")
    String hello();
}
```
* 定义ConsumerController来实现service层的调用
```
@RestController
public class ConsumerController {

    @Autowired
    HelloService helloService;

    @RequestMapping(value = "/feign-consumer")
    public String helloConsumer() {
        return helloService.hello();
    }
}
```
当然最后不要忘记在配置文件中添加注册中心的地址并且指定端口号，由于这是我们springcloud学习过程中搭建的第四个项目，所以我这里port指定为4444
```
spring.application.name=feign-consumer
server.port=4444

eureka.client.service-url.defaultZone = http://localhost:1111/eureka/,http://localhost:1112/eureka/
```
我们已经简单的搭建了feign客户端，现在启动项目并且通过接口调用http://localhost:4444/feign-consumer ，你会发现和之前直接使用ribbon-consumer的调用结果是一样的，但是于ribbon不同的是，通过feign我们只需要定义服务接口，以声明式的方法优雅处理了服务调用

## 参数绑定
实际开发过程中这样的无参数接口几乎是很少见的，大部分情况下我们需要入参，接下来我们在hello-service中增加两个带入参的接口：
```
public String hello2(String name) {
        return "Hello " + name;
    }

    public String hello3(User user) {
        return "Hello " + user.getName() + ", " + user.getAge();
    }
```

在HelloController中增加对应的方法：
```
@RequestMapping(value = "/hello2", method = RequestMethod.GET)
    public String hello2(@RequestParam String name) {
        return helloService.hello2(name);
    }

@RequestMapping(value = "/hello3", method = RequestMethod.POST)
public String hello3(@RequestBody User user) {
    return helloService.hello3(user);
}
```

最后在feign-consumer中增加相应的实现，有一点需要注意的是，可能平时开发过程中习惯了在@RequestParam后面不会再指定入参的名字，因为在Spring MVC中将参数名作为默认值，但是在Feign中这个参数的value值是必须的，否则会报错：
```
Caused by: java.lang.IllegalStateException: RequestParam.value() was empty on parameter 0
```
这里比较容易犯错误的点就在于如果使用feign发送GET请求的话，那么在所有参数前都必须加上@RequestParam("xxx")，而如果是发送POST请求的话，那么可以加@RequestBody也可以不加，因为如果没有注解的话默认是@RequestBody

<!--more-->
## Ribbon配置
### 全局配置
Feign客户端的负载均衡是通过Ribbon实现的，所以在项目中我们可以对Ribbon进行配置，比如设置全局变量的超时时间：
```
ribbon.ConnectTimeout=500
ribbon.ReadTimeout=5000
```
### 指定服务配置
全局服务固然简单，但是像微服务这种项目显然各个服务的超时时间不可能统一，而是应该根据各自服务的特点进行调整，所以可以采用&lt;client&gt;.ribbon.key=value进行配置：
```
hello-service.ribbon.ReadTimeout=2000
hello-service.ribbon.ConnectTimeout=500
hello-service.ribbon.OkToRetryOnAllOperations=true
hello-service.ribbon.MaxAutoRetriesNextServer=2
hello-service.ribbon.MaxAutoRetries=1
```

### 重试机制
上面的配置中已经包含了关于重试的配置，我们可以修改hello-service进行一下验证：
* 在hello-service接口实现中加入随机延迟，只修改其中一个hello-service实例，另一个保持接口正常有利于后续观察结果，这里我们修改了2221端口的实例
```
//模拟超时
int time = new Random().nextInt(3000);
Thread.sleep(time);
```

* 在配置中我们配置了几个参数，hello-service.ribbon.MaxAutoRetries表示请求失败后会重试同一个实例n次，如果n次后仍然失败才会切换实例访问，而hello-service.ribbon.MaxAutoRetriesNextServer控制更换实例的次数

* 配置完成后全部重新启动，我们仍然通过http://localhost:4444/feign-consumer 的接口调用，会发现当负载均衡到port=2222的实例时请求总是正常的，当请求到port=2221时，由于我们模拟了接口超时，我遇到了这样的结果：
```
INFO 19584 --- [nio-2221-exec-1] o.zyc.springcloud.service.HelloService   : /hello, host :localhost, service_id :HELLO-SERVICE
INFO 19584 --- [nio-2221-exec-1] o.zyc.springcloud.service.HelloService   : sleep for 1781ms
INFO 19584 --- [nio-2221-exec-3] o.zyc.springcloud.service.HelloService   : /hello, host :localhost, service_id :HELLO-SERVICE
INFO 19584 --- [nio-2221-exec-3] o.zyc.springcloud.service.HelloService   : sleep for 397ms
INFO 19584 --- [nio-2221-exec-4] o.zyc.springcloud.service.HelloService   : /hello, host :localhost, service_id :HELLO-SERVICE
INFO 19584 --- [nio-2221-exec-4] o.zyc.springcloud.service.HelloService   : sleep for 508ms
INFO 19584 --- [nio-2221-exec-5] o.zyc.springcloud.service.HelloService   : /hello, host :localhost, service_id :HELLO-SERVICE
INFO 19584 --- [nio-2221-exec-6] o.zyc.springcloud.service.HelloService   : /hello, host :localhost, service_id :HELLO-SERVICE
INFO 19584 --- [nio-2221-exec-5] o.zyc.springcloud.service.HelloService   : sleep for 2443ms
INFO 19584 --- [nio-2221-exec-6] o.zyc.springcloud.service.HelloService   : sleep for 725ms
```
我们可以清楚的看到，前三次请求正常，因为超时时间设置了2000ms，如何第四次请求2443ms超过了设置，所以根据当前实例重试一次的配置，会再次请求port=2221的实例，这次随机了725ms，接口仍然正常返回。

**这里有一点需要注意的是，ribbon的超时和hytrix超时完全是两个概念，一个属于接口允许的超时时间和重试次数，另一个则是为了保护系统而设置的最差结果，所以hytrix超时时间必须大于ribbon超时时间，否则接口直接熔断，配置的重试机制也就毫无意义了**

## Hystrix配置
那么接下来我们就看看如何在feign中使用hystrix，在默认情况下，Spring Cloud Feign会将所有的Feign客户端方法都封装到Hystrix命令中进行服务保护

### 禁用Hystrix
我们可以通过**feign.hystrix.enabled=false**来关闭hystrix功能，但是如果不想全局关闭Hystrix支持的话，可以通过配置Feign.builder实例完成功能
* 创建一个配置类
```
@Configuration
public class DisableHystrixConfiguration {

    @Bean
    @Scope("prototype")
    public Feign.Builder feignBuilder() {
        return Feign.builder();
    }
}
```
* 在HelloService的@FeignClient注解中加入上面的配置类
```
@FeignClient(name = "hello-service", configuration = DisableHystrixConfiguration.class)
```

### 服务降级配置
在Feign中我们无法像前面是Hystrix那样在@HystrixCommand注解中加入fallback参数，但是Feign也提供了一种更为简单的方式：
* 创建一个服务接口的服务降级实现类，例如HelloService的实现类HelloServiceFallback，重写所有方法并针对每个方法提供降级策略即可
```
@Component
public class HelloServiceFallback implements HelloService {
    @Override
    public String hello() {
        return "error";
    }

    @Override
    public String hello(@RequestParam String name) {
        return "error";
    }

    @Override
    public String hello(@RequestBody User user) {
        return "error";
    }
}
```
* 在HelloService的@FeignClient中添加fallback属性
```
@FeignClient(name = "hello-service", fallback = HelloServiceFallback.class)
```

**1.要在配置文件中加入feign.hystrix.enabled=true才可以启动hystrix**
**2.记得把之前DisableHystrixConfiguration配置去掉**

此时启动feign-consumer之后不启动hello-service类，这是调用接口会发现返回了error，也就是我们预先设置的服务降级处理。