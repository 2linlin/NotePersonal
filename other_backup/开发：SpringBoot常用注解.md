

Spring是个大容器，那么其实它的注解广义上可以分为两类：注册Bean的和使用Bean的。

### @Bean

1.@Bean专门用在方法上。

2.用于注入第三方。自定义的类有@Component、@Controller、@Service、@Repository等。而如果想注入第三方，那么只有用@Bean和@Import。

3.用于动态注入。放在方法上提供了动态注入的能力。

实际上，@Bean注解和xml配置中的bean标签的作用是一样的

### @[Import](https://www.cnblogs.com/yichunguo/p/12122598.html)

@Import主要用来导入第三方包的Bean，或者没配到扫描路径时手动导入。

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

### 其他常用

```java
@RestController // 组合注解：@Controller + @ResponseBody

@RequestMapping("/hello")

@PostMapping("/hi")	// 等价于@RequestMapping(method = {RequestMethod.POST})


```