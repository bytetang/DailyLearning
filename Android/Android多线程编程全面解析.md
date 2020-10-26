多线程是开发中较为高阶的知识，但是涉及到的知识点非常的多。设计或写多线程的代码需要谨慎处理，前提是我们要有比较系统性的认识。

# 一、从Thread Stack认识线程
在android开发中我们通常会在发生卡顿或者ANR的时候，去拿手机里的traces.txt做分析，这里面dump了ANR出现时的线程状态。我们以traces.txt里面的堆栈为例，认识下线程。

位置: data/anr/traces.txt

cmd: `adb pull data/anr/traces.txt ~/Downloads/anr/`

```
"main" prio=5 tid=1 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x73fa0bb0 self=0x7efc4a3a00
  | sysTid=21720 nice=0 cgrp=default sched=0/0 handle=0x7f014829b0
  | state=S schedstat=( 180625504 192128139 822 ) utm=11 stm=6 core=1 HZ=100
  | stack=0x7fd0700000-0x7fd0702000 stackSize=8MB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/21720/stack)
  native: #00 pc 0000000000069068  /system/lib64/libc.so (__epoll_pwait+8)
  native: #01 pc 000000000001f640  /system/lib64/libc.so (epoll_pwait+48)
  native: #02 pc 0000000000015c80  /system/lib64/libutils.so (_ZN7android6Looper9pollInnerEi+144)
  native: #03 pc 0000000000015b68  /system/lib64/libutils.so (_ZN7android6Looper8pollOnceEiPiS1_PPv+108)
  native: #04 pc 000000000011a59c  /system/lib64/libandroid_runtime.so (???)
  native: #05 pc 000000000020081c  /system/framework/arm64/boot-framework.oat (Java_android_os_MessageQueue_nativePollOnce__JI+140)
  at android.os.MessageQueue.nativePollOnce(Native method)
  at android.os.MessageQueue.next(MessageQueue.java:379)
  at android.os.Looper.loop(Looper.java:144)
  at android.app.ActivityThread.main(ActivityThread.java:7425)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.Zygote$MethodAndArgsCaller.run(Zygote.java:245)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:921)
```

Thread 状态字段分析：

  字段  | 分析
  ---- | ----
  main | 线程名,这里表示主线程
  prio | 线程优先级，在android中value = [1-10] ,值越大，优先级越高。默认的优先级是5,Android中优先级[-20,20],-20最高，默认为0；
  tid | 线程的id
  group | 线程所在的线程组，线程组是个树结构，可以用来批量的管理线程或者线程组。
  sCount | 线程挂起的次数
  dsCount | 因为调试挂起的次数。
  obj | 当前线程java对象的内存地址
  self |  当前线程Native的地址
  sysTid | 线程号，主线程的线程好和进程号相同
  nice | 默认值是0，区别于prio优先级,属于非实时的优先级，一般不关注
  cgrp | 所属进程的调度组
  shed | CPU调度的策略
  handle | 函数处理的地址
  state | 线程的状态，S=Sleeping.具体的参考https://developer.android.com/reference/java/lang/Thread.State
  schedstat | cpu调度的时间统计，( 180625504 192128139 822 )第一个值为cpu运行的时间(ns);第二个值是在队列等待的时间（ns);第三个值为cpu切换的次数;utm为该线程在用户态执行的时间(jiffies);stm内核执行的时间(jiffies);
  core |  该线程的最后运行所在核的序号
  stack | 线程栈的地址范围
  stackSize | 栈的大小
  held mutexes | 所持有的锁的类型，exclusive lock(独占锁)/ shared lock(共享锁)，决定了是不是其他线程可以获取锁。

后面的部分为native和java的堆栈信息

`native: #00 pc 0000000000069068  /system/lib64/libc.so (__epoll_pwait+8)`

字段  | 分析
----- | -----
native | 标记是native堆栈
#00 | 堆栈行数
pc 0000000000069068 | 程序计数器，记录在so中的偏移位置，这里的偏移值是0x69068
__epoll_pwait+8 | 标记在函数__epoll_pwait偏移量为8的位置的指令

解析下来就是,第一个行堆栈产生是来自libc.so库中偏移量为0x69068的位置（也就是函数名__epoll_pwait+8)发生了调用。
  

认识了一个线程有哪些状态（构成），基本上也初步认识了线程，但是线程是怎么工作的，还要从它的生命周期说起。接下来再了解下线程的生命周期和装换状态。

