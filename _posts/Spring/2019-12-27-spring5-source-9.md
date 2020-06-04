---
layout: post
title: spring5源码分析系列（九）——基于Annotation的依赖注入
category: Spring
tags: [Spring]
---

&ensp;&ensp;&ensp;&ensp;前言：从Spring2.0以后的版本开始，Spring引入了基于注解(Annotation)方式的配置，注解(Annotation)是JDK1.5中引入的一个新特性，
用于简化Bean的配置，某些场合可以取代XML配置文件。注解可以大大简化配置，提高开发速度，但不能完全取代XML配置方式。XML方式更加灵活，并且发展的相对成熟，
这种配置方式为大多数Spring开发者熟悉；注解方式使用起来非常简洁，尚处于发展阶段，XML配置文件和注解(Annotation)可以相互配合使用。

&ensp;&ensp;&ensp;&ensp;Spring IOC容器对于类级别的注解和类内部的注解有两种处理策略： 
<br/>
(1)类级别的注解：如@Component、@Repository、@Controller、@Service以及JavaEE6的@ManagedBean和@Named注解，都是添加在类上面的类级别注解，
Spring容器根据注解的过滤规则扫描读取注解Bean定义类，并将其注册到Spring IOC容器中；
<br/>
(2)类内部的注解：如@Autowire、@Value、@Resource以及EJB和WebService相关的注解等，都是添加在类内部的字段或者方法上的类内部注解，
SpringIOC容器通过Bean后置注解处理器解析Bean内部的注解。

&ensp;&ensp;&ensp;&ensp;接下来将根据这两种处理策略，分析Spring处理注解相关的源码。

### 1.AnnotationConfigApplicationContext对注解Bean初始化

&ensp;&ensp;&ensp;&ensp;Spring中，管理注解Bean定义的容器有两个：AnnotationConfigApplicationContext和AnnotationConfigWebApplicationContex。
这两个类是专门处理Spring注解方式配置的容器，直接依赖于注解作为容器配置信息来源的IOC容器。
AnnotationConfigWebApplicationContext是AnnotationConfigApplicationContext的web版本，两者的用法以及对注解的处理方式几乎没有什么差别。
AnnotationConfigApplicationContext源码如下：

```
public class AnnotationConfigApplicationContext extends GenericApplicationContext implements AnnotationConfigRegistry {
    //保存一个读取注解Bean定义的读取器，并将其设置到容器中
	private final AnnotatedBeanDefinitionReader reader;
	//保存一个扫描指定类路径中注解Bean定义的扫描器，并将其设置到容器中
	private final ClassPathBeanDefinitionScanner scanner;
	
	//默认构造函数，初始化一个空容器，容器不包含任何 Bean 信息，需要在稍后通过调用其register()方法注册配置类，并调用refresh()方法刷新容器，触发容器对注解Bean的载入、解析和注册过程
    public AnnotationConfigApplicationContext() {
        this.reader = new AnnotatedBeanDefinitionReader(this);
        this.scanner = new ClassPathBeanDefinitionScanner(this);
    }

	public AnnotationConfigApplicationContext(DefaultListableBeanFactory beanFactory) {
		super(beanFactory);
		this.reader = new AnnotatedBeanDefinitionReader(this);
		this.scanner = new ClassPathBeanDefinitionScanner(this);
	}    
    
	//最常用的构造函数，通过将涉及到的配置类传递给该构造函数，以实现将相应配置类中的Bean自动注册到容器中
	public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
		this();
		register(annotatedClasses);
		refresh();
	}

	//该构造函数会自动扫描以给定的包及其子包下的所有类，并自动识别所有的Spring Bean，将其注册到容器中
	public AnnotationConfigApplicationContext(String... basePackages) {
		this();
		scan(basePackages);
		refresh();
	}
	
	@Override
	public void setEnvironment(ConfigurableEnvironment environment) {
		super.setEnvironment(environment);
		this.reader.setEnvironment(environment);
		this.scanner.setEnvironment(environment);
	}
	
	//为容器的注解Bean读取器和注解Bean扫描器设置Bean名称产生器
	public void setBeanNameGenerator(BeanNameGenerator beanNameGenerator) {
		this.reader.setBeanNameGenerator(beanNameGenerator);
		this.scanner.setBeanNameGenerator(beanNameGenerator);
		getBeanFactory().registerSingleton(
				AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR, beanNameGenerator);
	}
	
	//为容器的注解Bean读取器和注解Bean扫描器设置作用范围元信息解析器
	public void setScopeMetadataResolver(ScopeMetadataResolver scopeMetadataResolver) {
		this.reader.setScopeMetadataResolver(scopeMetadataResolver);
		this.scanner.setScopeMetadataResolver(scopeMetadataResolver);
	}

	//为容器注册一个要被处理的注解Bean，新注册的Bean，必须手动调用容器的refresh()方法刷新容器，触发容器对新注册的Bean的处理
	public void register(Class<?>... annotatedClasses) {
		Assert.notEmpty(annotatedClasses, "At least one annotated class must be specified");
		this.reader.register(annotatedClasses);
	}
	
	//扫描指定包路径及其子包下的注解类，为了使新添加的类被处理，必须手动调用refresh()方法刷新容器
	public void scan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		this.scanner.scan(basePackages);
	}
    ...
}
```

