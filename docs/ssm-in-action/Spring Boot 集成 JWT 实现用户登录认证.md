# Spring Boot 集成 JWT 实现用户登录认证

## JWT 简介

### 什么是 JWT

JWT 是 JSON Web Token 的缩写，是为了在网络应用环境间传递声明而执行的一种基于 `JSON` 的开放标准（(RFC 7519)。定义了一种简洁的，自包含的方法用于通信双方之间以 `JSON` 对象的形式安全的传递信息。因为数字签名的存在，这些信息是可信的，JWT 可以使用 `HMAC` 算法或者是 `RSA` 的公私秘钥对进行签名。

### JWT请求流程

![JWT 请求流程](http://img.uprogrammer.cn/static/20200831144942.png)

1. 用户使用账号和密码发起 POST 请求；
2. 服务器使用私钥创建一个 JWT；
3. 服务器返回这个 JWT 给浏览器；
4. 浏览器将该 JWT 串在请求头中像服务器发送请求；
5. 服务器验证该 JWT；
6. 返回响应的资源给浏览器。

### JWT 的主要应用场景

身份认证在这种场景下，一旦用户完成了登录，在接下来的每个请求中包含 JWT，可以用来验证用户身份以及对路由，服务和资源的访问权限进行验证。由于它的开销非常小，可以轻松的在不同域名的系统中传递，所有目前在单点登录（SSO）中比较广泛的使用了该技术。 信息交换在通信的双方之间使用 JWT 对数据进行编码是一种非常安全的方式，由于它的信息是经过签名的，可以确保发送者发送的信息是没有经过伪造的。

### JWT 数据结构

JWT 是由三段信息构成的，将这三段信息文本用 `.` 连接一起就构成了 JWT 字符串。

JWT 的三个部分依次为头部：Header，负载：Payload 和签名：Signature。

![JWT 数据结构](http://img.uprogrammer.cn/static/20200831123809.png)

#### Header

Header 部分是一个 JSON 对象，描述 JWT 的元数据，通常是下面的样子。

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

上面代码中，`alg` 属性表示签名的算法（algorithm），默认是 HMAC SHA256（写成 HS256）；`typ` 属性表示这个令牌（token）的类型（type），JWT 令牌统一写为 `JWT`。

最后，将上面的 JSON 对象使用 Base64URL 算法转成字符串。

#### Payload

Payload 部分也是一个 JSON 对象，用来存放实际需要传递的有效信息。有效信息包含三个部分：

1. 标准中注册的声明
2. 公共的声明
3. 私有的声明

**标准中注册的声明 (建议但不强制使用) ：**

- iss (issuer)：签发人
- exp (expiration time)：过期时间，必须要大于签发时间
- sub (subject)：主题
- aud (audience)：受众
- nbf (Not Before)：生效时间
- iat (Issued At)：签发时间
- jti (JWT ID)：编号，JWT 的唯一身份标识，主要用来作为一次性 `token`，从而回避重放攻击。

**公共的声明 ：**

公共的声明可以添加任何的信息，一般添加用户的相关信息或其他业务需要的必要信息。但不建议添加敏感信息，因为该部分在客户端可解密。

**私有的声明 ：**

私有声明是提供者和消费者所共同定义的声明，一般不建议存放敏感信息，因为 `base64` 是对称解码的，意味着该部分信息可以归类为明文信息。

这个 JSON 对象也要使用 Base64URL 算法转成字符串。

#### Signature

Signature 部分是对前两部分的签名，防止数据篡改。

首先，需要指定一个密钥（secret）。这个密钥只有服务器才知道，不能泄露给用户。然后，使用 Header 里面指定的签名算法（默认是 HMAC SHA256），按照下面的公式产生签名。

```javascript
HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)
```

算出签名以后，把 Header、Payload、Signature 三个部分拼成一个字符串，每个部分之间用"点"（`.`）分隔，就可以返回给用户。

#### Base64URL

前面提到，Header 和 Payload 串型化的算法是 Base64URL。这个算法跟 Base64 算法基本类似，但有一些小的不同。

JWT 作为一个令牌（token），有些场合可能会放到 URL（比如 `api.example.com/?token=xxx`）。Base64 有三个字符 `+`、 `/` 和 `=`，在 URL 里面有特殊含义，所以要被替换掉：`=` 被省略、`+` 替换成 `-`，`/` 替换成 `_` 。这就是 Base64URL 算法。

### JWT 的使用方式

客户端收到服务器返回的 JWT 之后需要在本地做保存。此后，客户端每次与服务器通信，都要带上这个 JWT。一般的的做法是放在 HTTP 请求的头信息 `Authorization` 字段里面。

```javascript
Authorization: Bearer <token>
```

这样每个请求中，服务端就可以在请求头中拿到 JWT  进行解析与认证。

### JWT 的特性

1. JWT 默认是不加密，但也是可以加密的。生成原始 Token 以后，可以用密钥再加密一次。

2. JWT 不加密的情况下，不能将秘密数据写入 JWT。

3. JWT 不仅可以用于认证，也可以用于交换信息。有效使用 JWT，可以降低服务器查询数据库的次数。

4. JWT 的最大缺点是，由于服务器不保存 session 状态，因此无法在使用过程中废止某个 token，或者更改 token 的权限。也就是说，一旦 JWT 签发了，在到期之前就会始终有效，除非服务器部署额外的逻辑。

5. JWT 本身包含了认证信息，一旦泄露，任何人都可以获得该令牌的所有权限。为了减少盗用，JWT 的有效期应该设置得比较短。对于一些比较重要的权限，使用时应该再次对用户进行认证。

6. 为了减少盗用，JWT 不应该使用 HTTP 协议明码传输，要使用 HTTPS 协议传输。

## 基于 nimbus-jose-jwt 简单封装

nimbus-jose-jwt 是最受欢迎的 JWT 开源库，基于Apache 2.0开源协议，支持所有标准的签名(JWS)和加密(JWE)算法。nimbus-jose-jwt 支持使用对称加密（HMAC）和非对称加密（RSA）两种算法来生成和解析 JWT 令牌。

下面我们对 nimbus-jose-jwt 进行简单的封装，提供以下功能的支持：

1. 支持使用 HMAC 和 RSA 算法生成和解析 JWT 令牌
2. 支持私有信息直接作为 Payload，以及标准信息+私有信息作为 Payload。内置支持后者。
3. 提供工具类及可扩展接口，方便自定义扩展开发。

### pom 中添加依赖

首先我们在 pom.xml 中引入 nimbus-jose-jwt 的依赖。

```xml
<dependency>
  <groupId>com.nimbusds</groupId>
  <artifactId>nimbus-jose-jwt</artifactId>
  <version>8.20</version>
</dependency>
```

### JwtConfig

这个类用于统一管理相关的参数配置。

```java
public class JwtConfig {

    // JWT 在 HTTP HEADER 中默认的 KEY
    private String tokenName = JwtUtils.DEFAULT_TOKEN_NAME;

    // HMAC 密钥，用于支持 HMAC 算法
    private String hmacKey;

    // JKS 密钥路径，用于支持 RSA 算法
    private String jksFileName;

    // JKS 密钥密码，用于支持 RSA 算法
    private String jksPassword;

    // 证书密码，用于支持 RSA 算法
    private String certPassword;

    // JWT 标准信息：签发人 - iss
    private String issuer;

    // JWT 标准信息：主题 - sub
    private String subject;

    // JWT 标准信息：受众 - aud
    private String audience;

    // JWT 标准信息：生效时间 - nbf，未来多长时间内生效
    private long notBeforeIn;
    
    // JWT 标准信息：生效时间 - nbf，具体哪个时间生效
    private long notBeforeAt;

    // JWT 标准信息：过期时间 - exp，未来多长时间内过期
    private long expiredIn;

    // JWT 标准信息：过期时间 - exp，具体哪个时间过期
    private long expiredAt;
}  
```

`hmacKey` 字段用于支持 HMAC 算法，只要该字段不为空，则使用该值作为 HMAC 的密钥对 JWT 进行签名与验证。

`jksFileName`、`jksPassword`、`certPassword` 三个字段用于支持 RSA 算法，程序将读取证书文件作为 RSA 密钥对 JWT 进行签名与验证。

其他几个字段用于设置 Payload 中需要携带的标准信息。

### JwtService

JwtService 是提供 JWT 签名与验证的接口，内置了 HMACJwtServiceImpl 提供 HMAC 算法的实现和 RSAJwtServiceImpl 提供 RSA 算法的实现。两种算法在获取密钥的方式上是有差别的，这里也提出来成了接口方法。后续如果要自定义实现，只需要再写一个具体实现类。

```java
public interface JwtService {

    /**
     * 获取 key
     *
     * @return
     */
    Object genKey();

    /**
     * 对信息进行签名
     *
     * @param payload
     * @return
     */
    String sign(String payload);

    /**
     * 验证并返回信息
     *
     * @param token
     * @return
     */
    String verify(String token);
}
```

```java
public class HMACJwtServiceImpl implements JwtService {

    private JwtConfig jwtConfig;

    public HMACJwtServiceImpl(JwtConfig jwtConfig) {
        this.jwtConfig = jwtConfig;
    }

    @Override
    public String genKey() {
        String key = jwtConfig.getHmacKey();
        if (JwtUtils.isEmpty(key)) {
            throw new KeyGenerateException(JwtUtils.KEY_GEN_ERROR, new NullPointerException("HMAC need a key"));
        }
        return key;
    }

    @Override
    public String sign(String info) {
        return JwtUtils.signClaimByHMAC(info, genKey(), jwtConfig);
    }

    @Override
    public String verify(String token) {
        return JwtUtils.verifyClaimByHMAC(token, genKey(), jwtConfig);
    }
}
```

```java
public class RSAJwtServiceImpl implements JwtService {

    private JwtConfig jwtConfig;

    private RSAKey rsaKey;

    public RSAJwtServiceImpl(JwtConfig jwtConfig) {
        this.jwtConfig = jwtConfig;
    }

    private InputStream getCertInputStream() throws IOException {
        // 读取配置文件中的证书路径
        String jksFile = jwtConfig.getJksFileName();
        if (jksFile.contains("://")) {
            // 从本地文件读取
            return new FileInputStream(new File(jksFile));
        } else {
            // 从 classpath 读取
            return getClass().getClassLoader().getResourceAsStream(jwtConfig.getJksFileName());
        }
    }

    @Override
    public RSAKey genKey() {
        if (rsaKey != null) {
            return rsaKey;
        }
        InputStream is = null;
        try {
            KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
            is = getCertInputStream();
            keyStore.load(is, jwtConfig.getJksPassword().toCharArray());
            Enumeration<String> aliases = keyStore.aliases();
            String alias = null;
            while (aliases.hasMoreElements()) {
                alias = aliases.nextElement();
            }
            RSAPrivateKey privateKey = (RSAPrivateKey) keyStore.getKey(alias, jwtConfig.getCertPassword().toCharArray());
            Certificate certificate = keyStore.getCertificate(alias);
            RSAPublicKey publicKey = (RSAPublicKey) certificate.getPublicKey();
            rsaKey = new RSAKey.Builder(publicKey).privateKey(privateKey).build();
            return rsaKey;
        } catch (IOException | CertificateException | UnrecoverableKeyException
                | NoSuchAlgorithmException | KeyStoreException e) {
            e.printStackTrace();
            throw new KeyGenerateException(JwtUtils.KEY_GEN_ERROR, e);
        } finally {
            if (is != null) {
                try {
                    is.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    @Override
    public String sign(String payload) {
        return JwtUtils.signClaimByRSA(payload, genKey(), jwtConfig);
    }

    @Override
    public String verify(String token) {
        return JwtUtils.verifyClaimByRSA(token, genKey(), jwtConfig);
    }
}
```

### JwtUtils

JwtService 的实现类中比较简洁，因为主要的方法都在 JwtUtils 中提供了。如下是 Payload 中只包含私有信息时，两种算法的签名与验证实现。可以使用这些方法方便的实现自己的扩展。

```java
   /**
     * 使用 HMAC 算法签名信息（Payload 中只包含私有信息）
     *
     * @param info
     * @param key
     * @return
     */
    public static String signDirectByHMAC(String info, String key) {
        try {
            JWSHeader jwsHeader = new JWSHeader.Builder(JWSAlgorithm.HS256)
                    .type(JOSEObjectType.JWT)
                    .build();

            // 建立一个载荷 Payload
            Payload payload = new Payload(info);

            // 将头部和载荷结合在一起
            JWSObject jwsObject = new JWSObject(jwsHeader, payload);

            // 建立一个密匙
            JWSSigner jwsSigner = new MACSigner(key);

            // 签名
            jwsObject.sign(jwsSigner);

            // 生成 token
            return jwsObject.serialize();
        } catch (JOSEException e) {
            e.printStackTrace();
            throw new PayloadSignException(JwtUtils.PAYLOAD_SIGN_ERROR, e);
        }
    }

    /**
     * 使用 RSA 算法签名信息（Payload 中只包含私有信息）
     *
     * @param info
     * @param rsaKey
     * @return
     */
    public static String signDirectByRSA(String info, RSAKey rsaKey) {
        try {
            JWSSigner signer = new RSASSASigner(rsaKey);
            JWSObject jwsObject = new JWSObject(
                    new JWSHeader.Builder(JWSAlgorithm.RS256).keyID(rsaKey.getKeyID()).build(),
                    new Payload(info)
            );
            // 进行加密
            jwsObject.sign(signer);

            return jwsObject.serialize();
        } catch (JOSEException e) {
            e.printStackTrace();
            throw new PayloadSignException(JwtUtils.PAYLOAD_SIGN_ERROR, e);
        }
    }

    /**
     * 使用 HMAC 算法验证 token（Payload 中只包含私有信息）
     *
     * @param token
     * @param key
     * @return
     */
    public static String verifyDirectByHMAC(String token, String key) {
        try {
            JWSObject jwsObject = JWSObject.parse(token);
            // 建立一个解锁密匙
            JWSVerifier jwsVerifier = new MACVerifier(key);
            if (jwsObject.verify(jwsVerifier)) {
                return jwsObject.getPayload().toString();
            }
            throw new TokenVerifyException(JwtUtils.TOKEN_VERIFY_ERROR, new NullPointerException("Payload can not be null"));
        } catch (JOSEException | ParseException e) {
            e.printStackTrace();
            throw new TokenVerifyException(JwtUtils.TOKEN_VERIFY_ERROR, e);
        }
    }

    /**
     * 使用 RSA 算法验证 token（Payload 中只包含私有信息）
     *
     * @param token
     * @param rsaKey
     * @return
     */
    public static String verifyDirectByRSA(String token, RSAKey rsaKey) {
        try {
            RSAKey publicRSAKey = rsaKey.toPublicJWK();
            JWSObject jwsObject = JWSObject.parse(token);
            JWSVerifier jwsVerifier = new RSASSAVerifier(publicRSAKey);
            // 验证数据
            if (jwsObject.verify(jwsVerifier)) {
                return jwsObject.getPayload().toString();
            }
            throw new TokenVerifyException(JwtUtils.TOKEN_VERIFY_ERROR, new NullPointerException("Payload can not be null"));
        } catch (JOSEException | ParseException e) {
            e.printStackTrace();
            throw new TokenVerifyException(JwtUtils.TOKEN_VERIFY_ERROR, e);
        }
    }
```

### JwtException

定义统一的异常类，可以屏蔽 nimbus-jose-jwt 以及其他诸如加载证书错误抛出的异常，并且在其他项目集成我们封装好的库的时候，方便的进行异常处理。

在 JwtService 实现的不同阶段，我们封装了不同的 JwtException 子类，来方便外部根据需要做对应的处理。如异常是 KeyGenerateException，则处理成服务器处理错误；如异常是 TokenVerifyException，则处理成 Token 验证失败，无权限。

### JwtContext

JWT 用于用户认证，经常在 Token 验证完成后，程序中需要获取到当前登录的用户信息， JwtContext 中提供了通过线程局部变量保存信息的方法。

```java
public class JwtContext {

    private static final String KEY_TOKEN = "token";
    private static final String KEY_PAYLOAD = "payload";

    private static ThreadLocal<Map<Object, Object>> context = new ThreadLocal<>();

    private JwtContext() {}

    public static void set(Object key, Object value) {
        Map<Object, Object> locals = context.get();
        if (locals == null) {
            locals = new HashMap<>();
            context.set(locals);
        }
        locals.put(key, value);
    }

    public static Object get(Object key) {
        Map<Object, Object> locals = context.get();
        if (locals != null) {
            return locals.get(key);
        }
        return null;
    }

    public static void remove(Object key) {
        Map<Object, Object> locals = context.get();
        if (locals != null) {
            locals.remove(key);
            if (locals.isEmpty()) {
                context.remove();
            }
        }
    }

    public static void removeAll() {
        Map<Object, Object> locals = context.get();
        if (locals != null) {
            locals.clear();
        }
        context.remove();
    }

    public static void setToken(String token) {
        set(KEY_TOKEN, token);
    }

    public static String getToken() {
        return (String) get(KEY_TOKEN);
    }

    public static void setPayload(Object payload) {
        set(KEY_PAYLOAD, payload);
    }

    public static Object getPayload() {
        return get(KEY_PAYLOAD);
    }
}
```

### @AuthRequired

在项目实战中，并不是所有 Controller 中的方法都必须传 Token，通过 @AuthRequired 注解来区分方法是否需要校验 Token。

```java
/**
 * 应用于 Controller 中的方法，标识是否拦截进行 JWT 验证
 */
@Target({ElementType.METHOD, ElementType.TYPE})
public @interface AuthRequired {

    boolean required() default true;
}
```

## Spring Boot 集成 JWT 实例

有了上面封装好的库，我们在 SpringBoot 项目中集成 JWT。创建好 Spring Boot 项目后，我们编写下面主要的类。

### JwtDemoInterceptor

在 Spring Boot 项目中，通过自定义 HandlerInterceptor 的实现类可以对请求和响应进行拦截，我们新建 JwtDemoInterceptor 类进行拦截。

```java
public class JwtDemoInterceptor implements HandlerInterceptor {

    private static final Logger logger = LoggerFactory.getLogger(JwtDemoInterceptor.class);

    private static final String PREFIX_BEARER = "Bearer ";

    @Autowired
    private JwtConfig jwtConfig;

    @Autowired
    private JwtService jwtService;

    /**
     * 预处理回调方法，实现处理器的预处理（如检查登陆），第三个参数为响应的处理器，自定义 Controller
     * 返回值：
     * true 表示继续流程（如调用下一个拦截器或处理器）；
     * false 表示流程中断（如登录检查失败），不会继续调用其他的拦截器或处理器，此时我们需要通过 response 来产生响应。
     */
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
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

        String token = request.getHeader(jwtConfig.getTokenName());

        logger.info("token: {}", token);

        if (StringUtils.isEmpty(token) || token.trim().equals(PREFIX_BEARER.trim())) {
            return true;
        }

        token = token.replace(PREFIX_BEARER, "");

        String payload = jwtService.verify(token);

        // 设置线程局部变量中的 token
        JwtContext.setToken(token);
        JwtContext.setPayload(payload);
        return true;
    }

    /**
     * 后处理回调方法，实现处理器的后处理（但在渲染视图之前），此时我们可以通过 modelAndView（模型和视图对象）对模型数据进行处理或对视图进行处理，modelAndView 也可能为null。
     */
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {

    }

    /**
     * 整个请求处理完毕回调方法，即在视图渲染完毕时回调，如性能监控中我们可以在此记录结束时间并输出消耗时间，还可以进行一些资源清理，类似于 try-catch-finally 中的 finally
     * 但仅调用处理器执行链中 preHandle 返回 true 的拦截器的 afterCompletion。
     */
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        JwtContext.removeAll();
    }
}
```

`preHandle`、`postHandle`、`afterCompletion` 三个方法的具体作用，可以看代码上的注释。

`preHandle` 中这段代码中的逻辑如下：

1. 拦截被 @AuthRequired 注解的方法，只要不是 `required = false` 都会进行 Token 的校验。
2. 从请求中解析出 Token，对 Token 进行验证。如果验证异常，会在方法中抛出异常。
3. Token 验证通过，会在线程局部变量中设置相关信息，以便后续程序获取处理。

`afterCompletion` 中这段代码对线程变量进行了清理。

### InterceptorConfig

定义 InterceptorConfig，通过 @Configuration 注解，Spring 会加载该类，并完成装配。

`addInterceptors` 方法中设置拦截器，并拦截所有请求。

`jwtDemoConfig` 方法中注入 JwtConfig，并设置了 HMACKey。

`jwtDemoService` 方法会根据注入的 JwtConfig 配置，生成具体的 JwtService，这里是 HMACJwtServiceImpl。

```java
@Configuration
public class InterceptorConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(jwtDemoInterceptor()).addPathPatterns("/**");
    }

    @Bean
    public JwtDemoInterceptor jwtDemoInterceptor() {
        return new JwtDemoInterceptor();
    }

    @Bean
    public JwtConfig jwtDemoConfig() {
        JwtConfig jwtConfig = new JwtConfig();
        jwtConfig.setHmacKey("cb9915297c8b43e820afd2a90a1e36cb");

        return jwtConfig;
    }

    @Bean
    public JwtService jwtDemoService() {
        return JwtUtils.obtainJwtService(jwtDemoConfig());
    }

}
```

###  编写测试 Controller

```java
@RestController
public class UserController {

    @Autowired
    private ObjectMapper objectMapper;

    @Autowired
    private JwtService jwtService;

    @GetMapping("/sign")
    @AuthRequired(required = false)
    public String sign() throws JsonProcessingException {

        UserDTO userDTO = new UserDTO();
        userDTO.setName("fatfoo");
        userDTO.setPassword("112233");
        userDTO.setSex(0);

        String payload = objectMapper.writeValueAsString(userDTO);

        return jwtService.sign(payload);
    }

    @GetMapping("/verify")
    public UserDTO verify() throws IOException {
        String payload = (String) JwtContext.getPayload();
        return objectMapper.readValue(payload, UserDTO.class);
    }
}
```

`sign` 方法对用户信息进行签名并返回 Token；由于 `@AuthRequired(required = false)` 拦截器将不会对其进行拦截。

`verify` 方法在 Token 通过验证后，获取解析出的信息并返回。

### 用 Postman 进行测试

访问 sign 接口，返回签名 Token。

![](http://img.uprogrammer.cn/static/20200911172638.png)

在 Header 中添加 Token 信息，请求 verify 接口，返回用户信息。

![](http://img.uprogrammer.cn/static/20200911172829.png)

### 测试 RSA 算法实现

上面我们只设置了 JwtConfig 的 hmacKey 参数，使用的是 HMAC 算法进行签名和验证。本节我们演示 RSA 算法进行签名和验证的实现。

#### 生成签名文件

使用 Java 自带的 keytool 工具可以方便的生成证书文件。

```shell
➜  resources git:(master) ✗ keytool -genkey -alias jwt -keyalg RSA -keystore jwt.jks
输入密钥库口令:
密钥库口令太短 - 至少必须为 6 个字符
输入密钥库口令: ronjwt
再次输入新口令: ronjwt
您的名字与姓氏是什么?
  [Unknown]:  ron
您的组织单位名称是什么?
  [Unknown]:  ron
您的组织名称是什么?
  [Unknown]:  ron
您所在的城市或区域名称是什么?
  [Unknown]:  Xiamen
您所在的省/市/自治区名称是什么?
  [Unknown]:  Fujian
该单位的双字母国家/地区代码是什么?
  [Unknown]:  CN
CN=ron, OU=ron, O=ron, L=Xiamen, ST=Fujian, C=CN是否正确?
  [否]:  是

输入 <jwt> 的密钥口令
	(如果和密钥库口令相同, 按回车):

Warning:
JKS 密钥库使用专用格式。建议使用 "keytool -importkeystore -srckeystore jwt.jks -destkeystore jwt.jks -deststoretype pkcs12" 迁移到行业标准格式 PKCS12。
```

文件生成后，复制到项目的 resources 目录下。

#### 设置 JwtConfig 参数

修改上节 InterceptorConfig 中的 `jwtDemoConfig` 方法，这是 jksFileName、jksPassword、certPassword 3 个参数。

```java
@Bean
public JwtConfig jwtDemoConfig() {
    JwtConfig jwtConfig = new JwtConfig();
//        jwtConfig.setHmacKey("cb9915297c8b43e820afd2a90a1e36cb");

    jwtConfig.setJksFileName("jwt.jks");
    jwtConfig.setJksPassword("ronjwt");
    jwtConfig.setCertPassword("ronjwt");
    return jwtConfig;
}
```

不要设置 hmacKey 参数，否则会加载 HMACJwtServiceImpl。因为 `JwtUtils#obtainJwtService` 方法实现如下：

```java
/**
 * 获取内置 JwtService 的工厂方法。
 *
 * 优先采用 HMAC 算法实现
 *
 * @param jwtConfig
 * @return
 */
public static JwtService obtainJwtService(JwtConfig jwtConfig) {
    if (!JwtUtils.isEmpty(jwtConfig.getHmacKey())) {
        return new HMACJwtServiceImpl(jwtConfig);
    }

    return new RSAJwtServiceImpl(jwtConfig);
}
```

这样就可以进行 RSA 算法签名与验证的测试了。运行程序并使用 Postman 测试，请自行查看区别。

###### - End -

---

本文只是 Spring Boot 集成 JWT 的第一篇，在后续我们还将继续对这个库进行封装，构建 spring-boot-starter，自定义 @Enable 注解来方便在项目中引入。

请关注我的公众号：精进Java(ID：**craft4j**)，第一时间获取知识动态。

如果你对项目的完整源码感兴趣，可以在公众号中回复 `jwt` 来获取。