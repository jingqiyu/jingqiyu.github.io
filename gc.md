## javaGC机制学习

#### java的GC和c++析构的区别


think in java中说道 

- 对象可能不被垃圾回收
- 垃圾回收不等同于“析构” 
 

说说 finalize() 的工作方式

垃圾回收发生时，系统会调用 finalize()方法， 并在下一次垃圾回收时，真正的释放对象的资源

但是finalize方法往往是用来释放非java代码控制的内存，比如使用java的本地方法调用了c中的malloc方法占用了某块内存，那么可能会需要来定义一个finalize方法来释放内存。不过官方是极其不推荐使用finalize方法进行内存释放的。这往往是不可预知和危险的。

---

垃圾回收可以提高其他对象的创建速度，听起来很奇怪，就像存储空间的释放影响了存储空间的分配。我们知道Java的对象是存储在堆上的。正是JVM特殊的机制，使得java在堆上分配内存的速度可以和其他语言在堆栈内分配内存的速度相媲美。


垃圾回收的几种方式

- 引用计数 （php中使用的方式、可能存在互相引用等问题，而且做GC回收时，需要对所有对象遍历循环释放、效率极低，java中并未使用此种技术进行垃圾回收）
- stop-copy 停止后复制技术。简单说实现上生成2个堆，找到活的对象并复制到另外一个堆中，并紧凑排列，使得堆指针能够指向排列后的位置
- mark-weep 标记清扫技术，并不会每次都复制所有的内存到新的堆中

java中采用的是自适应的方式。就是时而采用stop-copy， 时而采用mark-sweep的方式。

- 分代回收算法 


![image](http://ww3.sinaimg.cn/mw690/7178f37egw1etbmycakylj20fn08kgmb.jpg)

第一层也叫做年轻代 

1. 所有新生成的对象首先都是放在年轻代的。年轻代的目标就是尽可能快速的收集掉那些生命周期短的对象。
2. 新生代内存按照8:1:1的比例分为一个eden区和两个survivor(survivor0,survivor1)区。一个Eden区，两个Survivor区(一般而言)。大部分对象在Eden区中生成。**回收时先将eden区存活对象复制到一个survivor0区，然后清空eden区，当这个survivor0区也存放满了时，则将eden区和survivor0区存活对象复制到另一个survivor1区，然后清空eden和这个survivor0区，此时survivor0区是空的，然后将survivor0区和survivor1区交换，即保持survivor1区为空， 如此往复**。
3. 当survivor1区不足以存放 eden和survivor0的存活对象时，就将存活对象直接存放到老年代。若是老年代也满了就会触发一次Full GC，也就是新生代、老年代都进行回收


第二层年老代（Old Generation）

1.在年轻代中经历了N次垃圾回收后仍然存活的对象，就会被放到年老代中。因此，可以认为年老代中存放的都是一些生命周期较长的对象。

2.内存比新生代也大很多(大概比例是1:2)，当老年代内存满时触发Major GC即Full GC，Full GC发生频率比较低，老年代对象存活时间比较长，存活率标记高。


第三层持久代（Permanent Generation）

用于存放静态文件，如Java类、方法等。持久代对垃圾回收没有显著影响，但是有些应用可能动态生成或者调用一些class，例如Hibernate 等，在这种时候需要设置一个比较大的持久代空间来存放这些运行过程中新增的类。


HotSpot Jvm 1.6中采用的是如下算法

![image](http://ww1.sinaimg.cn/mw690/7178f37egw1etbmycjfvoj20e40engmi.jpg)


Serial收集器（复制算法)

新生代单线程收集器，标记和清理都是单线程，优点是简单高效。

Serial Old收集器(标记-整理算法)

老年代单线程收集器，Serial收集器的老年代版本。

ParNew收集器(停止-复制算法)　

新生代收集器，可以认为是Serial收集器的多线程版本,在多核CPU环境下有着比Serial更好的表现。

Parallel Scavenge收集器(停止-复制算法)

并行收集器，追求高吞吐量，高效利用CPU。吞吐量一般为99%， 吞吐量= 用户线程时间/(用户线程时间+GC线程时间)。适合后台应用等对交互相应要求不高的场景。

Parallel Old收集器(停止-复制算法)

Parallel Scavenge收集器的老年代版本，并行收集器，吞吐量优先

CMS(Concurrent Mark Sweep)收集器（标记-清理算法）

高并发、低停顿，追求最短GC回收停顿时间，cpu占用比较高，响应时间快，停顿时间短，多核cpu 追求高响应时间的选择




### GC的执行机制

由于对象进行了分代处理，因此垃圾回收区域、时间也不一样。GC有两种类型：Scavenge GC和Full GC。

- Scavenge GC

一般情况下，当新对象生成，并且在Eden申请空间失败时，就会触发Scavenge GC，对Eden区域进行GC，清除非存活对象，并且把尚且存活的对象移动到Survivor区。然后整理Survivor的两个区。这种方式的GC是对年轻代的Eden区进行，不会影响到年老代。因为大部分对象都是从Eden区开始的，同时Eden区不会分配的很大，所以Eden区的GC会频繁进行。因而，一般在这里需要使用速度快、效率高的算法，使Eden去能尽快空闲出来。

- Full GC

对整个堆进行整理，包括Young、Tenured和Perm。Full GC因为需要对整个堆进行回收，所以比Scavenge GC要慢，因此应该尽可能减少Full GC的次数。在对JVM调优的过程中，很大一部分工作就是对于FullGC的调节。有如下原因可能导致Full GC：

1.年老代（Tenured）被写满

2.持久代（Perm）被写满

3.System.gc()被显示调用

4.上一次GC之后Heap的各域分配策略动态变化
