# 多线程基础


# 多线程基础

## 并发与并行

### 并发执行

![bingfa](https://minio.dionysunrsshub.top:443/myimages/2024-img/bingfa.png)
同一时间只能处理一个任务，每个任务轮着做（时间片轮转），只要我们单次处理分配的时间足够的短，在宏观看来，就是三个任务在同时进行。
而我们Java中的线程，正是这种机制，当我们需要同时处理上百个上千个任务时，很明显CPU的数量是不可能赶得上我们的线程数的，所以说这时就要求我们的程序有良好的并发性能，来应对同一时间大量的任务处理

### 并行执行
![bingxing](https://minio.dionysunrsshub.top/myimages/2024-img/bingxing.png)
突破了同一时间只能处理一个任务的限制，同一时间可以做多个任务，比如分布式计算模型MapReduce

## 锁机制

使用`synchronized`关键字来实现锁，其一定是和某个对象关联的，即提供一个对象来作为锁本身，究其根本在于每个对象的对象头信息中的`Mark Word`

在将`synchronized`实现锁的代码变成字节码后，我们发现，其调用了`monitorenter`和`monitorexit`指令，分别对应加锁和释放锁，且其通过两次`monitorexit`来实现异常处理![monitor](https://minio.dionysunrsshub.top/myimages/2024-img/monitor.png)

对于`Mark Word`，其在不同状态下，它存储的数据结构不同：
![objectHead](https://minio.dionysunrsshub.top/myimages/2024-img/objectHead.png)
\^markword

### 重量级锁

在JDK6之前，`synchronized`被称为重量级锁，`monitor`依赖于底层操作系统的Lock实现，Java的线程是映射到操作系统的原生线程上，切换成本较高

每一个对象都有一个`monitor`相关联，在JVM中，`monitor`是由`ObjetMonitor`实现的：

    ObjectMonitor() {
        _header       = NULL;
        _count        = 0; //记录个数
        _waiters      = 0,
        _recursions   = 0;
        _object       = NULL;
        _owner        = NULL;
        _WaitSet      = NULL; //处于wait状态的线程，会被加入到_WaitSet
        _WaitSetLock  = 0 ;
        _Responsible  = NULL ;
        _succ         = NULL ;
        _cxq          = NULL ;
        FreeNext      = NULL ;
        _EntryList    = NULL ; //处于等待锁block状态的线程，会被加入到该列表
        _SpinFreq     = 0 ;
        _SpinClock    = 0 ;
        OwnerIsThread = 0 ;
    }

而每个等待锁的线程都会被封装成ObjectWaiter对象，进入如下机制：
![synchronized1](https://minio.dionysunrsshub.top/myimages/2024-img/synchronized1.png)
ObjectWaiter首先会进入 Entry Set等待，当线程获取到对象的`monitor`后进入 The Owner 区域并把`monitor`中的`owner`变量设置为当前线程，同时`monitor`中的计数器`count`加1，若线程调用`wait()`方法，将释放当前持有的`monitor`，`owner`变量恢复为`null`，`count`自减1，同时该线程进入 WaitSet集合中等待被唤醒。若当前线程执行完毕也将释放`monitor`并复位变量的值，以便其他线程进入获取对象的`monitor`

但是在大多数应用上，每一个线程占用同步代码块的时间并不是很长，完全没有必要将竞争中的线程挂起然后又唤醒，并且现代CPU基本都是多核心运行的，因此引入了自旋锁
![synchronized2](https://minio.dionysunrsshub.top/myimages/2024-img/synchronized2.png)

对于自旋锁，它不会将处于等待状态的线程挂起，而是通过无限循环的方式不断检测是否能获取锁，并且在等待时间太长的情况下，为了避免CPU继续运算循环浪费资源，会升级为重量级锁机制，自旋的次数限制是可以自适应变化的，比如在同一个锁对象上，自旋等待刚刚成功获得过锁，并且持有锁的线程正在运行，那么这次自旋也是有可能成功的，所以会允许自旋更多次。当然，如果某个锁经常都自旋失败，那么有可能会不再采用自旋策略，而是直接使用重量级锁

### 轻量级锁

轻量级锁的目标是，在无竞争的情况下减少重量级锁的性能消耗（赌一手同一时间只有一个线程在占用资源），不向操作系统申请互斥量等

在即将开始执行同步代码块中的内容时，会首先检查对象的`Mark Word`，查看锁对象是否被其他线程占用，如果没有任何线程占用，那么会在当前线程中所处的栈帧中建立一个名为锁记录（Lock Record）的空间，用于复制并存储对象目前的Mark Word信息（官方称为Displaced Mark Word），
接着，虚拟机将使用CAS操作将对象的Mark Word更新为轻量级锁状态（数据结构变为指向Lock Record的指针，指向的是当前的栈帧）

如果CAS操作失败了的话，那么说明可能这时有线程已经进入这个同步代码块了，这时虚拟机会再次检查对象的`Mark Word`，是否指向当前线程的栈帧，如果是，说明不是其他线程，而是当前线程已经有了这个对象的锁，直接放心大胆进同步代码块即可。如果不是，那确实是被其他线程占用了。
这时，轻量级锁一开始的想法就是错的（这时有对象在竞争资源，已经赌输了），所以说只能将锁膨胀为重量级锁，按照重量级锁的操作执行（注意锁的膨胀是不可逆的）
![lightLock](https://minio.dionysunrsshub.top/myimages/2024-img/lightLock.png)

解锁过程同样采取CAS算法，如果对象的MarkWord仍然指向线程的锁记录，那么就用CAS操作把对象的MarkWord和复制到栈帧中的Displaced Mark Word进行交换。如果替换失败，说明其他线程尝试过获取该锁，在释放锁的同时，需要唤醒被挂起的线程。

总体来说，流程为：**轻量级锁 -\&gt; 失败 -\&gt; 自适应自旋锁 -\&gt; 失败 -\&gt; 重量级锁**

&gt; \[!NOTE\] 无锁机制
&gt; 在并发执行过程中，Double-Check、自旋等待&#43;CAS修改是在不获取重量锁，即OS相关的线程操作时，保证原子性和正确性的重要手段，在[AQS](#队列同步器AQS源码分析 &#34;wikilink&#34;)中也是如此

### 偏向锁

偏向锁比轻量级锁更纯粹，实际上是专门为单个线程而生的，当某个线程第一次获得锁时，如果接下来都没有其他线程获取此锁，那么持有锁的线程将不再需要进行同步操作。可以从之前的[Mark Word](#^markword &#34;wikilink&#34;)结构中看到，偏向锁也会通过CAS操作记录线程的ID，如果一直都是同一个线程获取此锁，那么完全没有必要在进行额外的CAS操作。当然，如果有其他线程来抢了，那么偏向锁会根据当前状态，决定是否要恢复到未锁定或是膨胀为轻量级锁。

所以，最终的锁等级为：未锁定 \&lt; 偏向锁 \&lt; 轻量级锁 \&lt; 重量级锁

值得注意的是，如果对象通过调用`hashCode()`方法计算过对象的一致性哈希值，那么它是不支持偏向锁的，会直接进入到轻量级锁状态，因为Hash是需要被保存的，而偏向锁的Mark Word数据结构，无法保存Hash值；如果对象已经是偏向锁状态，再去调用`hashCode()`方法，那么会直接将锁升级为重量级锁，并将哈希值存放在`monitor`（有预留位置保存）中。
![biasLock](https://minio.dionysunrsshub.top/myimages/2024-img/biasLock.png)

### 锁消除和锁粗化

锁消除和锁粗化都是在运行时的一些优化方案，比如我们某段代码虽然加了锁，但是在运行时根本不可能出现各个线程之间资源争夺的情况，这种情况下，完全不需要任何加锁机制，所以锁会被消除。锁粗化则是我们代码中频繁地出现互斥同步操作，比如在一个循环内部加锁，这样明显是非常消耗性能的，所以虚拟机一旦检测到这种操作，会将整个同步范围进行扩展

## JMM内存模型

JVM中的内存模型是虚拟机规范对整个内存区域的规划，而Java内存模型，是在JVM内存模型之上的抽象模型，具体实现依然是基于JVM内存模型实现的

### Java内存模型

Java中采取了与OS中相似的高速缓存与主内存的解决方式，通过`Save`和`Load`操作实现缓存一致性协议
![JMM](https://minio.dionysunrsshub.top/myimages/2024-img/JMM.png)

JMM（Java Memory Model）内存模型规定如下：

- 所有的变量全部存储在主内存（注意这里包括下面提到的变量，指的都是会出现竞争的变量，包括成员变量、静态变量等，而局部变量这种属于线程私有，不包括在内）
- 每条线程有着自己的工作内存（可以类比CPU的高速缓存）线程对变量的所有操作，必须在工作内存中进行，不能直接操作主内存中的数据。
- 不同线程之间的工作内存相互隔离，如果需要在线程之间传递内容，只能通过主内存完成，无法直接访问对方的工作内存。

也就是说，每一条线程如果要操作主内存中的数据，那么得先拷贝到自己的工作内存中，并对工作内存中数据的副本进行操作，操作完成之后，也需要从工作副本中将结果拷贝回主内存中，具体的操作就是`Save`（保存）和`Load`（加载）操作。

结合JVM中的内存规划，有：
- 主内存：对应堆中存放对象的实例的部分。
- 工作内存：对应线程的虚拟机栈的部分区域，虚拟机可能会对这部分内存进行优化，将其放在CPU的寄存器或是高速缓存中。比如在访问数组时，由于数组是一段连续的内存空间，所以可以将一部分连续空间放入到CPU高速缓存中，那么之后如果我们顺序读取这个数组，那么大概率会直接缓存命中。

### 重排序

在编译或执行时，为了优化程序的执行效率，编译器或处理器常常会对指令进行重排序，有以下情况：

1.  编译器重排序：Java编译器通过对Java代码语义的理解，根据优化规则对代码指令进行重排序。
2.  机器指令级别的重排序：现代处理器很高级，能够自主判断和变更机器指令的执行顺序。

在多线程情况下就会有抢占和顺序的问题

### `volatile`关键字

首先引入三个概念：
- 原子性：就是要做什么事情要么做完，要么就不做，不存在做一半的情况。
- 可见性：指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。
- 有序性：即程序执行的顺序按照代码的先后顺序执行。

一个代码案例：

``` java
public class Main {
    private static int a = 0;
    public static void main(String[] args) throws InterruptedException {
        new Thread(() -&gt; {
            while (a == 0);
            System.out.println(&#34;线程结束！&#34;);
        }).start();

        Thread.sleep(1000);
        System.out.println(&#34;正在修改a的值...&#34;);
        a = 1;   //很明显，按照我们的逻辑来说，a的值被修改那么另一个线程将不再循环
    }
}
```

在该案例中，虽然主线程中修改了a的值，但是另一个线程并不知道a的值发生了改变，所以循环中依然是使用旧值在进行判断，因此，普通变量是不具有可见性的。

一种情况是对于该代码块进行`synchronized`加锁，因为其也符合[\#Happens-Before原则](#Happens-Before原则 &#34;wikilink&#34;)，会添加相应的[\#内存屏障(Memory Barriers)](#内存屏障(Memory Barriers) &#34;wikilink&#34;)保证可见性

另一种解决方法是使用`volatile`关键字。此关键字的第一个作用，就是保证变量的可见性。当写一个`volatile`变量时，JMM会把该线程本地内存中的变量强制刷新到主内存中去，并且这个写会操作会导致其他线程中的`volatile`变量缓存无效，这样，另一个线程修改了这个变量时，当前线程会立即得知，并将工作内存中的变量更新为最新的版本；但是该关键字无法解决原子性问题，因为在底层字节码实现中是拆分为多个CPU指令执行的。该关键字的第二个作用就是禁止指令重排，保证有序性，具体来说，是通过在指令序列中插入`内存屏障(Memory Barriers)`来禁止特定类型的处理器重排序。

**我们可以认为，`volatile`和`synchronized`以相同的方式解决了数据可见性，指令有序性，但是后者解决了原子性，代价为获取锁的额外开销**

&gt; \[!NOTE\] volatile结合内存屏障实现的可见性
&gt; 在 Java 中，对 `volatile` 变量的读操作确保了所有写入该变量的操作对其他线程可见。这是通过在读操作后加入 `LoadLoad` 和 `LoadStore` 内存屏障来实现的，这些屏障确保对 `volatile` 变量的读取不会从缓存中获取过时的数据。下面是这两个内存屏障工作机制的具体解释：
&gt;
&gt; `LoadLoad` 屏障
&gt; &#43; **作用**: `LoadLoad` 屏障放在 `volatile` 读操作之后，确保所有后续的读操作（包括对 `volatile` 和非 `volatile` 变量的读取）必须在读取 `volatile` 变量之后执行。这样的排序确保了对 `volatile` 变量的读取操作完成后，任何依赖于该变量的读取都能观察到其最新值。
&gt; - **实现**: 在处理器层面，这个屏障防止处理器将后续的读指令重新排序到 `volatile` 读之前，从而保证了内存操作的正确顺序。
&gt;
&gt; `LoadStore` 屏障
&gt; - **作用**: `LoadStore` 屏障确保在读取 `volatile` 变量之后的任何写操作必须等到 `volatile` 变量读取完成后才能执行。这保证了任何基于 `volatile` 变量读取结果的写操作都不会过早地执行，从而维护了读写之间的依赖关系。
&gt; - **实现**: 这个屏障阻止处理器将读取 `volatile` 变量后的写操作提前到读操作之前，确保了依赖于 `volatile` 变量的状态的写操作正确地观察到了 `volatile` 读取的最新结果。
&gt;
&gt; 保证不读取过时的数据
&gt;
&gt; 当线程进行 `volatile` 变量的读取时，`LoadLoad` 和 `LoadStore` 屏障一起工作，确保了以下几点：
&gt; - CPU 在执行读操作前必须先确认任何可能的写操作已同步到主内存，这通常涉及到刷新或检查本地缓存行的状态，确保它们与主内存保持一致。
&gt; - 如果本地缓存行被标记为无效（因为其他处理器已经修改了对应于 `volatile` 变量的内存地址），则处理器必须从主内存重新加载数据，而不是使用缓存中的过时数据。
&gt; 这些内存屏障的合作最终保证了 `volatile` 读操作总是从主内存中获取最新数据，而不是从可能包含过时数据的本地缓存中读取。这样的机制是 `volatile` 变量能够作为轻量级同步机制提供内存可见性和有序性保证的关键。

### 内存屏障(Memory Barriers)

内存屏障（Memory Barrier）又称内存栅栏，是一个CPU指令，它的作用有两个：
1. 保证特定操作的顺序
由于编译器和处理器都能执行指令重排的优化，如果在指令间插入一条Memory Barrier则会告诉编译器和CPU，不管什么指令都不能和这条Memory Barrier指令重排序。
2. 保证某些变量的内存可见性（volatile的内存可见性，其实就是依靠这个实现的）
Memory Barrier的另外一个作用是强制刷出各种CPU的缓存数据，因此任何CPU上的线程都能读取到这些数据的最新版本。通常涉及到刷新一些本地处理器缓存中的值到主内存，或者无效化缓存项，迫使后续的读操作从主内存中重新加载数据。

内存屏障主要分为以下几类：

1.  **LoadLoad**：确保之前的读取操作完成后，后续的读取操作才能进行。
2.  **StoreStore**：确保之前的写入操作完成后，后续的写入操作才能进行。
3.  **LoadStore**：确保之前的读取操作完成后，后续的写入操作才能进行。
4.  **StoreLoad**：确保之前的写入操作完成后，后续的读取操作才能进行。这是最强的内存屏障，确保所有之前的存储都对之后的读取可见。

在Java中，`volatile`和`synchronized`关键字在某种程度上用来实现类似内存屏障的功能，尽管它们的主要目的和使用场景有所不同。
\#### volatile

在Java中，`volatile`变量的读写会插入内存屏障指令，保证了volatile变量的可见性和部分顺序性。当一个字段被声明为volatile，编译器和运行时都会在访问该字段时插入内存屏障，确保不会有指令重排序发生，使得一个线程写入的值对其他线程立即可见。

- **写入volatile变量**相当于插入一个`StoreStore`屏障后跟一个`StoreLoad`屏障。
- **读取volatile变量**会插入一个`LoadLoad`屏障后跟一个`LoadStore`屏障。

#### synchronized

`synchronized`关键字在Java中用于实现线程间的互斥和内存一致性。当进入一个synchronized块时，会在开始处插入一个`LoadLoad`和`LoadStore`屏障，确保之前的操作不会与进入的synchronized块重排序。退出synchronized块时，会插入一个`StoreStore`和`StoreLoad`屏障，确保synchronized块内的所有变化对接下来将要执行的操作可见。

对于==Java并发编程图册==中的例子(volatile读-写内存语义)，是由于`volatile`关键字用在`flag`变量上产生的`Happens-Before`关系，即在`volatile`变量的写操作到该变量的读操作之间建立了一个内存可见的桥梁
\### Happens-Before原则

JMM提出了`happens-before`（先行发生）原则，定义一些禁止编译优化的场景，来向各位程序员做一些保证，只要我们是按照原则进行编程，那么就能够保持并发编程的正确性。

其基本定义为：在 Java 内存模型中，如果一个操作 A happens-before 另一个操作 B，那么 A 的结果对 B 是可见的，并且 A 的执行顺序在 B 之前。这种关系有助于解决多线程环境中的可见性问题和指令重排问题

常见的几种典型情况：
&#43; **程序顺序规则**：同一个线程中，按照程序的顺序，前面的操作happens-before后续的任何操作。
&#43; 同一个线程内，代码的执行结果是有序的。其实就是，可能会发生指令重排，但是保证代码的执行结果一定是和按照顺序执行得到的一致，程序前面对某一个变量的修改一定对后续操作可见的，不可能会出现前面才把a修改为1，接着读a居然是修改前的结果，这也是程序运行最基本的要求
&#43; **监视器锁规则**（Monitor - `synchronized`）：对一个锁的解锁 happens-before 随后对这个锁的加锁
&#43; 就是无论是在单线程环境还是多线程环境，对于同一个锁来说，一个线程对这个锁解锁之后，另一个线程获取了这个锁都能看到前一个线程的操作结果。比如前一个线程将变量`x`的值修改为了`12`并解锁，之后另一个线程拿到了这把锁，对之前线程的操作是可见的，可以得到`x`是前一个线程修改后的结果`12`（所以synchronized是有happens-before规则的）
&#43; **`volatile` 变量规则**：对一个 `volatile` 字段的写操作 happens-before 任何后续对这个 `volatile` 字段的读操作
&#43; 就是如果一个线程先去写一个`volatile`变量，紧接着另一个线程去读这个变量，那么这个写操作的结果一定对读的这个变量的线程可见，并且会刷新其余的变量，例如书中例子，但是是隐式的
&#43; **线程启动规则**：从线程 A 启动线程 B 的动作 happens-before 线程 B 中的任何动作。
&#43; 在主线程A执行过程中，启动子线程B，那么线程A在启动子线程B之前对共享变量的修改结果对线程B可见
&#43; **线程终止规则**：线程 A 的所有操作 happens-before 其他线程检测到线程 A 已经终止的动作（通过 `join` 或其他方式）
&#43; **传递性规则：** 如果A happens-before B，B happens-before C，那么A happens-before C。

# 多线程核心 (JUC)

## 锁框架

在JDK 5之后，并发包中新增了Lock接口（以及相关实现类）用来实现锁功能，Lock接口提供了与synchronized关键字类似的同步功能，但需要在使用时手动获取锁和释放锁。

### Lock和Condition接口

``` java
public interface Lock {
      //获取锁，拿不到锁会阻塞，等待其他线程释放锁，获取到锁后返回
    void lock();
      //同上，但是等待过程中会响应中断
    void lockInterruptibly() throws InterruptedException;
      //尝试获取锁，但是不会阻塞，如果能获取到会返回true，不能返回false
    boolean tryLock();
      //尝试获取锁，但是可以限定超时时间，如果超出时间还没拿到锁返回false，否则返回true，可以响应中断
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
      //释放锁
    void unlock();
      //暂时可以理解为替代传统的Object的wait()、notify()等操作的工具
    Condition newCondition();
}
```

``` java
public interface Condition {
      //与调用锁对象的wait方法一样，会进入到等待状态，但是这里需要调用Condition的signal或signalAll方法进行唤醒（感觉就是和普通对象的wait和notify是对应的）同时，等待状态下是可以响应中断的
      void await() throws InterruptedException;
      //同上，但不响应中断（看名字都能猜到）
      void awaitUninterruptibly();
      //等待指定时间，如果在指定时间（纳秒）内被唤醒，会返回剩余时间，如果超时，会返回0或负数，可以响应中断
      long awaitNanos(long nanosTimeout) throws InterruptedException;
      //等待指定时间（可以指定时间单位），如果等待时间内被唤醒，返回true，否则返回false，可以响应中断
      boolean await(long time, TimeUnit unit) throws InterruptedException;
      //可以指定一个明确的时间点，如果在时间点之前被唤醒，返回true，否则返回false，可以响应中断
      boolean awaitUntil(Date deadline) throws InterruptedException;
      //唤醒一个处于等待状态的线程，注意还得获得锁才能接着运行
      void signal();
      //同上，但是是唤醒所有等待线程
      void signalAll();
}
```

在使用`Condition`时，`await()`的线程需要先获取锁，`signal()`的线程也需要获取锁，且唤醒后也需要再次获取锁才能继续运行，且不同的`Condition`对象有不同的等待队列[源码实现](源码实现 &#34;wikilink&#34;)，因此无法跨对象唤醒。

### 可重入锁(ReentrantLock)

常见API使用
\#### 公平锁与非公平锁

如果线程之间争抢同一把锁，会暂时进入到等待队列中，根据策略会产生不同的效果：
- 公平锁：多个线程按照申请锁的顺序去获得锁，线程会直接进入队列去排队，永远都是队列的第一位才能得到锁。
- 非公平锁：多个线程去获取锁的时候，会直接去尝试获取，获取不到，再去进入等待队列，如果能获取到，就直接获取到锁。

由AQS源码可知，公平锁不一定是公平的，直到线程进入等待队列后，才能保证公平机制
\### 读写锁(ReadWriteLock)

``` java
public interface ReadWriteLock {
    //获取读锁
    Lock readLock();
      //获取写锁
    Lock writeLock();
}
```

和可重入锁不同的地方在于，可重入锁是一种排他锁，当一个线程得到锁之后，另一个线程必须等待其释放锁，否则一律不允许获取到锁。而读写锁在同一时间，是可以让多个线程获取到锁的，它其实就是针对于读写场景而出现的。

读写锁维护了一个读锁和一个写锁，这两个锁的机制是不同的。
- 读锁：在没有任何线程占用写锁的情况下，同一时间可以有多个线程加读锁。
- 写锁：在没有任何线程占用读锁的情况下，同一时间只能有一个线程加写锁。

#### 锁降级和锁升级

锁降级指的是写锁降级为读锁。当一个线程持有写锁的情况下，虽然其他线程不能加读锁，但是线程自己是可以加读锁的，在同时加了写锁和读锁的情况下，释放写锁，其他的线程就可以一起加读锁，这种操作，就被称之为&#34;锁降级&#34;（注意不是先释放写锁再加读锁，而是持有写锁的情况下申请读锁再释放写锁）

注意在仅持有读锁的情况下去申请写锁，属于&#34;锁升级&#34;，ReentrantReadWriteLock是不支持的
\### 队列同步器AQS源码分析

从`ReentrantLock`的公平锁策略入手，解析AQS源码。
`ReentrantLock#lock()`方法调用的是其内部类`Sync`中的`sync#lock()`方法，而`Sync`类是继承自`AbstractQueuedSynchronizer(AQS)`，调用AQS的内置方法。

- ☐ ==整理AQS中的各类变量，数据结构，与方法之间的调用关系，以图的方式复习总结==
- ☐ Unsafe类的CAS
  \#### AQS底层实现(JDK17)

AQS内部封装了包括锁的获取、释放、等待队列等。一个锁（排他锁为例）的基本功能就是获取锁、释放锁、当锁被占用时，其他线程来争抢会进入等待队列，AQS已经将这些基本的功能封装完成，接下来会依次介绍AQS中的核心部分

##### AQS中的基础变量

``` java
static final int WAITING   = 1;          // must be 1  
static final int CANCELLED = 0x80000000; // must be negative  
static final int COND      = 2;          // in a condition wait

/**  
 * Head of the wait queue, lazily initialized. 
 */
private transient volatile Node head;  
/**  
 * Tail of the wait queue. After initialization, modified only via casTail. */ private transient volatile Node tail;  
  
/**  
 * The synchronization state. */ 
private volatile int state;
```

AQS中采取*Dummy Node*的方式维护等待队列双链表，且是lazily initialized，即在初始化AQS时不会创建相应的等待队列Node
\##### AQS中的等待队列(核心数据结构)

``` java
abstract static class Node {  
    volatile Node prev;       // initially attached via casTail  
    volatile Node next;       // visibly nonnull when signallable  
    Thread waiter;            // visibly nonnull when enqueued  
    volatile int status;      // written by owner, atomic bit ops by others  
  
    // methods for atomic operations    
    final boolean casPrev(Node c, Node v) {  // for cleanQueue  
        return U.weakCompareAndSetReference(this, PREV, c, v);  
    }  
    final boolean casNext(Node c, Node v) {  // for cleanQueue  
        return U.weakCompareAndSetReference(this, NEXT, c, v);  
    }  
    final int getAndUnsetStatus(int v) {     // for signalling  
        return U.getAndBitwiseAndInt(this, STATUS, ~v);  
    }  
    final void setPrevRelaxed(Node p) {      // for off-queue assignment  
        U.putReference(this, PREV, p);  
    }  
    final void setStatusRelaxed(int s) {     // for off-queue assignment  
        U.putInt(this, STATUS, s);  
    }  
    final void clearStatus() {               // for reducing unneeded signals  
        U.putIntOpaque(this, STATUS, 0);  
    }  
  
    private static final long STATUS  
        = U.objectFieldOffset(Node.class, &#34;status&#34;);  
    private static final long NEXT  
        = U.objectFieldOffset(Node.class, &#34;next&#34;);  
    private static final long PREV  
        = U.objectFieldOffset(Node.class, &#34;prev&#34;);  
}
```

##### BigMap

##### `Sync#lock()`

``` java
final void lock() {  
    if (!initialTryLock())  
        acquire(1);  
}
```

##### `[Fair|Unfair]Sync#initialTryLock()`

``` java
/**  
 * Acquires only if reentrant or queue is empty. */
final boolean initialTryLock() {  
    Thread current = Thread.currentThread();  
    int c = getState();  
    if (c == 0) {  
        if (!hasQueuedThreads() &amp;&amp; compareAndSetState(0, 1)) {  
            setExclusiveOwnerThread(current);  
            return true;  
        }  
    } else if (getExclusiveOwnerThread() == current) {  
        if (&#43;&#43;c &lt; 0) // overflow  
            throw new Error(&#34;Maximum lock count exceeded&#34;);  
        setState(c);  
        return true;  
    }  
    return false;  
}
```

该函数流程简单，首先获取当前AQS的状态`State`，若等于0则表示当前没有任何进程获得锁，然后通过`hasQueuedThreads()`方法判断当前等待队列是否有其余线程，并且CAS原子地设置状态为1，获取排他锁，注意，由于该方法本身没有加事实意义上的锁，因此在任意时刻状态都可能会变化（[hasQueuedThreads()](#`AQS hasQueuedThreads()` &#34;wikilink&#34;)注释写道），因此在获取state==0后再次检验队列，可以看作是一种recheck机制；若不等于0则判断是否是当前进程已经持有锁，且没有溢出。

##### `AQS#hasQueuedThreads()`

``` java
/**  
 * Queries whether any threads are waiting to acquire. Note that * because cancellations due to interrupts and timeouts may occur * at any time, a {@code true} return does not guarantee that any  
 * other thread will ever acquire. * * @return {@code true} if there may be other threads waiting to acquire  
 */
 public final boolean hasQueuedThreads() {  
    for (Node p = tail, h = head; p != h &amp;&amp; p != null; p = p.prev)  
        if (p.status &gt;= 0)  
            return true;  
    return false;  
}
```

##### `AQS#acquire(int arg)`

``` java
/**  
 * Acquires in exclusive mode, ignoring interrupts.  Implemented * by invoking at least once {@link #tryAcquire},  
 * returning on success.  Otherwise the thread is queued, possibly * repeatedly blocking and unblocking, invoking {@link  
 * #tryAcquire} until success.  This method can be used  
 * to implement method {@link Lock#lock}.  
 * * @param arg the acquire argument.  This value is conveyed to  
 *        {@link #tryAcquire} but is otherwise uninterpreted and  
 *        can represent anything you like. */
public final void acquire(int arg) {  
    if (!tryAcquire(arg))  
        acquire(null, arg, false, false, false, 0L);  
}
```

该方法调用AQS中待实现类实现的`tryAcquire()`方法，以自定义的方式尝试获取一次锁，若获取失败，则调用`AQS#acquire(Node node, int arg, boolean shared, boolean interruptible, boolean timed, long time)`方法

##### `FairSync#tryAcquire(int acquires)`

``` java
/**  
 * Acquires only if thread is first waiter or empty */
protected final boolean tryAcquire(int acquires) {  
    if (getState() == 0 &amp;&amp; !hasQueuedPredecessors() &amp;&amp;  
        compareAndSetState(0, acquires)) {  
        setExclusiveOwnerThread(Thread.currentThread());  
        return true;  
    }  
    return false;  
}
```

同样的，该函数的大体逻辑与`initialTryLock()`相似，前者是在锁为空或者等待队列为空时获取锁，该函数是在等待队列为空或者该线程为等待队列中的第一位时，进行锁的获取（公平锁，且同样使用CAS原子地设置）

##### `AQS#hasQueuedPredecessors()`

``` java
/**  
 * Queries whether any threads have been waiting to acquire longer * than the current thread. * * &lt;p&gt;An invocation of this method is equivalent to (but may be  
 * more efficient than): * &lt;pre&gt; {@code  
 * getFirstQueuedThread() != Thread.currentThread()  
 *   &amp;&amp; hasQueuedThreads()}&lt;/pre&gt;  
 *  
 * &lt;p&gt;Note that because cancellations due to interrupts and  
 * timeouts may occur at any time, a {@code true} return does not  
 * guarantee that some other thread will acquire before the current * thread.  Likewise, it is possible for another thread to win a * race to enqueue after this method has returned {@code false},  
 * due to the queue being empty. * * &lt;p&gt;This method is designed to be used by a fair synchronizer to  
 * avoid &lt;a href=&#34;AbstractQueuedSynchronizer.html#barging&#34;&gt;barging&lt;/a&gt;.  
 * Such a synchronizer&#39;s {@link #tryAcquire} method should return  
 * {@code false}, and its {@link #tryAcquireShared} method should  
 * return a negative value, if this method returns {@code true}  
 * (unless this is a reentrant acquire).  For example, the {@code  
 * tryAcquire} method for a fair, reentrant, exclusive mode  
 * synchronizer might look like this: * * &lt;pre&gt; {@code  
 * protected boolean tryAcquire(int arg) {  
 *   if (isHeldExclusively()) { *     // A reentrant acquire; increment hold count *     return true; *   } else if (hasQueuedPredecessors()) { *     return false; *   } else { *     // try to acquire normally *   } * }}&lt;/pre&gt;  
 *  
 * @return {@code true} if there is a queued thread preceding the  
 *         current thread, and {@code false} if the current thread  
 *         is at the head of the queue or the queue is empty * @since 1.7  
 */
public final boolean hasQueuedPredecessors() {  
    Thread first = null; Node h, s;  
    if ((h = head) != null &amp;&amp; ((s = h.next) == null ||  
                               (first = s.waiter) == null ||  
                               s.prev == null))  
        first = getFirstQueuedThread(); // retry via getFirstQueuedThread  
    return first != null &amp;&amp; first != Thread.currentThread();  
}
```

正如注释所写，其等价于`hasQueuedThreads()`然后判断是否为`First`，证明大体逻辑的正确性。

考虑以下场景：当前线程调用此方法，并且返回false，此时另外的线程（没有入队）开始抢占锁，在当前线程CAS修改state之前先获取到了锁，此时当前线程无法获取锁，即使此方法返回了false，并且大体逻辑是在公平锁语境下。因此只有已经进入队列的线程才能保证公平性。

回到该函数，如果判断头节点不为空，则等待队列可能拥有元素，并且在头节点的下一个节点为空，或者头节点的下一个节点的等待线程为空，或者头节点的下一个线程的`prev`字段为空，证明等待队列处于一个不一致的情况，或者是过渡状态（节点正在添加或移除），会显式调用`getFirstQueuedThread()`方法可靠地获取队列中的第一个线程，否则在判断的过程中就会将`fisrt`字段赋值（短路操作）。最后判断`first`是否是当前线程即可。

&gt; \[!NOTE\] GPT解释
&gt; Detailed Explanation of hasQueuedPredecessors():
&gt; This method provides a way to determine if the calling thread should wait in line or attempt to acquire the lock directly, based on whether there are other threads ahead of it in the queue.
&gt;
&gt; 1.  Checking the Queue:
&gt;     The method starts by initializing Thread first to null and declaring Node h and Node s.
&gt;     It then checks if the head of the queue (h) is not null. If the head exists, it proceeds to check the next node (s = h.next).
&gt; 2.  Evaluating Conditions:
&gt;     If the head&#39;s next node (s) is null, or s.waiter (the thread in the s node) is null, or s.prev (the link back to the head) is null, it implies a possibility of inconsistency in the queue or that the queue might be transitioning states (e.g., nodes being added or removed).
&gt;     In such cases, it uses getFirstQueuedThread() to reliably get the first thread in the queue and reassess the situation. This call is more robust but potentially less efficient, hence used as a fallback.
&gt;     3.Return Logic:
&gt;     Finally, the method returns true if first (the first queued thread determined either directly or through the fallback) is not null and is not the current thread. This means there is at least one thread that has been waiting longer than the current thread.
&gt;     If first is null or it is the current thread, it returns false, indicating either the queue is empty or the current thread is at the head of the queue.

##### `AQS#acquire(Node node, int arg, boolean shared, boolean interruptible, boolean timed, long time)`

``` java
final int acquire(Node node, int arg, boolean shared,
                      boolean interruptible, boolean timed, long time) {
        Thread current = Thread.currentThread();
        byte spins = 0, postSpins = 0;   // retries upon unpark of first thread
        boolean interrupted = false, first = false;
        Node pred = null;               // predecessor of node when enqueued

        /*
         * Repeatedly:
         *  Check if node now first
         *    if so, ensure head stable, else ensure valid predecessor
         *  if node is first or not yet enqueued, try acquiring
         *  else if queue is not initialized, do so by attaching new header node
         *     resort to spinwait on OOME trying to create node
         *  else if node not yet created, create it
         *     resort to spinwait on OOME trying to create node
         *  else if not yet enqueued, try once to enqueue
         *  else if woken from park, retry (up to postSpins times)
         *  else if WAITING status not set, set and retry
         *  else park and clear WAITING status, and check cancellation
         */

        for (;;) {
            if (!first &amp;&amp; (pred = (node == null) ? null : node.prev) != null &amp;&amp;
                !(first = (head == pred))) {
                if (pred.status &lt; 0) {
                    cleanQueue();           // predecessor cancelled
                    continue;
                } else if (pred.prev == null) {
                    Thread.onSpinWait();    // ensure serialization
                    continue;
                }
            }
            if (first || pred == null) {
                boolean acquired;
                try {
                    if (shared)
                        acquired = (tryAcquireShared(arg) &gt;= 0);
                    else
                        acquired = tryAcquire(arg);
                } catch (Throwable ex) {
                    cancelAcquire(node, interrupted, false);
                    throw ex;
                }
                if (acquired) {
                    if (first) {
                        node.prev = null;
                        head = node;
                        pred.next = null;
                        node.waiter = null;
                        if (shared)
                            signalNextIfShared(node);
                        if (interrupted)
                            current.interrupt();
                    }
                    return 1;
                }
            }
            Node t;
            if ((t = tail) == null) {           // initialize queue
                if (tryInitializeHead() == null)
                    return acquireOnOOME(shared, arg);
            } else if (node == null) {          // allocate; retry before enqueue
                try {
                    node = (shared) ? new SharedNode() : new ExclusiveNode();
                } catch (OutOfMemoryError oome) {
                    return acquireOnOOME(shared, arg);
                }
            } else if (pred == null) {          // try to enqueue
                node.waiter = current;
                node.setPrevRelaxed(t);         // avoid unnecessary fence
                if (!casTail(t, node))
                    node.setPrevRelaxed(null);  // back out
                else
                    t.next = node;
            } else if (first &amp;&amp; spins != 0) {
                --spins;                        // reduce unfairness on rewaits
                Thread.onSpinWait();
            } else if (node.status == 0) {
                node.status = WAITING;          // enable signal and recheck
            } else {
                long nanos;
                spins = postSpins = (byte)((postSpins &lt;&lt; 1) | 1);
                if (!timed)
                    LockSupport.park(this);
                else if ((nanos = time - System.nanoTime()) &gt; 0L)
                    LockSupport.parkNanos(this, nanos);
                else
                    break;
                node.clearStatus();
                if ((interrupted |= Thread.interrupted()) &amp;&amp; interruptible)
                    break;
            }
        }
        return cancelAcquire(node, interrupted, interruptible);
    }
```

这是AQS获取锁的核心代码，所有暴露的`acquire`方法都会调用这个方法，其通过自旋&#43;CAS的方式进行无锁并发，只有在成功获取、超时、中断的情况下会退出自旋。

针对当前节点不是`first`的情况，首先尝试获取当前节点的前驱，若前驱不为空，则再次判断当前节点是否为`first`并赋值，若再次判断后不为`first`，则继续判断：如果前驱为CANCELLED，调用`cleanQueue()`尝试清理状态为CANCELLED的节点，优化等待队列，帮助保持头节点的稳定性，清除操作后，Head节点的下一个节点将指向一个有效的、未取消的节点，从而使得锁的获取更加顺畅，并调用`Thread.onSpinWait()`确保前驱节点的有效性。

若当前节点为`first`或者尚未入队，再次调用实现类的`tryAcquire()`方法尝试获取锁，如果成功获取，且当前节点为`first`，则调整等待队列的`head`，并且在共享锁模式下尝试唤醒其余可能唤醒的节点，处理中断。

如果当前`tail`为null，证明等待队列尚未初始化，调用`tryInitializeHead()`初始化等待队列（证明AQS的等待队列为lazy initialize）

如果当前队列已经初始化，但是当前节点为null，则根据模式创建相应的节点。

如果当前队列已经初始化，且节点已经初始化，但尚未入队（`pred==null`），尝试进行入队，将节点的`waiter`更改为当前线程，CAS尝试修改`tail`

如果当前节点是`first`并且已经被`unparked`，减少自旋值增加公平性。

如果当前节点的状态是0（node.status == 0），将其状态设置为WAITING并进行recheck。在recheck时，其status虽然被设定为WAITING，但如果当前node为`first`且成功获取锁，说明有其他线程unlock并且signalNext将其status设定为0，依旧保证status为0的情况下才能获取锁。[signalNext()中的判断条件可以解释](#`AQS signalNext()` &#34;wikilink&#34;)

否则，将该线程挂起`park`，在唤醒后将其的status设定为0，并处理中断

&gt; \[!NOTE\] GPT对于Status的解释
&gt; Understanding Status Management
&gt; 1. Role of WAITING Status:

    &gt;   &#43;  When a node (representing a thread in a queue) is set to WAITING, it typically indicates that the thread is actively waiting and should remain parked until explicitly signalled. The WAITING status is used to manage thread wake-up correctly and to avoid lost wake-ups.

&gt; 1.  Role of Status Zero (0):

    &gt;   &#43; A status of 0 generally indicates that the node is not in any particular waiting or signal state. This can mean several things depending on the context:
        &gt;       &#43; The thread is not currently waiting.
        &gt;       &#43; The thread has been woken up and is about to retry acquiring the lock.
        &gt;       &#43; The thread has completed its operation and is being cleaned up.

&gt; 1.  Acquiring the Lock with Status Zero:

    &gt;   &#43; Setting the status to zero does not by itself grant the lock to the node. Instead, it signifies that the node is in a state eligible to attempt to acquire the lock. When a thread (node) attempts to acquire the lock, having a status of zero implies that it is neither parked nor scheduled to be parked. This status allows it to enter the lock acquisition logic without being delayed by unnecessary waits.

&gt; 1.  Transition from WAITING to Zero:
&gt;
&gt; - The transition from WAITING to 0 typically occurs when:
&gt;   - The node is signalled (either by LockSupport.unpark() or similar mechanisms) that it should wake up and retry acquiring the lock.
&gt;   - The thread successfully acquires the lock and subsequently clears its status to indicate it is no longer waiting.
&gt;   - The thread is aborting its wait, possibly due to a timeout or an interrupt, and needs to clear its status as part of cleanup operations.
&gt;
&gt; Practical Implication
&gt; &#43; In a Blocking Scenario (Park):
&gt; &#43; While the node is WAITING, the thread is typically parked (LockSupport.park()) and will remain so until it is unparked or otherwise signalled. The WAITING status helps ensure that the node remains correctly identified as being in need of a wake-up signal.
&gt; &#43; In a Lock Acquisition Scenario:
&gt; &#43; A node may attempt to acquire the lock regardless of its initial status (either 0 or transitioning from WAITING). If the lock acquisition is successful, any status related to waiting is irrelevant post-acquisition; thus, clearing the status to 0 is often an administrative or cleanup action, preparing the node for potential reuse or ensuring it does not remain marked as waiting unnecessarily.

##### `AQS#cleanQueue()`

``` java
/**
 * Possibly repeatedly traverses from tail, unsplicing cancelled
 * nodes until none are found. Unparks nodes that may have been
 * relinked to be next eligible acquirer.
 */
private void cleanQueue() {
    for (;;) {                               // restart point
        for (Node q = tail, s = null, p, n;;) { // (p, q, s) triples
            if (q == null || (p = q.prev) == null)
                return;                      // end of list
            if (s == null ? tail != q : (s.prev != q || s.status &lt; 0))
                break;                       // inconsistent
            if (q.status &lt; 0) {              // cancelled
                if ((s == null ? casTail(q, p) : s.casPrev(q, p)) &amp;&amp;
                    q.prev == p) {
                    p.casNext(q, s);         // OK if fails
                    if (p.prev == null)
                        signalNext(p);
                }
                break;
            }
            if ((n = p.next) != q) {         // help finish
                if (n != null &amp;&amp; q.prev == p) {
                    p.casNext(n, q);
                    if (p.prev == null)
                        signalNext(p);
                }
                break;
            }
            s = q;
            q = q.prev;
        }
    }
}
```

该方法尝试遍历整个等待队列，使用p,q,s三个变量表示当前节点的前驱，当前节点，当前节点的后继。
&#43; 判断是否已达到队列的起点，若已达到则退出循环
&#43; 若当前节点的状态不是一致性，则退出内层循环，从尾部重新开始清理
&#43; 若当前节点`q`的状态是CANCELLED，尝试通过CAS更改当前节点的前驱，并且修改当前节点的后继的CAS操作可以失败，在再次循环中会判断这种情况（前驱正确，但是后继不正确），并帮助完成链接
&#43; 并且在修改链接关系后，判断当前节点是否可能为`head`节点（`p.prev==null`），若可能，则调用`signalNext(p)`唤醒下一个线程

&gt; \[!NOTE\] 唤醒下一个线程
&gt; 1. **维持锁的可用性**：如果 `p` 是 `head` 或近似于 `head`，并且 `p` 的 `next` 指向另一个有效的等待节点，那么这个节点现在可能有机会获取锁。因此，唤醒该节点上的线程是必要的，以便它可以尝试获取锁。
&gt; 2. **避免线程饥饿**：在多线程并发控制中，保持队列的公平性和活跃性非常重要。如果 `p` 的 `next` 节点的线程处于等待状态，不及时唤醒它可能导致线程饥饿，即线程长时间等待而不得执行。
&gt; 3. **响应队列变化**：当 `cleanQueue()` 方法移除一个或多个已取消的节点时，队列的状态发生了变化。更新 `head` 并唤醒相应的线程是响应这种变化、保证锁机制正常运作的必要步骤。

##### `Sync#unlock()`

内部仅调用`AQS#release(1)`
\##### `AQS#release(int arg)`
与`AQS#acquire(int arg)`相似，调用实现类实现的方法`tryRelease(int arg)`，如果成功释放，则唤醒等待队列中的第一个可用线程节点
\##### `Sync#tryRelease()`&amp;`AQS#signalNext()`
两者逻辑较为简单，后者的判断条件决定了，只有当`status == WAITING`时，才能被唤醒，此时存在被`park`或者正在`acquire`中进行recheck，都保证了获取锁之前会将`status`置为0

``` java
protected final boolean tryRelease(int releases) {  
    int c = getState() - releases;  
    if (getExclusiveOwnerThread() != Thread.currentThread())  
        throw new IllegalMonitorStateException();  
    boolean free = (c == 0);  
    if (free)  
        setExclusiveOwnerThread(null);  
    setState(c);  
    return free;  
}
/**
 * Wakes up the successor of given node, if one exists, and unsets its
 * WAITING status to avoid park race. This may fail to wake up an
 * eligible thread when one or more have been cancelled, but
 * cancelAcquire ensures liveness.
 */
private static void signalNext(Node h) {
    Node s;
    if (h != null &amp;&amp; (s = h.next) != null &amp;&amp; s.status != 0) {
        s.getAndUnsetStatus(WAITING);
        LockSupport.unpark(s.waiter);
    }
}
```

##### `Condition`

`condition`类实际上是代替传统对象的`wait &amp; notify`操作的，实现等待/通知模式，并且同一把锁下面可以创建复数个`condition`对象

在AQS内部，通过复用等待队列的Node结构实现`condition`等待队列，但是其中的Node节点状态为`COND`且仅维护后继节点（普通的单链表），并且`condition`队列是由`ConditionObject`实现类进行维护，其与AQS的等待队列结构相似，仅是修改了节点定义，实现了相关方法

``` java
static final class ConditionNode extends Node
    implements ForkJoinPool.ManagedBlocker {
    ConditionNode nextWaiter;            // link to next waiting node
    // ......
}
/**
 * Condition implementation for a {@link AbstractQueuedSynchronizer}
 * serving as the basis of a {@link Lock} implementation.
 *
 * &lt;p&gt;Method documentation for this class describes mechanics,
 * not behavioral specifications from the point of view of Lock
 * and Condition users. Exported versions of this class will in
 * general need to be accompanied by documentation describing
 * condition semantics that rely on those of the associated
 * {@code AbstractQueuedSynchronizer}.
 *
 * &lt;p&gt;This class is Serializable, but all fields are transient,
 * so deserialized conditions have no waiters.
 */
public class ConditionObject implements Condition, java.io.Serializable {
    private static final long serialVersionUID = 1173984872572414699L;
    /** First node of condition queue. */
    private transient ConditionNode firstWaiter;
    /** Last node of condition queue. */
    private transient ConditionNode lastWaiter;
    // ......
}
```

其核心在于`await()`与`signal()`，接下来依次解析相应源码
\##### `ConditionObject#await()`

``` java
/**
 * Implements interruptible condition wait.
 * &lt;ol&gt;
 * &lt;li&gt;If current thread is interrupted, throw InterruptedException.
 * &lt;li&gt;Save lock state returned by {@link #getState}.
 * &lt;li&gt;Invoke {@link #release} with saved state as argument,
 *     throwing IllegalMonitorStateException if it fails.
 * &lt;li&gt;Block until signalled or interrupted.
 * &lt;li&gt;Reacquire by invoking specialized version of
 *     {@link #acquire} with saved state as argument.
 * &lt;li&gt;If interrupted while blocked in step 4, throw InterruptedException.
 * &lt;/ol&gt;
 */
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    ConditionNode node = newConditionNode();
    if (node == null)
        return;
    int savedState = enableWait(node);
    LockSupport.setCurrentBlocker(this); // for back-compatibility
    boolean interrupted = false, cancelled = false, rejected = false;
    while (!canReacquire(node)) {
        if (interrupted |= Thread.interrupted()) {
            if (cancelled = (node.getAndUnsetStatus(COND) &amp; COND) != 0)
                break;              // else interrupted after signal
        } else if ((node.status &amp; COND) != 0) {
            try {
                if (rejected)
                    node.block();
                else
                    ForkJoinPool.managedBlock(node);
            } catch (RejectedExecutionException ex) {
                rejected = true;
            } catch (InterruptedException ie) {
                interrupted = true;
            }
        } else
            Thread.onSpinWait();    // awoke while enqueuing
    }
    LockSupport.setCurrentBlocker(null);
    node.clearStatus();
    acquire(node, savedState, false, false, false, 0L);
    if (interrupted) {
        if (cancelled) {
            unlinkCancelledWaiters(node);
            throw new InterruptedException();
        }
        Thread.currentThread().interrupt();
    }
}
```

该函数的大体逻辑较为清晰：
&#43; 创建新的`ConditionNode`
&#43; 调用`ConditionObject#enableWait()`进行当前锁状态的存储与释放，设定状态为`COND | WAITING`，添加进入`condition`等待队列
&#43; 循环调用`ConditionObject#canRequire()`判断该节点是否可以获取锁，**该方法通过判断该节点是否已经从`condition`等待队列移入`AQS`等待队列**
&#43; 然后判断中断，通过之前设定的`COND | WAITING`状态进行判断是否在`signal`之前就被`interrupt`，具体来说，在`signal`之后，`status`中的`COND`位会被移除，若在此处移除`COND`位之前尚未被移除，说明该中断在`signal`之前
&#43; 并且在至少拥有`COND`状态的情况下调用`AQS#block()`进行`park`等待唤醒
&#43; 否则调用`Thread.onSpinWait()`等待入队进程完成，因为不满足前者的情况下说明现在是过渡态
&#43; 若已经成功移入`AQS`等待队列，清除当前状态为0，调用`AQS#acquire(argvs...)`进行普通的锁获取

##### `ConditionObject#enableWait()`

``` java
/**
 * Adds node to condition list and releases lock.
 *
 * @param node the node
 * @return savedState to reacquire after wait
 */
private int enableWait(ConditionNode node) {
    if (isHeldExclusively()) {
        node.waiter = Thread.currentThread();
        node.setStatusRelaxed(COND | WAITING);
        ConditionNode last = lastWaiter;
        if (last == null)
            firstWaiter = node;
        else
            last.nextWaiter = node;
        lastWaiter = node;
        int savedState = getState();
        if (release(savedState))
            return savedState;
    }
    node.status = CANCELLED; // lock not held or inconsistent
    throw new IllegalMonitorStateException();
}
```

该函数逻辑简单，在获得排他锁的情况下将新节点加入`condition`等待队列，调用`AQS#release()`方法释放当前的锁

##### `ConditionObject#canRequire()`

``` java
/**
 * Returns true if a node that was initially placed on a condition
 * queue is now ready to reacquire on sync queue.
 * @param node the node
 * @return true if is reacquiring
 */
private boolean canReacquire(ConditionNode node) {
    // check links, not status to avoid enqueue race
    Node p; // traverse unless known to be bidirectionally linked
    return node != null &amp;&amp; (p = node.prev) != null &amp;&amp;
        (p.next == node || isEnqueued(node));
}
```

该函数同样简单，检测队列的完整性，并且判断是否进入AQS等待队列，注意，只有AQS等待队列维护前驱，即已经进入AQS队列后才可能返回true，并且使用了recheck方法

##### `Node#getAndUnsetStatus(int v)`

``` java
final int getAndUnsetStatus(int v) {     // for signalling  
    return U.getAndBitwiseAndInt(this, STATUS, ~v);  
}
```

该工具函数为按位取反再与运算，返回值为操作之前的状态
\##### `ConditionObject#signal()` -\&gt; `ConditionObject#doSignal()`

``` java
/**
 * Removes and transfers one or all waiters to sync queue.
 */
private void doSignal(ConditionNode first, boolean all) {
    while (first != null) {
        ConditionNode next = first.nextWaiter;
        if ((firstWaiter = next) == null)
            lastWaiter = null;
        if ((first.getAndUnsetStatus(COND) &amp; COND) != 0) {
            enqueue(first);
            if (!all)
                break;
        }
        first = next;
    }
}
```

该函数的核心在于判断当前节点是否有`COND`位并取消`COND`位，若拥有，则调用`ConditionNode#enqueue()`将该节点从`condition`队列中移入`AQS`等待队列

##### `ConditionNode#enqueue()`

``` java
/**
 * Enqueues the node unless null. (Currently used only for
 * ConditionNodes; other cases are interleaved with acquires.)
 */
final void enqueue(ConditionNode node) {
    if (node != null) {
        boolean unpark = false;
        for (Node t;;) {
            if ((t = tail) == null &amp;&amp; (t = tryInitializeHead()) == null) {
                unpark = true;             // wake up to spin on OOME
                break;
            }
            node.setPrevRelaxed(t);        // avoid unnecessary fence
            if (casTail(t, node)) {
                t.next = node;
                if (t.status &lt; 0)          // wake up to clean link
                    unpark = true;
                break;
            }
        }
        if (unpark)
            LockSupport.unpark(node.waiter);
    }
}
```

该函数的核心思想在于`unpark`唤醒，若初始化`AQS`等待队列失败（OOME），或者该节点的前驱（之前的tail）处于`CANCELLED`，则需要唤醒当前节点（同样是一种recheck机制），唤醒后的节点一定会尽快进入`AQS#acquire`，无论是在哪个等待队列，对于`condition`队列，会调用`await()`之后的代码进入`acquire`，对于AQS队列，其已经进入`acquire`再被`park`。对于前者，会再次初始化抛出OOME，对于后者，会调用`AQS#cleanQueue()`，确保AQS等待队列的完整性，优化效率与内存

&gt; \[!NOTE\] Why unpark when predcessor is CANCELLED
&gt; - **Why Wake Up on Cancelled Status?**: If the previous tail is cancelled, it might be necessary to wake up or signal other threads because the presence of a cancelled node at the tail can disrupt normal lock acquisition processes. The cancelled node may not be properly participating in the queue dynamics (like signaling next nodes), so handling or removing it quickly is crucial.

#### 自行实现锁类

- 重写`Lock`接口中的方法
- 重写`AQS`提供的五个`try`方法中所需要使用的，并与`Lock`接口的重写方法相关联即可
  \## 原子类

### 原子类介绍

- AtomicInteger
- AtomicLong
- AtomicBoolean
- AtomicReference\&lt;?\&gt;
- AtomicIntegerArray
- AtomicLongArray
- AtomicReferenceArray
- DoubleAdder
- LongAdder

本质上是采取了`volatile`关键字&#43;自旋CAS操作保证原子性
\### ABA问题

![ABA](https://minio.dionysunrsshub.top/myimages/2024-img/ABA.png)
JUC提供了带版本号的引用类型，只要每次操作都记录一下版本号，并且版本号不会重复，那么就可以解决ABA问题了，类比Redis

## 并发容器

### 传统容器

以ArrayList\&lt;\&gt;为例，其add方法如下：

``` java
public boolean add(E e) { 
    ensureCapacityInternal(size &#43; 1); // Increments modCount!! 
    elementData[size&#43;&#43;] = e; //这一句出现了数组越界 
    return true; 
}
```

在多线程的情况下就会出现数组越界的情况，而HashMap也存在相应的问题，于是我们需要线程安全的解决方法
\### 并发容器

使用`synchronized`关键字是一个可靠的解决方法，但是其效率较为低下，JUC包中提供了相应的线程安全集合类

#### `CopyOnWriteArrayList&lt;&gt;`

在写操作中获取锁，复制并扩容，修改数组并回写。
在读操作中不获取锁。

#### `ConcurrentHashMap&lt;&gt;`

![HashMap](https://minio.dionysunrsshub.top/myimages/2024-img/HashMap.jpg)
HashMap就是利用了哈希表，哈希表的本质其实就是一个用于存放后续节点的头结点的数组，数组里面的每一个元素都是一个头结点（也可以说就是一个链表），当要新插入一个数据时，会先计算该数据的哈希值，找到数组下标，然后创建一个新的节点，添加到对应的链表后面。当链表的长度达到8时，会自动将链表转换为红黑树，这样能使得原有的查询效率大幅度升高！当使用红黑树之后，我们就可以利用二分搜索的思想，快速地去寻找我们想要的结果，而不是像链表一样挨个去看

JDK7之前，是将所有数据进行分段，对于每一段数据共享一把锁
![concurrentHashMap1](https://minio.dionysunrsshub.top/myimages/2024-img/concurrentHashMap1.png)
JDK8之后，是对于每一个头节点给予一把锁
![concurrentHashMap2](https://minio.dionysunrsshub.top/myimages/2024-img/concurrentHashMap2.png)

其核心在于`putVal()`与`get()`

``` java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());    //计算键的hash值，用于确定在哈希表中的位置
    int binCount = 0;   //用来记录链表长度
    for (Node&lt;K,V&gt;[] tab = table;;) {    //CAS自旋锁
        Node&lt;K,V&gt; f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();    //如果数组（哈希表）为空肯定是要进行初始化的，然后再重新进下一轮循环
        else if ((f = tabAt(tab, i = (n - 1) &amp; hash)) == null) {   //如果哈希表该位置为null，直接CAS插入结点作为头结即可（注意这里会将f设置当前哈希表位置上的头结点）
            if (casTabAt(tab, i, null,
                         new Node&lt;K,V&gt;(hash, key, value, null)))  
                break;                   // 如果CAS成功，直接break结束put方法，失败那就继续下一轮循环
        } else if ((fh = f.hash) == MOVED)   //头结点哈希值为-1，正在扩容
            tab = helpTransfer(tab, f);   //帮助进行迁移，完事之后再来下一次循环
        else {     //特殊情况都完了，这里就该是正常情况了，
            V oldVal = null;
            synchronized (f) {   //在前面的循环中f肯定是被设定为了哈希表某个位置上的头结点，这里直接把它作为锁加锁了，防止同一时间其他线程也在操作哈希表中这个位置上的链表或是红黑树
                if (tabAt(tab, i) == f) {
                    if (fh &gt;= 0) {    //头结点的哈希值大于等于0说明是链表，下面就是针对链表的一些列操作
                        ...实现细节略
                    } else if (f instanceof TreeBin) {   //肯定不大于0，肯定也不是-1，还判断是不是TreeBin，所以不用猜了，肯定是红黑树，下面就是针对红黑树的情况进行操作
                          //在ConcurrentHashMap并不是直接存储的TreeNode，而是TreeBin
                        ...实现细节略
                    }
                }
            }
              //根据链表长度决定是否要进化为红黑树
            if (binCount != 0) {
                if (binCount &gt;= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);   //注意这里只是可能会进化为红黑树，如果当前哈希表的长度小于64，它会优先考虑对哈希表进行扩容
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}

public V get(Object key) {
    Node&lt;K,V&gt;[] tab; Node&lt;K,V&gt; e, p; int n, eh; K ek;
    int h = spread(key.hashCode());   //计算哈希值
    if ((tab = table) != null &amp;&amp; (n = tab.length) &gt; 0 &amp;&amp;
        (e = tabAt(tab, (n - 1) &amp; h)) != null) {
          // 如果头结点就是我们要找的，那直接返回值就行了
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null &amp;&amp; key.equals(ek)))
                return e.val;
        }
          //要么是正在扩容，要么就是红黑树，负数只有这两种情况
        else if (eh &lt; 0)
            return (p = e.find(h, key)) != null ? p.val : null;
          //确认无误，肯定在列表里，开找
        while ((e = e.next) != null) {
            if (e.hash == h &amp;&amp;
                ((ek = e.key) == key || (ek != null &amp;&amp; key.equals(ek))))
                return e.val;
        }
    }
      //没找到只能null了
    return null;
}
```

综上，ConcurrentHashMap的put操作，实际上是对哈希表上的所有头结点元素分别加锁，理论上来说哈希表的长度很大程度上决定了ConcurrentHashMap在同一时间能够处理的线程数量，这也是为什么`treeifyBin()`会优先考虑为哈希表进行扩容的原因。显然，这种加锁方式比JDK7的分段锁机制性能更好。
\### 阻塞队列

#### `BlockingQueue&lt;E&gt; extends Queue&lt;E&gt;`

阻塞队列本身也是队列，但是是适应多线程环境下的，基于`ReentrantLock`实现的，接口定义如下

``` java
public interface BlockingQueue&lt;E&gt; extends Queue&lt;E&gt; {
    boolean add(E e);
    
    //入队，如果队列已满，返回false否则返回true（非阻塞）
    boolean offer(E e);
    
    //入队，如果队列已满，阻塞线程直到能入队为止
    void put(E e) throws InterruptedException;
    
    //入队，如果队列已满，阻塞线程直到能入队或超时、中断为止，入队成功返回true否则false
    boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException;
        
    //出队，如果队列为空，阻塞线程直到能出队为止
    E take() throws InterruptedException;
    
    //出队，如果队列为空，阻塞线程直到能出队超时、中断为止，出队成功正常返回，否则返回null
    E poll(long timeout, TimeUnit unit)
        throws InterruptedException;

    //返回此队列理想情况下（在没有内存或资源限制的情况下）可以不阻塞地入队的数量，如果没有限制，则返回 Integer.MAX_VALUE
    int remainingCapacity();

    boolean remove(Object o);

    public boolean contains(Object o);

    //一次性从BlockingQueue中获取所有可用的数据对象（还可以指定获取数据的个数）
    int drainTo(Collection&lt;? super E&gt; c);

    int drainTo(Collection&lt;? super E&gt; c, int maxElements);
```

其常用的三个实现类，即常用的阻塞队列有：
- ArrayBlockingQueue：有界带缓冲阻塞队列（就是队列是有容量限制的，装满了肯定是不能再装的，只能阻塞，数组实现）
- SynchronousQueue：无缓冲阻塞队列（相当于没有容量的ArrayBlockingQueue，因此只有阻塞的情况）
- LinkedBlockingQueue：无界带缓冲阻塞队列（没有容量限制，也可以限制容量，也会阻塞，链表实现）

基于这些实现类，可以轻易实现生产者消费者模型。

#### `ArrayBlockingQueue`

##### 构造方法与基础变量

``` java
/** Main lock guarding all access */  
final ReentrantLock lock;  
  
/** Condition for waiting takes */  
@SuppressWarnings(&#34;serial&#34;)  // Classes implementing Condition may be serializable.  
private final Condition notEmpty;  
  
/** Condition for waiting puts */  
@SuppressWarnings(&#34;serial&#34;)  // Classes implementing Condition may be serializable.  
private final Condition notFull;

/**
 * Creates an {@code ArrayBlockingQueue} with the given (fixed)
 * capacity and the specified access policy.
 *
 * @param capacity the capacity of this queue
 * @param fair if {@code true} then queue accesses for threads blocked
 *        on insertion or removal, are processed in FIFO order;
 *        if {@code false} the access order is unspecified.
 * @throws IllegalArgumentException if {@code capacity &lt; 1}
 */
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity &lt;= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}
```

内部使用`ReentrantLock`与两个`Condition`对象，完成出队与入队的线程阻塞控制

##### `put() &amp; offer()`

``` java
public boolean offer(E e) {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;    //使用了类里面的ReentrantLock进行加锁操作
    lock.lock();    //保证同一时间只有一个线程进入
    try {
        if (count == items.length)   //直接看看队列是否已满，如果没满则直接入队，如果已满则返回false
            return false;
        else {
            enqueue(e);
            return true;
        }
    } finally {
        lock.unlock();
    }
}

public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;    //同样的，需要进行加锁操作
    lock.lockInterruptibly();    //注意这里是可以响应中断的
    try {
        while (count == items.length)
            notFull.await();    //当队列已满时会直接挂起当前线程，在其他线程出队操作时会被唤醒
        enqueue(e);   //直到队列有空位才将线程入队
    } finally {
        lock.unlock();
    }
}
```

##### `poll() &amp; take()`

``` java
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();    //出队同样进行加锁操作，保证同一时间只能有一个线程执行
    try {
        return (count == 0) ? null : dequeue();   //如果队列不为空则出队，否则返回null
    } finally {
        lock.unlock();
    }
}

public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();    //可以响应中断进行加锁
    try {
        while (count == 0)
            notEmpty.await();    //和入队相反，也是一直等直到队列中有元素之后才可以出队，在入队时会唤醒此线程
        return dequeue();
    } finally {
        lock.unlock();
    }
}
```

##### `enqueue() &amp; dequeue()`

``` java
/**
 * Inserts element at current put position, advances, and signals.
 * Call only when holding lock.
 */
private void enqueue(E e) {
    // assert lock.isHeldByCurrentThread();
    // assert lock.getHoldCount() == 1;
    // assert items[putIndex] == null;
    final Object[] items = this.items;
    items[putIndex] = e;
    if (&#43;&#43;putIndex == items.length) putIndex = 0;
    count&#43;&#43;;
    notEmpty.signal();
}

/**
 * Extracts element at current take position, advances, and signals.
 * Call only when holding lock.
 */
private E dequeue() {
    // assert lock.isHeldByCurrentThread();
    // assert lock.getHoldCount() == 1;
    // assert items[takeIndex] != null;
    final Object[] items = this.items;
    @SuppressWarnings(&#34;unchecked&#34;)
    E e = (E) items[takeIndex];
    items[takeIndex] = null;
    if (&#43;&#43;takeIndex == items.length) takeIndex = 0;
    count--;
    if (itrs != null)
        itrs.elementDequeued();
    notFull.signal();
    return e;
}
```

注意在出队时会通知`notFull`，入队时通知`notEmpty`

#### `SynchronousQueue`

该阻塞队列没有任何容量，正常情况下出队必须和入队操作成对出现，即直接以生产者消费者模式进行的，直接通过内部抽象类维护的方法`Transferer&lt;E&gt;#transfer()`来对生产者和消费者之间的数据进行传递，具体来说，通过对传入`transfer()`方法的参数，来区别是`put`还是`take`相关方法。

同样地，该阻塞队列中也存在非公平和公平两种实现（前者是通过`TransferStack&lt;E&gt;`实现，后者是`TransferQueue&lt;E&gt;`），我们以公平模式为例

##### 构造方法和基础变量

``` java
/** Head of queue */
transient volatile QNode head;
/** Tail of queue */
transient volatile QNode tail;
/**
 * Reference to a cancelled node that might not yet have been
 * unlinked from queue because it was the last inserted node
 * when it was cancelled.
 */
transient volatile QNode cleanMe;

TransferQueue() {
    QNode h = new QNode(null, false); // initialize to dummy node.
    head = h;
    tail = h;
}

static final class QNode implements ForkJoinPool.ManagedBlocker {  
    volatile QNode next;          // next node in queue  
    volatile Object item;         // CAS&#39;ed to or from null  
    volatile Thread waiter;       // to control park/unpark  
    final boolean isData;  
  
    QNode(Object item, boolean isData) {  
        this.item = item;  
        this.isData = isData;  
    }
    //......
}
```

我们可以发现，==其维护的`QNode`与AQS中的Node节点十分相似==
\##### `TransferQueue#transfer`

``` java
/**
 * Puts or takes an item.
 */
@SuppressWarnings(&#34;unchecked&#34;)
E transfer(E e, boolean timed, long nanos) {
    /* Basic algorithm is to loop trying to take either of
     * two actions:
     *
     * 1. If queue apparently empty or holding same-mode nodes,
     *    try to add node to queue of waiters, wait to be
     *    fulfilled (or cancelled) and return matching item.
     *
     * 2. If queue apparently contains waiting items, and this
     *    call is of complementary mode, try to fulfill by CAS&#39;ing
     *    item field of waiting node and dequeuing it, and then
     *    returning matching item.
     *
     * In each case, along the way, check for and try to help
     * advance head and tail on behalf of other stalled/slow
     * threads.
     *
     * The loop starts off with a null check guarding against
     * seeing uninitialized head or tail values. This never
     * happens in current SynchronousQueue, but could if
     * callers held non-volatile/final ref to the
     * transferer. The check is here anyway because it places
     * null checks at top of loop, which is usually faster
     * than having them implicitly interspersed.
     */

    QNode s = null;                  // constructed/reused as needed
    boolean isData = (e != null);
    for (;;) {
        QNode t = tail, h = head, m, tn;         // m is node to fulfill
        if (t == null || h == null)
            ;                                    // inconsistent
        else if (h == t || t.isData == isData) { // empty or same-mode
            if (t != tail)                       // inconsistent
                ;
            else if ((tn = t.next) != null)      // lagging tail
                advanceTail(t, tn);
            else if (timed &amp;&amp; nanos &lt;= 0L)       // can&#39;t wait
                return null;
            else if (t.casNext(null, (s != null) ? s :
                               (s = new QNode(e, isData)))) {
                advanceTail(t, s);
                long deadline = timed ? System.nanoTime() &#43; nanos : 0L;
                Thread w = Thread.currentThread();
                int stat = -1; // same idea as TransferStack
                Object item;
                while ((item = s.item) == e) {
                    if ((timed &amp;&amp;
                         (nanos = deadline - System.nanoTime()) &lt;= 0) ||
                        w.isInterrupted()) {
                        if (s.tryCancel(e)) {
                            clean(t, s);
                            return null;
                        }
                    } else if ((item = s.item) != e) {
                        break;                   // recheck
                    } else if (stat &lt;= 0) {
                        if (t.next == s) {
                            if (stat &lt; 0 &amp;&amp; t.isFulfilled()) {
                                stat = 0;        // yield once if first
                                Thread.yield();
                            }
                            else {
                                stat = 1;
                                s.waiter = w;
                            }
                        }
                    } else if (!timed) {
                        LockSupport.setCurrentBlocker(this);
                        try {
                            ForkJoinPool.managedBlock(s);
                        } catch (InterruptedException cannotHappen) { }
                        LockSupport.setCurrentBlocker(null);
                    }
                    else if (nanos &gt; SPIN_FOR_TIMEOUT_THRESHOLD)
                        LockSupport.parkNanos(this, nanos);
                }
                if (stat == 1)
                    s.forgetWaiter();
                if (!s.isOffList()) {            // not already unlinked
                    advanceHead(t, s);           // unlink if head
                    if (item != null)            // and forget fields
                        s.item = s;
                }
                return (item != null) ? (E)item : e;
            }

        } else if ((m = h.next) != null &amp;&amp; t == tail &amp;&amp; h == head) {
            Thread waiter;
            Object x = m.item;
            boolean fulfilled = ((isData == (x == null)) &amp;&amp;
                                 x != m &amp;&amp; m.casItem(x, e));
            advanceHead(h, m);                    // (help) dequeue
            if (fulfilled) {
                if ((waiter = m.waiter) != null)
                    LockSupport.unpark(waiter);
                return (x != null) ? (E)x : e;
            }
        }
    }
}
```

该方法通过判断e是否为null，设定`isData`变量，`true`表示消费者反之表示生产者。

方法入口依旧是自旋，猜测下面是复数个CAS方法，维持多线程中代码操作的正确性和原子性。该方法的主要目的在于：
&#43; 将当前节点入队：当队列为空或者队列中都是状态相同的节点（全是生产者或者全是消费者）
&#43; 满足一个等待中的`transfer`：当队列中存在与当前状态相反的节点时，取出
接下来是代码核心循环逻辑：
&#43; 如果`h`或者`t`为空，证明正在初始化，队列一致性检验不通过，继续循环
&#43; 如果`h == t`，即当前队列为空，或者当前节点的状态与队列中的一致
&#43; 同样判断队列一致性
&#43; 在多线程上下文中，判断`t`是否仍为`tail`，并且更新（lagging tail check）
&#43; 判断是否超时，超时直接返回null
&#43; 否则，证明当前状态和队列都合法，开始尝试进行入队，使用CAS更改`t.next`字段（QNode s是lazily instantiated），若成功则修改`tail`
&#43; 通过判断当前节点中`item`的值是否改变，维持`park`等待或者自旋等待
&#43; 处理超时和中断情况
&#43; recheck `item`的值是否改变，常见的多线程recheck操作
&#43; 判断stat \&lt;= 0，初始值为-1
&#43; 判断队列有效性，即`t.next==s`，如果有效，尝试改变stat状态
&#43; 如果stat\&lt;0，即未改变过，且s的前驱t已经得到满足（`t.isFulfilled()`该方法中检查`isData`字段是否和当前`item`的状态相符，并且再次检查`item`字段是否已经取消 - 对应post-loop中`item != null`方法，已经满足的节点会将`item`设定为`this`），说明当前节点已经为`first`，更改一次stat，并且调用`Thread.yield()`等待
&#43; 如果已经改变过一次，则直接将stat置1，设置当前节点的等待线程`s.waiter=w`，准备被`park`调用（`unpark()`传入的参数是线程）
&#43; 根据是否可以超时进行`park`等待
&#43; 已经退出循环，证明可以被满足，无需等待（post-loop），根据stat和当前节点s的状态设置对应的队列状态，根据消费者或者是生产者返回相应的数据
&#43; 否则，recheck判断当前队列是否不为空，且队列满足一致性，不在过渡态（常规recheck），若满足，则证明队列中的`first`可以与当前节点配对，互相满足
&#43; 还是常规的recheck判断，与`isFulfilled()`相似，并且尝试CAS设置`first.item`为当前节点的`e`
&#43; 注意这里可以直接调用`advanceHead(h,m)`修改`head`，因为如果前面的CAS失败了，说明有其他线程已经抢先满足，那么也是满足修改`head`的前置条件的
&#43; 如果可以满足，并且该线程拥有待唤醒的线程，直接调用`unpark`（使得被阻塞等待的线程唤醒）并且返回相应的值。

总体来说，被阻塞的线程核心在第二个if条件，可以满足被阻塞线程的线程核心在第三个if条件

##### \`TransferStack#transfer

大体思路一致，只不过将队列变为了stack，满足非公平模式

#### `LinkedTransferQueue`

该对象保留了SynchronousQueue的匹配交接机制，并且与等待队列进行融合，我们知道，SynchronousQueue并没有使用锁，而是采用CAS操作保证生产者与消费者的协调，但是它没有容量，而LinkedBlockingQueue虽然是有容量且无界的，但是内部基本都是基于锁实现的，性能并不是很好，这时，我们就可以将它们各自的优点单独拿出来，揉在一起，就成了性能更高的LinkedTransferQueue

相比 `SynchronousQueue` ，它多了一个可以存储的队列，我们依然可以像阻塞队列那样获取队列中所有元素的值，简单来说，`LinkedTransferQueue`其实就是一个多了存储队列的`SynchronousQueue`，插入时，会先检查是否有其他线程等待获取，如果是，直接进行交接，否则插入到存储队列中，不会像SynchronousQueue那样必须等一个匹配的才可以，并且可以打印队列中的元素

（前者被阻塞，进入内部队列，对外不可见，后者是可见的）

#### `PriorityBlockingQueue`

是一个支持优先级的阻塞队列，元素的获取顺序按优先级决定
\#### `DelayQueue`

是一个支持延迟获取元素的队列，同样支持优先级，即考虑延迟的情况下也要考虑优先级，如果优先级更大的元素的延迟尚未结束，后面优先级靠后的元素，即使延迟已经结束也无法获取

其类定义与接口如下：

``` java
public class DelayQueue&lt;E extends Delayed&gt; extends AbstractQueue&lt;E&gt;  
    implements BlockingQueue&lt;E&gt; { 
    
    private final transient ReentrantLock lock = new ReentrantLock();
    private final PriorityQueue&lt;E&gt; q = new PriorityQueue&lt;E&gt;();
    /**
     * Thread designated to wait for the element at the head of
     * the queue.  This variant of the Leader-Follower pattern
     * (http://www.cs.wustl.edu/~schmidt/POSA/POSA2/) serves to
     * minimize unnecessary timed waiting.  When a thread becomes
     * the leader, it waits only for the next delay to elapse, but
     * other threads await indefinitely.  The leader thread must
     * signal some other thread before returning from take() or
     * poll(...), unless some other thread becomes leader in the
     * interim.  Whenever the head of the queue is replaced with
     * an element with an earlier expiration time, the leader
     * field is invalidated by being reset to null, and some
     * waiting thread, but not necessarily the current leader, is
     * signalled.  So waiting threads must be prepared to acquire
     * and lose leadership while waiting.
     */
    private Thread leader;

    /**
     * Condition signalled when a newer element becomes available
     * at the head of the queue or a new thread may need to
     * become leader.
     */
    private final Condition available = lock.newCondition();
    // ......
}

public interface Delayed extends Comparable&lt;Delayed&gt; {

    /**
     * Returns the remaining delay associated with this object, in the
     * given time unit.
     *
     * @param unit the time unit
     * @return the remaining delay; zero or negative values indicate
     * that the delay has already elapsed
     */
    long getDelay(TimeUnit unit);
```

可以看到此类只接受Delayed的实现类作为元素，并且Delayed类继承了`Comparable`，支持优先级，其内部维护的`leader`变量减少不必要的等待，具体解释在类定义的注释中

##### `offer()`

``` java
public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        q.offer(e); // 向内部维护的优先队列添加元素
        if (q.peek() == e) { //如果入队后队首就是当前元素，那么直接进行一次唤醒操作（因为有可能之前就有其他线程等着take了）
            leader = null;
            available.signal();
        }
        return true;
    } finally {
        lock.unlock();
    }
}
```

##### `take()`

``` java
/**
 * Retrieves and removes the head of this queue, waiting if necessary
 * until an element with an expired delay is available on this queue.
 *
 * @return the head of this queue
 * @throws InterruptedException {@inheritDoc}
 */
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            E first = q.peek();
            if (first == null)
                available.await();
            else {
                long delay = first.getDelay(NANOSECONDS);
                if (delay &lt;= 0L)
                    return q.poll();
                first = null; // don&#39;t retain ref while waiting
                if (leader != null)
                    available.await();
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        available.awaitNanos(delay);
                    } finally {
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        if (leader == null &amp;&amp; q.peek() != null)
            available.signal();
        lock.unlock();
    }
}
```

该方法同样先获取锁，并且同样有自旋的操作，逻辑流程如下：
&#43; 首先获取队首元素，如果为空，那么说明队列一定为空，调用`await()`等待
&#43; 否则，获取队首元素的延迟时间，如果延迟结束，直接返回，延迟没有结束，放弃`first`，等待下一轮循环再次获取
&#43; 判断是否拥有`leader`线程，如果拥有，说明有其他的线程正在调用可超时的等待，当前线程直接`await()`
&#43; 否则，将当前线程设定为`leader`，并且设定当前线程的`await()`超时时间为`delay`，在超时后重新设定`leader = null`，继续循环获取队首元素进行判断
&#43; 在获取到元素后，如果判断没有可超时的等待（`leader == null`）并且队首元素不为空，则手动唤醒一个其他永久等待下的线程

# 多线程进阶

## 线程池

利用多线程，我们的程序可以更加合理地使用CPU多核心资源，在同一时间完成更多的工作。但是，如果我们的程序频繁地创建线程，由于线程的创建和销毁也需要占用系统资源，因此这样会降低我们整个程序的性能，为了解决这个开销，可以将已创建的线程复用，利用池化技术，就像数据库连接池一样，我们也可以创建很多个线程，然后反复地使用这些线程，而不对它们进行销毁。

比如我们的Tomcat服务器，要在同一时间接受和处理大量的请求，那么就必须要在短时间内创建大量的线程，结束后又进行销毁，这显然会导致很大的开销，因此这种情况下使用线程池显然是更好的解决方案。

由于线程池可以反复利用已有线程执行多线程操作，所以它一般是有容量限制的，当所有的线程都处于工作状态时，那么新的多线程请求会被阻塞，直到有一个线程空闲出来为止，实际上这里就会用到阻塞队列。

### 线程池的使用

直接通过解析`ThreadPoolExecutor()`对象的构造方法入手：

``` java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue&lt;Runnable&gt; workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize &lt; 0 ||
        maximumPoolSize &lt;= 0 ||
        maximumPoolSize &lt; corePoolSize ||
        keepAliveTime &lt; 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.acc = System.getSecurityManager() == null ?
            null :
            AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

其中的各项参数为：
- corePoolSize：**核心线程池大小**，我们每向线程池提交一个多线程任务时，都会创建一个新的`核心线程`，无论是否存在其他空闲线程，直到到达核心线程池大小为止，之后会尝试复用线程资源。当然也可以在一开始就全部初始化好，调用 `prestartAllCoreThreads()`即可。
- maximumPoolSize：**最大线程池大小**，当目前线程池中所有的线程都处于运行状态，并且等待队列已满，那么就会直接尝试继续创建新的`非核心线程`运行，但是不能超过最大线程池大小。
- keepAliveTime：**线程最大空闲时间**，当一个`非核心线程`空闲超过一定时间，会自动销毁。
- unit：**线程最大空闲时间的时间单位**
- workQueue：**线程等待队列**，当线程池中核心线程数已满时，就会将任务暂时存到等待队列中，直到有线程资源可用为止，这里可以使用阻塞队列。
- threadFactory：**线程创建工厂**，我们可以干涉线程池中线程的创建过程，进行自定义。
- handler：**拒绝策略**，当等待队列和线程池都没有空间时，来了个新的多线程任务，那么只能拒绝了，这时就会根据当前设定的拒绝策略进行处理。

最为重要的就是线程池大小的限定，合理地分配大小会使得线程池的执行效率事半功倍：

- 首先我们可以分析一下，线程池执行任务的特性，是CPU 密集型还是 IO 密集型
  - **CPU密集型：** 主要是执行计算任务，响应时间很快，CPU一直在运行，这种任务CPU的利用率很高，那么线程数应该是根据 CPU 核心数来决定，CPU 核心数 = 最大同时执行线程数，以 i5-9400F 处理器为例，CPU 核心数为 6，那么最多就能同时执行 6 个线程。
  - **IO密集型：** 主要是进行 IO 操作，因为执行 IO 操作的时间比较较长，比如从硬盘读取数据之类的，CPU就得等着IO操作，很容易出现空闲状态，导致 CPU 的利用率不高，这种情况下可以适当增加线程池的大小，让更多的线程可以一起进行IO操作，一般可以配置为CPU核心数的2倍。

其核心方法为`ThreadPoolExecutor#execute()`，接受一个`Runnable`作为线程待执行的任务

通常的拒绝策略有四个：
- AbortPolicy(默认)：直接抛异常。
- CallerRunsPolicy：直接让提交任务的线程运行这个任务，比如在主线程向线程池提交了任务，那么就直接由主线程执行。
- DiscardOldestPolicy：丢弃队列中oldest的一个任务，替换为当前任务。
- DiscardPolicy：什么也不用做。

同样的，我们也可以重写`RejectedExecutionHandler`接口，实现自定义handler，`ThreadFactory`也是一个可重写的接口，提供干涉新线程创建的窗口

当线程池中的线程由于异常中断时，会进行销毁。

此外，`Executors`也提供了几个静态方法来快速创建线程池：
&#43; newFixedThreadPool
&#43; 内部实现是coreThreadSize=maxThreadSize，使用`LinkedBlockingQueue&lt;&gt;`作为等待队列
&#43; newSingleThreadExecutor
&#43; 该方法将创建的`ExecutorService`对象封装为`FinalizableDelegatedExecutorService`，提供一层保护，防止用户更改线程池大小（前者可以调用.setPoolSize()方法）- 与Spring中的Delegated方法一样，不是真正的进行销毁，而是进行保留复用

``` java
static class DelegatedExecutorService extends AbstractExecutorService {
    private final ExecutorService e;    //被委派对象
    DelegatedExecutorService(ExecutorService executor) { e = executor; }   //实际上所以的操作都是让委派对象执行的，有点像代理
    public void execute(Runnable command) { e.execute(command); }
    public void shutdown() { e.shutdown(); }
    public List&lt;Runnable&gt; shutdownNow() { return e.shutdownNow(); }
    // ...
}
```

- newCachedThreadPool
  - 是一个核心线程数为0，最大线程数为Integer.MAX_VALUE

### 执行带返回值的任务

`AbstractExecutorService#submit()`可以接受三种形式的参数：
&#43; Runnable
&#43; Runnable &#43; Result value
&#43; Callable
或者是直接传入`FutureTask&lt;&gt;`对象（该对象相当于后两者情况）

返回一个`Future&lt;?&gt;`对象，可以通过该对象获取当前任务的一些状态

### 执行定时任务

通过`ScheduledThreadPoolExecutor`来提交定时任务，它继承自`ThreadPoolExecutor`，并且所有的构造方法都要求最大线程池容量为Integer.MAX_VALUE，采用`DelayedQueue`作为等待队列

同样的，`ScheduledThreadPoolExecutor#schedule()`方法支持返回值任务，通过`ScheduledFuture&lt;?&gt;`对象进行接受

*猜测，所有的任务先进入`DelayedQueue`后再进行取出*

`ScheduledThreadPoolExecutor#scheduleAtFixedRate`、`ScheduledThreadPoolExecutor#scheduleWithFixedDelay`两个方法可以以一定的频率不断执行任务

### 线程池实现原理

同样的，我们先从核心变量入手，然后walkThrough其核心方法`execute`和`shutdown`
\#### 核心变量

``` java
//使用AtomicInteger，用于同时保存线程池运行状态和线程数量（使用原子类是为了保证原子性）
//它是通过拆分32个bit位来保存数据的，前3位保存状态，后29位保存工作线程数量
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;    //29位，线程数量位
private static final int CAPACITY   = (1 &lt;&lt; COUNT_BITS) - 1;   //计算得出最大容量

// 所有的运行状态，注意都是只占用前3位，不会占用后29位

// 接收新任务，并等待执行队列中的任务
private static final int RUNNING    = -1 &lt;&lt; COUNT_BITS;   //111 | 数量位
// 不接收新任务，但是依然等待执行队列中的任务
private static final int SHUTDOWN   =  0 &lt;&lt; COUNT_BITS;   //000 | 数量位
// 不接收新任务，也不执行队列中的任务，并且还要中断正在执行中的任务
private static final int STOP       =  1 &lt;&lt; COUNT_BITS;   //001 | 数量位
// 所有的任务都已结束，线程数量为0，即将完全关闭
private static final int TIDYING    =  2 &lt;&lt; COUNT_BITS;   //010 | 数量位
// 完全关闭
private static final int TERMINATED =  3 &lt;&lt; COUNT_BITS;   //011 | 数量位

// 封装和解析ctl变量的一些方法
// 取前三位运行状态
private static int runStateOf(int c)     { return c &amp; ~CAPACITY; } 
// 取后29位线程数量
private static int workerCountOf(int c)  { return c &amp; CAPACITY; }
// 将运行状态与线程数量拼接
private static int ctlOf(int rs, int wc) { return rs | wc; }   

//指定的阻塞队列
private final BlockingQueue&lt;Runnable&gt; workQueue;
```

#### `ThreadPoolExecutor#execute(Runnable)`

``` java
/**
 * Executes the given task sometime in the future.  The task
 * may execute in a new thread or in an existing pooled thread.
 *
 * If the task cannot be submitted for execution, either because this
 * executor has been shutdown or because its capacity has been reached,
 * the task is handled by the current {@link RejectedExecutionHandler}.
 *
 * @param command the task to execute
 * @throws RejectedExecutionException at discretion of
 *         {@code RejectedExecutionHandler}, if the task
 *         cannot be accepted for execution
 * @throws NullPointerException if {@code command} is null
 */
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn&#39;t, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
    int c = ctl.get();
    if (workerCountOf(c) &lt; corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) &amp;&amp; workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) &amp;&amp; remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```

该方法的核心逻辑有三，在官方注释中已经详尽介绍：
&#43; 判断当前运行的线程数量，如果小于核心线程数量则尝试调用`addWorker`创建一个新的核心线程，将当前`Runnable`设定为该线程的任务，否则，证明在创建新线程过程中有其他线程抢先，需要重新获取线程池状态(`ctl`)继续判断
&#43; 进入当前条件判断的前提是运行线程数量不小于核心线程大小；判断当前线程池是否处于RUNNING态，并尝试将当前`Runnable`任务加入阻塞队列，同样的，由于该方法没有加锁，需要进行double-check，再次判断当前线程池的状态
&#43; 若当前线程池的状态不为RUNNING（进入了SHUTDOWN态），则证明该`Runnable`任务不该加入阻塞队列，从队列中取出并执行`reject`
&#43; 或者该线程池处于运行状态，但由于其他线程可能导致的不一致性与过渡态，或者线程池中的线程(worker)由于初始化、超时、中断等原因结束了其生命周期，调用`addWorker`添加一个`first`任务为空的非核心线程，确保新加入阻塞队列的`Runnable`可以被预期执行，并且维护其中的队列规则，例如FIFO，priority-based
&#43; 如果进入阻塞队列失败，或者线程池不处于RUNNING状态，尝试调用`addWorker`添加一个`first`为当前`Runnable`的非核心线程，若失败直接调用`reject`

#### `ThreadPoolExecutor#addWorker(Runnable, boolean)`

``` java
/**
 * Checks if a new worker can be added with respect to current
 * pool state and the given bound (either core or maximum). If so,
 * the worker count is adjusted accordingly, and, if possible, a
 * new worker is created and started, running firstTask as its
 * first task. This method returns false if the pool is stopped or
 * eligible to shut down. It also returns false if the thread
 * factory fails to create a thread when asked.  If the thread
 * creation fails, either due to the thread factory returning
 * null, or due to an exception (typically OutOfMemoryError in
 * Thread.start()), we roll back cleanly.
 *
 * @param firstTask the task the new thread should run first (or
 * null if none). Workers are created with an initial first task
 * (in method execute()) to bypass queuing when there are fewer
 * than corePoolSize threads (in which case we always start one),
 * or when the queue is full (in which case we must bypass queue).
 * Initially idle threads are usually created via
 * prestartCoreThread or to replace other dying workers.
 *
 * @param core if true use corePoolSize as bound, else
 * maximumPoolSize. (A boolean indicator is used here rather than a
 * value to ensure reads of fresh values after checking other pool
 * state).
 * @return true if successful
 */
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (int c = ctl.get();;) {
        // Check if queue empty only if necessary.
        if (runStateAtLeast(c, SHUTDOWN)
            &amp;&amp; (runStateAtLeast(c, STOP)
                || firstTask != null
                || workQueue.isEmpty()))
            return false;

        for (;;) {
            if (workerCountOf(c)
                &gt;= ((core ? corePoolSize : maximumPoolSize) &amp; COUNT_MASK))
                return false;
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateAtLeast(c, SHUTDOWN))
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int c = ctl.get();

                if (isRunning(c) ||
                    (runStateLessThan(c, STOP) &amp;&amp; firstTask == null)) {
                    if (t.getState() != Thread.State.NEW)
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    workerAdded = true;
                    int s = workers.size();
                    if (s &gt; largestPoolSize)
                        largestPoolSize = s;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                container.start(t);
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

同样该方法考虑多线程情况，因此使用两个for循环实现自旋锁，保证线程安全，确保阻塞队列状态与线程池工作状态合法，并且可以添加，才会进入实际的worker添加代码段

- 对于外层循环，其主要任务是判断线程池的状态
  - 首先判断当前线程池是否处于RUNNING状态
  - 若不处于RUNNING状态，且处于STOP以上状态，或者处于SHUTDOWN状态（该状态下线程池不再接受新线程，但会执行剩余的线程）但`firstTask`不为空，或者处于SHUTDOWN状态且阻塞队列为空（满足状态进一步切换 - `tryTerminate()`），返回`false`，表明无法添加worker
- 对于内层循环，其主要任务是将线程池中的worker计数增加，采取自旋&#43;CAS方式，增加成功才会执行实际的worker添加代码段
  - 首先判断当前线程池的线程数量(worker)是否未超过设定值（核心与非核心），如果超过直接返回`false`
  - 若满足线程数量要求，尝试增加线程池中的worker数量，若CAS成功，退出外层循环，进入worker添加段
  - CAS操作失败，重新获取当前线程池的状态`ctl`，若当前线程池状态处于SHUTDOWN及以上状态，证明线程池状态已经不再处于RUNNING，退出内层循环，重新进行外层循环，判断线程池的状态，否则，重新进行内层循环，仅仅是CAS线程池的worker数量失败，不涉及线程池状态的变化
- 退出了双层循环，进入了实际添加worker的代码段
- 将当前的`Runnable`任务封装为`Worker`对象，该对象继承自`AQS`，本质上也是一个队列同步器，并且根据`Worker`对象获取线程，double-check其初始化过程
- 尝试获取线程池中的`ReentrantLock`，在获取锁之后，进行一次recheck，判断当前线程池是否处于RUNNING状态，或者是SHUTDOWN状态且`Runnable`为`null`（含义为创建新线程Worker执行队列中的任务，但不接受新`Runnable`任务）
  - 判断当前`Worker`中的线程是否开始执行，若已经开始执行则抛出异常
  - 否则，将当前`Worker`对象加入线程池维护的可用线程对象集合
- 如果成功将当前`Worker`对象加入线程池维护的可用线程对象集合，开始运行该线程

接下来分析`Worker`对象及其是如何开始运行`Runnable`任务的
\#### `Worker`

该类是继承自AQS，本质上也是一把锁，也重写了`tryAcquire`和`tryRelease`方法
\##### 基础变量

``` java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
    /** Thread this worker is running in.  Null if factory fails. */
    @SuppressWarnings(&#34;serial&#34;) // Unlikely to be serializable
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    @SuppressWarnings(&#34;serial&#34;) // Not statically typed as Serializable
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;

    /**
     * Creates with given first task and thread from ThreadFactory.
     * @param firstTask the first task (null if none)
     */
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    /** Delegates main run loop to outer runWorker. */
    public void run() {
        runWorker(this);
    }
    // Lock methods  
    //  
    // The value 0 represents the unlocked state.  
    // The value 1 represents the locked state.
    protected boolean isHeldExclusively() {  
        return getState() != 0;  
    }
    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }
    // .....
}
```

#### `ThreadPoolExecutor#runWorker()`

``` java
/**
 * Main worker run loop.  Repeatedly gets tasks from queue and
 * executes them, while coping with a number of issues:
 *
 * 1. We may start out with an initial task, in which case we
 * don&#39;t need to get the first one. Otherwise, as long as pool is
 * running, we get tasks from getTask. If it returns null then the
 * worker exits due to changed pool state or configuration
 * parameters.  Other exits result from exception throws in
 * external code, in which case completedAbruptly holds, which
 * usually leads processWorkerExit to replace this thread.
 *
 * 2. Before running any task, the lock is acquired to prevent
 * other pool interrupts while the task is executing, and then we
 * ensure that unless pool is stopping, this thread does not have
 * its interrupt set.
 *
 * 3. Each task run is preceded by a call to beforeExecute, which
 * might throw an exception, in which case we cause thread to die
 * (breaking loop with completedAbruptly true) without processing
 * the task.
 *
 * 4. Assuming beforeExecute completes normally, we run the task,
 * gathering any of its thrown exceptions to send to afterExecute.
 * We separately handle RuntimeException, Error (both of which the
 * specs guarantee that we trap) and arbitrary Throwables.
 * Because we cannot rethrow Throwables within Runnable.run, we
 * wrap them within Errors on the way out (to the thread&#39;s
 * UncaughtExceptionHandler).  Any thrown exception also
 * conservatively causes thread to die.
 *
 * 5. After task.run completes, we call afterExecute, which may
 * also throw an exception, which will also cause thread to
 * die. According to JLS Sec 14.20, this exception is the one that
 * will be in effect even if task.run throws.
 *
 * The net effect of the exception mechanics is that afterExecute
 * and the thread&#39;s UncaughtExceptionHandler have as accurate
 * information as we can provide about any problems encountered by
 * user code.
 *
 * @param w the worker
 */
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &amp;&amp;
                  runStateAtLeast(ctl.get(), STOP))) &amp;&amp;
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                try {
                    task.run();
                    afterExecute(task, null);
                } catch (Throwable ex) {
                    afterExecute(task, ex);
                    throw ex;
                }
            } finally {
                task = null;
                w.completedTasks&#43;&#43;;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

该方法的核心逻辑也在官方注释中已经详尽介绍：
&#43; 首先尝试获取`Worker`类中的`Runnable`，并且调用`unlock()`允许中断
&#43; 若`Worker`中存在`first Runnable`，或者调用`getTask()`成功获取`Runnable`任务（`getTask()会阻塞线程直到超时或者得到任务`），尝试获取`Worker`维护的锁，此处的锁在于在`shutdown`时保护此`Worker`完成任务
&#43; 判断线程池是否处于STOP及以上状态并且运行线程池的线程没有被中断标记，打上中断标记
&#43; 尝试获取当前线程是否被中断，重置中断标记，若处于STOP及以上状态，打上中断标记
&#43; 此处保证线程池STOP及以上状态时被中断，否则没有被中断 ==ShutdownNow==
&#43; 真正执行`Runnable`的`run`方法，然后解锁继续循环获取任务
&#43; 退出了循环获取任务，证明该`Worker`可以被丢弃，直接调用`processWorkerExit()`

#### `ThreadPoolExecutor#getTask()`

``` java
/**
 * Performs blocking or timed wait for a task, depending on
 * current configuration settings, or returns null if this worker
 * must exit because of any of:
 * 1. There are more than maximumPoolSize workers (due to
 *    a call to setMaximumPoolSize).
 * 2. The pool is stopped.
 * 3. The pool is shutdown and the queue is empty.
 * 4. This worker timed out waiting for a task, and timed-out
 *    workers are subject to termination (that is,
 *    {@code allowCoreThreadTimeOut || workerCount &gt; corePoolSize})
 *    both before and after the timed wait, and if the queue is
 *    non-empty, this worker is not the last thread in the pool.
 *
 * @return task, or null if the worker must exit, in which case
 *         workerCount is decremented
 */
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();

        // Check if queue empty only if necessary.
        if (runStateAtLeast(c, SHUTDOWN)
            &amp;&amp; (runStateAtLeast(c, STOP) || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // Are workers subject to culling?
        boolean timed = allowCoreThreadTimeOut || wc &gt; corePoolSize;

        if ((wc &gt; maximumPoolSize || (timed &amp;&amp; timedOut))
            &amp;&amp; (wc &gt; 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

该方法的核心思想也在官方注释中有详解，具体逻辑为：
&#43; 该方法不显式加锁，使用自旋锁，猜测使用了CAS，进行循环
&#43; 首先判断当前线程池状态，若线程池状态为SHUTDOWN且等待队列为空，或者线程池状态为STOP及以上，减少一个工作线程`worker`的计数，返回null告知`runWorker`方法
&#43; 判断当前线程池中的工作线程(`worker`)是否大于最大线程容量（通常为容量大小被动态修改）或者当前`worker`已经超时，并且线程池中的工作线程`worker`大于1（避免`Last Thread Scenario`及不一致性与过渡态，因为该方法的超时判断或容量判断是没有显式加锁的）或等待队列为空
&#43; 判断当前`worker`是否可超时，根据核心线程是否允许超时或者当前工作线程数量大于核心线程数量（当前`worker`为非核心线程）
&#43; 正常通过阻塞队列获取任务，根据是否可超时决定

#### `ThreadPoolExecutor#shutdown()`

``` java
/**
 * Initiates an orderly shutdown in which previously submitted
 * tasks are executed, but no new tasks will be accepted.
 * Invocation has no additional effect if already shut down.
 *
 * &lt;p&gt;This method does not wait for previously submitted tasks to
 * complete execution.  Use {@link #awaitTermination awaitTermination}
 * to do that.
 *
 * @throws SecurityException {@inheritDoc}
 */
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(SHUTDOWN);
        interruptIdleWorkers();
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}
```

该方法会优先获取锁，然后：
&#43; 调用`checkShutdownAccess()`检查是否有权限终止
&#43; 调用`advanceRunState(SHUTDOWN)`CAS将线程池的状态改为SHUTDOWN
&#43; 调用`interruptIdleWorkers()`让空闲的线程中断
&#43; 最后调用`tryTerminate()`尝试中止线程池
\#### `ThreadPoolExecutor#interruptIdleWorkers()`

``` java
/**
 * Interrupts threads that might be waiting for tasks (as
 * indicated by not being locked) so they can check for
 * termination or configuration changes. Ignores
 * SecurityExceptions (in which case some threads may remain
 * uninterrupted).
 *
 * @param onlyOne If true, interrupt at most one worker. This is
 * called only from tryTerminate when termination is otherwise
 * enabled but there are still other workers.  In this case, at
 * most one waiting worker is interrupted to propagate shutdown
 * signals in case all threads are currently waiting.
 * Interrupting any arbitrary thread ensures that newly arriving
 * workers since shutdown began will also eventually exit.
 * To guarantee eventual termination, it suffices to always
 * interrupt only one idle worker, but shutdown() interrupts all
 * idle workers so that redundant workers exit promptly, not
 * waiting for a straggler task to finish.
 */
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            if (!t.isInterrupted() &amp;&amp; w.tryLock()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```

该方法逻辑简单，核心点在于判断语句中的`w.tryLock()`，该语句解释了为什么在`runWorker`中的添加锁是为了防止`shutdown()`时错误停止任务

\*\*`shutdown()`和`tryTerminate()`都会调用这个方法，唯一的区别是参数不同，在注释中有写，当使用`tryTerminate()`方法调用时，仅会中断一个线程，这和前面的`wc &gt; 1`，即Last Thread Scenario相关

#### `ThreadPoolExecutor#tryTerminate()`

``` java
/**
 * Transitions to TERMINATED state if either (SHUTDOWN and pool
 * and queue empty) or (STOP and pool empty).  If otherwise
 * eligible to terminate but workerCount is nonzero, interrupts an
 * idle worker to ensure that shutdown signals propagate. This
 * method must be called following any action that might make
 * termination possible -- reducing worker count or removing tasks
 * from the queue during shutdown. The method is non-private to
 * allow access from ScheduledThreadPoolExecutor.
 */
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        if (isRunning(c) ||
            runStateAtLeast(c, TIDYING) ||
            (runStateLessThan(c, STOP) &amp;&amp; ! workQueue.isEmpty()))
            return;
        if (workerCountOf(c) != 0) { // Eligible to terminate
            interruptIdleWorkers(ONLY_ONE);
            return;
        }

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                    terminated();
                } finally {
                    ctl.set(ctlOf(TERMINATED, 0));
                    termination.signalAll();
                    container.close();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
        // else retry on failed CAS
    }
}
```

该方法可以认为：判断当前线程池是否可以关闭或者已经接近关闭，如果不是（SHUTDOWN且阻塞队列为空或者是STOP），则中断一个空闲状态下的线程，再次自旋尝试

#### `ThreadPoolExecutor#shutdownNow()`

``` java
public List&lt;Runnable&gt; shutdownNow() {
    List&lt;Runnable&gt; tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(STOP);
        interruptWorkers();
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```

该方法与`shutdown()`的区别仅为：1. 将状态设定为STOP。2.调用`interruptWorkers()`不尝试获取锁，若开始直接中断。3. 将队列中等待的任务返回。
其余核心逻辑一致
\## 并发工具类
\### `CountDownLatch`

本质上是使用共享锁机制的AQS实现，初始化锁数量为设定值，因此支持多个线程同时等待多个线程的情况
\### `CyclicBarrier`

与`CountDownLatch`的区别：
- CountDownLatch：
1. 它只能使用一次，是一个一次性的工具
2. 它是一个或多个线程用于等待其他线程完成的同步工具
- CyclicBarrier
1. 它可以反复使用，允许自动或手动重置计数
2. 它是让一定数量的线程在同一时间开始运行的同步工具

该对象内部方法较为简单，维护了一个`ReentrantLock`及其`Condition`对象，直接源码分析即可，注意其可重用性，reset，及broken状态即可
\### `Semaphore`

同样的，本质上是使用共享锁机制的AQS实现
\### `Exchanger`

代码示例：

``` java
public static void main(String[] args) throws InterruptedException {
    Exchanger&lt;String&gt; exchanger = new Exchanger&lt;&gt;();
    new Thread(() -&gt; {
        try {
            System.out.println(&#34;收到主线程传递的交换数据：&#34;&#43;exchanger.exchange(&#34;AAAA&#34;));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }).start();
    System.out.println(&#34;收到子线程传递的交换数据：&#34;&#43;exchanger.exchange(&#34;BBBB&#34;));
}
```

### `Fork/Join Framework`

其核心逻辑可以看作是拆分任务，并使用多线程，即多线程Context下的递归/分治算法，并且可以利用工作窃取算法，提高线程的利用率

&gt; **工作窃取算法：** 是指某个线程从其他队列里窃取任务来执行。一个大任务分割为若干个互不依赖的子任务，为了减少线程间的竞争，把这些子任务分别放到不同的队列里，并为每个队列创建一个单独的线程来执行队列里的任务，线程和队列一一对应。但是有的线程会先把自己队列里的任务干完，而其他线程对应的队列里还有任务待处理。干完活的线程与其等着，不如帮其他线程干活，于是它就去其他线程的队列里窃取一个任务来执行。

Arrays工具类提供的并行排序也是利用了ForkJoinPool来实现。

- ☐ TODO: 单例模式，懒汉，饿汉，静态内部类


&lt;!--more--&gt;


---

> Author: Shiping Guo  
> URL: http://localhost:1313/posts/583bc6c/  

