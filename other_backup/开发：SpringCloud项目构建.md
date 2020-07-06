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

测试目标：同一个服务中，从A库读出数据，再从B库读出数据。多数据源的集成也是为之后的合并部署做准备。

主要思路：自己初始化数据源的`SqlSessionFactoryBean`和`MapperScannerConfigurer`，注入多个数据源。数据源的配置可以从自定义的属性获取，每个数据源配置好扫码的mapper接口路径及xml路径即可。

我们知道，在传统的SSM集成中，集成Mybatis的时候，需要我们提供两个Bean，一个`SqlSessionFactoryBean`，还有一个是`MapperScannerConfigurer`。Spring Boot在检测到没有这两个Bean的时候，会自动为我们配置(@ConditionalOnMissingBean，参见集成Mybatis部分讲解)。现在我们要集成多个数据源，那么就需要数据源相关的初始化工作可以由我们自己来把控。也就是这两个Bean的初始化及注入需要我们自己来完成。自己的Bean注入之后，mybatis-stater就会取消自动注入。

整个数据源的自动配置可以参考mybatis-spring-boot-autoconfigure的实现。

#### 1.自定义配置

```java
// ---------------------------------
@Configuration
@EnableConfigurationProperties({RbowDatasourceProperties.class})
public class RbowDatasourceAutoConfig {...}
// ---------------------------------
@Getter
@Setter
@ConfigurationProperties(prefix="rbow")
public class RbowDatasourceProperties {
    // 多数据源配置
    private Map<String, RbowSingleDatasourceProperties> datasources;
}
// ---------------------------------
public class RbowSingleDatasourceProperties {
    // jdbc驱动类名称，默认com.mysql.cj.jdbc.Driver
    private String driverClassName = RbowDatasourceConstant.DEFAULT_DRIVER_NAME;
    // jdbc的url
    private String jdbcurl;
    // 数据库用户名
    private String username;
    // 数据库密码
    private String password;
    // mapper接口扫描包路径。如com.xx.xxx
    private String mapperInterfaceLocation;
    // mapper.xml扫描文件路径。如classpath*:com/xxx/mapper/**/*Mapper.xml
    private String mapperXmlLocation;
    // 事务控制所在的顶层包名，多个包用英文逗号隔开。如com.xx.xxx
    // private String transactionBasePackages;

    // =========================== Hikari连接池参数 ========================
    // 获取连接超时时间，默认30s
    private long connectionTimeout = 30000;
    // 验证一次数据库连接池连接是否为null的时间 默认3s
    private long validationTimeout = 3000;
    // 连接池最大连接（包括空闲和正在使用的连接）默认最大200
    private int maximumPoolSize = 200;
    // 连接池最小连接（包括空闲和正在使用的连接）默认最小10
    private int minimumIdle = 10;
    // 连接空闲时间。该设置仅适用于minimumIdle设置为小于maximumPoolSize的情况 默认:60000(1分钟)
    private long idleTimeout = 60000;
    // 一个连接的生命时长（毫秒），超时而且没被使用则被释放（retired），缺省:10分钟
    private long maxLifetime = 600000;
}
// ---------------------------------yml配置：
rbow:
  datasources:
    systemmana:
      jdbcurl: ${cdm_app_db_url:jdbc:mysql://rainbowdemo.com:33306/rbdm_syst?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai}
      username: ${cdm_app_db_username:root}
      password: ${cdm_app_db_password:123456}
      mapper-interface-location: com.rainbowdemo.service.basic.systemmana.mapper
      mapper-xml-location: classpath*:com/rainbowdemo/service/basic/systemmana/mapper/mybatis/**/*.xml
#      transaction-base-packages: asd
    systemmana_bk: # 备份数据源，用于测试多数据源注入
      jdbcurl: ${cdm_app_db_url:jdbc:mysql://rainbowdemo.com:33306/rbdm_syst_ds2bk?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai}
      username: ${cdm_app_db_username:root}
      password: ${cdm_app_db_password:123456}
      mapper-interface-location: com.rainbowdemo.service.basic.systemmana.mapperbk
      mapper-xml-location: classpath*:com/rainbowdemo/service/basic/systemmana/mapperbk/mybatis/**/*.xml
```

#### 2.自动配置+多数据源注入

