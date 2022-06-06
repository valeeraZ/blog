---
title: Spring IoC
summary: Spring IoC container
date: 2020-06-04
tags: ['Java', 'Design Pattern', 'Spring']
---

IoC(Inversion of Control)：IoC就是应用本身不依赖对象的创建和维护而是交给外部容器(这里为spring),这要就把应用和对象之间解耦,控制权交给了外部容器。即Don’t call me,I’ll call you！所以IoC也称DI(依赖注入)对象的创建和维护依赖于外部容器.

我们可能会经常听到另一个词：DI,因为IOC确实不够开门见山，因此业界曾进行了广泛的讨论，最终软件界的泰斗级人物MartinFowIer提出了DI（依赖注入：Dependency Injection)的概念用以代替ioc,即让调用类对某一接口实现类的依赖关系由第三方（容器或协作类）注入，以移除调用类对某一接口实现类的依赖。“依赖注入”这个名词显然比“控制反转”直接明了、易于理解。

Java程序员都知道：java程序中的每个业务逻辑至少需要两个或以上的对象来协作完成，通常，每个对象在使用他的合作对象时，自己均要使用像new object（） 这样的语法来完成合作对象的申请工作。你会发现：对象间的耦合度高了。而IOC的思想是：Spring容器来实现这些相互依赖对象的创建、协调工作。对象只需要关系业务逻辑本身就可以了。从这方面来说，对象如何得到他的协作对象的责任被反转了（IOC、DI）。

IoC的一个重点是在系统运行中，动态的向某个对象提供它所需要的其他对象。这一点是通过DI（Dependency Injection，依赖注入）来实现的。比如对象A需要操作数据库，以前我们总是要在A中自己编写代码来获得一个Connection对象，有了 spring我们就只需要告诉spring，A中需要一个Connection，至于这个Connection怎么构造，何时构造，A不需要知道。在系统运行时，spring会在适当的时候制造一个Connection，然后像打针一样，注射到A当中，这样就完成了对各个对象之间关系的控制。A需要依赖 Connection才能正常运行，而这个Connection是由spring注入到A中的，依赖注入的名字就这么来的。那么DI是如何实现的呢？ Java 1.3之后一个重要特征是反射（reflection），它允许程序在运行的时候动态的生成对象、执行对象的方法、改变对象的属性，spring就是通过反射来实现注入的。

IOC和DI描述的是一件事情，只是从不同的角度来描述：

- IOC控制反转：说的是创建对象实例的控制权从代码控制剥离到IOC容器控制，实际上就是我们现在说的第三方，侧重于原理。
- DI依赖注入：说的是创建对象实例时，为这个对象注入属性值或其它对象实例，侧重于实现。

几种注入方式：

1. 接口注入
2. `getter`，`setter`方式注入
3. 构造器注入



对象与对象之间的关系可以简单的理解为对象之间的依赖关系:

> A类需要B类的一个实例来进行某些操作,比如在A类的方法中需要调用B类的方法来完成功能,叫做A类依赖于B类。
> 控制反转是一种将组件依赖关系的创建和管理置于程序外部的技术,由容器控制程序之间的关系,而不是由代码直接控制。



#### 接口注入

```java
public class ClassA {
    private InterfaceB clzB;
    public void doSomething() {
        Ojbect obj = Class.forName(Config.BImplementation).newInstance();
        clzB = (InterfaceB)obj;
        clzB.doIt();
    }
    
}
```

上面代码中,ClassA依赖于InterfaceB的实现,如何获得InterfaceB实现类的实例?传统的方法是在代码中创建 InterfaceB实现类的实例,并将赋予clzB。这样一来,ClassA在编译期即依赖于InterfaceB的实现.为了将调用者与实现者在编译 期分离,于是有了上面的代码。

我们根据预先在配置文件中设定的实现类的类名(Config.BImplementation),动态加载实现类,并通过InterfaceB强制转型后为ClassA所用,这就是接口注入的一个最原始的雏形。

```java
public class ClassA {
    private InterfaceB clzB;
    public Object doSomething(InterfaceB b) {
        clzB = b;
        return clzB.doIt();
    }
    
}
```

上面代码中,加载接口实现并创建其实例的工作由容器完成。

在运行期,InterfaceB实例将由容器提供。即使在IOC的概念尚未确立时,这样的方法也已经频繁出现在我们的代码中。

```java
public class MyServlet extends HttpServlet {
    public void doGet(HttpServletRequest request,HttpServletResponse response)throws ServletException, IOException {
        //do something
    }
}
```

HttpServletRequest和HttpServletResponse实例由Servlet Container在运行期动态注入。

#### SETTER设置注入

基于设置模式的依赖注入机制更加直观,也更加自然.

```java
public class ClassA {
    private InterfaceB clzB;
    public void setClzB(InterfaceB clzB) {
        this.clzB = clzB;
    }
    //do something
}
```



#### 构造器注入

```java
public class DIByConstructor {
    private final DataSource dataSource;
    public DIByConstructor(DataSource ds) {
        this.dataSource = ds;
    }
    //do something
}
```

构造器注入,即通过构造函数完成依赖关系的设定,容器通过调用类的构造方法将其所需的依赖关系注入其中。

#### 三种注入方式比较:

- 接口注入：
  接口注入模式因为具备侵入性，它要求组件必须与特定的接口相关联，因此并不被看好，实际使用有限。
- Setter 注入：
  对于习惯了传统 javabean 开发的程序员，通过 setter 方法设定依赖关系更加直观。
  如果依赖关系较为复杂，那么构造器注入模式的构造函数也会相当庞大，而此时设值注入模式则更为简洁。
  如果用到了第三方类库，可能要求我们的组件提供一个默认的构造函数，此时构造器注入模式也不适用。
