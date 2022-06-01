---
layout: post
title:  "【多线程与高并发原理篇：4_深入理解synchronized】"
date:   2022-06-01 19:45:10 +0800
categories: cnblog
---
**目录**

- 1. 前言
- 2. 从顶层到底层
    - 2.1 应用层
    - 2.2 字节码层面
    - 2.3 java虚拟机规范层面
    - 2.4 操作系统层面
    - 2.5 hotspot实现层面
    - 2.6 汇编层
    - 2.7 CPU指令层
- 3. synchronized锁优化
- 4. hotspot关于synchronized的源码分析
- 5. 总结

* * *
 
回到顶部

## 1. 前言
 
越是简单的东西，在深入了解后发现越复杂。想起了曾在初中阶段，语文老师给我们解说《论语》的道理，顺便给我们提了一句，说老子的无为思想比较消极，学生时代不要太关注。现在有了一定的生活阅历，再来看老子的《道德经》，发现那才是大智慧，《论语》属于儒家是讲人与人的关系，《道德经》属于道家讲人与自然的关系，效法天之道来行人之道，儒家讲入世，仁义礼智信，道家讲出世，无为而无不为。老子把道比作水、玄牝（女性的生殖器）、婴儿、山谷等，高深莫测，却滋养万物，源源不断的化生万物并滋养之，而不居功，故能天长地久。儒家教我们做圣人，道家教我们修成仙，显然境界更高。
 
`synchronized`的设计思想就像道家的思想一样，看着用起来很简单，但是底层极其复杂，好像永远看不透一样。一直想深入写一篇`synchronized`的文章，却一直不敢动手，直到最近读了几遍hotspot源码后，才有勇气写一些自己的理解。下文就从几个层面逐步深入，谈谈对`synchronized`的理解，总结精华思想并用到我们自己的项目设计中。
 
回到顶部

## 2. 从顶层到底层
 
首先通过一张图来概览synchronized在各层面的实现细节与原理，并在下面的章节逐一分析  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220527153338644-902398345.png
 
### 2.1 应用层
 
**临界区与临界资源**
 
一段代码块内如果存在对共享资源的多线程读写操作，称这段代码块为临界区，其共享资源为临界资源。举例如下：
  
以上的结果可能是正数、负数、零。为什么呢？因为 Java 中对静态变量的自增、自减并不是原子操作，个线程执行过程中会被其他线程打断。
 
**竞态条件**  
 多个线程在临界区内执行，由于代码的执行序列不同而导致结果无法预测，称之为发生了竞态条件。为了避免临界区的竞态条件发生，有多种手段可以达到目的：

- 阻塞式的解决方案：synchronized，Lock
- 非阻塞式的解决方案：原子变量

synchronized 同步块是 Java 提供的一种原子性`内置锁`，Java 中的每个对象都可以把它当作一个同步锁来使用，这些 Java 内置的使用者看不到的锁被称为内置锁，也叫作监视器锁，目的就是保证多个线程在进入synchronized代码段或者方法时，将并行变串行。
 
**使用层面**  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220526205336885-1006774184.png
 
解决之前的共享问题，只需要在两个方法前面加上synchronized，就可以得到期望为零的结果。
  
### 2.2 字节码层面
 
通过IDEA自带的工具view-&gt;show bytecode with jclasslib，查看到访问标志0x0029  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220526220250407-538873815.png
 
通过查询java 字节码手册，synchronized对应的字节码为ACC\_SYNCHRONIZED，0x0020，三个修饰符加起来正好是0x0029。  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220526215534591-1433369360.png
 
如果加在代码块或者对象上， 对应的字节码是`monitorenter`与`monitorexit`。  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220526221633361-1903302692.png
 
### 2.3 java虚拟机规范层面
 
**同步方法**  
 The Java® Virtual Machine Specification针对同步方法的说明

> Method-level synchronization is performed implicitly, as part of method invocation and return. A synchronized method is distinguished in the run-time constant pool’s method\_info structure by the ACC\_SYNCHRONIZED flag, which is checked by the method invocation instructions. When invoking a method for which ACC\_SYNCHRONIZED is set, the executing thread enters a monitor, invokes the method itself, and exits the monitor whether the method invocation completes normally or abruptly. During the time the executing thread owns the monitor, no other thread may enter it. If an exception is thrown during invocation of the synchronized method and the synchronized method does not handle the exception, the monitor for the method is automatically exited before the exception is rethrown out of the synchronized method.

