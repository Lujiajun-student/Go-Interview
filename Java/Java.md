# 1. 基本数据类型

Java有8种基本数据类型。

* 数字型。四种整数型：`byte`, `short`, `int`, `long`，2种浮点型：`float`, `double`。
* 字符型。`char`。
* 布尔型。`boolean`。

# 2. 基本类型和包装类型的区别

除了定义常量和局部变量外，其他地方基本都使用包装类型。基本数据类型的局部变量存放到栈的局部变量表中，成员变量会存放到堆中，而包装类型属于对象，对象实例会存放到对中。

包装类型如果不赋值，默认为null。基本数据类型会进行初始化，不为null。如果要比较的话，基本数据类型使用`==`来比较，包装类型使用`equals()`方法来比较。

# 3. 包装类型的缓存机制

整数的包装类，Byte, Short, Integer, Long创建[-127, 128]的缓存数据，Character创建了[0, 127]范围的缓存数据，Boolean只需缓存True或False。这样能够将经常使用的数事先加载出来，避免频繁地创建对象。

# 4. 自动装箱与拆箱

装箱是将基本类型用引用类型包装起来，拆箱是将包装类型转换为基本类型。

将基本数据类型赋值给包装类型时，就会将这个基本类型自动装箱。

将包装类型赋值给基本数据类型是，就会将包装类自动拆箱。

```java
Integer i = 10; // 装箱
int n = i; // 拆箱
```

# 5. 浮点数运算的精度丢失

计算机保存数据是，使用二进制数来保存。而十进制下很多小数没办法转换为二进制小数，比如0.2转化到二进制数中是循环小数，遵循IEEE 754 双精度标准来截断小数。这样的话，浮点数运算时，用两个被截断的浮点数来进行运算，就会出现精度丢失的现象。较好的方法是使用Big Decimal，通过字符串来运算能够避免精度丢失的问题。还有就是结果比较的时候添加误差范围，如果两个数的差异小于误差，就判断这两个数是相等的。但这种方法不太精确。

# 6. 超大数据的存储

基本数据类型都会有一定的范围。像64位的long整型也可能会出现数据溢出的情况，导致从边界值跳转为另一个边界值的情况。这样就可以使用BigInteger来保存。它的内部使用int[]数组来存储任意大小的整型数据。这种运算的效率会比较低。

# 7. 字符型常量和字符串常量的区别

字符常量是一个字符，字符串常量是若干个字符组成的常量。

字符常量能够根据ASCII码作为整型值使用，参与表达式运算。但字符串常量代表一个地址。

字符常量在Java中占用2字节，字符串常量的空间与字符数有关。

# 8. 静态方法能否调用非静态成员

静态方法不能调用非静态成员，因为静态方法属于类，在类加载的时候就会分配内存，通过类名能够直接访问。但非静态成员属于实例对象的属性，需要先实例化对象才能使用，通过类的实例来访问。

# 9. 静态方法与实例方法的区别

静态方法可以通过类名来调用，也可以通过实例对象来调用，但不推荐，容易与实例方法混淆。

静态方法由于不需要实例化对象，所以只能调用静态成员，不能使用非静态成员。

# 10. 重载和重写的区别

重载是一个方法根据输入的不同，作出不同的处理。重写就是子类继承父类的相同方法时，输入数据一样但需要不同的响应，就要修改这个函数的内容来重写父类方法。

# 11. 可变长参数

可变长参数就是允许调用方法时传入不定长度的参数。

```java
public static void method1(String ...args){
    for (String s : args) {
        System.out.println(s);
    }
}
```

传入后，不定长的参数会被转化为数组。

# 12. 面向对象和面向过程的区别

面向过程POP是将解决问题的过程拆为一个个方法，通过执行方法来解决问题。

面向对象OOP是先抽象出对象，调用对象执行的方法来解决问题。

# 13. 对象相等与引用相等的区别

对象相等指的是对象的内容是否相等，对象类需要重写`equals()`方法。/

引用相等指的是对象的内存地址是否想等，使用`==`来比较。

# 14. 面向对象

## 14.1 特性

### 14.1.1 封装

封装指的是将对象的属性与行为绑定到一个类中，通过访问控制修饰符来对外隐藏内部细节，外部使用暴露出来的公共接口来操作对象。

### 14.1.2 继承

继承允许子类从父类中获取属性和方法，实现代码复用。子类可以通过方法重写来修改父类的实现，也可以通过方法重载来拓展功能。

### 14.1.3 多态

多态是指同一个方法可以作用于不同类型的对象，并产生不同的行为。也就是一个接口，多种实现。运行时多态遵循里氏替换原则，父类引用可以指向任意子类对象，程序的行为不会受到影响。

## 14.2 基本原则

1. 单一职责原则。类的功能需要单一，不能包含多个职责。
2. 开放封闭原则。需求变化时应该添加新的代码，而不是修改原有的代码。
3. 里氏替换原则。子类可以替换父类，而且功能不受影响。
4. 依赖倒置原则。高层次的模块不应该依赖于底层次的模块，应该都依赖于抽象。
5. 接口分离原则。设计多个接口实现多个行为比设计通用接口更好。

# 15. 接口和抽象类的共同点和区别

接口和抽象类有两个共同点。

* 实例化。接口和抽象类都不能直接实例化，只能被实现或继承后才能创建具体的对象。
* 抽象方法。接口和抽象类都可以包含抽象方法。抽象方法没有方法体，需要在子类或实现类中实现。

接口和抽象类有四个区别。

* 目的。接口主要用于对类的行为进行约束，实现某个接口就有对应的行为。抽象类用于代码复用，强调的所属关系（如Person和Student的关系）。
* 继承和实现。一个类只能继承一个类，但一个类可以实现多个接口，一个接口也可以继承多个其他接口。
* 成员变量。接口的成员变量必须用public static final修饰，不能修改且必须有初始值。这是因为接口的目的是行为，行为本身没有状态，应该由实现类来给出状态。而抽象类的成员变量则没有限制，可以在子类中重新定义或者赋值。

从Java 8 开始，接口可以有默认的方法实现，也可以有属于接口本身的静态方法。

# 16. 深拷贝和浅拷贝

* 浅拷贝会在堆中创建新的对象，如果原对象是基本类型，浅拷贝会复制对象的值，而引用类型会复制对象的地址，也就是浅拷贝和原对象指向同一个内存地址。
* 深拷贝也会在堆中创建新的对象，不关原对象是什么类型，都会完整地复制整个对象。

# 17. ==与equals的区别