- 构造器注入：
  在构造期间完成一个完整的、合法的对象。
  所有依赖关系在构造函数中集中呈现。
  依赖关系在构造时由容器一次性设定，组件被创建之后一直处于相对“不变”的稳定状态。

只有组件的创建者关心其内部依赖关系，对调用者而言，该依赖关系处于“黑盒”之中。

Spring使用注入方式，为什么使用注入方式，这系列问题实际归结起来就是一句话，Spring的注入和IoC反转控制是一回事。

理论上：第三种注入方式（构造函数注入）在符合java使用原则上更加合理，第二种注入方式（setter注入）作为补充。

实际上：第二种注入方式（setter注入）可以取得更加直观的效果，在使用工作上有不可比拟的优势，所以setter注入依赖关系应用更加广泛。

# 实现一个自己的IoC

## 约定

- 所有bean的生命周期交由IoC容器管理
- 所有被依赖的Bean通过构造方法注入
- 被依赖的Bean优先创建



## 代码示例

### 工程简介

新建一个maven项目，大致要实现下面的一个工程：

![](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856587305_8810.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

对于Car接口，定义四个方法：

```java
package Car;

public interface Car {
    public void start();
    public void stop();
    public void turnLeft();
    public void turnRight();
}
```

定义两种车，实现Car接口：

```java
package Car;

public class Renault implements Car {
    public void start() {
        System.out.println("---Renault starts---");
    }

    public void stop() {
        System.out.println("---Renault stops---");
    }

    public void turnLeft() {
        System.out.println("Renault turns left");
    }

    public void turnRight() {
        System.out.println("Renault turns right");
    }
}
```

```java
package Car;

public class Toyota implements Car {
    public void start() {
        System.out.println("---Toyota starts---");
    }

    public void stop() {
        System.out.println("---Toyota stops---");
    }

    public void turnLeft() {
        System.out.println("Toyota turns left");
    }

    public void turnRight() {
        System.out.println("Toyota turns right");
    }
}
```

定义User接口，对于一个User，他的唯一功能就是`goHome()`

```java
package User;

public interface User {
    public void goHome();
}
```

在这个例子中，User使用Car回家，那么定义一个抽象类Driver，实现User接口。Driver是抽象类的原因是：对于不同的Driver，所要回的家不一样，也就是用户的行为不同，因而需要在下面UserA和UserB中再定义。

```java
package User;

import Car.Car;

public abstract class Driver implements User {
    protected Car car;

    //所有被依赖的Bean通过构造方法注入
    public Driver(Car car){
        this.car = car;
    }

    public abstract void goHome();
}
```

定义UserA和UserB，继承Driver。

```java
package User;

import Car.Car;

public class UserA extends Driver{
    public UserA(Car car) {
        super(car);
    }

    public void goHome() {
        car.start();
        car.turnLeft();
        car.stop();
    }
}
```

```java
package User;

import Car.Car;

public class UserB extends Driver{
    public UserB(Car car) {
        super(car);
    }

    public void goHome() {
        car.start();
        car.turnRight();
        car.turnLeft();
        car.stop();
    }
}
```

### 实现IocContainer

```java
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/*
1.实例化bean
2.保存bean
3.提供bean
4.每一个bean要产生一个唯一的id与之相对应
 */
public class IoCContainer {
    private Map<String, Object> beans;

    IoCContainer(){
        beans = new ConcurrentHashMap<String, Object>();
    }
    /**
     * 根据beanId获取bean
     * @param beanId beanId
     * @return 获取的bean
     */
    public Object getBean(String beanId){
        return beans.get(beanId);
    }

    /**
     * 委托IoC容器创建bean
     * @param clazz 要创建的bean的类
     * @param beanId beanId
     * @param paramBeanIds 要创建的bean的类的构造方法所需要的beanIds
     */
    public void setBean(Class<?> clazz, String beanId, String... paramBeanIds){
        //1.组装构造方法所需要的参数值
        Object[] paramValues = new Object[paramBeanIds.length];
        for(int i = 0; i < paramBeanIds.length; i++){
            paramValues[i] = beans.get(paramBeanIds[i]);
        }
        //2.调用构造方法实例化bean
        Object bean = null;
        for (Constructor<?> constructor : clazz.getConstructors()) {
            try {
                bean = constructor.newInstance(paramValues);
            } catch (InstantiationException e) {
            } catch (IllegalAccessException e) {
            } catch (InvocationTargetException e) {
            }
            //不处理exception，因为每个bean可能用到的构造方法签名不同，最终总会实例化bean
        }

        if(bean == null){
            throw new RuntimeException("找不到合适的构造方法实例化bean");
        }
        //3.将实例化的bean放入map beans 中
        beans.put(beanId, bean);
    }
}
```

### 测试

```java
import Car.Renault;
import Car.Toyota;
import User.User;
import User.UserA;
import User.UserB;
import org.junit.Before;
import org.junit.Test;

public class IocContainerTest {

    private IoCContainer ioCContainer = new IoCContainer();

    @Before
    public void beforeTest(){
        //被依赖的Bean优先创建
        ioCContainer.setBean(Renault.class, "Renault");
        ioCContainer.setBean(Toyota.class, "Toyota");
        ioCContainer.setBean(UserA.class, "UserA", "Renault");
        ioCContainer.setBean(UserB.class, "UserB", "Toyota");
    }

    @Test
    public void test(){
        User userA = (User) ioCContainer.getBean("UserA");
        User userB = (User) ioCContainer.getBean("UserB");
        userA.goHome();
        userB.goHome();
    }
}
```

结果如下：

```
---Renault starts---
Renault turns left
---Renault stops---
---Toyota starts---
Toyota turns right
Toyota turns left
---Toyota stops---
```

