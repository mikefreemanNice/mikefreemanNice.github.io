---
title: 如何解决spring中抽象类无法注入
date: 2018-09-30 17:14:41
tags: [Springboot]
comments: false
categories: [Springboot]
description: 提供两种解决方式SpringBeanLoader & ApplicationContextHolder
---

# 问题引入
首先明确一个问题：抽象类不能生成实例对象，spring无法注入。

原因：spring的原理是启动服务器时读取配置文件，取得类名后利用反射机制在spring上下文中生成一个单例的对象，由spring注入属性并维护此对象的状态，抽象类在反射生成对象时就已经失败了，后面的不会进行

# 如何解决
## 方案1
限于springboot方式启动，
我们编写一个SpringBeanLoader的类，在应用启动时加载org.springframework.context.ApplicationContext，对外通过ApplicationContext提供获取bean的方法，代码如下：

应用启动时设置applicationContext

```
public class Application {

    public static void main(String[] args) {

        ApplicationContext applicationContext = SpringApplication.run(Application.class, args);

        SpringBeanLoader.setApplicationContext(applicationContext);
    }
}
```

SpringBeanLoader结构

```
public class SpringBeanLoader {

    private static ApplicationContext applicationContext;

    /**
     * 获取SpringApplicationContext
     *
     * @return ApplicationContext
     */

    private static ApplicationContext getApplicationContext() {
        return applicationContext;
    }

    /**
     * 设置SpringApplicationContext
     *
     * @param applicationContext
     */
    public static void setApplicationContext(ApplicationContext applicationContext) {
        SpringBeanLoader.applicationContext = applicationContext;
    }

    /**
     * 获取Spring中注册的Bean
     *
     * @param beanClass
     * @param beanId
     * @return
     */
    public static <T> T getSpringBean(String beanId, Class<T> beanClass) {
        return getApplicationContext().getBean(beanId, beanClass);
    }

    /**
     * 获取Spring中注册的Bean
     *
     * @param beanClass
     * @return
     */
    public static <T> T getSpringBean(Class<T> beanClass) {
        return getApplicationContext().getBean(beanClass);
    }
}
```

下面测试一下
定义一个HelloService和HelloServiceImpl

```
public interface HelloService {
    String echo(String str);
}
```

```
@Component
public class HelloServiceImpl implements HelloService {
    @Override
    public String echo(String str) {
        return str;
    }
}
```

抽象类

```
public abstract class AbstractService {

    public String testSpringIoc() {

        HelloService helloService = SpringBeanLoader.getSpringBean(HelloService.class);
        return helloService.echo("test");
    }
}
```

调用类

```
public class TestService extends AbstractService{

    public String test(){
        return testSpringIoc();
    }

}
```

编写测试类

```
@RunWith(SpringJUnit4ClassRunner.class)
public class AbstractServiceTest {

    @Before
    public void init() {
        ApplicationContext applicationContext = new SpringApplicationBuilder(Application.class).web(true).run();
        SpringBeanLoader.setApplicationContext(applicationContext);
    }

    @Test
    public void test_Ioc() {
        TestService testService = new TestService();
        System.out.println(testService.testSpringIoc());
    }
}
```

输出结果：
```
test
```

## 方案2
通过构造函数将org.springframework.context.ApplicationContext注入到抽象类中，编辑一个ApplicationContext的持有类：ApplicationContextHolder

```
@Component
public class ApplicationContextHolder implements ApplicationContextAware {

    private ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    /**
     * 获取Spring中注册的Bean
     *
     * @param beanClass
     * @param beanId
     * @return
     */
    public <T> T getSpringBean(String beanId, Class<T> beanClass) {
        return applicationContext.getBean(beanId, beanClass);
    }

    /**
     * 获取Spring中注册的Bean
     *
     * @param beanClass
     * @return
     */
    public <T> T getSpringBean(Class<T> beanClass) {
        return applicationContext.getBean(beanClass);
    }
}
```

测试：

抽象类：

```
public abstract class AbstractService2 {

    private final ApplicationContextHolder applicationContextHolder;

    public AbstractService2(ApplicationContextHolder applicationContextHolder) {
        this.applicationContextHolder = applicationContextHolder;
    }

    public String testSpringIoc() {

        HelloService helloService = applicationContextHolder.getSpringBean(HelloService.class);
        return helloService.echo("test");
    }
}
```

调用类：

```
public class TestService2 extends AbstractService2{

    public TestService2(ApplicationContextHolder applicationContextHolder) {
        super(applicationContextHolder);
    }

    public String test(){
        return testSpringIoc();
    }
}
```

测试类：

```
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest(classes = Application.class)
public class TestService2Test {

    @Resource
    private ApplicationContextHolder applicationContextHolder;

    @Test
    public void test_ioc(){
        TestService2 testService2 = new TestService2(applicationContextHolder);
        System.out.println(testService2.test());
    }

}
```
输出:
```
test
```

# 总结
- 方案1仅适用于引入springboot的项目，方式比较简单。
- 方案2使用与springboot项目或通过其他容器加载的项目，方式相比方案1来说复杂一下，需要提供调用方的构造器。
- 笔者推荐在springboot项目下选择方案1.