这段话适合好好读读，大致含义如下：

> 同步方法是隐式的。一个同步方法会在运行时常量池中的method\_info结构体中存放ACC\_SYNCHRONIZED标识符。当一个线程访问方法时，会去检查是否存在ACC\_SYNCHRONIZED标识，如果存在，则先要获得对应的`monitor锁`，然后执行方法。当方法执行结束(不管是正常return还是抛出异常)都会释放对应的`monitor锁`。如果此时有其他线程也想要访问这个方法时，会因得不到monitor锁而阻塞。当同步方法中抛出异常且方法内没有捕获，则在向外抛出时会先释放已获得的monitor锁。

**同步代码块**
 
同步代码块使用monitorenter和monitorexit两个指令实现同步， The Java® Virtual Machine Specification中有关于这两个指令的介绍：

> Each object is associated with a monitor. A monitor is locked if and only if it has an owner. The thread that executes monitorenter attempts to gain ownership of the monitor associated with objectref, as follows:  
>  If the entry count of the monitor associated with objectref is zero, the thread enters the monitor and sets its entry count to one. The thread is then the owner of the monitor.  
>  If the thread already owns the monitor associated with objectref, it reenters the monitor, incrementing its entry count.  
>  If another thread already owns the monitor associated with objectref, the thread blocks until the monitor's entry count is zero, then tries again to gain ownership.

大致含义如下：

> 每个对象都会与一个monitor相关联，当某个monitor被拥有之后就会被锁住，当线程执行到monitorenter指令时，就会去尝试获得对应的monitor。步骤如下：  
>  1.每个monitor维护着一个记录着拥有次数的计数器。未被拥有的monitor的该计数器为0，当一个线程获得monitor（执行monitorenter）后，该计数器自增变为 1 。
> 
> - 当同一个线程再次获得该monitor的时候，计数器再次自增；
> - 当不同线程想要获得该monitor的时候，就会被阻塞。

> 2.当同一个线程释放 monitor（执行monitorexit指令）的时候，计数器再自减。当计数器为0的时候。monitor将被释放，其他线程便可以获得monitor。

### 2.4 操作系统层面
 
**Monitor（管程/监视器）**  
 monitor又称监视器，操作系统层面为`管程`，管程是指管理共享变量以及对共享变量操作的过程，让它们支持并发，所以管程是操作系统层面的同步机制。
 
**MESA模型**  
 在管程的发展史上，先后出现过三种不同的管程模型，分别是Hasen模型、Hoare模型和MESA模型。现在正在广泛使用的是MESA模型。下面我们便介绍MESA模型  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220530113330490-138945955.png
 
管程中引入了条件变量的概念，而且每个条件变量都对应有一个等待队列。条件变量和等待队列的作用是解决线程之间的同步问题。
 
synchronized关键字和wait()、notify()、notifyAll()这三个方法是Java中实现管程技术的组成部分，设计思想也是来源与操作系统的管程。在应用层面有个经典的`等待通知范式`。
 
**等待方**  
 等待方遵循如下的原则：

- 获取对象的锁
- 如果条件不满足，那么调用对象的wait方法，被通知后仍然需要检查条件
- 条件满足则继续执行对应的逻辑

**通知方**  
 通知方遵循如下的原则：

- 获得对象的锁
- 改变条件
- 通知所有等待在对象上的线程

唤醒的时间和获取到锁继续执行的时间是不一致的，被唤醒的线程再次执行时可能条件又不满足了，所以循环检验条件。MESA模型的wait()方法还有一个超时参数，为了避免线程进入等待队列永久阻塞。
 
**Java语言的内置管程synchronized**  
 Java 参考了 MESA 模型，语言内置的管程（synchronized）对 MESA 模型进行了精简。MESA模型中，条件变量可以有多个，ReentrantLock可以实现多个条件变量以及对应的队列，Java 语言内置的管程里只有一个条件变量。模型如下图所示。  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220530114228253-1086850882.png
 
**内核态**  
 jdk1.5之前，synchronized是重量级锁，会直接由操作系统内核态控制，jdk1.6以后，对synchronized做了大量的优化，如锁粗化（Lock Coarsening）、锁消除（Lock Elimination）、轻量级锁（Lightweight Locking）、偏向锁（Biased Locking）、自适应自旋（Adaptive Spinning）等技术来减少锁操作的开销，内置锁的并发性能已经基本与Lock持平，只有升级到重量级锁后才会有大的开销。
 
