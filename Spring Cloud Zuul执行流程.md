# Spring Cloud Zuul执行流程

# @EnableZuulProxy执行流程

大致流程如下:

![Spring-cloud-zuul-workflow](/Users/chenkun/Documents/学习/notbook/imgs/Spring-Cloud-Zuul-WorkFlow.png)

Spring Cloud 会为Zuul创建一个ServletContent,在ServletContent中做如下操作:

1. 创建RequsetContext;
2. 调用ZuulRunner的`preRoute`方法;
   1. `ZuulRunner`调用`FilterProcess`的`preRoute`方法;
   2. 调用`FilterProcess`的`runFilters`方法；
   3. 调用`FilterProcess`的`proccessZuulFilter(ZuulFilter filter)`方法;
   4. 调用`ZuulFilter`的`shouldFilter`方法如果需要执行，进行下一步；
   5. 调用`runFilter`方法调用`ZuulFilter`的`run`方法获得一个返回值；
   6. 将封装`ZuulFilterResult`。
3. `preRoute`方法执行完之后执行`route`方法；调用链参考步骤2；
4. `route`方法执行完之后调用`postRoute`方法封装返回值，调用链参见步骤2；
5. 执行完`route`之后重置`RequestContext`；

> 注意:
>
> 1. 如果在步骤二执行捕获异常会继续执行:`error`方法—>`postRoute`然后返回；
> 2. 如果步骤三执行捕获异常会继续执行:`error`方法—>`postRoute`方法，然后返回;
> 3. 如果步骤四执行捕获异常会继续执行`error`，然后返回。
>
> 虽然每个`ZuulFilter`的`run`方法都有返回值，但是`FilterProcess`仅仅根据:
>
> ```java
> if (result != null && result instanceof Boolean) {
>                     bResult |= ((Boolean) result);
>                 }
> ```
>
> 来给定返回值，所以返回值永远是个boolean类型。那么需要共享数据必须在`RequsetContext`中设置。
>
> **因为所有操作会在`postRoute`执行之后返回**,因此，如果需要处理一下情形的需要在`post`的类型的`ZuulFilter` 中进行相应的数据转换:
>
> 1. 需要统一处理异常的；
> 2. 需要对后端服务再封装的。

## @EnableZuulProxy默认加载的Filter

| FilterClass           | Filter Type | Order | shouldFilter                                                 | 处理逻辑                 | 备注 |
| --------------------- | ----------- | ----- | ------------------------------------------------------------ | ------------------------ | ---- |
| FormBodyWrapperFilter | PRE         | -1    | content-type包含APPLICATION_FORM_URLENCODED和MULTIPART_FORM_DATA | 将这两种请求格式进行封装 |      |
| DebugFilter           | PRE         | 1     | zuul.debug.parameter=true时触发                              | 打印日志                 |      |
| SendForwardFilter     | ROUTE       | 500   | ctx.containsKey(FORWARD_TO_KEY)       && !ctx.getBoolean(SEND_FORWARD_FILTER_RAN, false); |                          |      |
| SendResponseFilter    | POST        | 1000  | context.getThrowable() == null       && (!context.getZuulResponseHeaders().isEmpty()          \|\| context.getResponseDataStream() != null          \|\| context.getResponseBody() != null) |                          |      |

