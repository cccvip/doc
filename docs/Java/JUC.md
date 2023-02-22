聊聊认识的锁?


## JUC框架锁
- synchorized
> 主要就是实现原子性操作和解决共享变量的内存可见性问题。

> 从内存来说，加锁的过程会清除工作内存中的共享变量，再从主内存读取，而释放锁的过程则是将工作内存中的共享变量写回主内存。

### 使用

第一种 代码块
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

第二种 方法
```java
public class TestMain {

    Object object = new Object();

    public synchronized void method() {
        //
    }
}
```
> 作用于方法，称为同步方法。同步方法被调用时，会自动执行加锁操作，只有加锁成功，方法体才会得到执行。如果被 synchronized 修饰的方法是实例方法，那么**这个实例的监视器**会被锁定。如果是 static 方法，线程会锁住相应的 **Class 对象的监视器**。方法体执行完成或者异常退出后，会自动执行解锁操作。

DDL（单例模式）

```java
public class Singleton {
    private static volatile Singleton singleton;

    private Singleton(){

    }

    public Singleton getInstance(){
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

> 如果不加volatile,会有问题吗？为什么

第一个线程在初始化初始化对象，设置 instance 指向内存地址时。第二个线程进入时，有指令重排。在判断 if (instance == null) 时就会有出错的可能，因为这会可能 instance 可能还没有初始化成功。

### 锁升级
无锁-偏向锁-轻量级锁-重量级锁
无锁 000
偏向锁 101
轻量级锁 10
重量级锁 11

锁修改的是对象头的markWord的锁标识位,以及设置threadId。




> 控制并发的时候,大部分都会选择使用Synchorized,因为在1.8之后的版本升级,锁的膨胀对性能带来的损耗已经是微乎其微(追求极致的性能除外)

- 分布式锁?

> 详见分布式那一块章节


## 底层概念
- 乐观锁
> 在读多写少的场景中使用,并发性低的场景中使用,默认是不会上锁,一般的实现是基于CAS
- 悲观锁
> 在读少写多的场景中使用,并发性高的场景中使用,每次写数据的时候都会上锁,ReentrantLock跟Synchorized都是这种实现之一

## 底层代码
- AQS队列?
voliate: 状态
CLH:双端优先线程队列
CAS:线程抢占

两种模式: 独占模式/共享模式

- 独占模式

> ReentrantLock,两种机制公平锁VS非公平锁。怎么体现公平,主要就看这个队列的数据是顺序出还是随机出
- 共享模式
> CountDownLatch,Semaphore等实现,大差不差的实现逻辑,扣细节就真的只能靠记忆了

> 过度深入的细节,还是参考其他人写的文章吧,但整体是很好理解的。强烈推荐 [https://javadoop.com/](https://javadoop.com/)

## 难点
- 偏向锁 
无实际竞争，且将来只有第一个申请锁的线程会使用锁。
> 修改对象头的锁状态,并设置线程ID,如果同一个线程有多次调用,则计数+1,这也是可重入的机制.

什么时候升级?
线程2抢占资源的时候,如果发现当前资源依然被线程一持有,锁就会升级会轻量级锁



- 轻量级锁 无实际竞争，多个线程交替使用锁；允许短时间的锁竞争。

什么时候升级?
参考如上面一个过程,这一点很好理解。假如只有一个卫生间,但是有两个人要去抢占,一个人抢到了,第二个就会不停的去问,有没有空间。经过多次尝试都失败后。
第一个人干脆就把大门锁上,根本不会让第二个人来问。

- 重量级锁 有实际竞争，且锁竞争时间长。