分析可知Spring对注解的处理分为两种方式：
<br/>
(1)直接将注解Bean注册到容器中：可以在初始化容器时注册，也可以在容器创建之后手动调用注册方法向容器注册，然后通过手动刷新容器，使得容器对注册的注解Bean进行处理；
<br/> 
(2)通过扫描指定的包及其子包下的所有类：在初始化注解容器时指定要自动扫描的路径，如果容器创建以后向给定路径动态添加了注解Bean，则需要手动调用容器扫描的方法，
然后手动刷新容器，使得容器对所注册的Bean进行处理。

### 2.AnnotationConfigApplicationContext注册注解Bean

&ensp;&ensp;&ensp;&ensp;当创建注解处理容器时，如果传入的初始参数是具体的注解Bean定义类时，注解容器读取并注册。
AnnotationConfigApplicationContext通过调用注解Bean定义读取器AnnotatedBeanDefinitionReader的register方法向容器注册指定的注解Bean，注解Bean定义读取器向容器注册注解Bean的源码如下：

```
//注册多个注解Bean定义类
public void register(Class<?>... annotatedClasses) {
    for (Class<?> annotatedClass : annotatedClasses) {
        registerBean(annotatedClass);
    }
}

//注册一个注解Bean定义类
public void registerBean(Class<?> annotatedClass) {
    doRegisterBean(annotatedClass, null, null, null);
}

//Bean定义读取器向容器注册注解Bean定义类
@SuppressWarnings("unchecked")
public void registerBean(Class<?> annotatedClass, String name, Class<? extends Annotation>... qualifiers) {
    doRegisterBean(annotatedClass, null, name, qualifiers);
}

//Bean定义读取器向容器注册注解Bean定义类
<T> void doRegisterBean(Class<T> annotatedClass, @Nullable Supplier<T> instanceSupplier, @Nullable String name,
        @Nullable Class<? extends Annotation>[] qualifiers, BeanDefinitionCustomizer... definitionCustomizers) {

    //根据指定的注解Bean定义类，创建Spring容器中对注解Bean的封装的数据结构
    AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);
    if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
        return;
    }

    abd.setInstanceSupplier(instanceSupplier);
    //解析注解Bean定义的作用域，若@Scope("prototype")，则Bean为原型类型；若@Scope("singleton")，则Bean为单态类型
    ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
    //为注解Bean定义设置作用域
    abd.setScope(scopeMetadata.getScopeName());
    //为注解Bean定义生成Bean名称
    String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));

    //处理注解Bean定义中的通用注解
    AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
    //如果在向容器注册注解Bean定义时，使用了额外的限定符注解，则解析限定符注解。
    //主要是配置的关于autowiring自动依赖注入装配的限定条件，即@Qualifier注解
    //Spring自动依赖注入装配默认是按类型装配，如果使用@Qualifier则按名称
    if (qualifiers != null) {
        for (Class<? extends Annotation> qualifier : qualifiers) {
            //如果配置了@Primary注解，设置该Bean为autowiring自动依赖注入装配时的首选
            if (Primary.class == qualifier) {
                abd.setPrimary(true);
            }
            //如果配置了@Lazy注解，则设置该Bean为延迟初始化，如果没有配置，则该Bean为预实例化
            else if (Lazy.class == qualifier) {
                abd.setLazyInit(true);
            }
            //如果使用了除@Primary和@Lazy以外的其他注解，则为该Bean添加一个autowiring自动依赖注入装配限定符，该Bean在进autowiring自动依赖注入装配时，根据名称装配限定符指定的Bean
            else {
                abd.addQualifier(new AutowireCandidateQualifier(qualifier));
            }
        }
    }
    for (BeanDefinitionCustomizer customizer : definitionCustomizers) {
        customizer.customize(abd);
    }

    //创建一个指定Bean名称的Bean定义对象，封装注解Bean定义类数据
    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
    //根据注解Bean定义类中配置的作用域，创建相应的代理对象
    definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
    //向IOC容器注册注解Bean类定义对象
    BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
```

