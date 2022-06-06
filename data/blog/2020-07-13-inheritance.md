---
title: Spring-Bean's Inheritance
summary: Inheritance 
date: 2020-07-13
tags: ['Java', 'Design Pattern', 'Spring']
---

A bean definition can contain a lot of configuration information, including constructor arguments, property values, and container-specific information, such as the initialization method, a static factory method name, and so on. A child bean definition inherits configuration data from a parent definition. The child definition can override some values or add others as needed. Using parent and child bean definitions can save a lot of typing. Effectively, this is a form of templating.

# Bean的继承

假设我们现在有这么一个业务关系：

![](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856595333_1315.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

其中Java代码是这样的：

```java
public class Family {
    private String name;
    private String address;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getAddress() {
        return address;
    }

    public void setAddress(String address) {
        this.address = address;
    }
}

public class MemberA extends Family {
    private String firstName;
    private int age;

    public String getFirstName() {
        return firstName;
    }

    public void setFirstName(String firstName) {
        this.firstName = firstName;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "MemberA{" +
                "name='" + getName() + '\'' +
                ", address='" + getAddress() + '\'' +
                ", firstName='" + firstName + '\'' +
                ", age=" + age +
                '}';
    }
}
//MemberB同理
```



我们如何用xml配置文件，简化Bean的继承？

```xml
<bean class="Family" id="family" abstract="true">
  <property name="name" value="Lee"/>
  <property name="address" value="1 Rue de Paris"/>
</bean>

<bean class="MemberA" id="memberA" parent="family">
  <property name="firstName" value="Sylvain"/>
  <property name="age" value="20"/>
</bean>

<bean class="MemberB" id="memberB" parent="family">
  <property name="firstName" value="Sylvie"/>
  <property name="age" value="19"/>
</bean>
```

可见，两个Member都继承了Family，并且在xml中有`parent="family"`，指向Family类的id。但Family是一个抽象类，它不需要得到实例化，因此我们使用`abstract="true"`来表示，这个类在容器中不会被实例化。测试一下逻辑关系，两个Member的姓和地址应该是一样的：

```java
public void test(){
  ApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");
  MemberA memberA = context.getBean("memberA",MemberA.class);
  System.out.println("memberA = " + memberA);
  MemberB memberB = context.getBean("memberB",MemberB.class);
  System.out.println("memberB = " + memberB);
}
```

```
memberA = MemberA{name='Lee', address='1 Rue de Paris', firstName='Sylvain', age=20}
memberB = MemberB{name='Lee', address='1 Rue de Paris', firstName='Sylvie', age=19}
```

# 抽象出样板

如果我们有多个类，使用部分相同的配置信息，但不构成继承关系，我们也可以通过xml配置，减少代码量。

假如我们不再使用Family，而是用MemberA，B表示所有信息，像这样：

```java
public class MemberA {
    private String name;
    private String address;
    private String firstName;
    private int age;

    public String getFirstName() {
        return firstName;
    }

    public void setFirstName(String firstName) {
        this.firstName = firstName;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getAddress() {
        return address;
    }

    public void setAddress(String address) {
        this.address = address;
    }


    @Override
    public String toString() {
        return "MemberA{" +
                "name='" + name + '\'' +
                ", address='" + address + '\'' +
                ", firstName='" + firstName + '\'' +
                ", age=" + age +
                '}';
    }
}
```

MemberB同理，仅仅是复制粘贴。两个Member的姓和地址应该还是不变，那就可以提取出相同的信息，做成模版。

```xml
<bean id="family" abstract="true">
  <property name="name" value="Lee"/>
  <property name="address" value="1 Rue de Paris"/>
</bean>

<bean class="MemberA" id="memberA" parent="family">
  <property name="firstName" value="Sylvain"/>
  <property name="age" value="20"/>
</bean>

<bean class="MemberB" id="memberB" parent="family">
  <property name="firstName" value="Sylvie"/>
  <property name="age" value="19"/>
</bean>
```

唯一的变化是，没有了Family这个类，两个Member独立出来，但为了使用同一套模版中的信息，还是要指向一个相同的Parent。这个Parent，没有class属性，因为他在Java代码中并不存在，是完全抽象的。

测试结果还是不变的。

```
memberA = MemberA{name='Lee', address='1 Rue de Paris', firstName='Sylvain', age=20}
memberB = MemberB{name='Lee', address='1 Rue de Paris', firstName='Sylvie', age=19}
```

The parent bean cannot be instantiated on its own because it is incomplete, and it is also explicitly marked as `abstract`. When a definition is `abstract`, it is usable only as a pure template bean definition that serves as a parent definition for child definitions. Trying to use such an `abstract` parent bean on its own, by referring to it as a ref property of another bean or doing an explicit `getBean()` call with the parent bean ID returns an error. Similarly, the container’s internal `preInstantiateSingletons()` method ignores bean definitions that are defined as abstract.

> `ApplicationContext` pre-instantiates all singletons by default. Therefore, it is important (at least for singleton beans) that if you have a (parent) bean definition which you intend to use only as a template, and this definition specifies a class, you must make sure to set the *abstract* attribute to *true*, otherwise the application context will actually (attempt to) pre-instantiate the `abstract` bean.

