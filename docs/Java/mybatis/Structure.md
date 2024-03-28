# 设计模式
学习完Mybatis源码后,可以看到大量设计模式的使用

## 代理模式

## 模版模式
例如: BaseExecutor方法中,执行update方法,子类可以实现固定的算法实现
```java
public abstract class BaseExecutor implements Executor {
    
    protected abstract int doUpdate(MappedStatement ms, Object parameter) throws SQLException;

    @Override
    public int update(MappedStatement ms, Object parameter) throws SQLException {
        ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
        if (closed) {
            throw new ExecutorException("Executor was closed.");
        }
        clearLocalCache();
        return doUpdate(ms, parameter);
    }
}
```
## 策略模式
例如: 类型处理器,sql返回的值类型多种多样,实现类也存在多种。
```java
public interface TypeHandler<T> {
    
}
public class DoubleTypeHandler extends BaseTypeHandler<Double> {
    
}
```


