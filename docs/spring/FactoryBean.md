FactoryBean 是用于获取Bean，但和 BeanFactory 不太一样，BeanFactory 获取的是静态的类，就比如我们写的 Service 的各个实现类，它的逻辑就是我拿到你的 Bean 定义，然后一步步构造后作为 Spring 的 Bean。

FactoryBean 不关心怎么构造出来这个 Bean，对于他来讲，只要你实现了我的 getObject() 方法，我就能拿到你这个 Bean，这样他就更灵活，可以给一些第三方来用。

举个例子 Mybatis 的 Dao,只需要写一个 Dao 的接口可以，这就是 Mybatis 通过 FactoryBean 实现了一个MapperFactoryBean 可以获取到一个 BaseMapepr ，其中封装了可以从数据库获取数据的 CURD 等操作，这样我们写 DAO 只需要继承BaseMapper，MapperFactoryBean 会帮我们把 BaseMapper 所有实现的DAO都注入，并在其中封装好查询操作，我们使用时啥都不需要关心。 也就是说，FacotryBean 可以帮我们注入一类 Bean，这类 Bean 所需要的通用依赖或者信息我们可以通过 FactoryBean 实现类统一处理，并注入。