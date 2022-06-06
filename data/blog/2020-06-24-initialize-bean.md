---
title: Spring-Initialize bean
summary: 3 ways to initialize a bean in Spring
date: 2020-06-24
tags: ['Java', 'Design Pattern', 'Spring']
---

A bean definition is essentially a recipe for creating one or more objects. The container looks at the recipe for a named bean when asked and uses the configuration metadata encapsulated by that bean definition to create (or acquire) an actual object.

本文章介绍三种实例化Bean的方式

- 构造函数方式
- 静态工厂方式
- 实例方法方式

以及对于Bean的别名的介绍

首先新建maven项目，引入依赖

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



## 通过构造函数方式实例化

这也是最常用的一种。  

新建`Bean1.java`，在`spring.xml`中加入这个Bean，并测试

```java
//Bean1.java
public class Bean1 {
    public Bean1(){
        System.out.println("Bean1.Bean1");
    }
}
```

将我们的bean交由Spring管理：在ressources文件夹下，新建配置文件spring.xml（这一步可以在Intellij IDEA中右键新建XML Configuration file - Spring config）并加入我们的bean

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean class="Bean1" id="bean1"/>
</beans>
```

```java
import org.junit.Test;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class test {
    @Test
    public void test(){
        ApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");
        //构造方式生成Bean
        Bean1 bean1 = context.getBean("bean1",Bean1.class);
        System.out.println("bean1 = " + bean1);
    }

}
```

最终，我们可以得到测试结果如下

```
Bean1.Bean1
bean1 = Bean1@307f6b8c
```

## 通过静态工厂方式生成Bean

实际上也就是设计模式中的工厂设计模式。  

新建`Bean2.java`和`Bean2Factory.java`，插入如下代码：  

```java
//Bean2.java
public class Bean2 {
    public Bean2(){
        System.out.println("Bean2.Bean2");
    }
}
```

```java
//Bean2Factory.java
public class Bean2Factory {
    public static Bean2 getBean2(){
        return new Bean2();
    }
}
```

在`spring.xml`中，新加入如下Bean的配置：  

```xml
<bean class="Bean2Factory" factory-method="getBean2" id="bean2" />
```

可以看到，这里使用了属性`factory-method`，引入的bean不是bean本身，而是一个带有`getBean2`方法的工厂类，用于生成Bean2。

测试这个Bean2：在test中加入

```java
 //通过静态工厂方式生成Bean
Bean2 bean2 = context.getBean("bean2",Bean2.class);
System.out.println("bean2 = " + bean2);
```

## 实例方法生成Bean

其实仅仅是实例的工厂方法，不是静态的了。

```java
//Bean3.java
public class Bean3 {
    public Bean3(){
        System.out.println("Bean3.Bean3");
    }
}
```

```java
//Bean3Factory.java
public class Bean3Factory {
    public Bean3 getBean3(){
        return new Bean3();
    }
}
```

注意此处在`spring.xml`中，要加入两行：

```xml
<bean class="Bean3Factory" id="bean3Factory"/>
<bean class="Bean3" factory-bean="bean3Factory" factory-method="getBean3" id="bean3"/>
```

第一行是为了引入实例化工厂方法而加入的Factory的bean，第二行是用第一行中的factory bean来生成我们需要的bean3。

测试：

```java
//通过实例化方式生成Bean
Bean3 bean3 = context.getBean("bean3",Bean3.class);
System.out.println("bean3 = " + bean3);
```

最终我们将得到如下的测试结果：

```
Bean1.Bean1
Bean2.Bean2
Bean3.Bean3
bean1 = Bean1@307f6b8c
bean2 = Bean2@7a187f14
bean3 = Bean3@6f195bc3
```

# Bean的别名

每一个Bean都可以注册一个或多个标识符，但这些标识符必须在容器管理的范围内保持唯一。通常我们一个Bean只会注册一个标识符，其他的我们用别名来表示。

在XML配置文件中，你可以使用`id`或者`name`属性来区分Bean。`id`属性中你有且只能有一个id，以字母数字的方式组成(如：myBean, fooService等)，但如果你想要起多个别名，你可以在`name`属性中使用逗号(,)，分号(;)或者空格来间隔开。

```xml
<bean class="Bean1" id="bean1" name="bean1_1,bean1_2"/>
```



在对bean进行定义时，除了使用id属性来指定名称之外，为了提供多个名称，需要通过name属性来加以指定。而所有的这些名称都指向同一个bean，在某些情况下提供别名非常有用，比如为了让应用的每一个组件能更容易的对公共组件进行引用。

然而，在定义bean时就指定所有的别名并不是总是恰当的。有时我们期望 能在当前位置为那些在别处定义的bean引入别名。在XML配置文件中，可用 `<alias/>` 元素来完成bean别名的定义。如：

```xml
<alias name="bean1" alias="bean1_3"/>
```

但，无论如何取别名，bean在内存中都指向同一个对象。