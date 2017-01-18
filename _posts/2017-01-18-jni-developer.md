---
layout: post
title:  "JNI开发手记"
date:   2016-01-18 21:10:35 +0800
categories: Java
tags: [JNI, C/C++]
---

JNI开发过程中的关键问题分析和解决方法。

## 前言

已经有不少前辈已经对这个技术有了介绍了。但我仍然想要从原理的出发来讲述使用JNI技术的关键点。

### 目标

目标观众需要对线程、JVM内存管理有一定了解。
本文将介绍如下内容：

+ **内存**
+ **线程**
+ **对象及引用**

## 内存

分两大部份：

+ JVM堆：JVM堆管理的内存，通过垃圾回收机制管理内存；
+ Natvie内存：JVM Native内存。不在垃圾回收机制控制范围；

两者的区别：

```C

jobject obj = env->NewObject(...); // 建一个Java类对象，在JVM堆上申请内存。

int *pArray = (int *)malloc(sizeof(int) * 10); // Native堆上申请内存

```

对于JVM堆上申请的内存，我们不需要通过代码释放内存。但是，Java是通过引用计数来回收垃圾内存的，在一些场景下仍然有需要一些额外的代码来让垃圾回收器正确回收。文章后面会介绍。

### Java对象如何与Natvie内存关联呢？

Java程序可以通过一个long类型的属性，保存Native内存的一个指针指。

```java
class JNIDemo {
    private long nativePtr;
    public JNIDemo() {
        nativePtr = construct0(); // 保存native指针
    }
    public int getId() {
        return getId0(nativePtr); // 传入native指针
    }
    private native long construct0();

    private native int getId0(long ptr); 
}
```

C程序代码：
```cpp
typedef struct demo_native_t {
    int id_;
    char *data_;
} demo_native_t;

jlong JNIDemo_construct0(jenv *env, jobject obj) {
    demo_native_t *demo = (demo_native_t *)malloc(sizeof(demo_native_t)); // 这个函数最好在
    demo.id_ = 1;
    damo.data_ = NULL;
    return (jlong)demo;  /* 返回native指针 */
}

jint JNIDemo_getId0(jenv *env, jobject obj, jlong ptr) {
    demo_native_t *demo = (demo_native_t *)ptr; /* 强制转换 *／
    return demo.id_;
}

```

## 线程