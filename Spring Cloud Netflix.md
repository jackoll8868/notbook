# Spring Cloud Netflix

该子项目提供一个将Netflix OSS通过自动配置，绑定到 Spring Enviroment和其他Spring方言的方式相互继承的方案。通过几个简单的annotations，你就可以快速的开启和配置相关的通用模式然后建立一个大信的分布式系统。Netflix包含的模块有*Service Discovery(Eureka)*、*Circuit Breaker(Hystrix)*、*Intelligent Routing(Zuul)*以及客户端的负载均衡*Ribbon*。

## 1 服务发现:Erueka Clients 

服务发现是微服务架构中最核心的一个组成部分。手动配置每个客户端是非常困难和容易出错的。Eureka 是Netflix体系中的服务发现服务和客户端。服务端可以通过配置、部署成为高可用的，每个服务节点会自动将每个注册服务的状态同步(replicate)到其它节点。

### 1.1 如何引入EurekaClient

引入EurekaClient需要在项目依赖中引入:`org.springframework.cloud`项目中的`spring-cloud-starter-netflix-eureka-client`。

### 1.2 注册Eureka

当一个客户端注册到Eureka，它会将它的原始数据提供给Eureka。例如host、port、健康监测URL、home page等。Eureka 会接受注册的所有client的心跳信息。如果心跳监测失败超过超时时间，那么对应的client讲从Eureka Server 剔除掉。

**Example eureka client:**

```Java
@Configuration
@ComponentScan
@EnableAutoConfiguration
@RestController
public class Application{
  @RequestMapping("/home“){
    return "Hello world";
  }
  
  public static void main(String[] args){
    new SpringApplicationBuilder(Application.class).web(true).run(args);
  }
}
```

 配置文件:

```Properties
eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka/
```

默认的应用名(ServiceId),虚拟主机和非安全的端口，均从*Enviroment*获取，分别映射为:`${spring.application.name}`,`${spring.application.name}`,`${server.port}`。

> **注:**
>
> 将`spring-cloud-starter-netflix-eureka-client`引入到项目的*classpath*之后，会使这个应用既是Eureka实例(自己注册给自己)也是Eureka client(可以查询注册信息来定位其他服务)。Eureka 服务的相关配置以`eureka.instance.*`定义。如果确保应用有一个`spring.application.name`配置项，那么可以使用默认的配置项。如果要关闭Eureka Discovery Client可以配置:`eureka.client.enabled=false`。
>
> 

### 1.3 Authenticating with the Eureka Server

HTTP模式的基本认证可以通过在eureka client的配置项中添加相关信息来完成。`eureka.client.serviceUrl.detaultZone=http://user:passowrd@eureka-server.com/eureka`

 如果需要更复杂的认证方式可以:

1. 定义一个`DiscoveryClientOptionArgs`的@Bean;
2. 将`ClientFilter`注入到步骤1的@Bean里边。

```Java
public class MutableDiscoveryClientOptionalArgs extends DiscoveryClientOptionalArgs {
    private Collection<ClientFilter> additionalFilters;

    public MutableDiscoveryClientOptionalArgs() {
    }

    public void setAdditionalFilters(Collection<ClientFilter> additionalFilters) {
        Collection<ClientFilter> additionalFilters = new LinkedHashSet(additionalFilters);
        this.additionalFilters = additionalFilters;
        super.setAdditionalFilters(additionalFilters);
    }

    public Collection<ClientFilter> getAdditionalFilters() {
        return this.additionalFilters;
    }
}

```

### 1.4 状态页和健康监测

Eureka实体的状态页和健康监测默认分别只想`/info`和`/health`.这两个请求通常默认由springboot程序的actuator实现。如果想要改变默认参数可以通过改变相关配置实现:

```Properties
management.contextPath=/admin
eureka.instance.statusPageUrlPath=${management.contextPath}/info
eureka.instance.healthCheckUrlPath=${management.contextPath}/health
```

这些连接可以展示Eureka 应用的原始信息，客户端可以用来判断是否需要像对应实体发送请求。

### 1.5 注册有安全机制的应用

如果app之间想通过https进行数据交换，可以设置`EurekaInstanceConfig`里边的两个参数来解决：

`eureka.instance.[nonSecurePortEnabled,securePortEnabled]=[false,true]`。

### 1.6 健康检测

