# Eureka学习笔记

## 基础架构
- ***服务注册中心***
eureka提供，服务注册与发现功能，即eureka-server

- ***服务提供者***
springboot应用或其他平台提供的服务eureka通信机制的应用

- ***服务消费者***
从注册中心获取服务列表，得知从哪里可以获得服务，如ribbon、feign

## 服务治理机制

![服务架构](./assets/eureka架构.png)

### 服务提供者
- ***服务注册***
服务提供者想注册中心发送REST请求，附上元数据信息。 
eureka server收到请求后将元数据储存到双层结构map中，第一层的key为服务名，第二层的key为具体服务的实例名。
注册时许确认:
```
eureka.client.register-with-eureka=true
```

- ***服务同步***
高可用注册中心，注册中心间互相注册服务，当服务向一个注册中心发送注册请求时，该请求会被转发至其他相连的注册中心，实现服务同步

- ***服务续约***
服务提供者维护“心跳”，告诉eureka server，服务还存在，即为***服务续约***
心跳属性：
```
eureka.instance.lease-renewal-interval-in-seconds=30
eureka.instance.lease-expiration-duration-in-seconds=90
```

### 服务消费者
- ***获取服务***
服务消费者启动时发送rest请求到注册中心，注册中心返回一份*只读*的服务清单。
```
eureka.client.fetch-registery=true
eureka.client.registery-fetch-interval-seconds=30
```

- ***服务调用***
消费者获取清单后，通过服务名可以获得具体服务实例名及元数据信息，ribbon默认采用轮询的方式进行调用，达到负载均衡。

- ***Region和Zone***
访问实例的选择在eureka中有region和zone的概念，每个region中可以包含多个zone，每个服务器客户端需要注册到一个zone中。
每个客户端对应一个region和一个zone。
调用服务时，优先调用同一个zone中的服务提供方，若访问不到则访问其他的zone。

- ***服务下线***
实例发送rest请求给eureka server进行服务下线。

### 服务注册中心
- ***失效剔除***
服务实例不一定会正常下线，注册中心会每隔60s将清单中超时（90s）没有续约的服务剔除。

- ***自我保护***
统计心跳失败，15分钟内是否低于85%，如果出现低于的情况，eureka server会将当前实例注册信息保存，确保不会过期。
保护期间若实例出现问题，客户端会拿到已经出现问题的实例，因此要求客户端需要有容错机制，如请求重试、断路器机制。
保护机制参数：(可关闭，确保不可用实例被剔除)
```
eureka.server.enable-self-preservation=true
```

## 功能对应源码

eureka客户端即服务提供方、服务消费方，与eureka server通信，需配置注解:
```
@EnableDiscoveryClient
```
![DiscoveryClient接口示意图](./assets/WX20210708-155709@2x.png)
注解源码为：
```
@Target(ElementType.Type)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(EnableDiscoveryClientImportSelector.class)
public @Interface EnableDiscoveryClient {

}
```
指定注册中心位置：(在application.properties中配置)
```
eureka.client.serviceUrl.defaulZone=...
```
真正实现服务发现功能的是DiscoveryClient类。该类位于netflix的包中。

- ***eureka client负责的任务***
注册服务实例；
租约；
服务关闭期间，取消租约；
查询服务实例列表；
配置一个eureka server的URL列表。

- ***Region和Zone***
一个微服务应用只对应一个region，可以通过```getReigion()```函数获得返回值。如果不配置，则设置为默认region，可通过```eureka.client.region```属性来配置。
通过```getAvailabilityZones()```函数，获得zone信息，若无配置，则为默认指定zone，可通过以下属性来配置：
```
eureka.client.serviceUrl.defaulZone
eureka.client.availibility-zones
```
获取具体url的函数为```getEurekaServerServiceUrls()```，通过该函数可知，eureka.client.serviceUrl.defaulZone可配置多个zone，并用逗号','隔开。
使用ribbon进行服务调用时，可以在负载均衡时实现亲和性：优先访问相同zone中的服务提供实例。

- ***服务注册***
最终的注册操作由```register()```方法完成，由```instanceInfoRepicator```类中的```run()```方法定时调用。
```register()```最终使用的是```DiscoveryClient.instanceInfo```对象，即客户端提供给服务端的元数据。

- ***服务获取和服务续约***
```renew()```方法
服务获取会根据是否为第一次获取发送不同的rest请求。```fetchRegistry()```等方法。

- ***服务注册中心处理***
服务注册：```ApplicationResource```类的```addInstance()```方法