`==`对于基本类型和引用类型的效果不同。对于基本类型来说比的是值，引用数据类型比的是内存地址。也就是说，两个对象用`==`来比较，就是判断这两个对象是否为同一个对象。而equals不能用于判断基本类型变量，只能判断两个对象是否想等。equals方法需要在对象类中进行重写，不重写的话等价于`==`，重写后就能用来比较两个对象的属性是否想等。

# 18. hashCode

## 18.1 hashCode的作用

hashCode的作用是获取对象的哈希码。哈希码的作用是确定对象在哈希表中的索引位置。

## 18.2 为什么需要hashCode

hashCode的作用是提高效率。在大量数据中查找是否存在某一个特定数据，需要调用大量的equals来进行比较，时间复杂度为O(n)。而有了hashCode，只需要计算哈希值，与对应下标的桶里的对象进行比较即可，时间复杂度能够提高到O(1)。

## 18.3 为什么重写equal时必须重写hashCode

这是维护Java对象的等价性。如果两个对象通过equals的判断是相等的，那么hashCode也必须相等。如果重写了equals没有重写hashCode，那么equals等价的两个对象可能hashCode不相等，导致map能够存入对象，但取出时key的hashCode不相等导致无法取出对象。

# 19. String、StringBuilder、StringBuffer

首先，String是不可变的。如果代码中修改了某个字符串，其实是底层在堆中创建了拼接后的字符串对象，然后指向这个新字符串。由于不可变，String是线程安全的，且可以放入字符串常量池来节省内存。

StringBuilder是用来解决String频繁拼接导致频繁创建对象引起的性能问题。StringBuilder维护一个可变的字符数组，调用append方法时在数组上进行操作。它的速度很快，修改字符串不需要创建新对象。但在多线程环境下会出现线程安全问题。

StringBuffer的行为与StringBuilder一致，但几乎所有方法都添加了synchronized关键字，也就是使用锁来保证线程安全。

# 20. String不可变

String不变可以有三个优点。

1. 实现字符串常量池。Java会把所有字符串字面量都放到堆的常量池里。像创建`String s1 = "abc"`时，在字符串常量池中会创建`"abc"`这个字面量，然后`String s2 = "abc"`时，就不会再创建对象，而是直接指向字符串常量池的`"abc"`对象即可。如果String可变，此时s1修改时，s2也会被修改，导致字符串错乱。
2. 缓存到HashCode提高性能。因为String不可变，于是hashCode只需要在创建时计算并存储即可。如果可变，那么它的hashCode也会变，导致根据字符串的hashCode存入hashMap的数据再也找不出来。
3. 安全性。字符串通常保存敏感信息，如网络连接参数、数据库的用户名或密码等。如果读取前被其他线程修改了字符串的内容，就会导致系统不可用。
4. 线程安全。多个线程读取同一个String对象不需考虑线程安全问题。

# 21. String s1 = new String("abc")中创建了几个对象

会创建1个或两个对象。

首先会查找这个"abc"在字符串常量池中是否存在，存在则直接使用，不存在就现在先在常量池创建"abc"对象。然后在堆中创建新的对象，使用常量池的"abc"来初始化。

# 22. String的intern

intern是String对象的一个native方法。调用时查看字符串常量池中是否有对应的String对象，有则返回字符串常量池中对象的引用，没有则在字符串常量池中创建对象，然后返回对象的引用。主要的作用是针对`String s = new String("a") + new String("b")`中的s，运行时生成的字符串。这样能够节省内存，避免s指向新创建的String对象，而不指向可以重复引用的字符串常量池对象。

# 23. String与基本类型的加法运算

如果String与常量进行加法运算，那么在编译阶段就会算出结果。

如果String与变量进行拼接，就会执行两个步骤。

1. 类型转换。Java会将基本类型调用类型的包装类的toString方法，将变量转换为字符串。
2. 对象拼接。获取两个字符串后，底层通过StringBuilder来进行拼接操作。Java9之后使用StringConcatFactory来动态分配字符数组长度，避免StringBuilder多次扩容的开销。

# 24. Exception和Error的区别

Exception是程序本身可以处理的异常，通过catch来捕获。Exception分为Checked Exception受检异常和Runtime Exception运行时异常。

Error是程序无法处理的错误，不建议通过catch来捕获，例如虚拟机运行错误，虚拟机内存不足错误，或者OOM等。这些错误发生时一般会终止线程。

# 25. Throwable类常用方法

`String toString()`返回异常的简要描述。

`String getMessage()`返回异常的详细信息。

`void printStackTrace()`控制台打印Throwable封装的异常信息。

# 26. Finally语句

finally语句也有可能会不执行，有四种情况。

* `System.exit(0)`。在try中直接退出JVM，finally就不会执行。
* 死锁。如果try无限等待，finally就无法执行。
* 线程死亡。
* CPU关闭。

# 27. try-with-resources和try-catch-finally

如果在try中使用需要关闭的资源，如scanner或者打开文件，使用try-catch-finally的时候需要在finally中关闭资源。但使用try-with-resources时只需要在try后的括号中定义资源即可，执行完就会自动关闭资源。

而且try-with-resources解决了异常覆盖的问题，在try-catch-finally中，如果try中抛出异常，finally也抛出异常，那么最终显示出来的只有finally的异常。而try-with-resources能够通过e.getSuppressed()来找回全部异常。

# 28. 泛型

泛型允许编写代码时不指定具体的类型，而是使用类型占位符，等到代码被使用时再决定参数类型。泛型类型通常由尖括号和大写字母来表示。

```java
public class Box<T> {
    private T data;
    public void set(T data) {this.data = data;}
    public T get() {return data;}
}
// 在使用时指定类型
Box<String> stringBox = new Box<>();
stringBox.set("Hello");// 由于使用时指定了类型，因此stringBox只能存放String类型数据。
```

## 28.1 泛型擦除

在编译过程中，Java编译器会检查泛型约束来保证类型安全。检查后就会替换为原始的类型。没有限制的泛型会替换为Object，有限制泛型`<T extends Number>`替换为边界类型Number。从泛型中取值时，会自动进行类型转换，转换的类型就是编译后替换的类型。

# 29. 反射

反射允许程序在运行状态中动态地获取类的信息并操作类或者对象。

Java中编写了java文件并编译成字节码文件后，JVM会加载到内存中，并创建`java.lang.Class`对象。这个对象记录了类名、接口、构造方法、成员变量和方法等所有信息。而反射能够在运行时判定任意一个对象的类、调用对象的方法、获取对象的成员变量和方法等。这样能够让框架更加灵活，比如String不需要知道用户实现了什么类，只需要读取配置文件或注解，通过反射将类实例化并注入到容器中。

# 30. 代理

代理允许在不修改源代码的情况下，对一个类或对象的方法进行功能增强。

代理主要有两种，分别为静态代理和动态代理。