默认情况下 Eureka 根据客户端的心跳来判断客户端是否运行。在不重新定制的情况下，Discovery Client不会讲当前的健康监测状态对外输出。换言之只要客户端启动成功那么该客户端的状态可能一直都是”UP"。通常可以通过修改`eureka.client.healthcheck=true`来完成。如果你想有更多的控制权可以实现`com.netflix.appinfo.HealthCheckHandler`。

> 注:
>
> `eureka.client.healthcheck.enabled=true`这个配置项只能配置在**application.properties**。如果配置在*bootstrap.properties*可能导致 注册的时候状态为UNKNOWN。

### 1.7 原始信息

Eureka 默认的标准原始信息包含:hostname,ip address,port numbers,status page,health check。这些原始信息被发布在服务注册的时候，客户端可以直接访问对端服务。如果想额外的添加参数可以配置`eureka.instance.metadataMap`，额外的参数可以被客户端读取，但是不会改变实例的默认操作。

### 1.8 使用EurekaClient

如果你有一个 discovery client 的应用你可以用来从Eureka发现服务。一种方式是使用本地的`com.netflix.discovery.EurekaClient`

```Java
@Autowired
private EurekaClient discoveryClient;
public String serviceUrl(){
  InstanceInfo instance = discoveryClient.getNextServerFromEureka("STORES",false);
  return instance.getHomeUrl();
}
```

#### 1.8.1 EurekaClient without Jersey

 默认情况下，EurekaClient使用Jersey 来进行 HTTP 请求。如果你不想使用Jersey,可以在  依赖中排除这个依赖。Spring Cloud将会自动配置一个transport client,通常为:RestTemplate

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
    <exclusions>
        <exclusion>
            <groupId>com.sun.jersey</groupId>
            <artifactId>jersey-client</artifactId>
        </exclusion>
        <exclusion>
            <groupId>com.sun.jersey</groupId>
            <artifactId>jersey-core</artifactId>
        </exclusion>
        <exclusion>
            <groupId>com.sun.jersey.contribs</groupId>
            <artifactId>jersey-apache-client4</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

### 1.9 其他的Netflix Eureka Client

你不一定非要使用`EurekaClient`。Spring Cloud已经支持*Feign*(一个REST客户端工具)同事Spring 的*RestTemplate*也可以访问Eureka Service 的物理地址。想要配置Ribbon包含一些物理服务的列表，可以配置`<client>.ribbon.listOfServers`，`<client>`是Client的 ID。

也可以使用`org.springframework.cloud.client.discovery.DiscoveryClient`这个通用的 API 调用客户端。

```java
@Autowried
private DiscoveryClient discoveryClient;

public String homePage(){
  List<ServiceInstance> list = discoveryClient.getInstance("STORES");
  if(null =! list && !list.isEmpty()){
    return list.get(0).getUri();
  }
  return null;
}
```

### 1.10 为什么注册服务的时候会很慢

因为实例会周期性的每隔30S执行一次心跳操作。如果心跳操作失败，因为服务端和客户端都会在本地缓存service服务的信息，那么在心跳检测周期内该service可能是不可用的。可以通过配置`eureka.instance.leaseRenewalIntervalInSeconds`来改变默认值。

> 注:
>
> 因为服务端和客户端都会缓存注册的服务的信息到本地，所以如果注册的服务已经停止运行之后，那么在三个心跳周期类这个服务的状态在Eureka还是 UP，但是实际上已经 DOWN 了。这样客户端直接调用Service的时候可能不能够及时切换，为了能快速响应可以通过调整上述的参数来加快服务 DOWN 掉被发现的几率。

### 1.11 Zones

如果你讲EurekaClient发布到多个区，并且希望优先消费这个区内的服务那么需要直接配置Eureka 客户端。

首先你必须确保每个Zone都有部署Eureka Server，并且不同Zone的Eureka Server能相互访问。

接下来告诉Eureka你说访问的服务在哪个区。未达到这个目的可以通过配置`metadataMap`来实现。例如`service-1`在`zone1`和`zone2`都有发布，那么需要如下配置:

Service in Zone 1

```properties
eureka.instance.matadataMap.zone=zone1
eureka.client.perferSameZoneEureka=true
```

Service in Zone 2

```Properties
eureka.instance.matadataMap.zone=zone2
eureka.client.perferSameZoneEureka=true
```

## 2 Eureka Server

### 2.1 引入Eureka Server

引入Eureka Server需要在系统依赖中引入：`org.springframework.cloud`的`spring-cloud-starter-netflix-eureka-server`。

