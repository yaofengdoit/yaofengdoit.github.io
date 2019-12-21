---
layout: post
title: spring5源码分析系列（三）——IOC容器的初始化（一）
category: Spring
tags: [Spring]
no-post-nav: true
---

前言：
IOC容器的初始化包括BeanDefinition的Resource定位、载入、注册三个基本过程。
本文以ApplicationContext为例讲解，XmlWebApplicationContext、ClasspathXmlApplicationContext等都属于这个继承体系，这些都是我们日常开发中很熟悉的。其继承体系如下图：

![](https://yaofengdoit.github.io/assets/images/2019/spring/3-1.png)

ApplicationContext允许上下文嵌套，通过接口中的此方法保持父上下文，可以维持一个上下文体系：
```
@Nullable
ApplicationContext getParent();
```
对于Bean的查找，可以在这个上下文体系中发生，首先检查当前上下文，其次是父上下文，逐级向上，
这样就为不同的Spring应用提供了一个共享的Bean定义环境。

接下来我们分别演示下两种IOC容器的创建过程。

1.XmlBeanFactory IOC容器的创建过程

首先看一下XmlBeanFactory的源码：
```
public class XmlBeanFactory extends DefaultListableBeanFactory {
    private final XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this);
    
    public XmlBeanFactory(Resource resource) throws BeansException {
        this(resource, null);
    }
    
    public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
        super(parentBeanFactory);
        this.reader.loadBeanDefinitions(resource);
    }
}
```
我们可以根据以下简单的例子跟进源码，理解定位、载入、注册的全过程：
```
// 根据Xml配置文件创建Resource资源对象，该对象中包含了BeanDefinition的信息 
ClassPathResource resource = new ClassPathResource("spring-test.xml"); 
// 创建DefaultListableBeanFactory 
DefaultListableBeanFactory factory = new DefaultListableBeanFactory(); 
// 创建XmlBeanDefinitionReader读取器，用于载入BeanDefinition(需要BeanFactory作为参数，是因为会将读取的信息回调配置给factory) 
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory); 
// XmlBeanDefinitionReader执行载入BeanDefinition的方法，最后会完成Bean的载入和注册。 
// 完成后Bean就成功的放置到IOC容器当中，以后我们就可以从中取得Bean来使用
reader.loadBeanDefinitions(resource);
```

2.FileSystemXmlApplicationContext IOC容器的创建过程

```
ApplicationContext = new FileSystemXmlApplicationContext(xmlPath);
```

先看下它的构造函数：
```
public FileSystemXmlApplicationContext(String configLocation) throws BeansException {
    this(new String[] {configLocation}, true, null);
}
```
实际调用的构造函数是：
```
public FileSystemXmlApplicationContext(String[] configLocations, boolean refresh, @Nullable ApplicationContext parent) throws BeansException {
    super(parent);
    setConfigLocations(configLocations);
    if (refresh) {
        refresh();
    }
}
```

(1)设置资源加载器和资源定位

通过分析FileSystemXmlApplicationContext的源码可知，创建FileSystemXmlApplicationContext容器时，构造方法做了以下两项重要工作： 
首先，调用父类容器的构造方法(super(parent)方法)为容器设置好Bean资源加载器；
然后，调用父类AbstractRefreshableConfigApplicationContext的setConfigLocations(configLocations)方法设置Bean定义资源文件的定位路径。

通过追踪FileSystemXmlApplicationContext的继承体系，发现其父类的父类AbstractApplicationContext中初始化IOC容器所做的主要源码如下：
```
public abstract class AbstractApplicationContext extends DefaultResourceLoader implements ConfigurableApplicationContext {
    // 静态初始化块，在整个容器创建过程中只执行一次
	static {
		// 为了避免应用程序在Weblogic8.1关闭时出现类加载异常加载问题，加载IoC容器关闭事件(ContextClosedEvent)类
		ContextClosedEvent.class.getName();
	}
	public AbstractApplicationContext() {
        this.resourcePatternResolver = getResourcePatternResolver();
    }
    public AbstractApplicationContext(@Nullable ApplicationContext parent) {
        this();
        setParent(parent);
    }
    // 获取一个Spring Source的加载器用于读入Spring Bean定义资源文件
    protected ResourcePatternResolver getResourcePatternResolver() {
        //AbstractApplicationContext继承DefaultResourceLoader，因此也是一个资源加载器(Spring资源加载器，其getResource(String location)方法用于载入资源)
        return new PathMatchingResourcePatternResolver(this);
    }
}
```

AbstractApplicationContext构造方法中调用了PathMatchingResourcePatternResolver的构造方法创建Spring资源加载器：
```
public PathMatchingResourcePatternResolver(ResourceLoader resourceLoader) {
    Assert.notNull(resourceLoader, "ResourceLoader must not be null");
    // 设置Spring的资源加载器
    this.resourceLoader = resourceLoader;
}
```

设置了容器的资源加载器后，FileSystemXmlApplicationContext执行setConfigLocations方法，
通过调用其父类AbstractRefreshableConfigApplicationContext的方法进行对Bean定义资源文件的定位，该方法的源码如下：

```
// 处理单个资源文件路径为一个字符串的情况
public void setConfigLocation(String location) {
    // String CONFIG_LOCATION_DELIMITERS = ",; /t/n";
    // 即多个资源文件路径之间用” ,; \t\n”分隔，解析成数组形式
    setConfigLocations(StringUtils.tokenizeToStringArray(location, CONFIG_LOCATION_DELIMITERS));
}
// 解析Bean定义资源文件的路径，处理多个资源文件字符串数组
public void setConfigLocations(@Nullable String... locations) {
    if (locations != null) {
        Assert.noNullElements(locations, "Config locations must not be null");
        this.configLocations = new String[locations.length];
        for (int i = 0; i < locations.length; i++) {
            // resolvePath为同一个类中将字符串解析为路径的方法
            this.configLocations[i] = resolvePath(locations[i]).trim();
        }
    }
    else {
        this.configLocations = null;
    }
}
```

通过这两个方法的源码可以看出，既可以使用一个字符串来配置多个Spring Bean定义资源文件，也可以使用字符串数组，即下面两种方式都是可以的：
```
// 多个资源文件路径之间可以是用” , ; \t\n”等分隔
ClasspathResource res = new ClasspathResource(“a.xml,b.xml,……”);
ClasspathResource res = new ClasspathResource(newString[]{“a.xml”,”b.xml”,……});
```

至此，Spring IOC容器在初始化时将配置的Bean定义资源文件定位为Spring封装的Resource。

(2)AbstractApplicationContext的refresh函数载入Bean定义过程

Spring IOC容器对Bean定义资源的载入是从refresh()函数开始的，refresh()是一个模板方法，refresh()方法的作用是：
在创建IOC容器前，如果已经有容器存在，则需要把已有的容器销毁和关闭，以保证在refresh之后使用的是新建立起来的IOC容器。
refresh的作用类似于对IOC容器的重启，在新建立好的容器中对容器进行初始化，对Bean定义资源进行载入。

FileSystemXmlApplicationContext通过调用其父类AbstractApplicationContext的refresh()函数启动整个IOC容器对Bean定义的载入过程：

```
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // 调用容器准备刷新的方法，获取容器的当时时间，同时给容器设置同步标识
        prepareRefresh();
        // 告诉子类启动refreshBeanFactory()方法，Bean定义资源文件的载入从子类的refreshBeanFactory()方法启动
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        // 为BeanFactory配置容器特性，例如类加载器、事件处理器等
        prepareBeanFactory(beanFactory);
        try {
            // 为容器的某些子类指定特殊的BeanPost事件处理器
            postProcessBeanFactory(beanFactory);
            // 调用所有注册的BeanFactoryPostProcessor的Bean
            invokeBeanFactoryPostProcessors(beanFactory);
            // 为BeanFactory注册BeanPost事件处理器. BeanPostProcessor是Bean后置处理器，用于监听容器触发的事件
            registerBeanPostProcessors(beanFactory);
            // 初始化信息源，和国际化相关.
            initMessageSource();
            // 初始化容器事件传播器.
            initApplicationEventMulticaster();
            // 调用子类的某些特殊Bean初始化方法
            onRefresh();
            // 为事件传播器注册事件监听器.
            registerListeners();
            // 初始化所有剩余的单例Bean
            finishBeanFactoryInitialization(beanFactory);
            // 初始化容器的生命周期事件处理器，并发布容器的生命周期事件
            finishRefresh();
        }
        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                        "cancelling refresh attempt: " + ex);
            }
            // 销毁已创建的Bean
            destroyBeans();
            // 取消refresh操作，重置容器的同步标识.
            cancelRefresh(ex);
            throw ex;
        }
        finally {
            resetCommonCaches();
        }
    }
}
```

refresh()方法主要为IOC容器Bean的生命周期管理提供条件，Spring IOC容器载入Bean定义资源文件从其子类容器的refreshBeanFactory()方法启动，
所以整个refresh()中“ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();”这句以后代码的都是注册容器的信息源和生命周期事件，载入过程就是从这句代码启动。 

refresh()方法的作用是：在创建IOC容器前，如果已经有容器存在，则需要把已有的容器销毁和关闭，以保证在refresh之后使用的是新建立起来的IOC 容器。
refresh的作用类似于对IOC容器的重启，在新建立好的容器中对容器进行初始化，对Bean定义资源进行载入。

(3)AbstractApplicationContext的obtainFreshBeanFactory()方法

其调用了子类容器的refreshBeanFactory()方法，启动容器载入Bean定义资源文件的过程，代码如下：

```
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    // 使用了委派设计模式，父类定义了抽象的refreshBeanFactory()方法，具体实现调用子类容器的refreshBeanFactory()方法
    refreshBeanFactory();
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    if (logger.isDebugEnabled()) {
        logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
    }
    return beanFactory;
}
```

AbstractApplicationContext类中只抽象定义了refreshBeanFactory()方法，容器真正调用的是其子类AbstractRefreshableApplicationContext 
实现的refreshBeanFactory()方法，源码如下：

```
@Override
protected final void refreshBeanFactory() throws BeansException {
    // 如果已经有容器，销毁容器中的bean，关闭容器
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    try {
        // 创建IOC容器
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        beanFactory.setSerializationId(getId());
        // 对IOC容器进行定制化，如设置启动参数，开启注解的自动装配等
        customizeBeanFactory(beanFactory);
        // 调用载入Bean定义的方法，这里又使用了一个委派模式，在当前类中只定义了抽象的loadBeanDefinitions方法，具体的实现调用子类容器
        loadBeanDefinitions(beanFactory);
        synchronized (this.beanFactoryMonitor) {
            this.beanFactory = beanFactory;
        }
    }
    catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
    }
}
```

AbstractRefreshableApplicationContext中只定义了抽象的loadBeanDefinitions方法，容器真正调用的是其子类AbstractXmlApplicationContext对该方法的实现，

(4)AbstractXmlApplicationContext类的loadBeanDefinitions方法

AbstractXmlApplicationContext的主要源码如下：

```
public abstract class AbstractXmlApplicationContext extends AbstractRefreshableConfigApplicationContext {
    // 实现父类抽象的载入Bean定义方法
    @Override
    protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
        // 创建XmlBeanDefinitionReader，即创建Bean读取器，并通过回调设置到容器中去，容器使用该读取器读取Bean定义资源
        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
        // 为Bean读取器设置Spring资源加载器，AbstractXmlApplicationContext的祖先父类AbstractApplicationContext继承DefaultResourceLoader，因此，容器本身也是一个资源加载器
        beanDefinitionReader.setEnvironment(this.getEnvironment());
        beanDefinitionReader.setResourceLoader(this);
        // 为Bean读取器设置SAX xml解析器
        beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
        // 当Bean读取器读取Bean定义的Xml资源文件时，启用Xml的校验机制
        initBeanDefinitionReader(beanDefinitionReader);
        // Bean读取器真正实现加载的方法
        loadBeanDefinitions(beanDefinitionReader);
    }
    
    protected void initBeanDefinitionReader(XmlBeanDefinitionReader reader) {
        reader.setValidating(this.validating);
    }
    
    // Xml Bean读取器加载Bean定义资源
    protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
        // 获取Bean定义资源的定位
        Resource[] configResources = getConfigResources();
        if (configResources != null) {
            // Xml Bean读取器调用其父类AbstractBeanDefinitionReader读取定位的Bean定义资源
            reader.loadBeanDefinitions(configResources);
        }
        // 如果子类中获取的Bean定义资源定位为空，则获取FileSystemXmlApplicationContext构造方法中setConfigLocations方法设置的资源
        String[] configLocations = getConfigLocations();
        if (configLocations != null) {
            // Xml Bean读取器调用其父类AbstractBeanDefinitionReader读取定位的Bean定义资源
            reader.loadBeanDefinitions(configLocations);
        }
    }
    
    // 这里又使用了一个委托模式，调用子类的获取Bean定义资源定位的方法，该方法在ClassPathXmlApplicationContext中进行实现
    @Nullable
    protected Resource[] getConfigResources() {
        return null;
    }
}
```

Xml Bean读取器(XmlBeanDefinitionReader)调用其父类AbstractBeanDefinitionReader的reader.loadBeanDefinitions方法读取Bean定义资源。 
由于我们使用FileSystemXmlApplicationContext作为例子分析，因此getConfigResources的返回值为null，程序执行reader.loadBeanDefinitions(configLocations)分支。

IOC容器初始化内容较多，分了几篇来写，此为第一篇，欢迎关注其余内容，感谢！









