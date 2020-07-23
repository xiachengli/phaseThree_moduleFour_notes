####  

#### Spring Cloud概述

###### Spring Cloud是什么

Spring Cloud是一系列框架的有序集合，如服务注册发现、配置中心、消息总线、负载均衡、断路器等。它利用springboot的特性简化了分布式系统基础设施的开发。

###### 架构及其核心组件

核心组件：

| 组件           | 第一代Spring Cloud         | 第二代Spring Cloud                  |
| -------------- | -------------------------- | ----------------------------------- |
| 注册中心       | Netflix Eureka             | Nacos                               |
| 客户端负载均衡 | Netflix Ribbon             | Dubbo LB、Spring Cloud LoadBalancer |
| 熔断器         | Netflix Hystrix            | Sentinel                            |
| 网关           | Netflix Zuul               | Spring Cloud Gateway                |
| 配置中心       | Spring Cloud Config        | Nacos                               |
| 服务调用       | Netflix Feign              | Dubbo RPC                           |
| 消息驱动       | Spring Cloud Stream        |                                     |
| 链路追踪       | Spring Cloud Sleuth/Zipkin |                                     |
| 分布式事务     |                            | seata分布式事务方案                 |

spring Cloud体系结构：

![image-20200723132021389](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200723132021389.png)

###### Spring Cloud VS Dubbo

Dubbo是一款高性能的远程调用框架，基于RPC调用。而Spring Cloud是微服务下的一系列解决方案，基于HTTP。

###### Spring Cloud VS Spring Boot

Spring Cloud利用了Spring Boot的特性使得开发人员能够快速上手实现微服务组件的开发

#### Eureka服务注册中心

分布式微服务中，服务注册中心用于存储服务提供者的信息。服务消费者通过主动查询或被动通知的方式获取服务提供者的信息。

注册中心一般性原理：

![image-20200723132709447](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200723132709447.png)

1. 服务提供者启动

2. 服务提供者将相关信息主动注册到注册中心

3. 服务消费者获取服务注册信息

   poll模式：主动拉取

   push模式：被动推送

4. 服务消费者调用服务

Eureka架构：

![image-20200723133832339](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200723133832339.png)

Eureka包含两个组件：Eureka Server和Eureka Client,client是一个java客户端，用于简化与Eureka Server的交互。Eureka Server提供服务发现的能力，各个微服务启动时，会通过Eureka Client向Eureka Server进行服务注册。

有关架构图的补充说明：

- 图中us-east-1c、us-east-1d、us-east-1e代表不同的区也就是不同的机房
- 图中每一个Eureka Server都是一个集群
- 图中Application Service作为服务提供者向Eureka Server中注册服务，Eureka Server接受到注册事件会在集群和分区中进行数据同步。Application Client作为消费端可以从注册中心获取服务提供者的信息
- 微服务启动后，会周期性地向Eureka Server发送心跳（默认周期为30秒）以续约自己的信息
- Eureka Server在一定时间内没有接收到某个微服务的心跳，Eureka Server将会注销该节点（默认90s）
- 每个Eureka Server同时也是Eureka Client，多个Eureka Server之间通过复制方式完成服务注册列表的同步
- Eureka Client会缓存Eureka Server中的信息

Eureka元数据详解：Eureka的元数据有两种，标准元数据和自定义元数据

标准元数据：主机名、IP地址、端口号等信息，这些信息都会被发布在服务注册表中，用于服务之间的调用

自定义元数据：可以使用eureka.instance.metadata-map配置

```yml
instance:    
	prefer-ip-address: true   
	metadata-map: 
	#自定义元数据<k,v>
		key: value
```

服务续约：服务每隔30秒会自动向注册中心发送心跳（也称续约或保活），如果没有续约。租约在90秒后到期，然后服务会被失败

```yml
#向Eureka服务注册中心注册服务
eureka:  
	instance:   
	#发送心跳时间间隔，默认30秒
	lease-renewal-interval-in-seconds: 30  
	#租约到期，服务失效时间，默认值90秒（服务超过90秒没有发生心跳，EurekaServer会将服务从列表移除）
    lease-expiration-duration-in-seconds: 90 

```