### 2.2 如何运行Eureka Server

```java
@SpringBootApplication
@EnableEurekaServer
public class Application{
  public static void main(String[] args){
    new SpringApplicationBuilder(Appliaction.class).web(true).run(args);
  }
}
```

服务端有一个 UI 界面，同时有一个 HTTP API 的服务接口，通常暴露在`/eureka/*`

### 2.3 高可用:Zones & Regions

Eureka Server是不存在物理存储的。但是各个注册了的服务实体必须定时发送心跳信息以保证服务在 EurekaServer 的状态为UP。同时在各个客户端也会在本地缓存数据。

默认情况下每个 Eureka Server 自身也是一个Eureka client，也需要至少一个 server URL用来确定自己的服务地址。

### 2.4 独立模式

在独立模式下需要关闭服务本身客户端模式的行为(企图从其他的Eureka节点同步数据、发送心跳等)。

application.properties

```Properties
spring.application.name=Eureka-Server
server.port=8761
eureka.instance.hostname=localhost
eureka.client.registerWithEureka=false
eureka.client.fetchRegistry:false
eureka.client.serviceUrl.detaultZone=http://${eureka.instance.hostname}:${server.port}/eureka/
```

### 2.5 点对点感知

Eureka 可以多机部署，各节点间相互注册，从而来保证 Eureka 的高可用性。这也是 Eureka 的默认行为。

例如:

eureka server 1

```properties
spring.application.name=Eureka-Server-1
server.port=8761
eureka.instance.hostname=eureka-server-1
#有坑建议使用IP
eureka.client.serviceUrl.defaultZone=http://eureka-server-2/eureka
```

eureka serever 2

```Properties
spring.application.name=Eureka-Server-2
server.port=8762
eureka.instance.hostname=eureka-server-2
#有坑建议使用IP
eureka.client.serviceUrl.defaultZone=http://eureka-server-1/eureka
```

## 3 断路器 Circuit Breaker:Hystrix Clients

### 3.1 如何引入Hystrix

需要引入Hystrix需要在依赖中添加`org.spring.framework.cloud`下的`spring-cloud-starter-netflix-hystrix`。

```Java
@SpringBootApplication
@EnableCircuitBreaker
public class Application {

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).web(true).run(args);
    }

}

@Component
public class StoreIntegration {

    @HystrixCommand(fallbackMethod = "defaultStores")
    public Object getStores(Map<String, Object> parameters) {
        //do stuff that might fail
    }

    public Object defaultStores(Map<String, Object> parameters) {
        return /* something useful */;
    }
```

## 4 客户端负载均衡:Ribbon

Ribbon是一个客户端侧的负债均衡组件，能给你许多针对HTTP和 TCP 的操作。Feign已经使用了Ribbon，所以如果使用了`@FeignClient`那么该特性自动包含。

Ribbon的核心概念就是**命名的客户端**(named client)。每个负载均衡器都是一个通远程服务通信组件的一部分。这个组件在开发阶段会被给定一个名称，例如:(`@FeignClient`)。Spring Cloud为每一个使用`RibbonClientConfiguration`命名的组件创建一个新的`ApplicationContext`。这个Context包含一个`ILoadBalancer`,一个`RestClient`以及一个`ServerListFilter`。

### 4.1 如何引入 Ribbon

引入`org.springframework.cloud`的`spring-cloud-starter-netflix-ribbon`依赖。

### 4.2 定制Ribbon客户端

可以通过外部配置一些参数来定制 Ribbon，例如:`<client>.ribbon.*`。除此之外，也可以使用SpringBoot的配置文件。

Spring Cloud允许用户完全自定义相关配置使用`@RibbonClient`注解。

```java
@Configuration
@RibbonClient(name="foo",configuration=FooConfiguration.class)
public class TestConfiguration{}
```

这种情况下，客户端是`RibbonClientConfiguration`组件的一部分。

> `FooConfiguration`必须是`@Configuration`。但需要注意的是它不能在`@Component`的范围内，否则它将被共享给所有的`@RibbonClients`。如果使用了`@ComponentScan`或者`@SpringbootApplication`，你需要去阻止它被包含到全局的扫描域中。通常你可以：
>
> 1. 新建一个独立的不会被扫描的包；
> 2. 在`@ComponentScan`中指定排除这个地址。

Spring Cloud Netflix提供了如下的Bean默认供Ribbon使用{BeanType:ClassNames}:

