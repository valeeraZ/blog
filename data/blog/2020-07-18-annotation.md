---
title: Spring-Annotation
summary: Use Annotations instead of XML 
date: 2020-07-19
tags: ['Java', 'Design Pattern', 'Spring']
---

本篇介绍使用注解来进行Bean的实例化，依赖注入，设置作用域，懒加载以及生命周期的初始化及销毁。

# 基于Java配置

The `@Bean` annotation is used to indicate that a method instantiates, configures, and initializes a new object to be managed by the Spring IoC container. For those familiar with Spring’s `` XML configuration, the `@Bean` annotation plays the same role as the `` element. You can use `@Bean`-annotated methods with any Spring `@Component`. However, they are most often used with `@Configuration` beans.

Annotating a class with `@Configuration` indicates that its primary purpose is as a source of bean definitions. Furthermore, `@Configuration` classes let inter-bean dependencies be defined by calling other `@Bean` methods in the same class. The simplest possible `@Configuration` class reads as follows:

```java
@Configuration
public class AppConfig {

    @Bean
    public MyService myService() {
        return new MyServiceImpl();
    }
}
```

函数`myService`的名字，即为所设置的Bean的id。如若不想用这种方法，可以在注解上添加：`@Bean(value = "IDxxx")`来设置Bean的id。

The preceding `AppConfig` class is equivalent to the following Spring- XML:

```xml
<beans>
    <bean id="myService" class="com.acme.services.MyServiceImpl"/>
</beans>
```

但这种方法其实与xml设置没有什么不同，代码量也没有减少太多，并且在修改时工作量太大。

在测试时，我们需要使用`AnnotationConfigApplicationContext`:

```java
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
MyService myService = ctx.getBean(MyService.class);
myService.doStuff();
```

# 基于包扫描（常用）

通过包扫描，将包下所有注解类，注入到spring容器中。

```java
package com.example;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan(value = "com.example")
public class MyConfiguration {

}
```

这个配置类，会将包`com.example`下所有注解的类扫描进容器。那么怎么注解一个Bean的类呢？

```java
@Component
public class Bean1 {}
```

声明bean的注解

- `@Service` 在业务逻辑层service层使用

- `@Component` 组件，没有明确的角色

- `@Repository` 在数据访问层dao层使用

- `@Controlle` 在控制层使用

接下来我们都会使用`@ComponentScan(value = "com.example")`+`@Component`的方法，来进行注解。

## 依赖注入

### 使用@Autowired自动装配

假设我们有两个类：Bean和AnotherBean

```java
@Component
public class Bean {
    private AnotherBean anotherBean1;
    private AnotherBean anotherBean2;
    private AnotherBean anotherBean3;
}

@Component
public class AnotherBean {
}
```

我们会用三种不同的方式，注入anotherBean1，2，3三个属性。

我们可以在构造函数使用`@Autowired`:

```java
@Autowired
public Bean(AnotherBean anotherBean1) {
    this.anotherBean1 = anotherBean1;
}
```

我们可以在setter上使用：

```java
@Autowired
public void setAnotherBean2(AnotherBean anotherBean2) {
    this.anotherBean2 = anotherBean2;
}
```

我们甚至可以直接在属性上使用：（不推荐）

```java
@Autowired
private AnotherBean anotherBean3;
```

当然了，这三个anotherBean1，2，3打印出来，都是一样的，因为AnotherBean是单例模式。

### 集合类的注入

#### List注入

假设有一个List属性，需要注入，我们使用@Autowired的setter来注入。

```java
private List<String> stringList;

public List<String> getStringList() {
        return stringList;
}

@Autowired
public void setStringList(List<String> stringList) {
  this.stringList = stringList;
}
```

那么，String类型的对象，他们的值从哪来呢？

在`MyConfiguration`中，使用Java配置来注入。

```java
@Configuration
@ComponentScan(value = "com.example")
public class MyConfiguration {
    @Bean
    public List<String> stringList(){
        List<String> list = new ArrayList<String>();
        list.add("hello");
        list.add("world");
        return list;
    }
}
```

Spring会在其上下文中，寻找所有Spring类型的对象，来注入到List中，因此我们也可以使用如下方法：

```java
@Bean
public String string1(){
  return "i am";
}

@Bean
public String string2(){
  return "sylvain";
}
```

这种方法优先级被通过完全注入`List<String>`的优先级要高，因此如果同时使用这两种方法，我们只会得到`[i am, sylvain]`。

使用这种方法，List的元素顺序不能得到保证，因此使用另一个注解来指定顺序。

```java
@Bean
@Order(1)
public String string1(){
    return "i am";
}

@Bean
@Order(0)
public String string2(){
    return "sylvain";
}
```

