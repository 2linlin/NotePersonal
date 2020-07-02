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

### 集成Mybatis

#### 1.配置pom

```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
</dependency>

<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
```

#### 2.配置application.yml数据源

```yml
spring:
  datasource:
    url: ${cdm_app_db_url:jdbc:mysql:///test01?useUnicode=true&characterEncoding=utf-8}
    username: ${cdm_app_db_username:root}
    password: ${cdm_app_db_password:root}
    # type=com.alibaba.druid.pool.DruidDataSource
```

#### 3.配置Mapper.class接口扫描路径

共有两种方式，最终采用了方式二。

- 方式一：直接在Mapper接口上加@Mapper注解
- 方式二：在XXXApplication.class启动类上配置@MapperScan注解

方式一：

```java
@Mapper
public interface UserMapper {String selectUserNameByUserId(Long userId);}
```

方式二：

```java
@SpringBootApplication
@MapperScan(basePackages = com)
public class SystemmanaApplication {
    public static void main(String[] args) {
        SpringApplication.run(SystemmanaApplication.class);
    }
}
```

#### 4.配置Mapper.xml文件扫描路径

共有三种主要的方式。最终采用了方式三

- 方式一：直接在接口方法上加@Select等注解指明执行语句。
- 方式二：Mapper.xml文件直接放到Mapper接口同一目录下。不过Maven打包时会忽略java目录下的mapper文件。需要在pom.xml配置打包时包含xml文件。
- 方式三：Mapper.xml文件放到resource目录下。此时只要在Application.class/application.yml启动类上配置好Mapper的扫描路径就OK。
- 方式四：这时一个有点geek的方法，xml文件放到resource目录下，但是目录结构路径和Mapper.java目录结构一模一样。这样maven打包时，.class文件就会和.xml文件打包到同一目录。这样就省去了方式二的配置xml文件打包，或者方式三的手动配置mapper.xml扫描路径。

方式一：

```java
public interface UserMapper {
    @Select("select t.user_name from t_syst_user t where t.user_id=#{userId}")
    String selectUserNameByUserId(Long userId);
}
```

方式二：

Mapper.java接口：

```java
public interface UserMapper {
    String selectUserNameByUserId(Long userId);
}
```

Mapper.xml文件，放到接口同一个目录下：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.rainbowdemo.service.basic.systemmana.mapper.UserMapper">
    <select id="selectUserNameByUserId"
            resultType="java.lang.String">
		select t.user_name from t_syst_user t where t.user_id=#{userId}
	</select>
</mapper>
```

pom.xml配置，maven打包时java目录下不要忽略xml文件

```xml
<project>    
    <build>
        <resources>
            <resource>
                <directory>src/main/java</directory>
                <includes>
                    <include>**/*.xml</include>
                </includes>
            </resource>
            <resource>
                <directory>src/main/resources</directory>
            </resource>
        </resources>
    </build>
</project>
```


方式三：

Mapper.java：

```java
public interface UserMapper {
    String selectUserNameByUserId(Long userId);
}
```
Mapper.xml，放到resource目录下：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.rainbowdemo.service.basic.systemmana.mapper.UserMapper">
    <select id="selectUserNameByUserId"
            resultType="java.lang.String">
		select t.user_name from t_syst_user t where t.user_id=#{userId}
	</select>
</mapper>
```


application.yml配置mapper.xml扫描路径

```yml
mybatis:
  mapper-locations: classpath:mapper/*.xml
```

#### 5.其他

在SSM整合中，开发者需要自己提供两个Bean，一个`SqlSessionFactory`，还有一个是`MapperScannerConfigurer`，在Spring Boot中，这两个东西虽然不用开发者自己提供了，但是并不意味着这两个Bean不需要了，在`org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration`类中，我们可以看到Spring Boot提供了这两个Bean，部分源码如下：

```java
@org.springframework.context.annotation.Configuration
@ConditionalOnClass({ SqlSessionFactory.class, SqlSessionFactoryBean.class })
@ConditionalOnSingleCandidate(DataSource.class)
@EnableConfigurationProperties(MybatisProperties.class)
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
public class MybatisAutoConfiguration implements InitializingBean {

  @Bean
  @ConditionalOnMissingBean
  public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
    SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
    factory.setDataSource(dataSource);
    return factory.getObject();
  }
    
  @Bean
  @ConditionalOnMissingBean
  public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
    ExecutorType executorType = this.properties.getExecutorType();
    if (executorType != null) {
      return new SqlSessionTemplate(sqlSessionFactory, executorType);
    } else {
      return new SqlSessionTemplate(sqlSessionFactory);
    }
  }
    
  @org.springframework.context.annotation.Configuration
  @Import({ AutoConfiguredMapperScannerRegistrar.class })
  @ConditionalOnMissingBean(MapperFactoryBean.class)
  public static class MapperScannerRegistrarNotFoundConfiguration implements InitializingBean {

    @Override
    public void afterPropertiesSet() {
      logger.debug("No {} found.", MapperFactoryBean.class.getName());
    }
  }
}
```

从类上的注解可以看出，当当前类路径下存在SqlSessionFactory、 SqlSessionFactoryBean以及DataSource时，这里的配置才会生效，SqlSessionFactory和SqlTemplate都被提供了。后面的多数据源配置会涉及到这里的内容。

### 集成tk-mybatis

整体步骤：

```txt
1.配置pom引入maven依赖
2.启动类XXXApplication.java或自定义Mybatis配置类上，使用tkMybatis的@MapperScan扫描Mapper接口
3.在application.yml文件中配置mapper.xml文件地址[可选]，自定义的SQL可以写在这些文件中
4.配置实体类注解，如@Table,@Id等
5.Mapper接口继承tkMybatis的Mapper接口
6.使用tkMybatis即可
```

#### 1.配置pom.xml

```xml
<dependency>
    <groupId>tk.mybatis</groupId>
    <artifactId>mapper-spring-boot-starter</artifactId>
    <version>2.0.2</version>
</dependency>
```

#### 2.更换@MapperScan接口扫码注解

把原来Mybatis的@MapperScan注解替换为tkMybatis的@MapperScan注解

```java
@SpringBootApplication
@MapperScan(basePackages = "com.rainbowdemo.service.basic.systemmana.mapper")
public class SystemmanaApplication {
    public static void main(String[] args) {
        SpringApplication.run(SystemmanaApplication.class);
    }
}
```

#### 3.配置Mapper.xml路径

需要写自定义的SQL的话就配下。直接用Mybatis原来的配置就行，不用改。用法就是原生的Mybatis用法不变。

```yml
mybatis:
  mapper-locations: classpath*:com/rainbowdemo/service/basic/systemmana/mapper/mybatis/**/*.xml