使用`@EnableAutoConfiguration`允许自动配置

```java
// 注意，我取消了jdbc.DataSourceAutoConfiguration的自动注入。后面改成了自己注入数据源，所以不需要这个。
@SpringBootApplication
@EnableAutoConfiguration(excludeName = "org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration")
public class SystemmanaApplication {
    public static void main(String[] args) {
        SpringApplication.run(SystemmanaApplication.class);
    }
}
```

在spring.factories指明自动配置类

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.rainbow.starter.mybatis.autoconfig.RbowDatasourceAutoConfig
```

以下用了@Import，BeanDefinitionRegistry.registerBeanDefinition()两种注入方式。

自动配置类RbowDatasourceAutoConfig：

```java
/**
 * @Desc 数据源相关的自动配置入口
 *
 * 功能：完成Bean的初始化工作：RbowDatasourceInitInvoker, RbowDatasourceInitPostProcessor,
 *      Datasource, SqlSessionFactoryBean, MapperScannerConfigurer(tk的)
 * 初始化顺序：
 * 1.ImportBeanDefinitionRegistrar 接口为入口，在该接口的方法中手动注入RbowDatasourceInitPostProcessor的Bean
 *   。对应下面的@Import({RbowDatasourceAutoConfig.RegistPostProcessor.class})
 * 2.RbowDatasourceInitPostProcessor的Bean实现BeanPostProcessor接口，该接口的
 *   postProcessAfterInitialization方法监听容器Bean注入情况，监听ProxyTransactionManagementConfiguration注入
 * 3.本类的@EnableTransactionManagement注解触发ProxyTransactionManagementConfiguration注入
 * 4.监听到ProxyTransactionManagementConfiguration的Bean注入后，在监听的方法中实现RbowDatasourceInitInvoker这
 *   个Bean的初始化。对应下面的@Import({RbowDatasourceInitInvoker.RegistPostProcessor.class})
 * 5.RbowDatasourceInitInvoker的初始化触发数据源的注入及初始化
 * 6.数据源直接根据配置生成注入
 * 7.SqlSessionFactoryBean根据配置生成后，使用beanFactory手动注入
 * 8.MapperScannerConfigurer的Bean使用beanFactory无效，需要手动使用BeanDefinitionRegistry方式注入
 * 9.多数据源注入，beanName一样时，需要手动生成Bean名称。手动增加了UniqueBeanNameGenerator。
 *
 * 总体流程：
 * -> @Import({RbowDatasourceAutoConfig.RegistPostProcessor.class})完成注入，入口
 * -> RbowDatasourceInitPostProcessor.class完成注入，并监听ProxyTransactionManagementConfiguration
 * -> @Import({RbowDatasourceInitInvoker.class})
 * -> @EnableTransactionManagement，触发ProxyTransactionManagementConfiguration，从而触发RbowDatasourceInitInvoker.class初始化
 * -> 数据源注入
 * -> 配套的SqlSessionFactoryBean注入
 * -> 配套的MapperScannerConfigurer注入
 * -> 有多个数据源则循环多次
 *
 */
@Configuration
@EnableConfigurationProperties({RbowDatasourceProperties.class})
@Import({RbowDatasourceInitInvoker.class, RbowDatasourceAutoConfig.RegistPostProcessor.class})
@EnableTransactionManagement
public class RbowDatasourceAutoConfig {
//    @Bean
//    public InitTransactionalValue initTransactionalValue() {
//        return new InitTransactionalValue();
//    }

    // 注入RbowDatasourceInitPostProcessor
    public static class RegistPostProcessor implements ImportBeanDefinitionRegistrar {
        private static final String REGIST_BEAN_NAME = RbowDatasourceInitPostProcessor.class.getSimpleName();

        // 用于后续MapperScannerConfigurer的Bean注入
        @Getter
        private static BeanDefinitionRegistry registry;