0在1之前，因此得到的List是`[sylvain, i am]`。顺序的数字不需要是连续的，只需要是你所需要的数字顺序即可。

#### 指定id

如果有方法，需要特定注入方式，那么需要指定id来注入。这里`@Bean("stringList")`指定了id。

```java
@Bean("stringList")
public List<String> stringList(){
  List<String> list = new ArrayList<String>();
  list.add("hello");
  list.add("world");
  return list;
}
```

在Bean的setter上，使用`@Qualifier`来指定id

```java
@Autowired
@Qualifier("stringList")
public void setStringList(List<String> stringList) {
  this.stringList = stringList;
}
```

这样在上下文中就不会和其他方法混淆。

#### Map注入

注入Map属性：

```java
public Map<String, Integer> getIntegerMap() {
        return integerMap;
    }

@Autowired
public void setIntegerMap(Map<String, Integer> integerMap) {
  this.integerMap = integerMap;
}
```

和List一样，在MyConfiguration注入：

```java
@Bean
public Map<String, Integer> integerMap(){
  Map<String, Integer> map = new HashMap<String, Integer>();
  map.put("a",1);
  map.put("b",2);
  return map;
}
```

我们会获得`{a=1, b=2}`。与List一样，我们同样可以使用如下方法：

```java
@Bean
public Integer integer1(){
  return 3;
}

@Bean
public Integer integer2(){
  return 4;
}
```

这样会得到`{integer1=3, integer2=4}`。此处Map中的key就是函数名，如果不想使用函数名，像上一部分一样，在注解上加入`@Bean("id")`来指定key的名字。

### 普通类型注入

若在Bean中有一个String属性，我们可以直接使用`@Value()`来注解他的值。

```java
private String string;

public String getString() {
  return string;
}

@Value("hello")
public void setString(String string) {
  this.string = string;
}
```

那么这个string属性的值就会是hello。

### 上下文注入

我们可以在Bean中注入上下文，以此访问上下文中其他的Bean。

```java
private ApplicationContext context;

public ApplicationContext getContext() {
  return context;
}

@Autowired
public void setContext(ApplicationContext context) {
  this.context = context;
}
```

在测试中，可以试一下用这个Context获取anotherBean：

```java
ApplicationContext context = new AnnotationConfigApplicationContext(MyConfiguration.class);
Bean bean = context.getBean("bean", Bean.class);
System.out.println("bean = " + bean);

AnotherBean anotherBean = bean.getContext().getBean("anotherBean", AnotherBean.class);
System.out.println("anotherBean = " + anotherBean);
```

我们可以细心的发现，两个context是一样的，因此，在Bean中的anotherbean和bean的上下文获取的anotherbean也是一样的。

```
bean = Bean{anotherBean1=com.example.AnotherBean@7714e963, anotherBean2=com.example.AnotherBean@7714e963, anotherBean3=com.example.AnotherBean@7714e963}

anotherBean = com.example.AnotherBean@7714e963
```

## 作用域

其实很简单，只需要一行代码：

```java
@Component
@Scope("prototype")
public class TestBean {
}
```

