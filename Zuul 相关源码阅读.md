# Zuul 相关源码阅读

## PreDecorationFilter

根据RouteLocator决定该路由到哪儿以及如何路由。同时设置一些相关的zuul的头部信息。

### order&type

`PRE_TYPE`  5

### shouldFilter

1. `FORWARD_TO_KEY(forward.to)`为空,`forward.do`是`SendForwardFilter`使用的参数；
2. `SERVICE_ID_KEY（serviceId）`为空,`serviceId`是`RibbonRoutingFilter`使用的参数；

当serviceId或者forward.to参数没设置的时候执行

###  执行流程

1. 从`HttpServletRequest`中获取`requestURI`；

   1. 从`HttpServletRequest`参数中获取:`javax.servlet.include.context_path`，如果为空调用`getContextPath`方法获取值；
   2. 使用`URLDecoder`算法`decode`;

2. 从`RouteLocator`获取匹配的`Route`；

   1. 如果`Route`为空，首先判断是否为zuulServletRequest,如果是去掉`servletPath`，如果不是去掉`dispatcherServletPath`;
   2. `forwardURI = fallbackPrefix + fallBackUri`
   3. 在`RequestContext` 中设置`FORWARD_TO_KEY`的值为`forwardURI`

3. 如果匹配到`Route`:

   1. 获取`route.getLocation()`，如果location不为空在`RequestContext`中添加如下两个参数然后设置敏感头信息;
      1. `REQUEST_URI_KEY`:`requestURI`=`route.getPath()`
      2. `PROXY_KEY`:`proxy`=`route.getId()`
   2. 如果`route.getRetryable()`不为空，设置`RETRYABLE_KEY=route.getRetryable()`
   3. 如果location以HTTP:开头或者以HTTPS开头,设置routeHost 为`location`，设置`ctx.addOriginResponseHeader(SERVICE_HEADER, location)`。
   4. 如果location以`forward:`开头设置`FORWARD_TO_KEY`为去掉`forward:`的location 和route.getPath()的组合，设置routeHost为null；
   5. 如果不满足条件3和条件4，设置`SERVICE_ID_KEY(serviceId)` 为`location`，设置`routeHost=null`，添加`X-Zuul-ServiceId=location` 的头信息。

   > 备注:
   >
   > **Route**字段解析:
   >
   > 1. 如果`zuul.routers.xxx.serviceId`配置了，那么Route的字段含义如下:
   >
   >  `id`—>`xxx`
   >
   > `location`—>`service-id`
   >
   > `path`—>`请求地址中的URI`

   

   

   ## RibbonRoutingFilter

   用于通过Discovery Service模式获取到服务方并发送请求

   ### order &type

   type:ROUTE_TYPE

   Order:10

   ### shouldFilter

   RequestContext中的routeHost为null,并且SERVICE_ID不为空，且`ctx.sendZuulResponse()`为false

   ### 执行流程

   1. 添加IgnoredHeaders;

   2. 创建`RibbonCommandContext`

      1. 封装requestHeader，去除:host、connection、content-length、content-encoding、server、transfer-encoding、x-application-context之后封装到`RequestContext`的`ZuulRequestHeaders`中;
      2. 将请求URL中的请求参数封装成`Map<String, List<String>>`格式；
      3. 获取请求方法:`verb`;
      4. 获取RequestBody,返回的是一个InputStream.获取方式:

      `(InputStream) RequestContext.getCurrentContext().get(REQUEST_ENTITY_KEY)`，REQUEST_ENTITY_KEY为FilterConstants 的requestEntity;

      5. 如果HttpServletRequest的content-length<0并且请求方法不为:GET，在`RequestContext`中设置{chunkedRequestBody:true}；
      6. 获取`serviceId`、`retryable`、`loadbalanceKey`
      7. 构建请求`zuulRequestUri`，规则为:如果`RequestContext` 中存在`REQUEST_URI_KEY`按照request.getCharacterEncoding进行URLEncode；
      8. 设置contentlength;
      9. 创建`RibbonCommandContext`；

   3. 转发请求:

      1. 从`RibbonCommandFactory` 创建`RibbonCommand`;
      2. 执行HTTP请求；
      3. 返回响应

   4. 设置Response

      1. 在`RequestContext`中设置一个Key:`zuulResponse`，值为response的值；
      2. 在`ProxyRequestHelper`设置response。

**注意**

可以通过往RequestContext设置REQUEST_ENTITY_KEY的值为InputStream的方式向routeFilter传递值。