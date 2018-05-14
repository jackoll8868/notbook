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

## 6 Zuul

### 6.1 Zuul: 路由器&过滤器

​	Route是一个完整的服务项目中必不可少的部分。例如`/`可能对应于你的web服务，`/api/users`对应于用户服务,`/api/shops`对应于商城服务。Zuul 是一个基于JVM的服务端负载均衡器。

Netflix使用zuul做如下的事情:

- 认证(*Authentication*)
- 服务观察(*Insights*)
- 压力测试(*Stress Testing*)
- 局部测试?(*Canary Testing*):A change in programming is pushed to small group of end user after testing in sandbox environment without the end user knowledge & is reversed when the team find the code is having bug.
- 动态路由(*Dynamic Routing*)
- 服务迁移(*Service Migration*)
- 负载均衡(*Load Shedding*)
- 安全控制(Security)
- 静态资源处理(*Static Response handling*)
- 开启/关闭流量管理(*Active/DeActive traffic management)

Zuul 的规则引擎允许用任意基于jvm的语言来实现`rule`和`filter`，但是默认支持`Java`和`Groovy`。

> 注意：
>
> 1. 配置参数`zuul.max.host.connections`已经被拆分为两个参数`zuul.host.maxTotalConnections`和`zuul.host.maxPreRouteConnections`其默认值分别为:200和20。
> 2. 默认所有router的Hytrix隔离模式(*ExecutionIsolationStrategy*)是`SEMAPHORE`(信号量)。`zuul.ribbonIsolationStrategy`可以被修改为`THREAD`(线程)。

### 6.2 如何引入zuul

引入`org.springframework.cloud`的`spring-cloud-starter-neflix-zuul`模块即可。

### 6.3 zuul 内嵌的反向代理(Reverse Proxy)

Spring Cloud已经创建了一个内嵌的Zuul Proxy来将会常用的调用(UI应用想请求调用一个或者多个后端服务)。这种特性对于使用用户自定义接口代理后端服务的场景，这种场景通常是不需要为所有的后端服务单独管理跨域和认证的情景。

在Spring Boot 主程序入口class(main class)加上`@EnableZuulProxy`即可。根据约定，一个名为服务 ID (*serviceID*)为"users"的服务，将会受到来自`/users`的代理请求。该代理使用Ribbon从Eureka Server发现服务的实体，所有的请求都在`hystrix command`中执行，如果请求执行失败会在Hystrix mertics中有所反应，如果断路器已打开，那么代理不会再去请求对应的服务。

默认情况下Zuul starter不包含服务发现客户端，所以要实现根据service id来代理的功能需要在classpath提供一个Discover Client。例如:spring-cloud-starter-netflix-eureka就是个不错的选择。

想要阻止服务被自动添加到代理中，可以在`zuul.ignored-services`中增加service id。如果一个服务匹配到ignore-services但是同时也声明在route map，那么它不会被忽略。例如:

```properties
zuul.ignoredServices='*
zuul.routes.users=/myusers/**
```

在这个例子中，除了users服务外其别的服务都会被忽略。

如果想参数化或者改变路由规则，可以增加外部配置项来实现:

```properties
zuul.routes.users=/myusers/**
```

则表示调用`myusers`的 HTTP 请求将会被转发到`users`服务，例如(`myusers/101`将会被转发到users服务的`/101`)。

想要更精细的控制route，可以指定单独的指定`path`和`serviceId`:

```properties
zuul.routes.users.path=/myusers/**
zuul.routes.users.serviceId=users_service
```

这表示调用`myusers`的 HTTP 请求会被转发到`users_service`服务。`path`支持ant-style的匹配模式，所以`/myusers/*`仅匹配一种模式，但是`/myusers/**`可以匹配继承的模式。

后端服务要么以`serviceId`(从Discovery获取例如Eureka)要么以`url`(物理地址)界定，例如:

```properties
zuul.routes.users.path=/myusers/**
zuul.routes.users.url=http://user-service-stg1/users_service
```

这种简单的url-routes不能在`HystrixCommand`中执行也不能通过`Ribbon`在多 URL 中负载均衡。要达到这个目的，可以在静态服务列表中指定`serviceId`。

