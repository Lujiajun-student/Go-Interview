# 1. 什么是协程？

协程是用户态的轻量级并发执行单元，是线程调度的基本单位。在函数前加上go关键字就能实现并发，一个线程能够运行大量的协程，且协程之间的切换不涉及内核态，切换成本低。

## 1.1 生命周期

协程包含创建、运行、阻塞、唤醒、结束。

通过go关键字调用函数能够创建协程，此时协程准备开始运行。如果需要请求资源，那么就会进入阻塞状态，直到获取到资源并被唤醒，执行完或者出现异常就会结束协程。

# 2. 协程、线程和进程之间的区别

进程是拥有独立功能的程序，是系统资源分配和调度的最小单位。每个进程 都有独立的内存空间，不同进程通过管道、共享内存等进行通信。且进程比较大，切换开销很大。 

线程是CPU调度的基本单位，同一个进程的多个线程共享这个进程的内存空间，但拥有独立的程序计数器和栈。线程之间的通信主要通过共享内存，但会出现死锁和同步的问题。

协程是用户态的并发执行单元。协程之间的切换在用户态进行，不涉及内核态，调度是由用户来控制的。而且协程的栈很小，可以在一个线程创建大量的协程进行并发运行。

# 3. make和new的区别

make和new都用于内存分配的函数，但make只能用于创建slice、map、channel三种类型，会**初始化引用类型**和分配内存。而new只会分配内存，返回的是指向这个内存的指针，**不会初始化引用类型**。

make特殊，是因为slice、map和channel不是基础类型，除了元素，还需要通过make来指定slice的类型、长度和容量，指定map的类型和容量，指定channel的类型和缓冲区大小。**如果使用new来创建slice，只会分配内存空间，而长度和容量不会进行初始化，导致slice不可用。**其他引用类型也类似。

# 4. 数组和切片的区别

数组是值类型，长度在初始化时就固定了。而且数组长度是类型的一部分，不同长度的数组类型不一样。数组是值类型，如果传入函数，函数内部会创建这个数组的拷贝来使用，函数内部的修改不会改变外部的数组。

切片可以改变长度。切片是对数组的封装，描述了数组的一个片段。切片的数据结构由三个字段组成，一个指向底层数组的指针，切片长度和容量。因此切片属于引用类型，传入函数后，函数内部操作的是同一片内存，修改会反应到外部的切片上。

# 5. for range 的循环变量地址问题

在Go 1.22之前，for range中的迭代变量的内存地址不会发生改变，每次迭代时，更新的是同一个迭代变量。

```go
for index, value := range collection {}
```

如果在for range 里启动协程，那么由于协程中引用的迭代变量value与循环体的value是同一个变量，这导致了协程启动的时候，循环体其实已经结束了，所有协程读取到的value是循环结束后的value。

因此Go 1.22之后，for range进行迭代的时候，迭代变量每次都会创建新的变量。这样循环体内每次创建的协程，获取的就是那一次迭代得到的value值。

# 6. 字符串拼接方式

1. 使用`+`进行拼接。由于字符串是不可修改的，因此拼接时会开辟新的空间来保存拼接后的结果，对空间不友好。
2. `fmt.Sprintf()`。支持格式化，但底层使用反射来解析字符串，额外开销大。
3. `strings.Join`。这种方法适合将字符串切片拼接为一个字符串。它会计算字符串的总长度，一次性分配内存，因此效率比较高。
4. `bytes.Buffer`。维护一个可增长的字节缓冲区，通过逐个拼接字符串来实现字符串拼接的效果。如果先定义好容量，那么性能会比较好。

```go
var buf bytes.Buffer
buf.Grow(64)
buf.WriteString("Hello")
buf.WriteString(", ")
buf.WriteString("World")
s := buf.String()
```

5. `strings.Builder`。对`bytes.Buffer`进行改进，能够更高效地进行拼接。

# 7. defer的执行顺序和使用场景

defer的作用是延迟执行，让一个函数或者语句在当前函数即将返回前或者遇到异常时执行。

defer的执行顺序按照后进先出的原则。如果函数内按顺序出现三个defer，那么会先运行最后一个defer，然后往上运行。

其中，defer虽然最后执行，但值是在编写的时候确定的。即`defer fmt.Println(i)`的时候，是在这一行先计算并存储了i，接下来对i的修改对这个defer无效。

defer通常被用于处理成对的操作，如开启连接和关闭连接、加锁和释放锁等。

如果在defer中修改了返回的结果，如果函数声明时指定了返回的结果名，那么defer的修改会影响到返回结果，但如果没有指定返回的结果名，在return时会创建临时变量来存储返回结果，最终返回的是这个临时变量，defer的修改不会对返回结果有效。

```go
func test() (result int) {
    defer func() {
        result++ // 在 return 之后，函数真正结束前，把 0 变成 1
    }()
    return 0
}
// 最终调用 test() 得到的结果是 1
```

如果想要让defer使用变量最终值，就需要传递指针，或者将defer放到闭包里。