        @Override
        public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
                                            BeanDefinitionRegistry registry) {
            if (registry.containsBeanDefinition(REGIST_BEAN_NAME)) {
                return;
            }

            RegistPostProcessor.registry = registry;    // 用于后续MapperScannerConfigurer的Bean注入

            // 注入RbowDatasourceInitPostProcessor
            GenericBeanDefinition beanDef = new GenericBeanDefinition();
            beanDef.setBeanClass(RbowDatasourceInitPostProcessor.class);
//            beanDef.setRole();??
            beanDef.setSynthetic(true); // 标记为由程序注入
            registry.registerBeanDefinition(REGIST_BEAN_NAME, beanDef);
        }
    }
}
```

RbowDatasourceInitPostProcessor，触发RbowDatasourceInitInvoker的初始化:

```java
/**
 * @Desc RbowDatasourceInitInvoker初始化
 *   由@EnableTransactionManagement触发ProxyTransactionManagementConfiguration
 *   从而触发RbowDatasourceInitInvoker的初始化
 */
public class RbowDatasourceInitPostProcessor implements BeanPostProcessor, Ordered {
    @Resource
    private BeanFactory beanFactory;

    @Override
    public int getOrder() {
        return 0;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        // ProxyTransactionManagementConfiguration的Bean由@Configuration类的@EnableTransactionManagement触发
        // 该Bean注入要早于dispatcherServlet
        if (bean instanceof ProxyTransactionManagementConfiguration) {
            // force initialization of this bean as soon as we see a
            // ProxyTransactionManagementConfiguration
            beanFactory.getBean(RbowDatasourceInitInvoker.class);
        }
        return bean;
    }
}
```

多数据源初始化（SqlSessionFactoryBean注入，MapperScannerConfigurer注入）：

```java
/**
 * @Desc 数据源配置初始化
 */
@Slf4j
public class RbowDatasourceInitInvoker extends AbstractDatasourceInitInvoker {

    public RbowDatasourceInitInvoker(final ConfigurableBeanFactory beanFactory,
                                     final RbowDatasourceProperties rbowDatasourceProperties) {
        // 初始化逻辑在抽象类里
        super(beanFactory, rbowDatasourceProperties);
    }

    @Override
    protected String initSqlSessionFactory(String dsName, DataSource ds, RbowSingleDatasourceProperties dsProp) {
        // 1.生成Bean
        SqlSessionFactoryBean fcBean = new SqlSessionFactoryBean();
        fcBean.setDataSource(ds);

        PathMatchingResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
        try {
            fcBean.setMapperLocations(resolver.getResources(dsProp.getMapperXmlLocation()));
        } catch (IOException e) {
            e.printStackTrace();
            log.error(e.getMessage(), e);
        }

        // 2.注册Bean
        String sqlSessionFactoryBeanName = super.addSuffixBeanClassName(
                dsName, RbowDatasourceConstant.SQL_SESSION_FACTORY_BEAN_NAME_SUFFIX);
        super.registerBean(sqlSessionFactoryBeanName, fcBean);
        return sqlSessionFactoryBeanName;
    }

    // 注意：因为使用了tk，注入的是tk的MapperScannerConfigurer
    @Override
    protected void initMapperScannerConfigurer(
            String dsName, String sqlSessionFactoryBeanName, RbowSingleDatasourceProperties dsProp) {
        // 1.生成Bean
        MapperScannerConfigurer scanConf = new MapperScannerConfigurer();
        // 1.1 配置tk特有的属性
        Properties prop = new Properties();
        prop.setProperty("IDENTITY", "MYSQL");
        prop.setProperty("notEmpty", "true");
        prop.setProperty("safeUpdate", "true");
        scanConf.setProperties(prop);
        // 1.2 配置其他特性
        scanConf.setSqlSessionFactoryBeanName(sqlSessionFactoryBeanName);
        scanConf.setBasePackage(dsProp.getMapperInterfaceLocation());
        scanConf.setNameGenerator(new UniqueBeanNameGenerator()); // 避免多数据源bean名称冲突

        // 2.注册Bean
        String scanConfBeanName = super.addSuffixBeanClassName(
                dsName, MapperScannerConfigurer.class.getSimpleName());
        // super.registerBean(scanConfBeanName, scanConf);
        // registerBean失效，必须采用postProcessBeanDefinitionRegistry注入
        scanConf.postProcessBeanDefinitionRegistry(RbowDatasourceAutoConfig.RegistPostProcessor.getRegistry());
    }
}
```

多数据源初始化（数据源注入）：

```java
/**
 * @Desc 数据源初始化，抽象类
 */