```properties
zuul.routes.users.path=/myusers/**
zuul.routes.users.serviceId=myusers-service
zuul.routes.users.stripPrefix=true

hystrix.command.myusers-service.execution.isolation.thread.timeoutInMilliseconds=500
myusers-service.ribbon.NIWSServerListClassName=com.netflix.loadbalancer.ConfigurationBasedServerList
myusers-service.ribbon.ListOfServers=http://user-service.com,http://user-stg1.com
myusers-service.ribbon.ConnectTimeout=1000
myusers-service.ribbon.ReadTimeout=3000
myusers-service.ribbon.MaxTotalHttpConnections=500
myusers-service.ribbon.MaxConnectionsPerHost=100
```

另一种办法是指定一个服务路由并且用serviceId配置Ribbon客户端。

```properties
zuul.routes.users.path=/myusers/**
zuul.routes.users.serviceId=user-service

ribbon.eureka.enabled=false
users.ribbon.listOfServers=user-service.com,user-stg .com
```

除此之外，还可以通过使用正则表达式的方式来实现。例如:

```java
@Configuration
public class ZuulConfig{
    @Bean
    public PatternServiceRouteMapper serviceRouteMapper(){
        return new PatternServiceRouteMapper(
            "(?<name>^.+)-(?<version>v.+$)",
            "${version}/${name}"
        );
    }
}
```

这表示服务Id为`myusers-v1`的服务将会被映射到`/v1/myusers/**`。任意的正则表达式都可以被接收，但是所有被命名的组在`servicePattern`和`routePattern`中都有所体现。如果`servicePattern`不匹配`serviceId`，将会使用默认操作。在上边这个例子中服务Id为`myusers`的服务将会被映射到`/myusers/**`(因为这个Id没有版本号)。这个特性默认是关闭的并且仅仅只支持:服务注册中心模式。

如果要给所有请求都添加一个前缀可以为`zuul.prefix`设置一个值，例如`/api`。默认情况下代理前缀在请求被转发到特定服务之前会被去除掉。关闭掉这个服务可以配置`zuul.stripPrefix=false`。你也可以为每个特定的服务关闭这个功能:

```properties
zuul.routes.users.path=/myusers/**
zuul.routes.users.stripPrefix=false
```

> 注意:`zuul.stripPrefix`仅仅适用于`zuul.prefix`。对于定义在特定的路由的`path`属性没有任何作用。

在上边这个例子中`myusers/101`会被转发到`users` 服务的`myusers/101`而不是`/101`。

`zuul.routes`实体实际上会绑定一个`zuulProperties`。如果你查看这个参数你会发现它还含有一个`retrykable`的标识。如果设置这个参数为`true`ribbon客户端将会自动的重试失败的请求。你也可以修改ribboin客户端相关重试操作的参数。

`X-Forwared-Host` 头会被默认添加到转发请求中。要关闭这个服务可以设置`zuul.addProxyHeads=false`。默认情况下前缀路径会被截取掉，调用后端的服务会获取一个叫`X-Forwarded-Prefix`的头信息，例如:`/myusers`。

一个拥有`@EnableZuulProxy`的应用如果设置默认的路由为`/`,那么它可以作为一个单独的实例运行，例如:`zuul.route.home=/`例如(`/**`)将会转发所有的请求到`home`服务。

如果需要更精细化的忽略规则，可以指定特定的规则。这些规则会在路由地址之前被分析，也就是说前缀也会被包含到你先被匹配的模式中。忽略模式跨越所有服务并且取代任意其他的个性化设置。

```properties
zuul.ignoredPatterns=/**/admin/**
zuul.routes.users=/myusers/**
zuul.routes.users.serviecId=user-services
```

该配置表示`/myusers/101`的请求将会被转发到users对应的`user-services`服务。但是包含`/admin`的将不会被代理。

> 注意：
>
> 如果你想你的路由规则包含顺序性(*order*)那么你必须使用YAML文件，因为使用propreties文件将会丢失顺序。个人认为应该是properties对应的是一个无顺序特性的Map.
>
> 例如:
>
> ```yaml
> zuul:
>  routes:
>   users:
>    path:/myuers/**
>   legacy:
>    path:/**
> ```
>
> 如果你使用properties文件,`legacy`路径可能在`users`路径之前被截止，从而导致`users`服务不能被访问。



