## 单例模式


### 饿汉式
```java
public class Singleton {
    private static Singleton singleton = new Singleton();

    private Singleton(){

    }
    public static Singleton getInstance(){
        return  singleton;
    }
  }
```
### 懒汉式

```java
public class Singleton {
    private static volatile Singleton singleton = null;

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



