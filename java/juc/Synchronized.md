
# 前言


参阅本篇文章时，可以带着以下问题：
- Synchronized可以作用在哪里? 分别通过对象锁和类锁进行举例。 
- Synchronized本质上是通过什么保证线程安全的? 分三个方面回答：加锁和释放锁的原理，可重入原理，保证可见性原理。 
- Synchronized由什么样的缺陷?  Java Lock是怎么弥补这些缺陷的？
- Synchronized和Lock的对比，如和选择? 
- Synchronized在使用时有何注意事项? 
- Synchronized修饰的方法在抛出异常时,会释放锁吗? 
- 多个线程等待同一个snchronized锁的时候，JVM如何选择下一个获取锁的线程?
- Synchronized使得同时只有一个线程可以执行，性能比较差，有什么提升的方法? 
- 我想更加灵活地控制锁的释放和获取(现在释放锁和获取锁的时机都被规定死了)，怎么办? 
- 什么是锁的升级和降级? 
- 什么是JVM里的偏向锁、轻量级锁、重量级锁? 
- 不同的JDK中对Synchronized有何优化?

# 一、synchronized
## 临界区和静态条件
在讲synchronized关键字之前，先要了解两个概念：临界区和静态条件。
**临界区：**
- 一个程序运行多个线程本身是没有问题的
- 问题出在多个线程访问共享资源
--  多个线程读共享资源其实也没有问题
-- 在多个线程对共享资源读写操作时发生指令交错，就会出现问题
- 一段代码块内如果存在对共享资源的多线程读写操作，称这段代码块为临界区
例如，下面代码中的临界区
```java
static int counter = 0;
static void increment() 
// 临界区 
{   
    counter++; 
}
static void decrement() 
// 临界区 
{ 
    counter--; 
}
```
**竞态条件**
多个线程在临界区内执行，由于代码的执行序列不同而导致结果无法预测，称之为发生了竞态条件

## 使用synchornized及其作用
来看下面的代码：
```java
 static int res = 0;
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(()->{
            for(int i = 0;i < 50000;i ++) {
                res ++;
            }
        },"t1");

        Thread t2 = new Thread(()->{
            for(int i = 0;i < 50000;i ++) {
                res --;
            }
        },"t2");
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(res);
    }
```
如果不考虑线程安全问题，实际输出结构应该为0。但事实并非如此，可以自行实验。这是由于底层的JVM实现中，将Java转换为字节码
```
 		 0: getstatic     #2                  // 获取 res
         3: iconst_1							// 获取常量 1
         4: iadd								// 将res和1相加
         5: putstatic     #2                  // 放回res，下面同理
         8: getstatic     #2                  // Field res:I
        11: iconst_1
        12: isub
        13: putstatic     #2                  // Field res:I
```
其中0、3、4、5是i ++ 的操作，而8，11，12，13是i -- 的操作，具体可以看注释，JVM内容就不展开了。当这8条指令顺序执行的时候，是没有问题的，结果为0；但在多线程的环境下，这并不是一个原子性的操作，有可能会在第二条或者第三条的时候停止，切换到另一个线程去执行。举个例子：
假设i = 0；
当线程1执行i ++ 操作时，先执行0、3、4三步，但并没有保存i变量；在此时，CPU分配的时间片到了，上下文切换，轮到线程2执行i -- 操作，当他获取i的值的时候，上一个i ++ 的结果并没有放回，所以他获取到的值为0，然后执行8、11、12、13四条操作指令后放回为-1，然后回到线程1操作，放回变量i为1，跟我们预想的结果并不一致。这就是造成线程不安全的原因。

## synchronized使用
使用synchronized关键字可以解决这个问题。synchronized它采用互斥的方式让同一 时刻至多只有一个线程能持有锁，其它线程再想获取这个锁时就会阻塞住(blocked)。这样就能保证拥有锁的线程可以安全的执行临界区内的代码，不用担心线程上下文切换。

