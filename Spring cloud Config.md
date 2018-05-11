# Spring cloud Config

Spring Cloud Config提供服务端和客户端级别的分布式个性化配置系统。在 Config Server 你可以集中化管理所有环境中所有application的个性化配置。客户端和服务端都有对Spring *Environment*和*PropertySource*的抽象，所以他们能很好的适配Spring应用，同事也能应用于其他任意语言的程序。当一个项目从开发(*dev*)和测试(*test*)阶段转移到生产阶段，你可以管理这些配置，同时保证应用所需要的所有信息都被移植。默认情况下Config Server采用git来作为存储，这样方便进行标签化的版本管理。当然也可以通过配置来添加其他模式的存储方案，例如:Subversion。

## 1. Quick Start

Demo data:

```json
curl localhost:8888/foo/development
{"name":"foo","label":"master","propertySources":[
  {"name":"https://github.com/scratches/config-repo/foo-development.properties","source":{"bar":"spam"}},
  {"name":"https://github.com/scratches/config-repo/foo.properties","source":{"foo":"bar"}}
]}
```

默认的确认property文件的策略是，从git仓库(*spring.cloud.config.server.git.uri*)克隆配置文件，然后用来初始化一个迷你的*SpringApplication*。这个迷你应用的*Environment*通过一个 JSON 服务来列举配置文件信息。

HTTP 服务有如下的格式:

```
/{application}/{profile}[/{label}]
/{application}-{profile}.yml
/{label}/{application}-{profile}.yml
/{application}-{profile}.properties
/{label}/{application}-{profile}.properties
```

- 其中{application}对应于SpringApplication的*spring.config.name*配置参数(通常在application.properties中配置);
- {profile}对应于spring.profiles.active参数;
- {label}对应于git的标签，默认为:master

一个从git仓库拉取客户端配置的服务通常配置参数如下:

```properties
spring.cloud.config.server.git.uri=https://github.com/jackoll8868/spring-cloud-demo-config
spring.cloud.config.server.git.search-paths=/**
spring.cloud.config.label=master
spring.cloud.config.server.git.username=***
spring.cloud.config.server.git.password=***
```

### 1.1 客户端建议

客户端仅需要创建一个springboot项目然后依赖*spring-cloud-config-client* 即可。最方便的方式是添加*spring-cloud-starter-parent* *pom*的依赖。

```xml
<parent>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-parent</artifactId>
       <version>1.5.10.RELEASE</version>
       <relativePath /> <!-- lookup parent from repository -->
   </parent>

<dependencyManagement>
	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-dependencies</artifactId>
			<version>Edgware.SR2</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>
</dependencyManagement>

<dependencies>
	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-config</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-test</artifactId>
		<scope>test</scope>
	</dependency>
</dependencies>

<build>
	<plugins>
           <plugin>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-maven-plugin</artifactId>
           </plugin>
	</plugins>
</build>
```

然后创建一个springboot应用:

```java
@SpringBootApplication
@RestController
public class Application{
  @RequestMapping("/")
  public String home(){
    return "Hello world";
  }
  
  public static void main(String[] args){
    SpringApplication.run(Application.class,args);
  }
}
```

当程序运行的时候，程序内部会默认从*localhost:8888*的服务地址获取程序的配置信息(如果config server有运行的话)。如果想修改这个地址可以修改:*bootstrap.properties*(类似于*applicaiton.properties*):

> *spring.cloud.config.uri=http://myconfig-server.com*

*bootstrap.properties*会通过*/env*服务暴露出来

## 2 Config Server

服务端提供一个HTTP,resource-basedAPI 来提供配置。将 ConfigServer集成到springboot application仅仅需要添加*@EnabelConfigServer*注解即可。

