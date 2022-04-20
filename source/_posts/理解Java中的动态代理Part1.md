---
title: 理解Java中的动态代理Part1
date: 2021-07-25 17:19:43
comments: true #是否可评论
toc: true #是否显示文章目录
categories: "基础" #分类
tags:   #标签
    - JAVA
---
> 对Java动态代理的理解，有助于我们理解复杂框架的设计，是提高自身java水平的一个重要的点。
  像Spring和Hadoop等开源框架中都有使用动态代理。
##### **代理模式**
首先说一下代理模式，它有一个subject(接口），一个RealSubject实现类和一个Proxy实现类。
客户端不会直接访问到RealSubject,只会访问Proxy对象，Proxy对象会有一个RealSubject对象
的实例作为成员变量，在subject定义的方法中做一些增强功能然后再调用RealSubject中实现的接口中
的方法。
##### **为什么需要代理?**
这是不是多此一举，我们直接new RealSubject,然后调用方法不就可以吗？
是的，通常就是这么使用的，但是在框架的设计中，通常会有一些统一的统计或监控等工作，譬如记录
每一个方法执行的开始结束时间，运行时长和方法被调用时传入的参数等信息，你不用在每一个方法实现
中都加入这一段功能代码，你可以拦截方法的执行，在方法执行前后加一些统一的代码。
##### **动态代理中的动态是什么意思？**
静态代理中代理类是在代码中提前写好的。而使用JDK反射包下的Proxy和InvocationHandler来实现的
代理中，代理类事先并不存在，是在运行过程中根据传入的类加载器和接口动态生成的字节码，然后加载成
类对象，并构造一个接口的实现类实例对象供客户端使用。
##### **JAVA中的动态代理怎么使用？**
Java的反射包下面有两个类，Proxy和InvocationHandler接口。
InvocationHandler接口中有一个invoke方法,它接收一个方法元数据和方法参数数组，因为事先不知道
被代理对象的具体类型，所以这里使用反射来操作，要告诉它执行什么方法，参数是什么。
首先我们做代理的目标是要增强功能，我们把我们要增强的功能写在哪里？就是写在这个invoke方法里面。
所以我们需要写一个类来实现这个InvocationHandler接口，这个类需要有一个Object对象（如果不使用泛型）
的成员变量来表示被代理的类实例；提供一种对象的注入方式，譬如set或者构造器都可以。
然后在invoke方法中把我们的增强逻辑实现，然后需要写method.invoke(obj,args);obj就是被代理对象，
args就是传入的method方法的参数。
Proxy类是来做什么的呢？
按理说，有了实现InvocationHandler的实现类就可以使用了，但是这样使用起来太不方便。Proxy可以用来动
态生成一个代理类,生成的代理类可以转成被代理的接口对象。
``` java
Proxy.newProxyInstance(RealSubject.class.getClassLoader(),
                RealSubject.class.getInterfaces(),
                handler);
```
它根据传入的参数动态生成字节码，并命名为$proxy + num，这个类扩展了Proxy并实现了RealSubject实现的接口。
同时会有几个静态的static Method成员变量代表接口中的方法，后面供InvocationHandler的invoke方法使用。
Proxy类有一个构造器是接收InvocationHandler作为参数，所以newProxyInstance时传入的handler就会被传入
新生成的类中使用；同时它还会有RealSubject中要实现的方法，在这个方法中会调用handler的invoke方法。

代码示例如下，你可以下面的代码中体会一下：
``` java
package org.java;

import sun.misc.ProxyGenerator;
import java.io.FileOutputStream;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;


public class Test {

    public static void main(String[] args) {

        Add adder = new Adder();
        CalculateIH handler = new CalculateIH(adder);
        Add adder1 = (Add) Proxy.newProxyInstance(Adder.class.getClassLoader(),
                Adder.class.getInterfaces(),
                handler);
        System.out.println( adder1.add(1,100));


        Multiplyer m = new Multiplyer();
        CalculateIH handler2 = new CalculateIH(m);
        multiply multiply = (multiply) Proxy.newProxyInstance(Multiplyer.class.getClassLoader(),
                Multiplyer.class.getInterfaces(),
                handler2);
        System.out.println( multiply.multiply(12,12));

        byte[] classFile = ProxyGenerator.generateProxyClass("$Proxy10", Adder.class.getInterfaces());
        String path = "./AdderProxy.class";
        try(FileOutputStream fos = new FileOutputStream(path)) {
            fos.write(classFile);
            fos.flush();
            System.out.println("代理类class文件写入成功");
        } catch (Exception e) {
            e.printStackTrace();
            System.out.println("写文件错误");
        }

    }

}

interface  Add {
    int add(int add1, int add2);
}

class Adder implements  Add {
    @Override
    public int add(int add1, int add2) {
        return add1 + add2;
    }
}

interface  multiply {
    int multiply (int m1, int m2);
}

class Multiplyer implements  multiply {

    @Override
    public int multiply(int m1, int m2) {
        return m1 * m2;
    }
}

class CalculateIH implements InvocationHandler {

    Object obj;

   public CalculateIH(Object obj) {
        this.obj = obj;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        System.out.println("start at : " + System.nanoTime());
        Object ret  = method.invoke(obj, args);
        System.out.println("executed result is " + ret);
        System.out.println("end at : " + System.nanoTime());
        return ret;
    }
}

```
以下是对生成的代理类反编译的java代码：
``` java 
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;
import org.java.Add;

public final class $Proxy10 extends Proxy implements Add {
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;

    public $Proxy10(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int add(int var1, int var2) throws  {
        try {
            return (Integer)super.h.invoke(this, m3, new Object[]{var1, var2});
        } catch (RuntimeException | Error var4) {
            throw var4;
        } catch (Throwable var5) {
            throw new UndeclaredThrowableException(var5);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m3 = Class.forName("org.java.Add").getMethod("add", Integer.TYPE, Integer.TYPE);
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}

```