```go
i := 1
defer func() {
    fmt.Println(i)
}()
i++
```

# 8. rune类型

Go语言有两种字符类型，第一种是byte型，表示ASCII的字符，每个字符占用一个字节。但这里的字符表示的是英文字符，日文、中文等字符占用更多的字节。因此为了表示UTF-8字符，需要使用rune类型。rune表示的是int32类型。比如`len()`方法对`你好`字符，会计算出占用了6个长度。但如果先将字符转换为rune类型，然后计算`len()`，就会得到正确的字符长度2。

# 9. tag

Tag表示结构体的标签，为结构体成员提供属性。比如添加json表示属性名在json序列化或者反序列化时的名称，db表示数据库字段名，form表示前端的数据字段名等，方便进行结构体到数据的映射。

# 10. `%v`、`%+v`和`%#v`的区别

`%v`一般用于输出格式不明确的值。例如复合类型或用`any`声明的类型。

如果只关心值本身，使用`%v`可以打印值本身，不包含字段名和类型名。

如果需要打印值和类型，就要使用`%+v`。

如果要打印字面量形式，包含完整包路径、类型名和字段名，就要使用`%#v`。

```go
type User struct {
    Name string
    Age  int
}

u := User{"Alice", 30}

fmt.Printf("%v\n", u) // {Alice 30}

fmt.Printf("%+v\n", u) // {Name:Alice Age:30}

fmt.Printf("%#v\n", u) // main.User{Name:"Alice", Age:30}
```

# 11. struct{}

struct是Go的关键字，用来声明结构体类型。

而`struct{}`是一个类型，也就是没有任何字段的结构体类型。常用于channel的信号量。

```go
done := make(chan struct{})

go func() {
    fmt.Println("工作完成")
    done <- struct{}{} // 发送信号
}()

<-done // 等待信号
```

使用`struct{}`的原因是`struct{}`本身不会占用空间。

而`struct{}{}`则是创建了空结构体实例。空结构体实例也不会占用内存。

但存在一个特点，如果两个空结构体实例没有发生逃逸，那么这两个空结构体实例是不一样的。但如果发生了逃逸，那么所有在堆上的空结构体实例都会指向`Runtime.zerobase`，所有空结构体实例的地址一样。

# 12. Go语言执行顺序

Go程序的执行顺序如下：

```
import -> const -> var -> init() -> main()
```

每个包首先加载import的类，然后初始化包作用域的常量，再初始化包作用域的变量，然后再执行包的`init()`函数，最后再执行`main()`函数。

其中，init函数会从上到下按顺序执行。

# 13. interface{}比较规则

```go
var a, b interface{}
```

这里如果两个`interface{}`的动态类型和动态值都相等，那么interface值就相等。只要类型或者动态值有一个不同，就不相等。

```go
a = 42
b = 42
fmt.Println(a == b) // true，类型相同(int)，值相同(42)

a = 42
b = "42"
fmt.Println(a == b) // false，类型不同

a = nil
b = nil
fmt.Println(a == b) // true
```

如果interface的动态类型为slice, map或者func，这些类型本身不可比较，运行时会panic。

# 14. nil比较

两个nil之间可能会不相等。

interface底层是二元组。

```go
interface = (type, value)
```

只有type和value同时为nil，接口才等于nil。

# 15. Go mod 模块管理

`go mod init`。用来初始化当前的目录，创建`go.mod`文件。使用`go mod init <module-name>`用来创建一个模块。

`go mod tidy`。用来整理依赖关系。将没有使用的包删除，将没有的包添加进去。

`go mod download`。用来将`go.mod`的所有依赖下载到本地。

# 16. 变量声明方式

`var name string = "Gemini"`。这种是标准的声明方式。var表示变量，name表示变量名，string表示类型。

`var name = "Gemini"`省略类型，根据右侧的值自动推导。

`var name string`。不赋值，默认给定初始化的零值。

`name := "Gemini"`。短变量声明。只能在函数内使用。

# 17. goto

首先需要给出标签`exit`，然后使用goto就能直接跳转到同一函数内的其他位置。

```go
func myFunc() {
    if someError {
        goto exit
    }
    return
    
    exit:
    fmt.Println("出现错误")
}
```

# 18. 闭包

闭包是指函数内的匿名函数使用了匿名函数之外的局部变量，导致局部变量从栈逃逸到堆中，延长了这个变量的生命周期。

```go
for i := 1; i < 3; i ++ {
    go func() {
        fmt.Println(i) // Go 1.22之前会导致所有的协程输出最终值3，因为所有的协程共享同一个i的地址，协程运行时，for循环已经结束，导致读取到的时候已经是退出循环时的i=3。
    }()
}
```

# 19. panic和recover

常规的错误通过error接口来实现。而如果遇到不可恢复的致命错误，可以使用panic来返回。

遇到panic，会直接中断当前函数的执行，然后运行当前函数已经注册的defer语句，返回上一级函数，引发panic，触发上一级的defer，直到最顶级的main也退出，程序崩溃。

