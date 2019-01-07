---
categories: java
layout: post
---

本文翻译自[Java Garbage Collection Basics](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html)

- Table
{:toc}
# 描述垃圾回收

自动化垃圾回收是着眼于堆内存的过程，确认哪些对象正被使用，哪些没有，并删除不被使用的对象。一个被使用的对象，或称之为被引用对象，意味着在你的程序中的某处还保留了指向它的一个指针。一个不被使用的对象，或称之为不被引用对象，则是你的程序任何处都不再引用它。所以被这些不被引用对象所占据的内存就可以被回收。

再一个像C的编程语言中，分配和回收内存是一个手动的过程。在Java中，回收对象的过程是被垃圾回收器自动处理的。基本的过程如下描述：

## 步骤1：标记

过程的第一步称为标记，在这里垃圾回收器确定哪些内存片段被使用，哪些没有。

![](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/images/gcslides/Slide3.png)

被引用的对象显示为蓝色。不被已用的对象则显示为金色。在标记步骤所有的对象都会被扫描以做出决定。如果一个系统中的所有对象都必须被扫描，那么有可能会称为一个非常耗时的过程，

## 步骤2：通常删除

通常删除（Normal deletion）移除不被引用的对象，仅留下被引用的对象，以及指向空闲空间的指针。

![](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/images/gcslides/Slide1b.png)

内存分配器持有指向这些空闲空间块的引用，用于将来分配新的对象。

## 步骤2a：带整理的删除

为了进一步提高性能，除了删除不被引用的对象，你还可以整理剩下被引用的对象。通过移动被引用对象到一处，这将导致之后对空闲内存的分配更加快速和简单。

![](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/images/gcslides/Slide4.png)

## 为什么要分代进行垃圾回收

如早先所说，必须标记和整理一个JVM中所有中的对象是低效的。随着越来越多的对象被分配，对象列表不断扩大，导致了越来越久的垃圾回收过程。然而，对应用做经验分析，显示了大部分的对象非常短命。

下面是这样数据的一个样例。Y轴显示总共分配的字节数，X轴表示生存的时间长度。

![](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/images/ObjectLifetime.gif)

如你所见，随着生命周期的增长，剩下的对象越来越少。实际上，大部分对象就如图中左边高峰处一样非常短命。

## JVM分代

从对象分配行为中得到的信息可以用于增强JVM的性能。因此，堆被分作数个较小的部分—代。堆分为：年轻代（young generation），年老代（old generation）或称为终生代（tenured generation），以及永久代（permanent generation）。

![](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/images/gcslides/Slide5.png)

年轻代是所有对象创建和计算年龄的地方。当年轻代被耗尽，这导致了一次微垃圾回收（minor garbage collection）。在对象高死亡率的假设下微回收可以进行优化。一个填充满了死去对象的年轻代可以非常快速地进行回收。一些存活下来的对象被增大年纪并最终移动到年老代。

停止世界事件 — 所有的微回收都是“停止世界”事件。这意味着所有的应用线程都会被暂停直到操作完成。微回收总是停止世界事件。

年老代用于存储存活较久的对象。典型地，会为一个年轻代对象设置一个阈值，一旦年龄抵达这个阈值，该对象就会被移动到年老代。最终，年老代也会需要被回收，这样的事件称为主垃圾回收（major garbage collection）。

主垃圾回收也是停止世界事件。通常一次主回收会涉及所有存活对象，所以会慢得多。所以对于响应式应用，主回收应该尽量不触发。还要注意，主回收的停止世界事件的时长会受到年老代使用的垃圾回收器种类的影响。

永久代包含描述应用用到的类和方法所需的元数据。永久代由JVM在运行时用使用到的类进行装填。此外，Java SE库中的类和方法也可能会存储在其中。

如果JVM发现一些类不再被需要，且它的空间被其它类所需要，这些不再被使用的类可能会被回收。永久代包含在完整的垃圾回收中。

# 分代垃圾回收过程

现在你已经知道了为什么堆要切分为不同的代，正是时候看看这些空间实际上是怎样交互的。

1.第一，所有新的对象都在伊甸园空间（eden space）中分配。而另外两个幸存者空间（survivor space）初始时都是空的。

![](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/images/gcslides/Slide13.png)



2.当伊甸园空间被耗尽了，就会触发一次微垃圾回收。

![](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/images/gcslides/Slide14.png)



3.被引用对象首先会被移动到第一个幸存者空间。而未被引用的对象会在伊甸园清空的时候被删除。

![](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/images/gcslides/Slide6.png)

4.在下一次微回收时，相同的事情在伊甸园发生。未被引用的对象被删除且引用的对象被移动到一个幸存者空间。然而，这次它们被移动到第二个幸存者空间（S1）。除此之外，由第一次微回收移动到第一个幸存者空间（S0）中的对象的年龄会增加并且移动到S1。一旦所有幸存者对象都移动到了S1，S0和伊甸园会被一同清空。注意现在在幸存者空间中包含了不同年龄的对象。

