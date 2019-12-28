
## Spring Boot

### Spring Boot有什么作用    

Spring Boot通过自动化的配置简化Spring原有的样板化的配置。   

Spring Boot提供了很多现成的starter，可以快速的实现各种服务。  

### actuator   

actuator用于监控和管理服务信息。   



## 服务治理：Eureka

有了，找到了！

### Eureka的功能   

Eureka的功能是提供服务的注册和服务的发现。

实现一个Eureka服务非常简单：   
```java
@EnableEurekaServer
@SpringBootApplication
public class EurekaServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaServerApplication.class, args);
	}
}
```

配置文件：   
```
server.port=1111

eureka.instance.hostname=localhost
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
eureka.client.serviceUrl.defaultZone=http://${eureka.instance.hostname}:${server.port}/eureka/
```

服务提供者同样简洁，只需要`EnableDiscoveryClient`：   
```java
@EnableDiscoveryClient
@SpringBootApplication
public class HelloApplication {

	public static void main(String[] args) {
		SpringApplication.run(HelloApplication.class, args);
	}
}
```

提供controller:   
```java
@RestController
public class HelloController {
    @RequestMapping("/hello")
    public String index() {
        return "Hello Spring Boot";
    }
}
```

配置文件application.properties:   
```
server.port=8888
spring.application.name=hello-service
eureka.client.serviceUrl.defaultZone=http://111.222.83.251:1111/eureka/
```

服务消费者如下，通过url指明服务名称和接口，使用`RestTemplate`进行http请求。

```java
@Service
public class HelloService {
    @Autowired
    RestTemplate restTemplate;

    public String helloService() {
        return restTemplate.getForEntity("http://HELLO-SERVICE/hello", String.class).getBody();
    }
}
```

配置文件：

```
eureka.client.serviceUrl.defaultZone=http://111.222.83.251:1111/eureka/
```

### Eureka的运行    

有三种方法可以运行：   
1. 可以通过maven的编译、打包，最后`java -jar`运行。  
2. 用`spring-boot-maven-plugin`的`mvn spring-boot:run`运行。  
3. 在Idea中直接启动main方法。  

我本机java环境为9，在项目中设置的java为1.8，导致只有第3种方法有效，因为其它两种依赖于本机安装的java运行环境，尽量保持编译环境和运行环境一致。   

### 服务提供者的运行机制

Eureka用了双层map结构来维护注册的服务信息，第一层Key为服务的名称，第二层key为服务的实例名称。    

当某一个服务向一个Eureka节点注册服务时，Eureka会将该请求转发给其它的Eureka节点，从而可以同步服务注册信息。     

服务通过周期性的心跳来通知Eureka自己的情况，又称为服务续约（renew）。 

### 服务消费者的运行机制   

通过REST请求想Eureka获取服务列表，周期性的更新服务列表缓存。   

服务下线后，Eureka会通知给服务消费者。

### 服务注册中心的运行机制   

为了防止服务的异常下线，会周期性的清理列表中未续约的服务。    

### Region和Zone   

一个服务可以属于一个Region和多个Zone。   

服务用Region和Zone来刻画自己所处的物理位置，方便负载均衡器就近的选择同一个Zone服务。同时，服务也可以根据Region和Zone来选择应该向哪个Eureka注册。 



## 客户端负载均衡：Ribbon   

### 用RestTemplate发送rest请求    

Spring提供了方便的RestTemplate像目标服务发送请求，有GET、POST、PUT和DELETE等基本操作。

### 如何让客户端具有负载均衡能力   

 LoadBalanced注解通过向RestTemplate添加拦截器，使其具备负载均衡的能力。

 在服务消费者中实现如下，在RestTemplate上加入`LoadBalanced`注解。

 ```java
@SpringCloudApplication
public class RibbonConsumerApplication {

    @Bean
    @LoadBalanced
    RestTemplate restTemplate() {
        return new RestTemplate();
    }

    public static void main(String[] args) {
        SpringApplication.run(RibbonConsumerApplication.class, args);
    }
}
```

### 负载均衡器的基本功能   

