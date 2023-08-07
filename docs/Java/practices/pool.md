# what
Java池化技术主要用于提高系统的性能和资源利用率。

- 连接池：连接池技术用于管理数据库连接，避免频繁地创建和关闭连接，从而提高数据库操作的效率。连接池会预先创建一定数量的连接，并将其保存在连接池中，当需要连接时，直接从连接池中获取，而不是每次都创建新的连接。

- 线程池：线程池技术用于有效地管理和复用线程，避免频繁地创建和销毁线程，从而提高系统的并发能力和响应速度。线程池会维护一定数量的线程，并通过任务队列来管理待执行的任务，线程池会自动调度和执行任务。

- 对象池：对象池技术用于管理特定类型的对象，避免频繁地创建和销毁对象，从而提高系统的性能和资源利用率。对象池会预先创建一定数量的对象，并将其保存在对象池中，当需要对象时，直接从对象池中获取，而不是每次都创建新的对象。

- 字符串池：字符串池技术用于避免重复创建相同内容的字符串对象，从而节省内存空间。字符串池会维护一个池，保存已经创建的字符串对象，当需要创建字符串时，首先检查池中是否已经存在相同内容的字符串，如果存在，则返回已有的字符串对象，否则创建新的字符串对象并放入池中。

总的来说，池化技术可以通过复用资源来减少创建和销毁的开销，提高系统的性能和资源利用率，同时也可以防止出现资源耗尽的问题。

## 对象池
Tomcat和Jetty都用到了对象池技术，这是因为处理一次HTTP请求的时间比较短，但是这个过程中又需要创建大量复杂对象。对象池技术可以减少频繁创建和销毁对象带来的成本，实现对象的缓存和复用。

```java
protected static class ConnectionHandler<S> implements AbstractEndpoint.Handler<S> {
        //对象池
        private final RecycledProcessors recycledProcessors = new RecycledProcessors(this);
}
protected static class RecycledProcessors extends SynchronizedStack<Processor> {
    
}
public class SynchronizedStack<T> {
    
    //以空间换时间,并且还能减少GC的次数与时间
    private Object[] stack;

    public SynchronizedStack(int size, int limit) {
        if (limit > -1 && size > limit) {
            this.size = limit;
        } else {
            this.size = size;
        }
        this.limit = limit;
        stack = new Object[size];
    }
}
```
## 线程池
阿里巴巴的《Java开发手册》中强制规定「线程资源必须通过线程池提供，不允许在应用中自行显式创建线程」
```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }
}
```
## 实战
挖坑
Mybaits连接池设计分析

# other
池化技术更多的是一种思想,最核心的点(以空间换时间)。