静态代理由程序员手动创建并编译。代理类内部有对目标对象的引用。但这种方法需要为所有需要代理的类手动创建代理类，维护成本高。

动态代理在程序运行时，通过反射等机制动态生成的对象。

Java中主要有两种动态代理。

1. JDK动态代理。利用反射来生成一个实现了接口的类的代理对象。这种代理要求目标类至少实现一个接口。
2. CGLib类。通过修改字节码来生成目标类的子类。这种代理不需要实现接口也能代理，但由于通过继承来实现代理，无法代理final修饰的类或方法。

# 31. 序列化和反序列化

序列化是将数据结构或对象转换成可以存储或传输的形式，通常是二进制字节流，也可以是JSON，XML等文本格式。

反序列化是将序列化之后的数据转换为原始数据结构或对象。

如果不希望变量被序列化，需要使用`transient`来修饰。但结果就是反序列化后，变量会被初始化为默认值。

# 32. IO流

IO流是数据输入到计算机以及数据输出到外部的过程类似水流，所以称为IO流。

为什么要分为字节流和字符流？因为字节流搬运的是每个单位的字节，如果不清楚编码类型，如中文占3个字节，会导致乱码的问题。而字符流会根据字符集将多个字节拼接为完整的字符，保证读取的数据是完整可读的。

# 33. ArrayList和Array的区别

Array数组的大小固定，一旦创建就不可改变；数据类型支持基本类型和对象，不具备动态添加和删除数据的能力。

而ArrayList会动态扩容或缩容，使用泛型来保证类型安全。但不能存储基本类型，只能存储对象，基本类型需要使用包装类；ArrayList提供add、remove的方法来动态添加和删除数据。

ArrayList初始化之后，默认的容量是0，空数组。只有调用add添加元素是，才会初始化数组，默认容量是10。如果数组已满，仍要添加元素是，就会触发扩容。源码中，ArrayList维护的容量会采用位运算，将容量变为原来的1.5倍。而删除元素后，ArrayList不会自动缩容，需要调用trimToSize()来将容量调整为当前元素个数，也就是缩容后，当前数组满元素。

# 34. Comparable和Comparator的区别

Comparable在`java.lang`包下，如果一个类实现了Comparable接口，那么这个类的对象有排序能力。方法为`public int compareTo(T o)`。这种方式修改类的结构，一个类只能有一种compareTo的实现。调用方法为`obj1.compareTo(obj2)`，这样类的两个对象就有了比较的能力，调用Collection.sort就能进行排序。

Comparator在`java.utils`包下，是独立的排序工具，可以定义不同的Comparator来满足不同的排序要求。

```java
// 定义一个按名字排序的外部比较器
Comparator<Student> nameComparator = (s1, s2) -> s1.getName().compareTo(s2.getName());

// 在初始化 TreeSet 时传入该比较器
Set<Student> set = new TreeSet<>(nameComparator);
```

# 35. HashSet,LinkedHashSet和TreeSet

这三个都是Set的实现类，都保证元素唯一，都是线程不安全的。

HashSet底层结构是哈希表，LinkedHashSet是链表和哈希表，数据的插入和取出顺序满足FIFO。TreeSet是红黑树，元素有序。

所以HashSet适合不关注顺序的场景，LinkedHashSet适合FIFO存取数据的场景，TreeSet适合需要排序的场景。

# 36. Queue和Deque的区别

Queue是单向队列，只能从一端插入元素，另一端删除元素，遵循先进先出的原则。Queue扩展了Collection的接口。

Deque是双端队列，在队列的两端都可以插入或删除元素。可以选择遵循FIFO或者LIFO原则。Deque扩展的是Queue的接口，增加了在队首和队尾插入和删除的方法。还提供了push和pop

# 37. ArrayDeque和LinkedList的区别

ArrayDeque和LinkedList都实现了Deque接口，都可以作为栈或队列来使用。

但ArrayDeque是循环数组，基于可变长的数组和双指针来实现，不暴露索引接口导致不可随机访问，不支持存储null值。

LinkedList通过链表来实现，可以存储null值，每次插入数据时都需要申请新的堆空间来创建结点。

# 38. HashMap和HashTable的区别

HashMap是非线程安全的，HashTable是线程安全的，因为HashTable内部的方法都经过synchronized修饰。

由于线程问题，HashTable的性能较低，且已经废弃，线程安全需要使用ConcurrentHashMap。

HashMap可以存储null的key和value，但key为null只能出现一次，而HashTable不允许有null存在。

HashMap默认大小为16，扩容后容量变为2倍。HashTable默认大小为11， 扩容后变为2倍+1。

# 39. HashMap和HashSet的区别

HashSet是基于HashMap实现的。

HashMap实现了Map接口，用来存储键值对，调用put来添加元素，使用key来计算哈希值并放到对应的桶中。

HashSet实现Set接口，仅存储对象，调用add方法来向Set中添加元素。计算哈希值是用对象本身来计算哈希值，去重时先用hashCode比较，然后用equals再比较，因为不同对象的哈希值可能会相同，需要通过equals来比较内容是否相同。

*Set的去重逻辑是内容相同，不是同一个对象，因此不能使用`==`来比较*。例如`new String ('a')`和`new String ('a')`是两个对象，但内容相同，Set不允许这两个对象进入同一个set中。

# 40. HashMap和TreeMap的区别

HashMap底层是数组+链表+红黑树，无序，不保证插入和遍历顺序，增删查的时间复杂度为O(1)。Key比较时使用HashCode和equals来比较，有数组扩容、树化、退化逻辑。

TreeMap底层是红黑树，能够按key排序，增删查时间复杂度为O(logN)。key比较时使用Comparable，无法扩容，只用来存储数据。

但TreeMap实现了NavigableMap，有搜索能力。可以使用ceilingEntry(), floorEntry(), heigherEntry(), lowerEntry()来寻找大于等于、小于等于、严格大于、严格小于给定key的最近键值对，还可以有自己操作，创建原集合的子集视图等。

# 41. HashSet检查重复逻辑

HashSet的add方法是将元素作为key，调用HashMap的put方法，并判断返回值来确保是否有重复。而HashMap判断重复的逻辑是先算key的哈希值，然后比较hashCode，最后比equals。

# 42. HashMap的实现

JDK 1.7时，HashMap底层是数组和链表结合使用的链表散列。如果出现哈希冲突，就使用拉链法解决冲突。

JDK 1.8之后，HashMap添加了红黑树。如果链表长度大于8，且数组长度小于64，就会转换成红黑树，将链表的查询时间缩短到O(logN)。而数组桶元素小于6时，就会从红黑树转换为链表。

# 43. HashMap的长度为什么是2的幂次方

