---
layout: post
title: spring5源码分析系列（八）——基于XML的依赖注入（二）
category: Spring
tags: [Spring]
no-post-nav: true
---

&ensp;&ensp;&ensp;&ensp;前言：上一篇讲到了populateBean方法对Bean属性的依赖注入，此篇继续后面的内容。

7.BeanDefinitionValueResolver解析属性值

&ensp;&ensp;&ensp;&ensp;当容器在对属性进行依赖注入时，如果发现属性值需要进行类型转换，比如属性值是容器中另一个Bean实例对象的引用，
则容器首先需要根据属性值解析出所引用的对象，然后才能将该引用对象注入到目标实例对象的属性上去，对属性进行解析由resolveValueIfNecessary方法实现，源码如下：

```
//解析属性值，对注入类型进行转换
@Nullable
public Object resolveValueIfNecessary(Object argName, @Nullable Object value) {
    //对引用类型的属性进行解析
    if (value instanceof RuntimeBeanReference) {
        RuntimeBeanReference ref = (RuntimeBeanReference) value;
        //调用引用类型属性的解析方法
        return resolveReference(argName, ref);
    }
    //对属性值是引用容器中另一个Bean名称的解析
    else if (value instanceof RuntimeBeanNameReference) {
        String refName = ((RuntimeBeanNameReference) value).getBeanName();
        refName = String.valueOf(doEvaluate(refName));
        //从容器中获取指定名称的Bean
        if (!this.beanFactory.containsBean(refName)) {
            throw new BeanDefinitionStoreException(
                    "Invalid bean name '" + refName + "' in bean reference for " + argName);
        }
        return refName;
    }
    //对Bean类型属性的解析，主要是Bean中的内部类
    else if (value instanceof BeanDefinitionHolder) {
        // Resolve BeanDefinitionHolder: contains BeanDefinition with name and aliases.
        BeanDefinitionHolder bdHolder = (BeanDefinitionHolder) value;
        return resolveInnerBean(argName, bdHolder.getBeanName(), bdHolder.getBeanDefinition());
    }
    else if (value instanceof BeanDefinition) {
        BeanDefinition bd = (BeanDefinition) value;
        String innerBeanName = "(inner bean)" + BeanFactoryUtils.GENERATED_BEAN_NAME_SEPARATOR +
                ObjectUtils.getIdentityHexString(bd);
        return resolveInnerBean(argName, innerBeanName, bd);
    }
    //对集合数组类型的属性解析
    else if (value instanceof ManagedArray) {
        ManagedArray array = (ManagedArray) value;
        Class<?> elementType = array.resolvedElementType;
        if (elementType == null) {
            //获取数组元素的类型
            String elementTypeName = array.getElementTypeName();
            if (StringUtils.hasText(elementTypeName)) {
                try {
                    //使用反射机制创建指定类型的对象
                    elementType = ClassUtils.forName(elementTypeName, this.beanFactory.getBeanClassLoader());
                    array.resolvedElementType = elementType;
                }
                catch (Throwable ex) {
                    throw new BeanCreationException(
                            this.beanDefinition.getResourceDescription(), this.beanName,
                            "Error resolving array type for " + argName, ex);
                }
            }
            //没有获取到数组的类型，也没有获取到数组元素的类型
            //则直接设置数组的类型为Object
            else {
                elementType = Object.class;
            }
        }
        //创建指定类型的数组
        return resolveManagedArray(argName, (List<?>) value, elementType);
    }
    //解析list类型的属性值
    else if (value instanceof ManagedList) {
        return resolveManagedList(argName, (List<?>) value);
    }
    //解析set类型的属性值
    else if (value instanceof ManagedSet) {
        return resolveManagedSet(argName, (Set<?>) value);
    }
    //解析map类型的属性值
    else if (value instanceof ManagedMap) {
        return resolveManagedMap(argName, (Map<?, ?>) value);
    }
    //解析props类型的属性值，props其实就是key和value均为字符串的map
    else if (value instanceof ManagedProperties) {
        Properties original = (Properties) value;
        //创建一个拷贝，用于作为解析后的返回值
        Properties copy = new Properties();
        original.forEach((propKey, propValue) -> {
            if (propKey instanceof TypedStringValue) {
                propKey = evaluate((TypedStringValue) propKey);
            }
            if (propValue instanceof TypedStringValue) {
                propValue = evaluate((TypedStringValue) propValue);
            }
            if (propKey == null || propValue == null) {
                throw new BeanCreationException(
                        this.beanDefinition.getResourceDescription(), this.beanName,
                        "Error converting Properties key/value pair for " + argName + ": resolved to null");
            }
            copy.put(propKey, propValue);
        });
        return copy;
    }
    //解析字符串类型的属性值
    else if (value instanceof TypedStringValue) {
        TypedStringValue typedStringValue = (TypedStringValue) value;
        Object valueObject = evaluate(typedStringValue);
        try {
            //获取属性的目标类型
            Class<?> resolvedTargetType = resolveTargetType(typedStringValue);
            if (resolvedTargetType != null) {
                //对目标类型的属性进行解析，递归调用
                return this.typeConverter.convertIfNecessary(valueObject, resolvedTargetType);
            }
            //没有获取到属性的目标对象，则按Object类型返回
            else {
                return valueObject;
            }
        }
        catch (Throwable ex) {
            throw new BeanCreationException(
                    this.beanDefinition.getResourceDescription(), this.beanName,
                    "Error converting typed String value for " + argName, ex);
        }
    }
    else if (value instanceof NullBean) {
        return null;
    }
    else {
        return evaluate(value);
    }
}

//解析引用类型的属性值
@Nullable
private Object resolveReference(Object argName, RuntimeBeanReference ref) {
    try {
        Object bean;
        //获取引用的Bean名称
        String refName = ref.getBeanName();
        refName = String.valueOf(doEvaluate(refName));
        //如果引用的对象在父类容器中，则从父类容器中获取指定的引用对象
        if (ref.isToParent()) {
            if (this.beanFactory.getParentBeanFactory() == null) {
                throw new BeanCreationException(
                        this.beanDefinition.getResourceDescription(), this.beanName,
                        "Can't resolve reference to bean '" + refName +
                        "' in parent factory: no parent factory available");
            }
            bean = this.beanFactory.getParentBeanFactory().getBean(refName);
        }
        //从当前的容器中获取指定的引用Bean对象，如果指定的Bean没有被实例化,则会递归触发引用Bean的初始化和依赖注入
        else {
            bean = this.beanFactory.getBean(refName);
            //将当前实例化对象的依赖引用对象
            this.beanFactory.registerDependentBean(refName, this.beanName);
        }
        if (bean instanceof NullBean) {
            bean = null;
        }
        return bean;
    }
    catch (BeansException ex) {
        throw new BeanCreationException(
                this.beanDefinition.getResourceDescription(), this.beanName,
                "Cannot resolve reference to bean '" + ref.getBeanName() + "' while setting " + argName, ex);
    }
}
```