- BaseLoaderBalancer
    - 维护该服务下的所有节点列表。  
    - 定义IPing对象，会定时轮询列表中的服务，检查服务是否存活。   
    - 定义负载均衡的规则IRule，这里是线性轮询的策略。   
    - 定义添加、获取服务的方法。   
- DynamicServerListLoadBalancer   
    - 在BaseLoaderBalancer基础上实现了动态获取服务的能力，实现了从Eureka查询服务的方法。    
    - 动态的更新服务列表。策略有定时更新和Eureka提示更新两种。
    - 定义Filter过滤出需要的服务节点，比如*区域感知*需求的服务会选择同一个区域的其它服务，比如根据服务的质量评估节点健康状况，选择更健康的服务节点进入列表。   
- ZoneAwareLoadBalancer   
    - 对DynamicServerListLoadBalancer进一步扩展，根据zone对服务实例划分，然后采用策略先筛选zone，最后再选一个服务实例。

### 负载均衡的策略   

- RandomRule   
    - 随机从列表中选择一个    
- RoundRobbinRule   
    - 线性轮询服务   
- RetryRule   
    - 在RoundRobbinRule基础上进行一定的重试
- WeightedResponseTimeRule   
    - 根据服务节点的响应时间，计算节点的权重，响应时间短的更容易被选中。
- 其它。
    - 采用自定义的过滤规则，先过滤出符合条件的节点结合，然后采用轮询的方式依次使用。



## 服务容错保护：Hystrix   

### Hystrix的功能   

服务降级、服务熔断、线程和信号隔离、请求缓存、请求合并以及服务监控等。

### 为依赖的服务提供舱壁   

对于每一个依赖的服务，Hystrix提供了专用的线程池，防止某个依赖服务影响其它的依赖服务，这种模式叫做“舱壁模式”（Bulkhead Pattern）。

### 断路器实现逻辑    

Hystrix通过注解`EnableCircuitBreaker`和`hystrixCommand`提供了断路保护功能，在下游服务调用产生异常时进行功能降级。

在如下条件下打开断路器：   
1. 在每秒请求数量QPS大于阈值时。   
2. 在错误百分比大于阈值时。   

当断路器打开时，如果打开时间已经到达设定睡眠时间，则去尝试发送请求，测试下游服务是否已经恢复正常，如果请求成功，则关闭断路器，恢复正常。

保留10秒的bucket历史数据，记录请求成功、失败、延迟和拒绝次数作为开闭断路器的依据。

消费者端实现如下：

```java
@Service
public class HelloService {
    @Autowired
    RestTemplate restTemplate;

    @HystrixCommand(fallbackMethod = "helloFallback")
    public String helloService() {
        return restTemplate.getForEntity("http://HELLO-SERVICE/hello", String.class).getBody();
    }

    public String helloFallback() {
        return "error";
    }
}
```

### 怎么减少网络请求？Hystrix提供了缓存功能   

根据请求参数，想结果缓存入线程安全的map结构。

Hystrix提供了`CacheResult`来实现缓存。

### 还能怎么减少网络请求？Hystrix提供了合并请求的功能

Hystrix提供了`HystrixCollapser`进行请求的合并，将一小段时间内的请求合并为一个，不仅减少了网络请求的次数，同时减少了线程池资源的占用。

## 声明式服务调用：Feign

Feign整合了Ribbon和Hystrix，除了这两者的功能，还提供了声明式的Web服务客户端的定义方式。

在调用其它服务时，可以通过如下简单封装实现。`FeignClient`指明了服务的名称，`RequestMapping`指明了服务的具体接口。和传统的方式相比，更为简洁方便。

```java
@FeignClient("hello-service")
public interface HelloService {
    @RequestMapping("/hello")
    String hello();
}
```

### 客户端和服务端共享接口定义

Feign在客户端声明的服务接口和服务端定义的接口是对应的，可以说形式完全相同。所以，可以将这些接口发布出来，供客户端和服务端共同使用，从而减少客户端绑定配置。


## API网关服务：Zuul