**等待通知范式案例**  
 用上面讲的等待通知范式，实现一个数据库连接池；连接池有获取连接，释放连接两个方法，DBPool模拟一个容器放连接，初始化20个连接，获取连接的线程得到连接池为空时，阻塞等待，唤起释放连接的线程；
   
测试类：总共50个线程，每个线程尝试20次
  
结果：  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220530203525993-1380361008.png
 
### 2.5 hotspot实现层面
 
hotspot源码ObjectMonitor.hpp定义了ObjectMonitor的数据结构
  
大致执行过程如下如所示：  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220527165214491-752849598.png  
 大量线程进入监视器的等待队列EntryList，只有通过CAS拿到锁的线程，把进入锁的标识变量\_recursions置为1，如果方法是递归循环调用，支持锁重入，\_recursions可以累加，并将\_owner置为获取锁的线程ID，调用wait方法后，释放锁，再重新进入等待队列EntryList。
 
**对象的内存布局**  
 上述过程可以看到，加锁是加在对象头上的，Hotspot虚拟机中，对象在内存中存储的布局可以分为三块区域：对象头（Header）、实例数据  
 （Instance Data）和对齐填充（Padding）。

- 对象头：比如 hash码，对象所属的年代，对象锁，锁状态标志，偏向锁（线程）ID，偏向时间，数组长度（数组对象才有）等。
- 实例数据：存放类的属性数据信息，包括父类的属性信息；
- 对齐填充：由于虚拟机要求， 对象头字节大小必须是8字节的整数倍。填充数据不是必须存在的，仅仅是为了字节对齐。  

https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220528160306951-694557563.png

**对象头详解**  
 HotSpot虚拟机的对象头包括：

- Mark Word  

用于存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等，这部分数据的长度在32位和64位的虚拟机中分别为32bit和64bit，官方称它为"Mark Word"。
- Klass Pointer  

对象头的另外一部分是klass类型指针，即对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。 32位4字节，64位开启指针压缩或最大堆内存&lt;32g时4字节，否则8字节。jdk1.8默认开启指针压缩后为4字节，当在JVM参数中关闭指针压缩（-XX:-UseCompressedOops）后，长度为8字节。
- 数组长度（只有数组对象有）如果对象是一个数组，那在对象头中还必须有一块数据用于记录数组长度，开启压缩时，数组占4字节，不开启压缩数组占8字节，因为数组本质上也是指针。

**使用JOL工具查看内存布局**  
 给大家推荐一个可以查看普通java对象的内部布局工具JOL(JAVA OBJECT LAYOUT)，使用此工具可以查看new出来的一个java对象的内部布局,以及一个普通的java对象占用多少字节。引入maven依赖
  
使用方法：
  
测试
  
利用jol查看64位系统java对象（空对象），默认开启指针压缩，总大小显示16字节，前12字节为对象头。  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220528163248054-1121876726.png

- OFFSET：偏移地址，单位字节；
- SIZE：占用的内存大小，单位为字节；
- TYPE DESCRIPTION：类型描述，其中object header为对象头；
- VALUE：对应内存中当前存储的值，二进制32位；

关闭指针压缩后，对象头为16字节：­XX:­UseCompressedOops  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220528163707895-733250329.png
 
**Mark Word的结构**  
 Hotspot通过markOop类型实现Mark Word，具体实现位于markOop.hpp文件中。MarkWord 结构搞得这么复杂，是因为需要节省内存，让同一个内存区域在不  
 同阶段有不同的用处。
 
- hash： 保存对象的哈希码。运行期间调用System.identityHashCode()来计算，延迟计算，并把结果赋值到这里。
- age： 保存对象的分代年龄。表示对象被GC的次数，当该次数到达阈值的时候，对象就会转移到老年代。
- biased\_lock： 偏向锁标识位。由于无锁和偏向锁的锁标识都是 01，没办法区分，这里引入一位的偏向锁标识位。
- lock： 锁状态标识位。区分锁状态，比如11时表示对象待GC回收状态, 只有最后2位锁标识有效。
- JavaThread\*： 保存持有偏向锁的线程ID。偏向模式的时候，当某个线程持有对象的时候，对象这里就会被置为该线程的ID。 在后面的操作中，就无需再进行尝试获取锁的动作。这个线程ID并不是JVM分配的线程ID号，和Java Thread中的ID是两个概念。
- epoch： 用来计算偏向锁的批量撤销与批量重偏向。