应用Sychronized关键字时需要把握如下注意点： 
- 一把锁只能同时被一个线程获取，没有获得锁的线程只能等待；
-  每个实例都对应有自己的**一把锁**(this),不同实例之间互不影响；例外：锁对象是*.class以及synchronized修饰的是static方法的时候，所有对象公用同一把锁 
- synchronized修饰的方法，无论方法正常执行完毕还是抛出异常，都会释放锁
所以上述代码修改一下，就能解决安全问题。
```java
 static int res = 0;
    static Object lock = new Object();
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(()->{
           synchronized (lock) {
               for(int i = 0;i < 50000;i ++) {
                   res ++;
               }
           }
        },"t1");

        Thread t2 = new Thread(()->{
            synchronized(lock) {
                for(int i = 0;i < 50000;i ++) {
                    res --;
                }
            }
        },"t2");
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(res);
    }
```
## 对象锁和类锁
包括方法锁(默认锁对象为this,当前实例对象)和同步代码块锁(自己指定锁对象)
### 对象锁
**代码块形式：手动指定锁定对象，也可是是this,也可以是自定义的锁**
上面的代码就是同步代码块形式，就不举例子了。

**方法锁形式：synchronized修饰普通方法，锁对象默认为this**
```java
public class SynchronizedObjectLock implements Runnable {
    static SynchronizedObjectLock instence = new SynchronizedObjectLock();

    @Override
    public void run() {
        method();
    }

    public synchronized void method() {
        System.out.println("我是线程" + Thread.currentThread().getName());
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + "结束");
    }

    public static void main(String[] args) {
        Thread t1 = new Thread(instence);
        Thread t2 = new Thread(instence);
        t1.start();
        t2.start();
    }
}
  
```
### 类锁
指synchronize修饰静态的方法或指定锁对象为Class对象

**synchronize修饰静态方法**
```java
public class SynchronizedObjectLock implements Runnable {
    static SynchronizedObjectLock instence1 = new SynchronizedObjectLock();
    static SynchronizedObjectLock instence2 = new SynchronizedObjectLock();

    @Override
    public void run() {
        method();
    }

    // synchronized用在静态方法上，默认的锁就是当前所在的Class类，所以无论是哪个线程访问它，需要的锁都只有一把
    public static synchronized void method() {
        System.out.println("我是线程" + Thread.currentThread().getName());
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + "结束");
    }

    public static void main(String[] args) {
        Thread t1 = new Thread(instence1);
        Thread t2 = new Thread(instence2);
        t1.start();
        t2.start();
    }
}
```

**synchronized指定锁对象为Class对象**
```java


public class SynchronizedObjectLock implements Runnable {
    static SynchronizedObjectLock instence1 = new SynchronizedObjectLock();
    static SynchronizedObjectLock instence2 = new SynchronizedObjectLock();

    @Override
    public void run() {
        // 所有线程需要的锁都是同一把
        synchronized(SynchronizedObjectLock.class){
            System.out.println("我是线程" + Thread.currentThread().getName());
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName() + "结束");
        }
    }

    public static void main(String[] args) {
        Thread t1 = new Thread(instence1);
        Thread t2 = new Thread(instence2);
        t1.start();
        t2.start();
    }
}
```