获取服务列表详情：每隔30秒服务会从注册中心拉去服务提供者列表，这个时间可以通过配置进行修改

```yml
eureka:
	client:
		registry-fetch-interval-seconds: 30
```

服务下线：当服务正常关闭时，会发送服务下线的REST请求给EurekaServer。服务中心接收到请求后，将服务置为下线状态

失效剔除：Eureka Server会定期进行健康检查，如果发现实例在配置的时间内没有发送心跳，则注销此实例

自我保护：在15分钟内超过85%的客户端节点都没有正常的心跳，那么Eureka server就会认为客户端与注册中心出现了网络故障，那Eureka Server会进入自我保护机制状态

当处于自我保护模式时：

1. 不会剔除任何服务实例（保证大多数服务依然可用）
2. Eureka Server依然能够接受新服务的注册和查询请求，但是不会被同步到其它节点上。待网络稳定时，当前Eureka Server新的注册信息会被同步到其它节点上
3. 在Eureka Server工程中通过eureka.server.enable-self-preservation配置可关闭/开启自我保护模式（默认开启）

#### Ribbon客户端负载均衡

负载均衡分为服务器端负载均衡和客户端负载均衡。服务器端负载均衡指的是请求到达服务器之后由负载均衡器根据一定的算法将请求路由到某个目标服务器进行处理；客户端负载均衡指的是服务消费者拥有一个服务提供者地址列表，调用方在请求前通过一定的负载均衡算法选择一个目标服务器进行访问

![image-20200723143659735](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200723143659735.png)

Ribbon负载均衡策略：

| 负载均衡策略                          | 描述                                                         |
| ------------------------------------- | ------------------------------------------------------------ |
| RoundRobinRule轮询策略                | 默认超过10次获取到的server都不可用的话，会返回一个空的server |
| RandomRule随机策略                    | 如果随机到的server为null或者不可用的话，会while不停的循环选取 |
| RetryRule重试策略                     | 一定时间内循环重试                                           |
| BestAvailableRule最小连接数策略       | 遍历服务列表，选取可用且连接数最小的一个server               |
| AvailabilityFilteringRule可用过滤策略 | 扩展了轮询策略。先通过轮询选取一个server，然后判断是否超时，当前连接数是否超限，判断成功再返回 |
| ZoneAvoidanceRule区域权衡策略（默认） | 扩展了轮询策略。除了过滤超时和链接数过多的server外，还会过滤掉不符合要求的zone区域里面的节点 |

修改负载均衡策略

```yml
#针对被调用方的微服务名称，不加就是全局生效
service-resume:   
	ribbon:    
		NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule 
```

Ribbon负载均衡原理：

![image-20200723144948484](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200723144948484.png)

####  Hystrix熔断器

什么是雪崩效应：微服务中，一个请求可能需要调用多个微服务接口才能完成，会形成了复杂的调用链路。如果对下游的某个微服务调用响应时间过长或者不可用，导致对上游的某个微服务的调用会占用越来越多的系统资源，进而引起系统崩溃的现象

雪崩效应的解决方案：

- 服务熔断

  当扇出链路的某个微服务不可用或者响应时间太长时，熔断该节点微服务的调用，进行服务的降级，快速返回错误的响应信息。当检测到该节点服务调用正常后，恢复调用链路

- 服务降级

  当某个服务熔断之后，此时客户端可以执行本地的fallback回调返回一个缺省值

- 服务限流

  限制服务的并发量，措施有：

  - 限制总并发数（比如数据库连接池、线程池）
  - 限制瞬时并发数（如nginx限制瞬时并发连接数）
  - 限制时间窗口内的平均速率（如Guava的RateLimiter、nginx的limit_req模块，限制每秒的平均速率）
  - 限制远程接口调用速率，限制MQ的消费速率

Hystrix时netflix开源的一个延迟和容错库，用于隔离访问远程系统，防止级联失败，从而提高系统的可用性和容错性。Hystrix主要通过以下几点实现延迟和容错