32位JVM下的对象结构描述  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220528190729059-365333926.png  
 64位JVM下的对象结构描述  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220528190744600-1540348103.png

- ptr\_to\_lock\_record：轻量级锁状态下，指向栈中锁记录的指针。当锁获取是无竞争时，JVM使用原子操作而不是OS互斥，这种技术称为轻量级锁定。在轻量级锁定的情况下，JVM通过CAS操作在对象的Mark Word中设置指向锁记录的指针。
- ptr\_to\_heavyweight\_monitor：重量级锁状态下，指向对象监视器Monitor的指针。如果两个不同的线程同时在同一个对象上竞争，则必须将轻量级锁定升级到Monitor以管理等待的线程。在重量级锁定的情况下，JVM在对象的ptr\_to\_heavyweight\_monitor设置指向Monitor的指针。

**Mark Word中锁标记枚举**
  
更直观的理解方式  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220528191216949-1120247820.png
 
**偏向锁**  
 在大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，对于没有锁竞争的场合，偏向锁有很好的优化效果。
  
当JVM启用了偏向锁模式（jdk6默认开启），新创建对象的Mark Word中的Thread Id为0，说明此时处于可偏向但未偏向任何线程，也叫做匿名偏向状态(anonymously biased)
 
**偏向锁延迟偏向**  
 偏向锁模式存在偏向锁延迟机制：HotSpot 虚拟机在启动后有个 4s 的延迟才会对每个新建的对象开启偏向锁模式。因为JVM启动时会进行一系列的复杂活动，比如装载配置，系统类初始化等等。在这个过程中会使用大量synchronized关键字对对象加锁，且这些锁大多数都不是偏向锁。待启动完成后再延迟打开偏向锁。
   
https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220528194556610-1888411897.png  
 5s后偏向锁为可偏向或者匿名偏向状态，此时ThreadId=0；
 
**偏向锁状态跟踪**
  
https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220528201450359-821992831.png  
 上图可以看出，只有一个线程时，从匿名偏向到偏向锁，并在偏向锁后面带上了线程id。
 
**思考：如果对象调用了hashCode,还会开启偏向锁模式吗？**  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220528201846401-879102739.png  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220528203005499-170144001.png  
 **偏向锁撤销之调用对象HashCode**  
 匿名偏向后，调用对象HashCode，导致偏向锁撤销。因为对于一个对象，其HashCode只会生成一次并保存，偏向锁是没有地方保存hashcode的。

- 轻量级锁会在锁记录中记录 hashCode
- 重量级锁会在 Monitor 中记录 hashCode  

当对象处于匿名偏向（也就是线程ID为0）和已偏向的状态下，调用HashCode计算将会使对象再也无法偏向。
- 当对象匿名偏向，MarkWord将变成未锁定状态，并只能升级成轻量锁；
- 当对象正处于偏向锁时，调用HashCode将使偏向锁强制升级成重量锁。  

如下图  

https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220528203952487-558678048.png  

https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220528204803172-1951463971.png  

https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220528204814392-993825686.png

**偏向锁撤销之调用wait/notify**  
 偏向锁状态执行obj.notify() 会升级为轻量级锁，调用obj.wait(timeout) 会升级为重量级锁
  
https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220528205619644-1934663869.png
  
https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220528205624482-1779422099.png
 
**轻量级锁**  
 倘若偏向锁失败，虚拟机并不会立即升级为重量级锁，它还会尝试使用一种称为轻量级锁的优化手段，此时Mark Word 的结构也变为轻量级锁的结构。轻量级锁所适应的场景是线程交替执行同步块的场合，如果存在同一时间多个线程访问同一把锁的场合，就会导致轻量级锁膨胀为重量级锁。
 
**轻量级锁跟踪**
  
https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220528211859593-543506564.png
 
**偏向锁升级轻量级锁**  
 模拟两个线程轻度竞争场景
  
https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220529113148557-53128936.png  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220529113158709-1302322572.png
 
**轻量级锁膨胀为重量级锁**
  
将上述控制线程竞争时机的代码注掉，让线程2与线程1发生竞争，线程2就由原来的偏向锁升级到重量级锁。  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220529151520001-1062050929.png
 
下面思考几个问题：  
 `思考1：重量级锁释放之后变为无锁，此时有新的线程来调用同步块，会获取什么锁？`

