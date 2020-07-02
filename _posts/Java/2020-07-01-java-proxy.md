---
layout: post
title: 代理模式
category: Java
tags: [设计模式]
---

代理模式是面向对象编程中比较常见的设计模式。在Java中分为静态代理、动态代理，其中动态代理又可以分为JDK动态代理、cglib动态代理。

## 一、静态代理

- 接口
- 目标对象
- 代理对象

- 优点： 代理对象与目标对象要实现相同的接口，可以做到在不修改目标对象的基础上，对程序功能进行扩展。
- 缺点： 一旦接口增加或者减少方法，那么目标对象和代理对象都需要维护。

接口：
```java
public interface Movie {
    void play();
}
```

目标类：
```java
public class RealMovie implements Movie {

    @Override
    public void play() {
        System.out.println("葫芦娃");
    }
}
```

代理类：
```java
public class ProxyMovie implements Movie {

    private RealMovie realMovie;

    public ProxyMovie(RealMovie realMovie) {
        this.realMovie = realMovie;
    }

    @Override
    public void play() {
        // 在被代理对象前后增加业务
        before();
        realMovie.play();
        after();
    }

    public void before() {
        System.out.println("西游记");
    }

    public void after() {
        System.out.println("大头儿子");
    }
}
```

测试：
```java
public class Test {

    public static void main(String[] args) {
        Movie movie = new ProxyMovie(new RealMovie());
        movie.play();
    }

}
```


## 二、动态代理

代理类在程序运行时创建的代理方式被称为动态代理。也就是说，代理类并不需要在Java代码中定义，而是在运行时动态生成的。相比于静态代理，
动态代理的优势在于可以很方便的对代理类的函数进行统一的处理，而不用修改每个代理类的函数。

动态代理主要运用于框架中，例如在Spring的AOP中:
- 如果加入容器的目标对象有实现接口,使用JDK代理
- 如果目标对象没有实现接口,则使用Cglib代理

### 1、JDK动态代理

代理对象，不需要实现接口，目标对象一定要实现接口，利用JDK的API动态的在内存中构建代理对象。

动态代理的代理类需要实现InvocationHandler接口，通过reflect.Proxy的类的newProxyInstance方法可以得到这个接口的实例。

JDK代理类：
```java
public class MyInvocationHandler implements InvocationHandler {

    private Object object;

    public MyInvocationHandler(Object object) {
        this.object = object;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        before();
        Object invoke = method.invoke(object, args);
        after();
        return invoke;
    }

    private void before() {
        System.out.println("西游记");
    }

    private void after() {
        System.out.println("大头儿子");
    }
}
```

测试：
```java
public class Test {

    public static void main(String[] args) {
        RealMovie realMovie = new RealMovie();
        InvocationHandler handler = new MyInvocationHandler(realMovie);
        Movie movie = (Movie) Proxy.newProxyInstance(RealMovie.class.getClassLoader(), RealMovie.class.getInterfaces(), handler);
        movie.play();
    }

}
```


### 2、cglib动态代理

静态代理和JDK动态代理都要求 目标对象 实现接口。但有时目标对象只是一个单独的对象，没有实现任何接口，这个时候就可以使用以目标对象子类
的方式实现代理，这种方法叫做Cglib代理。

Cglib包的底层是通过使用字节码处理框架ASM来转换字节码并生成新的类。

- 用CGlib生成代理类是目标类的子类。
- 用CGlib生成代理类不需要接口。
- 用CGLib生成的代理类重写了父类的各个方法。
- 拦截器中的intercept方法内容就是代理类中的方法体。


目标类
```java
public class MovieImpl {

    public void play() {
        System.out.println("葫芦娃");
    }

}
```

引入cglib pom依赖

代理类：
```java
public class CglibProxyInterceptor implements MethodInterceptor {

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        before();
        Object object = methodProxy.invokeSuper(o, objects);
        after();
        return object;
    }

    private void before() {
        System.out.println("西游记");
    }

    private void after() {
        System.out.println("大头儿子");
    }
}
```

测试：
```java
public class Test {

    public static void main(String[] args) {
        // 创建Enhancer对象，类似于JDK动态代理的Proxy类
        Enhancer enhancer = new Enhancer();
        // 设置目标类的字节码文件
        enhancer.setSuperclass(MovieImpl.class);
        // 设置回调函数
        enhancer.setCallback(new CglibProxyInterceptor());
        // creat方法就是正式创建代理类
        MovieImpl movie = (MovieImpl)enhancer.create();
        // 调用代理类的play方法
        movie.play();
    }

}
```