public abstract class AbstractDatasourceInitInvoker {
     private final ConfigurableBeanFactory beanFactory;

    // 在构造方法中初始化。加载Bean时即可自动执行。且自动获取入参的Bean。
    public AbstractDatasourceInitInvoker(final ConfigurableBeanFactory beanFactory,
             final RbowDatasourceProperties rbowDatasourceProperties) {
        this.beanFactory = beanFactory;
        this.injectDataSources(rbowDatasourceProperties);
    }

    // 把全部的数据源都进行注入到容器
    private void injectDataSources(RbowDatasourceProperties rbowDatasourceProperties) {
        // 1.多数据源配置校验
        this.validDatasourceProperties(rbowDatasourceProperties);

        // 2.根据多数据源配置将多数据源Bean注册至容器
        this.registDatasources(rbowDatasourceProperties);
    }

    // 多数据源配置校验
    private void validDatasourceProperties(RbowDatasourceProperties rbowDatasourceProperties) {
        // 1.入参不可为空
        Map<String, RbowSingleDatasourceProperties> dsProps = rbowDatasourceProperties.getDatasources();
        if (CollectionUtils.isEmpty(dsProps)) {
            throw new IllegalStateException("rbow.datasources未配置！");
        }

        // 2.各数据源的各个属性不可为空
        for (Map.Entry<String, RbowSingleDatasourceProperties> dsProp : dsProps.entrySet()) {
            RbowSingleDatasourceProperties properties = dsProp.getValue();

            boolean isAnyBlank = StringUtils.isEmpty(properties.getDriverClassName())
                    || StringUtils.isEmpty(properties.getJdbcurl())
                    || StringUtils.isEmpty(properties.getUsername())
                    || StringUtils.isEmpty(properties.getPassword())
                    || StringUtils.isEmpty(properties.getMapperInterfaceLocation())
                    || StringUtils.isEmpty(properties.getMapperXmlLocation());
            Assert.state(!isAnyBlank,
                    RbowSingleDatasourceProperties.class.getCanonicalName()+"attriutes未配置");
        }
    }

    // 根据多数据源配置将多数据源Bean注册至容器
    private void registDatasources(RbowDatasourceProperties rbowDatasourceProperties) {
        Map<String, RbowSingleDatasourceProperties> dsProps = rbowDatasourceProperties.getDatasources();

        dsProps.forEach((dsName, dsProp) -> {
            // 生成数据源
            DataSource ds = this.createDatasource(dsProp);
            // 注册数据源
            this.registDatasource(dsName, ds);
            // 初始化数据源。如：注入SqlSessionFactory和MapperScannerConfigurer等
            this.initDataSource(dsName, ds, dsProp);
        });
    }

    private DataSource createDatasource(RbowSingleDatasourceProperties properties) {
        return createHikariDataSource(properties);
    }

    private DataSource createHikariDataSource(RbowSingleDatasourceProperties dsProp) {
        HikariDataSource hds = new HikariDataSource();
        hds.setDriverClassName(dsProp.getDriverClassName());
        hds.setJdbcUrl(dsProp.getJdbcurl());
        hds.setUsername(dsProp.getUsername());
        hds.setPassword(dsProp.getPassword());
        //Hikari连接池其他配置
        hds.setConnectionTimeout(dsProp.getConnectionTimeout());
        hds.setValidationTimeout(dsProp.getValidationTimeout());
        hds.setIdleTimeout(dsProp.getIdleTimeout());
        hds.setMaxLifetime(dsProp.getMaxLifetime());
        hds.setMaximumPoolSize(dsProp.getMaximumPoolSize());
        hds.setMinimumIdle(dsProp.getMinimumIdle());
        return hds;
    }

    /**
     * 注册数据源
     * @param dsName 该数据源在application.yml配置中的名字
     */
    private void registDatasource(String dsName, DataSource ds) {
        String dsBeanName = addSuffixBeanClassName(dsName, HikariDataSource.class.getSimpleName());
        this.registerBean(dsBeanName, ds);
    }

    // 注册Bean
    protected void registerBean(String beanName, Object singletonObject) {
        beanFactory.registerSingleton(beanName, singletonObject);
    }