recover是内置函数，唯一作用是拦截panic，让程序恢复。

因为panic会终止函数的正常运行，recover需要提前在defer中注册才有效，这样panic的范围只有当前函数，对上一级函数不可见。

```go
func saveHandler() {
    defer func() {
        if err := recover(); err != nil {
            fmt.Println("panic accured...")
        }
    }()
    panic()
}

func main() {
    saveHandler()
    fmt.Println("Code completed...")
}
// 这段程序依然能够正常运行
```

但是recover存在局限性，只能捕获当前协程的panic，函数内的其他协程panic后会自行终止，不处理就会导致整个程序崩溃。因此每个协程内部如果存在panic风险，需要自行recover。



```go
func main() {
    defer func () {
        if err := recover(); err != nil {
            fmt.Println("panic accurred...")
        }
    }()
    
    go func () {
        panic()
    }
}
```

> 这里会导致整个程序崩溃。

# 20. 基本数据类型

1. 布尔类型。零值为false。
2. 整型类型。有int8, int16, int32, int64的int，uint8, uint16, uint32, uint64的uint。零值为0。
3. 浮点数类型。float32, float64。
4. 字符。有byte和rune。
5. 字符串string。

# 21. 指针

Go的指针与C的类似。使用`&`来取地址，用`*`来解引用。但Go的指针有三个特点。

1. 不可算术。地址不能通过运算移动指针，避免数组越界访问。
2. 不要求手动回收。Go会通过逃逸分析来自动回收。
3. 不能随意转换类型。

指针主要用于函数的传参。这样就能让函数内部的修改对实参生效。

还有可以提高性能，针对大结构体，函数传参进行拷贝会有很大开销，而传入指针开销小。

除了通过`&`获取地址来创建指针变量，也可以通过new来获取指针变量。

```go
p := new(int)
```

# 22.数组

数组是有固定长度和相同数据类型的元素序列。

数组的长度是类型的一部分，`[3]int`和`[4]int`是不同的类型。还有数组是值拷贝，参数传递或者赋值的时候会全量拷贝，不会传递引用。

创建方式有4种。

1. 先声明后赋值。`var arr [5]int`创建了长度为5，初始值为0的数组，接下来可以为数组赋值。
2. 使用字面量初始化。`var arr = [3]int{1, 2, 3}`或者短变量`arr := [3]int{1, 2, 3}`。
3. 自动推导长度。`var arr := [...]int{1, 2, 3, 4}`。
4. 指定索引赋值。`arr := [10]int{1: 10, 3: 30}`。

使用数组遍历时，获取的是值拷贝。

```go
for i, v := range arr {
    v++;
}
// 这里对数组的值修改对数组本身无效。
```

# 23. 切片

切片是一个只读的结构体，包含指向底层数组的指针、当前长度len和最大长度cap。

切片有4种创建方式。

1. 从数组中截取。

```go
arr := [5]int{1, 2, 3, 4, 5}
s := arr[1:4] // [2, 3, 4]
```

2. 使用make创建。

```go
s1 := make([]int, 5)
s2 := make([]string, 10)
```

3. 创建切片字面量。

```go
s := []string{"Apple", "Banana", "Cherry"}
```

4. 声明nil空切片。

```go
var s []int
```

假如切片是从数组中截取的，那么修改数组时，原数组也会改变。因为切片是使用原数组作为底层数组的指针的。

# 24. 值类型和引用类型

Go在内存分配上主要有栈和堆。栈用来存储函数调用的局部变量，分配和回收速度快。而堆是用来保存多个函数共享的数据，由垃圾回收器管理。

值类型是指将一个变量赋值给另一个变量，或者作为函数参数传递的时候，会创造一个新的副本来传参或者赋值。例如基本类型和array、struct的复合类型。值类型一般在栈上，出现闭包就会逃逸到堆。

引用类型的变量存储的时指向底层数据结构的指针。如果赋值一个引用变量或者传参，传入的是地址，因此会指向同一个地址，修改就会导致多个变量改变。

例如slice、map、channel、interface、pointer、Functions都是引用类型。

值类型的零值是类型本身的0值，而引用类型的零值都是nil。

# 25. 结构体

结构体可以通过字段名初始化，`Person{Name: "Gemini", Age: 1}`，或者按顺序初始化`Person{"Gemini", 1}`。而如果需要引用结构体，嗯需要使用`&`来取地址，或者通过new关键字来分配内存。

而如果直接赋值或者传参，那么会直接将整个结构体拷贝。而且结构体内的方法也需要使用指针接收来确保方法能够修改结构体属性。

```go
func (p *Person) SetAge(age int) {// 使用 *Person
    p.Age = age
}
```

如果结构体的字段可比较，那么可以使用`==`来进行比较。

## 25.1 继承

结构体可以将其他结构体作为匿名字段来实现继承。

