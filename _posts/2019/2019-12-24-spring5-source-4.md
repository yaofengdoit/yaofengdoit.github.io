---
layout: post
title: spring5源码分析系列（四）——IOC容器的初始化（二）
category: Spring
tags: [Spring]
no-post-nav: true
---

前言：上一篇讲到了Xml Bean读取器(XmlBeanDefinitionReader)调用其父类AbstractBeanDefinitionReader的reader.loadBeanDefinitions方法读取Bean定义资源，此篇我们继续后面的内容。

(5)AbstractBeanDefinitionReader的loadBeanDefinitions方法

方法源码如下：

```
//重载方法，调用下面的loadBeanDefinitions(String, Set<Resource>)方法
@Override
public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {
    return loadBeanDefinitions(location, null);
}

public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
    //获取在IoC容器初始化过程中设置的资源加载器
    ResourceLoader resourceLoader = getResourceLoader();
    if (resourceLoader == null) {
        throw new BeanDefinitionStoreException(
                "Cannot import bean definitions from location [" + location + "]: no ResourceLoader available");
    }

    if (resourceLoader instanceof ResourcePatternResolver) {
        // Resource pattern matching available.
        try {
            //将指定位置的Bean定义资源文件解析为Spring IOC容器封装的资源
            //加载多个指定位置的Bean定义资源文件
            Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
            //委派调用其子类XmlBeanDefinitionReader的方法，实现加载功能
            int loadCount = loadBeanDefinitions(resources);
            if (actualResources != null) {
                for (Resource resource : resources) {
                    actualResources.add(resource);
                }
            }
            if (logger.isDebugEnabled()) {
                logger.debug("Loaded " + loadCount + " bean definitions from location pattern [" + location + "]");
            }
            return loadCount;
        }
        catch (IOException ex) {
            throw new BeanDefinitionStoreException(
                    "Could not resolve bean definition resource pattern [" + location + "]", ex);
        }
    }
    else {
        // Can only load single resources by absolute URL.
        //将指定位置的Bean定义资源文件解析为Spring IOC容器封装的资源
        //加载单个指定位置的Bean定义资源文件
        Resource resource = resourceLoader.getResource(location);
        //委派调用其子类XmlBeanDefinitionReader的方法，实现加载功能
        int loadCount = loadBeanDefinitions(resource);
        if (actualResources != null) {
            actualResources.add(resource);
        }
        if (logger.isDebugEnabled()) {
            logger.debug("Loaded " + loadCount + " bean definitions from location [" + location + "]");
        }
        return loadCount;
    }
}

//重载方法，调用loadBeanDefinitions(String);
@Override
public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
    Assert.notNull(locations, "Location array must not be null");
    int counter = 0;
    for (String location : locations) {
        counter += loadBeanDefinitions(location);
    }
    return counter;
}
```
loadBeanDefinitions(Resource...resources)样也是调用XmlBeanDefinitionReader的loadBeanDefinitions方法。

对AbstractBeanDefinitionReader的loadBeanDefinitions方法源码分析可以看出该方法：
首先，调用资源加载器的获取资源方法resourceLoader.getResource(location)，获取到要加载的资源；
其次，真正执行加载功能是其子类XmlBeanDefinitionReader的loadBeanDefinitions方法。

(6)资源加载器获取要读入的资源

AbstractBeanDefinitionReader通过调用其父类DefaultResourceLoader的getResource方法获取要加载的资源，源码如下：

```
//获取Resource的具体实现方法
@Override
public Resource getResource(String location) {
    Assert.notNull(location, "Location must not be null");

    for (ProtocolResolver protocolResolver : this.protocolResolvers) {
        Resource resource = protocolResolver.resolve(location, this);
        if (resource != null) {
            return resource;
        }
    }
    //如果是类路径的方式，需要使用ClassPathResource来得到bean文件的资源对象
    if (location.startsWith("/")) {
        return getResourceByPath(location);
    }
    else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
        return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
    }
    else {
        try {
            // Try to parse the location as a URL...
            // 如果是URL方式，使用UrlResource作为bean文件的资源对象
            URL url = new URL(location);
            return (ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
        }
        catch (MalformedURLException ex) {
            // No URL -> resolve as resource path.
            //如果既不是classpath标识，又不是URL标识的Resource定位，则调用容器本身的getResourceByPath方法获取Resource
            return getResourceByPath(location);
        }
    }
}
```
FileSystemXmlApplicationContext容器提供了getResourceByPath方法的实现，就是为了处理既不是classpath 标识，又不是URL标识的Resource定位这种情况。
```
@Override
protected Resource getResourceByPath(String path) {
    if (path.startsWith("/")) {
        path = path.substring(1);
    }
    //这里使用文件系统资源对象来定义bean文件
    return new FileSystemResource(path);
}
```