- `IClientConfig` 代表ribbonClientConfig,对应于`DefaultClientConfigImpl`;
- `IRule` 代表ribbonRule，对应于`ZoneAvoidanceRule`;
- `IPing`代表ribbonPing,对应于`DummyPing`;
- `ServerList<Server>` 代表ribbonServerList，对用于`ConfigurationBasedServerList`;
- `ServerList Filter<Server>` 代表ribbonServerListFilter,对应于`ZonePrefernceServiceListFilter`;
- `ILoadBalancer`代表ribbonLoadBalancer,对应于`ZoneAwareLoadBalancer`;
- `ServerListUpdater`代表ribbonServerListUpdater,对应于`PollingServerListUpdater`。

创建上述任意类型的Bean同时配置在`@RibbonClient`中，例如上述的`FooCongfiguration`可以来重写每个Bean的定义。

```java
@Configuration
public static class FooConfiguration{
  @Bean
  Public ZonePrefernceServiceListFilter serverListFilter(){
    ZonePreferenceServerListFilter filter = new ZonePreferenceServerListFilter();
    filter.setZone("user-service");
    return filter;
  }
  @Bean
  public IPing ribbonPing(){
    return new PingUrl();
  }
}
```

这个例子将`NoOpPing`替换为`PingUrl`，并且定制化了`ZonePreferenceServerListFilter`。

### 4.3 为所有 Ribbon Client定制化默认配置

```java
@RibbonClients(defaultConfiguration = DefaultRibbonConfig.class)
public class RibbonClientDefaultConfigurationTestsConfig {

	public static class BazServiceList extends ConfigurationBasedServerList {
		public BazServiceList(IClientConfig config) {
			super.initWithNiwsConfig(config);
		}
	}
}

@Configuration
class DefaultRibbonConfig {

	@Bean
	public IRule ribbonRule() {
		return new BestAvailableRule();
	}

	@Bean
	public IPing ribbonPing() {
		return new PingUrl();
	}

	@Bean
	public ServerList<Server> ribbonServerList(IClientConfig config) {
		return new RibbonClientDefaultConfigurationTestsConfig.BazServiceList(config);
	}

	@Bean
	public ServerListSubsetFilter serverListFilter() {
		ServerListSubsetFilter filter = new ServerListSubsetFilter();
		return filter;
	}

}
```

### 4.4 通过配置文件来配置Ribbon客户端

可用参数(前缀为:`<clientname>.ribbon.`):

- `NFLoadBalancerClasName`—>`ILoadBalancer`；
- `NFLoadBalancerRuleClassName`—>`IRule`;
- `NFLoadBalancerPingClassName`—>`IPing`;
- `NIWSServerListClassName`—>`ServerList`;
- `NIWServerListFilterClassName`—>`ServerListFilter`

例如:

```properties
users.ribbon.NIWSServerListClassName=com.netfilix.loadbalancer.NIWSServerListClassName
users.ribbon.NFLoadBalancerRuleClassName=com.netfilix.loadbalancer.WeightedResponseTimeRule
```

### 4.5 Using Ribbon with Eureka

当Eureka和Ribbon同时存在于*classpath*。

1. `ribbonServerList`的实现会被重写为`DiscoveryENabledNIWSServerList`，该实现会从Eureka获取已注册服务的列表。
2. `Iping` 接口被`NIWSDiscoveryPing`代替，该类委托Eureka判断对应服务是否是 UP 状态。
3. `ServerList`被替换为`DomainExtractingServerList`，改实现可以使loadBalancer使用本地化的配置而不是远程调用 AWS AMI。默认情况下,serverlist在创建的是否会根据应用原始数据中的*zone*来创建(*eureka.instance.metadataMap.zone*);

#### 不和Eureka同时使用

`search.ribbon.listOfServers=www.baidu.com,www.google.com,www.bing.com,wwww.yahoo.com`

#### 关闭Ribbon配合Eureka的功能

`ribbon.eureka.enabled=false`

### 4.6 使用Ribbon API

```java
public class UserService{
  @Autowired
  private LoadBalancerClient loadBalancer;
  public void do(){
    ServiceInstance instance = loadBalancer.choose("user-service");
    URI userUri = URI.create(String.format("http://%s:%s),instance.getHost(),instance.getPort());
    //...do something
  }
}
```

## 5 Reign

### 5.1 声明式的 REST 客户端:Reign