```go
type Animal struct {
    Name string
}
func (a Animal) Move() {
    fmt.Println(a.Name + "移动")
}
type Dog struct {
    Animal // 实现继承
    Breed string
}

d := Dog{
    Animal: Animal{Name: "小王"},
    Breed: "金毛",
}

// 可调用父类方法
d.Move()
```

## 25.2 覆盖

子类直接写父类同名、同输入输出的方法，即可重写（覆盖）父类方法。

```go
type Animal struct {
    Name string
}
func (a Animal) Move() {
    fmt.Println(a.Name + "移动")
}
type Dog struct {
    Animal // 实现继承
    Breed string
}

func (d Dog) Move() {
    fmt.Println("狗走路")
}

d := Dog{
    Animal: Animal{Name: "小王"},
    Breed: "金毛",
}

// 调用的是子类重写的方法
d.Move()
```

而结构体字段如果首字母小写，就只有当前文件能够访问，如果首字母大写，其他文件也能访问字段。

## 25.2 内存对齐

为了提高读取结构体的效率，结构体的字段会进行填充。

1. 每个字段的偏移量必须是这个字段类型对齐值的整数倍。比如int32是4个字节，那么这个字段对齐值就是4。
2. 结构体大小必须是字段最大对齐值的整数倍。

```go
type BadStruct struct {
    A bool // 1字节
    B int64 // 8字节
    C int32 // 4字节
}
```

这里第一个字段保存的是1字节，但B字段需要起始位置能够被8整除，所以第一个字段需要填充7个字节，所以A、B两个总共占了16字节，再加上第三个，总共20字节，又因为总大小需要为最大对齐值的倍数，所以需要填充4字节，总大小为24字节。

如果更换了字段位置，可以改善大小。

```go
type GoodStruct struct {
    B int64 // 8字节
    C int32 // 4字节
    A bool // 1字节
}
```

第一个字段占8字节，然后第二个字段占4字节，最后第三个字段占1字节，共13字节。不会因为第一个字段较小，导致遇到后面较大的字段时填充字节导致空间变大。

一句话来说，**当前字段是否需要填充，就要看上一个字段添加后，大小是否为这个字段的倍数。如果不是，就要填充，然后加入当前字段。**

# 26. 接口

Go的接口实现是隐式的，一个类型实现了接口的所有方法，就是自动实现了接口。

```go
type Speaker interface {
    Say(message string) string
}
type Person struct {Name string}
func (p Person) Say(message string) string {
    return p.Name + " say " + msg
}
```

> 上面的Person添加了方法Say，形参和返回值一致，就属于实现了这个接口。

如果将接口作为参数传入，可以直接调用，只要有类型实现了接口。

```go
func Introduce(s Speaker) {
    fmt.Println(s.Say("你好"))
}
```

## 26.1 类型断言

如果拿到了接口，需要访问底层的类型，就需要使用类型断言。语法为`v := x.(T)`。如果x的类型不是T，就会panic。

```go
s := Speaker(Person{ Name: "Gemini" })
p := s.(Person) // 如果s的类型与Person不同就会报错

```

可以通过ok来检查类型是否正确，避免panic。

```go
if p, ok := s.(Person); ok {
    fmt.Println("类型正确")
} else {
    fmt.Println("类型错误")
}
```

如果要根据类型来做分支，可以用switch。

```go
switch x := i.(type) {
case int:
    fmt.Println("x is int")
case string:
    fmt.Println("x is string")
default:
    fmt.Println("other type")
}
```

而这个`.(type)`方法只能用于`interface{}`类型，对于普通类型会报错。

## 26.2 interface的使用

interface有两种用法。

1. `type Speaker interface{}`。这种是定义了一个接口，将接口起了别名叫Speaker，使用Speaker就是使用这个接口。
2. `var Speaker interface{}`。这个是定义了一个变量，类型是空接口，通过Speaker来使用这个接口。

interface{}变量能够存放任意类型的数据。

空接口也可以作为数组或者切片类型来使用。

```go
s := []any{"hello", 3, 3.1, true}
s := [2]any{"hello", 3}
```

但由于需要承接所有的数据类型，所以每一个元素需要16字节。而且any数据会逃逸到堆中。如果需要从这种数组或切片来获取数据，需要经过断言才能使用。

```go
var i any = 1
// var a int = i 报错
var a int = i.(int)
```

> any类型赋值，需要保证类型正确才能赋值。

# 27. 访问env文件

Go里面获取环境变量主要是通过os包。

写了`.env`文件，可以使用`godotenv`来读取。

```go
func main() {
    err := godotenv.Load()
    if err != nil {
        log.Fatal("Error loading env file...")
    }
    Host := os.Getenv("DB_HOST")
}
```

> 读取的.env默认是在运行程序的根目录下。读取后，就会存储这个文件，通过`os.Getenv()`来通过键获取值。

# 28. 原子操作

Go中，自增i++并不是原子操作，会有三个步骤。读取i的值，修改i的值，写入i的值。这样会出现线程安全问题。如果要原子自增，需要用`atomic.AddInt32(&i, 1)`来实现。

