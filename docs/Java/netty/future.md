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

## FutureTask锁竞争出现的情况

内部状态的并发访问
> FutureTask 的内部状态是通过一些变量来维护的，例如表示任务状态的 state 变量等。在多线程环境下，多个线程可能会同时访问和修改这些状态变量，从而导致锁竞争。例如，当一个线程正在执行任务并尝试更新任务的状态为完成状态时，另一个线程可能同时也在查询任务的状态，这就需要对状态变量的访问进行同步，否则可能会导致数据不一致的问题。

get 方法的阻塞等待
> FutureTask 的 get 方法用于获取异步计算的结果。当任务尚未完成时，调用 get 方法的线程会被阻塞，直到任务完成并返回结果。在这个阻塞等待的过程中，多个线程可能同时调用 get 方法等待同一个 FutureTask 的结果，这些线程会竞争获取与该 FutureTask 相关的锁，以确保它们能够正确地等待和获取结果。具体来说，FutureTask 内部使用了 LockSupport.park() 等机制来实现线程的阻塞等待，而这些操作通常需要获取和释放锁来保证线程的正确调度和数据的一致性，从而导致了锁竞争。

任务的重复提交与并发执行
> 如果多个线程同时提交相同的 FutureTask 实例到不同的线程池或直接作为线程的执行任务，就可能导致该 FutureTask 被多次执行，进而引发锁竞争。虽然 FutureTask 本身有一定的机制来处理任务的重复执行问题，但在并发情况下，多个线程同时判断任务是否已经开始执行或是否已经完成等操作时，仍然可能会产生竞争条件，需要通过锁来保证操作的原子性和正确性。

结果的缓存与并发访问
> FutureTask 在任务完成后会缓存计算结果，以便后续的 get 方法调用可以直接返回结果而无需重新计算。然而，当多个线程同时调用 get 方法且任务刚刚完成时，这些线程会竞争访问和获取缓存的结果，这也可能导致锁竞争。为了确保结果的一致性和线程安全，FutureTask 需要对结果的缓存和访问进行适当的同步，通常会使用锁来实现。

与其他并发操作的交互
> 在实际应用中，FutureTask 往往会与其他并发操作或数据结构一起使用，例如与线程池、并发集合等结合。当多个线程通过线程池提交 FutureTask 并与线程池中的其他任务或线程进行交互时，可能会因为资源的共享和并发访问而引发锁竞争。例如，线程池中的线程可能会同时竞争获取执行 FutureTask 的机会，或者在将 FutureTask 的结果存储到共享的数据结构中时产生竞争





