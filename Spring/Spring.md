# 1. Bean的生命周期

Bean的生命周期就是Bean从创建到销毁的过程。分别是实例化、属性赋值、初始化和销毁。

1. Spring首先通过反射来调用构造方法将原始对象New出来。但此时对象的属性还是空的。
2. 通过依赖注入，使用`@value`或者`@Autowired`将注解对应的值给填到属性中。如果使用了`@PostConstruct`注解，那么在这里就会触发这个方法。
3. 然后就是初始化，如果实现了`InitializingBean`接口，那么就会调用`afterPropertiesSet`方法，或者调用`init`方法。然后就是`BeanPostProcessor`的后置处理APO的动态代理就是在这里是生成的，将原始对象替换为代理对象然后放到单例池。这样就可以使用了。
4. 容器关闭时执行销毁步骤，如果有PostDestroy方法，那么会触发这个方法，然后调用destroy方法，最后调用destroy-method方法。

# 2. Spring IoC

IoC就是控制反转，将对象的创建和管理交给Spring来管理。

如果不使用IoC，那么对象之间的依赖需要手动New出来，比如接口需要使用特定的实现类来New出来，如果实现的方法更改了，那么这里的New方法也需要改变，耦合度高。

如果使用IoC那么只需要声明对象，使用Autowired就能让Spring自动创建实例。

# 3. 什么是Bean

Bean指的是被IoC容器管理的对象。需要告诉IoC容器帮助我们管理哪些对象，需要在XML文件的bean标签，或者注解来配置。

# 4. Bean注解

将类声明为Bean注解有四种。

* `@Component`。这是通用的注解，用来将任意一个类标注为Spring组件。
* `@Repository`。这是用于持久层，数据库相关操作的对象的Bean注解。
* `@Service`。这是用于服务层，处理业务逻辑的对象的Bean注解。
* `@Controller`。这是用于控制层，处理网络请求的对象的Bean注解。

# 5. @Component和@Bean的区别

@Component作用于类，将这个类作为Bean对象来交给Spring管理。@Bean作用于方法。@Component能够通过类路径扫描和自动装配到Spirng容器中，在@Configuration注解的类下的方法使用@Bean注解能够将这个方法返回的对象交给Spring管理，通过autowired自动注入来获取这个方法返回的对象。而且@Bean能够在方法内对需要返回的对象进行配置，返回一个配置好的对象。就比如引用第三方库时，就需要通过@Bean来获取。

# 6. 注入Bean的注解

Spring内置的@Autowired注解以及JDK内置的@Resource和@Inject都能注入Bean。

# 7. @Autowired和@Resource的区别是什么

@Autowired的注入逻辑上先按类型匹配，如果存在多个同类型Bean，再尝试按名称匹配。而如果一个接口有多个实现类，那么注入的时候会根据变量名与实现类名进行匹配，如果都不一致就会报错。

@Resource是JDK提供的注解，注入逻辑是先按名称匹配，如果存在多个同名称的Bean，就会按类型来匹配。而且Resource注解有两个属性，分别是name和type，如果指定类型，那么注入方式就是按类型注入，指定名称就是按名称来注入。

# 8. 注入Bean的方式

主要有三种注入方式。

1. 构造函数注入。在类中声明对象，然后在构造函数中的形参指定需要的Bean对象，就能自动注入该对象。

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    //...
}
```

2. Setter注入。先声明需要的对象，然后使用该对象的setter方法，并用@Autowired注解的方法来声明。

```java
@Service
public class UserService {

    private UserRepository userRepository;