# 29. 密码存储方式

密码不应该使用快哈希算法来加密。像MD5、SHA256之类的属于用安全性换取时间，它们计算哈希速度很快，但很容易被攻破。因此，需要使用Bcrypt来存储。

# 30. 跨域处理

浏览去处于安全考虑，默认执行同源策略，即前后端口不一致时，无法进行通信。例如简单的网络请求，前端发送数据到后端，后端返回的数据由于接口不同，浏览器拒绝接收数据，前端无法获取数据。以及复杂的网络请求，浏览器会先发送OPTIONS请求，如果后端没有返回正确的响应，那么浏览器就不会发送这个网络请求。

现在主要有两种处理方法，分别是Go后端处理和Nginx网关层处理。

Go后端可以通过cors来允许其他来源。

```go
func main() {
    r := gin.Default()
    
    config := cors.DefaultConfig()
    config.AllowOrigins = []string{"http://localhost:3000", "https://yourdomain.com"}
    config.AllowMethods = []string{"GET", "POST", "PUT", "DELETE"}
    config.AllowHeaders = []string{"Origin", "Content-Length", "Authorization"}
    config.AllowCredentials = true
    
    r.Use(cors.New(config))
    r.Run(":8080")
}
```

> 这里允许了`http://localhost:3000`的域名，这样，如果有前端访问后端，浏览器根据白名单和前端地址就能判断当前数据是否能够交给前端。

还有Nginx处理。

```nginx
server {
    listen 80;
    server_name example.com;
    location / {
        root /var/www/dist;
        index index.html;
    }
    
    location /api/ {
        proxy_pass http://localhost:8080/;
    }
}
```

这里前端访问`api`路径时，nginx就会反向代理到`http://localhost:8080/api`。不会遇到跨域的问题。

# 31. Select

select是通道的操作。能够监听多个通道，只要接收到一个数据，那么select就会结束，开始下面的代码。

```go
func main() {
    c1 := make(chan string, 1)
    c2 := make(chan string, 1)
    
    c2 <- "hello"
    
    // 随机监听，如果多个通道有数据，会随机挑选一个通道来获取数据。
    select {
        case msg1 := <- c1:
        	fmt.Println("c1 received: ", msg1)
    	case msg2 := <- c2:
        	fmt.Println("c2 received: ", msg2)
        default:
        	fmt.Println("no data received.")
    }
}
```

> 如果有default，就会瞬间检查每个通道，有数据就输出，没数据则执行default。如果没有default，就会阻塞监听，直到从某一个通道中获取数据。因此，为了避免程序死锁，需要在selece中写default，即使default为空。

或者，定一个定时器，通过time.sleep()后向通道传输一个空数据，这样也能在select没有default的情况下避免死锁。

还有，通道关闭了只是发出了结束信号，让当前读取通道的进程停止监听。通道关闭后，不可向通道发送数据，但可以继续监听通道，能够继续获取通道里的数据。

## 31.1 使用场景

1. 竞争模式。可以监听多个数据源，直到有一个case可执行。
2. 超时控制。如果多个case无法等到数据，可以创建一个case，超时自动执行。

```go
Loop:
for{
    select {
        case val := ch:
        	fmt.println("收到数据："， val)
        case <- time.After(2 * time.Second):
        	fmt.Println("等待超时")
        	break Loop
	}
}
```

3. 信号量。需要获取资源时，需要向管道添加一个空结构体实例，如果管道满了，其他获取资源的协程需要阻塞等待，直到获取资源的方法执行完毕，从管道释放一个实例，这样其他协程才能使用资源。

# 32. 反射

Go的反射主要由Type和Value两种类型，分别对应原对象的类型和值。

反射有三个定律。

1. 反射可以将接口类型变量转化为反射类型对象。

如果调用`reflect.TypeOf(x)`或者`reflect.ValueOf(x)`，变量x会转换为`interface{}`类型。反射会读取这个接口内部的类型和数据信息，封装为`reflect.Type`和`reflect.Value`对象。

2. 反射可以将反射类型对象转化为接口类型变量。

如果有一个`reflect.Value`对象，可以通过`Interface()`方法转换为`interface{}`类型，再通过类型断言转换为普通变量。

3. 如果要修改反射类型对象，类型必须是可写的。

如果要通过反射修改变量的值，那么反射对象必须要有原始变量的地址。

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
// v.SetFloat(7.1) 这里会报错，因为访问的不是原变量。
new_v := reflect.ValueOf(&x)
fmt.Println(new_v.CanSet()) // 这里返回True，表示new_v是可修改的
new_v.SetFloat(7.1) // 这里能够正常修改
```

## 32.1 Kind

kind能够获取反射对象更深一层的类型。

```go
type User struct {
    Name string
}

type MyInt int