- 包裹请求：使用@HystrixCommand包裹对依赖的调用逻辑
- 跳闸机制：当某服务的错误率超过一定的阈值时，Hystrix可以跳闸，停止请求该服务一段时间
- 资源隔离：Hystrix为每个依赖都维护了一个小型的线程池（舱壁模式），如果该线程池已满，发往该依赖的请求将立即被拒绝而不是排队等待，从而加速失败判定
- 监控：Hystrix可以近乎实时地监控运行指标和配置地变化
- 回退机制：当请求失败、超时、被拒绝或断路器打开时，执行回退逻辑
- 自我修复：断路器打开一段时间后，会自动进入“半开”状态

####  Feign远程调用

feign时Netflix开发地一个轻量级RESTful的HTTP服务客户端

- feign可以帮助我们更加便捷，优雅的调用HTTP API
- SpringCloud对Feign进行了增强，使Feign支持SpringMVC注解

feign对负载均衡的支持：feign本身集成了Ribbon的依赖和自动配置，因此不需要额外引入依赖，可以通过ribbon.xx进行全局配置，也可以通过服务名.ribbon.xx来对指定服务进行细节配置

feign的超时时长

```yml
#针对的被调用方微服务名称,不加就是全局生效
ribbon:
  #请求连接超时时间
  ConnectTimeout: 2000
  #请求处理超时时间
  ##########################################Feign超时时长设置
  ReadTimeout: 3000
  #对所有操作都进行重试
  OkToRetryOnAllOperations: true
  ####根据如上配置，当访问到故障请求的时候，它会再尝试访问一次当前实例（次数由MaxAutoRetries配置），
  ####如果不行，就换一个实例进行访问，如果还不行，再换一次实例访问（更换次数由MaxAutoRetriesNextServer配置），
  ####如果依然不行，返回失败信息。
  MaxAutoRetries: 0 #对当前选中实例重试次数，不包括第一次调用
  MaxAutoRetriesNextServer: 0 #切换实例的重试次数
  NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RoundRobinRule #负载策略调整
# 开启Feign的熔断功能
feign:
  hystrix:
    enabled: false
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            ##########################################Hystrix的超时时长设置
            timeoutInMilliseconds: 15000
```

####  GateWay网关

Spring Cloud GateWay异步非阻塞，基于Reactor模型

一个请求到来时，网关根据一定的条件进行匹配，匹配成功后将请求转发到指定的服务地址，而在这个过程中，我们可以进行一些比较具体的控制（限流、日志、黑白名单）

网关的核心概念：

- 路由：由一个Id、一个目标URL、一系列的断言和Filter过滤器组成。如果断言为true则匹配该路由
- 断言：可以匹配Http请求中的所有内容（请求头、请求参数）
- 过滤器：一个标准的spring webFilter，使用过滤器，可以在请求之前或者之后执行业务逻辑

GateWay工作流程：

![image-20200723154650312](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200723154650312.png)

GateWay核心逻辑：路由转发+执行过滤器链

application.yml部分配置内容：

```yml
server:
  port: 9002
eureka:
  client:
    serviceUrl: # eureka server的路径
      defaultZone: http://localhost:8761/eureka/ #把 eureka 集群中的所有 url 都填写了进来，也可以只写一台，因为各个 eureka server 可以同步注册表
  instance:
    #使用ip注册，否则会使用主机名注册了（此处考虑到对老版本的兼容，新版本经过实验都是ip）
    prefer-ip-address: true
    #自定义实例显示格式，加上版本号，便于多版本管理，注意是ip-address，早期版本是ipAddress
    instance-id: ${spring.cloud.client.ip-address}:${spring.application.name}:${server.port}:@project.version@
spring:
  application:
    name: server-gateway
  cloud:
    gateway:
      routes: # 路由可以有多个
        - id: server-user # 我们自定义的路由 ID，保持唯一
          #uri: http://127.0.0.1:8096  # 目标服务地址  自动投递微服务（部署多实例）  动态路由：uri配置的应该是一个服务名称，而不应该是一个具体的服务实例的地址
          uri: lb://server-user                                                                    # gateway网关从服务注册中心获取实例信息然后负载后路由
          predicates:                                         # 断言：路由条件，Predicate 接受一个输入参数，返回一个布尔值结果。该接口包含多种默 认方法来将 Predicate 组合成其他复杂的逻辑（比如：与，或，非）。
            - Path=/api/user/**
          filters:
            - StripPrefix=1
        - id: server-email      # 我们自定义的路由 ID，保持唯一
          uri: lb://server-email
          predicates:                                         # 断言：路由条件，Predicate 接受一个输入参数，返回一个布尔值结果。该接口包含多种默 认方法来将 Predicate 组合成其他复杂的逻辑（比如：与，或，非）。
            - Path=/api/email/**
          filters:
            - StripPrefix=1
        - id: server-code      # 我们自定义的路由 ID，保持唯一
          uri: lb://server-code
          predicates:                                         # 断言：路由条件，Predicate 接受一个输入参数，返回一个布尔值结果。该接口包含多种默 认方法来将 Predicate 组合成其他复杂的逻辑（比如：与，或，非）。
            - Path=/api/code/**
          filters:
            - StripPrefix=1

```