### 6.4 Zuul Http Client

zuul默认现在默认使用的 HTTP 客户端是 Apache HTTP Client 用来替代已经过时的Ribbon `RestClient`。如果想要使用`RestClient`或者使用`okhttp3.OkHttpClient`可以分别设置`ribbon.restclient.enabled=true`、`ribbon.okhttp.enabled=true`。如果想要定制化 Apache HTTP客户端或者OK HTTP 客户端，可以提供`ClosableHttpClient`或者`OkHttpClient`的实例Bean。

### 6.5 Cookies 和敏感的头信息

在同一个系统中的各个服务间共享头信息是可行的。但是你可能不希望敏感的头信息被分发到外部服务中。你可以指定一个需要忽略的头信息列表作为route配置的一部分。Cookies因为他们在浏览器中有良好的定义语义，所以他们总是被当做敏感的信息。如果你的代理是一个浏览器，那么这些下游服务的cookies会给用户造成困惑，因为所有的后端服务看起来来自同一个地方。

如果你小心翼翼的设计你的服务，例如:仅仅下游一个服务设置cookies,那么让cookies遵从从后端服务到调用者的规则。如果代理(zuul router)设置cookies，并且所有的后端服务都是同一个系统中的，那么也可以单纯的共享他们。初次之外，任何下游服务设置的cookies都不能被调用者使用。所以，请记住将`Set-Cookie`(至少)和`Cookie`加入到敏感头信息的router不是你服务中的一部分。即使router是服务的一部分，也要仔细考量。

敏感头信息可以参考如下配置:

```properties
zuul.routes.users.path=/myusers/**
zuul.routes.users.sensitiveHeaders=Cookie,Set-Cookie,Authorization
zuul.routes.users.serviceId=user-services
```

以上的敏感头信息是默认配置项，因此默认情况下zuul也不会转发这些头信息，如果你想要传递所有的头信息需要设置一个空的`sensitiveHeaders`:

```properties
zuul.routes.users.sensitiveHeaders=
```

### 6.6 管理服务(Endpoints)

如果`@EnableZuulProxy`和Spring Boot Actuator配合使用，那么摸了将会开启两个额外的服务:

- Routes
- Filters

#### 6.6.1 Routes EndPoints

 如有服务的`/routes`的 GET 请求方式将会返回一个映射的路由列表:

GET /routes

```json
{
    /stores/**:"http://localhost:8001"
}
```

额外的路由详情可以通过添加请求参数`?format=details`到`/routes`之后:

GET /routes?format=details

```json
{
  "/stores/**": {
    "id": "stores",
    "fullPath": "/stores/**",
    "location": "http://localhost:8081",
    "path": "/**",
    "prefix": "/stores",
    "retryable": false,
    "customSensitiveHeaders": false,
    "prefixStripped": true
  }
}
```

该路径的 POST 请求将会刷新已存在的路由。可以通过配置`endpoints.routes.enabled=false`来关闭这个服务。

#### 6.6.2 Filters Endpoint

`/filters`的GET请求将会返回一个zuul filter类型的map。

### 6.7 压缩模式和本地跳转

迁移一个已存在的应用或者 API 的通常做法是"勒死"老的服务，然后慢慢替换不同的实现。Zuul Proxy 是这种场景的很不错的工具，因为他可以处理所有老服务的流量，并转发部分请求到新的服务。例如:

application.yml

```yaml
zuul:
 routes:
  first:
   path:/first/**
   url:http://one.example.com
  second:
   path:/second/**
   url:forward:/second
  third:
   path:/third/**
   url:forward:/3rd
  legacy:
   path:/**
   url:http://legacy.example.com
   
```

 在这个例子中我们使用`legacy`实例来处理所有不能匹配其他规则的请求。`/first/**`路径下的请求被提取到一个新的 URL。`/second/**`下的服务也被转发，所以他们可以被本地化处理。例如一个普通的Spring`RequestMapping`。`/third/**`下的地址也被转发了，但是他们拥有不同的前缀，例如(`/third/foo`被转发到`/3rd/foo`)。

### 6.8 通过zuul上传文件