但我想要创建一个自定义的作用域，实现双例模式，像前一篇文章[Spring-Bean Scope](https://valeeraz.github.io/2020/07/03/Bean-Scope/)提到的那样。我的自定义作用域代码在我的[GitHub](https://github.com/valeeraZ/Spring-course/blob/master/6-Scope-Custom/src/main/java/MyScope.java)上。

`MyConfiguration`的代码设置如下：

```java
@Configuration
@ComponentScan("com.example.scope")
public class MyConfiguration {

    @Bean
    public MyScope myScope(){
        return new MyScope();
    }

    @Bean
    public CustomScopeConfigurer customScopeConfigurer(){
        CustomScopeConfigurer customScopeConfigurer = new CustomScopeConfigurer();
        customScopeConfigurer.addScope("myScope",myScope());
        return customScopeConfigurer;
    }
}
```

把TestBean的作用域设置为`@Scope(myScope)`

要想测试我们的双例模式，用一个循环来做测试：

```java
@Test
public void testScope(){
  ApplicationContext context = new AnnotationConfigApplicationContext(com.example.scope.MyConfiguration.class);
  for (int i = 0; i <10 ; i++) {
    TestBean testBean = context.getBean("testBean",TestBean.class);
    System.out.println("testBean = " + testBean);
  }
}
```

```
testBean = com.example.scope.TestBean@17046283
testBean = com.example.scope.TestBean@5bd03f44
testBean = com.example.scope.TestBean@5bd03f44
testBean = com.example.scope.TestBean@17046283
testBean = com.example.scope.TestBean@5bd03f44
testBean = com.example.scope.TestBean@5bd03f44
testBean = com.example.scope.TestBean@17046283
testBean = com.example.scope.TestBean@17046283
testBean = com.example.scope.TestBean@17046283
testBean = com.example.scope.TestBean@5bd03f44
```

## 方法注入

复习下方法注入的使用情况是：

> 一个单例的Bean A和一个非单例的bean B, A 依赖 B，每次容器只会初始化一次 A，B却每次都需要重新创建，当 B成为A的属性时，A内的B就无法每次重新创建，这样要么放弃控制反转，要么得想新办法解决这个问题。
>  所以，Spring 通过方法注入，来实现动态改变A内的B，注入利用了容器的覆盖受容器管理的bean方法的能力，从而返回指定名字的bean实例。

这里介绍查找注入的注解写法。这里TestBean是单例模式，依赖于多例模式的AnotherBean。但在Bean类内并不用依赖于属性，我们用`@Lookup`注解。

```java
@Component
@Scope("singleton")
public abstract class TestBean {

    @Lookup
    public abstract AnotherBean anotherBean();

    public void printAnotherBean(){
        System.out.println("anotherBean = " + anotherBean());
    }
}

@Component
@Scope("prototype")
public class AnotherBean {
}
```

测试代码，用一个循环来测试每个AnotherBean应该都是不同的：

```java
/**
* Test of Lookup Method Injection
*/
@Test
public void testLookup(){
  ApplicationContext context = new AnnotationConfigApplicationContext(com.example.scope.MyConfiguration.class);
  TestBean testBean = context.getBean("testBean",TestBean.class);
  for (int i = 0; i <10 ; i++) {
    testBean.printAnotherBean();
  }
}
```

## 懒加载

其实就一行：`@Lazy`。如果想要实现所有Bean默认懒加载，那么就在`Configuration`类上加这个注解。

## 实例化和销毁

> Multiple lifecycle mechanisms configured for the same bean, with different initialization methods, are called as follows:
>
> 1. Methods annotated with `@PostConstruct`
> 2. `afterPropertiesSet()` as defined by the `InitializingBean` callback interface
> 3. A custom configured `init()` method
>
> If multiple lifecycle mechanisms are configured for a bean and each mechanism is configured with a different method name, then each configured method is executed in the order listed after this note. However, if the same method name is configured — for example, `init()` for an initialization method — for more than one of these lifecycle mechanisms, that method is executed once.
>
> Destroy methods are called in the same order:
>
> 1. Methods annotated with `@PreDestroy`
> 2. `destroy()` as defined by the `DisposableBean` callback interface
> 3. A custom configured `destroy()` method

在[之前的文章](https://valeeraz.github.io/2020/07/09/life-cycle/)有提到过三种Bean的实例化和销毁的方法，这次我们可以使用`@PostConstruct`和`@PreDestroy`方法。对于自定义的`init()`和`destroy()` ，我们不能使用`@Component`注解，因为没有提供这个选项，只能在基于Java的`@Bean`中使用，可以查看一下这两种注解的源代码。

把三种方法结合到一起，如下：

```java
//@Component注意这里注释
public class TestBean implements InitializingBean, DisposableBean {
    public TestBean(){
        System.out.println("TestBean init");
    }

    public void destroy() throws Exception {
        System.out.println("接口-执行destroy方法");
    }

    public void afterPropertiesSet() throws Exception {
        System.out.println("接口-执行afterPropertiesSet方法");
    }

    public void initMethod() {
        System.out.println("自定义-执行init-method方法");
    }

    public void destroyMethod() {
        System.out.println("自定义-执行destroy-method方法");
    }

    @PostConstruct
    public void initMethodAnnotation() {
        System.out.println("注解-执行PostConstruct");
    }

    @PreDestroy
    public void destroyMethodAnnotation() {
        System.out.println("注解-执行PreDestroy");
    }
}

```

既然这个Bean目前没有被`@Component`注解，那么我们要使用`@Bean`+`@Configuration`来做Java配置。

```java
@Bean(initMethod = "initMethod",destroyMethod = "destroyMethod")
public TestBean testBean(){
    return new TestBean();
}
```

做一个测试，来看一下三种方法的执行顺序是不是和上面英文文档说的相同：

```java
@Test
public void testLifecycle(){
  AbstractApplicationContext context = new AnnotationConfigApplicationContext(com.example.lifecycle.MyConfiguration.class);
  System.out.println("context init");
  context.close();
}
```

注意这里要用`AbstractApplicationContext`，它可以调用`close()`函数，销毁Bean。

调用的顺序如下：

```
TestBean init
注解-执行PostConstruct
接口-执行afterPropertiesSet方法
自定义-执行init-method方法
context init
注解-执行PreDestroy
接口-执行destroy方法
自定义-执行destroy-method方法
```



