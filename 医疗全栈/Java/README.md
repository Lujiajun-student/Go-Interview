# 1. Java基础

## 1. 基础类型

Java有8中基本类型。

byte, short, int, long

float, double

char

boolean

## 2. String, StringBuilder, StringBuffer

String不可变。

如果发生String对象的修改，不是在原对象上修改，而是创建了新字符串。

StringBuilder可变。

底层维护了一个字符数组，通过append来进行拼接，但可变带来线程安全问题，但性能高。

```java
abstract class AbstractStringBuilder implements Appendable, CharSequence {
    // 1. 真正存储字符串数据的字节数组
    byte[] value;

    // 2. 标识编码格式（是用 LATIN1 还是 UTF16）
    byte coder;

    // 3. 当前已经使用的组件数量（即字符串的实际长度）
    int count;
}
```

StringBuffer可变。

与StringBuilder一致，但添加了synchronized锁。线程安全，但性能差。

## 3. equals()与hashCode()

==比较的是基本类型的值，对引用类型比较地址。

equals是通过自定义的方法来比较两个对象是否想等。

而如果自定义了对象，那么就需要重写equals和hashCode方法。

在HashMap等存储引用类型的结构中，首先会通过hashCode来判断当前的map是否存在该对象，但由于哈希值会有冲突，所以会对与目标hashCode值相同的所有对象再进行一次equals来比较。如果只重写了hashCode不重写equals，会导致存储了对象，但取不出来。

# 2. 面向对象

## 1. 封装

封装是将对象的属性和行为绑定到一个类中，通过访问控制修饰符来对外隐藏成员，只暴露公共接口来让外部操作对象。

## 2. 继承

继承指子类可以使用父类的属性和行为，并通过方法重写来修改父类的实现，或者方法重载来拓展父类的功能。

## 3. 多态

多态是同一个方法可以作用于不同类型的对象，并产生不同的行为。

## 4. 面向对象原则

单一职责原则、开放封闭原则、里式替换原则、依赖倒置原则、接口分离原则

# 3. 集合

Java主要有两类集合。

1. Collection。

Collection下分为三个类，分别为List、Set、Queue。

List有ArrayList、LinkedList、Vector。

Set有HashSet、LinkedHashSet、TreeSet。

Queue有PriorityQueue、BlockingQueue。

2. Map。

Map下有五类。分别为HashMap、LinkedHashMap、TreeMap、HashTable、ConcurrentHashMap。

> 如果讲到线程安全的Map，除了HashTable、ConcurrentHashMap，还有SynchronizedMap和ConcurrentSkipListMap。

## 1. ArrayList

底层是动态数组。

特点是随机访问快，时间复杂度O(1)。

尾部插入快。O(1)。

头部和中间插入删除慢，因为需要移动元素。

创建的时候，容量默认为10，扩容时变为原来的1.5倍。

## 2. LinkedList

底层是双向链表。

特点是随机访问慢，O(n)。

头尾插入快，O(1)。

可以作为队列或者双端队列来使用。

但不常用，因为不连续存储，局部性差，缓存不友好。

## 3. HashMap

JDK1.7以前，底层结构为数组+链表，如果多个元素放到同一个数组桶，那么就需要通过头插法拉链法来将多个元素连起来。

在JDK1.8以后，底层结构就变为数组+链表+红黑树。如果多个元素放到同一个数组桶，通过尾插法和拉链法将多个元素连起来。当元素个数超过8个，链表就会转化为红黑树保存。

HashMap有一些核心参数，如initialCapacity初始容量为16，loadFactor负载因子默认0.75，threshold扩容阈值，达到阈值就扩容。

## 4. LinkedHashMap

在HashMap的基础上维护双向链表，保证遍历顺序。

可以按插入顺序来获取元素。

## 5. TreeMap

底层为红黑树，key有序。根据key大小顺序来遍历取出。

## 6. HashSet

底层是HahsMap，通过add添加元素时，内部调用了map.put来添加。

## 7. 异常并发

如果ArrayList、HashMap等有线程安全问题的类型出现了并发修改，那么就会直接抛出`ConcurrentModificationException`，表明当前类型不支持并发。

# 4. 异常

Java的异常基类为Throwable。

Throwable有两个子类，分别为Error错误和Exception异常。

Exception异常有RuntimeException运行时异常和Checked Exception受检异常。

## 1. Error

