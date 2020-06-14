---
title: SpringCloud学习笔记——服务消费
date: 2019-12-16 23:18:47
tags:
categories:
- SpringCloud
---

在前文中我们已经搭建了高可用的注册中心，并向注册中心注册了两个服务，hello-service，现在已经有了服务的提供方，那么自然也要有服务的消费方，这篇文章就来搭建一个服务消费者，可以发现并且消费服务。服务的发现是由Eureka的客户端完成的，而服务的消费是由Ribbon完成的。Ribbon是一个负载均衡器，在客户端配置了ribbonServerList后即可达到负载均衡的效果，先不深究Ribbon的具体实现原理，**我们只需要理解它可以做到负载均衡，配合Eureka的服务发现机制可以达到选择服务实例的目的从而实现服务消费**

首先我们启动前文中搭建的两个注册中心和两个hello-service实例，启动后在注册中心的管理界面应该可以看到如下的服务：
![](/image/server_instance.jpg)

接下来仿照前面的项目，再创建一个新项目叫做ribbon-consumer，配置同样非常简单，在启动类上加入注解@EnableDiscoveryClient即可让应用注册为Eureka的客户端并具有发现服务的能力，同时在启动类中注入RestTemplate，并通过注解@LoadBalance注解开启该客户端的负载均衡功能，启动类代码如下：
```
@EnableDiscoveryClient
@SpringBootApplication
public class ConsumerApplication {

    @Bean
    @LoadBalanced
    RestTemplate restTemplate() {
        return new RestTemplate();
    }

    public static void main(String[] args) {
        SpringApplication.run(ConsumerApplication.class, args);
    }
}
```

配置了启动类之后我们再新建一个ConsumerController类，在该类中实现一个/ribbon-consumer接口，该接口通过RestTemplate调用hello-service的/hello接口，注意在代码中我们只需要指定service name，即HELLO-SERVICE，而并不需要输入具体的host和port，这也是服务治理框架中一个非常重要的特性，你需要知道的是服务，而不是背后具体的服务器，否则的话就和一个http调用的rpc接口请求没什么区别了，controller参考代码如下：
```
@RestController
public class ConsumerController {
    @Autowired
    RestTemplate restTemplate;

    @RequestMapping(value = "/ribbon-consumer")
    public String helloConsumer() {
        return restTemplate.getForEntity("http://hello-service/hello", String.class).getBody();
    }
}
```

当然最后不要忘记在配置文件application.yml中加入Eureka注册中心客户端的相关配置，注意端口号不要和前面的其他项目冲突：
```
spring:
  application:
    name: ribbon-consumer
server:
  port: 3333
eureka:
  client:
    service-url:
      defaultZone: http://localhost:1111/eureka
```
全部配置完成后启动ribbon-consumer项目，在Eureka的信息面板页面应该会发现多了一个ribbon-consumer应用，至此说明我们的服务发现和消费服务已经成功注册到了注册中心，接下来我们需要测试一下，不过在测试前我们可以优化一下之前hello-service的返回值，由于ribbon-consumer采用了负载均衡，而且我们有两个hello-service提供服务，所以我们可以通过在接口返回中打印具体的server端口号来区分调用被发送到了哪个server上，所以这里我们稍微改一下之前的/hello接口，在项目log和返回值中都加入一些信息：
```
@RestController
public class HelloController {
    private final Logger logger = LoggerFactory.getLogger(getClass());

    @Autowired
    private DiscoveryClient client;

    @Autowired
    private Registration registration;

    @Autowired
    private ServerConfig serverConfig;

    @RequestMapping(value = "/hello", method = RequestMethod.GET)
    public String index() {
        List<ServiceInstance> instances = client.getInstances(registration.getServiceId());
        if (CollectionUtils.isEmpty(instances)) {
            return null;
        }

        int port = serverConfig.getServerPort();
        for (ServiceInstance si : instances) {
            if (si.getPort() == port) {
                logger.info("/hello, host :" + si.getHost() + ", service_id :" + si.getServiceId());
                return "Hello World @" + port;
            }
        }
        return null;
    }
}
```

现在我们重启一下hello-service，然后可以直接在浏览器中访问http://localhost:3333/ribbon-consumer ，我们可以看到返回了**Hello World @2221**， 我们重新刷新一下页面，会发现变成了**Hello World @2222**，说明我们不仅成功的发现服务并且进行了消费，而且采用了轮询的负载均衡策略
![](/image/ribbon-consumer1.jpg)
![](/image/ribbon-consumer2.jpg)

另外有兴趣的朋友可以观察一下各个项目控制台中的日志，为我们打印了关于请求数，连接数，连接信息，失败信息等有用的信息供大家在实际工作学习排查问题。那么至此，微服务框架就已经具有了一个高可用注册中心，两个服务提供者hello-service，和一个服务发现者ribbon-consumer。