`Feign`是个声明式的网络服务客户端。它使网络调用变的更简单。想要使用 Feign 仅需要声明一个接口，然后添加上注解。它支持Feign注解和JAX-RS注解。Feign同样支持可插拔的编码和解码器。Spring Cloud使用同Spring MVC相同的`HttpMessageConvert`。使用Feign的时候Spring Cloud会继承Ribbon和Eureka来实现HTTP客户端的负载均衡。

### 5.2 如何引入 Feign

在依赖条件引入:`org.springframework.cloud`的`spring-cloud-starter-openfeign`。

E.G

```java
@Configuration
@ComponentScan
@EnableAutoConfiguration
@EnableFeignClients
public class Application{
    public static void main(String[] args){
        SpringApplication.run(Application.class,args);
    }
}
```

**StoreClient.java**

```Java
@FeignClient("stores)
public interface StoreClient{
  @RequestMapping(method=RequestMethod.GET,value="/stores")
  List<Store> getStores();
  
  @RequestMapping(method=RequestMethod.POST,value="/stores/{storeId}",consumer="application/json")
  Store update(@PathValue Long storeId,Store store);
 
}
```

`@FeignClient`value属性的值`stores`可以是任意一个客户端名称。它用来生成一个Ribbon负载均衡器。除此之外也可以设置`url`属性用URL的方式来做路由。路上的Bean的命名规则使用spring的默认命名规则，如果需要修改这个名称，可以使用`@FeignClient`的`qualifier`属性。

Ribbon 客户端将会通过服务发现机制来获取"stores"服务的物理地址。如果应用也是Eureka客户端，ribbon将会从 Eureka Service Registry 获取相关服务信息。如果不是Eureka服务，那么需要在配置文件中配置相关参数。

### 5.3 overriding Feign Default

Spring Cloud的Feign最核心的概念就是客户端的命名。每个feign客户端都是整个和远程服务进行数据交换(contract)组件的一部分。也就是说与远程服务进行数据交换的组件可以包含多个feign客户端。因为一个 APP 通常会调用 N 多个外部服务每个外部服务都可以是feign客户端(app(one to many)service)。Spring Cloud会通过`FeignClientsConfiguration`为每个`@FeignClient`上命名的服务新建一个`ApplicationContext`。`FeignClientsConfiguration`包含诸如`feign.Decoder`、`feign.Encoder`、`feign.Contract`等参数。

Spring Cloud允许用户通过定义额外的`configuration`(在`FeignClientsConfiguration`的基础上)来定制`@FeignClient`的所有特性。

```java
@FeignClient(name="stores",configuration=StoresConfiguration.class)
public interface StoreClient{
  
}
```

`@FeignClient`的`name`和`url`参数是支持占位符的:

```java
@FeignClient(name="${feign.name}",url="${feign.url}")
public interface StoreClient{
  
}
```

Spring Cloud Netflix提供了如下的默认实体给feign:

- `Decoder` feignDecoder:`ResponseEntityDecoder`(用来包装`SpringDecoder`);
- `Encoder` feignEncoder:`SpringEncoder`;
- `Logger` feignLogger:`Slf4jLogger`;
- `Contract` feignContract:`SpringMvcContract`;
- `Feign.Builder` feignBuilder:`HystrixFeign.Builder`;
- `Client` feignClient:如果Ribbon被开启对应`LoadBalancerFeignClient`,否则默认的feign client被使用。

OkHttpClient和ApacheHttpClient可以将okhttp或者apache http client依赖关系引入到classpath,再通过分别配置`feign.okhttp.enabled=true`和`feign.httpclient.enabled=true`来开启。在使用 HTTP Client 的时候可以**注入**要么`CloseableHttpClient`(使用apache http client的时候)或者`OkHttpClient`(使用 OkHttp 的时候)。

Spring Cloud Netflix默认不提供如下的类给Feign，但是却会在Context创建feign客户端的时候尝试获取这些类型的Bean。

-  `Logger.Level`
- `Retryer`
- `ErrorDecoder`
- `Request.Options`
- `Collection<RequestInterceptor>`
- `SetterFactor`

在创建`@FeignClient`的时候通过`Configuration`将这些 Bean 通知给Feign，你就可以重写这些特性.

```java
@Configuration
public class MyConf{
  @Bean
  public Contract feignContract(){
      return new feign.Contract.Default();
  }
  
  @Bean
  public BasicAuthRequestInteceptor basicAuthREquestInterceptor(){
      return new BaicAuthRequestInterceptor("username","password");
  }
}
```