func main() {
    u := User{Name : "Gemini"}
	var i MyInt = 10
    tU := reflect.TypeOf(u)
    tI := reflect.TypeOf(i)
    fmt.Println(tU.Name(), tU.Kind()) // User struct
    fmt.Println(tI.Name(), tI.Kind()) // MyInt int
}
```

kind能够用来判断类型。假如类型本身是`interface{}`，那么`TypeOf()`得到的是`interface{}`类型，此时就需要通过`kind`来获取它本身的类型。

## 32.2 反射的用处

类型转换。可以通过Int()等转换类型。

```go
func main() {
    var age int = 25
    v1 := reflect.ValueOf(age)
    fmt.Println("type: %T, value: %v \n", v1, v1) // type: reflect.Value value: 25
    v2 := v1.Int()
    fmt.Println("type: %T, value: %v\n", v2, v2) // type: int64, value: 25
    
    v3 := v2.Float()
    fmt.Println("type: %T, value: %v\n", v3, v3) // type: float64, value: 25.0
}
```

> 还有`String()`，`Bool()`等。

# 33. 静态类型和动态类型

静态类型是指编码时确定的类型。

```go
var age int // int 类型
var name string // string类型
```

动态类型是运行时才能知道的类型。

```go
var i interface{}
i = 18
i = "hello"
```

> 这里i的类型是`interface{}`，静态类型就是`interface{}`。而运行后，i的静态类型不变， 动态类型变成了int。

动态类型是针对接口变量的，只有接口变量有动态类型。接口变量底层是一个双元祖，为`{type, data}`。type决定变量类型，data为值。

而接口类型根据是否带有方法而分为两种。

1. 空接口eface。

```go
var empty interface{}
```

这样没有方法，它只需要用eface记录type和data就可以了。

2. 非空接口iface。

```go
var not_empty interface{
    Move()
}
```

因为这里存在方法签名，那么不仅要记录type和data，还要记录这个类型实现的方法在什么地方。

```go
// runtime/runtime2.go
// 非空接口
type iface struct {
    tab  *itab
    data unsafe.Pointer
}
 
// 非空接口的类型信息
type itab struct {
    inter  *interfacetype  // 接口定义的类型信息
    _type  *_type      // 接口实际指向值的类型信息
    link   *itab  
    bad    int32
    inhash int32
    fun    [1]uintptr   // 接口方法实现列表，即函数地址列表，按字典序排序
}

// runtime/type.go
// 非空接口类型，接口定义，包路径等。
type interfacetype struct {
   typ     _type
   pkgpath name
   mhdr    []imethod      // 接口方法声明列表，按字典序排序
}
// 接口的方法声明 
type imethod struct {
   name nameOff          // 方法名
   ityp typeOff                // 描述方法参数返回值等细节
}
```

# 34. make 和new的区别

变量在使用之前需要分配内存。而基本类型，如int和string之类，编译器会自动完成内存分配。复杂类型就需要我们手动分配。

make和new都是用来分配内存的，而new只负责分配内存，make不仅分配内存，还会对这块内存进行初始化。

1. new。

```go
func new(Type) *Type
```

上面是new方法的签名，能够发现，new传入变量类型，返回的是指针。它会为特定类型分配内存，将分配的内存初始化为类型的零值，然后返回这个内存的指针。

new可以用于任何类型，但能够用new的基本可以简化，不能简化的不建议用new。

```go
p1 := new(int)
*p1 = 10

// 简化
var v int
p2 := &v
p2 = 10

s1 := new(Student)
s1.Name = "Bob"
s1.Age = 25

// 简化
s2 := &Student{Name: "Bob", Age: 25}
```

而针对slice、map、channel三种情况，不应该使用new。这三种是引用类型，不仅需要分配内存，还需要对内部的结构进行初始化，否则拿到单纯的地址无法正常使用这三个类型。

2. make。

```go
func make(t Type, size ...IntegerType) Type
```

上面make的方法签名能够看到，最终返回的是类型本身。

make只能用于这三个类型，不仅会分配内存，还会对内部结构进行初始化。并且可以指定长度、容量等参数。

slice底层包含指向数组的指针、长度和容量，map底层是hash结构，chan底层是带缓冲区的队列结构。这些内部结构是需要通过make来初始化的。

```go
s1 := make([]int, 3, 5) // 长度为3，容量为5的切片
s1[0] = 10

s2 := make([]string, 2) // 长度为2、容量为2的切片
s2[0] = 1

m1 := make(map[string]int) // 键为string、值为int的map
m1["张三"] = 25 // 直接使用

m2 := make(map[string]string, 10) // 键为string、值为string的map，容量为10
m2["李四"] = "老师"
```

# 35. 空结构体

```go
type Lamp struct{} // 空结构体没有字段

func (l Lamp) On() {fmt.Println("On")}
func (l Lamp) Off() {fmt.Println("Off")}

func main() {
    var x Lamp
    x.On() // On
    fmt.Println(unsafe.Sizeof(x)) // 0
}
```

在这里能够看到，空结构体的大小为0字节。

## 35.1 常见用法

1. 做集合使用。

```go
seen := make(map[string]struct{})
seen["id1"] = struct{}{}
if _, ok := seen["id1"]; ok {
    // 已存在
}
```

这样，能够用map来实现集合。

2. 作为管道的信号。

```go
done := make(chan struct{})
go func() {
    // 工作
    done <- struct{}{}
}