![](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/images/gcslides/Slide8.png)

5.在下一次微回收，同样的过程重复发生。只是这一次幸存者空间切换了，引用的对象被移动到S0。之后幸存者对象年龄增大，伊甸园和S1被清空。

![](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/images/gcslides/Slide9.png)

6.下面的图展示了晋升过程。在一次微回收后，当对象达到了指定的年龄阈值（例子中为8），它们就会从年轻代晋升到年老代。

![](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/images/gcslides/Slide7.png)

7.就如同微回收继续发生，对象也将继续被晋升到年老代空间。



![](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/images/gcslides/Slide10.png)

8.上述完整流程在年轻代发生足够多次后。最终，在年老代触发了一次主回收，清理并整理了空间中的对象。

![](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/images/gcslides/Slide11.png)

# Java垃圾回收器

你已经了解了垃圾回收器的基础，并且通过一个样例实战观察过了垃圾回收器。在这一部分，你将会学到Java中可用的垃圾回收器，以及用于选择它们的命令行参数。

## 通用堆相关开关

存在许多与Java共同使用的不同的命令行开关。这一节描述其中最经常被使用的部分：

| 开关            | 描述                 |
| --------------- | -------------------- |
| -Xms            | 设置初始堆大小       |
| -Xmx            | 设置最大堆大小       |
| -Xmn            | 设置年轻代的大小     |
| -XX:PermSize    | 设置永久代的初始大小 |
| -XX:MaxPermSize | 设置永久代的最大大小 |

## 串行GC

对于使用Java SE5和6的客户端风格机器上执行的程序，默认使用的垃圾回收器是串行GC。利用串行化会收集，微回收和主回收被序列化执行（使用一个单独的虚拟CPU）。此外，它使用了标记整理回收方法。这个方法移动旧内存到堆的开始处，因此新内存的分配则是一个堆末尾的一个连续的内存块。这样内存的整齐使得为堆分配新的内存块十分快速。

### 用例

串行化GC是大多数不要求低延迟且运行在客户端风格机器上的应用的选择。它使用一个单独的虚拟处理器进行垃圾回收工作。在如今的硬件上，串行GC能有效地管理大量的带数百MB堆的重要应用，最坏情况下仅停止较短时间（一次全局垃圾回收大约数秒）。

另外一个串行GC的适用环境是在同一台机器上运行大量JVM进程。在这样的环境下，一旦JVM要进行GC，那么最好仅使用一个处理器来最小化与其它JVM的冲突，即使垃圾回收可能会变得很久。以及串行GC适合这种权衡。

### 命令行开关

要启用串行回收器：

```sh
-XX:+UseSerialGC
```

## 并行GC

并行垃圾回收器利用多线程执行年轻代垃圾回收。默认在一个带N核CPU的主机上，并行垃圾回收器使用N个线程进行垃圾回收。垃圾回收器的线程数可以通过命令行参数指定：

```sh
-XX:ParallelGCThreads=<desired number>
```

### 用例

并行GC也称为吞吐量回收器。由于它可以使用多个CPU来提高应用的吞吐量。当大量工作需要被处理并且较长的暂停是可接受的，那么就应该使用该收集器。比如，批量处理像打印报告或票据或执行多次数据库查询。

### 命令行开关

```sh
-XX:+UseParallelGC
```

使用这个命令行选项，你会得到一个年轻代并行垃圾回收器，以及一个年老代串行带整理回收器。

```sh
-XX:+UseParallelOldGC
```

带上这个选项，在年轻代和年老代都将使用并行带整理垃圾回收器。

## 并发标记清理（CMS）收集器

并发标记清理回收器（CMS）（也称为并发低延迟回收器）回收终生代。通过与应用线程并行运行，它尽可能最小化了由于垃圾回收造成的暂停。通常并发低延迟回收器不拷贝和整理存活的对象。一次垃圾回收不会移动存活对象。如果内存碎片成了一个问题，则会分配一个更大的堆。

注意：在年轻代CMS收集器使用与并行GC相同的算法。

### 用例

CMS收集器应该被用于在要求低延迟的应用。比如响应交互事件的桌面UI应用，响应请求的Web服务器，响应查询的数据库。

### 命令行开关

开启CMS收集器：

```sh
-XX:+UseConcMarkSweepGC
```

设置并发线程数

```sh
-XX:ParallelCMSThreads=<n>
```

## G1垃圾回收器

G1垃圾回收器在Java7中可用，并作为CMS回收器的长期替代物。G1回收器是并行的，并发的，并且增量整理。然而，细节讨论不在本文范围内。

### 命令行开关

启动G1回收器：

```sh
-XX:UseG1GC
```
