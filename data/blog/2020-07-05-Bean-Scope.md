---
title: Spring-Bean Scope
summary: Singleton, Prototype, Web & Custom
date: 2020-07-03
tags: ['Java', 'Design Pattern', 'Spring']
---

When you create a bean definition, you create a recipe for creating actual instances of the class defined by that bean definition. The idea that a bean definition is a recipe is important, because it means that, as with a class, you can create many object instances from a single recipe.

You can control not only the various dependencies and configuration values that are to be plugged into an object that is created from a particular bean definition but also control the scope of the objects created from a particular bean definition. This approach is powerful and flexible, because you can choose the scope of the objects you create through configuration instead of having to bake in the scope of an object at the Java class level. Beans can be defined to be deployed in one of a number of scopes. The Spring Framework supports six scopes, four of which are available only if you use a web-aware `ApplicationContext`. You can also create [a custom scope.](https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/core.html#beans-factory-scopes-custom)

| Scope                                                        | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [singleton](https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/core.html#beans-factory-scopes-singleton) | (Default) Scopes a single bean definition to a single object instance for each Spring IoC container. |
| [prototype](https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/core.html#beans-factory-scopes-prototype) | Scopes a single bean definition to any number of object instances. |
| [request](https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/core.html#beans-factory-scopes-request) | Scopes a single bean definition to the lifecycle of a single HTTP request. That is, each HTTP request has its own instance of a bean created off the back of a single bean definition. Only valid in the context of a web-aware Spring `ApplicationContext`. |
| [session](https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/core.html#beans-factory-scopes-session) | Scopes a single bean definition to the lifecycle of an HTTP `Session`. Only valid in the context of a web-aware Spring `ApplicationContext`. |
| [application](https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/core.html#beans-factory-scopes-application) | Scopes a single bean definition to the lifecycle of a `ServletContext`. Only valid in the context of a web-aware Spring `ApplicationContext`. |
| [websocket](https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/web.html#websocket-stomp-websocket-scope) | Scopes a single bean definition to the lifecycle of a `WebSocket`. Only valid in the context of a web-aware Spring `ApplicationContext`. |

本文章介绍Spring的bean作用域的两种：Singleton单例模式和Prototype多例模式

# Singleton单例模式

一张图来看一下什么是单例模式。![](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856590860_3463.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

当Bean的作用域为singleton时，Spring容器就只会存在一个共享的Bean实例，并且所有对Bean的请求，只要id相同就会返回同一个Bean实例。singleton作用域对于无会话的Bean是最合适的选择。

首先不要忘记引入maven依赖：`spring-core`, `spring-context`和单元测试`junit`。

准备两个Bean：Bean1和Bean2，其中Bean1依赖于Bean2。

```java
public class Bean1 {
    private Bean2 bean2;

    public Bean2 getBean2() {
        return bean2;
    }

    public void setBean2(Bean2 bean2) {
        this.bean2 = bean2;
    }

    @Override
    public String toString() {
        return "Bean1{" +
                "bean2=" + bean2 +
                '}';
    }
}
```

```java
public class Bean2 {}
```

用上一篇文章介绍过的setter注入模式来注入bean2：在spring.xml中插入如下配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean class="Bean2" id="bean2" scope="singleton"/>

    <bean class="Bean1" id="bean1">
        <property name="bean2" ref="bean2"/>
    </bean>
</beans>
```

可以看到，在Bean2的配置中，加入了`scope="singleton"`这一配置。单例模式对于Spring是缺省的。

写测试代码，来验证一下我们的理论。

```java
@Test
public void test(){
  ApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");
  Bean2 bean2_1 = context.getBean("bean2",Bean2.class);
  System.out.println("bean2_1 = " + bean2_1);
  Bean2 bean2_2 = context.getBean("bean2", Bean2.class);
  System.out.println("bean2_2 = " + bean2_2);
  Bean1 bean1 = context.getBean("bean1",Bean1.class);
  System.out.println("bean1 = " + bean1);
}
```

这里定义了两个Bean2和一个Bean1，其中Bean1中含有一个Bean2。按照我们之前的文章中的代码测试，三个Bean2应该是相同的。

```
bean2_1 = Bean2@87f383f
bean2_2 = Bean2@87f383f
bean1 = Bean1{bean2=Bean2@87f383f}
```

## 不同的context上下文

迄今为止，我们都是在同一个`ApplicationContext`上下文环境中获取Bean，那么如果运用单例模式，在多个上下文环境中尝试获取同一个Bean会怎样呢？

```java
@Test
    public void test(){
        ApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");
        Bean2 bean2_1 = context.getBean("bean2",Bean2.class);
        System.out.println("bean2_1 = " + bean2_1);
        Bean2 bean2_2 = context.getBean("bean2", Bean2.class);
        System.out.println("bean2_2 = " + bean2_2);
        Bean1 bean1 = context.getBean("bean1",Bean1.class);
        System.out.println("bean1 = " + bean1);
      
        System.out.println("------");
      
        ApplicationContext context1 = new ClassPathXmlApplicationContext("spring.xml");
        Bean2 bean2_3 = context1.getBean("bean2",Bean2.class);
        System.out.println("bean2_3 = " + bean2_3);
        Bean2 bean2_4 = context1.getBean("bean2", Bean2.class);
        System.out.println("bean2_4 = " + bean2_4);
        Bean1 bean1_2 = context1.getBean("bean1",Bean1.class);
        System.out.println("bean1_2 = " + bean1_2);
    }
```

可以看到，我新建了一个`ApplicationContext`，让我们看一下输出结果：

```
bean2_1 = Bean2@87f383f
bean2_2 = Bean2@87f383f
bean1 = Bean1{bean2=Bean2@87f383f}
------
bean2_3 = Bean2@4671e53b
bean2_4 = Bean2@4671e53b
bean1_2 = Bean1{bean2=Bean2@4671e53b}
```

同一个上下文环境中的输出是一样的，不同的上下文环境获取的Bean就不一样。

The scope of the Spring singleton is best described as being per-container and per-bean. This means that, if you define one bean for a particular class in a single Spring container, the Spring container creates one and only one instance of the class defined by that bean definition. 

# Prototype 多例模式

The non-singleton prototype scope of bean deployment results in the creation of a new bean instance every time a request for that specific bean is made. That is, the bean is injected into another bean or you request it through a `getBean()` method call on the container. As a rule, you should use the prototype scope for all stateful beans and the singleton scope for stateless beans.

![](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856591765_5306.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

(A data access object (DAO) is not typically configured as a prototype, because a typical DAO does not hold any conversational state. It was easier for us to reuse the core of the singleton diagram.)

每次通过容器的getBean方法获取prototype定义的Bean时，都将产生一个新的Bean实例。
对于singleton作用域的 Bean，每次请求该Bean都将获得相同的实例。容器负责跟踪Bean实例的状态，负责维护Bean实例的生命周期行为；如果一个Bean被设置成 prototype作用域，程序每次请求该id的Bean，Spring都会新建一个Bean实例，然后返回给程序。在这种情况下，Spring容器仅仅 使用new 关键字创建Bean实例，一旦创建成功，容器不在跟踪实例，也不会维护Bean实例的状态。
如果不指定Bean的作用域，Spring默认使用singleton作用域。Java在创建Java实例时，需要进行内存申请；销毁实例时，需要完成垃圾回收，这些工作都会导致系统开销的增加。因此，prototype作用域Bean的创建、销毁代价比较大。而singleton作用域的Bean实例一旦创建成功，可以重复使用。因此，除非必要，否则尽量避免将Bean被设置成prototype作用域。

要想试验我们的理论，只需要设置Bean2的`scope = prototype`就可以了。看一下测试结果：

```
bean2_1 = Bean2@87f383f
bean2_2 = Bean2@4eb7f003
bean1 = Bean1{bean2=Bean2@eafc191}
```

可以看到，3次调用Bean2的结果都不一样。每次向Spring上下文请求实例的时候，我们获得的都是一个新的实例。

# 单例模式Bean依赖于多例模式Bean

在我们的例子中，Bean1中有类型为Bean2的属性，也就是Bean1依赖于Bean2。  

如果Bean1是singleton，Bean2是prototype，那么每次请求Bean1时所包含的Bean2还相同吗？也就是如下这种情况：

```java
<bean class="Bean2" id="bean2" scope="prototype"/>

<bean class="Bean1" id="bean1" scope="singleton">
  <property name="bean2" ref="bean2"/>
</bean>
```

写一下测试代码，测试下多次请求Bean1的情况。

```java
@Test
public void test(){
  ApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");
  Bean1 bean1_1 = context.getBean("bean1",Bean1.class);
  System.out.println("bean1_1 = " + bean1_1);
  Bean1 bean1_2 = context.getBean("bean1",Bean1.class);
  System.out.println("bean1_2 = " + bean1_2);
  System.out.println("(bean1_1 == bean1_2) = " + (bean1_1 == bean1_2));

}
```

```
bean1_1 = Bean1{bean2=Bean2@87f383f}
bean1_2 = Bean1{bean2=Bean2@87f383f}
(bean1_1 == bean1_2) = true
```

可以看到，两次Bean2是同一个实例，两个Bean1也是相同的。Bean1是单例模式，因此当需要多次实例化Bean1时，Spring也只实例化唯一的Bean2来确保每个Bean1是相同的。

When you use singleton-scoped beans with dependencies on prototype beans, be aware that dependencies are resolved at instantiation time. Thus, if you dependency-inject a prototype-scoped bean into a singleton-scoped bean, a new prototype bean is instantiated and then dependency-injected into the singleton bean. The prototype instance is the sole instance that is ever supplied to the singleton-scoped bean.

如果我想让每次调用Bean1的*某个方法*拿到的Bean2都不同，该怎么办？

However, suppose you want the singleton-scoped bean to acquire a new instance of the prototype-scoped bean repeatedly at runtime. You cannot dependency-inject a prototype-scoped bean into your singleton bean, because that injection occurs only once, when the Spring container instantiates the singleton bean and resolves and injects its dependencies. If you need a new instance of a prototype bean at runtime more than once, see [Method Injection](https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/core.html#beans-factory-method-injection)

## 方法注入之查找注入

A problem arises when the bean lifecycles are different. Suppose singleton bean A needs to use non-singleton (prototype) bean B, perhaps on each method invocation on A. The container creates the singleton bean A only once, and thus only gets one opportunity to set the properties. The container cannot provide bean A with a new instance of bean B every time one is needed.

Lookup method injection is the ability of the container to override methods on container-managed beans and return the lookup result for another named bean in the container. The lookup typically involves a prototype bean, as in the scenario described in [the preceding section](https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/core.html#beans-factory-method-injection). 

修改我们的Bean1代码，去掉对Bean2的依赖，并新加两个方法：

```java
public abstract class Bean1 {

    public void printBean2(){
        System.out.println("Bean2 = " + createBean2());
    }

    protected abstract Bean2 createBean2();
}
```

在spring.xml中修改：

```xml
<bean class="Bean2" id="bean2" scope="prototype"/>

<bean class="Bean1" id="bean1" scope="singleton">
  <lookup-method name="createBean2" bean="bean2" />
</bean>
```

测试一下，是否单例的Bean1，每次产生的多例Bean2，是不同的：

```java
@Test
public void test(){
  ApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");
  Bean1 bean1 = context.getBean("bean1",Bean1.class);
  bean1.printBean2();
  bean1.printBean2();
  Bean1 bean1_1 = context.getBean("bean1",Bean1.class);
  bean1_1.printBean2();
  bean1_1.printBean2();
  System.out.println("(bean1 == bean1_1) = " + (bean1 == bean1_1));
}
```

```
Bean2 = Bean2@4148db48
Bean2 = Bean2@282003e1
Bean2 = Bean2@7fad8c79
Bean2 = Bean2@71a794e5
(bean1 == bean1_1) = true
```

可以看到，每次产生的Bean2都是不同的，而两个Bean1是相同的，遵守了singleton模式。

# Web应用作用域

The `request`, `session`, `application`, and `websocket` scopes are available only if you use a web-aware Spring `ApplicationContext` implementation (such as `XmlWebApplicationContext`). If you use these scopes with regular Spring IoC containers, such as the `ClassPathXmlApplicationContext`, an `IllegalStateException` that complains about an unknown bean scope is thrown.

为了学习这几个Web作用域，我们需要做一些准备工作。

1. 配置`webapp`上下文环境

   - 如果你使用的是`Spring Web MVC`，则只需要准备Spring的`DispatcherServlet`

     首先在pom.xml引入需要的依赖:

     ```xml
     <dependencies>
       <dependency>
         <groupId>org.springframework</groupId>
         <artifactId>spring-context</artifactId>
         <version>5.2.7.RELEASE</version>
       </dependency>
       <dependency>
         <groupId>org.springframework</groupId>
         <artifactId>spring-core</artifactId>
         <version>5.2.7.RELEASE</version>
       </dependency>
       <dependency>
         <groupId>org.springframework</groupId>
         <artifactId>spring-web</artifactId>
         <version>5.2.7.RELEASE</version>
       </dependency>
       <dependency>
         <groupId>org.springframework</groupId>
         <artifactId>spring-webmvc</artifactId>
         <version>5.2.7.RELEASE</version>
       </dependency>
     </dependencies>
     ```

     然后在web.xml中设置`DispatcherServlet`:

     ```xml
     <web-app xmlns="http://java.sun.com/xml/ns/javaee" version="2.5">
         <servlet>
             <servlet-name>SpringServlet</servlet-name>
             <servlet-class>
               org.springframework.web.servlet.DispatcherServlet
           	</servlet-class>
             <init-param>
                 <param-name>contextConfigLocation</param-name>
                 <param-value>classpath:spring.xml</param-value>
             </init-param>
         </servlet>
         <servlet-mapping>
             <servlet-name>SpringServlet</servlet-name>
             <url-pattern>/*</url-pattern>
         </servlet-mapping>
     </web-app>
     ```

   - 如果你使用的是Servlet 2.5 web container及以上，你需要注册`org.springframework.web.context.request.RequestContextListener` `ServletRequestListener`.

     ```xml
     <web-app>
         ...
         <listener>
             <listener-class>
                 org.springframework.web.context.request.RequestContextListener
             </listener-class>
         </listener>
         ...
     </web-app>
     ```

   - 如果你使用的版本较低，使用Spring的`RequestContextFilter`

     ```xml
     <web-app>
         ...
         <filter>
             <filter-name>requestContextFilter</filter-name>
             <filter-class>org.springframework.web.filter.RequestContextFilter</filter-class>
         </filter>
         <filter-mapping>
             <filter-name>requestContextFilter</filter-name>
             <url-pattern>/*</url-pattern>
         </filter-mapping>
         ...
     </web-app>
     ```

   实际上`DispatcherServlet`, `RequestContextListener`以及`RequestContextFilter`做的都是同一件事：绑定HTTP请求对象到要为该请求服务的线程中去 。这样创建的
   bean就是 request- 和session-scoped的作用域， 提供下一步调用链。

2. 配置tomcat

   ![Screenshot 2020-07-03 at 17.19.08](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856592547_7432.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

## request

此段落引用了来自[spring bean的作用域](https://www.jianshu.com/p/c53e8edabf6f)的内容。

在一次Http请求中，容器会返回该Bean的同一实例。而对不同的Http请求则会产生新的Bean，而且该bean仅在当前Http Request内有效。,针对每一次Http请求，Spring容器根据该bean的定义创建一个全新的实例，且该实例仅在当前Http请求内有效，而其它请求无法看到当前请求中状态的变化，当当前Http请求结束，该bean实例也将会被销毁。

编写我们的类RequestController

```java
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;
@Controller
public class RequestController {
    @RequestMapping("testRequest")
    @ResponseBody
    public String test(){
        return this.toString();
    }
}
```

在spring.xml中设置Bean

```xml
<bean class="RequestController" scope="request"/>
```

运行，在浏览器打开（通常为localhost:8080/testRequest）页面，会看到输出。每次刷新浏览器，Bean都会被新建一次。

## session

在一次Http Session中，容器会返回该Bean的同一实例。而对不同的Session请求则会创建新的实例，该bean实例仅在当前Session内有效。同Http请求相同，每一次session请求创建新的实例，而不同的实例之间不共享属性，且实例仅在自己的session请求内有效，请求结束，则实例将被销毁。

```java
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;
@Controller
public class SessionController {
    @RequestMapping("testSession")
    @ResponseBody
    public String test(){
        return this.toString();
    }
}
```

```xml
<bean class="SessionController" scope="session"/>
```

在浏览器测试，发现刷新页面，Bean也不会重新创建，查看JSESSIONID，发现也不会变化。打开第二个浏览器，会被判断为一个新的HTTP Session请求，因此会变化。

## application

在整个Web App应用中，bean只被创建一次，并被作为`ServletContext`的属性存储起来。Application作用域与Singleton作用域很相像，但有不同：Application的Bean是每一个`ServletContext`中都带有仅一个，而Singleton是每一个Spring环境下的`ApplicationContext`中都带有仅一个。

```java
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;
@Controller
public class ApplicationController {
    @RequestMapping("testApplication")
    @ResponseBody
    public String test(){
        return this.toString();
    }
}
```

```xml
<bean class="ApplicationController" scope="application"/>
```

无论如何刷新或打开浏览器，Web Application都只有一个，因此Bean也保持不变。

## websocket

`WebSocket`协议支持客户端和远程主机之间的双向通信，远程主机选择与客户端通信。`WebSocket`协议为两个方向的通信提供了一个单独的`TCP`连接。这对于具有同步编辑和多用户游戏的多用户应用程序特别有用。

在这种类型的`Web`应用程序中，`HTTP`仅用于初始握手。如果服务器同意，服务器可以以`HTTP`状态101（交换协议）进行响应。如果握手成功，则`TCP`套接字保持打开状态，客户端和服务器都可以使用该套接字向彼此发送消息。

请注意，`websocket`范围内的`bean`通常是单例的，并且比任何单独的`WebSocket`会话寿命更长。

# 自定义作用域

To integrate your custom scopes into the Spring container, you need to implement the `org.springframework.beans.factory.config.Scope` interface, which is described in this section. For an idea of how to implement your own scopes, see the `Scope` implementations that are supplied with the Spring Framework itself and the [`Scope`](https://docs.spring.io/spring-framework/docs/5.2.7.RELEASE/javadoc-api/org/springframework/beans/factory/config/Scope.html) javadoc, which explains the methods you need to implement in more detail.

The `Scope` interface has four methods to get objects from the scope, remove them from the scope, and let them be destroyed.

我们可以自定义一个作用域，像`Singleton`或者`Prototype`那样，尝试定义一个“双例模式”：每个Bean只会在容器中至多生成两次。

## “双例模式”

首先引入pom依赖

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <version>5.2.2.RELEASE</version>
  </dependency>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>5.2.2.RELEASE</version>
  </dependency>
  <dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.12</version>
    <scope>test</scope>
  </dependency>
</dependencies>
```

新建我们的类MyScope.java

```java
import org.springframework.beans.factory.ObjectFactory;
import org.springframework.beans.factory.config.Scope;

import java.util.Map;
import java.util.Random;
import java.util.concurrent.ConcurrentHashMap;

public class MyScope implements Scope {
    //double creation at most
    private Map<String, Object> map1 = new ConcurrentHashMap<String, Object>();
    private Map<String, Object> map2 = new ConcurrentHashMap<String, Object>();

    public Object get(String s, ObjectFactory<?> objectFactory) {
        if (!map1.containsKey(s)){
            Object o = objectFactory.getObject();
            map1.put(s,o);
            return o;
        }
        if(!map2.containsKey(s)){
            Object o = objectFactory.getObject();
            map2.put(s,o);
            return o;
        }
        int i = new Random().nextInt(2);
        if(i == 0){
            return map1.get(s);
        }else {
            return map2.get(s);
        }
    }

    public Object remove(String s) {
        if(map1.containsKey(s)){
            Object o = map1.get(s);
            map1.remove(s);
            return o;
        }
        if(map2.containsKey(s)){
            Object o = map2.get(s);
            map2.remove(s);
            return o;
        }
        return null;
    }

    public void registerDestructionCallback(String s, Runnable runnable) {

    }

    public Object resolveContextualObject(String s) {
        return null;
    }

    public String getConversationId() {
        return null;
    }
}

```

1. 这个类实现了接口`org.springframework.beans.factory.config.Scope`，也就是说我们要自定义域必须实现这个接口。
2. 两个Map存储双例，用`ConcurrentHashMap`的原因是保证线程安全。
3. `get`方法从作用域中获取对象，这里的逻辑是，先判断对象是否存在：如果我们需要的对象在map1中没有，那么就新建一个并存放在里面，map2同理。两个if之后，map1和map2肯定都包含了我们需要的对象。定义一个随机数，值为0或1，用来随机从map1或2中返回我们需要的对象。
4. `remove`方法恰恰相反，从作用域中删除对象。方法内删除对象后，会返回该对象，但是若找不到指定对象，则会返回`null`。
5. `void registerDestructionCallback(String s, Runnable runnable)`注册销毁回调函数，销毁是指对象销毁或者是作用域内对象销毁。销毁回调的详情请参看javadocs或者Spring 作用域实现。
6. `String getConversationId()`获取作用域会话标识。每个作用域的标识都不一样。比如，`session`作用域的实现中，标识就是`session`标识（应该是指sessionId）

然后新建一个Bean.java，不需要内容。

接下来在xml配置文件中配置Bean

```xml
<bean class="MyScope" id="myScope"/>
    
<bean class="org.springframework.beans.factory.config.CustomScopeConfigurer">
  <property name="scopes">
    <map>
      <entry key="myScope" value-ref="myScope"/>
    </map>
  </property>
</bean>

<bean class="Bean" id="bean" scope="myScope"/>
```

1. 第一个bean`MyScope`指向的是我们定义的作用域的类。
2. 第二个bean，指向的类是Spring提供的`CustomScopeConfigurer`，在这个Bean中必须包含名为scopes的属性，且必须为map。在我们现在的例子中，map包含一对键值对，key是这个作用域的名字，value-ref是即我们自定义的MyScope。当然，可以包含多个键值对（例如接下来的`SimpleThreadScope`），指向不同的Scope。
3. 第三个bean是刚刚新建的Bean.java，使用的作用域是我们自定义的MyScope

接下来做测试

```java
@Test
public void test(){
  ApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");
  for (int i = 0; i < 10; i++) {
    Bean bean = context.getBean("bean",Bean.class);
    System.out.println("bean = " + bean);
  }
}
```

我们用循环生成10个Bean对象，查看这10个其实应该只是两个Bean的重复。

```
bean = Bean@3891771e
bean = Bean@78ac1102
bean = Bean@78ac1102
bean = Bean@78ac1102
bean = Bean@3891771e
bean = Bean@78ac1102
bean = Bean@78ac1102
bean = Bean@78ac1102
bean = Bean@78ac1102
bean = Bean@3891771e
```

果然，尽管输出10次，但只有两个不同的对象。

## 使用`SimpleThreadScope`

一个线程级别的bean作用域，同一个线程中同名的bean是同一个实例，不同的线程中的bean是不同的实例。

我们只需要对spring.xml稍作修改：

```xml
<bean class="MyScope" id="myScope"/>
<bean class="org.springframework.context.support.SimpleThreadScope" id="simpleThreadScope"/>
<bean class="org.springframework.beans.factory.config.CustomScopeConfigurer">
  <property name="scopes">
    <map>
      <entry key="myScope" value-ref="myScope"/>
      <entry key="simpleThreadScope" value-ref="simpleThreadScope"/>
    </map>
  </property>
</bean>
<bean class="Bean" id="bean" scope="simpleThreadScope"/>
```

这里添加了一个类型为`SimpleThreadScope`的Bean，我们用它来实现我们的需求。在`CustomScopeConfigurer`的`<property>`里增加了一个键值对，指向定义的`simpleThreadScope`Bean。然后，把上一步建的Bean的作用域改为`simpleThreadScope`。

稍稍改一下测试代码，看看是不是符合我们的需求。

```java
@Test
public void test(){
  final ApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");
  for (int i = 0; i < 10; i++) {
    Bean bean = context.getBean("bean",Bean.class);
    System.out.println("bean = " + bean);
  }
  System.out.println("-----------");
  for (int i = 0; i < 10; i++) {
    new Thread(new Runnable() {
      public void run() {
        Bean bean = context.getBean("bean", Bean.class);
        System.out.println("bean = " + bean);
      }
    }).start();
  }
}
```

横线以上的10次循环，应该输出10个相同的bean；横线以下的，应该均不同，因为是10个不同的线程。

```
bean = Bean@12c8a2c0
bean = Bean@12c8a2c0
bean = Bean@12c8a2c0
bean = Bean@12c8a2c0
bean = Bean@12c8a2c0
bean = Bean@12c8a2c0
bean = Bean@12c8a2c0
bean = Bean@12c8a2c0
bean = Bean@12c8a2c0
-----------
bean = Bean@4a4b8be7
bean = Bean@7626c5fa
bean = Bean@71f3373b
bean = Bean@2ac5e7f4
bean = Bean@7a49a9e9
bean = Bean@abe194f
bean = Bean@60e2f7e3
bean = Bean@50f02888
bean = Bean@310bce6d
bean = Bean@340fddd0
```

果然如同我们猜想的那样。