> 通过实验可以得出，后面的线程会获得轻量级锁，相当于线程竞争不激烈，多个线程通过CAS就能轮流获取锁，并且释放。

https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220529150327464-184883137.png
 
`思考2：为什么有轻量级锁还需要重量级锁？`

> 因为轻量级锁时通过CAS自旋的方式获取锁，该方式消耗CPU资源的，如果锁的时间长，或者自旋线程多，CPU会被大量消耗；而重量级锁有等待队列，所有拿不到锁的进入等待队列，不需要消耗CPU资源。

`思考3：偏向锁是否一定比轻量级锁效率高吗？`

> 不一定，在明确知道会有多线程竞争的情况下，偏向锁肯定会涉及锁撤销，需要暂停线程，回到安全点，并检查线程释放活着，故撤销需要消耗性能，这时候直接使用轻量级锁。  
>  JVM启动过程，会有很多线程竞，所以默认情况启动时不打开偏向锁，过一段儿时间再打开。

**锁升级的状态图**  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220529160839043-1136703616.png  
 无锁是锁升级前的一个中间态，必须要恢复到无锁才能进行升级，因为需要有拷贝mark word的过程，并且修改指针。
 
**锁记录的重入**  
 轻量级锁在拷贝mark word到线程栈Lock Record中时，如果有重入锁，则在线程栈中继续压栈Lock Record记录，只不过mark word的值为空，等到解锁后，依次弹出，最终将mard word恢复到对象头中，如图所示  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220529162536338-450425262.png
 
锁升级的具体细节会稍后结合hotspot源码进行。
 
### 2.6 汇编层
 
上面谈到的锁升级，一直提到了一个CAS操作，CAS是英文单词CompareAndSwap的缩写，中文意思是：比较并替换。CAS需要有3个操作数：内存地址V，旧的预期值A，即将要更新的目标值B。
 
CAS指令执行时，当且仅当内存地址V的值与预期值A相等时，将内存地址V的值修改为B，否则就什么都不做。整个比较并替换的操作是一个原子操作。
 
这个方法是java native方法，在unsafe类中，
  
需要打开hotspot的unsafe.cpp
  
可以看到调用了Atomic::cmpxchg方法，Atomic::cmpxchg方法引入了汇编指令，
 
- mp是os::is\_MP()的返回结果，os::is\_MP()是一个内联函数，用来判断当前系统是否为多处理器。如果当前系统是多处理器，该函数返回1。否则，返回0。
- \_\_asm\_\_代表是汇编开始，volatile代表，禁止CPU指令重排，并且让值修改后，立马被其他CPU可见，保持数据一致性。
- LOCK\_IF\_MP(mp)会根据mp的值来决定是否为cmpxchg指令添加lock前缀。如果通过mp判断当前系统是多处理器（即mp值为1），则为cmpxchg指令添加lock前缀。否则，不加lock前缀。

内嵌汇编模板
  
这里涉及操作内存，与CPU的寄存器，大致意思就是在先判断CPU是否多核，如果是多核，禁止CPU级别的指令重排，并且通过Lock前缀指令，锁住并进行比较与交换，然后把最新的值同步到内存，其他CPU从内存加载数据取最新的值。
 
### 2.7 CPU指令层
 
上面LOCK\_IF\_MP在atomic\_linux\_x86.inline.hpp有详细宏定义
  
很明显，带了一个lock前缀的指令，lock 和cmpxchgl是CPU指令，lock指令是个前缀，可以修饰其他指令，cmpxchgl即为CAS指令，查阅英特尔操作手册，在Intel® 64 and IA-32 Architectures Software Developer’s Manual 中的章节LOCK—Assert LOCK# Signal Prefix 中给出LOCK指令的详细解释  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220529172400813-96604793.png  
 就是排他的使用共享内存。这里一般有两种方式，锁总线与锁cpu的缓存行

- 锁总线  

LOCK#信号就是我们经常说到的总线锁，处理器使用LOCK#信号达到锁定总线，来解决原子性问题，当一个处理器往总线上输出LOCK#信号时，其它处理器的请求将被阻塞，此时该处理器此时独占共享内存；总线锁这种做法锁定的范围太大了，导致CPU利用率急剧下降，因为使用LOCK#是把CPU和内存之间的通信锁住了，这使得锁定时期间，其它处理器不能操作其内存地址的数据 ，所以总线锁的开销比较大。
- 锁缓存行  