从源码可看出，注册注解Bean定义类的基本步骤：
<br/>
a.需要使用注解元数据解析器解析注解Bean中关于作用域的配置；
<br/>
b.使用AnnotationConfigUtils的processCommonDefinitionAnnotations方法处理注解Bean定义类中通用的注解；
<br/>
c.使用AnnotationConfigUtils的applyScopedProxyMode方法创建对于作用域的代理对象；
d.通过BeanDefinitionReaderUtils向容器注册Bean。 

(1)AnnotationScopeMetadataResolver解析作用域元数据

AnnotationScopeMetadataResolver通过resolveScopeMetadata方法解析注解Bean定义类的作用域元信息，即判断注册的Bean是原生类型(prototype)还是单态(singleton)类型，源码如下：

```
//解析注解Bean定义类中的作用域元信息
@Override
public ScopeMetadata resolveScopeMetadata(BeanDefinition definition) {
    ScopeMetadata metadata = new ScopeMetadata();
    if (definition instanceof AnnotatedBeanDefinition) {
        AnnotatedBeanDefinition annDef = (AnnotatedBeanDefinition) definition;
        //从注解Bean定义类的属性中查找属性为”Scope”的值，即@Scope注解的值
        //annDef.getMetadata().getAnnotationAttributes方法将Bean中所有的注解和注解的值存放在一个map集合中
        AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(
                annDef.getMetadata(), this.scopeAnnotationType);
        //将获取到的@Scope注解的值设置到要返回的对象中
        if (attributes != null) {
            metadata.setScopeName(attributes.getString("value"));
            //获取@Scope注解中的proxyMode属性值，在创建代理对象时会用到
            ScopedProxyMode proxyMode = attributes.getEnum("proxyMode");
            //如果@Scope的proxyMode属性为DEFAULT或者NO
            if (proxyMode == ScopedProxyMode.DEFAULT) {
                //设置proxyMode为NO
                proxyMode = this.defaultProxyMode;
            }
            //为返回的元数据设置proxyMode
            metadata.setScopedProxyMode(proxyMode);
        }
    }
    //返回解析的作用域元信息对象
    return metadata;
}
```

代码中的annDef.getMetadata().getAnnotationAttributes方法就是获取对象中指定类型的注解的值。

(2)AnnotationConfigUtils处理注解Bean定义类中的通用注解

AnnotationConfigUtils类的processCommonDefinitionAnnotations在向容器注册Bean之前，首先对注解Bean定义类中的通用Spring注解进行处理，源码如下：

```
//处理Bean定义中通用注解
static void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd, AnnotatedTypeMetadata metadata) {
    AnnotationAttributes lazy = attributesFor(metadata, Lazy.class);
    //如果Bean定义中有@Lazy注解，则将该Bean预实例化属性设置为@lazy注解的值
    if (lazy != null) {
        abd.setLazyInit(lazy.getBoolean("value"));
    }

    else if (abd.getMetadata() != metadata) {
        lazy = attributesFor(abd.getMetadata(), Lazy.class);
        if (lazy != null) {
            abd.setLazyInit(lazy.getBoolean("value"));
        }
    }
    //如果Bean定义中有@Primary注解，则为该Bean设置为autowiring自动依赖注入装配的首选对象
    if (metadata.isAnnotated(Primary.class.getName())) {
        abd.setPrimary(true);
    }
    //如果Bean定义中有@DependsOn注解，则为该Bean设置所依赖的Bean名称，容器将确保在实例化该Bean之前首先实例化所依赖的Bean
    AnnotationAttributes dependsOn = attributesFor(metadata, DependsOn.class);
    if (dependsOn != null) {
        abd.setDependsOn(dependsOn.getStringArray("value"));
    }

    if (abd instanceof AbstractBeanDefinition) {
        AbstractBeanDefinition absBd = (AbstractBeanDefinition) abd;
        AnnotationAttributes role = attributesFor(metadata, Role.class);
        if (role != null) {
            absBd.setRole(role.getNumber("value").intValue());
        }
        AnnotationAttributes description = attributesFor(metadata, Description.class);
        if (description != null) {
            absBd.setDescription(description.getString("value"));
        }
    }
}
```

