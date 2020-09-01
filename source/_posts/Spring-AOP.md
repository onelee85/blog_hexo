title: Spring系列-AOP
author: James
tags:
  - spring
  - aop
categories:
  - 框架
date: 2013-09-13 16:30:00
---

# 前言

AOP（Aspect Orient Programming），一般称为面向切面编程，作为面向对象的一种补充，用于处理系统中分布于各个模块的横切关注点。 OOP允许你定义从上到下的关系，但并不适合定义从左到右的关系。例如日志功能。日志代码往往水平地散布在所有对象层次中，而与它所散布到的对象的核心功能毫无关系。这种散布在各处的无关的代码被称为横切（cross-cutting）代码，在OOP设计中，它导致了大量代码的重复，而不利于各个模块的重用。AOP技术则恰恰相反，它利用一种称为“横切”的技术，剖解开封装的对象内部，并将那些影响了多个类的公共行为封装到一个可重用模块，并将其名为“Aspect”，即方面。所谓“方面”，简单地说，就是将那些与业务无关，却为业务模块所共同调用的逻辑或责任封装起来，便于减少系统的重复代码，降低模块间的耦合度，并有利于未来的可操作性和可维护性。

 <!-- more -->

# 用途

-  记录日志，在方法执行前后记录系统日志。 
- 性能监控，在方法调用前后记录调用时间，方法执行太长或超时报警。 
- 权限验证，方法执行前验证是否有权限执行当前方法，没有则抛出没有权限执行异常，由业务代码捕捉。 
- 缓存代理，缓存某方法的返回值，下次执行该方法时，直接从缓存里获取。 

 # 术语

- **Aspect（切面）**：一个关注点的模块化，这个关注点可以横切多个对象。
- **Joinpoint （连接点）**：在程序执行过程中某个特点的点，例如某方法调用的时候或者处理异常的时候。
- **Pointcut（切入点）**：匹配连接点的断言，通知和一个切入点表达式关联，满足这个切入点的连接点上运行。
- **Advice（通知）**：在切面的某个特点的连接点
  - 前置通知：在某连接点之前执行的通知，不能阻止连接点之前执行的流程
  - 后置通知：在连接点完成执行后的通知。
  - 环绕通知：包围一个连接点的通知，可以在方法调用的前后完成自定义的行为，可以选择是否继续执行连接点或者直接返回。
  - 异常通知：在方法抛出异常退出时执行的通知。
  - 最终通知：当某连接点退出的时候执行的通知。（不论是否正常返回还是异常）

# Spring 的 Aop  

Spring提供了两种方式来生成代理对象:  JDKProxy和Cglib，具体使用哪种方式生成由AopProxyFactory根据AdvisedSupport对象的配置来决定。默认的策略是如果目标类是接口，则使用JDK动态代理技术，否则使用Cglib来生成代理。 

主要以下几个步骤： 
1. 定义普通业务组件。
2. 定义切入点，一个切入点可能横切多个业务组件。
3. 定义增强处理，增强处理就是在 AOP 框架为普通业务组件织入的处理动作。

## 简单demo

```java
@Aspect
public class TransactionDemo {
    @Pointcut(value = "execution(* com.lijiao.core.service.*.*.*(..))")
    public void point() {
    }

    @Before(value = "point()")
    public void before() {
        System.out.println("transaction begin");
    }

    @AfterReturning(value = "point()")
    public void after() {
        System.out.println("transaction commit");
    }

    @Around("point()")
    public void around(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("transaction begin");
        joinPoint.proceed();
        System.out.println("transaction commit");
    }
}

```

## 要点

1. Spring 的 AOP 代理由 Spring 的 IoC 容器负责生成、管理，其依赖关系也由 IoC 容器负责管理。因此，AOP 代理可以直接使用容器中的其他 Bean 实例作为目标，这种关系可由 IoC 容器的依赖注入提供。 
2. Spring提供了两种方式来生成代理对象: JDKProxy和Cglib，具体使用哪种方式生成由AopProxyFactory根据AdvisedSupport对象的配置来决定。默认的策略是如果目标类是接口，则使用JDK动态代理技术，否则使用Cglib来生成代理。



*JDK动态代理*

- JDK动态代理主要涉及到java.lang.reflect包中的两个类：Proxy和InvocationHandler。InvocationHandler是一个接口，通过实现该接口定义横切逻辑，并通过反射机制调用目标类的代码，动态将横切逻辑和业务逻辑编制在一起。
- Proxy利用InvocationHandler动态创建一个符合某一接口的实例，生成目标类的代理对象。

*CGLib动态代理*

- CGLib全称为Code Generation Library，是一个强大的高性能，高质量的代码生成类库，可以在运行期扩展Java类与实现Java接口，CGLib封装了asm，可以再运行期动态生成新的class。和JDK动态代理相比较：JDK创建代理有一个限制，就是只能为接口创建代理实例，而对于没有通过接口定义业务方法的类，则可以通过CGLib创建动态代理。

 



 

 

 

 

 

 

 

 

 

 

 