```

#### 4.配置实体类注解

主要是表注解`@Table`和字段注解`@Column`，属于`javax.persistence`包。操作SQL时会自动进行驼峰和蛇形命名法的转换，如果驼峰名称自动转换后格式可以匹配数据库的蛇形命名的话，就无需加这两个注解。如下，字段名可以对应，因此无需用注解。

```java
@Data
@Table(name = "t_syst_user")  // 用于tkMybatis
public class User {
//    @Id
//    @Column(name = "user_id")
    private Long userId;
//    @Column(name = "user_name")
    private String userName;
//    @Column(name = "nick_name")
    private String nickName;
//    @Column(name = "password")
    private String password;
//    @Column(name = "create_user")
    private Long createUser;
//    @Column(name = "update_user")
    private Long updateUser;
//    @Column(name = "remark")
    private String remark;
//    @Column(name = "delete_flag")
    private Byte deleteFlag;
}
```

#### 5.Mapper接口继承tk的Mapper接口

```java
public interface UserMapper extends tk.mybatis.mapper.common.Mapper<User> {
    String selectUserNameByUserId(Long userId);
}
```

#### 6.使用示例

```java
@Service
public class DemoWebService {
    @Resource
    private UserMapper userMapper;
	
    // 自定义方式
    public String selectUserNameByUserId(Req<Long> req) {
        return userMapper.selectUserNameByUserId(req.getBody());
    }
	
    // tk方式
    public User selectOneUserByUserId(Req<Long> req) {
        User user = new User();
        user.setUserId(req.getBody());
        return userMapper.selectOne(user);
    }
}
```

### 集成自定义配置+自动配置+多数据源

测试目标：从A库读出数据，把数据写入B库。

主要思路：自己初始化数据源的`SqlSessionFactoryBean`和`MapperScannerConfigurer`，注入多个数据源。数据源的配置可以从自定义的属性获取，每个数据源配置好扫码的mapper接口路径及xml路径即可。

我们知道，在传统的SSM集成中，集成Mybatis的时候，需要我们提供两个Bean，一个`SqlSessionFactoryBean`，还有一个是`MapperScannerConfigurer`。Spring Boot在检测到没有这两个Bean的时候，会自动为我们配置(@ConditionalOnMissingBean，参见集成Mybatis部分讲解)。现在我们要集成多个数据源，那么就需要数据源相关的初始化工作可以由我们自己来把控。也就是这两个Bean的初始化及注入需要我们自己来完成。自己的Bean注入之后，mybatis-stater就会取消自动注入。

整个数据源的自动配置可以参考mybatis-spring-boot-autoconfigure的实现。

#### 1.自定义配置+自动配置





### 集成Pagehelper+日志+事务管理



### 集成Sharding-jdbc



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



### 集成分布式事务seata



### 集成负载均衡、熔断、限流或serviceMesh



### 集成MQ



### 集成Job定时任务



### 集成Config动态配置中心



### 框架约定



### 其他

#### 跨域问题

#### 多数据源

#### 分库分表

增加mybatis-plus

实践参考资料：[这里](http://springboot.javaboy.org/2019/0407/springboot-mybatis)