(3)AnnotationConfigUtils代理策略

AnnotationConfigUtils类的applyScopedProxyMode方法根据注解Bean定义类中配置的作用域@Scope注解的值，为Bean定义应用相应的代理模式，主要在Spring面向切面编程(AOP)中使用，源码如下：

```
//根据作用域为Bean应用引用的代码模式
static BeanDefinitionHolder applyScopedProxyMode(
        ScopeMetadata metadata, BeanDefinitionHolder definition, BeanDefinitionRegistry registry) {

    //获取注解Bean定义类中@Scope注解的proxyMode属性值
    ScopedProxyMode scopedProxyMode = metadata.getScopedProxyMode();
    //如果配置的@Scope注解的proxyMode属性值为NO，则不应用代理模式
    if (scopedProxyMode.equals(ScopedProxyMode.NO)) {
        return definition;
    }
    //获取配置的@Scope注解的proxyMode属性值，如果为TARGET_CLASS，则返回true，如果为INTERFACES，则返回false
    boolean proxyTargetClass = scopedProxyMode.equals(ScopedProxyMode.TARGET_CLASS);
    //为注册的Bean创建相应模式的代理对象
    return ScopedProxyCreator.createScopedProxy(definition, registry, proxyTargetClass);
}
```

### 3.AnnotationConfigApplicationContext扫描指定包及其子包下的注解Bean

&ensp;&ensp;&ensp;&ensp;当创建注解处理容器时，如果传入的初始参数是注解Bean定义类所在的包时，注解容器将扫描给定的包及其子包，将扫描到的注解Bean定义载入并注册。

(1)ClassPathBeanDefinitionScanner扫描给定的包及其子包

AnnotationConfigApplicationContext通过调用类路径Bean定义扫描器ClassPathBeanDefinitionScanner扫描给定包及其子包下的所有类，主要源码如下：

```
public class ClassPathBeanDefinitionScanner extends ClassPathScanningCandidateComponentProvider {
    //创建一个类路径Bean定义扫描器
    public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry) {
        this(registry, true);
    }
    
	//为容器创建一个类路径Bean定义扫描器，并指定是否使用默认的扫描过滤规则。
	//即Spring默认扫描配置：@Component、@Repository、@Service、@Controller注解的Bean，同时也支持JavaEE6的@ManagedBean和JSR-330的@Named注解
	public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters) {
		this(registry, useDefaultFilters, getOrCreateEnvironment(registry));
	}
	
	public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
			Environment environment, @Nullable ResourceLoader resourceLoader) {
		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
		//为容器设置加载Bean定义的注册器
		this.registry = registry;
		if (useDefaultFilters) {
			registerDefaultFilters();
		}
		setEnvironment(environment);
		//为容器设置资源加载器
		setResourceLoader(resourceLoader);
	}
	
	//调用类路径Bean定义扫描器入口方法
	public int scan(String... basePackages) {
		//获取容器中已经注册的Bean个数
		int beanCountAtScanStart = this.registry.getBeanDefinitionCount();
		//启动扫描器扫描给定包
		doScan(basePackages);
		//注册注解配置(Annotation config)处理器
		if (this.includeAnnotationConfig) {
			AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
		}
		//返回注册的Bean个数
		return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
	}		  
	
	//类路径Bean定义扫描器扫描给定包及其子包
	protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		//创建一个集合，存放扫描到Bean定义的封装类
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
		//遍历扫描所有给定的包
		for (String basePackage : basePackages) {
			//调用父类ClassPathScanningCandidateComponentProvider的方法扫描给定类路径，获取符合条件的Bean定义
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
			//遍历扫描到的Bean
			for (BeanDefinition candidate : candidates) {
				//获取Bean定义类中@Scope注解的值，即获取Bean的作用域
				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
				//为Bean设置注解配置的作用域
				candidate.setScope(scopeMetadata.getScopeName());
				//为Bean生成名称
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
				//如果扫描到的Bean不是Spring的注解Bean，则为Bean设置默认值，设置Bean的自动依赖注入装配属性等
				if (candidate instanceof AbstractBeanDefinition) {
					postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
				}
				//如果扫描到的Bean是Spring的注解Bean，则处理其通用的Spring注解
				if (candidate instanceof AnnotatedBeanDefinition) {
					//处理注解Bean中通用的注解，在分析注解Bean定义类读取器时已经分析过
					AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
				}
				//根据Bean名称检查指定的Bean是否需要在容器中注册，或者在容器中冲突
				if (checkCandidate(beanName, candidate)) {
					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
					//根据注解中配置的作用域，为Bean应用相应的代理模式
					definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
					beanDefinitions.add(definitionHolder);
					//向容器注册扫描到的Bean
					registerBeanDefinition(definitionHolder, this.registry);
				}
			}
		}
		return beanDefinitions;
	}
}
```

