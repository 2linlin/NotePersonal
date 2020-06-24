### 最简单的web服务

实现最简单的web服务，支持浏览器访问并返回字符串：

#### 1.实现spring-boot

pom:

```xml
<!-- 1.springboot服务启动核心包，支持SpringApplication.run(xxx.class)。 服务启动完就关闭。-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot</artifactId>
</dependency>
<!-- 2.自动配置包，支持@SpringBootApplication注解。服务启动完就关闭。-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure</artifactId>
</dependency>
```

java:

```java
@SpringBootApplication	// 增加jar包
public class SystemmanaApplication {
    public static void main(String[] args) {
        SpringApplication.run(SystemmanaApplication.class);	// 增加jar包
        System.out.println("started!");
    }
}


```

#### 2.实现tomcat服务器 + springmvc

pom:

```xml
<!-- 1.服务器包，支持tomcat。服务启动后不会立即关闭，会等待请求并响应。并且支持mvc相关注解，如@RestController、 @RequestMapping("/xx")-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

java:

```java
@RestController	// 注意是RestController，不是Controller。Controller是Spring的。
@RequestMapping("/hello")
public class DemoWebController {
    @RequestMapping("/hi")
    public String HelloBoot() {
        System.out.println("come in!");
        return "Hi!";
    }
}
```

### 集成Mybatis+Pagehelper

知识点



### 集成多数据源



### 集成Id生成器+Mybatis代码生成器



### 集成自动配置+自定义配置+自定义注解



### 集成log4j2日志



### 集成全局异常处理+事务+各种切面



### 集成Redis



### 集成微服务feign+eureka+gateway



### 集成权限验证+标准入参及校验+路由



### 集成加密及签名



### 集成多环境配置



### 集成测试框架



### 集成swagger



### 集成合并部署



### 集成分布式事务



### 集成负载均衡、熔断、限流或serviceMesh



### 集成MQ



### 集成Job定时任务



### 集成Config动态配置中心



### 框架约定



### 其他

#### 跨域问题

#### 多数据源

#### 分库分表





