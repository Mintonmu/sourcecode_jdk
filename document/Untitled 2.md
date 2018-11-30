**CAS****深入分析**

## 1、背景

我们平时在开发中或多或少都离不开多线程、并发等概念，虽然java为我们提供了一个便利的关键字synchronized，使用起来简单高效，但要想把synchronized、lock等并发相关的关键字和类库使用好也不是件容易的事，我们要透过现象看本质，去了解它们底层的实现原理，有助于我们理解并发中的一些概念，能更好的在开发中使用得当，本次就并发类库JUC中比较基础和重要的CAS概念进行分析。

## 2、概述

CAS ，Compare And Swap ，即比较并交换。Doug Lea 大神在实现同步组件时，大量使用CAS 技术，鬼斧神工地实现了Java 多线程的并发操作。整个 AQS 同步组件、Atomic 原子类操作等等都是基 CAS 实现的，甚至 ConcurrentHashMap 在 JDK 1.8 的版本中，也调整为 CAS + synchronized。可以说，CAS 是整个 J.U.C 的基石。

下图是JUC类库的基本结构：可以看出上层各种Lock 、并发容器、阻塞队列等类库底层都是依赖CAS+Volatile关键字的读写内存语义来实现的。

![201703090001](file:///C:\Users\wubinyu\AppData\Local\Temp\msohtmlclip1\01\clip_image002.jpg)

## 3、CAS分析

在 CAS 中有三个参数：内存值 V、旧的预期值 A、要更新的值 B ，当且仅当内存值 V 的值等于旧的预期值 A 时，才会将内存值V的值修改为 B ，否则什么都不干。其伪代码如下：

if (this.value == A) {    this.value = B    return  true;  } else {    return  false;  }

J.U.C 下的 Atomic 类，都是通过 CAS 来实现的。下面会以 AtomicInteger 为例，来阐述 CAS 原理。

### 3.1、n++问题

一个n++的问题。

![](file:///C:\Users\wubinyu\AppData\Local\Temp\msohtmlclip1\01\clip_image003.png)

打开cmd通过javap -verbose Case看看add方法的字节码指令

![](file:///C:\Users\wubinyu\AppData\Local\Temp\msohtmlclip1\01\clip_image005.jpg)

n++被拆分成了几个指令：

1.  执行getfield拿到原始n
2.  执行iadd进行加1操作
3.  执行putfield写把累加后的值写回n

通过volatile修饰的变量可以保证线程之间的可见性，但并不能保证这3个指令的原子执行，在多线程并发执行下，无法做到线程安全，得到正确的结果，那么应该如何解决呢？

### 3.2、如何解决

在add方法加上synchronized修饰解决。

![](file:///C:\Users\wubinyu\AppData\Local\Temp\msohtmlclip1\01\clip_image006.png)

这个方案当然可行，但是性能上差了点，还有其它方案么？

再来看一段代码

public int a = 1;

public boolean compareAndSwapInt(int b) {

 if (a == 1) {

 a = b;

 return true;

 }

 return false;

}

如果这段代码在并发下执行，会发生什么？

假设线程1和线程2都过了a==1的检测，都准备执行对a进行赋值，结果就是两个线程同时修改了变量a，显然这种结果是无法符合预期的，无法确定a的最终值。

解决方法也同样暴力，在compareAndSwapInt方法加锁同步，变成一个原子操作，同一时刻只有一个线程才能修改变量a。

除了低性能的加锁方案，我们还可以使用JDK自带的CAS方案，在CAS中，比较和替换是一组原子操作，不会被外部打断，且在性能上更占有优势。

### 3.3、AtomicInteger

下面以AtomicInteger的实现为例，分析一下CAS是如何实现的。

![](file:///C:\Users\wubinyu\AppData\Local\Temp\msohtmlclip1\01\clip_image008.jpg)

· Unsafe，是CAS的核心类，由于Java方法无法直接访问底层系统，需要通过本地（native）方法来访问，Unsafe相当于一个后门，基于该类可以直接操作特定内存的数据。

·  变量valueOffset，表示该变量值在内存中的偏移地址，因为Unsafe就是根据内存偏移地址获取数据的。

·  变量value用volatile修饰，保证了多线程之间的内存可见性。

看看AtomicInteger如何实现并发下的累加操作：

![](file:///C:\Users\wubinyu\AppData\Local\Temp\msohtmlclip1\01\clip_image010.jpg)

![](file:///C:\Users\wubinyu\AppData\Local\Temp\msohtmlclip1\01\clip_image012.jpg)

假设线程A和线程B同时执行getAndAdd操作（分别跑在不同CPU上）：

1.  AtomicInteger里面的value原始值为3，即主内存中AtomicInteger的value为3，根据Java内存模型，线程A和线程B各自持有一份value的副本，值为3。
2.  线程A通过getIntVolatile(var1, var2)拿到value值3，这时线程A被挂起。
3.  线程B也通过getIntVolatile(var1, var2)方法获取到value值3，运气好，线程B没有被挂起，并执行compareAndSwapInt方法比较内存值也为3，成功修改内存值为2。
4.  这时线程A恢复，执行compareAndSwapInt方法比较，发现自己手里的值(3)和内存的值(2)不一致，说明该值已经被其它线程提前修改过了，那只能重新来一遍了。
5.  重新获取value值，因为变量value被volatile修饰，所以其它线程对它的修改，线程A总是能够看到，线程A继续执行compareAndSwapInt进行比较替换，直到成功。

整个过程中，利用CAS保证了对于value的修改的并发安全，继续深入看看Unsafe类中的compareAndSwapInt方法实现。

![](file:///C:\Users\wubinyu\AppData\Local\Temp\msohtmlclip1\01\clip_image014.jpg)

Unsafe类中的compareAndSwapInt，是一个本地方法，该方法的实现位于unsafe.cpp中，

是jdk中的c++代码，这里我就不再深入c++源码分析了，感兴趣的可以去研究下。

UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x)) UnsafeWrapper("Unsafe_CompareAndSwapInt"); oop p = JNIHandles::resolve(obj); jint* addr = (jint *) index_oop_from_field_offset_long(p, offset); return (jint)(Atomic::cmpxchg(x, addr, e)) == e; UNSAFE_END

1. 先想办法拿到变量value在内存中的地址。

2. 通过Atomic::cmpxchg实现比较替换，其中参数x是即将更新的值，参数e是原内存的值。

### 3.4、CAS缺点

CAS存在一个很明显的问题，即ABA问题。

问题：如果变量V初次读取的时候是A，并且在准备赋值的时候检查到它仍然是A，那能说明它的值没有被其他线程修改过了吗？

如果在这段期间曾经被改成B，然后又改回A，那CAS操作就会误认为它从来没有被修改过。针对这种情况，java并发包中提供了一个带有标记的原子引用类AtomicStampedReference，它可以通过控制变量值的版本来保证CAS的正确性。