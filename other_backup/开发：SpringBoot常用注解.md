

Spring是个大容器，那么其实它的注解广义上可以分为两类：注册Bean的和使用Bean的。

### 注入第三方类的Bean

#### @Bean

1.@Bean专门用在方法上。

2.用于注入第三方Bean。自定义的Bean有@Component、@Controller、@Service、@Repository等。而如果想注入第三方，那么只有用@Bean和@Import。

3.用于动态注入。放在方法上提供了动态注入的能力。

实际上，@Bean注解和xml配置中的bean标签的作用是一样的

#### @[Import](https://www.cnblogs.com/yichunguo/p/12122598.html)

@Import主要用来导入第三方包的Bean，或者没配扫描时手动导入其他类，如加了@Configuration的配置类。

@Import只能用在类上。它有三种用法（实际就是注解中可以放三种class）：

- 直接填class数组方式

- ImportSelector方式【重点】

- ImportBeanDefinitionRegistrar方式

1、直接填class数组方式

实质是把要注入的Bean.class放到注解参数中。语法如下：

```javascript
@Import({ 类名.class , 类名.class... })
public class TestDemo {

}
```

注解参数中对应的Bean就直接加到容器中。此时容器中Bean的名称是**全类名**，例如：com.xxx.类名

2、ImportSelector接口方式【重点】

实质是把ImportSelector实现类的class放到注解参数中，然后要注入的Bean在实现类中标明。语法如下：

实现ImportSelector接口

```java
public class Myclass1 implements ImportSelector {
    // 1、返回值： 就是我们实际上要导入到容器中的组件全类名【重点 】
    // 2、参数： AnnotationMetadata表示当前被@Import注解给标注的所有注解信息【不是重点】
    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        return new String[]{"com.xxx.TestDemo1"};
    }
}
```

核心代码，放到@Import参数中。下面的`TestDemo2`和`TestDemo3`是用来做对比的。

```java
@Import({TestDemo2.class, Myclass1.class, Myclass2.class})
public class TestDemo {
        @Bean
        public TestDemo3 testDemo3(){
            return new TestDemo3();
        }
}
```

测试：

```java
// 打印容器中的组件测试
public class AnnotationTestDemo {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext=new AnnotationConfigApplicationContext(TestDemo.class);  //这里的参数代表要做操作的类

        String[] beanDefinitionNames = applicationContext.getBeanDefinitionNames();
        for (String name : beanDefinitionNames){
            System.out.println(name);
        }
    }
}
// 结果
......
testDemo
com.xxx.TestDemo2
com.xxx.TestDemo1
testDemo3
TestDemo4444
......
```

3、ImportBeanDefinitionRegistrar接口方式

实质是把ImportBeanDefinitionRegistrar实现类的class放到注解参数中。类似第2种用法。

实现接口：

```java
public class Myclass2 implements ImportBeanDefinitionRegistrar {
	//该实现方法默认为空
   	//  第一个参数：annotationMetadata 和之前的ImportSelector参数一样都是表示当前被@Import注解给标注的所有注解信息
	// 第二个参数表示用于注册定义一个bean
    @Override
    public void registerBeanDefinitions(AnnotationMetadata annotationMetadata, BeanDefinitionRegistry beanDefinitionRegistry) {
        //指定bean定义信息（包括bean的类型、作用域...）
        RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(TestDemo4.class);
        //注册一个bean指定bean名字（id）
        beanDefinitionRegistry.registerBeanDefinition("TestDemo4444",rootBeanDefinition);
    }
}
```

三种使用方式总结：

> 第一种用法：`@Import`（{ 要导入的容器中的组件 } ）：容器会自动注册这个组件，**id默认是全类名**
>
> 第二种用法：`ImportSelector`：返回需要导入的组件的全类名数组，springboot底层用的特别多【**重点** 】
>
> 第三种用法：`ImportBeanDefinitionRegistrar`：手动注册bean到容器

**以上三种用法方式皆可混合在一个@Import中使用，特别注意第一种和第二种都是以全类名的方式注册，而第三中可自定义方式。**

@Import注解本身在springboot中用的很多，特别是其中的第二种用法ImportSelector方式在springboot中使用的特别多，尤其要掌握！

### 注入application.yml文件配置Bean