**ConfigServer.java**

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServer{
  public staic void main(String[] args){
    SpringApplication.run(ConfigServer.class,args);
  }
}
```

**application.properties**

```properties
spring.application.name=config-server
server.port=8888
spring.cloud.config.server.git.uri=https://github.com/jackoll8868/spring-cloud-demo-config
spring.cloud.config.server.git.search-paths=/**
spring.cloud.config.label=master
spring.cloud.config.server.git.username=***
spring.cloud.config.server.git.password=***

```

### 2.1 Environment  仓库

你想把*ConfigServer*的配置数据存储到哪里？统一管理的策略是：*EnviromentRepository*(为*Enviroment*对象服务)。这里所指的*Enviroment*仅仅是浅显的从Spring *Envioment*拷贝。

其中:

- {application}映射于客户端的*spring.application.name*;
- {profile}映射于客户端的*spring.profiles.active*;
- {label}是一个服务端的版本控制特性。

仓库的实现，通常情况下功能和springboot应用加载配置文件，获取*spring.config.name*和{application}相等以及*spring.profiles.active*和{profile}相等的参数。

例如:一个客户端程序拥有如下的配置:

**bootstrap.properties**

```Properties
spring.application.name=foo
spring.profiles.active=dev,mysql
```

如果仓库是文件类型的，服务器端将会根据application.properties(所有客户端共享的)和foo.properties创建*Enviroment*。如果配置文件内部还有只想其他spring profile的文件，这些文件将被优先读取，同时如果存在profile-specific的配置文件他们同样拥有比默认的配置文件更高的读取优先级。高优先级的文件会被转换到*Enviroment* 的*PropertySource*列表中。

### 2.2 Git仓储

*EnviromentRepository*的默认实现是根据 Git 来实现的。因为使用 Git 能方便的管理更新以及物理环境，同时还方便查询更新记录。想要修改仓库的地址可以通过修改服务端的*spring.cloud.config.git.uri*参数来实现。如果该参数以*file:*开头，那么服务端将从本地仓库获取数据(这种模式不会从本地仓库clone，而是直接在本地仓库操作)。为了使服务端高响应和高可用，你必须使服务端的所有实例都指向同一个仓库，所以一个共享文件系统就可以实现。

### 模式匹配和多仓储

示例如下:

```Properties
spring.cloud.config.server.git.uri=https://github.com/spring-cloud-samples/config-repo
spring.cloud.config.server.git.repos.simple=https://github.com/simple/config-repo
spring.cloud.config.server.git.repose.special.pattern=special*/dev*,*special*/dev*
spring.cloud.config.server.git.repose.special.uri=https://github.com/special/config-repo
spring.cloud.config.server.git.repose.local.uri=file:/home/Documents/config-repo
spring.cloud.config.server.git.repose.special.pattern=local*
```

如果{application}/{profile}没有匹配所有的表达式，那么会默认从*spring.cloud.config.git.uri* 读取数据。

### 强制拉取 GIt 仓库

> spring.cloud.config.git.force-pull=true

### 删除本地Git副本多余的分支

> spring.cloud.config.git.deleteUntrackedBranches=true

### 共用的配置文件

在 Config 仓储中，以*application* *开头的文件会被所有的应用共享，例如:application.properties、application-*.propeties。

### 2.3 健康监测

ConfigServer有监测配置的*EnviromentRespository*是否正常运行的机子。默认情况下，它会去*EnviromentRespository*查询一个名为 **app**的应用。profile 和label使用*EnviromentRespository*默认的配置。

 也可以配置服务端监测多个应用

```Yaml
spring:
  cloud:
    config:
      server:
        health:
          repositories:
            myservice:
              label: mylabel
            myservice-dev:
              name: myservice
              profiles: development
```

### 2.4 内嵌的ConfigServer

配置服务器最好是以单独的实例运行，但是如果你需要将其内嵌到其别的应用中。可以

1. 使用@EnableConfigServer注解。
2. 另一个额外的参数可以用在这种场景，那就是*spring.cloud.config.server.bootstrap*。

如果先改变服务端暴露服务的地址可以修改*spring.cloud.config.server.prefix*参数。这个前缀必须以*/*开头。



## 3 通知推送和Spring Cloud Bus

许多的代码仓库提供商例如github、gitlab、比他buckiet等都会通过一个webhook来通知你仓库内容的更新。

## 4 Config客户端

springboot 应用可以完全利用Spring Config Server的优势，与此同时还能获取*Enviroment*改变事件的特性。

### 4.1 配置Bootstrap

获取classpath路径下的bootstrap是任何Spring cloud Config client项目的默认行为。当一个ConfigClient启动的时候他会根据*springl.cloud.config.uri*从服务器获取配置文件并初始化*Enviroment*。

### 4.2 Discovery First Bootstrap

 如果使用`DiscoveryClient`的实现例如`Spring Cloud Netflix`、`Eureka Service Discovery`、`Spring Cloud Consul`(目前`Spring Cloud Zookeeper`还不支持)，可以让Config Server Register注册 Discovery Service（如果你想的话)。

如果你先使用`DiscoveryClient`来标识ConfigServer，可以配置`spring.cloud.config.discovery.enabled=true`。开启改配置的结果是所有的客户端都需要一个`bootstrap.properties` 来给出discovery的相关配置。例如，如果使用 Spring cloud Netflix，你需要定义Eureka Server的地址`eureka.client.serviceUrl.defaultZone`。默认的ServiceId是*configServer*可以通过配置`spring.cloud.config.discovery.serviceId`来更改，或者在`application.properties`中配置`spring.application.name`参数。