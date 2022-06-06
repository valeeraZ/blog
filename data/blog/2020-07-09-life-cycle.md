---
title: Spring-Bean's lifecycle
summary: lazy-initialized, Depends-on, initialization and destruction 
date: 2020-07-09
tags: ['Java', 'Design Pattern', 'Spring']
---

# Lazy-initialized 懒加载

按照之前的方法，创建我们的Bean和Test：

```java
public class Bean {
    public Bean() {
        System.out.println("Bean has been created");
    }
}
```

```java
@Test
public void test(){
  ApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");
  System.out.println("Context has been created");
  Bean bean = context.getBean("bean",Bean.class);
  System.out.println("bean = " + bean);
}
```

运行测试，我们会发现：

```
Bean has been created
Context has been created
bean = Bean@436e852b
```

Bean是在Context初始化时就被实例化了。但有些时候，初始化Bean的成本太高，我们并不想这样做，该怎么办呢？可以在Spring配置文件中，为模式为Singleton的Bean设置懒加载，即仅当我们需要时，才实例化Bean。

By default, `ApplicationContext` implementations eagerly create and configure all [singleton](https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/core.html#beans-factory-scopes-singleton) beans as part of the initialization process. Generally, this pre-instantiation is desirable, because errors in the configuration or surrounding environment are discovered immediately, as opposed to hours or even days later. When this behavior is not desirable, you can prevent pre-instantiation of a singleton bean by marking the bean definition as being lazy-initialized. A lazy-initialized bean tells the IoC container to create a bean instance when it is first requested, rather than at startup.

In XML, this behavior is controlled by the `lazy-init` attribute on the `` element, as the following example shows:

```xml
<bean id="lazy" class="com.something.ExpensiveToCreateBean" lazy-init="true"/>
<bean name="not.lazy" class="com.something.AnotherBean"/>
```

When the preceding configuration is consumed by an `ApplicationContext`, the `lazy` bean is not eagerly pre-instantiated when the `ApplicationContext` starts, whereas the `not.lazy` bean is eagerly pre-instantiated.

However, when a lazy-initialized bean is a dependency of a singleton bean that is not lazy-initialized, the `ApplicationContext` creates the lazy-initialized bean at startup, because it must satisfy the singleton’s dependencies. The lazy-initialized bean is injected into a singleton bean elsewhere that is not lazy-initialized.

You can also control lazy-initialization at the container level by using the `default-lazy-init` attribute on the `` element, a the following example shows:

```xml
<beans default-lazy-init="true">
    <!-- no beans will be pre-instantiated... -->
</beans>
```

# Depends-on 加载顺序

如果Bean a直接或间接依赖Bean b，那么可以肯定Spring容器将首先创建Bean b。然而，如果两个Bean定义彼此之间没有直接或间接依赖，那么Bean创建的顺序就由Spring容器内部确定。此时，无法确保Bean b始终在Bean a之前被创建。有时，虽然Bean之间没有彼此依赖，但它们却需要彼此之间有一个特定的创建顺序。比如说，你的DAO Bean实例化之前你必须要先实例化Database Bean，DAO Bean并不需要持有一个Database Bean的实例。因为DAO的使用是依赖Database启动的，如果Database Bean不启动，那么DAO即使实例化也是不可用的。这种情况DAO对Database的依赖是不直接的。

  除了在DAO上使用构造函数注入Database Bean以外，Spring没有任何依赖注入的关系能够满足上面的情况。但是DAO也许根本不需要Database的实例被注入，因为DAO是通过JDBC访问数据库的，它不需要调用Database 上的任何方法和属性。

If a bean is a dependency of another bean, that usually means that one bean is set as a property of another. Typically you accomplish this with the [`element`](https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/core.html#beans-ref-element) in XML-based configuration metadata. However, sometimes dependencies between beans are less direct. An example is when a static initializer in a class needs to be triggered, such as for database driver registration. The `depends-on` attribute can explicitly force one or more beans to be initialized before the bean using this element is initialized. The following example uses the `depends-on` attribute to express a dependency on a single bean:

```xml
<bean id="beanOne" class="ExampleBean" depends-on="manager"/>
<bean id="manager" class="ManagerBean" />
```

To express a dependency on multiple beans, supply a list of bean names as the value of the `depends-on` attribute (commas, whitespace, and semicolons are valid delimiters):

```xml
<bean id="beanOne" class="ExampleBean" depends-on="manager,accountDao">
    <property name="manager" ref="manager" />
</bean>

<bean id="manager" class="ManagerBean" />
<bean id="accountDao" class="x.y.jdbc.JdbcAccountDao" />
```

The `depends-on` attribute can specify both an initialization-time dependency and, in the case of [singleton](https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/core.html#beans-factory-scopes-singleton) beans only, a corresponding destruction-time dependency. Dependent beans that define a `depends-on` relationship with a given bean are destroyed first, prior to the given bean itself being destroyed. Thus, `depends-on` can also control shutdown order.

# 初始化及销毁

Spring 容器中的 Bean 是有生命周期的，Bean 的生命周期是指 Bean 创建----> 初始化----> 销毁 的过程，并且 Spring 允许 Bean 在初始化完成后以及销毁前执行特定的操作。 下面是常用的指定特定操作的方法：

1. 通过`<bean>` 元素的`init-method`/`destroy-method`属性指定初始化之后 /销毁之前调用的操作方法；
2. 通过实现`InitializingBean`/`DisposableBean` 接口来定制初始化之后/销毁之前的操作方法；
3. 在指定方法上加上`@PostConstruct`或`@PreDestroy`注解来制定该方法是在初始化之后还是销毁之前调用，它是使用了`InitDestroyAnnotationBeanPostProcessor`后置处理器来实现的。（本文暂时不讲解这种方法）

如果使用第一种方法，即如下：

```java
public void initMethod() {
  System.out.println("XML配置-执行init-method方法");
}

public void destroyMethod() {
  System.out.println("XML配置-执行destroy-method方法");
}
```

需要在xml配置文件中标注：

```xml
<bean class="Bean" id="bean" init-method="initMethod" destroy-method="destroyMethod"/>
```

以告知spring ioc。

如果使用第二种方法，即需要`implements InitializingBean, DisposableBean `实现这两个接口。

```java
public void destroy() throws Exception {
  System.out.println("接口-执行destroy方法");
}

public void afterPropertiesSet() throws Exception {
  System.out.println("接口-执行afterPropertiesSet方法");
}
```

> We recommend that you do not use the `InitializingBean` interface, because it unnecessarily couples the code to Spring. Alternatively, we suggest using the [`@PostConstruct`](https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/core.html#beans-postconstruct-and-predestroy-annotations) annotation （这段之后再写）or specifying a POJO initialization method. In the case of XML-based configuration metadata, you can use the `init-method` attribute to specify the name of the method that has a void no-argument signature. With Java configuration, you can use the `initMethod` attribute of `@Bean`. See [Receiving Lifecycle Callbacks](https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/core.html#beans-java-lifecycle-callbacks).

> We recommend that you do not use the `DisposableBean` callback interface, because it unnecessarily couples the code to Spring. Alternatively, we suggest using the [`@PreDestroy`](https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/core.html#beans-postconstruct-and-predestroy-annotations) annotation （这段之后再写）or specifying a generic method that is supported by bean definitions. With XML-based configuration metadata, you can use the `destroy-method` attribute on the `<bean/>`. With Java configuration, you can use the `destroyMethod` attribute of `@Bean`. 

那么我们把两个方法组合在一起，看看执行顺序是什么：

```java
public class Bean implements InitializingBean, DisposableBean {

    public void destroy() throws Exception {
        System.out.println("接口-执行destroy方法");
    }

    public void afterPropertiesSet() throws Exception {
        System.out.println("接口-执行afterPropertiesSet方法");
    }

    public void initMethod() {
        System.out.println("XML配置-执行init-method方法");
    }

    public void destroyMethod() {
        System.out.println("XML配置-执行destroy-method方法");
    }
}
```

测试类：为了测试Bean的销毁，我们需要显式地销毁上下文（见下方）。注意，这里的ApplicationContext使用的是`AbstractApplicationContext`，带有函数`close()`，作用是关闭上下文。你也可以使用`registerShutdownHook()`，registerShutdownHook是注册一个关闭事件的回调方法，当context关闭时会回调该方法。

```java
@Test
public void test(){
  AbstractApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");
  Bean bean = context.getBean("bean",Bean.class);
  System.out.println("bean = " + bean);
  context.close();
}
```

执行的结果：

接口-执行afterPropertiesSet方法
XML配置-执行init-method方法
bean = Bean@587d1d39
接口-执行destroy方法
XML配置-执行destroy-method方法

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



## Shutting Down the Spring IoC Container Gracefully in Non-Web Applications

> This section applies only to non-web applications. Spring’s web-based `ApplicationContext` implementations already have code in place to gracefully shut down the Spring IoC container when the relevant web application is shut down.

If you use Spring’s IoC container in a non-web application environment (for example, in a rich client desktop environment), register a shutdown hook with the JVM. Doing so ensures a graceful shutdown and calls the relevant destroy methods on your singleton beans so that all resources are released. You must still configure and implement these destroy callbacks correctly.

To register a shutdown hook, call the `registerShutdownHook()` method that is declared on the `ConfigurableApplicationContext` interface, as the following example shows:

```java
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public final class Boot {

    public static void main(final String[] args) throws Exception {
        ConfigurableApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");

        // add a shutdown hook for the above context...
        ctx.registerShutdownHook();

        // app runs here...

        // main method exits, hook is called prior to the app shutting down...
    }
}
```