    // 拼接Bean名称
    protected String addSuffixBeanClassName(String datasourceName, String beanClassName) {
        return datasourceName + beanClassName;
    }


    // 初始化数据源。如：注入SqlSessionFactory和MapperScannerConfigurer等
    private void initDataSource(String dsName, DataSource ds, RbowSingleDatasourceProperties dsProp) {
        // 初始化SqlSessionFactory
        String sqlSessionFactoryBeanName = initSqlSessionFactory(dsName, ds, dsProp);

        // 初始化MapperScannerConfiguer
        this.initMapperScannerConfigurer(dsName, sqlSessionFactoryBeanName, dsProp);
    }

    // 注册SqlSessionFactory，并返回SqlSessionFactory的Bean名称
    protected abstract String initSqlSessionFactory(String dsName, DataSource ds, RbowSingleDatasourceProperties dsProp);

    protected abstract void initMapperScannerConfigurer(String dsName, String sqlSessionFactoryBeanName, RbowSingleDatasourceProperties dsProp);
}
```



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

合并部署时会有多数据源问题，已经在前面解决。

### 集成分布式事务seata



### 集成负载均衡、熔断、限流或serviceMesh



### 集成MQ



### 集成Job定时任务



### 集成Config动态配置中心



### 框架约定



### 其他

#### 多数据源

#### 分库分表

增加mybatis-plus

实践参考资料：[这里 ](http://springboot.javaboy.org/2019/0407/springboot-mybatis)

让微服务只允许网关直接访问

#### 跨域问题

app端集成：

1.注入CorsWebFilter的Bean:

```java
 */
@Configuration
public class CorsConfig {

    // 跨域第一步：关闭webFlux跨域
    @Bean
    public CorsWebFilter corsFilter() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedMethod("*");
        config.addAllowedOrigin("*");
        config.addAllowedHeader("*");

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);

        return new CorsWebFilter(source);
    }
}
```

2.注入WebMvcConfigurer配置Bean

```java
@Configuration
public class CrossConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("*")
                .allowedMethods("GET","HEAD","POST","PUT","DELETE","OPTIONS")
                .allowCredentials(true)
                .maxAge(3600)
                .allowedHeaders("*");
    }
}
```

网关端spring-cloud-gateway集成：

1.注入CorsWebFilter的Bean:

```java
 */
@Configuration
public class CorsConfig {

    // 跨域第一步：关闭webFlux跨域
    @Bean
    public CorsWebFilter corsFilter() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedMethod("*");
        config.addAllowedOrigin("*");
        config.addAllowedHeader("*");

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);

        return new CorsWebFilter(source);
    }
}
```

2.注入Filter

```java
public class CrossOriginFilter implements GlobalFilter, Ordered {
    private static final String ALL = "*";
    private static final String MAX_AGE = "18000L";

    // 跨域第二步：全局跨域Filter
    @Override
    public Mono<Void> filter(ServerWebExchange serverWebExchange, GatewayFilterChain gatewayFilterChain) {
        ServerHttpRequest request = serverWebExchange.getRequest();
//        if (!CorsUtils.isCorsRequest(request)) {
//            return gatewayFilterChain.filter(serverWebExchange);
//        }
        ServerHttpResponse response = serverWebExchange.getResponse();
        HttpHeaders headers = response.getHeaders();
        headers.add(HttpHeaders.ACCESS_CONTROL_ALLOW_ORIGIN, "*");
        headers.add(HttpHeaders.ACCESS_CONTROL_ALLOW_METHODS, "POST, GET, PUT, OPTIONS, DELETE, PATCH");
        headers.add(HttpHeaders.ACCESS_CONTROL_ALLOW_CREDENTIALS, "true");
        headers.add(HttpHeaders.ACCESS_CONTROL_ALLOW_HEADERS, "*");
        headers.add(HttpHeaders.ACCESS_CONTROL_EXPOSE_HEADERS, ALL);
        headers.add(HttpHeaders.ACCESS_CONTROL_MAX_AGE, MAX_AGE);
        if (request.getMethod() == HttpMethod.OPTIONS) {
            response.setStatusCode(HttpStatus.OK);
            return Mono.empty();
        }
        return gatewayFilterChain.filter(serverWebExchange);
    }

    @Override
    public int getOrder() {
        return -300;
    }
}
```



