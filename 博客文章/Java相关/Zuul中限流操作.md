# Zuul中限流操作

## 两种方案
1. 直接在网关中写Filter，在Filter中编写限流的逻辑
    1. 优点是，自己编写逻辑，代码灵活方便
    2. 缺点是，只支持单机模式。另外，需要自己实现限流算法或使用第三方库。一般使用谷歌的Guava RateLimiter
   
```java
package com.mininglamp.flowpanel.gateway.filter;
import com.google.common.util.concurrent.RateLimiter;
import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import com.netflix.zuul.exception.ZuulException;
import org.springframework.cloud.netflix.zuul.filters.support.FilterConstants;
import org.springframework.core.Ordered;
import javax.servlet.http.HttpServletResponse;

public class RateLimitFilter extends ZuulFilter {
    private final RateLimiter rateLimiter = RateLimiter.create(1);
    @Override
    public String filterType() {
        return FilterConstants.PRE_TYPE;
    }
    @Override
    public int filterOrder() {
        return Ordered.HIGHEST_PRECEDENCE;
    }
    @Override
    public boolean shouldFilter() {
        return true;
    }
    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletResponse response = ctx.getResponse();
        if (!rateLimiter.tryAcquire()) {
            String code = "TOO_MANY_REQUESTS";
            String msg = "请求超过限制";
            response.setContentType("application/json; charset=utf-8");
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(429);
            ctx.setResponseBody("{\"code\":\"" + code +"\", \"message\":  \""+msg+"\"}");
            return null;
        }
        return null;
    }
}
```
2. 使用`com.marcosbarbero.cloud:spring-cloud-zuul-ratelimit`包直接搭配Zuul做限流
    1. 优点是，这个包做了基础的和Zuul配合的基础工作，只需要在application.yml中配置规则即可
    2. 缺点也有，配置不如自己写代码灵活。

