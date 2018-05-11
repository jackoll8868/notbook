# Spring Cloud

## 1 Commons

## 1.1 @EnableDiscoveryClient

对于使用了@EnableDiscoveryClient的@Bean,springboot 默认会通过*META-INFO/spring.factories*寻找DiscoryClient的实现。DiscoryClient的实现被配置在*spring.factories* 的org.springframework.cloud.client.discovery.EnableDiscoveryC lient 属性下。DisCoveryClient的实现包括:

NetflixEureka,Spring Cloud Consul Discovery和Spring Cloud Zookeeper Discovery.

默认情况下，添加了@EnableDiscoveryClient的实现会默认项Discovery Server注册。可以通过设置@EnableDiscoveryClient的autoRegister=fasle来阻止这个操作。

### 1.1.1 Health Indicator

DiscoveryClient 的HealthIndicator通常通过DiscoveryHealthIndicator的实现完成。如果需要关闭HealthIndicator可以设置```spring.cloud.discovery.client.composite-indicator.enabled=false```。通常基于 DiscoveryClient 的 HealthIndicator 是自动配置的(DiscoveryClientHealthIndocator)。如果要关闭这个配置可以通过设置`spring.cloud.discovery.client.health-indicator.enabled=false` 来实现。

## 1.2 服务注册

现在spring cloud提供了一个ServiceRegistry的接口，该接口包含register(Registration)和deregister(Registration)的方法。其中Registration是一个marker interface.

```java
@Configuration
@EnableDiscoveryClient(autoRegister=false)
public MyConfiguration{
    private ServiceRegistry registry;
  public MyConfiguration(ServiceRegistry registry){
      this.registry = registry;
  }
  public void registry(){
      Registration registration = constructRegistration();
      this.registry.register(registration);
  }
}
```

每个ServiceRegistry实现拥有他自己的 Registry.参见EurekaServiceRegistry和EurekaRegistration。

通常情况下ServiceRegistry实现会自动注册运行的服务。要关闭这个特性有如下两种方式:

1. ``` @EnableDiscoveryClient(autoRegister=false)```
2. ```spring.cloud.service-registry.auto-registration.enabled=false```

默认/service-registry的服务会被提供。这个服务依赖于spring application context的Registration的实例。通过 GET 方式调用/service-registry/instance-status会返回Registration的status。通过 POST 的方式传递一个 String 类型的值会修改Registration的 status 值。



## 1.3 Spring RestTemplate as a Load Balancer Client

RestTemplate可以自动被配置为使用*ribbon*.创建一个负载均衡的 RestTemplate @Bean 可以使用@LoadBalance注解。

```java
@Conguration
public class MyConf{
  @LoadBalance
  @Bean
  RestTemplate restTemplate(){return new RestTemplate();}
}

public class MyClazz{
    @Autowired
  private RestTemplate restTemplate;
  public User work(){
      User response = restTemplate.getForObject("http://users/user",User.class) 
      return reponse;
  }
}
```

URI必须是虚拟主机名例如服务名，而不是host name.Ribbon 客户端用来创建完全的为地址。详情参见RibbonAutoConfiguration.

### 1.3.1 失败重试

一个负载均衡的RestTemplate可以被配置失败重试。默认情况下这个逻辑是关闭的，如果要开启可以在application classpath中添加Spring Retry依赖。负载均衡的RestTemplate对部分Ribbon配置感兴趣。如果想要关闭重试逻辑，可以配置```springl.cloud.loadbalancer.retry.enabled=false```。另外如下的参数是可以使用的:

-  ```client.ribbon.MaxAutoRetries```;


- ```client.ribbon.MaxAutoRetriesNextServer```
- ```client.ribbion.OkToRetryOnAllOperations```

如果想要实现```BackOffPolicy```需要创建一个```LoadBalancedBackOffPolicyFactory```的实体类并返回```BackOffPolicy```。

```java
@Configuration
public class MyConf{
    @Bean
    LoadBalanceBackOffPolicyFactroy bop(){
        return new LoadBanceBackOffPolicyFactory(){
            @Override
          	public BackOffPolicy createbackOffPolicy(String service){
                return new ExponentialBackOffPolicy();
            }
        };
    }
}
```

如果需要添加一个或者多个```RetryListener```，需要实现LoadBalancedRetryListenersFactory。

```Java
@Configuration
public class MyConfiguration {
    @Bean
    LoadBalancedRetryListenerFactory retryListenerFactory() {
        return new LoadBalancedRetryListenerFactory() {
            @Override
            public RetryListener[] createRetryListeners(String service) {
                return new RetryListener[]{new RetryListener() {
                    @Override
                    public <T, E extends Throwable> boolean open(RetryContext context, RetryCallback<T, E> callback) {
                        //TODO Do you business...
                        return true;
                    }

                    @Override
                     public <T, E extends Throwable> void close(RetryContext context, RetryCallback<T, E> callback, Throwable throwable) {
                        //TODO Do you business...
                    }

                    @Override
                    public <T, E extends Throwable> void onError(RetryContext context, RetryCallback<T, E> callback, Throwable throwable) {
                        //TODO Do you business...
                    }
                }};
            }
        };
    }
}
```

### 1.3.2 多RestTemplate实例

```Java
@Configuration
public class MyConfiguration {

    @LoadBalanced
    @Bean
    RestTemplate loadBalanced() {
        return new RestTemplate();
    }

    @Primary
    @Bean
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

public class MyClass {
    @Autowired
    private RestTemplate restTemplate;

    @Autowired
    @LoadBalanced
    private RestTemplate loadBalanced;

    public String doOtherStuff() {
        return loadBalanced.getForObject("http://stores/stores", String.class);
    }

    public String doStuff() {
        return restTemplate.getForObject("http://example.com", String.class);
    }
}
```

## 1.4 HTTP 客户端工厂

Spring Cloud Commons提供了Apache Http client(ApacheHttpClientFacotry)和OkHttp(OkHttpclientFactory)的实现。OkHttpClientFactory金丹OkHttp.jar在Classpath时才会被创建。同时SpringCloud为两种http client均提供了连接管理类，分别为*ApacheHttpClientConnectionManagerFactory*和*OkHttpClientConnectionPoolFactory*。如果想要实现定制化的。可以通过设置参数来关闭默认创建。

*spring.cloud.httpclientfactories.apache.enabled=fase*

*spring.cloud.httpclientfactoies.ok.enabled=false*

