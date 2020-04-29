---
layout: post
title: spring5源码分析系列（一）——spring5框架模块
category: Spring
tags: [Spring]
---

spring总共大约20个模块，这些模块被整合在核心容器（Core Container）、AOP和设备支持、数据访问及集成、Web、报文发送、Test 6个模块集合。

![](https://yaofengdoit.github.io/assets/images/2019/spring/1-1.png)

组成Spring框架的每个模块集合或者模块都可以单独存在，也可以一个模块或者多个模块联合实现。
模块组成和功能如下：

1、核心容器：spring-beans、spring-core、spring-context、spring-expression 4个模块组成。
- spring-beans和spring-core模块是Spring框架的核心模块，包含了控制反转（IOC）和依赖注入（DI）。BeanFactory 接口是 Spring 框架中的核心接口，它是工厂模式的具体实现。BeanFactory 使用控制反转对应用程序的配置和依赖性规范与实际的应用程序代码进行了分离。但 BeanFactory 容器实例化后并不会自动实例化 Bean，只有当Bean
被使用时BeanFactory容器才会对该Bean进行实例化与依赖关系的装配。
- spring-context模块构架于核心模块之上，他扩展了BeanFactory，为她添加了Bean 生命周期控制、框架事件体系以及资源加载透明化等功能。此外该模块还提供了许多企业级支持，如邮件访问、远程访问、任务调度等。ApplicationContext是该模块的核心接口，她是BeanFactory的超类，与BeanFactory不同，ApplicationContext 容器实例化后会自动对所有的单实例Bean进行实例化与依赖关系的装配，使之处于待用状态。
- spring-expression模块是统一表达式语言（EL）的扩展模块，可以查询、管理运行中的对象，同时也方便的可以调用对象方法、操作数组、集合等。它的语法类似于传统EL，但提供了额外的功能，最出色的要数函数调用和简单字符串的模板函数。这种语言的特性是基于Spring产品的需求而设计，他可以非常方便地同Spring IOC进行交互。

2、AOP和设备支持：spring-aop、spring-aspects、spring-instrument 3个模块组成。
- spring-aop是Spring的另一个核心模块，是AOP主要的实现模块。在 Spring 中，他是以JVM的动态代理技术为基础，然后设计出了一系列的AOP横切实现，比如前置通知、返回通知、异常通知等。同时，Pointcut接口来匹配切入点，可以使用现有的切入点来设计横切面，也可以扩展相关方法根据需求进行切入。
- spring-aspects模块集成自AspectJ框架，主要是为SpringAOP提供多种AOP实现方法。
- spring-instrument模块是基于JAVA SE中的"java.lang.instrument"进行设计的，是 AOP的一个支援模块，主要作用是在JVM启用时，生成一个代理类，程序员通过代理类在运行时修改类的字节，从而改变一个类的功能， 实现AOP的功能。

3、数据访问及集成：spring-jdbc、spring-tx、spring-orm、spring-oxm、spring-jms五个模块组成。
- spring-jdbc模块是Spring提供的JDBC抽象框架的主要实现模块，用于简化Spring JDBC。主要是提供JDBC模板方式、关系数据库对象化方式、SimpleJdbc方式、事务管理来简化JDBC编程，主要实现类是JdbcTemplate、SimpleJdbcTemplate以及 NamedParameterJdbcTemplate。
- spring-tx模块是Spring JDBC事务控制实现模块。使用Spring框架，它对事务做了很好的封装，通过它的AOP配置，可以灵活的配置在任何一层。事务是以业务逻辑为基础的，一个完整的业务应该对应业务层里的一个方法，如果业务操作失败，则整个事务回滚。持久层的设计则应该遵循一个很重要的原则：保证操作的原子性，即持久层里的每个方法都应该是不可以分割的。
- spring-orm模块是ORM框架支持模块，主要集成Hibernate，Java Persistence API (JPA) 和Java Data Objects (JDO) 用于资源管理、数据访问对象(DAO)的实现和事务策略。 
- spring-jms模块（Java Messaging Service）能够发送和接受信息，自Spring Framework 4.1以后，还提供了对spring-messaging模块的支撑。
- spring-oxm模块主要提供一个抽象层以支撑OXM（OXM 是 Object-to-XML-Mapping 的缩写，它是一个O/M-mapper，将java对象映射成 XML数据，或者将XML数据映射成 java对象）。

4、Web：spring-web、spring-webmvc、spring-websocket、spring-webflux 四个模块组成。
- spring-web模块为Spring提供了最基础Web支持，主要建立于核心容器之上，通过 Servlet或者Listeners来初始化IOC容器，也包含一些与Web相关的支持。 
- spring-webmvc模块是一 个Web-Servlet模块，实现了Spring MVC
（model-view-Controller）的Web应用。
- spring-websocket模块主要是与Web前端的全双工通讯的协议。 
- spring-webflux是一个新的非堵塞函数式Reactive Web框架，可以用来建立异步的、非阻塞事件驱动的服务，并且扩展性非常好。

5、报文发送：spring-messaging 1个模块组成。
- spring-messaging是从Spring4开始新加入的一个模块，主要职责是为Spring框架集成一些基础的报文传送应用。

6、Test：spring-test 1个模块组成。
- spring-test模块主要为测试提供支持的。


spring5各个模块之间的依赖关系：

![](https://yaofengdoit.github.io/assets/images/2019/spring/1-2.png)




