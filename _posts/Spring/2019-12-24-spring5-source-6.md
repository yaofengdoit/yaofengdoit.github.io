---
layout: post
title: spring5源码分析系列（六）——IOC容器的初始化（四）
category: Spring
tags: [Spring]
---

前言：上一篇讲到了解析<list>子元素，此篇我们继续后面的内容。

(15)解析过后的BeanDefinition在IOC容器中的注册

接下来分析DefaultBeanDefinitionDocumentReader对Bean定义转换的Document对象解析的流程中，在其parseDefaultElement方法中完成对Document对象的解析后得到封装BeanDefinition的BeanDefinitionHold对象，
然后调用BeanDefinitionReaderUtils的registerBeanDefinition方法向IOC容器注册解析的Bean，BeanDefinitionReaderUtils中注册源码如下：

```
//将解析的BeanDefinitionHold注册到容器中
public static void registerBeanDefinition(
        BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
        throws BeanDefinitionStoreException {

    //获取解析的BeanDefinition的名称
    String beanName = definitionHolder.getBeanName();
    //向IOC容器注册BeanDefinition
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

    //如果解析的BeanDefinition有别名，向容器为其注册别名
    String[] aliases = definitionHolder.getAliases();
    if (aliases != null) {
        for (String alias : aliases) {
            registry.registerAlias(beanName, alias);
        }
    }
}
```

当调用BeanDefinitionReaderUtils向IOC容器注册解析的BeanDefinition时，真正完成注册功能的是DefaultListableBeanFactory。

(16)DefaultListableBeanFactory向IOC容器注册解析后的BeanDefinition

DefaultListableBeanFactory中使用一个HashMap的集合对象存放IOC容器中注册解析的BeanDefinition，向IOC容器注册的主要源码如下：

```
//存储注册信息的BeanDefinition
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);

//向IOC容器注册解析的BeanDefiniton
@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
        throws BeanDefinitionStoreException {

    Assert.hasText(beanName, "Bean name must not be empty");
    Assert.notNull(beanDefinition, "BeanDefinition must not be null");

    //校验解析的BeanDefiniton
    if (beanDefinition instanceof AbstractBeanDefinition) {
        try {
            ((AbstractBeanDefinition) beanDefinition).validate();
        }
        catch (BeanDefinitionValidationException ex) {
            throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                    "Validation of bean definition failed", ex);
        }
    }

    BeanDefinition oldBeanDefinition;

    oldBeanDefinition = this.beanDefinitionMap.get(beanName);

    if (oldBeanDefinition != null) {
        if (!isAllowBeanDefinitionOverriding()) {
            throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                    "Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
                    "': There is already [" + oldBeanDefinition + "] bound.");
        }
        else if (oldBeanDefinition.getRole() < beanDefinition.getRole()) {
            // e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
            if (this.logger.isWarnEnabled()) {
                this.logger.warn("Overriding user-defined bean definition for bean '" + beanName +
                        "' with a framework-generated bean definition: replacing [" +
                        oldBeanDefinition + "] with [" + beanDefinition + "]");
            }
        }
        else if (!beanDefinition.equals(oldBeanDefinition)) {
            if (this.logger.isInfoEnabled()) {
                this.logger.info("Overriding bean definition for bean '" + beanName +
                        "' with a different definition: replacing [" + oldBeanDefinition +
                        "] with [" + beanDefinition + "]");
            }
        }
        else {
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Overriding bean definition for bean '" + beanName +
                        "' with an equivalent definition: replacing [" + oldBeanDefinition +
                        "] with [" + beanDefinition + "]");
            }
        }
        this.beanDefinitionMap.put(beanName, beanDefinition);
    }
    else {
        if (hasBeanCreationStarted()) {
            //注册的过程中需要线程同步，以保证数据的一致性
            synchronized (this.beanDefinitionMap) {
                this.beanDefinitionMap.put(beanName, beanDefinition);
                List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
                updatedDefinitions.addAll(this.beanDefinitionNames);
                updatedDefinitions.add(beanName);
                this.beanDefinitionNames = updatedDefinitions;
                if (this.manualSingletonNames.contains(beanName)) {
                    Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames);
                    updatedSingletons.remove(beanName);
                    this.manualSingletonNames = updatedSingletons;
                }
            }
        }
        else {
            this.beanDefinitionMap.put(beanName, beanDefinition);
            this.beanDefinitionNames.add(beanName);
            this.manualSingletonNames.remove(beanName);
        }
        this.frozenBeanDefinitionNames = null;
    }

    //检查是否有同名的BeanDefinition已经在IOC容器中注册
    if (oldBeanDefinition != null || containsSingleton(beanName)) {
        //重置所有已经注册过的BeanDefinition的缓存
        resetBeanDefinition(beanName);
    }
}
```