如果访问的内存区域已经缓存在处理器的缓存行中，P6系统和之后系列的处理器则不会声明LOCK#信号，它会对CPU的缓存中的缓存行进行锁定，在锁定期间，其它 CPU 不能同时缓存此数据，在修改之后，通过缓存一致性协议（在Intel CPU中，则体现为MESI协议）来保证修改的原子性，这个操作被称为缓存锁

lock指令会产生总线锁也可能会产生缓存锁，看具体的条件，有下面几种情况只能使用总线锁

- 当操作的数据不能被缓存在处理器内部，这个必须得使用总线锁了
- 操作的数据跨多个缓存行（cache line），缓存锁的前置条件是多个数据在一个缓存行里面
- 有些处理器不支持缓存锁定。对于Intel 486和奔腾处理器，就算锁定的内存区域在处理器的缓存行中也会调用总线锁定。

无论是总线锁还是缓存锁这都是CPU在硬件层面上提供的锁，肯定效率比软件层面的锁要高。
 
回到顶部

## 3. synchronized锁优化
 
**偏向锁批量重偏向&批量撤销**  
 从偏向锁的加锁解锁过程中可看出，当只有一个线程反复进入同步块时，偏向锁带来的性能开销基本可以忽略，但是当有其他线程尝试获得锁时，就需要等到safe point时，再将偏向锁撤销为无锁状态或升级为轻量级，会消耗一定的性能，所以在`多线程竞争频繁的情况下，偏向锁不仅不能提高性能，还会导致性能下降`。于是，就有了批量重偏向与批量撤销的机制。
 
**原理**
 
以class为单位，为每个class维护一个偏向锁撤销计数器，每一次该class的对象发生偏向撤销操作时，该计数器+1，当这个值达到重偏向阈值（默认20）时，JVM就认为该class的偏向锁有问题，因此会进行批量重偏向。
 
每个class对象会有一个对应的epoch字段，每个处于偏向锁状态对象的Mark Word中也有该字段，其初始值为创建该对象时class中的epoch的值。每次发生批量重偏向时，就将该值+1，同时遍历JVM中所有线程的栈，找到该class所有正处于加锁状态的偏向锁，将其epoch字段改为新值。下次获得锁时，发现当前对象的epoch值和class的epoch不相等，那就算当前已经偏向了其他线程，也不会执行撤销操作，而是直接通过CAS操作将其Mark Word的Thread Id 改成当前线程Id。
 
当达到重偏向阈值（默认20）后，假设该class计数器继续增长，当其达到批量撤销的阈值后（默认40），JVM就认为该class的使用场景存在多线程竞争，会标记该class为不可偏向，之后，对于该class的锁，直接走轻量级锁的逻辑。
 
**应用场景**
 
批量重偏向（bulk rebias）机制是为了解决：一个线程创建了大量对象并执行了初始的同步操作，后来另一个线程也来将这些对象作为锁对象进行操作，这样会导致大量的偏向锁撤销操作。批量撤销（bulk revoke）机制是为了解决：在明显多线程竞争剧烈的场景下使用偏向锁是不合适的。
 
**JVM的默认参数值**
 
设置JVM参数-XX:+PrintFlagsFinal，在项目启动时即可输出JVM的默认参数值
  
我们可以通过-XX:BiasedLockingBulkRebiasThreshold 和 -XX:BiasedLockingBulkRevokeThreshold 来手动设置阈值  
 **测试：批量重偏向**
  
当撤销偏向锁阈值超过 20 次后，jvm 会这样觉得，`我是不是偏向错了`，于是会在给这些对象加锁时重新偏向至加锁线程，`重偏向会重置象 的 Thread ID`  
 测试结果：  
 thread1:  创建50个偏向线程thread1的偏向锁 1-50 偏向锁  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220529212428587-973199037.png  
 thread2：  
 1-18 偏向锁撤销，升级为轻量级锁  （thread1释放锁之后为偏向锁状态）  
 19-40 偏向锁撤销达到阈值（20），执行了批量重偏向 （测试结果在第19就开始批量重偏向了）  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220529212520907-519709669.png  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220529212526095-1779175272.png  
 **测试：批量撤销**  
 当撤销偏向锁阈值超过 40 次后，jvm 会认为不该偏向，于是整个类的所有对象都会变为不可偏向的，新建的对象也是不可偏向的。  
 thread3：  
 1-18  从无锁状态直接获取轻量级锁  （thread2释放锁之后变为无锁状态）  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220529213338597-1886140780.png
 