类路径Bean定义扫描器ClassPathBeanDefinitionScanner主要通过findCandidateComponents方法调用其父类
ClassPathScanningCandidateComponentProvider类来扫描获取给定包及其子包下的类。

(2)ClassPathScanningCandidateComponentProvider扫描给定包及其子包的类

ClassPathScanningCandidateComponentProvider类的findCandidateComponents方法具体实现扫描给定类路径包的功能，主要源码如下：

```
public class ClassPathScanningCandidateComponentProvider implements EnvironmentCapable, ResourceLoaderAware {
    //保存过滤规则要包含的注解，即Spring默认的@Component、@Repository、@Service、@Controller注解的Bean，以及JavaEE6的@ManagedBean和JSR-330的@Named注解
    private final List<TypeFilter> includeFilters = new LinkedList<>();

    //保存过滤规则要排除的注解
    private final List<TypeFilter> excludeFilters = new LinkedList<>();
    
	//构造方法，该方法在子类ClassPathBeanDefinitionScanner的构造方法中被调用
	public ClassPathScanningCandidateComponentProvider(boolean useDefaultFilters) {
		this(useDefaultFilters, new StandardEnvironment());
	}
	
	public ClassPathScanningCandidateComponentProvider(boolean useDefaultFilters, Environment environment) {
		//如果使用Spring默认的过滤规则，则向容器注册过滤规则
		if (useDefaultFilters) {
			registerDefaultFilters();
		}
		setEnvironment(environment);
		setResourceLoader(null);
	}
	
	//向容器注册过滤规则
	@SuppressWarnings("unchecked")
	protected void registerDefaultFilters() {
		//向要包含的过滤规则中添加@Component注解类，注意Spring中@Repository、@Service和@Controller都是Component，因为这些注解都添加了@Component注解
		this.includeFilters.add(new AnnotationTypeFilter(Component.class));
		//获取当前类的类加载器
		ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
		try {
			//向要包含的过滤规则添加JavaEE6的@ManagedBean注解
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
			logger.debug("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
		}
		try {
			//向要包含的过滤规则添加@Named注解
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
			logger.debug("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
		}
	}	
	    
    //扫描给定类路径的包
    public Set<BeanDefinition> findCandidateComponents(String basePackage) {
        if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
            return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
        }
        else {
            return scanCandidateComponents(basePackage);
        }
    }
    
    private Set<BeanDefinition> addCandidateComponentsFromIndex(CandidateComponentsIndex index, String basePackage) {
		//创建存储扫描到的类的集合
		Set<BeanDefinition> candidates = new LinkedHashSet<>();
		try {
			Set<String> types = new HashSet<>();
			for (TypeFilter filter : this.includeFilters) {
				String stereotype = extractStereotype(filter);
				if (stereotype == null) {
					throw new IllegalArgumentException("Failed to extract stereotype from "+ filter);
				}
				types.addAll(index.getCandidateTypes(basePackage, stereotype));
			}
			boolean traceEnabled = logger.isTraceEnabled();
			boolean debugEnabled = logger.isDebugEnabled();
			for (String type : types) {
				//为指定资源获取元数据读取器，元信息读取器通过汇编(ASM)读取资源元信息
				MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(type);
				//如果扫描到的类符合容器配置的过滤规则
				if (isCandidateComponent(metadataReader)) {
					//通过汇编(ASM)读取资源字节码中的Bean定义元信息
					AnnotatedGenericBeanDefinition sbd = new AnnotatedGenericBeanDefinition(
							metadataReader.getAnnotationMetadata());
					if (isCandidateComponent(sbd)) {
						if (debugEnabled) {
							logger.debug("Using candidate component class from index: " + type);
						}
						candidates.add(sbd);
					}
					else {
						if (debugEnabled) {
							logger.debug("Ignored because not a concrete top-level class: " + type);
						}
					}
				}
				else {
					if (traceEnabled) {
						logger.trace("Ignored because matching an exclude filter: " + type);
					}
				}
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
		}
		return candidates;
	}    
    
	//判断元信息读取器读取的类是否符合容器定义的注解过滤规则
	protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
		//如果读取的类的注解在排除注解过滤规则中，返回false
		for (TypeFilter tf : this.excludeFilters) {
			if (tf.match(metadataReader, getMetadataReaderFactory())) {
				return false;
			}
		}
		//如果读取的类的注解在包含的注解的过滤规则中，则返回ture
		for (TypeFilter tf : this.includeFilters) {
			if (tf.match(metadataReader, getMetadataReaderFactory())) {
				return isConditionMatch(metadataReader);
			}
		}
		//如果读取的类的注解既不在排除规则，也不在包含规则中，则返回false
		return false;
	}    
}
```

