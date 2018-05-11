# Eureka

Eureka是一个 REST 风格的服务。主要用于在云平台上用以locating服务，以此来达到负载均衡以及中间件服务灾备的目的。我们称具有此种功能的服务为:*Eureka Server*。Eureka 同时提供一个java语言实现的客户端组件，*Eureka Client*,来简化通服务之间的交互。客户端同时包含一个内建的提供基本的*round-robin*（轮询）功能的负载均衡器。在Netflix,使用了更多复杂的负载均衡器来包装Eureka，例如：权重负债均衡器，可以根据服务的繁忙程度(*traffic*)、资源使用程度(*resource usage*)、错误场景(*error conditions*)等因素来和你的处理服务的负载。

## 背景

在云环境中，拥有固有的服务可以任意启停的特性。不像传统的负载均衡器可以使用服务的 IP 或者域名来负载。在云服务中，服务的注册(*registe*)和取消注册(*de-registe*)需要透明化，从而使得负载均衡变得更加复杂。因为 AWS 不提供中间件负载均衡，所以 Eureka 应运而生。

## Eureka 在Netflix的应用场景

在Netflix中，Eureka 应用场景：

1. 结合Asgard做灰度部署(*red/black*).Eureka结合Asgard保证新老服务的快速无缝切换。
2. 保证cassandra的发布；
3. 维护memcached服务列表，用以监测节点是否正常运行；

## Eureka 适用场景

1. 如果有一些中间件服务但是又不想直接暴露于公网。
2. 需要适用round-robin或者定制化负载均衡器；
3. 服务没有强制的session特性；

## 客户端和服务器如何数据交换

数据交换的方式可以是任意你喜欢的方式。Eureka 帮助你获取一些你需要的信息，但是不会额外的添加现在条件。实际使用场景中，你可以使用 Eureka 来获取服务的服务的地址或者服务使用的协议等。

### 基础架构

![eureka_architecture](/Users/chenkun/Documents/学习/spring-cloud/imgs/eureka_architecture.png)

上图展示了Eureka在Netflix是如何使用的，这通常也是大家在实际项目中的使用场景。

1. 每个地区(**region**)拥有一个Eureka集群，这个 Eureka 集群只知道自己地区(**region**)内的服务；
2. 每个地区(**zone**)至少有一个Eureka 来做灾备。
3. 服务在注册(**registe**)到 Eureka 之后每隔30s会发送心跳信息到 Eureka 来刷新它的存活续约；如果客户端多次无法刷新其存活续约，大概90S之后会从服务注册列表中剔除；
4. 注册信息以及心跳信息会自动分片到集群中的所有Eureka节点；
5. 任何 Eureka Client 可以查询注册信息(通常每30S一次)来定位他的服务来做远程调用。

### 非Java服务和客户端

自己实现 REST 服务的客户端调用。

### 可配置性(*Configurability*)

1. 使用 Eureka 你可以任意的增加或者删除集群的节点。
2. 可以调整线程池的超时*时间(*timeout*)；
3. 可以通过配置服务来动态更新参数，例如:spring-cloud-config或者Netflix的 archaius;

##  容错性(Resilience)

通过客户端的负载均衡可以来增加一定的容错性。