如果你使用`@EnableZuulProxy`你可以用来代理路径来上传文件。他仅仅支持上传文件很小的场景。对于大文件需要一个额外的路径来绕过 Spring的`DispatcherServlet到`/zuul/*路径下。例如是`zuul.routes.customers=/customers`,那么你可以通过 POST 方式发送到文件到`zuul/coustomers/`。Servlet路径是通过`zuul.servletPath`来具体化的。除此之外，大文件也需要增加超时时间的设置，如果proxy使用ribbon负载均衡器等。

application.yml

```Yaml
hystrix.command.default.execution.isolation.thread.timeoutInMillisecond:60000
ribbon:
 connectTimeout:3000
 ReadTimeout:600000
```

### 6.9 原始内嵌的zuul

你也可以运行Zuul 服务而不需要代理，或者有选择的开启代理平台。如果你使用`@EnableZuulServer`（来取代`@EnableZuulProxy`）。任何你加载到应用中的`ZuulFilter`实例都将被自动加载。如果他们是运行在`@EnableZuulProxy`项目中，任何的proxy filter都不会被自动加载。

在这种情况下，进入zuul server的route，依旧由`zuul.routes.*`的配置项来管理，但是没有服务发现和代理(客户端代理也就是客户端负载均衡)，所以`serviceId`和`url`配置会被忽略。例如:

```properties
zuul.routes.api=/api/**
```

 该配置表示所有'/api/**'路径的请求将会被映射到Zuul  filter chain.

### 6.10 关闭Zuul Filters

在Spring Cloud体系中Zuul在代理模式或者服务模式下都会自启动很多`ZuulFilter`。如果你想关闭某个Filter，可以设置`zuul.<SimpleClass name>.<filterType>.disable=true`。在`filters`包目录下的都是 `zuulFilters`。所以设置`zuul.SendResponseFilter.post.disable=true`可以关闭`org.srpingframework.cloud.netflix.zuul.filters.post.SendResponseFilter`。

### 6.11 为Routers提供Hystrix Fallbacks

当一个Zuul的router断路器打开了，你可以声明一个`ZuulFallbackProvider`的实体来创建一个Hystrix fallback.在这个实体内部，你需要指定fallback对应的route的 ID 并且提供一个返回`ClientHttpResponse`的方法给fallback。

```java
class MyZuulFallbackProvider implements ZuulFallbackProvider{
    @Ovveride
    public String getRoutes(){
        return "customers";
    }
    
    @Override
    public ClientHttpResponse fallbackResponse(){
        return new ClientHttpResponse(){
          @Override
            public HttpStatus getStatusCode() throws IOException{
                return HttpStatus.OK;
            }
            @Override
            public int getRawStatusCode() throws IOException{
                return 200;
            }
            @Override
            public String getStatusText() throws IOException{
                return "OK";
            }
            @Override
            public void close(){
                
            }
            @Override
            public InputStream getBody() throws IOException{
                return new ByteArrayInputStream("fallback".getBytes());
            }
            @Override
            public HttpHeads getHeaders(){
                HttpHeaders headers = new HttpHeaders();
                headers.setContentType(MediaType.APPLICATION_JSON);
                return headers;
            }
        };
    }
    
}
```

这个例子fallback如下的route:

```properties
zuul.routes.customers=/customers/**
```

如果你想给所有的routes提供一个默认的fallback，可以提供一个`getRoute()`返回`*`的`ZuulFallbackProvider`的实体。

```java
class MyFallbackProvider implements ZuulFallbackProvider {
    @Override
    public String getRoute() {
        return "*";
    }

    @Override
    public ClientHttpResponse fallbackResponse() {
        return new ClientHttpResponse() {
            @Override
            public HttpStatus getStatusCode() throws IOException {
                return HttpStatus.OK;
            }

            @Override
            public int getRawStatusCode() throws IOException {
                return 200;
            }

            @Override
            public String getStatusText() throws IOException {
                return "OK";
            }

            @Override
            public void close() {

            }

            @Override
            public InputStream getBody() throws IOException {
                return new ByteArrayInputStream("fallback".getBytes());
            }

            @Override
            public HttpHeaders getHeaders() {
                HttpHeaders headers = new HttpHeaders();
                headers.setContentType(MediaType.APPLICATION_JSON);
                return headers;
            }
        };
    }
}
```

如果想要实现返回基于失败原因的`response`的`FallbackProvider`，可以：

```java
class MyFallbackProvider implements FallbackProvider {

    @Override
    public String getRoute() {
        return "*";
    }

    @Override
    public ClientHttpResponse fallbackResponse(final Throwable cause) {
        if (cause instanceof HystrixTimeoutException) {
            return response(HttpStatus.GATEWAY_TIMEOUT);
        } else {
            return fallbackResponse();
        }
    }

    @Override
    public ClientHttpResponse fallbackResponse() {
        return response(HttpStatus.INTERNAL_SERVER_ERROR);
    }

    private ClientHttpResponse response(final HttpStatus status) {
        return new ClientHttpResponse() {
            @Override
            public HttpStatus getStatusCode() throws IOException {
                return status;
            }

            @Override
            public int getRawStatusCode() throws IOException {
                return status.value();
            }

            @Override
            public String getStatusText() throws IOException {
                return status.getReasonPhrase();
            }

            @Override
            public void close() {
            }

            @Override
            public InputStream getBody() throws IOException {
                return new ByteArrayInputStream("fallback".getBytes());
            }

            @Override
            public HttpHeaders getHeaders() {
                HttpHeaders headers = new HttpHeaders();
                headers.setContentType(MediaType.APPLICATION_JSON);
                return headers;
            }
        };
    }
}
```

### 6.13 Zuul  超时机制

#### 6.13.1 服务发现配置

如果Zuul使用服务发现机制，那么有两个超时时间需要配置。

1. Hystrix timeout，因为所有routes默认都被封装在Hystrix  Commands；
2. Ribbon Timeout。

Hystrix的超时时间需要考虑到:Ribbon 超时时间+所有重试所消耗的时间。默认情况下Spring Cloud Zuul会默认计算好Hystrix超时时间，除非你单独指定Hystrix超时时间。

Hystrix 超时时间按如下公式计算:

```
(ribbon.ConnectionTimeout+ribbon.ReadTimeout)*(ribbon.MaxAutoRetries+1)*(ribbon.MaxAutoRetriesNextServer+1)
```

例如，如下配置的Hystrix超时时间为:`2400ms`。

```yaml
ribbon:
	ReadTimeOut:100
	ConnectionTimeout:500
	MaxAutoRetries:1
	MaxAutpRetriesNextServer:1
```

> 注意:
>
> 1. 你可以通过配置`service.ribbon.*`来为每个单独的route设置Hystrix超时时间。
> 2. 如果你不配置超时时间，默认情况下将会是:`4000ms`。

如果你设置`hystrix.command.commandKey.execution.isolation.thread.timeoutInMilliseconds`，其中`commandKey`代表route id，或者设置`hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds`那么这些字将会被用作Hystrix超时时间同时会忽略你配置的`ribbon.*`的相关超时参数。

#### 6.13.2 URL 配置

如果你通过URL的方式来配置Zuul Routes。设置`zuul.host.connection-timeout-mills`和`zuul.host.socket-timeout-millis`。

### 6.14 重写`Location Header`

如果Zuul当做一个前置的WebApplication，那么可能需要重写`Location`来应对通过`3xx`的http状态码进行跳转的需求。如果不重写，浏览器不会跳转到实际的web application url，而是在zuul url处停止跳转。`LocationRewriteFilter`可以用来重写Location Header为zuul的url。

```java
import org.springframework.cloud.netflix.zuul.filters.post.LocationRewriteFilter;
...

@Configuration
@EnableZuulProxy
public class ZuulConfig {
    @Bean
    public LocationRewriteFilter locationRewriteFilter() {
        return new LocationRewriteFilter();
    }
}
```

### 6.15 zuul 开发者指南

#### 6.15.1 Zuul Servlet

Zuul 被实现为Servlet。通常情况下，zuul内嵌到Spring的Dispatch机制。这样允许Spring MVC控制路由。在这种场景下，Zuul被用来缓存请求。如果需要直接走Zuul而不是走请求缓存的路径(例如:大文件上传)，Servlet也在Spring Dispatcher之外被转载，默认情况下，这个路径为`/zuul`，可以通过`zuul.servlet-path`来修改。

#### 6.15.2 Zuul RequestContext

 想要在各个Filter之间传递信息，可以使用`RequestContext`。他存储的数据被保存在`ThreadLocal`中，是请求唯一的，即每个request对应它只的数据。这些信息包括**将请求路由到哪里**、**错误**以及实际的**HttpServletRequest**和**HttpServletResponse**。`RequestContext`继承制`ConcurrentHashMap`所以任何数据都可以被缓存在这里。`FilterConstants`包含被Spring Cloud Netflix使用的Key。

#### 6.15.3 @EnableZuulProxy VS. @EnabeleZuulServer

Spring Cloud Netflix 默认加载的filter, 在@EnableZuulProxy的情况下其加载的filter是@EnableZuulServer的超集。换句话说，`EnableZuulProxy`包含`@EnableZuulServer` 所有的Filter。

#### 6.15.4 @EnableZuulServer Filters

创建`SimpleRouteLocator`会默认加载springboot 配置下定义的route。

下列的Filter会被加载:

Pre Filters:

- `ServletDetectionFilter`:判断当前这个请求是否通过Spring Dispatch传递过来。可以设置通过`FilterConstants.IS_DISPATCHER_SERVLET_REQUEST_KEY`实现。
- `FormBodyWrapperFilter`:请求参数封装；
- `DebugFilter`:如果`debug`请求的参数被设置，这个filter设置`RequestContext.setDebugRouteing()`和`RequestContext.setDebugRequest()`为true;

Route Filter:

- `SendForwardFilter`:这个filter使用Servlet`RequestDispatcher`转发请求。转发地址存储在`RequsetContext`的`FilterConstants.FOWRARD_TO_KEY`参数中。这个对于转发请求到本地服务的方式很便利;

POST Filter:

- `SendResponseFilter`:将代理请求的response写到当前response.

 Error Filter:

- `SendErrorFilter`: 默认情况下如果`RequestContext.getThrowable()`是null，会被转发到`/error`路径下。默认的`/error`可以通过`error.path`参数配置。

#### 6.15.5 `@EnableZuulProxy` Filters

 创建一个`DiscoveryClientRouteLocator`会通过`DiscoveryClient`和`propperties`加载相关路由。`DiscoveryClient`会为每个`serviceId`创建一个route。当新的服务被添加，routes会被重新刷新。

默认加载的Filter:

PRE Filters:

- `PreDecorationFilters`:这个filter根据`RouteLocator`来判断代理到何处以及如何代理。同时它还会为下游的服务设置各种而言的头部信息。

Route Filters:

- `RibbonRoutingFilter`: 这个Filter使用Ribbon和Hystrix以及配置的HTTP客户端来发送请求。Service Id 可以通过`RequestContext`的`FilterConstants.SERVICE_ID_KEY`来获取。这个Filter可以使用不同的HTTP CLIENT:
  -  Apache `HttpClient`: 默认使用;
  - `OkHttpClient` V3，添加`com.squareup.okhttp:okhttp`依赖同时设置`ribbon.okhttp.enabled=true`。
  - Netflix Ribbon Http Client.设置`ribbon.restclient.enabled=true`即可使用。这种客户端有一定的局限性，例如不支持PATCH方法，但是它支持重试。
- `SimpleHostRoutingFilter`：这个Filter通过Apache  HttpClient 发送请求到预处理的URL。URL可以在`RequestContext.getRouteHost()`获取。

#### 6.15.6 自动以Zuul Filter

参见:[Sample Zuul Filters](https://github.com/spring-cloud-samples/sample-zuul-filters)。

#### 6.15.7 如何编写Pre Filter

```java
public class QueryParamPreFilter extends ZuulFilter {
	@Override
	public int filterOrder() {
		return PRE_DECORATION_FILTER_ORDER - 1; // run before PreDecoration
	}

	@Override
	public String filterType() {
		return PRE_TYPE;
	}

	@Override
	public boolean shouldFilter() {
		RequestContext ctx = RequestContext.getCurrentContext();
		return !ctx.containsKey(FORWARD_TO_KEY) // a filter has already forwarded
				&& !ctx.containsKey(SERVICE_ID_KEY); // a filter has already determined serviceId
	}
    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
		HttpServletRequest request = ctx.getRequest();
		if (request.getParameter("foo") != null) {
		    // put the serviceId in `RequestContext`
    		ctx.put(SERVICE_ID_KEY, request.getParameter("foo"));
    	}
        return null;
    }
}
```

这个 Filter通过request parameter的`foo`参数来获取`SERVER_ID`。实际中，是不允许这种直接映射的方式的。最好是从`foo`的值获取。

因为`SERVICE_ID`一旦被设置那么`PreDecorationFilter`不会被执行但是`RibbonRoutingFilter`会执行。如果想要路由到一个完整的URL，调用`ctx.setRouteHost(url)`。

#### 6.15.8 如何编写Route Filter

Route filters在pre filters之后执行，用来为其他服务创建请求。通常情况下这类filter根据客户的请求方式转换Request和Response。

```java
public class OkHttpRoutingFilter extends ZuulFilter {
	@Autowired
	private ProxyRequestHelper helper;

	@Override
	public String filterType() {
		return ROUTE_TYPE;
	}

	@Override
	public int filterOrder() {
		return SIMPLE_HOST_ROUTING_FILTER_ORDER - 1;
	}

	@Override
	public boolean shouldFilter() {
		return RequestContext.getCurrentContext().getRouteHost() != null
				&& RequestContext.getCurrentContext().sendZuulResponse();
	}

    @Override
    public Object run() {
		OkHttpClient httpClient = new OkHttpClient.Builder()
				// customize
				.build();

		RequestContext context = RequestContext.getCurrentContext();
		HttpServletRequest request = context.getRequest();

		String method = request.getMethod();

		String uri = this.helper.buildZuulRequestURI(request);

		Headers.Builder headers = new Headers.Builder();
		Enumeration<String> headerNames = request.getHeaderNames();
		while (headerNames.hasMoreElements()) {
			String name = headerNames.nextElement();
			Enumeration<String> values = request.getHeaders(name);

			while (values.hasMoreElements()) {
				String value = values.nextElement();
				headers.add(name, value);
			}
		}

		InputStream inputStream = request.getInputStream();

		RequestBody requestBody = null;
		if (inputStream != null && HttpMethod.permitsRequestBody(method)) {
			MediaType mediaType = null;
			if (headers.get("Content-Type") != null) {
				mediaType = MediaType.parse(headers.get("Content-Type"));
			}
			requestBody = RequestBody.create(mediaType, StreamUtils.copyToByteArray(inputStream));
		}

		Request.Builder builder = new Request.Builder()
				.headers(headers.build())
				.url(uri)
				.method(method, requestBody);

		Response response = httpClient.newCall(builder.build()).execute();

		LinkedMultiValueMap<String, String> responseHeaders = new LinkedMultiValueMap<>();

		for (Map.Entry<String, List<String>> entry : response.headers().toMultimap().entrySet()) {
			responseHeaders.put(entry.getKey(), entry.getValue());
		}

		this.helper.setResponse(response.code(), response.body().byteStream(),
				responseHeaders);
		context.setRouteHost(null); // prevent SimpleHostRoutingFilter from running
		return null;
    }
}
```

#### 6.15.9 如何编写Post Filters

Post Filters主要操作response，下边的例子我们添加了一个随机的`UUID`和`X-Foo`头信息。

```java
public class AddResponseHeaderFilter extends ZuulFilter {
	@Override
	public String filterType() {
		return POST_TYPE;
	}

	@Override
	public int filterOrder() {
		return SEND_RESPONSE_FILTER_ORDER - 1;
	}

	@Override
	public boolean shouldFilter() {
		return true;
	}

	@Override
	public Object run() {
		RequestContext context = RequestContext.getCurrentContext();
    	HttpServletResponse servletResponse = context.getResponse();
		servletResponse.addHeader("X-Foo", UUID.randomUUID().toString());
		return null;
	}
}
```



#### 6.15.10 Zuul 错误如何执行

如果在Zuul Filter执行期间有异常抛出，error filter会被执行。`SendErrorFilter`仅在`RequestContext.getThrowable()`不为空的情况下执行。然后它会指定`javax.servlet.error.*`参数到request然后将request转发到Spring  Boot 的错误页面。

#### 6.15.11 Zuul Eager Applicaiton Context Loading

Zuul内部使用的Ribbon客户端默认是使用lazy load的。如果要修改这个行为，可以使用:

`zuul.ribbon.eager-load.enabled=true`。