19-40 偏向锁撤销，升级为轻量级锁   （thread2释放锁之后为偏向锁状态）  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220529213457474-315492136.png  
 41-50   达到偏向锁撤销的阈值40，批量撤销偏向锁，升级为轻量级锁 （thread1释放锁之后为偏向锁状态）  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220529213611033-760964212.png  
 新创建的对象： 无锁状态  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220529213826016-1156529366.png
 
**总结**

- 批量重偏向和批量撤销是针对类的优化，和对象无关；
- 偏向锁重偏向一次之后不可再次重偏向；
- 当某个类已经触发批量撤销机制后，JVM会默认当前类产生了严重的问题，剥夺了该类的新实例对象使用偏向锁的权利。

**自旋优化**
 
量级锁竞争的时候，还可以使用自旋来进行优化，如果当前线程自旋成功（即这时候持锁线程已经退出了同步块，释放了锁），这时当前线程就可以避免阻塞。

- 自旋会占用 CPU 时间，单核 CPU 自旋就是浪费，多核 CPU 自旋才能发挥优势。
- 在 Java 6 之后自旋是自适应的，比如对象刚刚的一次自旋操作成功过，那么认为这次自旋成功的可能性会高，就多自旋几次；反之，就少自旋甚至不自旋，比较智能。
- Java 7 之后不能控制是否开启自旋功能。  

`注意：自旋的目的是为了减少线程挂起的次数，尽量避免直接挂起线程（挂起操作涉及系统调用，存在用户态和内核态切换，这才是重量级锁最大的开销）`

**锁粗化**  
 假设一系列的连续操作都会对`同一个对象反复加锁及解锁，甚至加锁操作是出现在循环体中的`，即使没有出现线程竞争，频繁地进行互斥同步操作也会导致不必要的性能损耗。如果JVM检测到有一连串零碎的操作都是对同一对象的加锁，将会扩大加锁同步的范围（即锁粗化）到整个操作序列的外部。
  
上述代码每次调用 buffer.append 方法都需要加锁和解锁，如果JVM检测到有一连串的对同一个对象加锁和解锁的操作，就会将其合并成一次范围更大的加锁和解锁操作，即在第一次append方法时进行加锁，最后一次append方法结束后进行解锁。
 
**锁消除**  
 锁消除即删除不必要的加锁操作。锁消除是Java虚拟机在JIT编译期间，通过对运行上下文的扫描，去除不可能存在共享资源竞争的锁，通过锁消除，可以节省毫无意义的请求锁时间。  
 StringBuffer的append是个同步方法，但是append方法中的 StringBuffer 属于一个局部变量，不可能从该方法中逃逸出去，因此其实这过程是线程安全的，可以将锁消除。
  
通过比较实际，开启锁消除用时4秒多，未开启锁消除用时6秒多。
 
测试代码：
  
通过测试发现，开启逃逸分析后，线程实例总共50万个，只有8万多个在堆中，其他都在栈上分配；  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220530115058588-1338378215.png
 
关闭逃逸分析后50万全部都在堆中。  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220530115106206-588384807.png
 
回到顶部

## 4. hotspot关于synchronized的源码分析
 
**首先准备好HotSpot源码**  
 jdk8 hotspot源码下载地址：[http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/，选择gz或者zip包下载。](http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/%EF%BC%8C%E9%80%89%E6%8B%A9gz%E6%88%96%E8%80%85zip%E5%8C%85%E4%B8%8B%E8%BD%BD%E3%80%82)  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220530210023989-294128516.png
 
目录结构  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220530210840115-2087627720.png

- cpu：和cpu相关的一些操作
- os：在不同操作系统上的一些区别操作
- os\_cpu：关联os和cpu的实现
- share：公共代码

share下面还有两个目录

- tools：一些工具类
- vm：公共源码

然后使用Source Insight工具打开源码工程，如下图  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220530212522225-774220496.png
 
首先看入口InterpreterRuntime:: monitorenter方法
  
看到上面注解`Retry fast entry if bias is revoked to avoid unnecessary inflation`，意思就是如果偏向锁打开，就直接进入ObjectSynchronizer的fast\_enter方法，避免不必要的膨胀，否则进入slow\_enter方法，由此可知偏向锁执行fast\_enter方法，锁的升级则进入slow\_enter方法。
 
接着进入synchronizer.cpp
   
