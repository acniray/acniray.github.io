---
layout: post
title: 关于sum.misc.Unsafe的一点记录
categories: Java
description: 最近在读Undertow的源码，对于ServletPrintWriterDelegate类的实现比较感兴趣，做个记录。
keywords: Java, Undertow, Unsafe
---
关于sum.misc.Unsafe的一点记录
==================

## 需求

源码github坐标：[undertow/ServletPrintWriterDelegate.java at master · undertow-io/undertow · GitHub](https://github.com/undertow-io/undertow/blob/master/servlet/src/main/java/io/undertow/servlet/spec/ServletPrintWriterDelegate.java)

该类继承的是PrintWriter，但是由于并不实用PrintWriter中的OutputStream，所以实用Unsafe.allocateInstance构造了一个newInstance，完美的绕过了父类的构造方法，并可用于所有适用接口。

做个记录，备忘。

## 生成实例：

```java
public static ServletPrintWriterDelegate newInstance(final ServletPrintWriter servletPrintWriter) {
    final ServletPrintWriterDelegate delegate;
    if (System.getSecurityManager() == null) {
        try {
            delegate = (ServletPrintWriterDelegate) UNSAFE.allocateInstance(ServletPrintWriterDelegate.class);
        } catch (InstantiationException e) {
            throw new RuntimeException(e);
        }
    } else {
        delegate = AccessController.doPrivileged(new PrivilegedAction<ServletPrintWriterDelegate>() {
            @Override
            public ServletPrintWriterDelegate run() {
                try {
                    return  (ServletPrintWriterDelegate) UNSAFE.allocateInstance(ServletPrintWriterDelegate.class);
                } catch (InstantiationException e) {
                    throw new RuntimeException(e);
                }
            }
        });
    }
    delegate.setServletPrintWriter(servletPrintWriter);
    return delegate;
}
```

## 获取Unsafe:

```java
private static Unsafe getUnsafe() {
    if (System.getSecurityManager() != null) {
        return AccessController.doPrivileged(new PrivilegedAction<Unsafe>() {
            public Unsafe run() {
                return getUnsafe0();
            }
        });
    }
    return getUnsafe0();
}

private static Unsafe getUnsafe0()  {
    try {
        Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
        theUnsafe.setAccessible(true);
        return (Unsafe) theUnsafe.get(null);
    } catch (Throwable t) {
        throw new RuntimeException("JDK did not allow accessing unsafe", t);
    }
}
```

关于Unsafe进行了简单的查看，除了用于绕过jvm生成对象之外，该类还提供了非常多的基于底层的操作，请参见：

sun.misc.Unsafe的理解

读完之后有几个问题尚未深究，回头考虑补充：

1、是否可以在一定程度上，使用该类去替代一些反射的调用。

2、据说Unsafe的操作，是会绕过jvm的GC的，那如同上文那样的操作，allocateInstance出来的对象，是否需要手动释放？

检查了下，通过allocateInstance生成的对象，内存空间属于Java Heap，会被gc，不需要也不可以进行手动释放。