分析到这里，Bean定义资源文件中配置的Bean被解析过后，已经注册到IOC容器中，被容器管理起来，真正完成了IOC容器初始化所做的全部工作。
现在IOC容器中已经建立了整个Bean的配置信息，这些BeanDefinition信息已经可以使用，并且可以被检索，IOC容器的作用就是对这些注册的Bean定义信息进行处理和维护。
这些的注册的Bean定义信息是IOC容器控制反转的基础，正是有了这些注册的数据，容器才可以进行依赖注入。

通过这四篇文章的分析，现在总结一下IOC容器初始化的基本步骤：
初始化的入口在容器实现中的refresh()调用；
对bean定义载入IOC容器使用的方法是loadBeanDefinition。

容器初始化全过程的时序图如下：
![](https://yaofengdoit.github.io/assets/images/2019/spring/6-1.png)

大致过程如下：通过ResourceLoader来完成资源文件位置的定位，DefaultResourceLoader是默认的实现，同时上下文本身就给出了ResourceLoader的实现，可以从类路径、文件系统、URL等方式来定位资源位置。
如果是XmlBeanFactory作为IOC容器，那么需要为它指定bean定义的资源，即bean定义文件时通过抽象成Resource来被IOC容器处理，
容器通过BeanDefinitionReader来完成定义信息的解析和Bean信息的注册, 通常使用XmlBeanDefinitionReader来解析bean的xml定义文件——实际的处理过程是委托给BeanDefinitionParserDelegate来完成的，
从而得到bean的定义信息，这些信息在Spring中使用BeanDefinition对象来表示——可想到loadBeanDefinition、RegisterBeanDefinition这些相关方法，
这些方法都是为处理BeanDefinitin服务的，容器解析得到BeanDefinition以后，需要把它在IOC容器中注册，这由IOC实现BeanDefinitionRegistry接口来实现。
注册过程就是在IOC容器内部维护的一个HashMap来保存得到的BeanDefinition的过程。这个HashMap是IOC容器持有Bean信息的场所，以后对Bean的操作都是围绕这个HashMap来实现的。

然后就可以通过BeanFactory和ApplicationContext来使用Spring IOC的服务了，在使用IOC容器的时候，除了少量粘合代码，绝大多数以正确IOC风格编写的应用程序代码完全不用关心如何到达工厂，
因为容器将把这些对象与容器管理的其他对象钩在一起。基本的策略是把工厂放到已知的地方，最好是放在对预期使用的上下文有意义的地方，以及代码将实际需要访问工厂的地方。 
Spring本身提供了对声明式载入web应用程序用法的应用程序上下文，并将其存储在ServletContext中的框架实现。

在使用Spring IOC容器的时候还需要区别两个概念: BeanFactory和FactoryBean。

BeanFactory指的是IOC容器的编程抽象，例如ApplicationContext、XmlBeanFactory等，这些都是IOC容器的具体表现，需要使用什么样的容器，由客户决定，Spring为我们提供了丰富的选择。
FactoryBean只是一个可以在IOC容器中被管理的一个Bean，是对各种处理过程和资源使用的抽象，FactoryBean在需要时产生另一个对象，而不返回FactoryBean本身，
我们可以把它看成是一个抽象工厂，对它的调用返回的是工厂生产的产品。所有的FactoryBean都实现特殊的org.springframework.beans.factory.FactoryBean接口，当使用容器中FactoryBean的时候，该容器不会返回FactoryBean本身,而是返回其生成的对象。
Spring包括了大部分的通用资源和服务访问抽象的FactoryBean的实现，包括：对JNDI查询的处理，对代理对象的处理，对事务性代理的处理，对RMI代理的处理等，
这些我们都可以看成是具体的工厂，看成是Spring为我们建立好的工厂。也就是说Spring通过使用抽象工厂模式为我们准备了一系列工厂来生产一些特定的对象，免除我们手工重复的工作，要使用时只需要在IOC容器里配置好就能很方便的使用了。