Error表明JVM层面的错误，不应该被捕获。比如OutOfMemoryError、StackOverflowError、NoClassDefFoundError等，这些错误无法被程序修复，应该直接报出来。

其中，NoClassDefFoundError是指编译通过，但部署的时候发现class文件损坏或者不存在，才会出现错误。

## 2. Checked Exception

受检异常，JVM认为方法存在异常的可能，必须处理或抛出。

如IOException、SQLException或者ClassNotFoundException。对外部的操作JVM认为不可预知，不可预知就需要考虑出错的可能，必须处理。

ClassNotFoundException指程序运行时寻找类找不到，才会报异常。

# 5. 泛型

泛型用来解决类型安全和代码复用的问题。

```java
class Box<T> {
    private T value;
    
    public void set(T value) {
        this.value = value;
    }
    
    public T get() {
        return value;
    }
}

public static void main (String[] args) {
    Box<String> box = new Box<>();
    box.set("Hello");
    System.out.println(box.get());
}
```

泛型能够适配多种类型，避免了为每种类型都编写一次方法。

## 1. 泛型擦除

Java泛型是伪泛型，在编译期存在，编译完成后就会被擦除。

```java
List<String> list1 = new ArrayList<>();
List<Integer> list2 = new ArrayList<>();

System.out.println(list1.getClass() == list2.getClass()); // true
```

在编译时，如果没有限制通配符`T extends Integer>`，那么泛型擦除后会替换为Object，否则会替换为具体类型，如Integer。

在具体使用中，会根据`<Type>`具体输入的类型来进行自动强转。

编译前：

```java
Holder<String> holder = new Holder<>();
String str = holder.getValue();
```

编译后：

```java
Holder holder = new Holder();
String str = (String) holder.getValue(); // 编译器自动帮你加了 (String)
```

这种伪擦除是为了实现向后兼容，泛型出来后，需要考虑泛型出来前的代码也能继续运行。

# 6. 反射

反射允许程序运行时获取类信息、创建对象、访问字段等。

```java
Class<?> c1 = String.class;
Class<?> c2 = "abc".getClass();
Class<?> c3 = Class.forName("java.lang.String");

// 创建对象
Class<?> clazz = Class.forName("com.example.User");
Object obj = clazz.getDeclearedConstructor().newInstance();

// 调用方法
Method method = clazz.getDeclearedMethod("setName", String.class);
method.setAccessible(true);
method.invoke(obj, "Tom");

// 访问字段
Field field = clazz.getDeclearedField("name");
field.setAccessable(true);
field.set(obj, "Tom");
```

反射用于Spring创建Bean、MyBatis映射数据库字段到对象、AOP创建代理类等。

反射的缺点是性能低于直接调用，破坏了封装，且反射属于运行时，出错了在编译期无法检查。

# 7. 注解

注解是标记在类、方法、字段上的元数据。Java会读取这些标签，并根据标签的内容作处理。

如@Override，会读取方法，看是否是重写了父类或接口的方法，如果不是就报错。

而注解有编译期处理和运行期处理，运行期处理通过生成代理对象来实现，编译期处理通过使用APT注解处理器实现，在编译期扫描注解并生成动态Java文件。

# 8. Lambda

Lambda本质是一个匿名函数，允许将一个方法或函数向传递参数一样进行传递。

在以前，创建新线程并运行的方法如下。

```java
// 为了实现一个简单的打印任务，不得不写 5 行模板代码
Runnable r = new Runnable() {
    @Override
    public void run() {
        System.out.println("线程运行中...");
    }
};
new Thread(r).start();
```

使用Lambda就能够进行简化。

```java
new Thread(() -> System.out.println("线程运行中...")).start();
```

其中，`->`是箭头操作符，将表达式分为两部分。`(参数列表) -> {方法体}`，不需要函数名和参数类型。如果只有一行代码且有返回值，那么return必须省略。`s -> s.length()`。

其中，匿名内部类编译后会产生新的.class文件，而Lambda没有新文件。程序遇到Lambda时，会在当前类创建私有的成员方法，在内存中创建代理类，然后运行这个代理类来代替Lambda函数。

# 9. Stream

Stream用于对集合数据的处理。

Stream有三个过程。

* 创建流。将现有的集合、数组等创建为流。
* 中间操作。对当前流进行操作。
* 终端操作。关闭流，将生成的数据返回。

```java
List<User> result = userList.stream()
    .filter(u -> u.getAge() > 18) // 选取年龄大于18岁
    .filter(u -> u.getName().startsWith("章")) // 过滤姓
    .limit(2) // 截取前两个
    .collect(Collectors.toList()); // 打包成List
```