HashMap定位元素时，会先计算HashCode，然后取模运算，来获取在数组中的下标。但如果除数是2的幂次方，就能够等价于哈希值对除数-1的与操作。而与操作能够大幅提高效率。

另外HashMap扩容后，计算HashMap的元素在新数组的位置时，如果数组长度保持为2的幂次方，扩容会更加均匀。扩容后需要计算原数组中所有元素在新数组中的位置。而计算了新数组位置后，如果高位为0，表示当前元素在数组中的位置不变，而高位为1，表示当前元素在扩容后的位置。

# 44. HashMap多线程死循环问题

JDK 1.7 版本之前，HashMap扩容时采用头插法。但如果多个线程同时对链表进行扩容，会导致链表的节点指向错误的位置，形成环形链表，使得查询操作陷入死循环。

为了解决这个问题，JDK1.8之后采用尾插法来放入元素，避免链表倒置，元素都是在链表末尾添加，不会出现环形结构。

但多线程环境下不建议使用HashMap，应该使用ConcurrentHashMap。

# 45. HashMap多线程问题

HashMap在多线程环境下除了JDK1.7之前的死循环问题，还有数据覆盖问题，这个问题在所有版本中都存在。主要原因是可能存在两个线程同时进行put操作，如果线程1和线程2都要在一个桶中存入数据，线程1发现没有冲突然后挂起，此时线程2完成插入，再到线程1时就不会再次判断是否冲突，而是直接存入桶，导致线程2的数据被覆盖。

而且如果线程1存入数据后挂起，线程2存入数据后map的size自增，回到线程1时再完成自增，此时一共存了两个数据，但size只自增了一次，线程2的size没有被保存。

# 46. HashMap的遍历方式

hashMap主要有六种遍历方式。

1. entrySet。通过增强for循环来获取map的每一个元素，且需要调用map.get来获取key。效率最高。

```java 
for (Map.Entry<Object, Object> entry : map.entrySet()) {
    System.out.println(entry.getKey() + "=" + entry.getValue());
}
```

2. keySet。通过遍历key集合来获取所有key元素。效率较低，获取key后还需要再进行get操作。

```java
for (Object key : map.keySet()) {
    System.out.println(key + "=" + map.get(key));
}
```

3. values。只取值，不获取key。

```java
for (Object val : map.values()) {
    System.out.println(val);
}
```

4. Iterator迭代器。在迭代器中可以删除元素。

```java
Iterator<Map.Entry<Object, Object>> it = map.entrySet().iterator();

Object target = "b";
while (it.hasNext()) {
    Map.Entry<Object, Object> entry = it.next();
    if (target.equals(entry.getKey())) {
        it.remove();
    }
}
```

5. Lambda forEach。代码简单。

```java
map.forEach((key, value) -> {
    System.out.println(key + "=" + value);
})
```

6. Stream流。

```java
map.entrySet().stream().forEach(entry -> 
	System.out.println(entry.getKey() + "=" + entry.getValue())
;)
```

# 47. ConcurrentHashMap和HashTable的区别

HashTable采用的是数组+链表的形式，而ConcurrenthashMap在1.7之前用分段的数组+链表实现，1.8之后也是用数组+链表+红黑树实现。

在JDK1.7时，ConcurrentHashMap对桶数组进行了分段，线程访问时只会锁住需要的段Segment，多线程访问不同数据段的数据就不会有锁竞争，提高了并发率。

JDK1.8后放弃了分段，而是使用节点数组+链表+红黑树来保存数据，并发控制使用的是synchronized关键字和CAS来操作。读操作不会上锁，写操作的时候只会锁当前数组桶，不会影响其他桶的操作。

HashTable本质上使用的是synchronized来保证线程安全，任何时刻只能有一个线程访问map，性能低下。

# 48. Collections工具类

Collections主要的用法是排序、查找和替换操作。如Collections.sort()能够对链表进行排序，或者窜如比较器来实现自定义排序，也可以反转、随机打乱等。

# 49. 进程和线程

进程是资源分配的最小单位，是程序的运行动态实例，运行程序就是一个进程从创建，运行到死亡的过程。

线程是CPU调度的最小单位，进程可以包含多个线程，多个线程共享进程的堆和方法区，每个线程有自己独立的程序计数器、虚拟机栈和本地方法栈。系统切换线程的开销比进程切换要小的多。

# 50. Java线程和系统线程的区别

线程主要有两类，第一类是用户县城，在用户空间中管理和调度的线程，还有内核线程，由操作系统管理和调度的线程。

在JDK1.2之前，Java使用的是绿色线程。也就是多个Java线程对应1个操作系统线程。这样线程的管理交给了JVM实现，线程切换在用户态完成，上下文切换快，但无法利用多核CPU，这多个Java线程只能使用1个CPU核心，无法并行。

JDK1.2之后，Java线程改用操作系统的线程，每个Java线程对应一个操作系统线程，线程的调度就交给了操作系统内核。不同的Java线程就可以分配不同的CPU核心，实现并行，而且一个线程堵塞不会影响其他线程的运行。但线程的创建、销毁和管理都需要内核态和用户态的切换，性能开销高。

# 51. 线程的程序计数器、虚拟机栈和本地方法栈

JVM中，程序计数器、虚拟机栈和本地方法栈是线程私有的，主要用于保证线程执行的独立性。

程序计数器记录当前线程执行的指令地址，每个线程都需要指向各自的指令，因此不能共享。虚拟机栈负责方法的入栈和出栈，不同线程的方法帧需要独立才能保证函数调用的正确性，而本地方法栈也一样，负责管理线程使用的native方法的入栈和出栈。而堆和方法区可以共享，实现线程的数据交换和资源复用。但多线程环境下也带来了线程安全问题。

# 52. 生命周期和状态

Java线程在运行的生命周期只会处于6种状态的其中一种。Java采用Thread.State枚举来定义了6种状态。

1. NEW。初始状态，线程被创建出来，还没被调用。
2. RUNNABLE。运行状态，线程被调用了，正在运行或者等待时间片，包含Ready和Running两个状态。
3. BLOCKED。阻塞状态，等待锁释放。
4. WAITING。等待状态，线程处于无限期等待，需要其他线程完成特定行为后才会被唤醒。
5. TIME_WAITING。超时等待状态，等待特定时间后自动唤醒。
6. TERMINATED。终止状态，线程运行完毕。

# 53. 线程上下文切换

线程执行中有自己的运行条件和状态，如果CPU时间片用完，或者调用阻塞操作，锁竞争，或者结束，线程都会从占用CPU的状态中退出。

