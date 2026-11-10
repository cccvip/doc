# what
在Java中，synchronized是一种关键字，用于实现线程同步。当一个方法或一个代码块被synchronized修饰时，它就被称为同步方法或同步代码块。

使用synchronized关键字可以确保在同一时间内只有一个线程可以访问被修饰的方法或代码块。当一个线程进入一个synchronized方法或代码块时，它会自动获取该方法或代码块的锁，其他线程将被阻塞，直到该线程释放锁。

# how

- 第一种 同步代码块
```java
public class TestMain {
    Object object = new Object();

    public void synchronizedParams() {
        synchronized (object) {
            
        }
    }
}
```
> synchronized(object) 在对某个对象上执行加锁时，会尝试在该对象的监视器上进行加锁操作，只有成功获取锁之后，线程才会继续往下执行。线程获取到了监视器锁后，将继续执行 synchronized 代码块中的代码，如果代码块执行完成，或者抛出了异常，线程将会自动对该对象上的监视器执行解锁操作。

- 第二种 同步方法
```java
public class TestMain {

    Object object = new Object();

    public synchronized void method() {
        //
    }
}
```
> 作用于方法，称为同步方法。同步方法被调用时，会自动执行加锁操作，只有加锁成功，方法体才会得到执行。如果被 synchronized 修饰的方法是实例方法，那么**这个实例的监视器**会被锁定。如果是 static 方法，线程会锁住相应的 **Class 对象的监视器**。方法体执行完成或者异常退出后，会自动执行解锁操作。

- DDL（单例模式）

```java
public class Singleton {
    private static volatile Singleton singleton;

    private Singleton(){

    }

    public static Singleton getInstance(){
        if(singleton==null){
            synchronized (Singleton.class){
                if(singleton==null){
                    singleton=new Singleton();
                }
            }
        }
        return  singleton;
    }
}
```

面试陷阱：如果不加volatile,会有问题吗？为什么

*需要先知道一个前置条件：JVM 中 `new` 一个对象大致分四步：①在堆上分配内存；②内存清零（零值初始化）；③设置对象头；④执行构造方法 `<init>` 完成真正的初始化。*

DCL 的问题就出在最后一步和"引用赋值"的顺序上。指令重排后，可能先把分配好的引用赋给 `singleton`，再执行构造方法：

> 线程 A 刚执行到第 ③ 步，引用已经指向了内存，但构造方法还没跑完；

> 线程 B 此时判断 `if (singleton == null)` 得到 false，直接拿去使用，而这时对象的属性还是默认值，并不是构造方法里赋的值。

## volatile

- volatile，会控制被修饰的变量在内存操作上主动把值刷新到主内存，JMM 会把该线程对应的CPU内存设置过期，从主内存中读取最新值。

- volatile 禁止指令重排靠的也是内存屏障，但具体是：**写** volatile 时，前面插 StoreStore、后面插 StoreLoad；**读** volatile 时，后面插 LoadLoad 和 LoadStore。这样能保证 volatile 写之前的普通写不会被重排到写之后，volatile 读之后的普通读写也不会被重排到读之前。（x86 上其中大部分会被优化成空操作，只剩 StoreLoad 有实际代价）

- volatile 并不能解决原子性，如果需要解决原子性问题，需要使用 synchronzied 或者 lock

- 针对上面的问题、volatile是如何规避这个风险?

> 参考第二点


## 锁升级
无锁-偏向锁-轻量级锁-重量级锁
- 无锁 001
- 偏向锁 101
- 轻量级锁 00
- 重量级锁 10
- （11 是 GC 标记）

锁修改的是对象头的markWord的锁标识位,以及设置threadId。

- 偏向锁
  无实际竞争，且将来只有第一个申请锁的线程会使用锁。
> 修改对象头的锁状态并记录持有线程 ID。偏向锁本身**不维护重入计数**；可重入是升级到轻量级锁之后靠 Lock Record 计数的。

什么时候升级?
线程2抢占资源的时候,如果发现当前资源依然被线程一持有,锁就会升级为轻量级锁


- 轻量级锁 无实际竞争，多个线程交替使用锁；允许短时间的锁竞争。

什么时候升级?
参考如上面一个过程,这一点很好理解。假如只有一个卫生间,但是有两个人要去抢占,一个人抢到了,第二个就会不停的去问,有没有空间。经过多次尝试都失败后。
第一个人干脆就把大门锁上,根本不会让第二个人来问。

- 重量级锁 有实际竞争，且锁竞争时间长。

> 控制并发的时候,大部分都会选择使用Synchorized,因为在1.8之后的版本升级,锁的膨胀对性能带来的损耗已经是微乎其微(追求极致的性能除外)


# other
指令重排是一种优化技术，可以提高程序的性能和执行效率。但在多线程编程时，需要注意指令重排可能引发的线程安全问题，并采取相应的措施来保证程序的正确性。