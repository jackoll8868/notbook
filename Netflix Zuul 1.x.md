# Netflix Zuul 1.x

## 1 How it Works

### 1.1 Filter 预览

Zuul的核心就是一系列用来在路由请求或者响应过程中执行的Filter.

Zuul Filter的核心概念如下:

- **Type**:通常用来定义Filter在路由流程中适用的边界；
- **Execution Order**:执行顺序，应用在**Type**上，用来定义Filter chain的执行顺序。
- **Criteria**:一个Filter是否应该被执行的必备条件；
- **Action**:如果规则符合，具体应该被执行何种操作。

Zuul提供了一个动态读取、编译和运行Filters的框架。Filters之间并不会相互交流，他们通过RequestContext共享状态。

Filters 目前使用Groovy编写，尽管Zuul支持任意基于JVM的语言。

## 1.2 Filter Type

在一个 Request 的生命周期类，zuul定义了很多标准的Filter 类型:

- **PRE**:在请求被路由到具体的服务之前执行。例如:请求认证，获取具体服务地址，日志等；
- **ROUTING**:用来处理将请求路由给服务。在这里通常创建 HTTP 请求并使用Apache Http Client或者Netflix Ribbon发送请求;
- **POST**: 请求被路由到服务之后执行。例如:添加标准的HTTP头信息到Response,收集相关控制信息，将响应从后端服务转发到客户端。
- **ERROR**：在上述任意步骤出现错误之后执行。

除了默认的Filter流程，zuul 允许用户创建自定义的filter type并执行。例如，我们有一个自定义的**STATIC**类型用来在Zuul中创建响应，而不是转发请求到具体的后端服务。

### 1.3 Zuul  请求声明周期

![Zuul Request LifeCyle](https://camo.githubusercontent.com/4eb7754152028cdebd5c09d1c6f5acc7683f0094/687474703a2f2f6e6574666c69782e6769746875622e696f2f7a75756c2f696d616765732f7a75756c2d726571756573742d6c6966656379636c652e706e67)