这里将`SpringMvcContract`替换为`feign.Contract.Default`，并且给`RequestInterceptor`添加了一个 HTTP 基本授权的拦截器。`@FeignClient`也可以通过配置文件来配置:

```yaml
feign:
  client:
    config:
      stores:
        connectTimeout: 5000
        readTimeout: 5000
        loggerLevel: full
        errorDecoder: com.example.SimpleErrorDecoder
        retryer: com.example.SimpleRetryer
        requestInterceptors:
          - com.example.FooRequestInterceptor
          - com.example.BarRequestInterceptor
        decode404: false
```

如果想做一个全局的默认配置，可以新建一个`default`的feignName:

```yaml
feign:
  client:
    config:
      default:
        connectTimeout: 5000
        readTimeout: 5000
        loggerLevel: basic
```

### 5.4 手动创建Feign Client

```java
@Import(FeignClientsConfiguration.class)
class StroeController{
    private StoreClient storeClient;
    private StoreClient adminClient;
    @Autowried
  	public FooController(Decoder decoder,Encoder encoder,Client client){
        this.storeClient = 
          Feign.builder().client(cllient)
          .encoder(encoder)
          .decoder(decoder)
          .requestInteceptor(new BasicAuthRequestInterceptor("username","password"))
          .target(StoreClient.class,"http://STORE-SERVICE");
      this.storeClient = 
          Feign.builder().client(cllient)
          .encoder(encoder)
          .decoder(decoder)
          .requestInteceptor(new BasicAuthRequestInterceptor("username","password"))
          .target(StoreClient.class,"http://STORE-SERVICE");
    }
}
```

### 5.5 Feign支持Hystrix

如果Hystrix在classpath中并且`feign.hystrix.enabled=true`，Feign将会为所有方法包装一个断路器。返回一个`com.netflix.hystrix.HystrixCommand` 也是可以的。

如果不想使用单例模式生成Hystrix，可以给`Feign.Builder`加上`prototype`的作用域.

```
@Configuration
public class FeignConf{
  @Bean
  @Scope("prototype")
  public Feign.Builder feignBuilder(){
    return Feign.builder();
  }
}
```

#### 5.5.1 Feign Hystrix 回调

Hystrix支持回调的概念:当断路器打开或者存在异常的时候调用的回调函数。要引入回调需要给`@FeignClient`设置`fallback`参数。 

```java
@FeignClient(name = "hello", fallback = HystrixClientFallback.class)
protected interface HystrixClient {
    @RequestMapping(method = RequestMethod.GET, value = "/hello")
    Hello iFailSometimes();
}

static class HystrixClientFallback implements HystrixClient {
    @Override
    public Hello iFailSometimes() {
        return new Hello("fallback");
    }
}
```

如果想要获取具体触发这个 fallback 的堆栈信息，需要使用fallbackFactory`参数:

```java
@FeignClient(name = "hello", fallbackFactory = HystrixClientFallbackFactory.class)
protected interface HystrixClient {
	@RequestMapping(method = RequestMethod.GET, value = "/hello")
	Hello iFailSometimes();
}

@Component
static class HystrixClientFallbackFactory implements FallbackFactory<HystrixClient> {
	@Override
	public HystrixClient create(Throwable cause) {
		return new HystrixClient() {
			@Override
			public Hello iFailSometimes() {
				return new Hello("fallback; reason was: " + cause.getMessage());
			}
		};
	}
}
```

### 5.6 Feign  requet/response 压缩

要使用GZIP特性:

```properties
feign.compression.request.enabled=true
feign.compression.response.enabled=true
feign.compression.request.mime-types=text/xml,application/xml,application/json
feign.compression.request.min-request-size=2048
```

### 5.7 Feign日志

每一个Feign客户端都会关联一个logger.默认logger的名字是`@FeignClient`对应的interface的classNme。Feign logger 仅仅支持 DEBUG 级别。

`logger.level.project.user.UserClient=DEBUG`

可配置的参数如下:

- `NONE`,不打印(默认)
- `BASIC`，仅仅打印请求方法、请求 URL，响应状态和执行时间;
- `HEADERS`，打印requset和response基本的头部信息;
- `FULL` 打印requset和response的header,body,metadata.

```java
@Configuration
public class StoreFeignConf{
  @Bean
  Logger.Level feignLogLeve(){
   return Logger.Level.FULL;
  }
}
```

