---
layout: post
title:  "JNI开发手记"
date:   2016-01-18 21:10:35 +0800
categories: Java
tags: [JNI, C/C++]
---

JNI开发过程中的关键问题分析和解决方法。

## 前言

已经有不少前辈已经对这个技术有了介绍了。但我仍然想要从原理的角度出发，讲述使用JNI技术的关键点。本文主要内容是在使用JNI技术开发过程中，翻阅官方文档的得到的理解，并辅以一点来说明问题。希望能对想了深入理解JNI技术的同学有一点帮助。

## 声明

+ 目标观众需要对线程、JVM内存管理有一定了解。
+ 本文下面的代码主要起示例作用，不保证正确性。
+ 对于想更深入理解JNI的同学，请参考:[Java Native Interface Specification](http://docs.oracle.com/javase/6/docs/technotes/guides/jni/spec/jniTOC.html)

## 概述

本文将介绍如下内容：

+ **内存**
+ **线程**
+ **C/C++调用Java方法**
+ **异常处理**

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

jlong Java_JNIDemo_construct0(JNIEnv *env, jobject obj) {
    // 这申请了一片native内存。需要找一个合适的时机释放掉。
    demo_native_t *demo = (demo_native_t *)malloc(sizeof(demo_native_t)); 
    demo.id_ = 1;
    damo.data_ = NULL;
    /* 返回native指针 */
    return (jlong)demo;
}

jint Java_JNIDemo_getId0(JNIEnv *env, jobject obj, jlong ptr) {
    /* 强制转换，就能用咯 */
    demo_native_t *demo = (demo_native_t *)ptr;
    return demo.id_;
}

void Java_JNIDemo_dispose0(JNIEnv *env, jobject obj, jlong ptr) {
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

jlong Java_JNIDemo_construct0(JNIEnv *env, jobject obj) {
    demo_native_t *demo = (demo_native_t *)malloc(sizeof(demo_native_t)); 
    demo.id_ = 1;
    demo.data_ = NULL;
    /* 引用 */
    demo.obj_ = env->NewGlobalRef(obj);
    /* 返回native指针 */
    return (jlong)demo;
}

void Java_JNIDemo_dispose0(JNIEnv *env, jobject obj, jlong ptr) {
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

jlong Java_JNIDemo_construct0(JNIEnv *env, jobject obj) {
    demo_native_t *demo = (demo_native_t *)malloc(sizeof(demo_native_t)); 
    demo.id_ = 1;
    demo.data_ = NULL;
    /* 弱引用，比阻止垃圾回收 */
    demo.obj_ = env->NewWeakGlobalRef(obj);
    /* 返回native指针 */
    return (jlong)demo;
}

void Java_JNIDemo_dispose0(JNIEnv *env, jobject obj, jlong ptr) {
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

著名的JNIEnv实例是Java线程独立的，JVM会为每一个Java线程创建一个JNIEnv实例。

由C/C++代码发起的线程称为Native线程。反之则为Java线程。（此处术语我瞎编的，能区分两者即可^_^）
当Java代码调用native方法时，使用的线程既是Java线程。这种场景下，不需要特殊的处理。
当C/C++代码发生回调时需要调用Java方法时，这时使用的线程为Native线程。这种场景下，需要在调用Java方法之前必须将Native线程关联成Java线程，并且在调用结束后解除关联。相关函数列表：

> + AttachCurrentThread
> + AttachCurrentThreadAsDaemon
> + DetachCurrentThread

```cpp

void demo_cb(JavaVM *jvm, ...)
{
    JNIEnv *env = NULL;
    JavaVMAttachArgs attachArgs = {
        JNI_VERSION_1_6, /* JNI版本号 */
        "demo_cb thread", /* 线程名称 */
        NULL   /* 线程组。这里就不设置啦 */
        };
    
    int rc = AttachCurrentThread(jvm, &env, &attachArgs);
    if (rc == JNI_OK)
    {
        // Call Java code.
        ...

        // 处理完后，记得detach哟
        DetachCurrentThread(jvm);
    }
    else
    {
         // handle error.
        ...
    }
}


```

## C/C++调用Java方法

关键函数列表：

> + FindClass
> + GetMethodID
> + NewObject
> + Call<type>Method Routines, Call<type>MethodA Routines, and Call<type>MethodV Routines
> + CallNonvirtual<type>Method Routines, CallNonvirtual<type>MethodA Routines, and CallNonvirtual<type>MethodV Routines

原始类型对照表：

> |Java Type|Native Type|Description|
> |:----|:----|:----|
> |boolean|jboolean|unsigned 8 bits|
> |byte|jbyte|signed 8 bits|
> |char|jchar|unsigned 16 bits|
> |short|jshort|signed 16 bits|
> |int|jint|signed 32 bits|
> |long|jlong|signed 64 bits|
> |float|jfloat|32 bits|
> |double|jdouble|64 bits|
> |void|void|N/A|

引用类型继承关系：

![](/assets/images/jni-reference-types.gif)

类型签名：

|Type Signature|Java Type|
|:----|:----|
|Z|boolean|
|B|byte|
|C|char|
|S|short|
|I|int|
|J|long|
|F|float|
|D|double|
|L fully-qualified-class ;|fully-qualified-class|
|[ type|type[]|
|( arg-types ) ret-type|method type|

例如：下面Java方法：

> long f (int n, String s, int[] arr); 

类型签名如下:

> (ILjava/lang/String;[I)J


## 异常处理

异常处理函数列表：

> + Throw             你懂的
> + ThrowNew          你懂的
> + ExceptionOccurred 检测是否有异常发生；
> + ExceptionDescribe 打印异常堆栈信息；
> + ExceptionClear    清理所有异常
> + FatalError        触发一个致命错误
> + ExceptionCheck    也是检查异常（还不太了解）

注意Throw和ThrowNew方法，并不会终止后面的C/C++代码。

## 参考文献

+ [Java Native Interface Specification](http://docs.oracle.com/javase/6/docs/technotes/guides/jni/spec/jniTOC.html)