首先是会挂起线程，然后保存上下文，如程序计数器、栈指针等，然后更新线程状态，从RUNNING变为READY、BLOCKED或者TERMINATED，再根据算法从就绪队列中选取新线程，加载新线程的上下文之后就可以执行。

# 54. Thread.sleep()方法和Object.wait()方法的区别

这两个方法都可以用来暂停线程。

但sleep方法不会释放锁，使用的目的是暂停执行。而且sleep方法执行完毕后，线程会自动苏醒。

wait方法会释放锁，使用的目的是用于线程的交互通信，如消费者线程等待信息。wait方法被调用后，线程不会自动苏醒，而是需要别的线程调用notify或者notifyAll来手动唤醒线程。

# 55. 为什么sleep在Thread中，wait在Object中

wait是让获取对象锁的线程等待，需要释放当前线程占用的对象锁。每个对象都有对象锁，那么需要释放锁的话，就是需要操作对象来实现的，所以wait定义在Object中。

sleep的目的是暂停当前的线程，不涉及对象，需要保持锁，因此就定义在Thread中。

# 56. Thread的start和run区别

start会启动一个线程并进入就绪状态，分配到时间片后就会运行，与主进程并发执行。而run方法只会作为一个普通方法，在函数中按顺序执行，不是多线程。

# 57. 线程死锁

线程死锁是指多个线程同时被阻塞，等待某个被其他阻塞线程持有的资源被释放，而陷入无限期等待的情况。

死锁产生有四个条件。

1. 互斥条件。资源独占，一段时间内只能由一个线程占用，其他线程此时请求只能等待。
2. 请求与保持。线程持有资源，但又请求新的资源，而新的资源被其他线程占有。这样线程就会阻塞，但保持对已经有的资源的占有状态。
3. 不可剥夺条件。线程获取的资源未使用完之前，不能被其他线程抢占，只能由线程自己释放。
4. 循环等待条件。存在线程资源的请求循环链。也就是线程A等待线程B占有的资源，线程b等待线程C占有的资源，线程C等待线程A占有的资源。

如果要检测死锁，可以使用jmap，jstack来查看线程栈和堆内存情况。

# 58. 避免死锁

预防死锁需要破坏死锁产生的必要条件。

1. 破坏请求与保持条件。要求线程一次性申请所有资源。
2. 破坏不剥夺条件。线程如果请求其他资源时需要阻塞，可以释放它战友的资源。
3. 破坏循环等待条件。按顺序来申请资源，释放资源反序释放。

避免死锁就是在资源分配时，根据算法对资源分配过程进行评估，保持安全状态就行。一般使用银行家算法。

# 59. Volatile

多线程环境下，所有线程共享主内存，但每个线程都会有自己的工作内存作为缓存。这样会存在可见性问题。线程A修改变量的值，会先写到工作内存，然后刷新到主内存。但线程B有可能事先获取了这个变量，保存到工作内存中，导致线程一致读取工作内存中的变量值，主内存的修改对它不可见。

这样，就需要使用volatile修饰变量来保证变量的可变性。变量被声明为volatile后，会通过两个规则保证变量的可见性。

1. 写操作强制刷新。某个线程修改了volatile变量，就会立刻刷新到主内存。
2. 读操作失效。线程读取volatile变量时，只能从主内存中读取。

## 59.1 指令重排序

为了提高性能，编译器通常会对指令进行重新排序，结果不会收到影响。但多线程环境下，重排序会出现问题。如初始化变量时，可能是先分配内存，初始化对象，然后将对象指向分配的内存地址，也可能是先分配内存，将对象指向内存地址，再初始化对象。这样如果在初始化对象之前，其他线程获取了对象，会出现空指针或者逻辑错误。

因此，使用volatile会在代码生成的字节码中插入内存屏障来禁止指令重排。在写操作之后插入StoreLoad屏障，在读操作之前插入LoadLoad或者LoadStore屏障，使得墙左边的操作必须在墙右边的操作之前完成。

| 内存屏障类型 | 示例                       | 说明                                                         |
| ------------ | -------------------------- | ------------------------------------------------------------ |
| LoadLoad     | Load1; LoadLoad; Load2     | 保证Load1数据在Load2之前装载。也就是读完第一个数据之前，不能读第二个数据。 |
| StoreStore   | Store1; StoreStore; Store2 | 确保Store的数据完成写操作后，才能进行第二个写操作。          |
| LoadStore    | Load1; LoadStore; Store2   | 读操作完成之前，不允许写操作。                               |
| StoreLoad    | Store1; StoreLoad; Load2   | 写操作完成之前，不允许读操作。                               |

由于写操作一般更持久，StoreLoad开销最大，但效果最好。

volatile写操作之前会插入StoreStore，后面插入StoreLoad；volatile读操作之后插入LoadLoad和LoadStore操作。

# 60. volatile的非原子性

volatile解决的是数据的可见性，保证数据修改对所有线程可见，但无法保证数据操作的原子性。如果要实现原子性，需要使用synchronized、AtomicInteger或者Reentrantlock来加锁。

# 61. 乐观锁和悲观锁

* 悲观锁总是认为资源每次访问都会出现问题，因此每次获取资源都会进行上锁，其他线程需要阻塞，等待资源释放后被唤醒。向Synchronized和ReentrantLock就是悲观锁思想。悲观锁能够保证资源的线程安全，但多线程环境下会出现频繁阻塞导致的性能问题和死锁问题。
* 乐观锁总是认为资源每次访问不会出现问题，不需要加锁也不需要等待，只需要修改时查看资源是否被修改了。如AtomicInteger、LongAdder使用乐观锁的CAS实现的。

# 62. 乐观锁实现机制

## 62.1 版本号机制

在数据表中添加version字段，表示数据被修改的次数。每修改一次，version就会加1。线程A要修改数据的时候读取version值，提交更新时再读取version，只有读取前后的version值相同才能更新，否则重试更新操作，只要读取前后的version相同，才会更新。

## 62.2 CAS算法

CAS是比较与交换。用一个预期值和要更新的变量值比较，两者相等才会更新。CAS属于原子操作。

CAS包含三个操作数，分别是要更新的变量值V，预期值E，写入的新值N。仅当V=E时，CAS通过原子方式用N来更新V。如果不等，就放弃更新。

但是，CAS存在ABA问题。如果一个变量V初次读取是A，然后被其他线程修改为B，再修改为A，CAS准备更新时发现变量依然是A，误以为没有被修改过。解决方法是在变量前面添加版本号或者时间戳。

# 63. Synchronized

synchronized是Java的关键字，用来解决多线程之间访问资源的同步性，保证被修饰的方法或者代码块同一时间只有一个线程执行。

Java早期属于重量级锁，锁依赖操作系统的Mutex实现，如果要挂起或唤醒线程，都需要切换到内核态，时间较高。