流的操作如果没有终止操作就不会运行。

# 10. IO

IO分为字节流和字符流。

字节流适合处理图片、视频等二进制文件。如InputStream, OutputStream等。

字符流适合处理文本文件。如Reader, Writer, FileReader等。

## 1. 装饰器

装饰器允许在不修改原有代码也不继承的情况下拓展新的功能。

```java
// 4. 具体装饰器：加奶
class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) { super(coffee); }

    @Override
    public double getCost() {
        return super.getCost() + 3.0; // 在原价基础上加 3 元
    }

    @Override
    public String getDescription() {
        return super.getDescription() + " + 加奶";
    }
}

// 4. 具体装饰器：加糖
class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee coffee) { super(coffee); }

    @Override
    public double getCost() { return super.getCost() + 1.0; } // 加 1 元

    @Override
    public String getDescription() { return super.getDescription() + " + 加糖"; }
}
```

## 2. NIO

NIO主要有Buffer，Channel，Selector三类。

Buffer是缓冲区，Channel是通道，Selector是多路复用器。

传统IO是阻塞式，一个线程调用IO时需要阻塞，而NIO是非阻塞，一个线程可以管理多个连接。

## 3. 零拷贝

普通的拷贝需要将数据读取到内核缓存，然后读取到用户缓冲区，再读取到Socket缓冲区，最后才能发送数据。这样会出现多次拷贝导致性能问题。

因此，可以用mmap()代替read，此时用户态和内核态共享一个物理内存，也就是从磁盘读取到缓冲区，从缓冲区读取到Socket缓冲区，减少了一次拷贝。

还有sendfile，CPU将数据的地址发送到Socket缓冲区，网卡读取地址后直接从内核缓冲区获取数据发送。

## 4. 序列化

序列化将对象转换成字节流，反序列化是将字节流恢复成对象。但Java序列化性能一般，后端一般采用JSON来进行序列化。

# 11. JVM内存

JVM运行时内存区域是指JVM将内存划分为哪些区域，Java内存模型JMM是指多线程下变量的可见性问题。

JVM内存模型包含堆、栈、方法区、程序计数器、本地方法栈。

* 程序计数器记录线程执行到哪个字节码指令，线程切换需要依靠它来恢复位置。

* 栈。每个线程都有自己的栈，用来存放方法的栈帧，栈帧存放局部变量表，操作数栈，方法返回地址等。
* 本地方法栈。给本地方法使用，用来调用C/C++的方法。
* 堆。对象和字符串常量池的存储位置。new出来的对象通常会在堆上，或者出现内存逃逸时，如返回了局部变量的指针，那么这个变量就会从局部变量表中逃逸到堆。
* 方法区。主要存放类信息、静态变量、常量等。

# 12. 类加载机制

Java类加载包含`加载，验证，准备，解析和初始化`。

* 加载首先通过类的全限定名获取二进制字节流，通过字节流在方法区添加类信息，在堆中new一个Class对象。
* 验证需要看字节码是否合法。
* 准备，为类静态变量分配内存并设置默认值。
* 解析，将符号引用转换为直接引用。
* 初始化，执行类构造器，为静态变量赋值，执行静态代码块。

## 12.1 双亲委派机制

类加载器从上到下分别为启动类加载器、拓展类加载器、应用程序类加载器、自定义类加载器。

一开始自下而上进行加载，当前类加载器优先将加载类请求交给父加载器，如果父加载器加载不了才到本类加载。

# 13. 垃圾回收

垃圾回收需要解决三个问题：哪些需要回收、什么时候回收、怎么回收。

## 1. 判断对象是否存活

有引用计数法和可达性分析，主要使用可达性分析。从GC Roots触发，能够访问到的对象是存活对象，标记出来后，将剩余不可达对象回收。包括栈引用的对象、方法区静态属性引用对象、常量对象、Synchronized持有的对象等。

## 2. 引用类型

有强引用、软引用、弱引用、虚引用。

强引用是直接new对象。只要引用存在就不会回收。

```java
Object obj = new Object();
```

软引用在内存不足的时候会被回收。

```java
SoftReference<Object> ref = new SoftReference<>(new Object());
```

弱引用在下一次GC就会回收。

```java
WeakReference<Object> ref = new WeakReference<>(new Object());
```

虚引用不创建对象，用来跟踪对象被回收的状态。

