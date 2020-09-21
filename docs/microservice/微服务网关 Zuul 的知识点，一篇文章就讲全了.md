# 微服务网关 Zuul 的知识点，一篇文章就讲全了

> 作者：kosamino
>
> 链接：https://www.cnblogs.com/jing99/p/11696192.html

Zuul 是 Spring Cloud 中的微服务网关。网关： 是一个网络整体系统中的前置门户入口。请求首先通过网关，进行路径的路由，定位到具体的服务节点上。

Zuul 是一个微服务网关，首先是一个微服务。也是会在 Eureka 注册中心中进行服务的注册和发现。也是一个网关，请求应该通过 Zuul 来进行路由。

Zuul 网关不是必要的。是推荐使用的。

使用 Zuul，一般在微服务数量较多（多于 10 个）的时候推荐使用，对服务的管理有严格要求的时候推荐使用，当微服务权限要求严格的时候推荐使用。

## 一、Zuul 网关的作用

网关有以下几个作用：

- 统一入口：为全部微服务提供一个统一的入口，网关起到外部和内部隔离的作用，保障了后端服务的安全性。
- 鉴权校验：识别每个请求的权限，拒绝不符合要求的请求。
- 动态路由：动态的将请求路由到不同的后端集群中。
- 减少客户端与服务端的耦合：服务可以独立发展，通过网关层来做映射。