从Java6开始，synchronized引入了自旋锁、锁消除、锁粗化、偏向锁、轻量级锁、锁升级等技术减少锁操作的开销。

## 63.1 锁升级

1. 无锁。在一开始，加锁的变量处于无锁状态。

2. 偏向锁。一个线程获取了锁之后，可以继续多次重复获取这个锁。
3. 轻量级锁。采用CAS来尝试获取锁。
4. 重量级锁。自旋多次失败，线程进入阻塞状态。

## 63.2 自适应自旋锁

轻量级锁竞争失败时，JVM会让线程进行循环，期望循环期间持有锁的线程能够释放锁。自适应是由上一次锁的自旋时间来确定。如果上一次自旋成功，JVM允许自旋更久；如果自旋很少成功，JVM可能会跳过自旋。

## 63.3 锁消除

这是基于JIT的编译优化。编译器如果发现有锁的对象只会被单个线程访问，就会认为当前锁是多余的，自动消除内部的锁。

## 63.4 锁粗化

一般来说，同步代码的范围越小越好。但如果连续操作都要对同一个对象进行加锁、解锁，或者循环体中有锁，那么这些操作会带来性能损耗。JVM会探测到这种连续的锁，将锁的范围拓展到整个操作中。

# 64. Synchronized原理

对同步代码块来说，每个对象都有关联的监视器monitor。指令执行时，也就是同步代码块开始时，通过monitorenter来获取对象monitor的所有权。代码块结束后通过monitorexit来释放所有权。而monitorenter之后会存在两个monitorexit指令，为了保证锁在同步代码块执行成功与否都会释放锁。

对于同步方法来说，synchronized会把当前方法通过ACC_SYNCHRONIZED来标识为同步方法。如果是实例方法，运行时会获取这个实例对象的锁。如果是静态方法，JVM会获取当前class的锁。

# 65. ReentrantLock

ReentrantLock实现了Lock接口，底层由AQS实现，提供了许多功能。

1. 显式加锁和锁释放。需要手动调用`lock()`和`unlock()`来进行锁操作。
2. 公平锁与非公平锁。默认是非公平锁，新来的线程尝试获取锁，获取不到就排队。非公平锁是新线程不会获取锁，而是直接进入队列等待。
3. 响应中断。synchronized进入阻塞就无法打断，但ReentrantLock可以通过interrupt()来打断阻塞。
4. 尝试获取锁。通过tryLock来尝试获取，获取不到立刻返回false而不是等待。

# 66. AQS

AQS是抽象队列同步器，位于java.util.concurrent.locks包中。

AQS提供了一个框架，用来实现多种同步器，如可重入锁、信号量和倒计时器。AQS将底层的同步机制封装起来，开发者只需要专注于同步逻辑即可。

主要的思想是如果被请求的共享资源空闲，那么就将当前请求资源的线程设置为有效地工作线程，将共享资源设置为锁定状态。如果被请求的共享资源被占用，那么就需要一套线程等待和被唤醒时锁分配的机制。

AQS主要包含三个部分。

1. 状态位volatile int state。代表了资源的状态。ReentrantLock中，state为0表示锁空闲，大于0表示锁被占用。Semaphore中，state表示可用的信号量数，CountDownLatch中，state表示还需要倒计时的次数。
2. CAS操作。AQS使用Unsafe类的CAS操作来原子性修改state的值。
3. CLH双向队列。线程获取资源失败时，AQS会将线程封装成一个Node节点放入双向FIFO队列中等待。

CLH队列是一个先进先出的队列，如果线程竞争资源失败，AQS就会将线程包装成Node节点，放入队列并等待。

AQS定义了两种资源共享方式。第一种是Exclusive独占方式，只有一个线程能够获取资源，如ReentrantLock，这又可以分为公平锁或者非公平锁。第二种是Share共享方式，多个线程可以同时执行，如Semaphore信号量、ReadWriteLock读写锁等。

## 66.1 工作流程

在独占模式中，流程有四个。

1. Try Acquire。线程尝试通过CAS修改state。如果成功，就获取资源。
2. Add Waiter。如果获取失败，AQS将线程包装成Node插入CLH队列尾部。
3. Acquire Queued。队列中的线程自旋观察自己的前驱结点是否为队列头节点，是则再次尝试获取，否则调用LockSupport.park()进入阻塞状态。
4. Release。持有锁的线程执行完毕，修改state，调用unpark唤醒队列中Head的下一个节点。

# 67. Synchronized和ReentrantLock的区别

1. 两个都是可重入锁。线程可以再次获取已持有资源的锁。
2. synchronized依赖JVM实现，内部逻辑不暴露。ReentrantLock在jDK层面实现，可以查看源代码。
3. ReentrantLock添加了可中断机制、公平锁、尝试获取锁超时的机制。

# 68. 可中断锁与不可中断锁的区别

区别是线程获取锁的过程被阻塞时，能否因为中断而放弃等待。

不可中断锁即使有中断信号也不会退出阻塞状态，而是继续等待。

可中断锁如果收到中断信号，会停止等待，抛出InterruptedException错误来处理。

# 69. ThreadLocal

一些变量不需要对其他线程可见，只需要在线程内部创建和使用，不存在安全问题，就可以使用ThreadLocal来保存变量。为了避免重复创建实例，ThreadLocal变量需要定义为static final类型。

每个Thread对象都有threadLocals成员变量，类型是ThreadLcoalMap，存放的键值对。Key是ThreadLocal本身，Value是存的值。调用threadLocal.get()时，会先获取当前线程，然后将当前线程对象作为key去查值。

## 69.1 ThreadLocal内存泄露

ThreadLocal数据存放在每个线程的ThreadLocalMap里，Map的Key是弱引用，Value是强引用。为了防止ThreadLocal本身泄露，Key改为了弱引用，避免不再引用ThreadLocal后无法被回收的问题。

但是Value依然是强引用，即使将ThreadLocal置为null，value本身依然存在，无法被回收，此时再调用threadLocal.get()就无法找到这个value。因此，使用完ThreadLocal后，需要显式调用remove()方法，将这个ThreadLocal从当前线程的Map中删除。

# 70. 线程池

线程池是管理线程的资源池。有任务处理时，直接从线程池获取线程来处理，任务结束后不会被销毁，而是放回线程池等待下一个任务。

## 70.1 使用线程池的原因

1. 降低资源消耗。线程可以重复利用，避免频繁创建和销毁带来的开销。
2. 提高响应速度。有任务时直接分配线程，省去了创建线程的时间，任务响应时间更快。
3. 提高线程的管理能力。如果频繁创建线程，可能会导致内存耗尽。线程池可以配置核心线程数、最大线程数等，能够控制并发线程的数量。

