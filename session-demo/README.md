# 分布式Session解决方案

**作者**：[*关注了吗??*](https://juejin.cn/user/1688446223263511/posts)

**Github**：[*Github*](https://github.com/YX-XiaoBai/)

**CSDN**：[*进去看看??*](https://blog.csdn.net/weixin_44425934)

**爱好**：*Americano More Ice !*

# 目录

- [目录](#目录)
    - [Session](#session)
    - [问题&痛点](#问题痛点)
        - [如何解决？](#如何解决)
    - [手把手-解决分布式共享Session](#手把手-解决分布式共享session)
        - [前置准备](#前置准备)
        - [话不多说，实战为主](#话不多说实战为主)

## Session

首先，要简单来说下什么是 `Session`?

- 存储: 存储在服务端，`key-value`数据结构存储
- 常用场景: 放入 `Cookie` 中用于会话保持
- 逻辑: `client` 首次访问 `server` 时会响应一个 `Session id`， `client` 暂存到本地 `Cookie` 中，后面访问时会将 `cookie` 中的 `sessionId` 放到 `headers` 中访问服务器，若没有找到服务器会创建一个新的 `Session id` 响应给 `client`

## 问题&痛点

在分布式集群中，必须要将 `Session` 进行共享。快问为什么，然后我告诉你！当然你不问，我也要告诉你hh🤭🤭

因为我们常常会遇到类似这样的问题，集群一般都会进行负载均衡，当一个 `client` 访问服务端时(被分配到其中一个资源节点)，而由于某种原因导致触发负载均衡机制 `client` 被分配到另一个资源节点，此时新分配的资源节点没有存储 `client` 的 `session` 信息，比如 `client` 已登录没拿到 `session` 可能会强制登出让用户重新登录。

### 如何解决？

本文主要推荐 `Spring-Session` 集成的解决方案，整体思路是将 `Session` 放在 `Redis` 中进行共享

- [分布式共享Session-源码](https://github.com/YX-XiaoBai/SpringFamily/tree/master/session-demo)

## 手把手-解决分布式共享Session

### 前置准备

- `Docker`
- `IDEA`编译环境

### 话不多说，实战为主

1. 下载`redis`（默认为`lastest`版本），作者下的是5.0版本的哈

```shell
$ docker pull redis:5.0
5.0: Pulling from library/redis
69692152171a: Already exists
a4a46f2fd7e0: Pull complete
bcdf6fddc3bd: Pull complete
1f499504197d: Pull complete
021b18181099: Pull complete
1fb4123902bc: Pull complete
Digest: sha256:c2b0f6fe0588f011c7ed7571dd5a13de58cff538e08d100f0a197a71ea35423a
Status: Downloaded newer image for redis:5.0
docker.io/library/redis:5.0
```

2. 运行容器

```shell
$ docker run -itd --name redis-test -p 6379:6379 redis
169ad20a55ab0a9e1f05ec237bbfe01dcd8011c4b937ff17a500d6b4210f6165
# 检查是否redis已经启动
$ docker ps | grep redis
169ad20a55ab        redis:5.0                "docker-entrypoint.s…"   5 hours ago         Up 5 hours          0.0.0.0:6379->6379/tcp   redis-test
```

3. 这是 `pox.xml` 文件，只列举关键部分，即`redis`依赖

```xml
...
    <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
            <groupId>org.springframework.session</groupId>
            <artifactId>spring-session-data-redis</artifactId>
    </dependency>
...
```

4. 写一个 `restful` 方法用于发请求，此块代码最重要的就是 `@EnableRedisHttpSession`，下面也会着重讲下这个注解的源码实现

```java
package geektime.spring.web.session;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.session.data.redis.config.annotation.web.http.EnableRedisHttpSession;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.servlet.http.HttpSession;

@SpringBootApplication
@RestController
# 这里是重点
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 86400*30)
public class SessionDemoApplication {

	public static void main(String[] args) {
		SpringApplication.run(SessionDemoApplication.class, args);
	}

	@RequestMapping("/hello")
	public String printSession(HttpSession session, String name) {
		String storedName = (String) session.getAttribute("name");
		if (storedName == null) {
			session.setAttribute("name", name);
			storedName = name;
		}
		return "hello " + storedName;
	}

}
```

> 对于重点，`@EnableRedisHttpSession`注解源码分析

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
// 注意
@Import({RedisHttpSessionConfiguration.class})
@Configuration
// 实现的接口注解
public @interface EnableRedisHttpSession {
    // Sessio默认过期时间，30*60s=1800s=30min
    int maxInactiveIntervalInSeconds() default 1800;
    // Session默认命名空间，`spring:session`
    String redisNamespace() default "spring:session";
    // 刷新Redis的Session默认方式，`ON_SAVE`: 仅当Response提交后Session提交到Redis
    RedisFlushMode redisFlushMode() default RedisFlushMode.ON_SAVE;
    // 清楚过期Session默认时间，1min
    String cleanupCron() default "0 * * * * *";
}
```

上面这段代码`注意`的部分，`RedisHttpSessionConfiguration` 是实现的关键，我们进去会发现它实则是继承了 `SpringHttpSessionConfiguration`

```java
public class RedisHttpSessionConfiguration 
        extends SpringHttpSessionConfiguration 
            implements BeanClassLoaderAware, EmbeddedValueResolverAware, 
                ImportAware, SchedulingConfigurer {
                ...
}
```

我们在进去 `SpringHttpSessionConfiguration` 看看它的实现，里面注册了个 `SessionRepositoryFilter`


```java
@Configuration
public class SpringHttpSessionConfiguration implements ApplicationContextAware {
    ...
    @Bean
    public <S extends Session> SessionRepositoryFilter<? extends Session> springSessionRepositoryFilter(SessionRepository<S> sessionRepository) {
        SessionRepositoryFilter<S> sessionRepositoryFilter = new SessionRepositoryFilter(sessionRepository);
        sessionRepositoryFilter.setServletContext(this.servletContext);
        sessionRepositoryFilter.setHttpSessionIdResolver(this.httpSessionIdResolver);
        return sessionRepositoryFilter;
    }
    ...
}
```

而且缺少了一个 `sessionRepository` 参数，是在 `RedisHttpSessionConfiguration` 就被注入的，已经实现 `session` 配置的核心代码


```java
public class RedisHttpSessionConfiguration extends SpringHttpSessionConfiguration implements BeanClassLoaderAware, EmbeddedValueResolverAware, ImportAware, SchedulingConfigurer {
    ...
    @Bean
    public RedisOperationsSessionRepository sessionRepository() {
        RedisTemplate<Object, Object> redisTemplate = this.createRedisTemplate();
        RedisOperationsSessionRepository sessionRepository = new RedisOperationsSessionRepository(redisTemplate);
        sessionRepository.setApplicationEventPublisher(this.applicationEventPublisher);
        if (this.defaultRedisSerializer != null) {
            sessionRepository.setDefaultSerializer(this.defaultRedisSerializer);
        }

        sessionRepository.setDefaultMaxInactiveInterval(this.maxInactiveIntervalInSeconds);
        if (StringUtils.hasText(this.redisNamespace)) {
            sessionRepository.setRedisKeyNamespace(this.redisNamespace);
        }

        sessionRepository.setRedisFlushMode(this.redisFlushMode);
        int database = this.resolveDatabase();
        sessionRepository.setDatabase(database);
        return sessionRepository;
    }
    ...
}
```

5. 刚刚来了点小插曲，现在让我们启动我们的 `springboot` 重新，成功后 `console` 会打印如下信息

```sh
/Library/Java/JavaVirtualMachines/jdk-15.0.1.jdk/Contents/Home/bin/java -javaagent:/Applications/IntelliJ IDEA CE.app/Contents/lib/idea_rt.jar=60016:/Applications/IntelliJ IDEA CE.app/Contents/bin -Dfile.encoding=UTF-8 -classpath /Users/lidean/gitee/geektime-spring-family/Chapter 8/session-demo/target/classes:/Users/lidean/.m2/repository/org/springframework/boot/spring-boot-starter-web/2.1.3.RELEASE/spring-boot-starter-web-2.1.3.RELEASE.jar:/Users/lidean/.m2/repository/org/springframework/boot/spring-boot-starter/2.1.3.RELEASE/spring-boot-starter-2.1.3.RELEASE.jar:/Users/lidean/.m2/repository/org/springframework/boot/spring-boot/2.1.3.RELEASE/spring-boot-2.1.3.RELEASE.jar:/Users/lidean/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/2.1.3.RELEASE/spring-boot-autoconfigure-2.1.3.RELEASE.jar:/Users/lidean/.m2/repository/org/springframework/boot/spring-boot-starter-logging/2.1.3.RELEASE/spring-boot-starter-logging-2.1.3.RELEASE.jar:/Users/lidean/.m2/repository/ch/qos/logback/logback-classic/1.2.3/logback-classic-1.2.3.jar:/Users/lidean/.m2/repository/ch/qos/logback/logback-core/1.2.3/logback-core-1.2.3.jar:/Users/lidean/.m2/repository/org/apache/logging/log4j/log4j-to-slf4j/2.11.2/log4j-to-slf4j-2.11.2.jar:/Users/lidean/.m2/repository/org/apache/logging/log4j/log4j-api/2.11.2/log4j-api-2.11.2.jar:/Users/lidean/.m2/repository/org/slf4j/jul-to-slf4j/1.7.25/jul-to-slf4j-1.7.25.jar:/Users/lidean/.m2/repository/javax/annotation/javax.annotation-api/1.3.2/javax.annotation-api-1.3.2.jar:/Users/lidean/.m2/repository/org/yaml/snakeyaml/1.23/snakeyaml-1.23.jar:/Users/lidean/.m2/repository/org/springframework/boot/spring-boot-starter-json/2.1.3.RELEASE/spring-boot-starter-json-2.1.3.RELEASE.jar:/Users/lidean/.m2/repository/com/fasterxml/jackson/core/jackson-databind/2.9.8/jackson-databind-2.9.8.jar:/Users/lidean/.m2/repository/com/fasterxml/jackson/core/jackson-annotations/2.9.0/jackson-annotations-2.9.0.jar:/Users/lidean/.m2/repository/com/fasterxml/jackson/core/jackson-core/2.9.8/jackson-core-2.9.8.jar:/Users/lidean/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jdk8/2.9.8/jackson-datatype-jdk8-2.9.8.jar:/Users/lidean/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jsr310/2.9.8/jackson-datatype-jsr310-2.9.8.jar:/Users/lidean/.m2/repository/com/fasterxml/jackson/module/jackson-module-parameter-names/2.9.8/jackson-module-parameter-names-2.9.8.jar:/Users/lidean/.m2/repository/org/springframework/boot/spring-boot-starter-tomcat/2.1.3.RELEASE/spring-boot-starter-tomcat-2.1.3.RELEASE.jar:/Users/lidean/.m2/repository/org/apache/tomcat/embed/tomcat-embed-core/9.0.16/tomcat-embed-core-9.0.16.jar:/Users/lidean/.m2/repository/org/apache/tomcat/embed/tomcat-embed-el/9.0.16/tomcat-embed-el-9.0.16.jar:/Users/lidean/.m2/repository/org/apache/tomcat/embed/tomcat-embed-websocket/9.0.16/tomcat-embed-websocket-9.0.16.jar:/Users/lidean/.m2/repository/org/hibernate/validator/hibernate-validator/6.0.14.Final/hibernate-validator-6.0.14.Final.jar:/Users/lidean/.m2/repository/javax/validation/validation-api/2.0.1.Final/validation-api-2.0.1.Final.jar:/Users/lidean/.m2/repository/org/jboss/logging/jboss-logging/3.3.2.Final/jboss-logging-3.3.2.Final.jar:/Users/lidean/.m2/repository/com/fasterxml/classmate/1.4.0/classmate-1.4.0.jar:/Users/lidean/.m2/repository/org/springframework/spring-web/5.1.5.RELEASE/spring-web-5.1.5.RELEASE.jar:/Users/lidean/.m2/repository/org/springframework/spring-beans/5.1.5.RELEASE/spring-beans-5.1.5.RELEASE.jar:/Users/lidean/.m2/repository/org/springframework/spring-webmvc/5.1.5.RELEASE/spring-webmvc-5.1.5.RELEASE.jar:/Users/lidean/.m2/repository/org/springframework/spring-aop/5.1.5.RELEASE/spring-aop-5.1.5.RELEASE.jar:/Users/lidean/.m2/repository/org/springframework/spring-context/5.1.5.RELEASE/spring-context-5.1.5.RELEASE.jar:/Users/lidean/.m2/repository/org/springframework/spring-expression/5.1.5.RELEASE/spring-expression-5.1.5.RELEASE.jar:/Users/lidean/.m2/repository/org/springframework/boot/spring-boot-starter-data-redis/2.1.3.RELEASE/spring-boot-starter-data-redis-2.1.3.RELEASE.jar:/Users/lidean/.m2/repository/org/springframework/data/spring-data-redis/2.1.5.RELEASE/spring-data-redis-2.1.5.RELEASE.jar:/Users/lidean/.m2/repository/org/springframework/data/spring-data-keyvalue/2.1.5.RELEASE/spring-data-keyvalue-2.1.5.RELEASE.jar:/Users/lidean/.m2/repository/org/springframework/data/spring-data-commons/2.1.5.RELEASE/spring-data-commons-2.1.5.RELEASE.jar:/Users/lidean/.m2/repository/org/springframework/spring-tx/5.1.5.RELEASE/spring-tx-5.1.5.RELEASE.jar:/Users/lidean/.m2/repository/org/springframework/spring-oxm/5.1.5.RELEASE/spring-oxm-5.1.5.RELEASE.jar:/Users/lidean/.m2/repository/org/springframework/spring-context-support/5.1.5.RELEASE/spring-context-support-5.1.5.RELEASE.jar:/Users/lidean/.m2/repository/org/slf4j/slf4j-api/1.7.25/slf4j-api-1.7.25.jar:/Users/lidean/.m2/repository/io/lettuce/lettuce-core/5.1.4.RELEASE/lettuce-core-5.1.4.RELEASE.jar:/Users/lidean/.m2/repository/io/netty/netty-common/4.1.33.Final/netty-common-4.1.33.Final.jar:/Users/lidean/.m2/repository/io/netty/netty-handler/4.1.33.Final/netty-handler-4.1.33.Final.jar:/Users/lidean/.m2/repository/io/netty/netty-buffer/4.1.33.Final/netty-buffer-4.1.33.Final.jar:/Users/lidean/.m2/repository/io/netty/netty-codec/4.1.33.Final/netty-codec-4.1.33.Final.jar:/Users/lidean/.m2/repository/io/netty/netty-transport/4.1.33.Final/netty-transport-4.1.33.Final.jar:/Users/lidean/.m2/repository/io/netty/netty-resolver/4.1.33.Final/netty-resolver-4.1.33.Final.jar:/Users/lidean/.m2/repository/io/projectreactor/reactor-core/3.2.6.RELEASE/reactor-core-3.2.6.RELEASE.jar:/Users/lidean/.m2/repository/org/reactivestreams/reactive-streams/1.0.2/reactive-streams-1.0.2.jar:/Users/lidean/.m2/repository/org/springframework/session/spring-session-core/2.1.4.RELEASE/spring-session-core-2.1.4.RELEASE.jar:/Users/lidean/.m2/repository/org/springframework/spring-jcl/5.1.5.RELEASE/spring-jcl-5.1.5.RELEASE.jar:/Users/lidean/.m2/repository/org/springframework/session/spring-session-data-redis/2.1.4.RELEASE/spring-session-data-redis-2.1.4.RELEASE.jar:/Users/lidean/.m2/repository/org/springframework/spring-core/5.1.5.RELEASE/spring-core-5.1.5.RELEASE.jar geektime.spring.web.session.SessionDemoApplication

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.1.3.RELEASE)

2021-06-04 10:34:48.122  INFO 97269 --- [           main] g.s.web.session.SessionDemoApplication   : Starting SessionDemoApplication on MacBook-Pro-9.local with PID 97269 (started by lidean in /Users/lidean/gitee/geektime-spring-family/Chapter 8/session-demo)
2021-06-04 10:34:48.125  INFO 97269 --- [           main] g.s.web.session.SessionDemoApplication   : No active profile set, falling back to default profiles: default
2021-06-04 10:34:49.209  INFO 97269 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Multiple Spring Data modules found, entering strict repository configuration mode!
2021-06-04 10:34:49.214  INFO 97269 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data repositories in DEFAULT mode.
2021-06-04 10:34:49.247  INFO 97269 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 20ms. Found 0 repository interfaces.
2021-06-04 10:34:50.151  INFO 97269 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2021-06-04 10:34:50.181  INFO 97269 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2021-06-04 10:34:50.182  INFO 97269 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.16]
2021-06-04 10:34:50.194  INFO 97269 --- [           main] o.a.catalina.core.AprLifecycleListener   : The APR based Apache Tomcat Native library which allows optimal performance in production environments was not found on the java.library.path: [/Users/lidean/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:.]
2021-06-04 10:34:50.720  INFO 97269 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2021-06-04 10:34:50.720  INFO 97269 --- [           main] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 2477 ms
2021-06-04 10:34:51.578  INFO 97269 --- [           main] io.lettuce.core.EpollProvider            : Starting without optional epoll library
2021-06-04 10:34:51.579  INFO 97269 --- [           main] io.lettuce.core.KqueueProvider           : Starting without optional kqueue library
2021-06-04 10:34:51.970  INFO 97269 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2021-06-04 10:34:52.889  INFO 97269 --- [           main] s.a.ScheduledAnnotationBeanPostProcessor : No TaskScheduler/ScheduledExecutorService bean found for scheduled processing
2021-06-04 10:34:52.920  INFO 97269 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2021-06-04 10:34:52.924  INFO 97269 --- [           main] g.s.web.session.SessionDemoApplication   : Started SessionDemoApplication in 5.234 seconds (JVM running for 5.782)
2021-06-04 10:34:53.851  INFO 97269 --- [nio-8080-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2021-06-04 10:34:53.851  INFO 97269 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2021-06-04 10:34:53.861  INFO 97269 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 10 ms
```

6. 现在我们首次访问 `http://localhost:8080/hello?name=spring` ，记得打开 `Chrome tool`，可以看到被创建的`session`

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0ff4571f24bf40988cb31dffd55bb2ca~tplv-k3u1fbpfcp-watermark.image)

直接进入 `redis` 看下存储的 `k-v` 数据，当然具体存储的我们信息格式也是能看到的

```sh
$ docker exec -it redis-test redis-cli
127.0.0.1:6379> keys *
1) "spring:session:expirations:1622794680000"
2) "spring:session:sessions:expires:1f74b429-8695-47b9-9b24-7adde8bd6aec"
3) "spring:session:sessions:1f74b429-8695-47b9-9b24-7adde8bd6aec"
type spring:session:sessions:1f74b429-8695-47b9-9b24-7adde8bd6aec
hash
127.0.0.1:6379> HGETALL spring:session:sessions:1f74b429-8695-47b9-9b24-7adde8bd6aec
1) "sessionAttr:name"
2) "\xac\xed\x00\x05t\x00\x06spring"
3) "creationTime"
4) "\xac\xed\x00\x05sr\x00\x0ejava.lang.Long;\x8b\xe4\x90\xcc\x8f#\xdf\x02\x00\x01J\x00\x05valuexr\x00\x10java.lang.Number\x86\xac\x95\x1d\x0b\x94\xe0\x8b\x02\x00\x00xp\x00\x00\x01y\xd5\xfdO\n"
5) "maxInactiveInterval"
6) "\xac\xed\x00\x05sr\x00\x11java.lang.Integer\x12\xe2\xa0\xa4\xf7\x81\x878\x02\x00\x01I\x00\x05valuexr\x00\x10java.lang.Number\x86\xac\x95\x1d\x0b\x94\xe0\x8b\x02\x00\x00xp\x00\x00\a\b"
7) "lastAccessedTime"
8) "\xac\xed\x00\x05sr\x00\x0ejava.lang.Long;\x8b\xe4\x90\xcc\x8f#\xdf\x02\x00\x01J\x00\x05valuexr\x00\x10java.lang.Number\x86\xac\x95\x1d\x0b\x94\xe0\x8b\x02\x00\x00xp\x00\x00\x01y\xd5\xff7\x1e"
````

7. 我们尝试修改访问`url`的`请求参数`，会发现请求参数不管怎么改变都不影响返回的结果，因为服务端优先使用的`Redis`缓存的`Session`数据信息

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c32e76924da94230a958ae5d270246db~tplv-k3u1fbpfcp-watermark.image)

大概实现思路就是上面的内容了~


#### 结束语：如果遇到什么疑问或者建议的，可直接留言评论！作者看到会马上一一回复

#### 如果觉得小白此文章不错或对你有所帮助，期待你的一键三连💫！❤️ ni ~
