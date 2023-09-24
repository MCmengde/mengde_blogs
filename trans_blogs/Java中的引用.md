# Java中的引用

## 简介

在 Java 中有四种引用：

- **强引用(Strong Referenc)**
- **软引用(Soft References)**
- **弱引用(Weak References)**
- **虚引用(Phantom References)**

不同的引用，主要是垃圾回收时有区别。如果你从来没有听说过这些引用，那说明你只使用过强引用，了解他们的区别或许对你有所帮助，特别是当你需要临时存储对象，却不能使用真实的缓存库（例如 eHcache 和 Guava）时。

因为这些引用类型的不同和垃圾回收有很大的关联，这里首先回顾一下 Java 中的垃圾回收，然后再讨论不同引用的区别。

## 垃圾回收器

Java 和 C++ 最大的区别就是内存管理。在 Java 中，开发者不用担心内存使用的问题（但是一个合格的开发者应当了解），因为 JVM 中的垃圾回收器会帮助管理内存。

当你创建了一个对象，JVM 将它放在堆中，但是堆内存的大小是有限的，因此，JVM 需要经常删除对象来释放堆内存空间。在删除对象之前，JVM 需要判断该对象是否正在被使用，如果一个对象被**“垃圾回收根(garbage collection root)”**引用（可以是传递性的），则认为它正在被使用。

例如：

- 如果对象 C 被对象 B 引用，对象 B 又被对象 A 引用，然后对象 A 被**垃圾回收根**引用，则三个对象都将被判断为正在使用的对象（Case 1）。
- 然而，如果 B 不在被 A 引用，那么 B 和 C 都将被判定为非活跃对象，可能被回收掉（Case 2）。

<img src="Untitled.assets/JVM.png" alt="img" style="zoom: 80%;"/>

大致上，垃圾回收根分为以下四种类型，本文重点并非垃圾回收，这里不做深入的解释：

1. 本地变量
2. 活跃的 Java 线程
3. 静态变量
4. JNI 引用，也就是包含本地代码的 Java 对象，其内存不由 JVM 管理。

Oracle 并没有指明应当怎样管理内存，所以每种 JVM 实现都有它自己的一套算法。不过思路却大致相同：

- JVM 使用一种递归算法，寻找不活跃的对象，然后标记它们
- 被标记的对象进行最终确认(调用`finalize()`方法)，然后销毁
- JVM 有时会移动部分存活的对象，以便堆中能有大的连续内存

## 问题

如果 JVM 能够管理内存的话，那么我们开发者还有什么需要担心的呢？实际上，JVM 自动管理内存并不意味着内存泄漏不会发生。

很多时候，你无意中使用了**垃圾回收根**。例如，当你需要在程序中存储对象时（对象的初始化是很耗费资源的），倘若你使用了静态的数据结构来存储对象，以便可以在代码中的任何地方调用：

```java
public class OOM {
    public static List<Integer> myCachedObjects = new ArrayList<>();
 
    public static void main(String[] args) {
        for (int i = 0; i < 100_000_000; i++) {
            myCachedObjects.add(i);
        }
    }
}
```

以上代码的输出为：

>*Exception in thread “main” java.lang.OutOfMemoryError: Java heap space*

Java 中提供了不同的引用类型，以避免`OutOfMemoryError`。

有些类型甚至允许 JVM 释放程序正在使用的对象，具体怎么处理这些情况由程序员自行决定。

## 强引用

**强引用**是标准引用，当你创建如下对象时：

```java
MyClass obj = new MyClass();
```

就创建了一个**强引用**`obj`，指向了一个新创建的`MyClass`类的对象。当垃圾回收器查找非活跃对象时，它只需要检查对象是否是**强可达**的，即该对象是否由**强引用**传递连接到垃圾回收根。

使用该类型的引用可以保证 JVM 将该对象保存在堆中，正如上面讲述垃圾回收器时提到的那样。

## 软引用

根据 Java API，软引用是：

> Soft reference objects, which are cleared at the discretion of the garbage collector in response to memory demand

也就是说，在不同的 JVM(Oracle的Hotspot、JRockit，IBM的J9) 上，**软引用**的作用可能是不尽相同的。

让我们看看 Oracle 的 HotSpot 虚拟机（标准的、也是最常用的 JVM）是怎么管理软引用的。根据文档：

> The default value is 1000 ms per megabyte, which means that a soft reference will survive (after the last strong reference to the object has been collected) for 1 second for each megabyte of free space in the heap