## 70.2 创建线程池方式

1. ThreadPoolExecutor构造函数创建。能够指定线程池参数，控制线程池的运行行为。
2. Executors工具类创建。Executors可以创建固定线程数量的线程池FixedThreadPool，或者只有一个线程的线程池SingleThreadExecutor等。

## 70.3 Executors弊端

FixedThreadPool和SingleThreadExecutor底层使用的队列的默认容量是Integer.MAX_VALUE，相当于无界队列。如果任务提交速度过快，会导致任务在队列中无限堆积，导致OOM。

CacheThreadPool和ScheduledThreadPool的最大线程数默认为Integer.MAX_VALUE，导致任务快会无限创建新线程，导致系统卡死或者内存溢出。

此外，Executors无法定义拒绝策略，系统负载高时默认使用AbortPolicy，直接抛出异常。而且线程无法自定义名称，导致线程名称无法对应业务。

所以最好方法是通过ThreadPoolExecutors来创建。

## 70.4 线程池参数

1. corePoolSize，核心线程数。核心线程即使没有任务，也不会被删除。
2. maximumPoolSize，最大线程数，线程池能够容纳的最大线程数量。如果核心线程忙，且队列满，就会添加额外的线程来执行，但总线程数不能超过maximumPoolSize。
3. keepAliveTime。非核心线程的存活时间。非核心线程执行完任务，等待一定时间后就会被销毁。
4. unit。时间单位，定义keepAliveTime的时间单位。
5. workQueue。任务队列，如果核心线程忙，就会将任务放到阻塞队列，满了才会让非核心线程执行。
6. threadFactory。线程工厂，用来创建新线程。
7. handler。拒绝策略，如果线程池和队列都满了，需要策略来处理额外任务。

## 70.5 拒绝策略

* AbortPolicy。直接抛出异常。
* CallerRunsPolicy。让提交任务的线程执行任务。但提交任务的线程通常为主线程，让主线程阻塞执行有可能影响程序的正常运行。
* DiscardOldestPolicy。丢弃队列最早的任务，新的任务入队。
* DiscardPolicy。直接丢弃任务，不报错。

## 70.6 阻塞队列

线程池可以选择多种阻塞队列。

* LinkedBlockingQueue。无界阻塞队列，包含FixedThreadPool和SingleThreadExecutor。两者的队列都永远不会满。
* SynchronousQueue同步队列，包含CachedThreadPool。没有容量，不存储元素，如果有空闲进程就使用空闲进程处理，否则创建新线程来处理。对最大线程数无限制，可能会创建大量线程。
* DelayedWorkQueue延迟队列，包含ScheduledThreadPool和SingleThreadScheduledExecutor。按延迟时间的长短对任务排序，保证出队的任务是当前队列中执行时间最早的。以及延迟队列会自动扩容，最大线程数对其无效，可能导致队列存储大量任务。
* ArrayBlockingQueue。有界阻塞队列，容量固定。

## 70.7 线程池处理流程

1. 如果当前运行的线程数小于核心线程数，就会创建线程执行任务。
2. 如果线程数大于核心线程数，就会尝试将当前任务放到任务队列中。
3. 如果任务队列已满，就会尝试创建新线程来执行任务。
4. 如果达到了最大线程数，那么任务就会被拒绝，调用拒绝策略的方法来处理。

## 70.8 线程异常处理

如果任务通过execute提交到线程池抛出异常，异常会被打印到控制台或者日志文件，然后创建新线程来替换旧线程。

如果任务通过submit提交到线程池，发生异常时不会打印出来，异常会封装到submit返回的Future对象，调用Future.get()方法能够捕获ExecutionException。

# 71. Future

Future用于一些需要执行耗时任务的场景，避免程序一直原地等待。耗时任务完成后，数据就会写入到Future类，程序就可以通过Future来获取数据。

Future主要有四个功能。

1. 取消任务`cancel(boolean myInterrupt)`。
2. 判断任务是否被取消`isCancelled()`。
3. 判断任务是否执行完成`isDone()`。
4. 获取任务执行结果`get()`。

主要流程就是主线程调用executor.submit(Callable)，线程池直接返回Future实现类，如FutureTask。此时没有获取到结果。工作线程能够不等待，直接暂时获取Future类，在后台执行其他任务。在需要结果时，调用future.get()来获取结果。

虽然这解决了异步获取结果的问题，但get方法依然是阻塞的，如果不想阻塞需要轮询isDone()。而且无法进行链式调用，即获取到结果后自动把结果交给其他任务。

# 72. 一个任务在其他任务执行后执行的设计

假设任务C必须在任务A和任务B完成后才能开始，最好的方案是使用CompletableFuture。

```java
CompletableFuture<String> taskA = CompletableFuture.supplyAsync(() -> {
    return "result A";
});
CompletableFuture<String> taskB = CompletableFuture.supplyAsync(() -> {
    return "结果B";
});
CompletableFuture<Void> combinedFuture = CompletableFuture.allOf(taskA, taskB);
combinedFuture.thenRun(() -> {
    try {
        String resultA = taskA.get();
        String resultB = taskB.get();
        System.out.println("get " + resultA + " and " + resultB);
        taskC();
    } catch (Exception e) {
        e.printStackTrace();
    }
})
```

通过CompletableFuture.allOf()来包含所有需要的任务，然后通过completableFuture.thenRun()来实现执行完所有需要的任务后才执行里面的代码块。

# 73. semaphore

semaphore是信号量，与synchronized和reentrantLock只允许一个线程访问某个资源不同，信号量用来控制同时访问资源的线程数量。假设信号量为N，线程想要访问资源，就需要通过acquire来获取许可，信号量变为N-1，执行完毕后通过release来释放许可，信号量变为N。如果信号量为0，线程需要阻塞等待。如果semaphore设为1，就会退化为不可重入的互斥锁。

# 74. CountDownLatch

CountDownLatch允许count个线程阻塞在同一个地方，直到所有线程的任务都执行完毕。主线程调用await时会阻塞自己，等待计数器归零，子线程执行任务后，调用countDown来减少state的值，直到state为0，主线程就会被唤醒。

此外，CountDownLatch也可以实现同时开始任务。首先初始化CountDownLatch为1，多个线程调用await等待，主线程调用countDown后，await的线程能够同时启动。

## 74.1 工作原理

countDownLatch底层是基于AQS实现的。AQS内部维护state变量，在`new CountDownLatch(N)`时，state会初始化为N，表示需要等待的任务数量。当线程完成任务，调用countDown时，这个操作通过CAS将state减1。只要state没有减到0，线程会继续执行后续逻辑。而如果countDown后state为0，就会触发AQS的唤醒，释放AQS队列中所有await阻塞的线程。