<- done // 等待任务结束
```

3. 有时只需要一个类型作为接口的实现类，但不需要接收状态，就可以用空结构体。

```go
type Animal interface {
    Action(msg string)
}

type Person struct{} // 空接口

func (p Person) Action(msg string) {
    fmt.Println(msg)
}

func main () {
    var a Animal = Person{}
    a.Send("Eat")
}
```

这里的Person只作为一个区分的类型标签。

# 36. 函数传参

Go的函数是一等公民，能够作为普通类型传入其他函数。

```go
func compute(a, b int, operation func(int, int) int) {
    fmt.Printf("结果为%d\n", operation(a, b))
}

func add (a, b int) int {
    return a + b
}

func multiply(a, b int) int {
    return a * b
}

func main() {
    compute(1, 2, add) // 结果为3
    compute(1, 2, multiply) // 结果为2
}
```

# 37. Channel

多个协程可能需要访问同一个共享内存来交换数据，这时候就需要管道。将管道作为参数传递给协程调用的函数即可。

而管道是发送方传输数据并关闭，接收方接收数据，直到!ok才结束接收。而range循环是语法糖，本身也是需要监听管道的ok值。

```go
ch1 := make(chan string)
ch2 := make(chan string, 2)

// 遇到nil退出
for msg := range ch1 {
    fmt.Println("获取到", msg)
}
fmt.Println("处理完毕")

// 通过ok检查
for {
    v, ok := <-ch2 {
        if !ok {
            fmt.Println("管道关闭")
            return
        }
        fmt.Println("获取到数据", v)
	}
}
```

而上面的ch1是无缓冲通道，发送放需要阻塞等待，直到消息被接收方接收；有缓冲通道能够直接放入信息，但如果通道满了，依然需要阻塞等待。

# 38. WaitGroup

WaitGroup的作用是等待一组协程执行完成。

```go
func task (id int, wg *sync.WaitGroup) {
    defer wg.Done() // 将wg的任务数-1
    // 开始任务
}

func main() {
    var wg sync.WaitGroup
    wg.Add(3) // 添加需要的任务数
    for i := 1; i <= 3; i ++ {
        go task(i, &wg)
    }
    wg.Wait() // 等待任务数为0
    fmt.Println("任务完成")
 }
```

# 39. 互斥锁和读写锁

## 39.1 互斥锁

互斥锁通过sync.Mutex来获取，需要通过lock()来获取锁，通过unlock()来释放锁。

```go
func add (count *int, wg *sync.WaitGroup, lock *sync.Mutex) {
    for i := 0; i < 1000; i ++ {
        lock.Lock()
        *count = *count + 1
        lock.unlock()
    }
    wg.Done()
}

func main() {
    var wg sync.WaitGroup
    lock := &sync.Mutex{}
    count := 0
    wg.Add(3)
    for i := 0; i < 3; i ++ {
        go add(&count, &wg, lock)
    }
    wg.Wait()
}
```

这样能够保证多线程下的线程安全，结果会是3000，而不会因为数据覆盖导致结果只有两千多。

## 39.2 读写锁

读写锁有四个方法：Lock(), Unlock(), RLock(), RUnlock()。分别对应写锁和读锁。读锁加锁时只能读不能写，写锁加锁时既不能读也不能写。

```go
var rwlock sync.RWMutex // 作为全局变量，保证所有协程共享一份，就不需要定义为指针类型

// 写操作：耗时较长
func write() {
    rwlock.Lock() // 2. 加写锁
    defer rwlock.Unlock() // 3. 释放写锁
    
    data++
    fmt.Printf("执行写操作，Data = %d\n", data)
    time.Sleep(10 * time.Millisecond) // 模拟写操作耗时
}

// 读操作：耗时较短
func read(id int) {
    rwlock.RLock() // 4. 加读锁
    defer rwlock.RUnlock() // 5. 释放读锁
    
    fmt.Printf("Reader %d 读取 Data = %d\n", id, data)
    time.Sleep(time.Millisecond) // 模拟读操作耗时
}

