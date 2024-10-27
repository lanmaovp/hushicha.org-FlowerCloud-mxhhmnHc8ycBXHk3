
## 一、对象结构和锁状态


synchronized关键字是java中的内置锁实现，内置锁实际上就是个任意对象，其内存结构如下图所示


![Java对象结构](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945323-1261090934.png)
其中，Mark Word字段在64位虚拟机下占64bit长度，其结构如下所示


![64位Mark Word的结构信息](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945230-1086259110.png)
可以看到Mark Word字段有个很重要的作用就是记录当前对象锁状态，最后3bit字段用来标记当前锁状态是无锁、偏向锁、轻量级锁还是重量级锁。


（1）lock：锁状态标记位，占两个二进制位，由于希望用尽可能少的二进制位表示尽可能多的信息，因此设置了lock标记。该标记的值不同，整个Mark Word表示的含义就不同。


（2）biased\_lock：对象是否启用偏向锁标记，只占1个二进制位。为1时表示对象启用偏向锁，为0时表示对象没有偏向锁。


lock和biased\_lock两个标记位组合在一起共同表示Object实例处于什么样的锁状态。二者组合的含义具体如下表所示


![image-20240919171225689](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945021-842564546.png)
在JDK 1\.6版本之前，所有的Java内置锁都是重量级锁。重量级锁会造成CPU在用户态和核心态之间频繁切换，所以代价高、效率低。JDK 1\.6版本为了减少获得锁和释放锁所带来的性能消耗，引入了偏向锁和轻量级锁的实现。所以，在JDK 1\.6版本中内置锁一共有4种状态：无锁状态、偏向锁状态、轻量级锁状态和重量级锁状态，这些状态随着竞争情况逐渐升级。内置锁可以升级但不能降级，意味着偏向锁升级成轻量级锁后不能再降级成偏向锁。这种能升级却不能降级的策略，其目的是提高获得锁和释放锁的效率。


随着出现竞争以及竞争升级，锁状态会依次从无锁状态变为偏向锁、轻量级锁、重量级锁，下面通过案例讲解这个锁膨胀的过程（基于Java8）。


## 二、无锁


无锁状态的对象内存结构如下所示


![image-20241018160620782](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171944987-1392780571.png)
一个对象没有被synchronized修饰的状态就是无锁状态，看以下代码



```
import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import org.openjdk.jol.info.ClassLayout;

/**
 * @author kdyzm
 * @date 2024/10/18
 */
@Slf4j
public class NoLockTest {

    public static void main(String[] args) {
        User user = new User();
        System.out.println(ClassLayout.parseInstance(user).toPrintable());
    }

    @Data
    public static class User {
        private String userName;
    }
}

```