# 75. 成员变量和局部变量的区别

1. 从语法上看，成员变量是属于累的，局部变量是属于方法的。成员变量可以用访问控制修饰符来决定是否对其他类或子类可见，局部变量的话不能有这些修饰符。
2. 从存储方式来看，成员变量属于对象的一部分，存储在堆中。而局部变量保存在栈中的局部变量表。
3. 从生命周期上看，成员变量随着对象的回收而消失，局部变量随着方法的结束而消失。
4. 从初始化的角度上看，成员变量会被赋初始值，而局部变量不会赋值。如果局部变量不赋值直接使用，就会编译报错。

# 76. final

final可以用于变量、方法和类。

1. final变量如果是基本类型，数值只能赋初始值，不能更改。如果是引用类型，那么数值也不能更改，这里的数值就是引用类型的地址。
2. final类表明当前类不能被继承。类的所有成员方法也会被指定为final。
3. final方法用来防止子类修改方法。类的所有private方法隐式指定为final。

# 77. transient

如果有不想序列化的变量，可以使用transient修饰。这样的话，将对象序列化时，这些变量就不会序列化，获取到这里的序列化对象后进行反序列化时，transient变量就会赋初始值，原来的值不可见。

而static变量默认不会序列化，因为transient是针对对象的，static变量随着类的产生而产生，反序列化后获取的static变量与序列化无关，是携带在类中的。

# 78. Java的跨平台性

Java是一次编译，到处运行的。因为编译后的class文件可以在JVM中直接运行，而不同平台有不同的JVM适配，这样的话class文件确实是能够在不同的平台上运行。

但也存在一些问题。比如本地方法，不同的系统本地方法不同，所以如果调用了特定平台的本地方法，其他平台的不一定能用。

还有操作系统的问题，如Windows文件路径用反斜杠，Linux用正斜杠，这种也是会出问题的。为了避免这些问题，最好的方法就是用Docker部署，确保运行环境一样就没有问题了。

# 79. LocalStorage

LocalStorage是Web的存储API，能够让Web程序在浏览器中以键值对的方式保存数据。

LocalStorage的数据没有过期时间。只有通过显式删除或者清空缓存才能删除数据。LocalStorage的数据是按协议+域名+端口的方式隔离的，不同的协议、域名和端口访问到的localStorage不同。

```java
localStorage.setItem('username', '张三');
let user = localStorage.getItem('username');
localStorage.removeItem('username');
localStorage.clear();
```

# 80. WebSocket

普通的HTTP请求是单工或者半双工的方式通信，由于需要一开始的握手，延迟会比较高。而使用WebSocket能够实现全双工通信，只要收到数据就能立刻传输，延迟低。而且HTTP请求需要在请求头携带数据，本身是无状态的。而WebSocket建立持久连接后，就有状态。

首先需要配置webSocket节点。

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.config.annotation.WebSocketConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketHandlerRegistry;

@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        // 注册处理类，并允许跨域
        registry.addHandler(new OrderStatusHandler(), "/ws/order/{userId}").setAllowedOrigins("*");
    }
}
```

然后在Handler中保存连接并发送消息。

```java
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.TextWebSocketHandler;
import java.util.concurrent.ConcurrentHashMap;

public class OrderStatusHandler extends TextWebSocketHandler {

    // 用来存放每个用户的 Session，实际开发中建议配合 Redis
    private static final ConcurrentHashMap<String, WebSocketSession> sessions = new ConcurrentHashMap<>();
	
    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        // 从路径截取 userId，建立映射
        String uri = session.getUri().toString();
        String userId = uri.substring(uri.lastIndexOf('/') + 1);
        sessions.put(userId, session);
        System.out.println("用户 " + userId + " 已上线");
    }

    // 模拟服务端主动推送的方法
    public static void sendOrderStatus(String userId, String status) throws Exception {
        WebSocketSession session = sessions.get(userId);
        if (session != null && session.isOpen()) {
            session.sendMessage(new TextMessage("您的订单状态更新为: " + status));
        }
    }
}
```

> 用户上限后，会在ConcurrentHashMap中添加关于userId的会话。那么只要调用sendOrderStatus方法，就会通知用户。

```java
@Service
public class OrderService {
    public void updateStatus(String orderId, String userId, String newStatus) {
        // 1. 修改数据库状态...
        
        // 2. 推送实时消息
        try {
            OrderStatusHandler.sendOrderStatus(userId, newStatus);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

> 在service层中修改了订单数据，然后在try中调用这个推送方法。

```js
const userId = "12345";
const socket = new WebSocket(`ws://localhost:8080/ws/order/${userId}`);

// 监听连接成功
socket.onopen = () => {
    console.log("WebSocket 连接已开启，准备接收订单更新...");
};

// 监听服务器发来的消息
socket.onmessage = (event) => {
    console.log("收到推送消息: ", event.data);
    alert(event.data); // 弹出提示：您的订单状态更新为: 骑手已出发
};

// 监听连接关闭
socket.onclose = () => {
    console.log("连接已关闭");
};
```

> 前端通过new WebSocket开启新的WebSocket连接，然后通过socket.onmessage来监听，收到消息就通过alert来提醒用户。

# 81. HttpClient

HttpClient是客户端编程接口，能够用来发送GET、PUT等请求。

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;

public class WeChatLoginService {

    // 微信接口地址
    private static final String WECHAT_TOKEN_URL = "https://api.weixin.qq.com/sns/oauth2/access_token";
    
    private static final String APP_ID = "YOUR_APP_ID";
    private static final String APP_SECRET = "YOUR_APP_SECRET";

    // 创建一个全局通用的 HttpClient 实例（建议单例，提高性能）
    private static final HttpClient client = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(10))
            .build();

    public String getWeChatAccessToken(String code) throws Exception {
        // 1. 构建请求 URL
        String url = String.format("%s?appid=%s&secret=%s&code=%s&grant_type=authorization_code",
                WECHAT_TOKEN_URL, APP_ID, APP_SECRET, code);

        // 2. 创建 HttpRequest
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(url))
                .GET() // 微信获取 token 使用的是 GET 请求
                .header("Accept", "application/json")
                .build();

        // 3. 发送请求并接收响应
        // HttpResponse.BodyHandlers.ofString() 会将响应体直接转为字符串
        HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

        // 4. 处理结果
        if (response.statusCode() == 200) {
            System.out.println("微信返回原始 JSON: " + response.body());
            return response.body(); // 这里通常会用 Jackson 或 Gson 解析成对象
        } else {
            throw new RuntimeException("微信接口请求失败，状态码: " + response.statusCode());
        }
    }
}
```

