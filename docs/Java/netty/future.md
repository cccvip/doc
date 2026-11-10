要理解Future源码实现,首先就要理解Treiber Stack,一种无锁化的栈,基于CAS实现
[Treiber stack](https://en.wikipedia.org/wiki/Treiber_stack)

> 每当我们 push 进去一个元素的时候，我们首先根据要添加的元素创建一个 Node，然后获取原栈顶结点，并将新结点的下一个结点指向原栈顶结点。此时我们使用 CAS 操作来更改栈顶结点，如果此时的栈顶和之前的相同，代表 CAS 操作成功，那么就把新插入的元素设为栈顶；如果此时的栈顶和之前的不同（即其他线程改变了栈顶结点），CAS 操作失败，那么需要重复上述操作（更新当前的栈顶元素并且重设 next），直到成功。pop 操作的原理也相似

理解上面这句话,就离理解FutureTask实现更近一步。FutureTask的成员变量, 看一下FutureTask的成员变量

```java
//Callable是java多线程中一个函数式接口
private Callable<V> callable;

//线程执行结果
/** The result to return or exception to throw from get() */
private Object outcome; // non-volatile, protected by state reads/writes

//执行线程
/** The thread running the callable; CASed during run() */
private volatile Thread runner;

//Treiber栈,无锁化的栈,存放等待执行的线程
/** Treiber stack of waiting threads */
private volatile WaitNode waiters;
```

```java
public class FutureTask<V> implements RunnableFuture<V> {

    static final class WaitNode {
        volatile Thread thread;
        volatile WaitNode next;

        WaitNode() {
            thread = Thread.currentThread();
        }
    }

    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }

    private int awaitDone(boolean timed, long nanos)
            throws InterruptedException {
        //time out 超时时间
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        //等待节点
        WaitNode q = null;
        //是否队列
        boolean queued = false;

        for (; ; ) {
            //判断线程是否终止
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }
            //状态
            int s = state;
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            //执行已经完成,则让CPU调度器执行其他
            else if (s == COMPLETING) // cannot time out yet
                Thread.yield();
                //第一次循环,创建一个前置节点
            else if (q == null)
                q = new WaitNode();
                //第一次循环的时候将节点q,放置到栈上, 执行CAS方法,如果成功,代表着成功放置到栈上。没有被其他线程占用
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                        q.next = waiters, q);
                //暂时不看此方法
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);
            }
            //第二次循环,则执行LockSupport.park,表示当前线程将会等待
            else {
                LockSupport.park(this);
            }
        }
    }

    //run方法执行完成后,执行此方法,该方法将会释放资源,回收线程
    private void finishCompletion() {
        // assert state > COMPLETING;
        //step-1 
        for (WaitNode q; (q = waiters) != null; ) {
            //如果这个原子操作成功，说明成功获取到了等待队列的头节点并将其置空，此时将进入内层的无限循环进行后续对该节点及其后续节点的处理
            // 如果waiters==q,则将waiters设置为空,并且将出栈,并且将引用设置为null,帮助垃圾回收
            //假如第一步q赋值后,如果原子操作失败，可能存在并发竞争情况（其他线程可能在此时也在操作等待队列的头节点），则会继续下一次外层循环，再次尝试获取和处理等待队列的头节点。
            //step-2       
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (; ; ) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();
        callable = null;        // to reduce footprint
    }
}
```
> get()方法执行后,线程将会挂起,一直阻塞等待结果。

## FutureTask 的并发控制：无锁，而不是加锁

FutureTask 内部**没有使用锁**，而是靠 `volatile` + CAS（`Unsafe.compareAndSwapObject`）+ `LockSupport.park/unpark` 完成并发控制。所以不存在"锁竞争"，只存在 CAS 竞争和线程挂起/唤醒。

- 状态 state：由 volatile 修饰，状态迁移通过 CAS 完成，保证原子性。
- get 阻塞等待：多个线程调用 get 时，各自封装一个 WaitNode，通过 CAS 压入 `waiters` 这个 Treiber 栈；任务完成后由 `finishCompletion` 逐个 unpark。整个过程没有锁。
- 重复执行：`run()` 用 CAS 把 state 从 NEW 改成 COMPLETING，只有一个线程能成功；重复调用 run 的线程会直接返回，不会重复执行任务（但同一个 FutureTask 被提交到两个线程池本身就属于用法错误）。
- 结果可见性：`outcome` 的写入发生在 state 变为 NORMAL/EXCEPTIONAL 之前，`get()` 先读 state，靠 volatile 的 happens-before 保证能读到完整结果，不需要额外加锁。
- 与线程池协作：线程池里竞争的是"谁来执行"，FutureTask 本身仍然是 CAS 抢占，不会因为 get 而阻塞执行线程。

一句话：理解 FutureTask 要看它是**无锁 + Treiber 栈**，而不是锁。