采用第二种方案。github地址：
[https://github.com/marcosbarbero/spring-cloud-zuul-ratelimit](https://github.com/marcosbarbero/spring-cloud-zuul-ratelimit)

## 配置
1. 添加依赖
```yml
implementation 'com.marcosbarbero.cloud:spring-cloud-zuul-ratelimit:2.4.2.RELEASE'
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
```

2. redis配置
```yml
spring:
  redis:
    port: 6666
    host: 192.168.1.101
    password: 1234
    timeout: 1000
    database: 0
```

3. 添加限流策略

## 原理
看其实现源码，也是实现了一个Filter。

```java
public class RateLimitFilter extends ZuulFilter {
    public static final String LIMIT_HEADER = "X-RateLimit-Limit";
    public static final String REMAINING_HEADER = "X-RateLimit-Remaining";
    public static final String RESET_HEADER = "X-RateLimit-Reset";

    public String filterType() {
        return "pre";
    }
    public int filterOrder() {
        return -1;
    }
    public boolean shouldFilter() {
        return this.properties.isEnabled() && this.policy(this.route()).isPresent();
    }
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletResponse response = ctx.getResponse();
        HttpServletRequest request = ctx.getRequest();
        Route route = this.route();
        this.policy(route).ifPresent((policy) -> {
            String key = this.rateLimitKeyGenerator.key(request, route, policy);
            Rate rate = this.rateLimiter.consume(policy, key);
            response.setHeader("X-RateLimit-Limit", policy.getLimit().toString());
            response.setHeader("X-RateLimit-Remaining", String.valueOf(Math.max(rate.getRemaining().longValue(), 0L)));
            response.setHeader("X-RateLimit-Reset", rate.getReset().toString());
            if(rate.getRemaining().longValue() < 0L) {
                ctx.setResponseStatusCode(HttpStatus.TOO_MANY_REQUESTS.value());
                ctx.put("rateLimitExceeded", "true");
                throw new ZuulRuntimeException(new ZuulException(HttpStatus.TOO_MANY_REQUESTS.toString(),                 HttpStatus.TOO_MANY_REQUESTS.value(), (String)null));
            }
        });
        return null;
    }
```

如果想看详细的算法解析，可参阅[这篇文章](https://segmentfault.com/a/1190000020745218)

## 限流规则配置
限流规则直接在`application.yml`中配置。

```yml
zuul:
  ratelimit:
    key-prefix: cbp-rate-limit    # key的前缀
    enabled: true                 # 是否启用限流
    repository: REDIS            # 数据存储，可选的有内存
    behind-proxy: true   
    default-policy:
        - limit: 5
          quota: 3
          refresh-interval: 15
    policy-list:        # 默认策略。针对所有路由的配置
      askaward:         # 这里指定的是serviceId，针对service进行限流设置
        - limit: 5       # 每个刷新时间窗口对应的请求数量限制
          quota: 3       # 每个刷新时间窗口对应的请求时间限制（秒）
          refresh-interval: 15    # 刷新时间窗口的时间，默认值 (秒)
          type:
            - url=/users
        - limit: 3      # 这里可以设置多个限流策略，针对同一个service，这样便可灵活设置限流策略。
          quota: 2
          refresh-interval: 5
          type:         # 每个策略的type可以设置多个，逻辑与的关系
            - url=/hello
```

### 限流策略
即上面的type设置。

| 限流粒度/类型 |            说明             |
| ----------- | -------------------------- |
| USER        | 使用经过身份验证的用户名或“匿名” |
| ORIGIN      | 使用用户原始请求              |
| URL         | 使用下游服务的请求路径         |
| ROLE        | 使用经过身份验证的用户角色      |
| HTTP_METHOD | 使用HTTP请求方法              |
| URL_PATTERN | 使用HTTP请求方法              |
| HTTP_HEADER | 使用HTTP请求方法              |

#### 一些说明
1. User或Role，如果使用了`Shiro`或`Spring Security`做认证鉴权，可以直接使用认证后的Principal。如果没有，则需要自己维护`request.UserPrincipal`

```java
//默认实现
public String key(final HttpServletRequest request, final Route route, final RateLimitProperties.Policy policy) {
    final List<Type> types = policy.getType();
    final StringJoiner joiner = new StringJoiner(":");
    joiner.add(properties.getKeyPrefix());
    if (route != null) {
        joiner.add(route.getId());
    }
    if (!types.isEmpty()) {
        if (types.contains(Type.URL) && route != null) {
            joiner.add(route.getPath());
        }
        if (types.contains(Type.ORIGIN)) {
            joiner.add(getRemoteAddr(request));
        }
        if (types.contains(Type.USER)) {
            joiner.add(request.getUserPrincipal() != null ? request.getUserPrincipal().getName() : ANONYMOUS_USER);
        }
    }
    return joiner.toString();
}
```
2. 这些策略可以组合使用。
3. 如果预设的策略不满足，可以自行编写key生成策略
    ```java
    @Bean
    public RateLimitKeyGenerator ratelimitKeyGenerator(RateLimitProperties properties, RateLimitUtils rateLimitUtils) {
        /**
         * 重写限流key获取机制
         * 限流计算就是使用这个key进行计算的
         */
        return new DefaultRateLimitKeyGenerator(properties, rateLimitUtils) {
            @Override
            public String key(HttpServletRequest request, Route route, RateLimitProperties.Policy policy) {
                String superKey = super.key(request, route, policy);
                String token = request.getHeader("x-auth-token");
                String keyStr = superKey;
                if (StringUtils.isNotBlank(token)) {
                    keyStr = superKey + ":" + token;
                }
                return keyStr;
            }
        };
    }
    ```

### 支持存储方式
- InMemory(ConcurrentHashMap)
- Redis
- Consul
- Spring Data JPA

如果只使用内存方式，那么将只支持单机部署。如需集群，则需要使用下面三种。我们使用Redis.
使用Redis只需要在`application.yml`中配置好Redis地址即可。

```yml
spring:
  redis:
    port: 6666
    host: 192.168.1.101
    password: 1234
    timeout: 1000
    database: 0
```

### 错误处理
默认如果超过限制，会返回429.错误如下：
![](https://img-vnote-1251075307.cos.ap-beijing.myqcloud.com/1630548877_20210902100629819_643274384.png)

默认的返回结构不符合我们现在的JSON格式，需要重写处理：

加一个Controller实现`ErrorController`，然后路由定义为`/error`。

```java
@RestController
public class AppErrorController implements ErrorController {

    private final static String ERROR_PATH = "/error";

    @Override
    public String getErrorPath() {
        return ERROR_PATH;
    }

    @RequestMapping(value = ERROR_PATH)
    @ResponseBody
    public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {

        Map<String, Object> result = new HashMap<>();

        Integer statusCode = (Integer) request.getAttribute("javax.servlet.error.status_code");

        if (statusCode == 429) {
            result.put("code", "too_many_requests");
            result.put("message", "请求太频繁，稍后再试");
        } else {
            result.put("code", "internal_server_error");
            result.put("message", "服务器异常");
        }

        return new ResponseEntity(result, HttpStatus.resolve(statusCode));
    }
}

```

#### 注意：
github文档中给出的一个重写ErrorHandler，并不是指返回的错误的Handler，而是指在处理限流时的错误。不要混淆。
```java
@Bean
public RateLimiterErrorHandler rateLimitErrorHandler() {
    return new DefaultRateLimiterErrorHandler() {
        @Override
        public void handleSaveError(String key, Exception e) {
            // 处理存储key异常
        }

        @Override
        public void handleFetchError(String key, Exception e) {
            // 处理查询key异常
        }

        @Override
        public void handleError(String msg, Exception e) {
            // 处理异常信息
        }
    }
}

```