![](http://img.uprogrammer.cn/static/20200821124456.png)

## 二、Zuul 网关的应用

### 　　 1、网关访问方式

通过 Zuul 访问服务的 URL 地址默认格式为：`http://zuul-host-ip:port/要访问的服务名称/服务中的 URL`

服务名称：`properties` 配置文件中的 `spring.application.name`。

服务的 URL：就是对应的服务对外提供的 URL 路径监听。

### 　　 2、网关依赖注入

```xml
<!-- Spring Cloud Eureka Client 启动器 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zuul</artifactId>
</dependency>
<!-- zuul 网关的重试机制，不是使用 ribbon 内置的重试机制
   是借助 spring-retry 组件实现的重试
   开启 zuul 网关重试机制需要增加下述依赖
 -->
<dependency>
   <groupId>org.springframework.retry</groupId>
   <artifactId>spring-retry</artifactId>
</dependency>
```

### 　　 3、网关启动器

```java
/**
 * @EnableZuulProxy - 开启 Zuul 网关。
 * 当前应用是一个 Zuul 微服务网关。会在 Eureka 注册中心中注册当前服务。并发现其他的服务。
 * Zuul 需要的必要依赖是 spring-cloud-starter-zuul。
 */
@SpringBootApplication
@EnableZuulProxy
public class ZuulApplication {
    public static void main(String[] args) {
        SpringApplication.run(ZuulApplication.class, args);
    }
}
```

### 　　 4、网关全局变量配置

#### 　　 4.1 URL 路径匹配

```properties
# URL pattern
# 使用路径方式匹配路由规则。
# 参数 key 结构： zuul.routes.customName.path=xxx
# 用于配置路径匹配规则。
# 其中customName自定义。通常使用要调用的服务名称，方便后期管理
# 可使用的通配符有： * ** ?
# ? 单个字符
# * 任意多个字符，不包含多级路径
# ** 任意多个字符，包含多级路径
zuul.routes.eureka-application-service.path=/api/**
# 参数key结构： zuul.routes.customName.url=xxx
# url用于配置符合path的请求路径路由到的服务地址。
zuul.routes.eureka-application-service.url=http://127.0.0.1:8080/
```

#### 　　 4.2 服务名称匹配

```properties
# service id pattern 通过服务名称路由
# key 结构： zuul.routes.customName.path=xxx
# 路径匹配规则
zuul.routes.eureka-application-service.path=/api/**
# key 结构 ： zuul.routes.customName.serviceId=xxx
# serviceId用于配置符合path的请求路径路由到的服务名称。
zuul.routes.eureka-application-service.serviceId=eureka-application-service
```

服务名称匹配也可以使用简化的配置：

```properties
# simple service id pattern 简化配置方案
# 如果只配置 path，不配置 serviceId，则 customName 相当于服务名称。
# 符合 path 的请求路径直接路由到 customName 对应的服务上。
zuul.routes.eureka-application-service.path=/api/**
```

#### 　　 4.3 路由排除配置

```properties
# ignored service id pattern
# 配置不被 zuul 管理的服务列表。多个服务名称使用逗号','分隔。
# 配置的服务将不被 zuul 代理。
zuul.ignored-services=eureka-application-service
# 此方式相当于给所有新发现的服务默认排除 zuul 网关访问方式，只有配置了路由网关的服务才可以通过 zuul 网关访问
# 通配方式配置排除列表。
zuul.ignored-services=*
# 使用服务名称匹配规则配置路由列表，相当于只对已配置的服务提供网关代理。
zuul.routes.eureka-application-service.path=/api/**

# 通配方式配置排除网关代理路径。所有符合 ignored-patterns 的请求路径都不被 zuul 网关代理。
zuul.ignored-patterns=/**/test/**
zuul.routes.eureka-application-service.path=/api/**
```

#### 　　 4.4 路由前缀配置

```properties
# prefix URL pattern 前缀路由匹配
# 配置请求路径前缀，所有基于此前缀的请求都由 zuul 网关提供代理。
zuul.prefix=/api
# 使用服务名称匹配方式配置请求路径规则。
# 这里的配置将为：http://ip:port/api/appservice/** 的请求提供 zuul 网关代理，可以将要访问服务进行前缀分类。
# 并将请求路由到服务 eureka-application-service 中。
zuul.routes.eureka-application-service.path=/appservice/**
```

### 　　 5 Zuul 网关配置总结

网关配置方式有多种，**默认、URL、服务名称、排除|忽略、前缀**。

网关配置没有优劣好坏，应该在不同的情况下选择合适的配置方案。

Zuul 网关其底层使用 ribbon 来实现请求的路由，并内置 Hystrix，可选择性提供网关 fallback 逻辑。使用 Zuul 的时候，并不推荐使用 Feign 作为 application client 端的开发实现。毕竟 Feign 技术是对 ribbon 的再封装，使用 Feign 本身会提高通讯消耗，降低通讯效率，只在服务相互调用的时候使用 Feign 来简化代码开发就够了。而且商业开发中，使用 Ribbon+RestTemplate 来开发的比例更高。

## 三、Zuul 网关过滤器

Zuul 中提供了过滤器定义，可以用来过滤代理请求，提供额外功能逻辑。如：权限验证，日志记录等。

Zuul 提供的过滤器是一个父类。父类是 `ZuulFilter`。通过父类中定义的抽象方法 filterType，来决定当前的 Filter 种类是什么。有前置过滤、路由后过滤、后置过滤、异常过滤。

- **前置过滤**：是请求进入 Zuul 之后，立刻执行的过滤逻辑。
- **路由后过滤**：是请求进入 Zuul 之后，并 Zuul 实现了请求路由后执行的过滤逻辑，路由后过滤，是在远程服务调用之前过滤的逻辑。
- **后置过滤**：远程服务调用结束后执行的过滤逻辑。
- **异常过滤**：是任意一个过滤器发生异常或远程服务调用无结果反馈的时候执行的过滤逻辑。无结果反馈，就是远程服务调用超时。

### 　　 3.1 过滤器实现方式

继承父类 `ZuulFilter`。在父类中提供了 4 个抽象方法，分别是：`filterType`, `filterOrder`, `shouldFilter`, `run`。其功能分别是：

- `filterType`：方法返回字符串数据，代表当前过滤器的类型。可选值有 - `pre`, `route`, `post`, `error`。

  - `pre` \- 前置过滤器，在请求被路由前执行，通常用于处理身份认证，日志记录等；
  - `route` \- 在路由执行后，服务调用前被调用；
  - `error` \- 任意一个 filter 发生异常的时候执行或远程服务调用没有反馈的时候执行（超时），通常用于处理异常；
  - `post` \- 在 `route` 或 `error` 执行后被调用，一般用于收集服务信息，统计服务性能指标等，也可以对 response 结果做特殊处理。

- `filterOrder`：返回 int 数据，用于为同 `filterType` 的多个过滤器定制执行顺序，返回值越小，执行顺序越优先。
- `shouldFilter`：返回 boolean 数据，代表当前 filter 是否生效。
- `run`：具体的过滤执行逻辑。如 pre 类型的过滤器，可以通过对请求的验证来决定是否将请求路由到服务上；如 post 类型的过滤器，可以对服务响应结果做加工处理（如为每个响应增加 footer 数据）。

### 　　 3.2 过滤器的生命周期

![](http://img.uprogrammer.cn/static/20200821125848.png)

### 　　 3.3 代码示例

```java
/**
 * Zuul 过滤器，必须继承 ZuulFilter 父类。
 * 当前类型的对象必须交由 Spring 容器管理。使用 @Component 注解描述。
 * 继承父类后，必须实现父类中定义的4个抽象方法。
 * shouldFilter、 run、 filterType、 filterOrder
 */
@Component
public class LoggerFilter extends ZuulFilter {

    private static final Logger logger = LoggerFactory.getLogger(LoggerFilter.class);

    /**
     * 返回 boolean 类型。代表当前 filter 是否生效。
     * 默认值为 false。
     * 返回 true 代表开启 filter。
     */
    @Override
    public boolean shouldFilter() {
        return true;
    }

    /**
     * run 方法就是过滤器的具体逻辑。
     * return 可以返回任意的对象，当前实现忽略。（Sprin Cloud Zuul官方解释）
     * 直接返回 null 即可。
     */
    @Override
    public Object run() throws ZuulException {
        // 通过 zuul，获取请求上下文
        RequestContext rc = RequestContext.getCurrentContext();
        HttpServletRequest request = rc.getRequest();

        logger.info("LogFilter1.....method={}, url={}",
                request.getMethod(),request.getRequestURL().toString());
        // 可以记录日志、鉴权，给维护人员记录提供定位协助、统计性能
        return null;
    }

    /**
     * 过滤器的类型。可选值有：
     * pre - 前置过滤
     * route - 路由后过滤
     * error - 异常过滤
     * post - 远程服务调用后过滤
     */
    @Override
    public String filterType() {
        return "pre";
    }

    /**
     * 同种类的过滤器的执行顺序。
     * 按照返回值的自然升序执行。
     */
    @Override
    public int filterOrder() {
        return 0;
    }
}
```

## 四、Zuul 网关的容错（与 Hystrix 的无缝结合）

在 Spring Cloud 中，Zuul 启动器中包含了 Hystrix 相关依赖，在 Zuul 网关工程中，默认是提供了 Hystrix Dashboard 服务监控数据的 (hystrix.stream)，但是不会提供监控面板的界面展示。可以说，在 Spring Cloud 中，Zuul 和 Hystrix 是无缝结合的。

### 　　 4.1 Zuul 中的服务降级处理

在 Edgware 版本之前，Zuul 提供了接口 `ZuulFallbackProvider` 用于实现 fallback 处理。从 Edgware 版本开始，Zuul 提供了 `ZuulFallbackProvider` 的子接口 `FallbackProvider` 来提供 fallback 处理。

Zuul 的 fallback 容错处理逻辑，只针对 timeout 异常处理，当请求被 Zuul 路由后，只要服务有返回（包括异常），都不会触发 Zuul 的 fallback 容错逻辑。

因为对于 Zuul 网关来说，做请求路由分发的时候，结果由远程服务运算的。那么远程服务反馈了异常信息，Zuul 网关不会处理异常，因为无法确定这个错误是否是应用真实想要反馈给客户端的。

### 　　 4.2 代码示例

```java
/**
 * 如果需要在 Zuul 网关服务中增加容错处理 fallback，需要实现接口 ZuulFallbackProvider
 * Spring Cloud 框架，在 Edgware 版本(包括)之后，声明接口 ZuulFallbackProvider 过期失效，
 * 提供了新的 ZuulFallbackProvider 的子接口 - FallbackProvider
 * 在老版本中提供的 ZuulFallbackProvider 中，定义了两个方法。
 *  - String getRoute()
 *    当前的 fallback 容错处理逻辑处理的是哪一个服务。可以使用通配符 '*' 代表为全部的服务提供容错处理。
 *    如果只为某一个服务提供容错，返回对应服务的 spring.application.name 值。
 *  - ClientHttpResponse fallbackResponse()
 *    当服务发生错误的时候，如何容错。
 * 新版本中提供的 FallbackProvider 提供了新的方法。
 *  - ClientHttpResponse fallbackResponse(Throwable cause)
 *    如果使用新版本中定义的接口来做容错处理，容错处理逻辑，只运行子接口中定义的新方法。也就是有参方法。
 *    是为远程服务发生异常的时候，通过异常的类型来运行不同的容错逻辑。
 */
@Component
public class TestFallBbackProvider implements FallbackProvider {

    /**
     * return - 返回 fallback 处理哪一个服务。返回的是服务的名称
     * 推荐 - 为指定的服务定义特性化的 fallback 逻辑。
     * 推荐 - 提供一个处理所有服务的 fallback 逻辑。
     * 好处 - 服务某个服务发生超时，那么指定的 fallback 逻辑执行。如果有新服务上线，未提供 fallback 逻辑，有一个通用的。
     */
    @Override
    public String getRoute() {
        return "eureka-application-service";
    }

    /**
     * fallback 逻辑。在早期版本中使用。
     * Edgware 版本之后， ZuulFallbackProvider 接口过期，提供了新的子接口 FallbackProvider
     * 子接口中提供了方法 ClientHttpResponse fallbackResponse(Throwable cause)。
     * 优先调用子接口新定义的 fallback 处理逻辑。
     */
    @Override
    public ClientHttpResponse fallbackResponse() {
        System.out.println("ClientHttpResponse fallbackResponse()");

        List<Map<String, Object>> result = new ArrayList<>();
        Map<String, Object> data = new HashMap<>();
        data.put("message", "服务正忙，请稍后重试");
        result.add(data);

        ObjectMapper mapper = new ObjectMapper();

        String msg = "";
        try {
            msg = mapper.writeValueAsString(result);
        } catch (JsonProcessingException e) {
            msg = "";
        }

        return this.executeFallback(HttpStatus.OK, msg, "application", "json", "utf-8");
    }

    /**
     * fallback 逻辑。优先调用。可以根据异常类型动态决定处理方式。
     */
    @Override
    public ClientHttpResponse fallbackResponse(Throwable cause) {
        System.out.println("ClientHttpResponse fallbackResponse(Throwable cause)");
        if(cause instanceof NullPointerException){

            List<Map<String, Object>> result = new ArrayList<>();
            Map<String, Object> data = new HashMap<>();
            data.put("message", "网关超时，请稍后重试");
            result.add(data);

            ObjectMapper mapper = new ObjectMapper();

            String msg = "";
            try {
                msg = mapper.writeValueAsString(result);
            } catch (JsonProcessingException e) {
                msg = "";
            }

            return this.executeFallback(HttpStatus.GATEWAY_TIMEOUT, msg, "application", "json", "utf-8");
        } else {
            return this.fallbackResponse();
        }
    }

    /**
     * 具体处理过程。
     * @param status 容错处理后的返回状态，如 200 正常 GET 请求结果，201 正常 POST 请求结果，404 资源找不到错误等。
     *  使用 Spring 提供的枚举类型对象实现。HttpStatus
     * @param contentMsg 自定义的响应内容。就是反馈给客户端的数据。
     * @param mediaType 响应类型，是响应的主类型， 如： application、text、media。
     * @param subMediaType 响应类型，是响应的子类型， 如： json、stream、html、plain、jpeg、png等。
     * @param charsetName 响应结果的字符集。这里只传递字符集名称，如： utf-8、gbk、big5等。
     * @return ClientHttpResponse 就是响应的具体内容。
     *  相当于一个HttpServletResponse。
     */
    private final ClientHttpResponse executeFallback(final HttpStatus status,
            String contentMsg, String mediaType, String subMediaType, String charsetName) {
        return new ClientHttpResponse() {

            /**
             * 设置响应的头信息
             */
            @Override
            public HttpHeaders getHeaders() {
                HttpHeaders header = new HttpHeaders();
                MediaType mt = new MediaType(mediaType, subMediaType, Charset.forName(charsetName));
                header.setContentType(mt);
                return header;
            }

            /**
             * 设置响应体
             * Zuul 会将本方法返回的输入流数据读取，并通过 HttpServletResponse 的输出流输出到客户端。
             */
            @Override
            public InputStream getBody() throws IOException {
                String content = contentMsg;
                return new ByteArrayInputStream(content.getBytes());
            }

            /**
             * ClientHttpResponse 的 fallback 的状态码返回 String
             */
            @Override
            public String getStatusText() throws IOException {
                return this.getStatusCode().getReasonPhrase();
            }

            /**
             * ClientHttpResponse 的 fallback 的状态码返回 HttpStatus
             */
            @Override
            public HttpStatus getStatusCode() throws IOException {
                return status;
            }

            /**
             * ClientHttpResponse 的fallback 的状态码返回int
             */
            @Override
            public int getRawStatusCode() throws IOException {
                return this.getStatusCode().value();
            }

            /**
             * 回收资源方法
             * 用于回收当前 fallback 逻辑开启的资源对象的。
             * 不要关闭 getBody 方法返回的那个输入流对象。
             */
            @Override
            public void close() {
            }
        };
    }
}
```

## 五、Zuul 网关的限流保护

Zuul 网关组件也提供了限流保护。当请求并发达到阀值，自动触发限流保护，返回错误结果。只要提供 error 错误处理机制即可。

Zuul 的限流保护需要额外依赖 spring-cloud-zuul-ratelimit 组件。

```xml
<dependency>
    <groupId>com.marcosbarbero.cloud</groupId>
    <artifactId>spring-cloud-zuul-ratelimit</artifactId>
    <version>1.3.4.RELEASE</version>
</dependency>
```

### 　　 5.1 全局限流配置

使用全局限流配置，Zuul 会对代理的所有服务提供限流保护。

```properties
# 开启限流保护
zuul.ratelimit.enabled=true
# 60s 内请求超过 3 次，服务端就抛出异常，60s 后可以恢复正常请求
zuul.ratelimit.default-policy.limit=3
zuul.ratelimit.default-policy.refresh-interval=60
# 针对 IP 进行限流，不影响其他 IP
zuul.ratelimit.default-policy.type=origin
```

### 　　 5.2 局部限流配置

使用局部限流配置，Zuul 仅针对配置的服务提供限流保护。

```properties
# 开启限流保护
zuul.ratelimit.enabled=true
# hystrix-application-client 服务 60s 内请求超过 3 次，服务抛出异常。
zuul.ratelimit.policies.hystrix-application-client.limit=3
zuul.ratelimit.policies.hystrix-application-client.refresh-interval=60
# 针对 IP 限流。
zuul.ratelimit.policies.hystrix-application-client.type=origin
```

### 　　 5.3 限流参数简介

![](http://img.uprogrammer.cn/static/20200821140552.png)

## 六、Zuul 网关性能调优：网关的两层超时调优

使用 Zuul 的 Spring Cloud 微服务结构图：

![](http://img.uprogrammer.cn/static/20200821140635.png)

从上图中可以看出。整体请求逻辑还是比较复杂的，在没有 Zuul 网关的情况下，app client 请求 app service 的时候，也有请求超时的可能。那么当增加了 Zuul 网关的时候，请求超时的可能就更明显了。

当请求通过 Zuul 网关路由到服务，并等待服务返回响应，这个过程中 Zuul 也有超时控制。Zuul 的底层使用的是 Hystrix + ribbon 来实现请求路由。结构如下：

![](http://img.uprogrammer.cn/static/20200821140737.png)

Zuul 中的 Hystrix 内部使用线程池隔离机制提供请求路由实现，其默认的超时时长为 1000 毫秒。ribbon 底层默认超时时长为 5000 毫秒。如果 Hystrix 超时，直接返回超时异常。如果 ribbon 超时，同时 Hystrix 未超时，ribbon 会自动进行服务集群轮询重试，直到 Hystrix 超时为止。如果 Hystrix 超时时长小于 ribbon 超时时长，ribbon 不会进行服务集群轮询重试。

那么在 Zuul 中可配置的超时时长就有两个位置：Hystrix 和 ribbon。具体配置如下：

```properties
# 开启 zuul 网关重试
zuul.retryable=true
# hystrix 超时时间设置
# 线程池隔离，默认超时时间1000ms
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=8000

# ribbon 超时时间设置：建议设置比 Hystrix 小
# 请求连接的超时时间: 默认 5000 ms
ribbon.ConnectTimeout=5000
# 请求处理的超时时间: 默认 5000 ms
ribbon.ReadTimeout=5000
# 重试次数：MaxAutoRetries 表示访问服务集群下原节点（同路径访问）；MaxAutoRetriesNextServer 表示访问服务集群下其余节点（换台服务器）
ribbon.MaxAutoRetries=1
ribbon.MaxAutoRetriesNextServer=1
# 开启重试
ribbon.OkToRetryOnAllOperations=true
```

Spring Cloud 中的 Zuul 网关重试机制是使用 spring-retry 实现的。工程必须依赖下述资源：

```xml
<dependency>
　　<groupId>org.springframework.retry</groupId>
　　<artifactId>spring-retry</artifactId>
</dependency>
```

# 更多好文

[学会MyBatis动态SQL，写SQL也能爽到飞起](https://mp.weixin.qq.com/s/RIPt-2LhMACHkle7P9JCtg)

[一文搞懂 Spring Boot 启动原理](https://mp.weixin.qq.com/s/YT9k8PpEKTWzXcBXA8fftw)

[线上服务器 CPU 100%, 如何一键定位](https://mp.weixin.qq.com/s/xmYpkaeq3lFjxx-2jOhEyA)

[冒着挂科的风险也要给你们看的 Spring Cloud 入门总结](https://mp.weixin.qq.com/s/aTuPgiyr2pJGs5iRuvq_uQ)