func main() {
    // 启动 10 个写操作（修改数据）
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            write()
            wg.Done()
        }()
    }

    // 启动 100 个读操作（查询数据）
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(id int) {
            read(id)
            wg.Done()
        }(i)
    }

    wg.Wait()
    fmt.Println("所有操作完成。")
}
```

而锁只需要定义，不需要初始化。默认情况下，锁是未锁定的状态，可以直接调用Lock加锁或者Unlock解锁。

# 40. Printf使用

与`Print()`和`Println()`不同，`Printf()`能够实现格式化输出。

通过占位符，能够指定变量的输出格式。

```go
type Profile struct {
    name string
    gender string
    age int
}
```

| 占位符 | 说明         | 示例输出                                             |
| ------ | ------------ | ---------------------------------------------------- |
| %v     | 默认格式     | {Alice female 25}                                    |
| %+v    | 显示字段名   | {name: Alice gender:female age:25}                   |
| %#v    | 显式Go语法   | main.Profile{name: "Alice", gender:"female", age:25} |
| %T     | 类型         | main.Profile                                         |
| %%     | 百分号字面值 | %                                                    |

还有的，布尔值用`%t`，字符串用`%s`，整型用`%d`，浮点数用`%f`，指针用`%p`。

# 41. Go的http使用

简单的请求可以使用`net/http`包来发送。

```go
func main() {
    resp, err := http.Get("https://baidu.com")
    if err != nil {
        fmt.Println("请求出错")
        return
    }
    defer resp.Body.Close()
    if resp.StatusCode != http.StatusOK {
        fmt.Println("请求失败")
        return 
    }
    body, err := io.ReadAll(resp.Body)
    if err != nil {
        fmt.Println("读取响应体出错:", err)
        return
    }

    fmt.Printf("响应状态: %s\n", resp.Status)
    fmt.Printf("响应体:\n%s\n", body)
}
```

```go
func main() {
    // 假设我们要发送的 JSON 数据
    jsonData := `{"title": "foo", "body": "bar", "userId": 1}`
    
    // 1. 发起 POST 请求，指定内容类型为 application/json
    resp, err := http.Post(
        "https://jsonplaceholder.typicode.com/posts", // URL
        "application/json",                          // Content-Type
        strings.NewReader(jsonData),                 // 请求体数据源
    )
    if err != nil {
        fmt.Println("请求出错:", err)
        return
    }
    defer resp.Body.Close()

    // ... (后续读取响应体的步骤与 GET 请求类似) ...
    body, _ := io.ReadAll(resp.Body)
    fmt.Printf("响应状态: %s\n", resp.Status)
    fmt.Printf("响应体:\n%s\n", body)
}
```

# 42. 单元测试

如果要对`math.go`文件的`Add`方法进行测试，测试需要满足三个要求。

1. 测试文件需要以`_test`结尾。测试`math.go`，那么测试文件的名称必须是`math_test.go`。
2. 测试文件与被测试文件必须位于同一个包中。
3. 测试函数必须以Test开头。`math_test.go`的测试方法必须写成`TestA`dd，才能测试`math.go`中的`Add`方法。

```go
func TestAdd(t *testing.T) {
    got := Add(3, 2)
    want := 5
    if got != want {
        t.Errorf("Add(%d, %d) = %d; want %d", 3, 2, got, want)
    }
}
```

如果测试用例过多，可以将这些测试用例放到切片中，通过range循环来获取每一个用例，对每一个用例进行测试。

运行测试的方法是`go test`，会运行当前目录下的所有测试。`go test -v`会打印详细信息，`go test -run TestAdd`会运行指定的测试函数，`go test -cover`会看到测试使用了被测试文件的多少代码。

而`*testing.T`有很多方法来报告测试结果。

`t.Log`用来输出日志，只有测试失败或者`-v`参数时显示。

`t.Error`输出错误，报告测试失败，但依然会执行后续代码。

`t.Fatal`会报告测试失败，终止测试。

`t.Skip`用来跳过，满足条件才会测试。

`t.Helper`。辅助函数，如果出错，就会打印调用者的行号，而不是指向`t.Errorf`的行。

最好的错误打印信息是`t.Errorf("Add(%d, %d) = %d, want %d", addend, augend, got, want)`，显示完整信息。

最好的做法是前置条件用`t.Fatalf`，类似打不开数据库、连接不上Redis等，不需要执行后续的代码。

## 42.1 子测试

在表格驱动测试中，可以只运行其中的部分数据。

```go
func TestSplit(t *testing.T) {
    type testCase struct {
        input string
        sep   string
        want  []string
    }
    
    tests := map[string]testCase{
        "simple":     {input: "a:b:c", sep: ":", want: []string{"a", "b", "c"}},
        "wrong_sep":  {input: "a:b:c", sep: ",", want: []string{"a:b:c"}},
        "multi_char": {input: "abcd", sep: "bc", want: []string{"a", "d"}},
    }
    
    for name, tc := range tests {
        t.Run(name, func(t *testing.T) {
            got := Split(tc.input, tc.sep)
            if !reflect.DeepEqual(got, tc.want) {
                t.Errorf("expected: %v, got: %v", tc.want, got)
            }
        })
    }
}
```

这里定义了多个testCase，而存在名字simple、wrong_sep等。可以通过`go test -run TestSplit/simple -v`来单独运行子测试。

## 42.2 并行测试

并行测试是在方法里调用`t.Parallel()`这样能够让当前的测试暂停，执行接下来非并行的测试，然后将所有有`t.Parallel`的方法一起运行。

# 43. Go命令

Go文件需要先编译再执行。通常有两种方法。

1. `go build`编译成二进制文件，然后执行文件。
2. `go run`运行go文件。

111