这样代码就回到了FileSystemXmlApplicationContext中来，他提供了FileSystemResource来完成从文件系统得到配置文件的资源定义。 
这样就可以从文件系统路径上对IOC配置文件进行加载，当然也可以按照这个逻辑从任何地方加载，在Spring中可以看到它提供的各种资源抽象，
比如ClassPathResource,URLResource,FileSystemResource等来供我们使用。上面是定位Resource的一个过程，这只是加载过程的一部分.

(7)XmlBeanDefinitionReader加载Bean定义资源

回到XmlBeanDefinitionReader的loadBeanDefinitions(Resource …)方法看到代表bean文件的资源定义以后的载入过程。
```
//XmlBeanDefinitionReader加载资源的入口方法
@Override
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
    //将读入的XML资源进行特殊编码处理
    return loadBeanDefinitions(new EncodedResource(resource));
}

//这里是载入XML形式Bean定义资源文件方法
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
    Assert.notNull(encodedResource, "EncodedResource must not be null");
    if (logger.isInfoEnabled()) {
        logger.info("Loading XML bean definitions from " + encodedResource.getResource());
    }

    Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
    if (currentResources == null) {
        currentResources = new HashSet<>(4);
        this.resourcesCurrentlyBeingLoaded.set(currentResources);
    }
    if (!currentResources.add(encodedResource)) {
        throw new BeanDefinitionStoreException(
                "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
    }
    try {
        //将资源文件转为InputStream的IO流
        InputStream inputStream = encodedResource.getResource().getInputStream();
        try {
            //从InputStream中得到XML的解析源
            InputSource inputSource = new InputSource(inputStream);
            if (encodedResource.getEncoding() != null) {
                inputSource.setEncoding(encodedResource.getEncoding());
            }
            //这里是具体的读取过程
            return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
        }
        finally {
            //关闭从Resource中得到的IO流
            inputStream.close();
        }
    }
    catch (IOException ex) {
        throw new BeanDefinitionStoreException(
                "IOException parsing XML document from " + encodedResource.getResource(), ex);
    }
    finally {
        currentResources.remove(encodedResource);
        if (currentResources.isEmpty()) {
            this.resourcesCurrentlyBeingLoaded.remove();
        }
    }
}

//从特定XML文件中实际载入Bean定义资源的方法
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource) throws BeanDefinitionStoreException {
    try {
        //将XML文件转换为DOM对象，解析过程由documentLoader实现
        Document doc = doLoadDocument(inputSource, resource);
        //这里是启动对Bean定义解析的详细过程，该解析过程会用到Spring的Bean配置规则
        return registerBeanDefinitions(doc, resource);
    }
    ...
}
```
通过源码分析，载入Bean定义资源文件的最后一步是将Bean定义资源转换为Document对象，该过程由documentLoader实现。

(8)DocumentLoader将Bean定义资源转换为Document对象

源码如下：
```
//使用标准的JAXP将载入的Bean定义资源转换成document对象
@Override
public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
        ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {

    //创建文件解析器工厂
    DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
    if (logger.isDebugEnabled()) {
        logger.debug("Using JAXP provider [" + factory.getClass().getName() + "]");
    }
    //创建文档解析器
    DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
    //解析Spring的Bean定义资源
    return builder.parse(inputSource);
}

protected DocumentBuilderFactory createDocumentBuilderFactory(int validationMode, boolean namespaceAware)
        throws ParserConfigurationException {

    //创建文档解析工厂
    DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
    factory.setNamespaceAware(namespaceAware);

    //设置解析XML的校验
    if (validationMode != XmlValidationModeDetector.VALIDATION_NONE) {
        factory.setValidating(true);
        if (validationMode == XmlValidationModeDetector.VALIDATION_XSD) {
            // Enforce namespace aware for XSD...
            factory.setNamespaceAware(true);
            try {
                factory.setAttribute(SCHEMA_LANGUAGE_ATTRIBUTE, XSD_SCHEMA_LANGUAGE);
            }
            catch (IllegalArgumentException ex) {
                ParserConfigurationException pcex = new ParserConfigurationException(
                        "Unable to validate using XSD: Your JAXP provider [" + factory +
                        "] does not support XML Schema. Are you running on Java 1.4 with Apache Crimson? " +
                        "Upgrade to Apache Xerces (or Java 1.5) for full XSD support.");
                pcex.initCause(ex);
                throw pcex;
            }
        }
    }
    return factory;
}
```
该解析过程调用JavaEE标准的JAXP标准进行处理。至此Spring IOC容器根据定位的Bean定义资源文件，将其加载读入并转换成为Document对象过程完成。
接下来继续分析Spring IOC容器将载入的Bean定义资源文件转换为Document对象之后，是如何将其解析为Spring IOC管理的Bean对象并将其注册到容器中的。

IOC容器初始化内容较多，分了几篇来写，此为第二篇，欢迎关注其余内容，感谢！



