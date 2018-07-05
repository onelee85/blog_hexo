title: Spring系列-IOC
author: James
tags:
  - spring
  - ioc
categories:
  - 框架
date: 2013-08-26 10:06:00
---

# 简介

Spring 最重要的概念之一就是 IOC（Inversion of Control，控制倒转 ）,一起来分析下 Spring 的 IOC 容器。

<!-- more -->

# 为什么有IOC

Java程序中的每个业务逻辑至少需要两个或以上的对象来协作完成。通常，每个对象在使用他的合作对象时，自己均要使用像new object  这样的语法来完成创建对象的工作。对象间的耦合度高了。而IOC的思想是：Spring容器来实现这些相互依赖对象的创建、协调工作，由它来负责控制对象的生命周期和对象间的关系。对象只需要关系业务逻辑本身就可以了。从这方面来说，对象如何得到他的协作对象的责任被反转了。

# 原理

IoC的一个重点是在系统运行中，动态的向某个对象提供它所需要的其他对象。这一点是通过DI（Dependency Injection，依赖注入）来实现的。

> 通过DI，对象的依赖关系将由系统中负责协调各对象的第三方组件在创建对象的时候进行设定，对象无需自行创建或管理它们的依赖关系，依赖关系将被自动注入到需要它们的对象当中去 

# spring IOC容器 

- 原理就是通过Java的**反射技术**来实现的！通过反射我们可以获取类的所有信息(成员变量、类名等等等)！
- 再通过配置文件(xml)或者注解来**描述**类与类之间的关系
- 我们就可以通过这些配置信息和反射技术来**构建**出对应的对象和依赖关系了！

## 初始化过程

过程主要就是**读取XML资源，并解析，最终注册到Bean Factory中** 

Bean的初始化过程： 



![](/images/spring-ioc/bean_init.jpg)

**步骤** : 

1. BeanDefinitionReader**读取Resource所指向的配置文件资源**，然后解析配置文件。配置文件中每一个`<bean>`解析成一个**BeanDefinition对象**，并**保存**到BeanDefinitionRegistry中。
2. 容器扫描BeanDefinitionRegistry中的BeanDefinition；调用InstantiationStrategy**进行Bean实例化的工作**；使用**BeanWrapper完成Bean属性的设置**工作 。
3. 单例Bean缓存池：Spring 在DefaultSingletonBeanRegistry类中提供了一个用于缓存单实例 Bean 的**缓存器**，它是一个用HashMap实现的缓存器，单实例的Bean**以beanName为键保存在这个HashMap**中。 



## ApplicationContext和BeanFactory 

- ApplicationContext会利用Java反射机制自动识别出配置文件中定义的BeanPostProcessor、 InstantiationAwareBeanPostProcesso 和BeanFactoryPostProcessor**后置器**，并**自动将它们注册到应用上**下文中。而BeanFactory需要在代码中通过**手工调用**`addBeanPostProcessor()`方法进行注册
- ApplicationContext在**初始化**应用上下文的时候**就实例化所有单实例的Bean**。BeanFactory在初始化容器的时候并未实例化Bean，**直到**第一次访问某个Bean时**才**实例化目标Bean。 

![](/images/spring-ioc/beanfactory.png)

## bean的生命周期 

1. Spring容器 从XML 文件中读取bean的定义，并实例化bean。
2. Spring根据bean的定义填充所有的属性。
3. 如果bean实现了BeanNameAware 接口，Spring 传递bean 的ID 到 setBeanName方法。
4. 如果Bean 实现了 BeanFactoryAware 接口， Spring传递beanfactory 给setBeanFactory 方法。
5. 如果有任何与bean相关联的BeanPostProcessors，Spring会在postProcesserBeforeInitialization()方法内调用它们。
6. 如果bean实现IntializingBean了，调用它的afterPropertySet方法，如果bean声明了初始化方法，调用此初始化方法。
7. 如果有BeanPostProcessors 和bean 关联，这些bean的postProcessAfterInitialization() 方法将被调用。
8. 如果bean实现了 DisposableBean，它将调用destroy()方法。

