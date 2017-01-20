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

## 内存

分两大部份：

> + JVM堆：JVM堆管理的内存，通过垃圾回收机制管理内存；
> + Native堆：JVM Native内存。不在垃圾回收机制控制范围；

两者的区别：

```cpp

jobject obj = env->NewObject(...); // 建一个Java类对象，在JVM堆上申请内存。

int *pArray = (int *)malloc(sizeof(int) * 10); // Native堆上申请内存

```

对于JVM堆上申请的内存，我们不需要通过代码释放内存。但是，Java是通过引用计数来回收垃圾内存的，在一些场景下仍然有需要一些额外的代码来让垃圾回收器正确回收。文章后面会介绍。

### Java对象引用Native指针

Java程序可以通过一个long类型的属性，保存Native内存的指针。

```java
class JNIDemo {
    private long nativePtr;
    public JNIDemo() {
        nativePtr = construct0(); // 保存native指针
    }
    public int getId() {
        return getId0(nativePtr); // 传入native指针
    }

    public void Dispose() {
        if (nativePtr > 0L) {
            dispose0(nativePtr);
            nativePtr = 0L;
        }
    }

    private native long construct0();

    private native int getId0(long ptr); 

    private native void dispose0(long ptr);
}
```

C程序代码：

```cpp
typedef struct demo_native_t {
    int id_;
    char *data_;
} demo_native_t;

jlong JNIDemo_construct0(jenv *env, jobject obj) {
    // 这申请了一片native内存。需要找一个合适的时机释放掉。
    demo_native_t *demo = (demo_native_t *)malloc(sizeof(demo_native_t)); 
    demo.id_ = 1;
    damo.data_ = NULL;
    /* 返回native指针 */
    return (jlong)demo;
}

jint JNIDemo_getId0(jenv *env, jobject obj, jlong ptr) {
    /* 强制转换，就能用咯 */
    demo_native_t *demo = (demo_native_t *)ptr;
    return demo.id_;
}

void JNIDemo_dispose0(jenv *env, jobject obj, jlong ptr) {
    demo_native_t *demo = (demo_native_t *)ptr;
    free(demo);
}

```

### Native代码引用Java对象

这就需要使用“引用”。相关函数列表：

#### [Global and Local References](http://docs.oracle.com/javase/6/docs/technotes/guides/jni/spec/functions.html#global_local)

> + NewGlobalRef
> + DeleteGlobalRef
> + DeleteLocalRef
> + EnsureLocalCapacity
> + PushLocalFrame
> + PopLocalFrame
> + NewLocalRef

#### [Weak Global References](http://docs.oracle.com/javase/6/docs/technotes/guides/jni/spec/functions.html#weak)

> + NewWeakGlobalRef
> + DeleteWeakGlobalRef

弱引用主要是为了防止native代码引用了Java对象从而阻止垃圾回收，造成内存泄漏。与c++的weak_ptr的目标是一致的。那么什么时候用GlobalRef，什么时候用LocalRef呢？官方对Local References的解释如下：

> Local references are valid for the duration of a native method call. They are freed automatically after the native method returns. Each local reference costs some amount of Java Virtual Machine resource. Programmers need to make sure that native methods do not excessively allocate local references. Although local references are automatically freed after the native method returns to Java, excessive allocation of local references may cause the VM to run out of memory during the execution of a native method.


我认为：线程是由C／C++代码发起时，需要手动调用DeleteLocalRef释放Local References。而线程由Java代码发起调用native方法时，不需要管理Local References。因为native方法返回时JVM会自动释放。

那么native代码需要引用一个Java对象，最好使用Global References或者Weak Global References。

Java代码：

```java
class JNIDemo {
    private long nativePtr;
    public JNIDemo() {
        nativePtr = construct0(); // 保存native指针
    }
    public int getId() {
        return getId0(nativePtr); // 传入native指针
    }

    public void Dispose() {
        if (nativePtr > 0L) {
            dispose0(nativePtr);
            nativePtr = 0L;
        }
    }

    private native long construct0();

    private native int getId0(long ptr); 

    private native void dispose0(long ptr);
}
```

C程序代码：

```cpp
typedef struct demo_native_t {
    int id_;
    char *data_;
    jobject obj_;
} demo_native_t;

jlong JNIDemo_construct0(jenv *env, jobject obj) {
    demo_native_t *demo = (demo_native_t *)malloc(sizeof(demo_native_t)); 
    demo.id_ = 1;
    demo.data_ = NULL;
    ／* 引用 *／
    demo.obj_ = env->NewGlobalRef(obj);
    /* 返回native指针 */
    return (jlong)demo;
}

void JNIDemo_dispose0(jenv *env, jobject obj, jlong ptr) {
    demo_native_t *demo = (demo_native_t *)ptr;
    /* 释放引用 */
    env->DeletelobalRef(demo->obj_);
    free(demo);
}

```

上面Java代码需要用户通过调用Dispose方法清理Native资源。当想自动释放该如何处理呢？

Java代码：

```java
class JNIDemo {
    private long nativePtr;
    public JNIDemo() {
        nativePtr = construct0(); // 保存native指针
    }
    public int getId() {
        return getId0(nativePtr); // 传入native指针
    }

    public void Dispose() {
        if (nativePtr > 0L) {
            dispose0(nativePtr);
            nativePtr = 0L;
        }
    }

    protected void finalize() {
        super.finalize();
        Dispose();
    }

    private native long construct0();

    private native int getId0(long ptr); 

    private native void dispose0(long ptr);
}
```

C程序代码：

```cpp
typedef struct demo_native_t {
    int id_;
    char *data_;
    jobject obj_;
} demo_native_t;

jlong JNIDemo_construct0(jenv *env, jobject obj) {
    demo_native_t *demo = (demo_native_t *)malloc(sizeof(demo_native_t)); 
    demo.id_ = 1;
    demo.data_ = NULL;
    ／* 弱引用，比阻止垃圾回收 *／
    demo.obj_ = env->NewWeakGlobalRef(obj);
    /* 返回native指针 */
    return (jlong)demo;
}

void JNIDemo_dispose0(jenv *env, jobject obj, jlong ptr) {
    demo_native_t *demo = (demo_native_t *)ptr;
    /* 释放引用 */
    env->DeleteWeakGlobalRef(demo->obj_);
    free(demo);
}

```

弱引用就是这么试用滴！

## 线程

和内存一样也分两类：

> + Java线程
> + Native线程

著名的jenv实例是Java线程独立的，JVM会为每一个Java线程创建一个jenv实例。

由C/C++代码发起的线程称为Native线程。反之则为Java线程。
当Java代码调用native方法时，使用的线程既是Java线程。这种场景下，不需要特殊的处理。
当C/C++代码发生回调时需要调用Java方法时，这时使用的线程为Native线程。这种场景下，需要在调用Java方法之前必须将Native线程关联成Java线程，并且在调用结束后解除关联。相关函数列表：

> + AttachCurrentThread
> + AttachCurrentThreadAsDaemon
> + DetachCurrentThread

## 参考文献

+ [Java Native Interface Specification](http://docs.oracle.com/javase/6/docs/technotes/guides/jni/spec/jniTOC.html)