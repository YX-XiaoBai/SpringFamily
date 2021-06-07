**作者**：[*帅的人都关注了*](https://juejin.cn/user/1688446223263511/posts)

**Github**：[*Github*](https://github.com/YX-XiaoBai/)

**CSDN**：[*进去看看??*](https://blog.csdn.net/weixin_44425934)

**爱好**：*Americano More Ice !*

**项目源码**: [*项目Github源码！Star完再走*](https://github.com/YX-XiaoBai/SpringFamily/tree/master/docker-demo)

# 目录

- [目录](#目录)
    - [前言](#前言)
    - [什么是Docker镜像](#什么是docker镜像)
    - [Dockerfile常用命令快查表](#dockerfile常用命令快查表)
    - [Maven构建Docker镜像-三步曲](#maven构建docker镜像-三步曲)
        - [1.准备工作](#1准备工作)
        - [2.执行构建](#2执行构建)
        - [3.检查结果](#3检查结果)
    - [扩展](#扩展)


## 前言

> `Docker`大部分应用在提供一些基础设施，如`Redis`/`Mongo`/`Mysql`等，本文主要讲述将自己编写的`Springboot`应用程序打包成`docker`镜像分发出去

## 什么是Docker镜像

掌握核心四点

- 它是一个静态的只读模板
- 它包含构建`Docker`容器的指令
- 它是分层的
- 它通过`Dockerfile`来创建镜像

## Dockerfile常用命令快查表

| 命令 | 作用 | 例子 |
| -- | -- | -- |
| FROM | 基于什么镜像 | FROM <image>[:<tag>] [AS <name>] |
| LABEL | 设置标签 | LABEL maintainer="Geektime" |
| RUN        | 运行安装命令         | RUN ["executable", "param1", "param2"]        |
| CMD        | 容器启动时命令(只能有一个) | CMD ["executable", "param1", "param2"]        |
| ENTRYPOINT | 容器启动后命令(只能有一个) | ENTRYPOINT ["executable", "param1", "param2"] |
| VOLUME     | 挂载命令           | VOLUME ["/data"]                              |
| EXPOSE     | 容器监听端口         | EXPOSE <port> [<port>/<protocol>]             |
| ENV        | 设置环境变量         | ENV <ket> <value>                             |
| ADD        | 添加文件           | ADD [--chown=<user>:<group>] <src>...<dest>   |
| WORKDIR    | 设置运行工作目录       | WORKDIR /path/to/workdir                      |
| USER       | 设置运行用户         | USER <user>[:<group>]                         |


## Maven构建Docker镜像-三步曲

### 1.准备工作

- 来看下 `Dockerfile` 文件

```sh
FROM java:8
EXPOSE 8080
ARG JAR_FILE
ADD target/${JAR_FILE} /waiter-service.jar
ENTRYPOINT ["java", "-jar","/waiter-service.jar"]
```

- 再看下`pox.xml`核心部分，配置`dockerfile-maven-plugin`

```xml
<plugin>
        <groupId>com.spotify</groupId>
        <artifactId>dockerfile-maven-plugin</artifactId>
        <version>1.4.10</version>
        <executions>
                <execution>
                        <id>default</id>
                        <goals>
                                <goal>build</goal>
                                <goal>push</goal>
                        </goals>
                </execution>
        </executions>
        <configuration>
                <repository>${docker.image.prefix}/${project.artifactId}</repository>
                <tag>${project.version}</tag>
                <buildArgs>
                        <JAR_FILE>${project.build.finalName}.jar</JAR_FILE>
                </buildArgs>
        </configuration>
</plugin>
```

### 2.执行构建


- 通过`maven`命令来打包生成镜像

```sh
$ mvn clean package -Dmaven.test.skip=true
[INFO] Scanning for projects...
[INFO]
[INFO] -------------< geektime.spring.springbucks:waiter-service >-------------
[INFO] Building waiter-service 0.0.1-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
...
...
...
[INFO] Step 2/5 : EXPOSE 8080
[INFO]
[INFO]  ---> Running in 6ae617f0acdf
[INFO] Removing intermediate container 6ae617f0acdf
[INFO]  ---> 6b182f57247c
[INFO] Step 3/5 : ARG JAR_FILE
[INFO]
[INFO]  ---> Running in 9a25b1d1b38c
[INFO] Removing intermediate container 9a25b1d1b38c
[INFO]  ---> 863117cd06f4
[INFO] Step 4/5 : ADD target/${JAR_FILE} /waiter-service.jar
[INFO]
[INFO]  ---> 064b021fac28
[INFO] Step 5/5 : ENTRYPOINT ["java", "-jar","/waiter-service.jar"]
[INFO]
[INFO]  ---> Running in c8b3c173ebad
[INFO] Removing intermediate container c8b3c173ebad
[INFO]  ---> 85fd7a4f7f3d
[INFO] Successfully built 85fd7a4f7f3d
[INFO] Successfully tagged springbucks/waiter-service:0.0.1-SNAPSHOT
[INFO]
[INFO] Detected build of image with id 85fd7a4f7f3d
[INFO] Building jar: /Users/lidean/gitee/geektime-spring-family/Chapter 10/docker-demo/target/waiter-service-0.0.1-SNAPSHOT-docker-info.jar
[INFO] Successfully built springbucks/waiter-service:0.0.1-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  07:18 min
[INFO] Finished at: 2021-06-07T10:35:33+08:00
[INFO] ------------------------------------------------------------------------
```

可以看到已经构建完了

### 3.检查结果

我们来看下打包好的镜像是否部署成功

```sh
$ docker images
REPOSITORY                                                    TAG                                              IMAGE ID            CREATED              SIZE
springbucks/waiter-service                                    0.0.1-SNAPSHOT                                   85fd7a4f7f3d        About a minute ago   683MB
```

通过`docker run`来执行镜像，再通过`docker logs`可以看到它还在启动

```sh
$ docker run --name waiter-service -d -p 8080:8080 springbucks/waiter-service:0.0.1-SNAPSHOT
28eba6685144634016d7e73c22000265aa14713276242505684e7eb62241954e
$ docker logs waiter-service

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.1.3.RELEASE)

2021-06-07 06:36:36.084  INFO 1 --- [           main] g.s.s.waiter.WaiterServiceApplication    : Starting WaiterServiceApplication v0.0.1-SNAPSHOT on 28eba6685144 with PID 1 (/waiter-service.jar started by root in /)
2021-06-07 06:36:36.101  INFO 1 --- [           main] g.s.s.waiter.WaiterServiceApplication    : No active profile set, falling back to default profiles: default
2021-06-07 06:36:38.937  INFO 1 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data repositories in DEFAULT mode.
2021-06-07 06:36:39.323  INFO 1 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 344ms. Found 2 repository interfaces.
2021-06-07 06:36:41.811  INFO 1 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.transaction.annotation.ProxyTransactionManagementConfiguration' of type [org.springframework.transaction.annotation.ProxyTransactionManagementConfiguration$$EnhancerBySpringCGLIB$$f2e71ce] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2021-06-07 06:36:43.553  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2021-06-07 06:36:43.683  INFO 1 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2021-06-07 06:36:43.684  INFO 1 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.16]
2021-06-07 06:36:43.727  INFO 1 --- [           main] o.a.catalina.core.AprLifecycleListener   : The APR based Apache Tomcat Native library which allows optimal performance in production environments was not found on the java.library.path: [/usr/java/packages/lib/amd64:/usr/lib/x86_64-linux-gnu/jni:/lib/x86_64-linux-gnu:/usr/lib/x86_64-linux-gnu:/usr/lib/jni:/lib:/usr/lib]
2021-06-07 06:36:44.205  INFO 1 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2021-06-07 06:36:44.206  INFO 1 --- [           main] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 7871 ms
2021-06-07 06:36:44.727  INFO 1 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
2021-06-07 06:36:45.430  INFO 1 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
2021-06-07 06:36:46.171  INFO 1 --- [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [
	name: default
	...]
2021-06-07 06:36:46.770  INFO 1 --- [           main] org.hibernate.Version                    : HHH000412: Hibernate Core {5.3.7.Final}
2021-06-07 06:36:46.775  INFO 1 --- [           main] org.hibernate.cfg.Environment            : HHH000206: hibernate.properties not found
2021-06-07 06:36:47.578  INFO 1 --- [           main] o.hibernate.annotations.common.Version   : HCANN000001: Hibernate Commons Annotations {5.0.4.Final}
2021-06-07 06:36:49.212  INFO 1 --- [           main] org.hibernate.dialect.Dialect            : HHH000400: Using dialect: org.hibernate.dialect.H2Dialect
2021-06-07 06:36:52.227  INFO 1 --- [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
2021-06-07 06:36:55.609  INFO 1 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2021-06-07 06:36:55.815  WARN 1 --- [           main] aWebConfiguration$JpaWebMvcConfiguration : spring.jpa.open-in-view is enabled by default. Therefore, database queries may be performed during view rendering. Explicitly configure spring.jpa.open-in-view to disable this warning
2021-06-07 06:36:56.808  INFO 1 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 15 endpoint(s) beneath base path '/actuator'
2021-06-07 06:36:57.082  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2021-06-07 06:36:57.089  INFO 1 --- [           main] g.s.s.waiter.WaiterServiceApplication    : Started WaiterServiceApplication in 22.508 seconds (JVM running for 24.292)
```

再检查下是否运行成功了，可以看到已经成功运行并映射到对应端口了

```sh
$ docker ps | grep springbucks/waiter-service:0.0.1-SNAPSHOT
28eba6685144        springbucks/waiter-service:0.0.1-SNAPSHOT   "java -jar /waiter-s…"   3 minutes ago       Up 3 minutes        0.0.0.0:8080->8080/tcp   waiter-service
```

最后，我们访问下实际端口，看是否返回对应数据

```sh
$ curl http://127.0.0.1:8080/coffee/1
{
  "id" : 1,
  "createTime" : "2021-06-07T14:36:45.551+0800",
  "updateTime" : "2021-06-07T14:36:45.551+0800",
  "name" : "espresso",
  "price" : 20.00
}%
```

## 扩展

`dockerfile-maven-plugin`也提供了一些辅助的方法，可以通过`deploy`到我指定的仓库里面。

可以尝试本地构建一个自己的 `docker` 镜像仓库 `push` 上去

#### 结束语：如果遇到什么疑问或者建议的，可直接留言评论！作者看到会马上一一回复

#### 如果觉得小白此文章不错或对你有所帮助，期待你的一键三连💫！❤️ ni ~