首先看`ObjectMonitor`生成的过程。
  
ObjectMonitor生成后，进入ObjectMonitor.enter方法，重量级加锁的逻辑都是在这里完成的。
  
上面步骤总结为

- 1.通过CAS尝试将\_owner变量设置为当前线程
- 2.如果是线程重入(下面有举例），则将\_recurisons++
- 3.如果线程是第一次进入，则将\_recurisons设置为1，将\_owner设置为当前线程，该线程获取锁成功并返回
- 4.如果获取锁失败，则等待锁的释放

接着进入所等待源码  
 在锁竞争源码中最后一步，如果获取锁失败，则等待锁的释放，由MonitorObject类中的EnterI()方法来实现
  
步骤分为如下几步：

- 1.首先tryLock再次尝试获取锁，之后再CAS尝试获取锁；失败后将当前线程信息封装成ObjectWaiter对象；
- 2.在for(;;)循环中，通过CAS将该节点插入到\_cxq链表的头部（这个时刻可能有多个获取锁失败的线程要插入），CAS插入失败的线程再次尝试获取锁；
- 3.如果还没获取到锁，则将线程挂起；等待唤醒；
- 4.当线程被唤醒时，再次尝试获取锁。

这里的`设计精髓是通过多次tryLock尝试获取锁和CAS获取锁无限推迟了线程的挂起操作`，你可以看到从开始到线程挂起的代码中，出现了多次的尝试获取锁；因为线程的挂起与唤醒涉及到了状态的转换(内核态和用户态），这种频繁的切换必定会给系统带来性能上的瓶颈。所以它的设计意图就是尽量推辞线程的挂起时间，取一个极限的时间挂起线程。另外源码中定义了负责线程\_Responsible，这种标识的线程调用的是定时的park(线程挂起），避免死锁.
 
锁释放源码
  
上述代码总结如下步骤：

- 1.将\_recursions减1，\_owner置空；
- 2.如果队列中等待的线程为空或者\_succ不为空（有"醒着的线程"，则不需要去唤醒线程了），直接返回即可；
- 3.第二条不满足，当前线程重新获取锁，去唤醒线程；
- 4.唤醒线程，根据QMode的不同，有不同的唤醒策略；

> QMode = 2且cxq非空：cxq中有优先级更高的线程，直接唤醒\_cxq的队首线程;  
>  QMode = 3且cxq非空：把cxq队列插入到EntryList的尾部；  
>  QMode = 4且cxq非空：把cxq队列插入到EntryList的头部；  
>  QMode = 0：暂时什么都不做，继续往下看；

只有QMode=2的时候会提前返回，等于0、3、4的时候都会继续往下执行：
 
`这里的设计精髓是首先就将锁释放，然后再去判断是否有醒着的线程，因为可能有线程正在尝试或者自旋获取锁，如果有线程活着，需要再让该线程重新获取锁去唤醒线程。`
 
最后通过流程两张图把创建ObjectMonitor对象的过程与进入enter方法加锁与解锁的过程呈现出来，把我主要流程，更多地代码细节也可以从源码的英文注解中得到答案。  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220531215007244-1616335415.png  
 https://img2022.cnblogs.com/blog/1765702/202205/1765702-20220531215014848-1022177471.png
 
回到顶部

## 5. 总结
 
通过本篇由浅入深，逐步分析了synchronized从各层角度实现的细节以及原理，我们可以从中学到一些思路，比如锁的设计，能够通过CAS，偏向锁，轻量级锁等方式来实现的时候，尽量不要升级到重量级锁，等到竞争太大浪费cpu开销的时候，才引入重量级锁；比如synchronized原子性是通过锁对象保证只有一个线程访问临界资源来实现，可见性通过源码里的汇编volatile结合硬件底层指令实现，有序性通过源码底层的读写屏障并借助于硬件指令完成。synchronized底层在锁优化的时候也用了大量的CAS操作，提升性能；以及等待队列与阻塞队列设计如何同步进入管程的设计，这些设计思想也是后面Reentrantlock设计的中引入条件队列的思想，`底层都是相同的，任何应用层的软件设计都能从底层的设计思想与精髓实现中找到原型与影子`，当看到最底层C、C++或者汇编以及硬件指令级别的实现时，一切顶层仿佛就能通透。


> 作者:小猪爸爸  
> 原文:https://www.cnblogs.com/father-of-little-pig/p/16314318.html  