Zuul针对外部客户端的访问，提供了请求路由、负载均衡和校验过滤等基本功能，还有与服务治理结合、请求转发的熔断机制、服务的聚合等。

创建一个简单的zuul，首先开启zuul：  
```java
@EnableZuulProxy
@SpringBootApplication
public class ApiGatewayApplication {

	public static void main(String[] args) {
		SpringApplication.run(ApiGatewayApplication.class, args);
	}
}
```

配置路由规则：   
```
spring.application.name=api-gateway
server.port=5555

zuul.routes.api-a.path=/api-a/**
zuul.routes.api-a.serviceId=hello-service

zuul.routes.api-b.path=/api-b/**
zuul.routes.api-b.serviceId=feign-consumer

eureka.client.serviceUrl.defaultZone=http://localhost:1111/eureka/
```
每一个路由规则包含两个部分，path指明了外部访问的url，对应的serviceId指明了所要映射到的服务。

### Zuul的过滤器   

Zuul不仅做路由转发，还会对请求进行过滤。过滤会在请求的各个阶段执行：   
- **pre**：在请求到达网关API时的阶段，先做一些前置加工。   
- **routing**：在请求已经开始路由到目标服务器时，进行处理。
- **error**：该过滤器在上述两阶段抛出异常时，会进入该阶段进行处理，该阶段一般会在上下文中设置error标志，告诉port阶段返回适当的error信息。当然该阶段有时候会在post阶段之后出现，因为post阶段也可能抛出异常，这就导致客户端可能无法收到返回的信息，当然，有多种解决该问题的方法，不再赘述。   
- **post**：在目标服务将结果返回后，可以对返回值进行处理，然后将正确的返回值或者error信息返回给客户端。   

其实Zuul的过滤器更像是设计模式中的责任链或者handler，一个请求在收到、转发、返回的各个阶段里，由多个过滤器依次处理。

### Zuul动态加载路由配置和过滤器   

Zuul结合Spring Cloud Config，可以动态的获取配置信息，从而可以实现动态的更新路由。

Zuul结合Groovy等动态语言，可以在运行时动态的加载自定义的过滤器。

## 分布式配置中心：Config    

创建一个Config server十分容易，首先创建一个远程git仓库，我在码云中创建了https://gitee.com/tiantianchi/spring-config仓库。

创建spring boot项目，加入如下配置：

```java

@EnableConfigServer
@SpringBootApplication
public class ConfigServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConfigServerApplication.class, args);
	}
}

```

在application.properties中加入：  

```
spring.application.name=config-server
server.port=7001

spring.cloud.config.server.git.uri=https://gitee.com/tiantianchi/spring-config
spring.cloud.config.server.git.searchPaths=config-repo
spring.cloud.config.server.git.username=username
spring.cloud.config.server.git.password=password
```

在仓库中创建配置文件`tc-dev.properties`，然后通过`http://localhost:7001/tc/dev`可以访问其中的配置内容。

Config客户端的实现如下，首先创建配置文件`boostrap.properties`文件，填入如下配置：   

```
spring.application.name=tc
spring.cloud.config.profile=dev
spring.cloud.config.label=master
spring.cloud.config.uri=http://localhost:7001/

server.port=7002
```

客户端就可以从Config server中获取tc-dev.properties配置文件。

## 消息总线：Bus

Spring Cloud Bus依赖于现有的消息队列框架，如RabbitMQ和Kafka，实现微服务之间异步消息的发布和订阅。

可以利用Bus实现配置的自动更新。在git仓库中的配置发生变化时，通过hook将更新请求发送到Config服务，Config通过Bus发送给相关服务，告知其进行配置的更新。 

在Bus中，将RabbitMQ等工具作为消息发布和获取的代理。每个服务中，由Listener负责监听和处理事件，EventPublisher负责与消息代理进行通信，它会收到本地发送的事件，并且发送到代理中，同时从代理中获取事件，发回给本地注册的Listener。Endpoint则负责暴露API，用户可以通过Endpoint提供的接口发起事件，Endpoint会利用EventPublisher将事件发到消息代理中。