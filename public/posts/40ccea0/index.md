# G1


# G1

G1也是属于并发分代收集器。

堆区域：
&#43; 年轻代区域
&#43; Eden区域 - 新分配的对象 - TLAB
&#43; Survivor区域 - 年轻代GC存活后但不需要晋升的对象 - RSet
&#43; 老年代区域
&#43; 晋升到老年代的对象
&#43; 直接分配到老年代的大对象，占用多个区域的对象，通常是大于Region的一半
![image.png](https://minio.dionysunrsshub.top/myimages/2024-img/20240624143329.png)

G1的Collector可以分为两个大部分：
&#43; **全局并发标记 (global concurrent marking)**
&#43; **拷贝存活对象(evacuation)**

其分代收集模式又有，区别在于选定的CSet：
&#43; **Young GC**
&#43; **Mixed GC**
&#43; **Full GC**
前两者都是标记-复制作为回收算法

其中的几个关键技术有：
&#43; 停顿预测模型
&#43; [TLAB](#TLAB &#34;wikilink&#34;)
&#43; RSet
&#43; SATB &amp; SATB MarkQueue &amp; Write barrier
&#43; Safe Point

## 全局并发标记

全局并发标记是基于SATB形式的并发标记，具体来说，其分为下面几个阶段：
1. **初始标记(Initial Marking)**: `STW` 扫描GC roots集合，标记所有从根集合可直接到达的对象并将它们的字段压入扫描栈 (marking stack)中等待后续扫描。G1使用外部的bitmap来记录mark信息，而不是用对象头的mark word里的mark bit。在分代式G1模式中，初始标记阶段借用Young GC的暂停，即没有额外、单独的STW。并且在**Mixed GC**中，还会根据Youn GC扫描后的Survivor的RSet作为根更新待扫描的GC Roots，将指向的对象同样压入扫描栈。
2. **并发标记(Concurrent marking)**: `Concurrent` 不断从扫描栈取出引用递归扫描整个堆里的对象图。每扫描到一个对象就会对其标记，并将其字段压入扫描栈。重复扫描过程直到扫描栈清空。过程中还会扫描SATB write barrier所记录下的引用。
3. **最终标记（final marking，在实现中也叫remarking）**: `STW` 在完成并发标记后，每个Java线程还会有一些剩下的SATB write barrier记录的引用尚未处理。这个阶段就负责把剩下的引用处理完。同时这个阶段也进行弱引用处理（reference processing）。注意这个暂停与CMS的remark有一个本质上的区别，那就是这个暂停只需要扫描SATB buffer，而CMS的remark需要重新扫描mod-union table里的dirty card外加整个根集合，而此时整个young gen（不管对象死活）都会被当作根集合的一部分，因而CMS remark有可能会非常慢
4. **清理（cleanup）**: `STW` 清点和重置标记状态。这个阶段有点像mark-sweep中的sweep阶段，不过不是在堆上sweep实际对象，而是在marking bitmap里统计每个region被标记为活的对象有多少。这个阶段如果发现完全没有活对象的region就会将其整体回收到可分配region列表中。

## 拷贝存活对象

Evacuation阶段是全暂停的。它负责把一部分region里的活对象拷贝到空region里去，然后回收原本的region的空间。  
Evacuation阶段可以自由选择任意多个region来独立收集构成收集集合（collection set，简称CSet），靠per-region remembered set（简称RSet）实现。这是regional garbage collector的特征。  
在选定CSet后，evacuation其实就跟ParallelScavenge的young GC的算法类似，采用并行copying（或者叫scavenging）算法把CSet里每个region里的活对象拷贝到新的region里，整个过程完全暂停。从这个意义上说，G1的evacuation跟传统的mark-compact算法的compaction完全不同：前者会自己从根集合遍历对象图来判定对象的生死，不需要依赖global concurrent marking的结果，有就用，没有拉倒；而后者则依赖于之前的mark阶段对对象生死的判定。

注意，该部分也与初始标记阶段相同，复用了Young GC的代码。

# TLAB

TLAB的核心思想在于优化各个线程从堆内存中新建对象的过程，通过估算每轮GC内每个线程使用的内存大小，提前分配好内存给线程，提高分配效率，避免由于撞针分配及CAS操作带来的性能损失，称之为TLAB(Thread Local Allocate Buffer)。*注意，TLAB*是线程私有的。

## JVM 对象堆内存分配流程简述

具体来说，当分配一个对象堆内存空间时(`new Object()`)，`CollectedHeap`首先检查是否启用了TLAB，如果启用了，则会尝试TLAB分配 - 如果当前线程的 TLAB 大小足够，那么从线程当前的 TLAB 中分配；如果不够，但是当前 TLAB 剩余空间小于**最大浪费空间限制**，则从堆上（一般是 Eden 区） 重新申请一个新的 TLAB 进行分配。否则，直接在 TLAB 外进行分配。TLAB 外的分配策略，不同的 GC 算法不同。例如G1：
1. 对于超过1/2 region大小的对象，直接分配在Humongous区域
2. 根据当前用户线程的状态进行region下标的分配

## G1 与 TLAB 的关联

new新对象 -\&gt; TLAB分配 -\&gt; TLAB不可容纳 -\&gt; TLAB外进行分配 -\&gt; 从Eden区获取新的TLAB -\&gt; Eden区不够 -\&gt; 判断回收时间，是触发Young GC还是将空闲分区加入Eden区再次分配

GC回收 -\&gt; 将Eden Region变为free region -\&gt; 更改相应TLAB参数，重新分配 -\&gt; 循环进行 `new 新对象`
![draw.io.drawio.png](https://minio.dionysunrsshub.top/myimages/2024-img/draw.io.drawio.png)

## TLAB的生命周期

TLAB 是线程私有的，**线程初始化的时候**，会创建并初始化 TLAB。同时，在 **GC 扫描对象发生之后，线程第一次尝试分配对象的时候**，也会创建并初始化 TLAB，即再分配。 TLAB 生命周期停止（TLAB 声明周期停止不代表内存被回收，只是代表这个 TLAB 不再被这个线程私有管理，即可以通过EMA计算大小后再分配）在：

- 当前 TLAB 不够分配，并且剩余空间小于**最大浪费空间限制**，那么这个 TLAB 会被退回 Eden，重新申请一个新的
- 发生 GC 的时候，TLAB 被回收。

==最大浪费空间可以看作是可容忍的`internal fragmentation`大小，即小于这个数值的内存碎片，是可以被容忍的==

### 直接从堆上分配

直接从堆上分配是最慢的分配方式。一种情况就是，如果当前 TLAB 剩余空间大于当前**最大浪费空间限制**，直接在堆上分配。并且，还会增加当前最大浪费空间限制，**每次有这样的分配就会增加 TLABWasteIncrement 的大小**，这样在一定次数的直接堆上分配之后，当前最大浪费空间限制一直增大会导致当前 TLAB 剩余空间小于当前**最大浪费空间限制**，从而申请新的 TLAB 进行分配。

## 返还TLAB给Eden区时，填充Dummy Object的必要性

当GC扫描时，此时的TLAB要返还给Eden区，即堆区域。

具体而言，填充Dummy Object实际发生在GC之前，以G1举例，此时要先确保堆内存是**可解析的**，即将所有线程的TLAB填充Dummy Object后，返还给堆，核心在于更快速的扫描堆上的对象，以及采样一些东西利于接下来TLAB大小的计算。

考虑不填充dummy object，此时堆内存中对于已经存在的对象不会存在影响，但是对于未使用的部分，GC线程并不知道其中是否会有对象，就会逐字节的扫描，影响效率。

填充Dummy Object是以对象头中的Mark word的一个`int[]`数组决定的，因此不能超过int\[\]数组的最大大小

## TLAB的再分配

通过EMA采样(Exponential Moving Averange, 指数平均数)，得到新的TLAB大小期望值，该期望值与Eden区的总大小有关，在[\#G1 与 TLAB 的关联](#G1 与 TLAB 的关联 &#34;wikilink&#34;)中，概述了这个Eden区的大小的变化过程

具体来说，EMA算法的核心在于**最小权重**，即**最小权重越大**，变化得越快，受**历史数据影响越小**。

每个线程 TLAB 初始大小 = `Eden区大小` / (`线程单个 GC 轮次内最多从 Eden 申请多少次 TLAB` \* `当前 GC 分配线程个数 EMA`)

GC 后，重新计算 TLAB 大小 = `Eden区大小` / (`线程单个 GC 轮次内最多从 Eden 申请多少次 TLAB` \* `当前 GC 分配线程个数 EMA`)

# RSet - Remember Set

G1将堆内存划分为大小相等的region，新创建的对象都是放在新生代的Eden区。为了加速Initial Marking阶段中的GC Roots根扫描阶段，引入了RSet这一概念，具体来说，RSet存储了Region间的引用关系，主要是记录了如下两种：
&#43; Old Region -\&gt; Young Region
&#43; Old Region -\&gt; Old Region

## 内部数据结构

### Card Table 卡表

卡表就是将Region进一步划分为若干个物理连续的512Byte的Card Page，这样每个Region就都有一个Card Table来映射，并且整个堆空间也有一个Global Card Table

### Sparse PRT 稀疏哈希表

&lt;figure&gt;
&lt;img
src=&#34;https://minio.dionysunrsshub.top/myimages/2024-img/20240624164003.png&#34;
alt=&#34;image.png&#34; /&gt;
&lt;figcaption aria-hidden=&#34;true&#34;&gt;image.png&lt;/figcaption&gt;
&lt;/figure&gt;

此方法内存开销较大，进一步缩减，得到细粒度PRT
### 细粒度 PRT

通过一个bit位来表示卡表
![image.png](https://minio.dionysunrsshub.top/myimages/2024-img/20240624164059.png)
### 粗粒度位图

再度优化细粒度PRT的内存，每个bit位表示一个Region

&lt;figure&gt;
&lt;img
src=&#34;https://minio.dionysunrsshub.top/myimages/2024-img/20240624164142.png&#34;
alt=&#34;image.png&#34; /&gt;
&lt;figcaption aria-hidden=&#34;true&#34;&gt;image.png&lt;/figcaption&gt;
&lt;/figure&gt;

# Refine线程

Refine线程的核心功能在于：
&#43; 处理新生代分区的抽样 - 更新Young Heap Region的数目
&#43; 使G1满足GC的预测停顿时间`-XX:MaxGCPauseMillis`
&#43; 管理RSet
&#43; 更新RSet
&#43; 将G1中更新的引用关系从DCQS - Dirty Card Queue Set 中更新到RSet中
&#43; 每个线程都有一个私有的DCQ，而DCQS是全局静态变量
&#43; 并发、异步处理

# SATB &amp; SATB MarkQueue &amp; Write barrier

SATB，SnapShot-At-The-Beginning，是维护并发GC的正确性的一个手段，G1 GC并发理论基础就是SATB。

抽象来说，G1 GC认为在一次GC开始的时候是活的对象就被认为是活的（GC Roots搜索），此时的对象图形成一个逻辑快照（SnapShot），然后在GC过程中新分配的对象都当作是活的，其他不可到达的对象就是死的。通过每个Region记录着的Top-At-Mark-Start，TAMS，指针，分别为`prevTAMS`和`nextTAMS`，在TAMS以上的对象就是新分配的。但是在并发GC中，Collector和Mutator线程都在进行，如果collector并发mark的过程中mutator覆盖了某些引用字段的值而collector还没mark到那里，那collector就得不到完整的snapshot了，因此，引入了SATB Write Barrier来解决这个问题。

## SATB Write Barrier

Write barrier是对&#34;对引用类型字段赋值&#34;这个动作的环切，也就是说赋值的前后都在barrier覆盖的范畴内。在赋值前的部分的write barrier叫做pre-write barrier，在赋值后的则叫做post-write barrier。

前面提到SATB要维持&#34;在GC开始时活的对象&#34;的状态这个逻辑snapshot。除了从root出发把整个对象图mark下来之外，其实只需要用pre-write barrier把每次引用关系变化时旧的引用值记下来就好了。这样，等concurrent marker到达某个对象时，这个对象的所有引用类型字段的变化全都有记录在案，就不会漏掉任何在snapshot里活的对象。当然，很可能有对象在snapshot中是活的，但随着并发GC的进行它可能本来已经死了，但SATB还是会让它活过这次GC。

## SATB Mark Queue

为了尽量减少write barrier对mutator性能的影响，G1将一部分原本要在barrier里做的事情挪到别的线程上并发执行。  
实现这种分离的方式就是通过logging形式的write barrier：mutator只在barrier里把要做的事情的信息记（log）到一个队列里，然后另外的线程从队列里取出信息批量完成剩余的动作。

以SATB write barrier为例，每个Java线程有一个独立的、定长的SATBMarkQueue，mutator在barrier里只把old_value压入该队列中。一个队列满了之后，它就会被加到全局的SATB队列集合SATBMarkQueueSet里等待处理，然后给对应的Java线程换一个新的、干净的队列继续执行下去。

并发标记（concurrent marker）会定期检查全局SATB队列集合的大小。当全局集合中队列数量超过一定阈值后，concurrent marker就会处理集合里的所有队列：把队列里记录的每个oop都标记上，并将其引用字段压到标记栈（marking stack）上等后面做进一步标记。

这个队列与DCQ的区别在于，前者是Refine线程处理，用于在堆的日常运维中追踪被修改的内存区域，优化垃圾收集的效率，SATB mark queue是mutator线程协助处理，用于记录并发标记阶段开始时对象的引用状态，确保标记的完整性和一致性。并且两者处理的数据类型和GC过程中的角色不一样：SATB Mark Queue 处理对象引用，特别是在堆修改前的状态；Dirty Card Queue 处理的是堆内存中的区域（card），这些区域在被修改时被标记为脏；SATB Mark Queue 在并发标记阶段发挥作用，帮助实现堆状态的一致性快照；Dirty Card Queue 在整个垃圾收集过程中都可能被使用，用于标记和处理那些自上次垃圾收集以来发生变化的堆区域。

# 选取CSet的子模式

## Young GC

选定所有young gen里的region。通过控制young gen的region个数来控制young GC的开销（Refine线程所做的事）

- 触发时机：

  - 新的对象创建会放入Eden区
  - Eden区满、G1会根据停顿预测模型-计算当前Eden区GC大概耗时多久
  - 如果回收时间远 \&lt; -XX:MaxGCPauseMills,则分配空闲分区加入Eden 区存放
  - 如果回收时间接近-XX:MaxGCPauseMills，则触发一次Young GC

- 年轻代初始占总堆5%，随着空闲分区加入而增加，最多不超过60%

- Young GC 会回收all新生代分区 - Eden区和Survivor 区

- Young GC 会STW(stop the world),暂停所有用户线程

- GC后重新调整新生代Region数目，每次GC回收Region数目不固定

- 回收过程：

  - 扫描根 - GC Roots
  - 更新RSet - Refine线程
  - 处理RSet - 扫描RSet
  - 复制对象 - Evacuation
  - 处理引用 - 重构RSet及卡表

## Mixed GC

选定所有young gen里的region，外加根据global concurrent marking统计得出收集收益高的若干old gen region。在用户指定的开销目标范围内尽可能选择收益高的old gen region。

- 回收过程
  - 复用 Young GC 的 Initial Marking，并根据Survivor区中的RSet再次根扫描，压入扫描栈
  - 并发标记阶段 - 从扫描栈与SATB MarkQueue中递归进行标记
  - 再标记阶段 - STW，完全处理SATB MarkQueue
  - 清理阶段 - 确认回收对象
  - 复制对象阶段 - 复用 Young GC 中的复制代码
    \## 分代式G1

分代式G1的正常工作流程就是在young GC与mixed GC之间视情况切换，背后定期做做全局并发标记。Initial marking默认搭在young GC上执行；当全局并发标记正在工作时，G1不会选择做mixed GC，反之如果有mixed GC正在进行中G1也不会启动initial marking。 在正常工作流程中没有full GC的概念，old gen的收集全靠mixed GC来完成。

如果mixed GC实在无法跟上程序分配内存的速度，导致old gen填满无法继续进行mixed GC，就会切换到G1之外的serial old GC来收集整个GC heap（注意，包括young、old、perm）。这才是真正的full GC。Full GC之所以叫full就是要收集整个堆，只选择old gen的部分region算不上full GC。进入这种状态的G1就跟-XX:&#43;UseSerialGC的full GC一样（背后的核心代码是两者共用的）。  
顺带一提，G1 GC的System.gc()默认还是full GC，也就是serial old GC。只有加上 -XX:&#43;ExplicitGCInvokesConcurrent 时G1才会用自身的并发GC来执行System.gc()------此时System.gc()的作用是强行启动一次global concurrent marking；一般情况下暂停中只会做initial marking然后就返回了，接下来的concurrent marking还是照常并发执行。

# Safe Point

正如 Safe Point 名称的寓意一样，Safe Point 是一个线程可以安全停留在这里的代码点。当我们需要进行 GC 操作的时候，JVM 可以让所有线程在 Safe Point 处停留下来，等到所有线程都停在 Safe Point 处时，就可以进行内存引用分析，从而确定哪些对象是存活的、哪些对象是不存活的。

为什么让大家更加场景化地理解 Safe Point 这个概念，可以设想如下场景：

1.  当需要 GC 时，需要知道哪些对象还被使用，或者已经不被使用可以回收了，这样就需要每个线程的对象使用情况。
2.  对于偏向锁（Biased Lock），在高并发时想要解除偏置，需要线程状态还有获取锁的线程的精确信息。
3.  对方法进行即时编译优化（OSR 栈上替换），或者反优化（bailout 栈上反优化），这需要线程究竟运行到方法的哪里的信息。

对于上面这些操作，都需要知道现场的各种信息，例如寄存器有什么内容，堆使用情况等等。在做这些操作的时候，线程需要暂停，等到这些操作完成才行，否则会有并发问题，这就需要 Safe Point 的存在。

**因此，我们可以将 Safe Point 理解成代码执行过程中的一些特殊位置，当线程执行到这个位置时，线程可以暂停。** Safe Point 处保存了其他位置没有的一些当前线程信息，可以提供给其他线程读取，这些信息包括：线程上下文信息，对象的内部指针等。

**而 Stop the World 就是所有线程同时进入 Safe Point 并停留在那里，等待 JVM 进行内存分析扫描，接着进行内存垃圾回收的时间。**

## 为啥需要 Safe Point

前面我们说到，Safe Point 其实就是一个代码的特殊位置，在这个位置时线程可以暂停下来。而当我们进行 GC 的时候，所有线程都要进入到 Safe Point 处，才可以进行内存的分析及垃圾回收。根据这个过程，其实我们可以看到：**Safe Point 其实就是栅栏的作用，让所有线程停下来，否则如果所有线程都在运行的话，JVM 无法进行对象引用的分析，那么也无法进行垃圾回收了。**

此外，另一个重要的 Java 线程特性 ------ interrupted 也是根据 Safe Point 实现的。当我们在代码里写入 `Thread.interrupt()` 时，只有线程运行到 Safe Point 处时才知道是否发生了 interrupted。**因此，Safe Point 也承担了存储线程通信的功能。**

# GC 日志打印

- 打印基本GC信息
  - `-XX:&#43;PrintGCDetails -XX:PrintGCDateStamps`
- 打印对象分布 - 根据Age
  - `-XX:&#43;PrintTenuringDistribution`
- GC后打印堆数据
  - `-XX:&#43;PrintHeapAtGC`
- 打印STW时间
  - `-XX:&#43;PrintGCApplicationStoppedTime`
- 打印 Safe Point 信息
  - `-XX:&#43;PringSafepointStatistics -XX:PrintSafepointStatisticsCount=1`
- 打印 Reference 处理信息
  - `-XX:&#43;PrintReferenceGC`
- 日志分割
  - `-Xloggc:/path/tp/gc.log` - GC日志输出的文件路径
  - `-XX:UseGCLogFileRotation` - 开启日志文件分割
- 时间戳命名文件
  - `-XX:PrintGCDetails -XX:&#43;PrintGCDataStamps -Xloggc:/path/to/gc-%t.log`

&gt; &#43; https://www.cnblogs.com/chanshuyi/p/head-first-of-jvm-safe-point.html
&gt; &#43; https://segmentfault.com/a/1190000039411521
&gt; &#43; https://juejin.cn/post/6949885566536138783?searchId=202406240957236755114659D501068D8D
&gt; &#43; https://blog.csdn.net/m0_63437643/article/details/122601042
&gt; &#43; https://tech.meituan.com/2016/09/23/g1.html
&gt; &#43; https://hllvm-group.iteye.com/group/topic/44381#post-272188
&gt; &#43; https://www.zhihu.com/question/53613423/answer/135743258
&gt; &#43; https://blog.csdn.net/oJieSi/article/details/134758659

&lt;!--more--&gt;


---

> Author: Shiping Guo  
> URL: http://localhost:1313/posts/40ccea0/  