注入application.yml文件配置常见有两种方式，@Value，@ConfigurationProperties

#### @Value

- 配置文件

```undefined
user:
  username: ls
  password: 456
  age: 99
```

注意：不要用 user:name , 因为这个是系统默认的名字 ，会被读到

- 把配置读取到对象中

```kotlin
@Component
public class User {
    //@Value :从配置文件中取值   SPEL
    @Value("${user.username}")
    private String username = "zs";
    @Value("${user.password}")
    private String password = "123";
    @Value("${user.age}")
    private int age = 18;
    . . . . . . 
}
```

#### @ConfigurationProperties

- 配置文件

```undefined
employee:
  username: ls
  password: 456
  age: 99
```

- 绑定配置对象

```java
@Component
@ConfigurationProperties(prefix = "employee")
public class Employee {
    private String username = "zs";
    private String password = "123";
    private int age = 18;
}
```

@ConfigurationProperties : 自动的根据前缀从配置中过滤出配置项目，然后根据当前对象的列名进行匹配，自动赋值

- 或者按需注入，以下是mybatis-starter自动注入的示例

```yml
mybatis:
  mapper-locations: classpath*:com/xxx/mapper/mybatis/**/*.xml
```

```java
@ConfigurationProperties(
    prefix = "mybatis"
)
public class MybatisProperties {...}
```

```java

@ConditionalOnClass({SqlSessionFactory.class, SqlSessionFactoryBean.class})// 非必须
@ConditionalOnSingleCandidate(DataSource.class)// 非必须
@AutoConfigureAfter({DataSourceAutoConfiguration.class})// 非必须
@Configuration
@EnableConfigurationProperties({MybatisProperties.class})
public class MybatisAutoConfiguration {...}
```



### [springboot声明bean的六种方式](https://www.jianshu.com/p/ed99d6111d46)

茴香豆的四种写法。。

----声明bean----
 **1.@Component 声明普通bean**
 **2.@Component 声明FactoryBean**

```java
@Component
public class MyFactory implements FactoryBean<MytestBean> {
 @Override
    public MytestBean getObject() throws Exception {
        return new MytestBean();
    }

    @Override
    public Class<?> getObjectType() {
        return MytestBean.class;
    }
}
```

**3.在配置类中使用@Bean**

```java
@Configuration
public class MyConfig {
    @Bean
    public Mytestbean create(){}
}
```

----手动注册BeanDefinition----
 **4.使用BeanDefinitionRegistryPostProcessor注册BeanDefinition**

```java
@Component
public class MyBeanRegister implements BeanDefinitionRegistryPostProcessor {
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
        beanDefinition.setBeanClass(MytestBean.class);
        beanDefinition.setSynthetic(true);
        MutablePropertyValues mpv = beanDefinition.getPropertyValues();
        mpv.addPropertyValue("age", "25");
        registry.registerBeanDefinition("mytestBean", beanDefinition);
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {

    }
}
```

**5.使用@Import + ImportBeanDefinitionRegister注册BeanDefinition**

```java
public class MyBeanImport implements ImportBeanDefinitionRegistrar{
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
        beanDefinition.setBeanClass(MytestBean.class);
        beanDefinition.setSynthetic(true);
        MutablePropertyValues mpv = beanDefinition.getPropertyValues();
        mpv.addPropertyValue("age", "25");
        registry.registerBeanDefinition("mytestBean", beanDefinition);
    }
}
在Bootstrap启动类或者一个配置类中使用：
@Import(MyBeanImport.class)
```

**6.引入xml配置文件，除了老项目升级基本上不用，没啥好说的**

不同注册方式执行的顺序不同，总体上顺序：ImportBeanDefinitionRegister > BeanDefinitionRegistryPostProcessor > @Bean = @Component

如果用注册FactoryBean<MyBean>的方式来提供bean的话，Spring并不能管理好依赖的先后顺序，所以如果依赖MyBean的bean先被spring装配，就会出现找不到bean的情况，这时候可以让bean同时依赖对应的FactoryBean以确保FactoryBean被先注册。  

### @SpringBootApplication

该注解等效于`@Configuration`，`@EnableAutoConfiguration`和`@ComponentScan`三个注解的和注解。