## 3. GC算法

* 标记清除。先标记存活对象，然后清除未标记对象。
* 标记复制。对象只在一半区域内创建，满了就复制存活对象到另一半区域，然后清空原一半区域。新生代用的这种。
* 标记整理。标记后将存活对象移动到一端，将边界外的内存清空。老年代用的这种。
* 分代收集。

思想是大多数对象朝生夕死，新对象先进入新生代的Eden区，如果满了，就会将Eden和From Survivor两个区域的存活对象存储到To Survivor，然后将From Survivor和To Survivor进行交换，保证From用来存储对象。

每进行一次Minor GC，对象的年龄就会増大。年龄达到15以上，就会保存到老年代。而且大对象也会保存到老年代中。

如果老年代满了，就会进行一次Full GC，此时会触发STW，暂停所有线程，将整个堆空间进行一次垃圾回收。

这是现在使用的G1，G1最大的特点是标记对象是能够和用户线程并行，但是转移对象的时候必须暂停用户线程，搬运完成后才能继续。

而ZGC包含了着色指针，在这个指针上能够直接查看对象状态，如是否被标记等。并且垃圾回收后剩余的对象需要转移时不需要暂停用户线程。如果此时用户恰好访问该对象，那么用户线程会触发读屏障，读屏障从对象的着色位查看当前对象正在被转移，那么用户线程就会将对象转移到正确位置，并修改自己的指针，也就是读屏障+自愈机制。并且ZGC早期没有分代，后期也分为新生代和老年代。

## 4. Full GC原因

老年代空间不足，元空间不足，调用`System.gc()`，大对象分配失败等。

## 5. JVM调优

常见参数如下。

```bash
-Xms2g                 初始堆大小
-Xmx2g                 最大堆大小
-Xmn512m               新生代大小
-Xss1m                 每个线程栈大小
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m
-XX:MaxDirectMemorySize=512m
```

# 14. 多线程

要让一个类有创建线程的能力，可以选择继承Thread，或者实现Runnable，或者实现Callable。

1. 继承Thread。

这里需要重写run方法，这样创建这个类的实例并调用start即可。

```java
public class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + " 正在运行...");
    }

    public static void main(String[] args) {
        MyThread t = new MyThread();
        t.start(); // 注意：必须调用 start() 来启动线程，直接调用 run() 只是普通方法调用
    }
}
```

2. 实现Runnable接口。

为了解决单继承的问题，让当前类实现接口更合适。

```java
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + " 正在运行...");
    }

    public static void main(String[] args) {
        // 利用 Lambda 表达式甚至可以更简写：
        // Runnable r = () -> System.out.println("运行中...");
        
        MyRunnable runnable = new MyRunnable();
        Thread t = new Thread(runnable);
        t.start();
    }
}
```

3. 实现Callable。

```java
import java.util.concurrent.Callable;
import java.util.concurrent.FutureTask;

public class MyCallable implements Callable<Integer> {
    @Override
    public Integer call() throws Exception {
        // 模拟耗时计算
        Thread.sleep(1000);
        return 666; 
    }

    public static void main(String[] args) throws Exception {
        MyCallable callable = new MyCallable();
        // 用 FutureTask 包装 Callable
        FutureTask<Integer> futureTask = new FutureTask<>(callable);
        
        new Thread(futureTask).start();

        // get() 方法会阻塞当前线程，直到多线程计算任务完成并返回结果
        Integer result = futureTask.get();
        System.out.println("线程返回的结果是: " + result);
    }
}
```

通过FutureTask来封装实现了Callable的类，然后运行futureTask，通过futureTask.get就能获取到方法执行完毕的返回值。

## 1. 生命周期

主要有NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED。

## 2. 线程中断

通过interrupt()不是杀死线程，而是设置中断标志。阻塞方法遇到中断会抛出InterruptedException异常。所以最好通过trycatch来运行，捕获到中断标志后可以在catch中运行结束代码。

# 15. 线程池

线程池能够避免频繁的创建和销毁线程带来的开销，并且可以对线程进行管理。

核心类是ThreadPoolExecutor。

## 1. 线程参数

主要有七大参数。

* corePoolSize是核心线程数。
* maximumPoolSize是最大线程数。
* keepAliveTime是非核心线程空闲存活时间。
* unit是时间单位。
* workQueue是任务队列。
* threadFactory是线程工厂。
* rejectedExecutionHandler是拒绝策略。