这里使用JOL工具输出了普通对象user的内存结构（关于java对象内存结构，详情查看《[深入理解Java对象结构](https://github.com)》）


![image-20241018155332377](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945461-2019671717.png)
可以看到，输出的64位MarkWord的值是十六进制数 `0x0000000000000001 (non-biasable; age: 0)` 甚至还贴心的给出了当前是非偏向锁状态以及当前的对象年龄。由于这里使用了JOL的高版本0\.17，输出的内容是大端序的，所以根据最后一个字节01可以知道它的二进制数是`0000 0001`，也就是说，biased\_lock和lock 3 bit的数值是`001`，


![image-20240919171225689](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945021-842564546.png)
根据锁状态标志，可以知道当前对象是无锁状态。需要注意的是，这里输出无锁状态是因为没开启偏向锁机制，如果开启了偏向锁标志，创建的对象默认自带偏向锁标志。


## 三、偏向锁


**偏向锁是指一段同步代码一直被同一个线程所访问，那么该线程会自动获取锁，降低获取锁的代价。**如果内置锁处于偏向状态，当有一个线程来竞争锁时，先用偏向锁，表示内置锁偏爱这个线程，这个线程要执行该锁关联的同步代码时，不需要再做任何检查和切换。偏向锁在竞争不激烈的情况下效率非常高。


需要注意的是，偏向锁状态并非第一次发生了锁占用才设置的，JVM默认启动4秒钟之后才会延迟启动偏向锁机制，这时候创建的对象会默认加上偏向锁标记。可以通过添加JVM启动参数：`-XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=0`，让系统默认开启偏向锁机制。


偏向锁状态的Mark Word会记录内置锁自己偏爱的线程ID，内置锁会将该线程当作自己的熟人。偏向锁状态下对象的Mark Word如下图所示


![image-20241018161149485](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945264-1284912153.png)
### 1、偏向锁核心原理


偏向锁的核心原理是：如果不存在线程竞争的一个线程获得了锁，那么锁就进入偏向状态，此时Mark Word的结构变为偏向锁结构，锁对象的锁标志位（lock）被改为01，偏向标志位（biased\_lock）被改为1，然后线程的ID记录在锁对象的Mark Word中（使用CAS操作完成）。以后该线程获取锁时判断一下线程ID和标志位，就可以直接进入同步块，连CAS操作都不需要，这样就省去了大量有关锁申请的操作，从而也就提升了程序的性能。


偏向锁的主要作用是消除无竞争情况下的同步原语，进一步提升程序性能，所以，在没有锁竞争的场合，偏向锁有很好的优化效果。但是，一旦有第二条线程需要竞争锁，那么偏向模式立即结束，进入轻量级锁的状态。


假如在大部分情况下同步块是没有竞争的，那么可以通过偏向来提高性能。即在无竞争时，之前获得锁的线程再次获得锁时会判断偏向锁的线程ID是否指向自己，如果是，那么该线程将不用再次获得锁，直接就可以进入同步块；如果未指向当前线程，当前线程就会采用CAS操作将Mark Word中的线程ID设置为当前线程ID，如果CAS操作成功，那么获取偏向锁成功，执行同步代码块，如果CAS操作失败，那么表示有竞争，抢锁线程被挂起，撤销占锁线程的偏向锁，然后将偏向锁膨胀为轻量级锁。


偏向锁的缺点：如果锁对象时常被多个线程竞争，偏向锁就是多余的，并且其撤销的过程会带来一些性能开销。


### 2、偏向锁案例


以下代码打印了单线程在占据锁前、已占有锁、释放锁后的内存结构（JOL工具输出），从而观察偏向锁状态（注意开启偏向锁功能：`-XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=0`）



```
import lombok.Getter;
import lombok.Setter;
import lombok.ToString;
import lombok.extern.slf4j.Slf4j;
import org.openjdk.jol.info.ClassLayout;


/**
 * @author kdyzm
 * @date 2024/10/18
 * 启动的时候注意加上JVM启动参数：-XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=0
 */
@Slf4j
public class BiasedLockTest {

    public static void main(String[] args) throws InterruptedException {
        User lock = new User();
        log.info("抢占锁前lock的状态：\n{}", lock.getObjectStruct());
        Thread thread = new Thread(() -> {
            for (int i = 0; i < 100; i++) {
                synchronized (lock) {
                    if (i == 99) {
                        log.info("占有锁lock的状态：\n{}", lock.getObjectStruct());
                    }
                }
            }
        }, "biased-lock-thread");
        thread.start();
        thread.join();
        log.info("释放锁后lock的状态：\n{}", lock.getObjectStruct());
    }

    @ToString
    @Setter
    @Getter
    public static class User {
        private String userName;

        //对象结构字符串
        public String getObjectStruct() {
            return ClassLayout.parseInstance(this).toPrintable();
        }
    }
}

```

输出结果


![image-20241019215404495](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945642-1223834552.png)
分析输出结果：


**抢占锁前**：打印锁内存结构，可以看到最后一个字节是`05`，对应着biased\+lock状态组合，倒数三个bit是`101`，也就是偏向锁状态。但是打印出来的结果有点不一样，是“biasable”状态，这时候锁对象还未锁定、未偏向，所以也就是“可偏向”的状态


**占有锁**：占有锁之后，可以看到最后一个字节还是`05`，但是输出已经提示是“biased”也就是偏向锁状态了，锁的MarkWord已经记录了占有锁的线程id，不过由于此线程ID不是Java中的Thread实例的ID，因此没有办法直接在Java程序中比对。


**释放锁后**：可以看到最后一个字节还是`05`，也就是还是偏向锁状态，这是因为锁释放需要一定的开销，而偏向锁是一种乐观锁，它认为还是有很大可能偏向锁的持有线程会继续获取锁，所以不会主动撤销偏向锁状态。


思考一个问题：同一个线程重复获取相同的锁，lock对象锁变成了偏向锁，那么如果当前线程结束后，新建一个线程并重新获取lock锁，lock锁中记录的线程id是否会被更新成新线程的线程id实现重偏向呢？



```
import lombok.Getter;
import lombok.Setter;
import lombok.ToString;
import lombok.extern.slf4j.Slf4j;
import org.openjdk.jol.info.ClassLayout;


/**
 * @author kdyzm
 * @date 2024/10/18
 */
@Slf4j
public class BiasedLockTest {

    public static void main(String[] args) throws InterruptedException {
        User lock = new User();
        Thread threadA = runThread(lock, "A");
        Thread threadB = runThread(lock, "B");
        threadA.start();
        threadA.join();
        threadB.start();
        threadB.join();
    }

    private static Thread runThread(User lock, String name) throws InterruptedException {
        Thread thread = new Thread(() -> {
            log.info("抢占锁前lock的状态：\n{}", lock.getObjectStruct());
            for (int i = 0; i < 100; i++) {
                synchronized (lock) {
                    if (i == 99) {
                        log.info("占有锁lock的状态：\n{}", lock.getObjectStruct());
                    }
                }
            }
            log.info("释放锁后lock的状态：\n{}", lock.getObjectStruct());
        }, name);
        return thread;
    }

    @ToString
    @Setter
    @Getter
    public static class User {
        private String userName;

        //JOL对象结构字符串
        public String getObjectStruct() {
            return ClassLayout.parseInstance(this).toPrintable();
        }
    }
}

```

线程A执行结果：


![image-20241021221902252](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945715-1983246300.png)
线程B执行结果：


![image-20241021224745019](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945802-1450605603.png)
可以看到线程B等到线程A结束之后获取到了lock锁，但是锁的状态并没有预想中的仍然保持偏向锁状态线程id指向线程B，而是直接锁升级成了轻量级锁。


从运行结果上来看，偏向锁更像是“**一锤子买卖**”，只要偏向了某个线程，后续其他线程尝试获取锁，都会变为轻量级锁，这样的偏向非常局限。**事实上并不是这样**，想要解释这个问题，需要先了解“**批量重偏向**”的知识。


### 3、批量重偏向


批量重偏向技术是JVM针对锁对象的Class的一种优化技术，如果某Class有很多个对象，每个对象都被当做对象锁来用，当首次被线程获取到之后，都会变成偏向锁状态；当都被释放之后，它们默认保持偏向锁状态：一方面防止短时间内对应的线程再次获取锁，另一方面撤销偏向状态一个两个还行，如果太多的话系统开销也比较大；已经被释放并且保持偏向锁状态的这些对象锁这时候如果被另外的线程访问，JVM就遇到一个问题：我是否应当立即修改对象锁的MarkWord中的线程id，改成新线程的线程id？


你一个新线程过来获取锁，上来就重偏向你，是不是不大合理？再怎么着也得考察考察你到底有没有资格受JVM“偏爱”吧，考察方式就是新线程尝试获取对象锁多少次，假设设置了阈值20，那就是我有20个对象锁都已经是偏向锁状态了，偏向了线程A，如果这时候新线程B又要依次获取这些对象锁，那前19次都不会重偏向到B线程，直接升级轻量级锁，到第20次的时候线程B又来了，那说明线程B以后大概率还是会尝试获取锁，这时候再把偏向锁重偏向线程B。


在在linux命令行或者Windows中的git bash中运行命令：`java -XX:+PrintFlagsFinal | grep BiasedLock` 查看偏向锁相关的配置参数



```
[root@lenovo ~]# java -XX:+PrintFlagsFinal | grep BiasedLock
     intx BiasedLockingBulkRebiasThreshold          = 20                                  {product}
     intx BiasedLockingBulkRevokeThreshold          = 40                                  {product}
     intx BiasedLockingDecayTime                    = 25000                               {product}
     intx BiasedLockingStartupDelay                 = 4000                                {product}
     bool TraceBiasedLocking                        = false                               {product}
     bool UseBiasedLocking                          = true                                {product}

```

其中，我们已经使用过了UseBiasedLocking以及BiasedLockingStartupDelay参数，之前使用它控制系统启动时就启用偏向锁并且设置偏向锁启动延迟时间为0：`-XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=0`，现在则关心下`BiasedLockingBulkRebiasThreshold`  参数




| 参数 | 释义 |
| --- | --- |
| BiasedLockingBulkRebiasThreshold\=20 | 批量偏向阈值，默认值20 |


这个批量偏向阈值为20的意思就是：一个类有20个对象，线程A对每个对象synchronized获取到了锁并将每个对象修改成了偏向锁，线程B重新获取这20个对象锁，尝试重偏向，前19次都失败了，这19个对象锁从偏向锁升级成了轻量级锁，第20个则到达批量偏向阈值，会发生重偏向，偏向锁从偏向A线程变成了偏向B线程。


下面通过代码验证



```
import lombok.extern.slf4j.Slf4j;
import org.openjdk.jol.info.ClassLayout;

import java.util.ArrayList;
import java.util.List;

/**
 * 偏向锁-批量重偏向测试
 * 运行前注意加 -XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=0 JVM启动参数
 **/
@Slf4j
public class BulkRebiasTest {

    static class User {

    }

    public static void main(String[] args) throws InterruptedException {
        final List list = new ArrayList<>();
        Thread A = new Thread() {
            @Override
            public void run() {
                for (int i = 0; i < 20; i++) {
                    User a = new User();
                    list.add(a);
                    //获取锁之后都变成了偏向锁，偏向线程A
                    synchronized (a) {

                    }
                }
            }
        };

        Thread B = new Thread() {
            @Override
            public void run() {
                for (int i = 0; i < 20; i++) {
                    User a = list.get(i);
                    //从list当中拿出都是偏向线程A
                    log.info("B 加锁前第 {} 次" + ClassLayout.parseInstance(a).toPrintable(), i + 1);
                    synchronized (a) {
                        //前19次撤销偏向锁偏向线程A，然后升级轻量级锁指向线程B线程栈当中的锁记录
                        //第20次开始重偏向线程B
                        log.info("B 加锁中第 {} 次" + ClassLayout.parseInstance(a).toPrintable(), i + 1);
                    }
                    //因为前19次是轻量级锁，释放之后为无锁不可偏向
                    //但是第20次是偏向锁 偏向线程B 释放之后依然是偏向线程B
                    log.info("B 加锁结束第 {} 次" + ClassLayout.parseInstance(a).toPrintable(), i + 1);
                }
                //第20次发生了重偏向以后，User类的epoch字段变成了1，新生成的对象中的epoch字段也是1
                log.info("新产生的对象：" + ClassLayout.parseInstance(new User()).toPrintable());
            }

        };
        A.start();
        Thread.sleep(1000);
        B.start();
        Thread.sleep(1000);
    }
}

```

运行结果如下


B线程前19循环次运行结果：


![image-20241023104105626](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945686-276684037.png)
B线程第20次运行结果：


![image-20241023104355105](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171946382-179912675.png)
最后，重新生成一个User类的实例，直接打印看结果


![image-20241023110816192](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945644-490575649.png)
这里出现了一个重要的字段：epoch


### 4、epoch


epoch字段是Mark Word中的一个占2bit的字段：


![image-20241018161149485](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945264-1284912153.png)
epoch在偏向锁中发挥了重要的作用，它决定了当前偏向锁走重偏向还是锁升级的逻辑。


每个**class类**维护了一个**偏向锁撤销计数器**，只要 **class 的对象**发生**偏向撤销**，该计数器 `+1`，当这个值达到重偏向阈值（默认 20）时：`BiasedLockingBulkRebiasThreshold=20`，JVM 就认为该 class 的偏向锁有问题，因此会进行批量重偏向, 它的实现方式就用到了`epoch`字段。


每个 class 对象会有一个对应的`epoch`字段，每个**处于偏向锁状态对象**的`mark word` 中也有该字段，其初始值为创建该对象时 class 中的`epoch`的值（此时二者是相等的）。


每次发生批量重偏向时，就将该值加 1，同时遍历 JVM 中所有线程的栈：


1. 找到该 class 所有**正处于加锁状态**的偏向锁对象，将其`epoch`字段改为新值
2. class 中**不处于加锁状态**的偏向锁对象（没被任何线程持有，但之前是被线程持有过的，这种锁对象的 markword 肯定也是有偏向的），保持 `epoch` 字段值不变


这样，当偏向锁被其他线程尝试获取锁时，就会检查**偏向锁对象类中的epoch**是否和**偏向锁Mark Word中的epoch**相同：


1. 如果不相同，表示偏向锁对象类中的epoch增加时，偏向锁的持有线程已经结束，所以偏向锁Mark Word中的epoch并没有增加，也就是说，**该对象的偏向锁已经无效了**，这时候可以走重偏向逻辑（起名epoch"纪元"也就是这个意思，来到新纪元，老纪元的事情就不管了）。
2. 如果相同，表示还是当前纪元内，当前的偏向锁没有发生过重偏向，又有新线程来竞争锁，那锁就要升级。


回到上面批量重偏向的案例，B线程循环20次对User类的偏向锁实例每次都获取到了锁。前19次循环，由于User类的epoch还是0，每个偏向锁对象Mark Word中的epoch也都是0，所以走了锁升级流程，偏向锁升级成了轻量级锁，由于升级轻量级锁的过程中发生了锁撤销，所以User类的偏向锁撤销计数器同时\+1；第20次的时候User类的epoch加1变成了20触发了批量重偏向，本应当走轻量级锁升级的走了批量重偏向。发生批量重偏向之后，User类的epoch字段加1变成了1，所以新建User类的实例默认epoch和User类保持一致，都是1。


### 5、批量撤销


这里讨论下关于偏向锁的另外一个参数：BiasedLockingBulkRevokeThreshold，




| 参数 | 释义 |
| --- | --- |
| BiasedLockingBulkRevokeThreshold\=40 | 批量撤销阈值，默认值40 |


从输出上来看，它的默认值是40



```
[root@lenovo ~]# java -XX:+PrintFlagsFinal | grep BiasedLock
     intx BiasedLockingBulkRebiasThreshold          = 20                                  {product}
     intx BiasedLockingBulkRevokeThreshold          = 40                                  {product}
     intx BiasedLockingDecayTime                    = 25000                               {product}
     intx BiasedLockingStartupDelay                 = 4000                                {product}
     bool TraceBiasedLocking                        = false                               {product}
     bool UseBiasedLocking                          = true                                {product}

```

BiasedLockingBulkRevokeThreshold用来设置偏向锁批量撤销的阈值，当偏向锁撤销次数到达这个值时会发生两件事情


1. class 的 markword 将被修改为不可偏向无锁状态，这样该class再生成对象将会变成无锁状态，当线程竞争锁时不再走偏向锁，直接进入轻量级锁状态。
2. 遍历所有当前存活的线程的栈，找到该 class 所有正处于偏向锁状态的锁实例对象，执行偏向锁的撤销操作。


下面看验证的例子



```
import lombok.extern.slf4j.Slf4j;
import org.openjdk.jol.info.ClassLayout;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.locks.LockSupport;

/**
 * @author kdyzm
 * @date 2024/10/23
 */
@Slf4j
public class BulkRevoteTest {

    static class User {

    }

    private static Thread A;
    private static Thread B;
    private static Thread C;

    public static void main(String[] args) throws InterruptedException {
        final List list = new ArrayList<>();
        //A线程创建40把偏向锁，并全部偏向A
        A = new Thread() {
            @Override
            public void run() {
                for (int i = 0; i < 40; i++) {
                    User a = new User();
                    list.add(a);
                    //获取锁之后都变成了偏向锁，偏向线程A
                    synchronized (a) {

                    }
                }
                //唤醒线程B
                LockSupport.unpark(B);
            }
        };

        //B线程撤销前20把偏向锁，达到批量重偏向阈值
        //对后20把偏向锁重偏向到线程B
        B = new Thread() {
            @Override
            public void run() {
                //等待线程A唤醒
                LockSupport.park();
                for (int i = 0; i < 40; i++) {
                    User a = list.get(i);
                    //从list当中拿出都是偏向线程A
                    log.info("B 加锁前第 {} 次" + ClassLayout.parseInstance(a).toPrintable(), i + 1);
                    synchronized (a) {
                        //前19次撤销偏向锁偏向线程A，然后升级轻量级锁指向线程B线程栈当中的锁记录
                        //第20次开始重偏向线程B
                        log.info("B 加锁中第 {} 次" + ClassLayout.parseInstance(a).toPrintable(), i + 1);
                    }
                    //因为前19次是轻量级锁，释放之后为无锁不可偏向
                    //但是第20次是偏向锁 偏向线程B 释放之后依然是偏向线程B
                    log.info("B 加锁结束第 {} 次" + ClassLayout.parseInstance(a).toPrintable(), i + 1);
                }
                //第20次发生了重偏向以后，User类的epoch字段变成了1，新生成的对象中的epoch字段也是1
                log.info("B 新产生的对象：" + ClassLayout.parseInstance(new User()).toPrintable());
                //唤醒线程C
                LockSupport.unpark(C);
            }

        };
		
        //C线程对后20把偏向到线程B的偏向锁进行偏向锁撤销
        //加上B线程撤销的20次共40次，达到偏向锁批量撤销的阈值
        C = new Thread() {
            @Override
            public void run() {
                //等待线程B唤醒
                LockSupport.park();
                //list数组20坐标以前是轻量级锁
                //20以及20以后是偏向线程B的偏向锁，所以从20坐标开始
                for (int i = 20; i < 40; i++) {
                    User a = list.get(i);
                    //这里拿出的都是偏向线程B的偏向锁
                    log.info("C 加锁前第 {} 次" + ClassLayout.parseInstance(a).toPrintable(), i - 20 + 1);
                    synchronized (a) {
                        //由于epoch相同都是1，全升级成了轻量级锁
                        log.info("C 加锁中第 {} 次" + ClassLayout.parseInstance(a).toPrintable(), i - 20 + 1);
                    }
                    //轻量级锁释放之后全变成了不可偏向的无锁状态
                    log.info("C 加锁结束第 {} 次" + ClassLayout.parseInstance(a).toPrintable(), i - 20 + 1);
                    //观察此时是否发生了批量撤销，若发生了批量撤销，新对象将会是无锁状态
                    log.info("C 第{}次生成新对象：{}", i - 20 + 1, ClassLayout.parseInstance(new User()).toPrintable());
                }
                //C线程再发生偏向锁撤销20次，达到批量撤销的阈值
                //此时创建的对象应当都是无锁状态
                log.info("C 新产生的对象：" + ClassLayout.parseInstance(new User()).toPrintable());
            }

        };
        A.start();
        B.start();
        C.start();
        C.join();
    }

}

```

运行结果如下


先看线程B二十次以前的运行结果，从第1个元素到第19个偏向锁都被撤销，锁升级到了轻量级锁，锁释放后变成了无锁状态


![image-20241023154629522](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945846-1368487348.png)
线程B获取第20个偏向锁时到达批量重偏向阈值，它和前19次不一样，发生了批量重偏向，线程保持偏向锁状态，并且偏向线程B；之后的20到40也都这样


![image-20241023160657936](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945824-490518411.png)
接着看线程C，线程C从第21个元素开始取元素，取出来的元素肯定都是偏向线程B的偏向锁，线程C尝试获取这些锁会导致锁升级成轻量级锁，第21到39的输出都和第39输出类似，以39为例输出如下


![image-20241023161648421](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945789-1992729105.png)
接下来是关键点，因为线程C第20次获取list中的第40个偏向锁，合计B线程撤销的20次，会达到40次，也就是偏向锁撤销的阈值，如果不出意外，这次过后会发生偏向锁批量撤销，再创建对象，会是无锁状态


![image-20241023162246144](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945915-1292368310.png)
果然，发生了批量撤销，证明了撤销40次会触发批量撤销，再创建的对象将变成无锁状态，这时候，该类的偏向锁实际上就彻底被禁用了。


### 6、批量撤销冷静期


这里讨论下关于偏向锁的另外一个参数：BiasedLockingDecayTime，




| 参数 | 释义 |
| --- | --- |
| BiasedLockingDecayTime\=25000 | 批量撤销时间阈值 |


从输出上来看，它的默认值是25s



```
[root@lenovo ~]# java -XX:+PrintFlagsFinal | grep BiasedLock
     intx BiasedLockingBulkRebiasThreshold          = 20                                  {product}
     intx BiasedLockingBulkRevokeThreshold          = 40                                  {product}
     intx BiasedLockingDecayTime                    = 25000                               {product}
     intx BiasedLockingStartupDelay                 = 4000                                {product}
     bool TraceBiasedLocking                        = false                               {product}
     bool UseBiasedLocking                          = true                                {product}

```

这个参数的意思是


1. 如果在距离上次批量重偏向发生的 25 秒之内，并且累计撤销计数达到 40，就会发生批量撤销（该类的偏向锁功能彻底被禁用）
2. 如果在距离上次批量重偏向发生超过 25 秒之外，就会重置在 `[20, 40)` 内的计数, 再给次机会


**这玩意我给他起了个名字叫“撤销冷静期”，在冷静期内别再去尝试获取锁发生偏向锁撤销，那过了冷静期之后就再给你1到20次的撤销机会，相当于把批量撤销的阈值提高了。**


可以在上一章节的批量撤销演示代码中加入延迟操作验证这个问题。为了减少等待时间，可以添加JVM启动参数：`-XX:BiasedLockingDecayTime=5000` 将批量撤销时间阈值改为5秒。在C线程循环第20次的时候停止线程6秒钟



```
C = new Thread() {
            @Override
            public void run() {
                LockSupport.park();
                //list数组20坐标以前是轻量级锁
                //20以及20以后是偏向线程B的偏向锁，所以从20坐标开始
                for (int i = 20; i < 40; i++) {
                    //TODO 尝试停止线程6秒钟、5秒钟、4.9秒钟观察最终的输出结果
                    if (i == 39) {
                        try {
                            Thread.sleep(4900);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    User a = list.get(i);
                    //这里拿出的都是偏向线程B的偏向锁
                    log.info("C 加锁前第 {} 次" + ClassLayout.parseInstance(a).toPrintable(), i - 20 + 1);
                    synchronized (a) {
                        //由于epoch相同都是1，全升级成了轻量级锁
                        log.info("C 加锁中第 {} 次" + ClassLayout.parseInstance(a).toPrintable(), i - 20 + 1);
                    }
                    //轻量级锁释放之后全变成了不可偏向的无锁状态
                    log.info("C 加锁结束第 {} 次" + ClassLayout.parseInstance(a).toPrintable(), i - 20 + 1);
                    //观察此时是否发生了批量撤销，若发生了批量撤销，新对象将会是无锁状态
                    log.info("C 第{}次生成新对象：{}", i - 20 + 1, ClassLayout.parseInstance(new User()).toPrintable());
                }
                //C线程再发生偏向锁撤销20次，达到批量撤销的阈值
                //此时创建的对象应当都是无锁状态
                log.info("C 新产生的对象：" + ClassLayout.parseInstance(new User()).toPrintable());
            }

        };

```

会发现暂停6秒钟、5秒钟之后，C线程最终创建的新对象都是可偏向的状态，而暂停4\.9秒钟，则C线程最终创建的线程变成了不可偏向的无锁状态。


### 7、流程图


偏向锁是个非常复杂的锁类型，由于其复杂性，在JDK15及其以后的版本已经被废弃（注意是废弃，而非移除）。当然，你发任你发，我用java8，嘿嘿。总之掌握了总比没掌握的好，根据理解，我画了张流程图展示偏向锁锁状态的转化过程，这个图应该有错误的，如有错误请留言，我来改正\~


![无锁、偏向锁、轻量级锁、重量级锁-偏向锁转化流程图.drawio](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945853-1555715653.png)
## 四、轻量级锁


引入轻量级锁的主要目的是在多线程竞争不激烈的情况下，通过CAS机制竞争锁减少重量级锁产生的性能损耗。重量级锁使用了操作系统底层的互斥锁（Mutex Lock），会导致线程在用户态和核心态之间频繁切换，从而带来较大的性能损耗。


### 1、轻量级锁核心原理


轻量锁存在的目的是尽可能不动用操作系统层面的互斥锁，因为其性能比较差。线程的阻塞和唤醒需要CPU从用户态转为核心态，频繁地阻塞和唤醒对CPU来说是一件负担很重的工作。同时我们可以发现，很多对象锁的锁定状态只会持续很短的一段时间，例如整数的自加操作，在很短的时间内阻塞并唤醒线程显然不值得，为此引入了轻量级锁。轻量级锁是一种自旋锁，因为JVM本身就是一个应用，所以希望在应用层面上通过自旋解决线程同步问题。


轻量级锁的执行过程：在抢锁线程进入临界区之前，如果内置锁（临界区的同步对象）没有被锁定，JVM首先将在抢锁线程的栈帧中建立一个**锁记录（Lock Record）**，用于存储对象目前Mark Word的拷贝，这时的线程堆栈与内置锁对象头大致如下图所示


![image-20241024141506721](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945451-659832022.png)
然后抢锁线程将使用CAS自旋操作，尝试将内置锁对象头的Mark Word的ptr\_to\_lock\_record（锁记录指针）更新为抢锁线程栈帧中锁记录的地址，如果这个更新执行成功了，这个线程就拥有了这个对象锁。然后JVM将Mark Word中的lock标记位改为00（轻量级锁标志），即表示该对象处于轻量级锁状态。抢锁成功之后，JVM会将Mark Word中原来的锁对象信息（如哈希码等）保存在抢锁线程锁记录的Displaced Mark Word（可以理解为放错地方的Mark Word）字段中，再将抢锁线程中锁记录的owner指针指向锁对象。


在轻量级锁抢占成功之后，锁记录和对象头的状态如图所示


![轻量级锁结果图示](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945499-1850015244.png)
### 2、轻量级锁案例


关于轻量级锁的案例在上一章节已经有提过，现在再拿出来瞧瞧



```
import lombok.Getter;
import lombok.Setter;
import lombok.ToString;
import lombok.extern.slf4j.Slf4j;
import org.openjdk.jol.info.ClassLayout;


/**
 * @author kdyzm
 * @date 2024/10/18
 * 启动的时候注意加上JVM启动参数：-XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=0
 */
@Slf4j
public class BiasedLockTest {

    public static void main(String[] args) throws InterruptedException {
        User lock = new User();
        Thread threadA = runThread(lock, "A");
        Thread threadB = runThread(lock, "B");
        threadA.start();
        threadA.join();
        threadB.start();
        threadB.join();
    }

    private static Thread runThread(User lock, String name) throws InterruptedException {
        Thread thread = new Thread(() -> {
            log.info("抢占锁前lock的状态：\n{}", lock.getObjectStruct());
            for (int i = 0; i < 100; i++) {
                synchronized (lock) {
                    if (i == 99) {
                        log.info("占有锁lock的状态：\n{}", lock.getObjectStruct());
                    }
                }
            }
            log.info("释放锁后lock的状态：\n{}", lock.getObjectStruct());
        }, name);
        return thread;
    }

    @ToString
    @Setter
    @Getter
    public static class User {
        private String userName;

        //JOL对象结构字符串
        public String getObjectStruct() {
            return ClassLayout.parseInstance(this).toPrintable();
        }
    }
}

```

线程A执行结果：


![image-20241021221902252](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945715-1983246300.png)
线程B执行结果：


![image-20241021224745019](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945802-1450605603.png)
可以看到线程B等到线程A结束之后获取到了lock锁，再次获取lock锁，锁就变成了轻量级锁，释放后变成了无锁状态。从上一个章节中已经解释过了，之所以没有发生重偏向，是因为批量重偏向需要达到批量重偏向的撤销次数的阈值，默认是20次，在此之前，任何新线程尝试获取锁的行为都会导致偏向锁升级成轻量级锁。


### 3、轻量级锁分类


轻量级锁本质上就是自旋锁，所谓的“自旋”本质上就是循环重试，它有两种类型：**普通自旋锁**和**自适应自旋锁**。


#### 普通自旋锁


所谓普通自旋锁，就是指当有线程来竞争锁时，抢锁线程会在原地循环等待，而不是被阻塞，直到那个占有锁的线程释放锁之后，这个抢锁线程才可以获得锁。


锁在原地循环等待的时候是会消耗CPU的，就相当于在执行一个什么也不干的空循环。所以轻量级锁适用于临界区代码耗时很短的场景，这样线程在原地等待很短的时间就能够获得锁了。


**在JDK 1\.6中，Java虚拟机提供\-XX:\+UseSpinning参数来开启自旋锁，默认情况下，自旋的次数为10次，使用\-XX:PreBlockSpin参数来设置自旋锁的等待次数。**


**在JDK 1\.7后的版本，自旋锁的参数被取消，虚拟机不再支持由用户配置自旋锁。自旋锁总是被执行，自旋次数也由虚拟机自行调整。**


#### 自适应自旋锁


所谓自适应自旋锁，就是等待线程空循环的自旋次数并非是固定的，而是会动态地根据实际情况来改变自旋等待的次数，自旋次数由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。自适应自旋锁的大概原理是：


（1）如果抢锁线程在同一个锁对象上之前成功获得过锁，JVM就会认为这次自旋很有可能再次成功，因此允许自旋等待持续相对更长的时间。


（2）如果对于某个锁，抢锁线程很少成功获得过，那么JVM将可能减少自旋时间甚至省略自旋过程，以避免浪费处理器资源。


自适应自旋解决的是“锁竞争时间不确定”的问题。自适应自旋假定不同线程持有同一个锁对象的时间基本相当，竞争程度趋于稳定。总的思想是：根据上一次自旋的时间与结果调整下一次自旋的时间。


### 4、流程图


以下是无锁状态的对象变成轻量级锁的流程图


![image-20241024164546231](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945599-579483225.png)
## 五、重量级锁


重量级锁和偏向锁、轻量级锁不同，偏向锁、轻量级锁本质上都是乐观锁，它们都是应用级别的锁（JVM本身就是一个应用），重量级锁则基于操作系统内核的互斥锁实现，会发生用户态和内核态的切换，开销要更大，所以才叫“重量级锁”。


### 1、重量级锁核心原理


之前曾经提过，关于synchronized关键字底层使用了监视器锁，JVM中每个对象都会有一个监视器，监视器和对象一起创建、销毁。监视器相当于一个用来监视这些线程进入的特殊房间，其义务是保证（同一时间）只有一个线程可以访问被保护的临界区代码块。


本质上，监视器是一种同步工具，也可以说是一种同步机制，主要特点是：


（1）同步。监视器所保护的临界区代码是互斥地执行的。一个监视器是一个运行许可，任一线程进入临界区代码都需要获得这个许可，离开时把许可归还。


（2）协作。监视器提供Signal机制，允许正持有许可的线程暂时放弃许可进入阻塞等待状态，等待其他线程发送Signal去唤醒；其他拥有许可的线程可以发送Signal，唤醒正在阻塞等待的线程，让它可以重新获得许可并启动执行。


在Hotspot虚拟机中，监视器是由C\+\+类ObjectMonitor实现的，ObjectMonitor类定义在`share/vm/runtime/objectMonitor.hpp`文件中，其构造器代码大致如下：



```
//Monitor结构体
 ObjectMonitor::ObjectMonitor() {  
   _header      = NULL;  
   _count       = 0;  
   _waiters     = 0,  

   //线程的重入次数
   _recursions  = 0;      
   _object       = NULL;  

//标识拥有该Monitor的线程
   _owner        = NULL;   

  //等待线程组成的双向循环链表
   _WaitSet             = NULL;  
   _WaitSetLock  = 0 ;  
   _Responsible  = NULL ;  
   _succ                = NULL ;  

  //多线程竞争锁进入时的单向链表
   cxq                  = NULL ; 
   FreeNext             = NULL ;  

  //_owner从该双向循环链表中唤醒线程节点
   _EntryList           = NULL ; 
   _SpinFreq            = 0 ;  
   _SpinClock           = 0 ;  
   OwnerIsThread = 0 ;  
 }

```

ObjectMonitor的Owner（\_owner）、WaitSet（\_WaitSet）、Cxq（\_cxq）、EntryList（\_EntryList）这几个属性比较关键。ObjectMonitor的WaitSet、Cxq、EntryList这三个队列存放抢夺重量级锁的线程，而ObjectMonitor的Owner所指向的线程即为获得锁的线程。


Cxq、EntryList、WaitSet这三个队列的说明如下：


（1）Cxq：竞争队列（Contention Queue），所有请求锁的线程首先被放在这个竞争队列中。


（2）EntryList：Cxq中那些有资格成为候选资源的线程被移动到EntryList中。


（3）WaitSet：某个拥有ObjectMonitor的线程在调用Object.wait()方法之后将被阻塞，然后该线程将被放置在WaitSet链表中。


内部抢锁过程如下


![image-20241025144330190](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945741-174003820.png)
设计C\+\+底层的视线，代码比较复杂，有兴趣的可以参考文章：


[Java多线程：objectMonito源码解读](https://github.com)


### 2、重量级锁案例


下面通过自增案例展示从无锁、偏向锁、轻量级锁，最终膨胀为重量级锁的案例。



```
import lombok.extern.slf4j.Slf4j;
import org.openjdk.jol.info.ClassLayout;

import java.util.concurrent.locks.LockSupport;

/**
 * @author kdyzm
 * @date 2024/10/25
 * 运行前注意加上JVM启动参数：-XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=0 -XX:BiasedLockingDecayTime=5000
 */
@Slf4j
public class FatLock {

    private static final FatLock LOCK = new FatLock();

    private static Long counter = 0L;

    public static void main(String[] args) throws InterruptedException {
        //A线程执行100次自增
        Thread A = new Thread(() -> {
            log.info("加锁前：{}", ClassLayout.parseInstance(LOCK).toPrintable());
            for (int i = 0; i < 100; i++) {
                synchronized (LOCK) {
                    counter++;
                    if (i == 99) {
                        log.info("加锁中：{}", ClassLayout.parseInstance(LOCK).toPrintable());
                    }
                }
            }
            log.info("加锁后：{}", ClassLayout.parseInstance(LOCK).toPrintable());
        }, "A");
        //B线程执行100次自增并在第100次时进入等待状态
        Thread B = new Thread(() -> {
            log.info("加锁前：{}", ClassLayout.parseInstance(LOCK).toPrintable());
            for (int i = 0; i < 100; i++) {
                synchronized (LOCK) {
                    counter++;
                    if (i == 99) {
                        log.info("加锁中：{}", ClassLayout.parseInstance(LOCK).toPrintable());
                        //进入等待状态创造重量级锁形成条件
                        LockSupport.park();
                    }
                }
            }
            log.info("加锁后：{}", ClassLayout.parseInstance(LOCK).toPrintable());
        }, "B");
        //C线程执行100次自增
        Thread C = new Thread(() -> {
            log.info("加锁前：{}", ClassLayout.parseInstance(LOCK).toPrintable());
            for (int i = 0; i < 100; i++) {
                synchronized (LOCK) {
                    counter++;
                    if (i == 99) {
                        log.info("加锁中：{}", ClassLayout.parseInstance(LOCK).toPrintable());
                    }
                }
            }
            log.info("加锁后：{}", ClassLayout.parseInstance(LOCK).toPrintable());
        }, "C");

        A.start();
        //等待A线程执行完成
        Thread.sleep(1000);
        B.start();
        //等待B线程执行完成并进入等待状态
        Thread.sleep(1000);
        C.start();
        //等待C线程进入阻塞状态
        Thread.sleep(1000);
        //恢复B线程的运行状态
        LockSupport.unpark(B);
    }
}

```

运行结果如下




| 线程名 | 加锁前 | 加锁中 | 加锁后 |
| --- | --- | --- | --- |
| A | biasable（可偏向的，未设置偏向线程id） | biased（已偏向A线程） | biased（已偏向A线程） |
| B | biased（已偏向A线程） | thin lock（轻量级锁） | fat lock（重量级锁） |
| C | thin lock（轻量级锁） | fat lock（重量级锁） | non\-biasable（无锁） |


流程图如下所示


![无锁、偏向锁、轻量级锁、重量级锁-重量级锁升级.drawio](https://img2024.cnblogs.com/blog/516671/202410/516671-20241025171945906-242875623.png)
## 六、参考文档


《JAVA高并发核心编程 卷2：多线程、锁、JMM、JUC、高并发设计模式》


[偏向锁\-批量重偏向和批量撤销测试](https://github.com)


[深入浅出偏向锁](https://github.com)


[深入JVM锁](https://github.com)


[偏向锁和重量级锁的多连问，你能接住几个？](https://github.com)


[Java多线程：objectMonitor源码解读（3）](https://github.com)


[偏向锁、轻量级锁、重量级锁，Synchronized底层源码终极解析！](https://github.com)



最后，欢迎关注我的博客：[https://blog.kdyzm.cn](https://github.com) \~


  * [一、对象结构和锁状态](#tid-ahSyWa)
* [二、无锁](#tid-fQC7ai)
* [三、偏向锁](#tid-m2XR4m)
* [1、偏向锁核心原理](#tid-SP5JsE)
* [2、偏向锁案例](#tid-fj2pSs)
* [3、批量重偏向](#tid-htCmRC)
* [4、epoch](#tid-AArJWD)
* [5、批量撤销](#tid-iKXMBF)
* [6、批量撤销冷静期](#tid-RJFNWY):[FlowerCloud机场](https://hanlianfangzhi.com)
* [7、流程图](#tid-pRtQkz)
* [四、轻量级锁](#tid-tjZYki)
* [1、轻量级锁核心原理](#tid-Bp6QGC)
* [2、轻量级锁案例](#tid-bN4CCm)
* [3、轻量级锁分类](#tid-CBEciQ)
* [普通自旋锁](#tid-58BTeX)
* [自适应自旋锁](#tid-SM36F3)
* [4、流程图](#tid-4Mfy4J)
* [五、重量级锁](#tid-tHzFSk)
* [1、重量级锁核心原理](#tid-AEZEEK)
* [2、重量级锁案例](#tid-XQ2TKG)
* [六、参考文档](#tid-TRbp72)

   \_\_EOF\_\_

       - **本文作者：** [狂盗一枝梅](https://github.com)
 - **本文链接：** [https://github.com/kuangdaoyizhimei/p/18502947](https://github.com)
 - **关于博主：** 评论和私信会在第一时间回复。或者[直接私信](https://github.com)我。
 - **版权声明：** 本博客所有文章除特别声明外，均采用 [BY\-NC\-SA](https://github.com "BY-NC-SA") 许可协议。转载请注明出处！
 - **声援博主：** 如果您觉得文章对您有帮助，可以点击文章右下角**【[推荐](javascript:void(0);)】**一下。
      
