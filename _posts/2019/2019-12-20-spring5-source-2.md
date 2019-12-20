---
layout: post
title: spring5源码分析系列（二）——spring核心容器体系结构
category: Spring
tags: [Spring]
no-post-nav: true
---

首先我们来认识下IOC和DI：
- IOC(Inversion of Control)控制反转：控制反转，就是把原先代码里面需要实现的对象创建、依赖的代码，反转给容器来帮忙实现。所以需要创建一个容器，并且需要一种描述来让容器知道需要创建的对象与对象的关系。这个描述最具体表现就是可配置的文件。
- DI(Dependency Injection)依赖注入：指对象是被动接受依赖类而不是自己主动去找，即对象不是从容器中查找它依赖的类，而是在容器实例化对象的时候主动将它依赖的类注入给它。

思考以下问题：
- 对象和对象关系怎么表示？可以用xml，properties文件等语义化配置文件表示。
- 描述对象关系的文件存放在哪里？classpath，filesystem，servletContext等。
- 有了配置文件，还需要对配置文件解析。不同的配置文件对对象的描述不一样，如标准的，自定义声明式的，如何统一呢？这就需要在内部有一个统一的关于对象的定义，所有外部的描述都必须转化成统一的描述定义。
- 如何对不同的配置文件进行解析？需要对不同的配置文件语法，采用不同的解析器。

接下来我们看Spring是怎么做到的。

1.BeanFactory

Spring使用工厂模式创建Bean，这一系列的Bean工厂，也即IOC 容器，为开发者管理对象间的依赖关系提供了很多便利和基础服务。Spring中有许多IOC 容器的实现供选择和使用，其相互关系如下：

![](https://yaofengdoit.github.io/assets/images/2019/spring/2-1.png)

BeanFactory作为最顶层的一个接口类，定义了IOC容器的基本功能规范，BeanFactory 有三个子类：ListableBeanFactory、HierarchicalBeanFactory和AutowireCapableBeanFactory。最终的默认实现类是DefaultListableBeanFactory，他实现了所有的接口。

为何要定义这么多层次的接口呢？查阅这些接口的源码和说明发现，每个接口都有他使用的场合，主要是为了区分在Spring内部在操作过程中对象的传递和转化过程中，对对象的数据访问所做的限制。例如ListableBeanFactory接口表示这些Bean是可列表的，HierarchicalBeanFactory表示这些 Bean是有继承关系的，也就是每个Bean有可能有父Bean。AutowireCapableBeanFactory接口定义 Bean的自动装配规则。这四个接口共同定义了Bean的集合、Bean之间的关系、Bean的行为。

我们看一下BeanFactory源码里的方法：

```
public interface BeanFactory {
    // 对FactoryBean的转义定义，因为如果使用bean的名字检索FactoryBean得到的对象是工厂生成的对象，
    // 如果需要得到工厂本身，需要转义
    String FACTORY_BEAN_PREFIX = "&";
    
    // 根据bean的名字，获取在IOC容器中得到bean实例
    Object getBean(String name) throws BeansException;
    	
    //根据bean的名字和Class类型来得到bean实例，增加了类型安全验证机制。
    <T> T getBean(String name, @Nullable Class<T> requiredType) throws BeansException;
    
    Object getBean(String name, Object... args) throws BeansException;
    <T> T getBean(Class<T> requiredType) throws BeansException;
    <T> T getBean(Class<T> requiredType, Object... args) throws BeansException;
    
    // 提供对bean的检索，看看是否在IOC容器有这个名字的bean
    boolean containsBean(String name);
    
    //根据bean名字得到bean实例，并同时判断这个bean是不是单例
    boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
    
    boolean isPrototype(String name) throws NoSuchBeanDefinitionException;
    boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;
    boolean isTypeMatch(String name, @Nullable Class<?> typeToMatch) throws NoSuchBeanDefinitionException;
    
    // 得到bean实例的Class类型
    @Nullable
    Class<?> getType(String name) throws NoSuchBeanDefinitionException;
    
    // 得到bean的别名，如果根据别名检索，那么其原名也会被检索出来
    String[] getAliases(String name);
}
```
对于FACTORY_BEAN_PREFIX，这里解释以下：与BeanFactory很相似的有一个叫FactoryBean的类，从名字上很容易混淆，BeanFactory首先是Factory,而FactoryBean是bean，是一种特殊的bean, 这种特殊的bean会生产另一种bean, 对于普通的bean,通过BeanFactory的getBean方法可以获取这个bean,而对于FactoryBean来说，通过getBean获得的是FactoryBean生产的bean,而不是FactoryBean本身，如果想要获取FactoryBean本身，那么可以加前缀&，spring就会明白，原来是需要FactoryBean 。

BeanFactory只对IOC容器的基本行为作了定义，不用关心Bean是如何定义怎样加载的。
要知道工厂是如何产生对象的，需要看具体的IOC 容器实现，Spring 提供了许多 IOC 容器的实现。比如XmlBeanFactory，ClasspathXmlApplicationContext 等。其中 XmlBeanFactory是针对最基本的IOC容器的实现，这个IOC 容器可以读取XML文件定义的BeanDefinition（XML文件中对bean的描述）。ApplicationContext是Spring提供的一个高级的IOC 容器，它除了能够提供IOC容器的基本功能外，还为用户提供了以下的附加服务：
- 支持信息源，可以实现国际化。（实现MessageSource接口）
- 访问资源。(实现ResourcePatternResolver接口)
- 支持应用事件。(实现ApplicationEventPublisher 接口)

2.BeanDefinition

SpringIOC容器管理了我们定义的各种Bean对象及其相互的关系，Bean对象在Spring实现中是
以BeanDefinition来描述的，其继承体系如下： 

![](https://yaofengdoit.github.io/assets/images/2019/spring/2-2.png)

Bean的解析过程非常复杂，功能被分的很细。Bean的解析主要是对Spring配置文件的解析。
解析过程主要通过下图中的类完成：

![](https://yaofengdoit.github.io/assets/images/2019/spring/2-3.png)