## synchronized原理分析
要分析synchronized原理，首先要知道monitor概念。
### Monitor概念
Monitor被翻译为**监视器**或者**管程**
![在这里插入图片描述](http://badwomen.asia/194b23141c994dbe8302811b2cfc7150.png)
Monitor之中一共有三个部分，WaitSet、EntryList和Owner。其中Owner表示当前持有Monitor的线程。WaitSet存放正在等待的线程。EntryList主要存放等待争抢锁的线程，他们是堵塞的。
当线程执行到临界区的代码，如果使用了synchronized关键字，会先查询synchronized所指定的对象(obj)是否绑定了Monitor。

- 假设Thread-2线程来到了临界区，执行synchronized(obj)，他会先去查询obj有没有绑定，如果没有则会先去去与Monitor绑定，并且将Owner设为当前线程。
- 如果此时又来了另一个线程Thread-1，也执行了synchronized(obj)，他也会去查询obj有没有绑定，此时发现obj已经被绑定了，就会去查看obj所绑定的Monitor的Owner是否有对象（此时Owner被Thread-2所占有），如果有，说明他无法占有该锁，则会进入EntryList，等待前一个线程执行完，唤醒他们。此时是阻塞状态。
- Thread-2执行完同步块代码，唤醒EntryList所有线程去竞争锁，竞争的时候是非公平的（先来的不一定先使用）。


### Monitor在Java中的使用
为什么要讲解Monitor？因为Monitor是synchronized的底层原理。
举个例子：
```java
public static void main(String[] args) {

        Object object = new Object();

            synchronized (object) {
                int i = 0;
            }
    }
```
这是最简单的一个synchronized的使用。
然后我们去获取它的字节码（JavaC 编译然后Javap进行反编译就可以获得）
```
  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=5, args_size=1
         0: new           #2                  // class java/lang/Object
         3: dup										
         4: invokespecial #1                  // Method java/lang/Object."<init>":()V
         7: astore_1								
         8: aload_1
         9: dup
        10: astore_2
        11: monitorenter
        12: iconst_0
        13: istore_3
        14: aload_2
        15: monitorexit
        16: goto          26
        19: astore        4
        21: aload_2
        22: monitorexit
        23: aload         4
        25: athrow
        26: return
      Exception table:
         from    to  target type
            12    16    19   any
            19    23    19   any
      LineNumberTable:
        line 4: 0
        line 6: 8
        line 7: 12
        line 8: 14
        line 9: 26
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 19
          locals = [ class "[Ljava/lang/String;", class java/lang/Object, class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 6
}
```
解释一下：
```
 		 0: new           #2                  // class java/lang/Object
         3: dup										
         4: invokespecial #1                  // Method java/lang/Object."<init>":()V
         7: astore_1								
         8: aload_1
         9: dup
        10: astore_2
```
新建一个obj对象，放入局部变量表1号槽位，复制一份放入局部变量表2号槽位；为什么要复制一份？为了解锁用的
```
		11: monitorenter
        12: iconst_0
        13: istore_3
        14: aload_2
        15: monitorexit
        16: goto          26
        26: return
```
这就是synchronized代码的字节码，可以看到两个关键字`monitorenter`和`monitorexit`。也就是上面讲的monitor。
其中`Monitorenter`和`Monitorexit`指令，会让对象在执行，使其锁计数器加1或者减1。每一个对象在同一时间只与一个monitor(锁)相关联，而一个monitor在同一时间只能被一个线程获得，一个对象在尝试获得与这个对象相关联的Monitor锁的所有权的时候，`monitorenter`指令会发生如下3中情况之一：
- monitor计数器为0，意味着目前还没有被获得，那这个线程就会立刻获得然后把锁计数器+1，一旦+1，别的线程再想获取，就需要等待
- 如果这个monitor已经拿到了这个锁的所有权，又重入了这把锁，那锁计数器就会累加，变成2，并且随着重入的次数，会一直累加
- 这把锁已经被别的线程获取了，等待锁释放


`monitorexit`指令：释放对于monitor的所有权，释放过程很简单，就是讲monitor的计数器减1，如果减完以后，计数器不是0，则代表刚才是重入进来的，当前线程还继续持有这把锁的所有权，如果计数器变成0，则代表当前线程不再拥有该monitor的所有权，即释放锁。

顺利运行完毕，跳转到26行return。
```
		19: astore        4
        21: aload_2
        22: monitorexit
        23: aload         4
        25: athrow
        26: return
```
那我们发现中间还多了一些代码。再往下看能看到一个table
```
Exception table:
         from    to  target type
            12    16    19   any
            19    23    19   any
```
这是异常的表，意思是从12-16不管发生任何一场，都跳转到19；19-23无论发生任何异常，也跳转到19。这部分代码能够保证synchronized锁能够正确的释放。
### 可重入原理
在刚才的代码中讲到了`Monitorenter`和`Monitorexit`指令，其中提到了可重入，那什么是可重入呢？上面的demo中在执行完同步代码块之后紧接着再会去执行一个静态同步方法，而这个方法锁的对象依然就这个类对象，那么这个正在执行的线程还需要获取该锁吗? 答案是不必的，从上图中就可以看出来，执行静态同步方法的时候就只有一条monitorexit指令，并没有monitorenter获取锁的指令。这就是锁的重入性，即在同一锁程中，线程不需要再次获取同一把锁。 
Synchronized先天具有重入性。每个对象拥有一个计数器，当线程获取该对象锁后，计数器就会加一，释放锁后就会将计数器减一。
### java对象头
Java对象保存在内存中时，由以下三部分组成：对象头、实例数据、对齐填充字节。这里只细讲对象头中的MarkWord属性，其余感兴趣的可以去JVM文章中了解。
ava的对象头由以下三部分组成：
- 1，Mark Word
- 2，指向类的指针
- 3，数组长度（只有数组对象才有）

以下拿64位虚拟机的Mark Word举例
![在这里插入图片描述](http://badwomen.asia/cfbea7e88c394c3eb8e1b2fd9128ae69.png)
 记住这张对象头格式。

## JVM中锁的优化
简单来说在JVM中**monitorenter**和**monitorexit**字节码依赖于底层的操作系统的Mutex Lock来实现的，但是由于使用Mutex Lock需要将当前线程挂起并从用户态切换到内核态来执行，这种切换的代价是非常昂贵的；然而在现实中的大部分情况下，同步方法是运行在单线程环境(无锁竞争环境)如果每次都调用Mutex Lock那么将严重的影响程序的性能。不**过在jdk1.6中对锁的实现引入了大量的优化**，如**锁粗化(Lock Coarsening)**、**锁消除(Lock Elimination)**、**轻量级锁(Lightweight Locking)**、**偏向锁(Biased Locking)**、**适应性自旋(Adaptive Spinning)**等技术来减少锁操作的开销。
- `锁粗化(Lock Coarsening)`：也就是减少不必要的紧连在一起的unlock，lock操作，将多个连续的锁扩展成一个范围更大的锁。
- `锁消除(Lock Elimination)`：通过运行时JIT编译器的逃逸分析来消除一些没有在当前同步块以外被其他线程共享的数据的锁保护，通过逃逸分析也可以在线程本地Stack上进行对象空间的分配(同时还可以减少Heap上的垃圾收集开销)。
- `轻量级锁(Lightweight Locking)`：这种锁实现的背后基于这样一种假设，即在真实的情况下我们程序中的大部分同步代码一般都处于无锁竞争状态(即单线程执行环境)，在无锁竞争的情况下完全可以避免调用操作系统层面的重量级互斥锁，取而代之的是在monitorenter和monitorexit中只需要依靠一条CAS原子指令就可以完成锁的获取及释放。当存在锁竞争的情况下，执行CAS指令失败的线程将调用操作系统互斥锁进入到阻塞状态，当锁被释放的时候被唤醒(具体处理步骤下面详细讨论)。
- ` 偏向锁(Biased Locking)`：是为了在无锁竞争的情况下避免在锁获取过程中执行不必要的CAS原子指令，因为CAS原子指令虽然相对于重量级锁来说开销比较小但还是存在非常可观的本地延迟。
- `适应性自旋(Adaptive Spinning)`：当线程在获取轻量级锁的过程中执行CAS操作失败时，在进入与monitor相关联的操作系统重量级锁(mutex semaphore)前会进入忙等待(Spinning)然后再次尝试，当尝试一定的次数后如果仍然没有成功则调用与该monitor关联的semaphore(即互斥锁)进入到阻塞状态。

## 锁类型
在Java SE 1.6里Synchronied同步锁，一共有四种状态：`无锁`、`偏向锁`、`轻量级锁`、`重量级锁`，它会随着竞争情况逐渐升级。锁可以升级但是不可以降级，目的是为了提供获取锁和释放锁的效率。
> 锁膨胀方向： 无锁 → 偏向锁 → 轻量级锁 → 重量级锁 (此过程是不可逆的)
### 轻量级锁
在JDK 1.6之后引入的轻量级锁，需要注意的是轻量级锁并不是替代重量级锁的，而是对在大多数情况下同步块并不会有竞争出现提出的一种优化。它可以减少重量级锁对线程的阻塞带来地线程开销。从而提高并发性能。
 如果要理解轻量级锁，那么必须先要了解HotSpot虚拟机中对象头地内存布局。上面介绍Java对象头也详细介绍过。在对象头中(Object Header)存在两部分。第一部分用于存储对象自身的运行时数据，HashCode、GC Age、锁标记位、是否为偏向锁等。一般为32位或者64位(视操作系统位数定)。官方称之为Mark Word，它是实现轻量级锁和偏向锁的关键。 另外一部分存储的是指向方法区对象类型数据的指针(Klass Point)，如果对象是数组的话，还会有一个额外的部分用于存储数据的长度。
其中最后两位表示锁状态，**00**表示**轻量级锁**，**10**表示**重量级锁**，**11**表示标记被**垃圾回收**。而当最后两位表示01时，要看倒数第三位是什么，如果是**001**表示**无偏向锁**，**101**表示**偏向锁**。
![在这里插入图片描述](http://badwomen.asia/cfbea7e88c394c3eb8e1b2fd9128ae69.png)在线程执行同步块之前，JVM会先在当前线程的栈帧中创建一个名为锁记录(Lock Record)的空间，用于存储锁对象目前的Mark Word的拷贝(JVM会将对象头中的Mark Word拷贝到锁记录中，官方称为Displaced Mark Ward)这个时候线程堆栈与对象头的状态如图：
![在这里插入图片描述](http://badwomen.asia/0540d616f97243d68f2c3375a38ade8c.png)
如果当前对象没有被锁定，那么锁标志位位01状态，JVM在执行当前线程时，首先会在当前线程栈帧中创建锁记录`Lock Record`的空间用于存储锁对象目前的`Mark Word`的拷贝。  然后，虚拟机使用CAS操作将标记字段`Mark Word`拷贝到锁记录中，并且将Mark Word更新为指向`Lock Record`的指针。如果更新成功了，那么这个线程就有用了该对象的锁，并且对象`Mark Word`的锁标志位更新为(`Mark Word`
中最后的2bit)00，即表示此对象处于轻量级锁定状态，如图：
![在这里插入图片描述](http://badwomen.asia/e07901a8990f4139b844ac7567f6c98a.png)
![在这里插入图片描述](http://badwomen.asia/12536a045c6e4516b46ce61ad39786c5.png)
如果这个更新操作失败，JVM会检查当前的`Mark Word`中是否存在指向当前线程的栈帧的指针，如果有，说明该锁已经被获取，可以直接调用。如果没有，则说明该锁被其他线程抢占了，如果有两条以上的线程竞争同一个锁，那轻量级锁就不再有效，直接膨胀位重量级锁，没有获得锁的线程会被阻塞。此时，锁的标志位为10。`Mark Word`中存储的时指向重量级锁的指针。
![在这里插入图片描述](http://badwomen.asia/b32cba954bf440efaadc0b3152d2307c.png)
![在这里插入图片描述](http://badwomen.asia/f86c33be266740b8839874c072b4804a.png)

### 自旋优化
重量级锁竞争时，还可以使用自选来优化，如果当前线程在自旋成功（使用锁的线程退出了同步块，释放了锁），这时就可以避免线程进入阻塞状态。阻塞状态要切换上下文，这是很消耗性能的。

- 第一种情况（成功）
![在这里插入图片描述](http://badwomen.asia/f524524343474664933ebc6279c2c87d.png)


- 第二种情况（失败）
![在这里插入图片描述](http://badwomen.asia/387ffb0e26674b03930a92233e2a31e6.png)
### 偏向锁
轻量级锁在没有竞争时，每次重入（该线程执行的方法中再次锁住该对象）操作仍需要cas替换操作，这样是会使性能降低的。
所以引入了偏向锁对性能进行优化：在第一次cas时会将线程的ID写入对象的Mark Word中。此后发现这个线程ID就是自己的，就表示没有竞争，就不需要再次cas，以后只要不发生竞争，这个对象就归该线程所有。
![在这里插入图片描述](http://badwomen.asia/654f9a2132b4479182196a57536d1940.png)

#### 撤销偏向
以下几种情况会使对象的偏向锁失效

- 调用对象的hashCode方法
- 多个线程使用该对象
- 调用了wait/notify方法（调用wait方法会导致锁膨胀而使用重量级锁）

### 批量重偏向
- 如果对象虽然被多个线程访问，但是线程间不存在竞争，这时偏向T1的对象仍有机会重新偏向T2。重偏向会重置Thread ID
- 当撤销超过20次后（超过阈值），JVM会觉得是不是偏向错了，这时会在给对象加锁时，重新偏向至加锁线程

### 批量撤销
当撤销偏向锁的阈值超过40以后，就会将整个类的对象都改为不可偏向的

### 锁消除
**是什么?**
锁消除就是消除那些根本不存在共享对象竞争却用到锁的代码块上的锁

**什么时候使用锁消除?**
通过逃逸分析判断来确认一段代码是否需要锁消除

**逃逸分析:** 一个对象在方法中产生, 如果被当作其他方法的参数, 这种叫方法逃逸, 如果被其他线程访问到, 则叫线程逃逸




## 锁的优缺点对比 
| 锁       | 优点                                                         | 缺点                                                         | 使用场景                           |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ---------------------------------- |
| 偏向锁   | 加锁和解锁不需要CAS操作，没有额外的性能消耗，和执行非同步方法相比仅存在纳秒级的差距 | 如果线程间存在锁竞争，会带来额外的锁撤销的消耗               | 适用于只有一个线程访问同步快的场景 |
| 轻量级锁 | 竞争的线程不会阻塞，提高了响应速度                           | 如线程成始终得不到锁竞争的线程，使用自旋会消耗CPU性能        | 追求响应时间，同步快执行速度非常快 |
| 重量级锁 | 线程竞争不适用自旋，不会消耗CPU                              | 线程阻塞，响应时间缓慢，在多线程下，频繁的获取释放锁，会带来巨大的性能消耗 | 追求吞吐量，同步快执行速度较长     |



# 再深入理解
synchronized是通过软件(JVM)实现的，简单易用，即使在JDK5之后有了Lock，仍然被广泛地使用。
 - **使用Synchronized有哪些要注意的？** 
 >1.锁对象不能为空，因为锁的信息都保存在对象头里
 >2.作用域不宜过大，影响程序执行的速度，控制范围过大，编写代码也容易出错
 >3.避免死锁
 >4.在能选择的情况下，既不要用Lock也不要用synchronized关键字，用java.util.concurrent包中的各种各样的类，如果不用该包下的类，在满足业务的情况下，可以使用synchronized关键，因为代码量少，避免出错 

 - **synchronized是公平锁吗？** 
>synchronized实际上是非公平的，新来的线程有可能立即获得监视器，而在等待区中等候已久的线程可能再次等待，不过这种抢占的方式可以预防饥饿。

# 参考文献
https://www.pdai.tech/
《深入理解Java虚拟机》
《Java并发编程的艺术》