    // 在 Spring 4.3 及以后的版本，特定情况下 @Autowired 可以省略不写
    @Autowired
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    //...
}
```

3. Field注入。在类中声明需要的对象，并且用@Autowired注解即可自动注入。

```java
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    //...
}
```

# 9. Bean的作用域

Spring提供了6种Bean作用域可以在xml文件或者scope注解中配置。

1. singleton。IoC容器只有唯一的Bean实例。默认就是单例模式。
2. prototype。IoC容器每次获取Bean对象都会创建新的Bean实例。
3. request。每次HTTP请求都会产生新的Bean，这个Bean只在当前request内生效。
4. session。HTTP的每个session都会创建新的Bean实例。
5. application。Web应用时创建一个Bean，只在当前应用启动时间内有效。
6. websocket。每一次WebSocket会产生一个新的Bean。

# 10. Bean的线程安全性

是否线程安全取决于它的作用域和状态。

在prototype下，每次获取都会创建新的Bean实例，不存在资源竞争的问题，因此线程安全。但singleton作用域下全局共享唯一实例，如果Bean对象有状态，就会存在线程安全问题。如果Bean对象无状态，只存在可调用的方法，那么依然是线程安全的。如果Bean必须要有状态的话，最好使用Synchronized或者ReentrantLock来同步，或者将变量保存在ThreadLocal中，保持线程独立。

# 11. Spring AOP

Spring AOP是面向切面编程，将那些与业务无关，却被业务模块共同调用的逻辑封装起来，能够减少系统的重复代码和耦合度。

Spring AOP是基于动态代理的，如果要代理的对象实现了某种接口，那么会使用Proxy来创建代理对象，而普通对象会使用Cglib生成一个被代理对象的子类来作为代理。

# 12. AOP 常见通知

* Before前置通知。目标对象方法调用前触发。
* After后置通知。目标对象方法调用后触发。
* AfterReturning返回通知。方法调用完成，返回结果值后触发。
* AfterThrowing异常通知。如果方法抛出或触发异常时触发。
* Around环绕通知。环绕通知能够直接获取目标方法，可以在通知内修改参数、修改返回值等，通过ProceddingJointPoint来控制目标方法的执行时间。

# 13. Spring MVC

MVC是Model、View和Controller的简写，思想是将业务逻辑、数据和显示分离来组织代码。

首先，前端的请求都会先被拦截到DispatcherServlet，DispatcherServlet根据请求信息来调用HandlerMapping，根据RequestMapping或者Restful的Mapping注解来构建Handler的映射表，然后根据URL来匹配对应的Controller，并且将Controller方法和Handler方法一起封装，使用HandlerAdapter来执行Handler。Handler处理完，会返回ModelAndView对象给DispatcherServlet，根据这里的view找到实际的view，将model传给实际的view后显示到浏览器。

但现在前后端分离的情况下，后端的controller只需要返回数据，Spring就能转换为JSON格式来返回给前端。

# 14. Spring 设计模式

1. 工厂模式。

* BeanFactory。能够用来创建和管理Bean对象。
* ApplicationContext。是BeanFactory的高级版本，添加了一些新的功能。

2. 单例模式Singleton Pattern。在获取Bean对象时保证整个容器只有一个实例。
3. 原形模式Prototype Pattern。每次获取Bean对象时都会创建新的实例。
4. 代理模式Proxy Pattern。用来实现AOP。如果目标类实现了接口，就使用JDK动态代理，如果没有则使用Cglib代理。
5. 装饰器模式Decorator Pattern。在不改变原类的基础上，动态拓展功能。
6. 适配器模式Adapter Pattern。能够将不兼容的接口转换为统一接口。比如MVC中用HandlerAdapter将各种Handler适配为统一的接口，DispatcherServlet直接调用handle方法即可。

# 15. Spring循环依赖

循环依赖是两个Bean对象相互依赖，导致双方均无法完成初始化。

```java
@Service
public class A {
    @Autowired
    private B b;
}
// 另一个文件
@Service
public class B {
    @Autowired
    private A a;
}
```

这样如果没有特殊处理，创建A对象时需要B，而创建B对象时需要A，而A还没创建完，就会死锁。

Spring使用三级缓存来解决这个问题。三层缓存都是使用map来构建的。

一级缓存是singletonObjects，保存的是已经完成创建并初始化的Bean对象，可以直接拿来使用。二级缓存是earlySingletonObjects，保存的是Bean对象的引用，此时对象已经创建，但还没初始化，需要进行赋值。而三级缓存是singletonFactories，保存的是ObjectFactory对象，需要通过getObject来获取Bean对象。

这里的流程如下。

首先需要创建A对象。Spring通过反射调用无参构造，将A实例化。此时A的属性还没有填充。实例化完成，就直接将生产A的ObjectFactory给存入三级缓存。然后开始填充A的属性，发现需要B，那么就为A注入依赖，开始创建B对象。同样的，B对象实例化后也将生产B的ObjectFactory放到三级缓存中。这时为B的属性赋值，逐一查看缓存，发现三级缓存存在A的ObjectFactory，就调用getObject的方式，将A对象存放到二级缓存，同时删除三级缓存的A。这样，就能用二级缓存的A对象为B的属性赋值，完成B的初始化，将这个可以使用的B对象放到一级缓存，然后将三级缓存的B工厂删除。B完成初始化，接下来回到A对象，A拿到B对象赋值，也放到一级缓存中，将二级缓存的A对象给删除，这样A和B就都初始化完毕了。

# 16. @Lazy能否解决循环依赖

@Lazy的关键是采用代理机制来初始化Bean对象。如果对Bean对象使用Lazy注解，这个Bean对象不会被创建，而是生成一个代理对象来注入。因此在创建A对象是，能够直接拿到B对象的代理对象，完成A的初始化。@Lazy能够解决构造器注入的问题。

# 17. 是否允许循环依赖

在Spring Boot 2.6之后就禁止了循环依赖的使用。如果出现循环依赖，就会报错。这时需要在配置文件中设置允许循环依赖，或者在循环依赖的Bean上使用延迟加载。

# 18. Spring管理事务的方式

Spring管理事务主要由两种。

* 声明式事务。在xml文件或使用注解的方式来声明事务。底层原理是AOP。在执行这个方法前开启事务，如果抛出异常就会回滚，顺利执行就提交。
* 编程式事务。在代码中使用TransactionTemplate来手动管理事务，如果遇到异常就手动回滚，顺利执行的话最后返回true就会提交。

# 19. 事务传播行为

当一个事务中开启另一个事务的时候，就涉及到事务传播行为。总共有7种传播行为，

1. Propagation required(TransactionDefinition.PROPAGATION_REQUIRED)。开启事务时，如果当前存在事务，就加入这个事务。如果没有事务，就创建一个新的事务。

2. Propagation requires new (TransactionDefinition.PROPAGATION_REQUIRES_NEW) 。无论当前是否存在事务，都会开启新事务。当前事务会被挂起。适合事务内存在必须提交的事件。
3. Propagation nested (TransactionDefinition.PROPAGATION_NESTED)。如果当前存在事务，就在嵌套事务内执行。如果嵌套事务执行失败，只会回滚这个嵌套事务，外部事务不受影响。
4. Propagation supports。如果当前存在事务，就加入事务。如果不存在事务，不会开启事务。
5. Propagation not supported。以非事务的方式执行操作。如果存在事务，就将当前事务挂起。
6. Propagation mandatory。如果存在事务就加入，如果不存在事务则抛出异常。
7. Propagation never。以非事务的方式执行，存在事务就抛出异常。

 # 20. 事务隔离级别

在高并发的环境下，多个事务同时操作数据库会导致数据不一致的问题。

1. 脏读。事务A读取了事务B未提交的数据，如果B回滚，那么A读取的数据是不存在的数据。
2. 不可重复读。同一个事务内多次读取同一行数据，结果不一致。这是两次读取之间，其他事务提交了数据。
3. 幻读。事务A多次查询，每次查询获取的数据行数不一致。

因此，Spring使用Isolation枚举类定义了四种隔离级别。

1. Read Uncommitted读未提交。允许读取其他事务未提交的数据。
2. Read Committed读已提交。只能读取其他事务已经提交的数据。这能解决**脏读**问题。
3. Repeatable Read可重复读。确保同一个事务中多次查询到的结果一致。解决脏读和不可重复读。
4. Serializable串行化。事务串行处理，一次只能执行一个事务。能够解决三个问题。

# 21. Transactional注解的rollbackFor属性

Exception分为运行时异常和非运行时异常。而普通的Transactional注解是只有遇到运行时异常RuntimeException或者Error才会回滚事务，而不会回滚受检异常Checked Exception。因为Spring认为运行时异常和Error是不可预期的错误，受检异常是可预期的，可以通过业务来处理。因此，可以通过rollbackFor和noRollbackFor来指定哪些异常需要回滚，哪些不需要。

# 22. Spring Boot内嵌Servlet容器

Spring Boot提供了三种内嵌的Web容器，分别是Tomcat、Jetty和Undertow。

引用了spring-boot-starter-web依赖后，Spring Boot会默认使用Tomcat来作为Servlet容器。如果要使用Jetty或者Undertow，就要在maven文件中排除Tomcat依赖，并添加对应Servlet容器的依赖。

* Tomcat适用于大部分的Web应用，但高并发场景下性能不足。
* Undertow轻量，启动快且占用资源少，支持非阻塞io，高并发场景性能良好。
* Jetty内存占用少，并且处理长连接较好。

# 23. Spring Boot自动装配

Spring Boot的自动装配通过@SpringBootApplication注解来启动的。这个注解中包含了@EnableAutoConfiguration注解，专门用来进行自动配置，允许SpringBoot自动配置Spring应用。还带有@ComponentScan来启动组件扫描，@Configuration来注册额外的Bean和导入配置类。

**还有其他的内容需要了解。**

# 24. Spring Boot如何加载配置文件

Spring Boot会从嘞路径的根目录来查找application.properties或者application.yaml文件。如果同时存在properties和yaml文件，会优先加载properties。而如果没有提供配置文件，Spring Boot就会加载默认的配置。

# 25. Spring Boot加载配置文件的优先级

Spring Boot加载配置文件的原则是后加载的覆盖先加载的，而且离用户越近，优先级越高。首先加载的是项目的config目录下的配置文件，然后加载根目录下的配置文件，再加载类路径的config目录的配置文件，最后加载类路径下的配置文件。这样，项目根目录的配置文件优先级最低，而类路径根目录下的配置文件优先级最高。