# 二、线程的状态转换

![image](https://note.youdao.com/yws/api/personal/file/WEBafda993df9992cf5b3a136ebad778f7e?method=download&shareKey=512e8dfcbf98cb5bdc9aab7349a22b78)

图片已经有很多前辈们总结了，就直接拿过来分析了。

- 新建(New)

  线程尚未开启的时候。

- 可运行(Runnable)

  线程对象已经创建，等待执行，改状态的线程可能在线程池中，等待被调度

- 运行 (Running)

  线程获取到了cpu时间片，开始运行。

- 阻塞 (BLOCKED)

  线程正在等待获取锁(monitor lock)的状态，比如A线程的Synchronized代码块尚未结束，当前线程B会被阻塞执行，一直等到A释放锁。

- 等待 (Watting)

  线程处于等待状态，等待的原因是调用了以下方法：
  - Object.wait 
  - Thread.join 
  - LockSupport.park

- 定时等待(TIMED_WAITING)

  线程处于等待状态，但会设定等待时间，一般是调用了以下方法：
  - Thread.sleep
  - Object.wait with timeout
  - Thread.join with timeout
  - LockSupport.parkNanos
  - LockSupport.parkUntil

- 死亡(Dead)
  线程run结束或退出，生命周期结束

这里面有几个概念需要区分清楚：

1. <b>Blocked 和 WAITING</b>

Blocked状态线程是会放弃cpu的占用，一旦锁被释放，jvm会自动解除Blocked的状态，系统自己的行为。Watting的阻塞会一直吃需要有消息通知才会接触，如调用notify()/notifyAll的方法。


2. <b>WATING vs TIMED_WAITING</b>

TIMED_WAITING状态的线程阻塞会有时间限制，一旦过了超时时间，线程将重新获取cpu运行程序,Thread.sleep(timeout)就属于这种类型。无需像WATING状态的线程一样等待notify。

3. <b>Blocked & WAITING & TIMED_WAITING 状态都是不会获取cpu时间片的，也就是不会占用CPU的资源。</b>

<br>
# 三、锁以及锁的优化

为了保证线程安全，我们通常都是借助锁（如中synchronize）来完成线程间的协作。

首先，了解JVM中锁是怎么实现的有助于我们更深次的了解锁以及锁的优化逻辑，在实践中正确的使用锁。

## 3.1 锁的实现原理

<b>认识jvm中对象的结构</b>


HotSpot虚拟机中，对象在内存中存储的布局可以分为三块区域：对象头（Header）、实例数据（Instance Data）和对齐填充（Padding）。  

![image](https://note.youdao.com/yws/api/personal/file/WEB4f26e3e42a970e622fde2eacfc1e300b?method=download&shareKey=ba3c7ea24634c8e53611bdc956ad841d)


看一下对象头的结构：定义在opp.hpp—>oopDesc

oopDesc定义了两个变量:

```
volatile markOop  _mark;

Klass*      _klass;
```

Klass的作用是把C++的class对象包装成java heap 对象。<b>_mark(mark_word)存储对象自身的运行数据，如Object hash,GC分代年龄，还有就是包含我们后面要介绍的锁的信息。</b> 查看hotspot 代码（jdk8)：[oop.hpp](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/oops/oop.hpp), [markOop.hpp](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/oops/markOop.hpp)

32位markword存储结构：

25 bit | 4bit | 1bit | 2bit
---- | ---- | ---- | ----
object hash | GC Generation age | 是否偏向锁 | 锁标志位

线程通过拷贝markword记录来进行锁的加锁和释放。线程栈存储一个叫Lock Record的空间，主要用于拷贝对象markword的记录，一旦拷贝成功，就说明当前线程拥有了这个对象的锁。

1bit（是否偏向锁）表示对象锁是否可以偏向，线程在标志为为1的情况下会去尝试使用偏向锁竞争。

最低位2位锁标志位记录了当前几种锁的状态，在线程锁的竞争和优化过程会更新状态。目前有状态有：

状态 | 值 | 说明
---- | ---- | ----
locked | 00 | 处于锁定状态,即线程栈的锁指针指向在object header。
unlocked | 01 | 锁已经被释放
monitor | 10 | 锁在膨胀的状态（如自旋锁转成重量级锁）
marked | 11 | 用来标记对象在任何其他时间无效

这里面涉及到了很多概念，比如<b>偏向锁</b>，<b>自旋锁</b>，<b>重量级锁</b>等，这些锁的设计是用于锁的优化的。很多时候我们理解syncnorize只是粗暴的同步代码块，其实不然，虚拟机的设计对synchnorized锁做了很深层次的优化。

> 以上关于对象结构和字段说明建议大家去读一首的资料，就是相关类的代码注释，比较详细。

下面对其优化过程做个详细的分析

## 3.2 锁以及锁的优化 

承上说的在使用syschnorized其实虚拟机会有个自发的优化过程，这个过程就是锁的膨胀过程，java包含的锁类型有偏向锁，轻量级锁，自旋锁以及重量级锁，锁的膨胀过程就是这些状态之前的转换过程。

### 3.2.1 为什么存在锁的优化问题？

了解线程调度的同学都知道，java是通过抢占式的调度方式来竞争的，系统会控制时间片的分配和线程的切换。线程切换是有开销的，线程的开销主要是在于每次切之前需要保存任务的状态，下次切回这个线程的时候再加载状态。

并发编程为了保证共享变量的安全又不得不使用锁，多线程在竞争锁的时候回引发上线文切换，并且一个线程持有锁会导致其它所有需要此锁的线程都会被挂起。如果需要更高的并发性能，就会需要锁的优化。


### 3.2.2 悲观锁 vs 乐观锁

在java中我们所说的悲观锁乐观锁是一种延伸的意义，其明确定义出自关系型数据库并发控制：

悲观并发控制（又名“悲观锁”，Pessimistic Concurrency Control，缩写“PCC”）是一种并发控制的方法。它可以阻止一个事务以影响其他用户的方式来修改数据。如果一个事务执行的操作读某行数据应用了锁，那只有当这个事务把锁释放，其他事务才能够执行与该锁冲突的操作。

乐观并发控制（又名“乐观锁”，Optimistic Concurrency Control，缩写“OCC”）是一种并发控制的方法。它假设多用户并发的事务在处理时不会彼此互相影响，各事务能够在不产生锁的情况下处理各自影响的那部分数据。在提交数据更新之前，每个事务会先检查在该事务读取数据后，有没有其他事务又修改了该数据。如果其他事务有更新的话，正在提交的事务会进行回滚。

> 我们可以看出来乐观锁相对于悲观锁来说，具备更高的并发性能。乐观锁的实现基于共享状态的check，可以减少上下文的切换，竞争的线程也不会因为未竞争到资源而被挂起。

### 3.2.3 内存同步机制-Volatile

线程安全严格来说需要保证线程间对共享变量访问的有序性、可见性、原子性,但是往往某些场景我们会使用开销更小的方案来保持平衡，如:使用volatile关键字，volatile不是严格意义上的锁，只是利用了内存同步机制来实现。



case：保证获取单例的有序性：

```java
public class Singleton {
   private volatile static Singleton instance; //声明成 volatile
   private Singleton (){}

   public static Singleton getSingleton() {
       if (instance == null) {                        
           synchronized (Singleton.class) {
               if (instance == null) {      
                   instance = new Singleton();
               }
           }
       }
       return instance;
   }
}
```

这样就可以避免获取单例方法的双重加锁。

volalite不能严格保证原子性，只能保证单个操作的原子性，复合操作就会出错。

>原子操作(atomic operation)是指不会被线程调度机制打断的操作；这种操作一旦开始，就一直运行到结束，中间不会有任何 context switch （切换到另一个线程）。

不能保证原子性的根本原因：内存访问乱序问题
如下两个CPU都做如下比较赋值操作等：

```
CPU 1		CPU 2
===============	===============
{ A == 1; B == 2 }
A = 3;		x = A;
B = 4;		y = B;
```

内存系统能看见的访问顺序可能会是如下的次序：

![ram_access](https://note.youdao.com/yws/api/personal/file/WEB1a19eed1aa03508d50e000b66d22a41e?method=download&shareKey=7e1aee78feff1e57c6406fd6b935650d)

为了保证多核内存访问次序的正确性，操作系统引入内存屏障来解决（memory barrier
）

插入一个内存屏障，相当于告诉CPU和编译器先于这个命令的必须先执行，后于这个命令的必须后执行。内存屏障另一个作用是强制更新一次不同CPU的缓存。

<b>内存屏障（memory barrier）和volatile什么关系？</b>

从Load到store到内存屏障，一共4步，其中最后一步jvm让这个最新的变量的值在所有线程可见，也就是最后一步让所有的CPU内核都获得了最新的值，但中间的几步（从Load到Store）是不安全的，中间如果其他的CPU修改了值将会丢失。

volatile保证变量对线程的可见性正式利用了内存屏障，但原子性并不能得到保证。


<b>复合操作指的是？</b>

如：int i = 0; i++;

内存中i++的步骤包含load、Increment、store


复合操作有哪些：


根本原因是因为volalite为了保证操作的有序性设定的重排序策略，如，A线程第一个操作如果是volalite多或写操作，后续A线程第二个操作是什么（这里假设是写变量操作）都不能重排序，B线程的volalite操作如果发生在A线程之前，就没法获取B的变量修改值，就不能保证B线程中变量读取的准确性。voliate的机制大有文章，具体的原理可以去深入了解下。

所以，volalite虽然可以提升性能，有一定的适用场景，但是并不能严格意义上来保证线程安全。如果需要严格意义的线程安全能平衡性能，可以选择CAS的机制。

### 3.2.4 乐观锁实现：CAS

CAS属于乐观锁，可以在不使用锁的情况下实现多线程之间的变量同步，也就是在没有线程被阻塞的情况下实现变量的同（Non-blocking Synchronization）

算法原理（Compare and Exchange）-> 操作三个变量值
- 需要读写的内存值 V
- 进行比较的值 A
- 拟写入的新值 B

B新值插入的条件当前仅当V=A的时候成立。即在插入B新值之前先检查一遍内存值和旧的值是否是一致的（未被修改），确认一致才去修改变量value为B

jdk中java.util.concurrent.atomic包下面提供了很多的类似的cas工具，其中较为常见的有AtomicInteger，AtomicBoolean。

他们都是借助系统指令CMPXCHG来完成算法的，[CMPXCHG指令详情看这里](https://www.felixcloutier.com/x86/cmpxchg)。

> CAS作为乐观锁也是存在问提的，即ABA问题，有很多文档不做详述。


### 3.2.5 Syschnorized原理与优化

有前面的知识作为铺垫，回到虚拟机的锁的优化问题，java虚拟机使用了哪些方案去优化锁呢，以及锁的状态装换流程是怎么样的呢？

<b>偏向锁（Biased Locking)</b>

是Java6引入的一项多线程优化。它会偏向于第一个访问锁的线程，如果在运行过程中，同步锁只有一个线程访问，不存在多线程争用的情况，则线程是不需要触发同步的，这种情况下，就会给线程加一个偏向锁。 
如果在运行过程中，遇到了其他线程抢占锁，则持有偏向锁的线程会被挂起，JVM会消除它身上的偏向锁，将锁恢复到标准的轻量级锁。


<b>轻量级锁</b>

轻量级锁是由偏向所升级来的，偏向锁运行在一个线程进入同步块的情况下，当第二个线程加入锁争用的时候，偏向锁就会升级为轻量级锁。它不是使用操作系统互斥量来实现锁， 而是通过CAS实现。当线程获得轻量级锁后，可以再次进入锁，即锁是可重入（Reentrance Lock）的。

<b>重量级锁</b>

重量级锁(monitor)就是一个互斥的锁了，这种同步方式的成本非常高，包括系统调用引起的内核态与用户态切换、线程阻塞造成的线程切换等。

在"锁的实现原理"那一节我们已经分析解释过分析锁的标志位的用途了，结合锁的装换状态及对象Mark Word的关系图我们可以更深入理解mark word的工作原理：

![偏向锁、轻量级锁的装换状态及对象Mark Word的关系](https://note.youdao.com/yws/api/personal/file/WEB8ce83ec60da33ae7eb06b7c925be1946?method=download&shareKey=d34fe90e0234c3725ab3a15b860bf211)

归纳mark word标志位状态如下：

是否偏向锁 |  锁标志位 | 对象锁的状态说明
---------- | ----------| -------------
1          | 01        | 可偏向的对象，处于偏向锁竞争锁定流程
0          | 01        | 未锁定，不可偏向的对象
--         | 00        | 已经被轻量级锁定的对象
--         | 10        | 被重量级锁定的对象

通过分析mark word就可以清楚地知道当前的锁的装换状态处于什么样一个阶段。

<b>线程是怎么从偏向锁如何转升级为轻量级锁以及轻量级是如何膨胀成重量级锁的。</b>

1.偏向锁的流程图：

![](https://note.youdao.com/yws/api/personal/file/WEBfaf01423a1c2da9508efcedbfe435708?method=download&shareKey=1537fafd0bae0b8ea3dcaa1f1eee066c)

如果CAS尝试竞争成功，就直接执行代码块。如果失败，发现有其他线程在竞争撤销偏向锁，升级成为轻量级锁。

2.升级为轻量级锁线程尝试使用CAS将对象头中的Mark Word替换为指向锁记录的指针。如果成功,当前线程获得锁,如果失败,表示其他线程竞争锁,当前线程便尝试使用<b>自旋</b>来获取锁。

>自旋锁尽可能的减少线程的阻塞，这对于锁的竞争不激烈，且占用锁时间非常短的代码块来说性能能大幅度的提升，因为自旋的消耗会小于线程阻塞挂起再唤醒的操作的消耗。

3.轻量级锁自旋获取锁依然失败，升级为重量级锁。

# 四、性能优化与测试
在并发编程中线程安全是我们首要要关注的问题，它是保证程序稳定运行的前提。但是锁的引入可能也是性能上的杀手。

## 4.1 多线程性能问题诱因

导致并发性能的主要诱因是"上下文切换"。

<b>上线文切换</b>

在多核的操作系统里，当一个线程被切换进来时，它所需要的数据可能不在当前处理器的缓存里，缓存的确实会使得这个线程初次的切换带来更多的开销。通常一次上下文的开销相当于5000~10000个cp4时钟周期。

引发上下切换的诱因是:<b>阻塞IO</b>,<b>等待获取竞争锁</b>等。

当线程发生阻塞或者等待获取竞争锁（cas,voliate都属于非竞争同步，JVM自行调度，无需操作系统介入)时，需要被挂起，这个过程将包含两次额外的上下文切换，被阻塞的线程在其执行的时间片还没有用完之前就被交换出去，而在随后的获取锁或者资源可用的时候再切换回来。

调用哪些方法会引起上线文切换（常见）：

- Thread.sleep(long millis)
- Object,wait(long timeout)
- Thread.join()/Thread.join(long timeout)

> 性能的影响因素也包含如内存同步等，这里只说上下文切换是主要诱因，可能人为的做优化的部分。

## 4.2 性能优化方案

<b>4.2.1 减少锁力粒度</b>

如果我们给public类每个方法加synchnorized(this)的同步，线程安全确实没问题。但是基本也等同于串行操作，违背了并发的本意。

在编码的时候尽量避免锁的力度太粗。避免占用锁的时间过长。

case：

```java
public class Grocery {
    private final ArrayList fruits = new ArrayList();
    private final ArrayList vegetables = new ArrayList();
    public synchronized void addFruit(int index, String fruit) {
        fruits.add(index, fruit);
    }
    public synchronized void removeFruit(int index) {
        fruits.remove(index);
    }
    public synchronized void addVegetable(int index, String vegetable) {
        vegetables.add(index, vegetable);
    }
    public synchronized void removeVegetable(int index) {
        vegetables.remove(index);
    }
}
```
这里可以看出来在添加和移除水果的时候也会阻塞蔬菜的添加和删除，对于水果和蔬菜的增删操作都是方法域的同步。如果拆分下锁：

```java
public class Grocery {
    private final ArrayList fruits = new ArrayList();
    private final ArrayList vegetables = new ArrayList();
    public void addFruit(int index, String fruit) {
        synchronized(fruits) fruits.add(index, fruit);
    }
    public void removeFruit(int index) {
        synchronized(fruits) {fruits.remove(index);}
    }
    public void addVegetable(int index, String vegetable) {
        synchronized(vegetables) vegetables.add(index, vegetable);
    }
    public void removeVegetable(int index) {
        synchronized(vegetables) vegetables.remove(index);
    }
}
```
锁的范围就被缩小到了针对每个资源fruits和vegetables的锁。通常这样的方式阻塞出现的情况会更加少，性能会有较大的提升。

<b>4.2.2 使用非竞争同步</b>

在cas和voliate关键字适用的场景可以尽量去选择使用此类乐观锁。

<b>4.2.3 分段锁</b>

分解锁可以带来一定的伸缩性和性能的提升，但是如果竞争条件还是比较激烈的话，还是会有问题，这时候可以通过分段锁来优化。jdk 1.7中的线程安全集合类ConcurrentHashMap就是利用了这个技术来实现高并发。

ConcurrentHashMap把容器分为多个 segment（片段） ，每个片段有一把锁，当多线程访问容器里不同数据段的数据时，线程间就不会存在竞争关系；一个线程占用锁访问一个segment的数据时，并不影响另外的线程访问其他segment中的数据。

```java
@SuppressWarnings("unchecked")
    public V put(K key, V value) {
        Segment<K,V> s;
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key.hashCode()); //获取hash值
        int j = (hash >>> segmentShift) & segmentMask; //mask计算出资源散列在哪个片段中
        if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
             (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
            s = ensureSegment(j);
        return s.put(key, hash, value, false);
    }
```

ensureSegment找到segment位置之后，就可以开始执行Segment put操作，Segment.put被使用开销更小的ReentrantLock.trylock即非公平锁。

> 非公平锁（Nonfair）：加锁时不考虑排队等待问题，直接尝试获取锁，获取不到自动到队尾等待。


<b>4.2.4 ReadWriteLock</b>

如果资源可以被多个读线程访问，或者只有一个写线程访问，可以使用读写锁。也就是存在多个线程同时存在读和写的场景不适用。

资源并发读写：

![readwitelock](https://note.youdao.com/yws/api/personal/file/WEBf11148170f2eeca961db80c60c7b750e?method=download&shareKey=4af004ea7a6c3a9e16a5fa072707097d)

红色部分表示被阻塞的线程

使用读写锁可以带来性能上的优化。Java 中的实现有ReentrantReadWriteLock。

实例读锁和写锁

```java
private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
private final Lock r = rwl.readLock();
private final Lock w = rwl.writeLock();
```

获取锁和释放锁
```java
lock()
unlock()
```

> 读锁的性能高于独占锁，读锁区别于写锁使用的是共享锁，具有更小的开销。

## 4.3 编写测试代码

多线程的正确性通常很难通过肉眼甚至是正常测试回归发现。对于我们的并发代码的正确性需要更多的依赖测试代码的验证，这一点很重要。

但是编写多线程的单元测试通常比写多线程的逻辑代码更难，需要一些技巧。

<b>4.3.1 覆盖基本的后验条件</b>

首先假定在串行的条件下，是否能正确的都到正确的结果。使用单元测试先去覆盖掉。比如队列在创建之前应该为空的严重，完成之后后验结果的准确性。确保非并发的环境下程序是争取的，这样可以避免把所有错误怀疑到多线程上，也可以为多线程的后续结果验证提供基础。

<b>4.3.2 使用栅栏模式(CyclicBarrier)设计"同时到达"</b>

CyclicBarrier可以使线程互相等待，任何一个线程达到某个状态之前，所有的线程都必须等待。等到所有线程都达到某个状态，线程才能继续执行。

CyclicBarrier的代码模型：

```java
//设计50个测试线程
int HREADS = 50 ;

CyclicBarrier barries = new CyclicBarrier(HREADS);

//in thread,wait to start
barries.await()

//do stuff

//in thread,wait to stop
barries.await();

```

case:

```java
class Party extends Thread {
    private int duration;
    private CyclicBarrier barrier;

    public Party(int duration, CyclicBarrier barrier, String name) {
        super(name);
        this.duration = duration;
        this.barrier = barrier;
    }

    @Override
    public void run() {
        try {
            Thread.sleep(duration);
            System.out.println(Thread.currentThread().getName() + " is calling await()");
            barrier.await();
            System.out.println(Thread.currentThread().getName() + " has started running again");
        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }
    }
}
```

Test:

```java
public static void main(String args[]) throws InterruptedException, BrokenBarrierException {
        
        CyclicBarrier barrier = new CyclicBarrier(4);
        Party first = new Party(1000, barrier, "PARTY-1");
        Party second = new Party(2000, barrier, "PARTY-2");
        Party third = new Party(3000, barrier, "PARTY-3");
        Party fourth = new Party(4000, barrier, "PARTY-4");
        
        first.start();
        second.start();
        third.start();
        fourth.start();
        
        System.out.println(Thread.currentThread().getName() + " has finished");

    }

```

<b>4.3.3 产生更多的交替操作</b>

低概率的事件即使是模拟多线程测试也很难验证出来，需要让程序产生更多的交替操作大概率去提高并发隐藏问题出现的概率。

借助Thread.sleep()/Thread.yield()，通常加到需要验证的逻辑之前。

```java
 
    public void run() {
        try {
           Thread.sleep(2);
           
           //do stuff
        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }
    }
```

Thread.sleep()/Thread.yield()都会阻塞使CPU让出时间片,让竞争更为激烈。