这里给出具体的例子：假设堆的大小为 512M，其中 400M 是空闲内存。

我们创建一个对象 A，将它**软引用**于对象缓存，**强引用**于对象 B，因为 A 强引用于 B，所以它是**强可达**的，垃圾回收器不会删除它。（图一）

假设此时 B 被删除了，那么 A 只剩下了对象缓存对它的**软引用**，如果在接下来的 400 秒之内，没有**强引用**连接到 A，那么 A 将在 400 秒之后被删除。（图二）

![soft_ref1](https://mengde-pic-bed.oss-cn-hangzhou.aliyuncs.com/img/soft_ref1.png)

这里给出操作**软引用**的代码：

```java
public class ExampleSoftRef {
    public static class A{
 
    }
    public static class B{
        private A strongRef;
 
        public void setStrongRef(A ref) {
            this.strongRef = ref;
        }
    }
    public static SoftReference<A> cache;
 
    public static void main(String[] args) throws InterruptedException{
        //initialisation of the cache with a soft reference of instanceA
        ExampleSoftRef.A instanceA = new ExampleSoftRef.A();
        cache = new SoftReference<ExampleSoftRef.A>(instanceA);
        instanceA=null;
        // instanceA  is now only soft reachable and can be deleted by the garbage collector after some time
        Thread.sleep(5000);
 
        ...
        ExampleSoftRef.B instanceB = new ExampleSoftRef.B();
        //since cache has a SoftReference of instance A, we can't be sure that instanceA still exists
        //we need to check and recreate an instanceA if needed
        instanceA=cache.get();
        if (instanceA ==null){
            instanceA = new ExampleSoftRef.A();
            cache = new SoftReference<ExampleSoftRef.A>(instanceA);
        }
        instanceB.setStrongRef(instanceA);
        instanceA=null;
        // instanceA a is now only softly referenced by cache and strongly referenced by B so it cannot be cleared by the garbage collector
 
        ...
    }
}
```

然而，即使**软引用**指向的对象被垃圾回收器删除了，**软引用**本身（也是一个对象）却并没有被删除，它们也需要被清理。假设，你有一个较小的堆，只有 64M(XMX64m)，即使你使用了软引用，也可能造成 OutOfMemoryError，以下代码给出了例子：

```java
public class TestSoftReference1 {
 
    public static class MyBigObject{
        //each instance has 128 bytes of data
        int[] data = new int[128];
    }
    public static int CACHE_INITIAL_CAPACITY = 1_000_000;
    public static Set<SoftReference<MyBigObject>> cache = new HashSet<>(CACHE_INITIAL_CAPACITY);
 
    public static void main(String[] args) {
        for (int i = 0; i < 1_000_000; i++) {
            MyBigObject obj = new MyBigObject();
            cache.add(new SoftReference<>(obj));
            if (i%200_000 == 0){
                System.out.println("size of cache:" + cache.size());
            }
        }
        System.out.println("End");
    }
}
```

输出结果为：

> size of cache:1
> size of cache:200001
> size of cache:400001
> size of cache:600001
> Exception in thread “main” java.lang.OutOfMemoryError: GC overhead limit exceeded

Oracle 提供了`ReferenceQueue`，它是一个**软引用**对象的集合，使用该队列，就可以清理**软引用**对象，避免 OutOfMemoryError。

使用`ReferenceQueue`之后，在相同的堆大小的情况下，实现与上面代码一样的功能，可以存储更多的数据：

```java
public class TestSoftReference2 {
    public static int removedSoftRefs = 0;
 
    public static class MyBigObject {
        //each instance has 128 bytes of data
        int[] data = new int[128];
    }
 
    public static int CACHE_INITIAL_CAPACITY = 1_000_000;
    public static Set<SoftReference<MyBigObject>> cache = new HashSet<>(
            CACHE_INITIAL_CAPACITY);
    public static ReferenceQueue<MyBigObject> unusedRefToDelete = new ReferenceQueue<>();
 
    public static void main(String[] args) {
        for (int i = 0; i < 5_000_000; i++) {
            MyBigObject obj = new MyBigObject();
            cache.add(new SoftReference<>(obj, unusedRefToDelete));
            clearUselessReferences();
        }
        System.out.println("End, removed soft references=" + removedSoftRefs);
    }
 
    public static void clearUselessReferences() {
        Reference<? extends MyBigObject> ref = unusedRefToDelete.poll();
        while (ref != null) {
            if (cache.remove(ref)) {
                removedSoftRefs++;
            }
            ref = unusedRefToDelete.poll();
        }
 
    }
}
```

输出为：

> End, removed soft references=4976899

当你需要存储许多对象，**软引用**是非常有用的，倘若这些对象被 JVM 删掉了，重新实例化的代价是很高的。

## 弱引用

**弱引用**是一个比**软引用**更模糊的概念，Java API 中的定义为：

> Suppose that the garbage collector determines at a certain point in time that an object is [weakly reachable](https://docs.oracle.com/javase/7/docs/api/java/lang/ref/package-summary.html#reachability). At that time it will atomically clear all weak references to that object and all weak references to any other weakly-reachable objects from which that object is reachable through a chain of strong and soft references. At the same time it will declare all of the formerly weakly-reachable objects to be finalizable. At the same time or at some later time it will enqueue those newly-cleared weak references that are registered with reference queues.

也就是说，垃圾回收器会检查所有的对象，如果有一个对象到垃圾回收根的引用是**弱引用**（既没有强引用也没有软引用指向它），该对象将会被标记，然后尽快删除。使用`WeakReference`的方法和`SofrReference`一样，参照上文软引用例子即可。

Oracle 提供了一种有趣的基于**弱引用**的类：`WeakHashMap`，该数据结构的`key`是**弱引用**的。`WeakHashMap`可以被用作标准的`Map`，唯一的区别就是，在`key`从堆中被删除之后，它可以自行清除自己：

```java
public class ExampleWeakHashMap {
    public static Map<Integer,String> cache = new WeakHashMap<Integer, String>();
 
    public static void main(String[] args) {
        Integer i5 = new Integer(5);
        cache.put(i5, "five");
        i5=null;
        //the entry {5,"five"} will stay in the Map until the next garbage collector call
 
        Integer i2 = 2;
        //the entry {2,"two"} will stay  in the Map until i2 is no more strongly referenced
        cache.put(i2, "two");
 
        //remebmber the OutOfMemoryError at the chapter "problem", this time it won't happen
        // because the Map will clear its entries.
        for (int i = 6; i < 100_000_000; i++) {
            cache.put(i,String.valueOf(i));
        }
    }
}
```

在本例中， 我使用了`WeakHashMap`去解决以下问题：保存交易的多种信息，其中哈希表的键是交易的编号，值是交易的简要信息，我需要在交易进行时一直保存这些信息。使用这个数据结构，可以保证我一定能从其中拿到我想要的数据，因为在交易结束之前，交易记录的键不会被删除。并且我也不用关心如何清除这个哈希表。

Oracle 建议使用`WeakHashMap`作为“标准”的键值映射数据结构。

## 虚引用

在垃圾回收期间，没有强引用和软引用的对象都会被删除，在删除之前，`finalize()`方法会被调用。当一个对象被标记为可删除之后，还没删除之前，它变成**虚可达**，也就是说它和垃圾回收根之间的引用是**虚引用**。

和**软引用**以及**弱引用**不同，对对象显式使用**虚引用**是为了防止对象被删除的。程序员需要显示或者隐式地删除对象的**虚引用**，那些被标记为可删除的对象才能被删除。显式删除**虚引用**需要使用`ReferenceQueue`，该队列元素为已经被标记为可删除的对象的引用。

**虚引用**是不能检索到对象的，**虚引用**的`get()`方法永远返回`null`，所以程序员不能将一个**虚可达**的对象重新变成 **强/软/弱 可达**的。而虚引用的意义在于，被标记为可删除的对象就可以不再起作用了，例如，当重载过的`finalize()`方法可以清理资源时。

我不知道**虚引用**有什么样的用法，因为被**虚引用**引用的对象是不可达的。有一种情况是，如果你想在对象被标记为可删除之后再做些什么的话，你不应该把操作放在`finalize()`方法中（这样可能会有性能问题）。

## 总结

我希望，你现在对这些引用有了更好的理解。大多数时候，你不需要显示的使用它们（你也不应该使用）。但是，有很多框架在使用它们，如果你想了解框架的基本原理的话，你应该了解这些概念。

如果你喜欢看视频学习，Webucator 将这篇文章做成了视频，放在上[YouTube](https://www.youtube.com/watch?v=7fvCrEiao1A)上。