GateWay路由规则详解：

![image-20200723155147176](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200723155147176.png)

时间点后匹配

```yml
spring:  
	cloud:    
		gateway:     
        	routes:       
            	- id: after_route        
                  uri: https://example.org          
                  predicates:            
                  	- After=2017-01-20T17:42:47.789-07:00[America/Denver]

```

时间点前匹配

```
spring:  
	cloud:    
		gateway:     
        	routes:       
            	- id: after_route        
                  uri: https://example.org          
                  predicates:            
                  	- Before=2017-01-20T17:42:47.789-07:00[America/Denver]
```

指定Cookie正则匹配

```
spring:  
	cloud:    
		gateway:     
        	routes:       
            	- id: after_route        
                  uri: https://example.org          
                  predicates:            
                  	- Cookie=chocolate,ch
```

请求路径正则匹配

```
spring:  
	cloud:    
		gateway:     
        	routes:       
            	- id: after_route        
                  uri: https://example.org          
                  predicates:            
                  	- Path=/api/user
```

GateWay过滤器

从生命周期时间点进行划分，过滤器可分为以下两种

| 生命周期时间点 | 作用                                             |
| -------------- | ------------------------------------------------ |
| pre            | 请求被路由之前调用。可实现统一认证、黑白名单过滤 |
| post           | 请求被路由之后执行。可实现日志记录、统计信息     |

从过滤器类型的角度分，过滤器分为GateWayFilter和GlobalFilter两种

#### Spring Cloud Config分布式配置中心

Spring Cloud Config是一个分布式配置管理中心，包含了Server端和Client端两个部分

server端：提供配置文件的存储，以接口的形式将配置文件的内容提供出去，通过@EnableConfigServer开启服务

Client端：通过接口获取配置数据并初始化自己的应用

![image-20200723161318764](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200723161318764.png)

```yml
spring:
  application:
    name: server-config
  cloud:
    config:
      server:
        git:
          uri: https://github.com/xxx/server-config-repo.git #配置git服务地址
          username: ******@qq.com #配置git用户名
          password: ******** #配置git密码
          search-paths:
            - server-config-repo
      # 读取分支
      label: master
```



####  Spring Cloud Stream消息驱动组件

spring Cloud Stream是一个构建消息驱动微服务的框架。应用程序通过inputs（相当于消费者）或者outputs（相当于生产者）来与Stream中的binder对象交互。Binder对象是用来屏蔽底层MQ细节的，它负责与具体的消息中间件进行交互

![image-20200723171242176](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200723171242176.png)

优点：屏蔽底层的消息中间件的细节差异，统一了MQ的编程模型，降低了学习、开发、维护MQ的成本

stream消息通信方式：Stream中的消息通信方式遵循了发布-订阅模式。当一条消息被投递到消息中间件后，它会通过共享的topic主题进行广播，消息消费者在订阅的主题中接收并触发业务逻辑处理

topic：spring cloud stream中的一个抽象概念，代表发布共享消息给消费者的地方。在不同的消息中间件中，topic对应着不同的概念，比如：rabbitMQ中它对应了exchange、kafka中对应了topic

spring cloud stream的相关注解

| 注解            | 描述                          |
| --------------- | ----------------------------- |
| @Input          | 标识输入通道                  |
| @Output         | 标识输出通道                  |
| @StreamListener | 监听队列                      |
| @EnableBinding  | 将Channel和Exchange绑定在一起 |