分析可知Spring是如何将引用类型，内部类以及集合类型等属性进行解析的，属性值解析完成后就可以进行依赖注入了，
依赖注入的过程就是Bean对象实例设置到它所依赖的Bean对象属性上去，依赖注入是通过bw.setPropertyValues方法实现的，该方法也使用了委托模式，
在BeanWrapper接口中定义了方法声明，依赖注入的具体实现交由其实现类BeanWrapperImpl来完成，接下来分析BeanWrapperImpl中依赖注入相关的源码。

8.BeanWrapperImpl对Bean属性的依赖注入

&ensp;&ensp;&ensp;&ensp;BeanWrapperImpl类主要是对容器中完成初始化的Bean实例对象进行属性的依赖注入，即把Bean对象设置到它所依赖的另一个Bean的属性中去。
BeanWrapperImpl中的注入方法由AbstractNestablePropertyAccessor来实现，源码如下：

```
//实现属性依赖注入功能
protected void setPropertyValue(PropertyTokenHolder tokens, PropertyValue pv) throws BeansException {
    if (tokens.keys != null) {
        processKeyedProperty(tokens, pv);
    }
    else {
        processLocalProperty(tokens, pv);
    }
}

//实现属性依赖注入功能
@SuppressWarnings("unchecked")
private void processKeyedProperty(PropertyTokenHolder tokens, PropertyValue pv) {
    //调用属性的getter(readerMethod)方法，获取属性的值
    Object propValue = getPropertyHoldingValue(tokens);
    PropertyHandler ph = getLocalPropertyHandler(tokens.actualName);
    if (ph == null) {
        throw new InvalidPropertyException(
                getRootClass(), this.nestedPath + tokens.actualName, "No property handler found");
    }
    Assert.state(tokens.keys != null, "No token keys");
    String lastKey = tokens.keys[tokens.keys.length - 1];

    //注入array类型的属性值
    if (propValue.getClass().isArray()) {
        Class<?> requiredType = propValue.getClass().getComponentType();
        int arrayIndex = Integer.parseInt(lastKey);
        Object oldValue = null;
        try {
            if (isExtractOldValueForEditor() && arrayIndex < Array.getLength(propValue)) {
                oldValue = Array.get(propValue, arrayIndex);
            }
            Object convertedValue = convertIfNecessary(tokens.canonicalName, oldValue, pv.getValue(),
                    requiredType, ph.nested(tokens.keys.length));
            //获取集合类型属性的长度
            int length = Array.getLength(propValue);
            if (arrayIndex >= length && arrayIndex < this.autoGrowCollectionLimit) {
                Class<?> componentType = propValue.getClass().getComponentType();
                Object newArray = Array.newInstance(componentType, arrayIndex + 1);
                System.arraycopy(propValue, 0, newArray, 0, length);
                setPropertyValue(tokens.actualName, newArray);
                //调用属性的getter(readerMethod)方法，获取属性的值
                propValue = getPropertyValue(tokens.actualName);
            }
            //将属性的值赋值给数组中的元素
            Array.set(propValue, arrayIndex, convertedValue);
        }
        catch (IndexOutOfBoundsException ex) {
            throw new InvalidPropertyException(getRootClass(), this.nestedPath + tokens.canonicalName,
                    "Invalid array index in property path '" + tokens.canonicalName + "'", ex);
        }
    }

    //注入list类型的属性值
    else if (propValue instanceof List) {
        //获取list集合的类型
        Class<?> requiredType = ph.getCollectionType(tokens.keys.length);
        List<Object> list = (List<Object>) propValue;
        //获取list集合的size
        int index = Integer.parseInt(lastKey);
        Object oldValue = null;
        if (isExtractOldValueForEditor() && index < list.size()) {
            oldValue = list.get(index);
        }
        //获取list解析后的属性值
        Object convertedValue = convertIfNecessary(tokens.canonicalName, oldValue, pv.getValue(),
                requiredType, ph.nested(tokens.keys.length));
        int size = list.size();
        //如果list的长度大于属性值的长度，则多余的元素赋值为null
        if (index >= size && index < this.autoGrowCollectionLimit) {
            for (int i = size; i < index; i++) {
                try {
                    list.add(null);
                }
                catch (NullPointerException ex) {
                    throw new InvalidPropertyException(getRootClass(), this.nestedPath + tokens.canonicalName,
                            "Cannot set element with index " + index + " in List of size " +
                            size + ", accessed using property path '" + tokens.canonicalName +
                            "': List does not support filling up gaps with null elements");
                }
            }
            list.add(convertedValue);
        }
        else {
            try {
                //将值添加到list中
                list.set(index, convertedValue);
            }
            catch (IndexOutOfBoundsException ex) {
                throw new InvalidPropertyException(getRootClass(), this.nestedPath + tokens.canonicalName,
                        "Invalid list index in property path '" + tokens.canonicalName + "'", ex);
            }
        }
    }

    //注入map类型的属性值
    else if (propValue instanceof Map) {
        //获取map集合key的类型
        Class<?> mapKeyType = ph.getMapKeyType(tokens.keys.length);
        //获取map集合value的类型
        Class<?> mapValueType = ph.getMapValueType(tokens.keys.length);
        Map<Object, Object> map = (Map<Object, Object>) propValue;
        TypeDescriptor typeDescriptor = TypeDescriptor.valueOf(mapKeyType);
        //解析map类型属性key值
        Object convertedMapKey = convertIfNecessary(null, null, lastKey, mapKeyType, typeDescriptor);
        Object oldValue = null;
        if (isExtractOldValueForEditor()) {
            oldValue = map.get(convertedMapKey);
        }
        //解析map类型属性value值
        Object convertedMapValue = convertIfNecessary(tokens.canonicalName, oldValue, pv.getValue(),
                mapValueType, ph.nested(tokens.keys.length));
        //将解析后的key和value值赋值给map集合属性
        map.put(convertedMapKey, convertedMapValue);
    }

    else {
        throw new InvalidPropertyException(getRootClass(), this.nestedPath + tokens.canonicalName,
                "Property referenced in indexed property path '" + tokens.canonicalName +
                "' is neither an array nor a List nor a Map; returned value was [" + propValue + "]");
    }
}
```

由此可知Spring IOC容器是这样将属性的值注入到Bean实例对象中的：
(1)对于集合类型的属性，将其属性值解析为目标类型的集合后直接赋值给属性；
(2)对于非集合类型的属性，大量使用了JDK的反射和内省机制，通过属性的getter方法(reader Method)获取指定属性注入以前的值，
同时调用属性的setter方法(writer Method)为属性设置注入后的值。

&ensp;&ensp;&ensp;&ensp;到这里Spring IOC容器对Bean定义资源文件的定位、载入、解析和依赖注入已经全部分析完了，现在Spring IOC容器中管理了一系列靠依赖关系联系起来的Bean，
程序不需要应用自己手动创建所需的对象，Spring IOC容器会在我们使用的时候自动为我们创建，并且注入好相关的依赖，这就是Spring核心功能的控制反转和依赖注入的相关功能。


