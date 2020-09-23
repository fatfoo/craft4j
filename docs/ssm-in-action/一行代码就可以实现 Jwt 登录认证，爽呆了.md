# 一行代码就可以实现 Jwt 登录认证，爽呆了

在上一篇《[Spring Boot 集成 JWT 实现用户登录认证](https://mp.weixin.qq.com/s/qwoPiuR_leNMS4piGF_WyQ)》中，我们已经基于 nimbus-jose-jwt 封装好了一个 jwt，并且只需要自定义 HandlerInterceptor 实现类和 WebMvcConfigurer 实现类就可以在项目中引入使用 jwt 做登录认证。

Spring Boot 中的 starter 是一种非常重要的机制，能够抛弃以前繁杂的配置，将其统一集成进 starter，应用只需要在 Maven 中引入 starter 依赖，Spring Boot 就能自动扫描到要加载的信息并启动相应的默认配置。starter 让我们摆脱了各种依赖库的处理，需要配置各种信息的困扰。Spring Boot 会自动通过 classpath 路径下的类发现需要的 Bean，并注册进 IOC 容器。Spring Boot 提供了针对日常企业应用研发各种场景的 spring-boot-starter 依赖模块。所有这些依赖模块都遵循着约定成俗的默认配置，并允许我们调整这些配置，即遵循“约定大于配置”的理念。

接下来，我们就在之前封装的 ron-jwt 基础上自定义的一个 starter，项目只要在依赖中引入这个 starter，再定义简单的配置就可以使用 jwt 的实现登录认证。

### 新建工程并配置依赖

Spring Boot 提供的 starter 以 `spring-boot-starter-xxx` 的方式命名的。官方建议自定义的 starter 使用 `xxx-spring-boot-starter` 命名规则，以区分 Spring Boot 生态提供的 starter。所以我们新建工程 ron-jwt-spring-boot-starter。

![](http://img.uprogrammer.cn/static/20200914182807.png)

在 pom.xml 中引入依赖

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-configuration-processor</artifactId>
  <optional>true</optional>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-autoconfigure</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
  <groupId>io.ron</groupId>
  <artifactId>ron-jwt</artifactId>
  <version>1.0-SNAPSHOT</version>
</dependency>
```

spring-boot-configuration-processor 主要的作用是在编译时在 META-INF 下生成 spring-configuration-metadata.json 文件，该文件主要为IDE 使用，即可以通过在 application.properties 文件中通过 ctrl + 点击进入配置属性所在的类中。

spring-boot-autoconfigure 主要作用是提供自动装配功能。

spring-boot-starter-web 则是因为我们将会内置 HandlerInterceptor 实现类和 WebMvcConfigurer 实现类。

ron-jwt 是我们在上一篇中封装好的 jwt 库。

### 定义配置项管理类

我们定义 JwtProperties 来声明 starter 的使用者可使用哪些配置项。

```java
@ConfigurationProperties(prefix = "ron.jwt")
public class JwtProperties {

    private String tokenName = JwtUtils.DEFAULT_TOKEN_NAME;

    private String hmacKey;

    private String jksFileName;

    private String jksPassword;

    private String certPassword;

    // 签发人
    private String issuer;

    // 主题
    private String subject;

    // 受众
    private String audience;

    private long notBeforeIn;

    private long notBeforeAt;

    private long expiredIn;

    private long expiredAt;
}
```

具体参数说明，可参考上一篇文章中的 JwtConfig。

`@ConfigurationProperties` 注解指定了所有配置项的前缀为 `ron.jwt`。

`@ConfigurationProperties` 的基本用法非常简单：我们为每个要捕获的外部属性提供一个带有字段的类。请注意以下几点:

- 前缀定义了哪些外部属性将绑定到类的字段上。
- 根据 Spring Boot 宽松的绑定规则，类的属性名称必须与外部属性的名称匹配。
- 我们可以简单地用一个值初始化一个字段来定义一个默认值。
- 类本身可以是包私有的。
- 类的字段必须有公共 setter 方法。

> Spring Boot 宽松的绑定规则（relaxed binding）：
>
> Spring Boot 使用一些宽松的绑定属性规则。因此，以下变体都将绑定到 tokenName 属性上：
>
> + ron.jwt.tokenname=Authorization
> + ron.jwt.tokenName=Authorization
> + ron.jwt.token_name=Authorization
> + ron.jwt.token-name=Authorization

### 实现相关的功能

在上一篇中，我们把 HandlerInterceptor 实现类和 WebMvcConfigurer 实现类放在具体的业务项目中自己实现。实际上这部分也是项目中比较通用的逻辑，因此我们考虑将这些实现放置在 starter 中。项目中可以不做多余的自定义，直接通过引入 starter 就可以使用 jwt 认证的功能。

#### JwtInterceptor

```java
public class JwtInterceptor implements HandlerInterceptor {

    private Logger logger = LoggerFactory.getLogger(JwtInterceptor.class);

    private static final String PREFIX_BEARER = "Bearer ";

    @Autowired
    private JwtProperties jwtProperties;

    @Autowired
    private JwtService jwtService;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                             Object handler) throws Exception {

        // 如果不是映射到方法直接通过
        if(!(handler instanceof HandlerMethod)){
            return true;
        }

        HandlerMethod handlerMethod = (HandlerMethod) handler;
        Method method = handlerMethod.getMethod();
        // 检查是否有 @AuthRequired 注解，有且 required() 为 false 则跳过
        if (method.isAnnotationPresent(AuthRequired.class)) {
            AuthRequired authRequired = method.getAnnotation(AuthRequired.class);
            if (!authRequired.required()) {
                return true;
            }
        }

        String token = request.getHeader(jwtProperties.getTokenName());

        logger.info("token: {}", token);

        if (StringUtils.isEmpty(token) || token.trim().equals(PREFIX_BEARER.trim())) {
            return true;
        }

        token = token.replace(PREFIX_BEARER, "");

        // 设置线程局部变量中的 token
        JwtContext.setToken(token);

        // 在线程局部变量中设置真实传递的数据，如当前用户信息等
        String payload = jwtService.verify(token);
        JwtContext.setPayload(payload);

        return onPreHandleEnd(request, response, handler, payload);
    }

    public boolean onPreHandleEnd(HttpServletRequest request, HttpServletResponse response,
                                  Object handler, String payload) throws Exception {
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response,
                           Object handler, ModelAndView modelAndView) throws Exception {

    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) throws Exception {
        // 务必在线程结束前清理线程局部变量
        JwtContext.removeAll();
    }
}
```

`onPreHandleEnd` 方法是一个默认实现，如业务上有必要，也可以继承 JwtInterceptor，在这个方法中添加自定义的逻辑。一个可能的场景是将 JWT 的 token 放到 Redis 中进行超时管理。

#### JwtInterceptorConfig

```java
public class JwtInterceptorConfig implements WebMvcConfigurer {

    private JwtInterceptor jwtInterceptor;

    public JwtInterceptorConfig(JwtInterceptor jwtInterceptor) {
        this.jwtInterceptor = jwtInterceptor;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(jwtInterceptor).addPathPatterns("/**");
    }
}
```

这里默认拦截了所有的请求，在上一篇文章中，我们提到可以配合 `@AuthRequired` 来过滤不需要拦截的请求。

### 编写自动配置逻辑

#### JwtAutoConfiguration

```java
@Configuration
@EnableConfigurationProperties(JwtProperties.class)
public class JwtAutoConfiguration {

    @Autowired
    private JwtProperties jwtProperties;

    @Bean
    public JwtConfig jwtConfig() {
        JwtConfig jwtConfig = new JwtConfig();
        BeanUtils.copyProperties(jwtProperties, jwtConfig);
        return jwtConfig;
    }

    @Bean
    public JwtService jwtService() {
        JwtConfig jwtConfig = jwtConfig();
        return JwtUtils.obtainJwtService(jwtConfig);
    }

    @Bean
    public JwtInterceptor jwtInterceptor() {
        return new JwtInterceptor();
    }

    @Bean
    public JwtInterceptorConfig jwtInterceptorConfig() {
        return new JwtInterceptorConfig(jwtInterceptor());
    }
}
```

`@EnableConfigurationProperties` 的作用是引入使用 `@ConfigurationProperties` 注解的类，并使其生效。

`@EnableConfigurationProperties` 文档中解释：当 `@EnableConfigurationProperties` 注解应用到你的`@Configuration` 时， 任何被 `@ConfigurationProperties` 注解的 beans 将自动被 Environment 属性配置。 这种风格的配置特别适合与 SpringApplication 的外部 YAML 配置进行配合使用。

### 集成 starter 使之生效

有两种方式可以让 starter 在应用中生效。

#### 通过 SPI 机制加载 - 被动生效

通过 Spring Boot 的 `SPI` 的机制来去加载我们的starter。

在 resources 目录下新建 WEB-INF/spring.factories 文件。

META-INF/spring.factories 文件是 Spring Boot 框架识别并解析 starter 的核心文件。spring.factories 文件是帮助 Spring Boot 项目包以外的 Bean（即在 pom 文件中添加依赖中的 Bean）注册到 Spring Boot 项目的 Spring 容器。由于 `@ComponentScan` 注解只能扫描Spring Boot 项目包内的 Bean 并注册到 Spring 容器中，因此需要 `@EnableAutoConfiguration` 注解来注册项目包外的 Bean。而 spring.factories 文件，则是用来记录项目包外需要注册的 Bean 类名。

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=io.ron.jwt.starter.JwtAutoConfiguration
```

#### 自定义 Enable 注解引入 - 主动生效

在 starter 组件集成到我们的 Spring Boot 应用时需要主动声明启用该 starter 才生效，我们通过自定义一个 `@Enable` 注解然后在把自动配置类通过 `Import` 注解引入进来。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import({JwtAutoConfiguration.class})
@Documented
@Inherited
public @interface EnableJwt {

}
```

**如果使用主动生效的方式，那么实现被动生效的 META-INF/spring.factories 文件需要移除。**

### 打包并发布 starter

使用 `mvn install` 可以打包并安装在本地；

使用 `mvn deploy` 可以发布到远程仓库。

> 后续我会写一篇关于 Maven 的文章，结合自己入行多年的经验，将开发人员会用到 Maven 技巧进行梳理。
>
> 欢迎关注我的公众号：精进Java(ID：**craft4j**)。

### 测试应用程序

可以复用上一篇中的项目，在 pom.xml 中修改依赖，引入 ron-jwt-spring-boot-starter。

```xml
<dependency>
  <groupId>io.ron</groupId>
  <artifactId>ron-jwt-spring-boot-starter</artifactId>
  <version>1.0-SNAPSHOT</version>
</dependency>
```

去掉 HandlerInterceptor 实现类和 WebMvcConfigurer 实现类。

在 application.yml 中配置 `ron.jwt.hmac-key` 的值，就可以提供 HMAC 算法实现的 JWT 签名与验证功能了。

如果 starter 采用的被动生效方式，现在就可以运行程序，然后使用 Postman 进行测试并观察结果。

如果 starter 采用的主动生效方式，需要在项目启动类上添加 `@EnableJwt` 注解将 jwt-starter 引入。

```java
@SpringBootApplication
public class JwtStarterApplication {

    public static void main(String[] args) {
        SpringApplication.run(JwtStarterApplication.class, args);
    }
}
```

###### - End -
---

通过两篇文章，我们展示了基于 nimbus-jose-jwt 的 jwt 库使用，在此基础上封装了我们自己的基础 jwt 库，并介绍了 Spring Boot 自定义 starter 的步骤，以及自定义 @Enable 注解的实现。

如果你觉得有收获，请关注我的公众号：精进Java(ID：**craft4j**)，第一时间获取 Java 后端与架构的知识动态。

如果你对项目的完整源码感兴趣，可以在公众号中回复 `jwt` 来获取。