### 4.AnnotationConfigWebApplicationContext载入注解Bean定义

&ensp;&ensp;&ensp;&ensp;AnnotationConfigWebApplicationContext是AnnotationConfigApplicationContext的Web版，它们对于注解Bean的注册和扫描基本相同，
但是AnnotationConfigWebApplicationContext对注解Bean定义的载入稍有不同，AnnotationConfigWebApplicationContext注入注解Bean定义源码如下：

```
//载入注解Bean定义资源
@Override
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) {
    //为容器设置注解Bean定义读取器
    AnnotatedBeanDefinitionReader reader = getAnnotatedBeanDefinitionReader(beanFactory);
    //为容器设置类路径Bean定义扫描器
    ClassPathBeanDefinitionScanner scanner = getClassPathBeanDefinitionScanner(beanFactory);

    //获取容器的Bean名称生成器
    BeanNameGenerator beanNameGenerator = getBeanNameGenerator();
    //为注解Bean定义读取器和类路径扫描器设置Bean名称生成器
    if (beanNameGenerator != null) {
        reader.setBeanNameGenerator(beanNameGenerator);
        scanner.setBeanNameGenerator(beanNameGenerator);
        beanFactory.registerSingleton(AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR, beanNameGenerator);
    }

    //获取容器的作用域元信息解析器
    ScopeMetadataResolver scopeMetadataResolver = getScopeMetadataResolver();
    //为注解Bean定义读取器和类路径扫描器设置作用域元信息解析器
    if (scopeMetadataResolver != null) {
        reader.setScopeMetadataResolver(scopeMetadataResolver);
        scanner.setScopeMetadataResolver(scopeMetadataResolver);
    }

    if (!this.annotatedClasses.isEmpty()) {
        if (logger.isInfoEnabled()) {
            logger.info("Registering annotated classes: [" +
                    StringUtils.collectionToCommaDelimitedString(this.annotatedClasses) + "]");
        }
        reader.register(this.annotatedClasses.toArray(new Class<?>[this.annotatedClasses.size()]));
    }

    if (!this.basePackages.isEmpty()) {
        if (logger.isInfoEnabled()) {
            logger.info("Scanning base packages: [" +
                    StringUtils.collectionToCommaDelimitedString(this.basePackages) + "]");
        }
        scanner.scan(this.basePackages.toArray(new String[this.basePackages.size()]));
    }

    //获取容器定义的Bean定义资源路径
    String[] configLocations = getConfigLocations();
    //如果定位的Bean定义资源路径不为空
    if (configLocations != null) {
        for (String configLocation : configLocations) {
            try {
                //使用当前容器的类加载器加载定位路径的字节码类文件
                Class<?> clazz = ClassUtils.forName(configLocation, getClassLoader());
                if (logger.isInfoEnabled()) {
                    logger.info("Successfully resolved class for [" + configLocation + "]");
                }
                reader.register(clazz);
            }
            catch (ClassNotFoundException ex) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Could not load class for config location [" + configLocation +
                            "] - trying package scan. " + ex);
                }
                //如果容器类加载器加载定义路径的Bean定义资源失败，则启用容器类路径扫描器扫描给定路径包及其子包中的类
                int count = scanner.scan(configLocation);
                if (logger.isInfoEnabled()) {
                    if (count == 0) {
                        logger.info("No annotated classes found for specified class/package [" + configLocation + "]");
                    }
                    else {
                        logger.info("Found " + count + " annotated classes in package [" + configLocation + "]");
                    }
                }
            }
        }
    }
}
```