- `@EnableAutoConfiguration`：启用[Spring Boot的自动配置机制](https://www.springcloud.cc/spring-boot.html#using-boot-auto-configuration)。使其他jar包的Bean可用通过spring.factories引入，参考[这里](https://www.jianshu.com/p/464d04c36fb1)。
- `@ComponentScan`：对应用程序所在的软件包启用`@Component`扫描（请参阅[最佳实践](https://www.springcloud.cc/spring-boot.html#using-boot-structuring-your-code)）
- `@Configuration`：允许在上下文中注册额外的beans或导入其他配置类

### SpringBoot实现自动配置

#### [场景](https://www.jianshu.com/p/464d04c36fb1)

在微服务中，引入了其他jar包。但是其他jar包里虽然进行了Bean注入，但是无法注入到微服务里面。我们需要其他jar包里面的Bean能够注入进来。

#### 解决方案

```txt
1.使用@EnableAutoConfiguration注解标注Application.class或者其他有@Confuguration的类。
2.在其他jar包里META-INF目录下新增spring.factories，key为自定配置类EnableAutoConfiguration
3.在其他jar包里定义好配置类，该类加了@Configuration注解
4.在该配置类内注入Bean即可。可以用@Import、@Bean、@EnableConfigurationProperties、ImportBeanDefinitionRegistrar.class等方式注入
```

具体实施如下：

1.XXXApplication.class：@EnableAutoConfiguration

`@SpringBootApplication`注解自带了`@EnableAutoConfiguration`注解。

```java
@SpringBootApplication
@MapperScan(basePackages = "com.rainbowdemo.service.basic.systemmana.mapper")
public class SystemmanaApplication {
    public static void main(String[] args) {
        SpringApplication.run(SystemmanaApplication.class, args);
    }
}
```

2.spring.factories：EnableAutoConfiguration

```java
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.wind.cloud.starter.mybatis.autoconfigure.WindDatasourceAutoConfiguration
```

3.XXXAutoConfiguration：@Configuration

```java
@Configuration
@EnableConfigurationProperties(WindDatasourceProperties.class)
@EnableTransactionManagement
@Import({ WindDatasourceInitializerInvoker.class, WindDatasourceAutoConfiguration.Registrar.class })
public class WindDatasourceAutoConfiguration {
    public static class Registrar implements ImportBeanDefinitionRegistrar {

        private static final String BEAN_NAME = WindDatasourceInitializerPostProcessor.class.getSimpleName();
        @Getter
        private static BeanDefinitionRegistry registry;

        @Override
        public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
                                            BeanDefinitionRegistry registry) {
            if (!registry.containsBeanDefinition(BEAN_NAME)) {
                WindDatasourceAutoConfiguration.Registrar.registry = registry;

                GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
                beanDefinition.setBeanClass(WindDatasourceInitializerPostProcessor.class);
                beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
                // We don't need this one to be post processed otherwise it can cause a
                // cascade of bean instantiation that we would rather avoid.
                beanDefinition.setSynthetic(true);
                registry.registerBeanDefinition(BEAN_NAME, beanDefinition);
            }
        }

    }

    @Bean
    public InitTransactionalValue initTransactionalValue() {
        return new InitTransactionalValue();
    }
}
```

#### 原理分析

核心注解是[`@EnableAutoConfiguration `](https://www.jianshu.com/p/464d04c36fb1)。

> @EnableAutoConfiguration 作用
> 从classpath中搜索所有META-INF/spring.factories配置文件然后，将其中org.springframework.boot.autoconfigure.EnableAutoConfiguration key对应的配置项加载到spring容器
> 只有spring.boot.enableautoconfiguration为true（默认为true）的时候，才启用自动配置
> @EnableAutoConfiguration还可以进行排除，排除方式有2中，一是根据class来排除（exclude），二是根据class name（excludeName）来排除
> 其内部实现的关键点有
> 1）ImportSelector 该接口的方法的返回值都会被纳入到spring容器管理中
> 2）SpringFactoriesLoader 该类可以从classpath中搜索所有META-INF/spring.factories配置文件，并读取配置



### 其他常用

```java
@RestController // 组合注解：@Controller + @ResponseBody

@RequestMapping("/hello")

@PostMapping("/hi")	// 等价于@RequestMapping(method = {RequestMethod.POST})


```