## 2. 执行流程

提交任务后，首先看当前线程池是否有线程，如果当前线程小于核心线程数，那么就会创建线程来运行。否则就会放到队列。如果任务队列满了，那么就看线程数是否达到最大线程数，如果没有，就开启临时线程来执行。如果满了，那么就会根据拒绝策略来处理任务。

## 3. 常见队列

ArrayBlockingQueue是有界数组阻塞队列。

LinkedBlockingQueue是无解阻塞队列。默认任务队列无限。

SynchronousQueue同步队列，不存储任务，直接寻找线程来执行。

PriorityBlockingQueue。优先级队列。任务带有优先级，优先选择优先级大的任务执行。

DelayQueue。延迟队列。

## 4. 拒绝策略

AbortPolicy。直接抛异常。

CallerRunsPolicy。由提交任务的线程执行。

DiscardPolicy。直接丢弃。

DiscardOldestPolicy。丢弃最老的任务。

## 5. Executors

不推荐使用Executors创建线程池。

```java
Executors.newFixedThreadPool()
Executors.newCachedThreadPool()
Executors.newSingleThreadExecutor()
```

newFixedThreadPool使用LinkedBlockingQueue，可能出现队列无限任务。

newCachedThreadPool最大线程数无限，可能创建过多线程。

因此推荐使用ThreadPoolExecutor来创建线程池。

## 6. 线程池大小计算

如果是CPU密集型任务，如作运算，需要保证线程不会经常切换，线程数=核心数+1。

如果是IO密集型任务，如读取文件，需要让线程能够轻松切换，线程数=核心数*2。

# 16. Synchronized

Synchronized是Java内置的重量级锁。

如果修饰实例方法，那么会锁当前实例，如果修饰静态方法，那么会将整个类锁住，如果修饰代码块，锁住的是使用到的对象。

Synchronized保证原子性、可见性和有序行。

进入synchronized会获取锁，退出synchronized会释放锁。释放锁之前会将工作内存的修改更新到主内存。获取锁会从主内存更新到工作内存中。

## 1. 锁升级

Synchronized有锁优化。

```
无锁->偏向锁->轻量级锁->重量级锁
```

在没竞争的时候不加锁，而有轻微竞争的时候锁升级，竞争激烈就升级为重量级锁。

## 2. wait/notify

wait, notify, notifyAll需要在synchronized中使用。

如果线程运行条件不满足，那么需要通过wait来等待，直到其他线程使用notify，才会唤醒该线程。

wait需要通过while来判断，如果不满足条件，需要进入等待。被唤醒后，需要重新检查条件，满足了才能执行业务。如果使用if，那么唤醒后不会检查条件，直接执行业务，就会存在时间差，唤醒和执行业务期间可能被其他线程修改条件，导致条件又不满足，执行业务就会出现错误。

# 17. volatile

volatile用于保证变量的可见性和禁止指令重排序。

```java
private volatile boolean running = true;
```

如果线程a修改了变量，那么线程b能够及时看到。

因为线程中有工作内存，从主内存读取数据后，读取同一个数据时只会从自己的工作内存中读取来保证性能，导致主内存的数据发生改变后，修改的数据对线程不可见。volatile能够保证当前修饰的变量的工作内存失效，线程不能从工作内存读取这个变量，只能从主内存中读取，保证了其他线程的修改对当前线程可见。

但volatile只保证可见性，不保证原子性，因为没有实现原子操作。

volatile也能防止指令重排序。

对象创建的过程有分配内存、初始化对象、修改引用三个步骤，如果有重排序，其他线程就可能获取到一个没有初始化的对象。

# 18. Lock

Lock是锁，实现是ReentrantLock。

```java
Lock lock = new ReentrantLock();

lock.lock();

try {
    // 临界区
} finally {
    lock.unlock();
}
```

ReentrantLock是可重入锁，同一个线程可以多次访问同一个锁，并且支持公平锁和非公平锁，支持中断等待，支持超时获取锁。

## 1. tryLock

这是尝试获取锁，如果获取锁，返回true，后续执行完毕就lock.unlock()。如果没有获取锁就会返回false。

## 2. Condition

Condition是资源，能够用来实现线程的同步，与wait和notify类似。

```java
Condition notEmpty = lock.newCondition();
lock.lock();
try {
    while(queue.isEmpty()) {
        notEmpty.await();
    }
} finally {
    lock.unlock();
}
```

其他县城通过notEmpty.signal()来唤醒这个阻塞线程。

# 19. AQS

AQS是抽象队列同步器，像ReentrantLock、Semaphore、CountDownLatch都是基于AQS实现的。

AQS包含一个state状态、等待队列，其中state的修改是通过CAS实现的。ReentrantLock中，state表示锁重入次数，Semaphore表示信号量，CountDownLatch中表示计数器。

## 1. 流程

尝试CAS修改state。

成功获取锁。

失败进入等待队列。

阻塞当前线程。

前驱节点释放后唤醒当前线程。

再次尝试获取锁。

# 20. CAS

CAS是比较与交换，无锁的方式实现锁。

首先读取资源的值，修改了资源后，查看当前资源值，如果与先前读取的一致，认为当时没有线程访问资源，可以直接修改资源。如果有，就重新读取，修改资源，查看当前资源值，通过自旋来重复修改。

缺点是高竞争下自旋消耗CPU，且只能保证单个变量的原子操作，还有ABA问题。但ABA问题通过版本控制可以解决。

# 21. ThreadLocal

ThreadLocal用于给每个线程保存独立的变量。

可以在线程中创建ThreadLocal对象并通过set设置数据，那么线程的所有方法都可以通过get来访问数据。主要用于保存用户上下文、数据库连接等。

每个线程内都有一个ThreadLocalMap成员，以ThreadLocal为key，值作为value。

但ThreadLocalMap的key是弱引用，而value是强引用。如果ThreadLocal对象被回收，key变成null，但value可能会继续持有。所以使用后需要通过threadLocal.remove()来释放内存。

# 22. ConcurrentHashMap

这是线程安全的高性能Map。

JDK1.7之前，底层通过Segment分段来实现线程安全。线程访问Map的某个数据时，将这个数据所属的段给锁住，其他线程不可访问这个段。而JDK1.8之后，就取消了分段，而是直接锁住数组桶的头节点。如果数据位置为空，通过CAS来插入数据。如果不为空，就通过synchronized来锁住头节点。

# 23. BlockingQueue

阻塞队列，生产者可以往里面放数据，如果满了就会阻塞，直到消费者取了数据后，有位置时才会唤醒生产者继续放数据。

# 24. CoundDownLatch

CountDownLatch主要用于线程等待。

一个线程需要等待多个线程执行完毕才能继续，那么可以使用CountDownLatch。

```java
CountDownLatch latch = new CountDownLatch(3);

for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        try {
            // 执行任务
        } finally {
            latch.countDown();
        }
    }).start();
}

latch.await();
System.out.println("所有任务完成");
```

通过await来等待，通过countDown来减少计数。

# 25. Semaphore

信号量，用于控制资源的访问数量。

如果一个资源最多可以让n个线程同时访问，那么可以将信号量设为n，线程访问时通过aquire获取信号量，处理完毕通过release来释放信号量。

主要用于限流、控制数据库连接数等。

# 26. JMM

JMM用来解决多线程共享变量的访问规则。

主要解决的问题是原子性、可见性、有序性。

可见性可以使用volatile、synchronized、lock都可以保证。

有序性可以通过volatile实现，或者使用happens-before实现。

动作Ahappens-before动作B，能够保证动作A在动作B之前运行。

可以在动作A完成后修改一个变量，动作B检测到变量修改后才执行。

# 27. 线上问题

如果线上出现问题，需要排查可能的问题。

1. 接口变慢。

这种情况需要查看监控，看CPU是否升高、GC频率、线程数、数据库连接池、网络问题等，可以定位到具体的线程。

通过jstack能够看线程栈，看是否大量线程处于BLOCKED、WAITING、RUNNABLE状态。

如果是BLOCKED，可能出现大量锁竞争，如果大量WAITING，可能线程池的线程数不足。如果大量TIMED_WAITING，可能出现大量的sleep或者超时等待。如果出现RUNNABLE，说明当前执行大量CPU密集型任务或死循环。

2. 内存上涨。

通过jstat查看老年代是否满。如果满了，可能出现内存泄露的情况。需要导出堆信息，看谁占用的内存最大。

3. CPU100%。

通过top查看哪个进程CPU占用100%，找到对应的线程id，用jstack打印栈信息，如果一直在执行某个业务中的代码代码，那么当前代码是热点，或者出现循环。

4. 死锁。

如果接口卡死，或者任务不推进，需要考虑死锁问题。jstack打印堆栈，如果一致停留在当前代码，可能就是出现了死锁。

