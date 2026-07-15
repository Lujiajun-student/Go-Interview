下面给你一个**完整 Spring 学习大纲**。我建议按这条主线学：

**Java 基础 → Maven/Gradle → Spring Framework 核心 → Spring Boot → Web 开发 → 数据访问 → 事务 → AOP → 测试 → 安全 → 缓存/异步/定时任务 → 源码 → 微服务与生产部署。**

现在 Spring 官方文档中，Spring Framework 覆盖 IoC、AOP、数据访问、事务、Spring MVC、WebFlux、测试、AOT 等核心模块；Spring Boot 的定位是快速创建可独立运行、生产级的 Spring 应用，并通过 starter、自动配置等机制减少配置成本。([Home](https://docs.spring.io/spring-framework/reference/index.html?utm_source=chatgpt.com))
实际学习时，我建议先以 **Spring Framework 6.x + Spring Boot 3.x** 为主，因为企业项目、教程、生态兼容性更广；之后再了解 Spring Framework 7 / Spring Boot 4 的变化。官方当前文档也同时维护 Spring Framework 7.x/6.x 与 Spring Boot 4.x/3.x 版本线。([Home](https://docs.spring.io/spring-framework/reference/core.html?utm_source=chatgpt.com))

------

# 一、学习前置知识

学习 Spring 之前，至少要掌握这些 Java 基础：

1. Java 面向对象：类、对象、接口、抽象类、继承、多态。
2. 注解：`@Override`、自定义注解、运行时注解。
3. 反射：`Class`、`Method`、`Field`、构造器调用。
4. 泛型：`List<T>`、`Map<K,V>`、通配符。
5. 异常处理：受检异常、运行时异常、异常传播。
6. 集合框架：`List`、`Set`、`Map`。
7. Lambda 与函数式接口。
8. JDBC 基础。
9. Servlet 基础。
10. Maven 或 Gradle。

Spring 本质上大量使用了：

```java
// 反射：Spring 可以通过 Class 创建对象
Class<?> clazz = UserService.class;
Object obj = clazz.getDeclaredConstructor().newInstance();

// 注解：Spring 通过扫描注解识别 Bean
@Service
public class UserService {
}
```

你可以把 Spring 理解成一个**大型对象管理器 + 企业开发基础设施集合**。它替你创建对象、管理对象关系、管理事务、处理 Web 请求、整合数据库、整合缓存、做测试支持、做安全控制等。

------

# 二、Spring 总体架构

Spring 不是单一技术，而是一组项目。

核心部分：

1. **Spring Core / Beans / Context**
   负责 IoC 容器、Bean 管理、依赖注入。
2. **Spring AOP**
   负责切面编程，例如日志、权限、事务、监控。
3. **Spring JDBC / Transaction / ORM**
   负责数据库访问、事务管理、JPA/MyBatis 整合。
4. **Spring MVC**
   基于 Servlet 的 Web 框架。Spring MVC 是 Spring 最早、最经典的 Web 模块之一，核心是 `DispatcherServlet` 分发请求到对应处理器。([Home](https://docs.spring.io/spring-framework/reference/web/webmvc.html?utm_source=chatgpt.com))
5. **Spring WebFlux**
   响应式 Web 框架，适合高并发、异步非阻塞场景。
6. **Spring Test**
   提供单元测试、集成测试、Mock MVC 等能力。
7. **Spring Boot**
   简化 Spring 项目创建、配置和部署。Spring Boot 支持打成可执行 jar，也可以部署为传统 war。([Home](https://docs.spring.io/spring-boot/index.html?utm_source=chatgpt.com))
8. **Spring Security**
   认证、授权、安全过滤器链。
9. **Spring Data**
   简化数据库、Redis、MongoDB、Elasticsearch 等数据访问。
10. **Spring Cloud**
    微服务生态，例如服务注册、配置中心、网关、熔断、链路追踪等。

------

# 三、Spring 最核心思想：IoC 与 DI

## 1. 什么是 IoC？

IoC，全称 Inversion of Control，控制反转。

普通 Java 写法：

```java
public class UserService {

    // UserService 自己创建 UserRepository
    private UserRepository userRepository = new UserRepository();
}
```

问题是：对象之间强耦合。`UserService` 依赖 `UserRepository` 的具体实现。

Spring 写法：

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    // UserRepository 不由 UserService 自己创建，而是由 Spring 注入
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

这里对象的创建权从程序员手里交给了 Spring 容器，所以叫控制反转。

## 2. 什么是 DI？

DI，全称 Dependency Injection，依赖注入。

IoC 是思想，DI 是实现方式。

常见注入方式有三种：

### 构造器注入，推荐

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    // Spring 会自动把 UserRepository 类型的 Bean 传进来
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

优点：

1. 依赖不可变。
2. 方便测试。
3. 能避免对象创建后缺依赖。
4. 可以减少循环依赖问题。

### Setter 注入

```java
@Service
public class UserService {

    private UserRepository userRepository;

    // Spring 会调用 set 方法注入依赖
    @Autowired
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

适合可选依赖。

### 字段注入，不推荐

```java
@Service
public class UserService {

    // 不推荐：测试困难，依赖不明显
    @Autowired
    private UserRepository userRepository;
}
```

字段注入虽然写起来简单，但不利于单元测试和依赖分析。

------

# 四、Bean 的概念

## 1. 什么是 Bean？

被 Spring 容器管理的对象，叫 Bean。

例如：

```java
@Component
public class SmsSender {
}
```

Spring 扫描到 `@Component` 后，会创建 `SmsSender` 对象，并放入容器。

常见注解：

```java
@Component  // 通用组件
@Service    // 业务层
@Repository // 数据访问层
@Controller // MVC 控制器
@RestController // REST 控制器，等于 @Controller + @ResponseBody
@Configuration // 配置类
@Bean       // 方法返回值注册为 Bean
```

## 2. 使用 `@Bean` 手动注册 Bean

```java
@Configuration
public class AppConfig {

    // 方法返回的对象会被 Spring 管理
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

适合注册第三方类，因为第三方类不能直接加 `@Component`。

## 3. Bean 默认是单例

```java
@Component
public class OrderService {
}
```

默认情况下，Spring 容器中只有一个 `OrderService` 对象。

可以修改作用域：

```java
@Component
@Scope("prototype")
public class TaskObject {
}
```

常见作用域：

| 作用域      | 含义                        |
| ----------- | --------------------------- |
| singleton   | 默认，全容器一个对象        |
| prototype   | 每次获取都创建新对象        |
| request     | 每个 HTTP 请求一个对象      |
| session     | 每个 HTTP Session 一个对象  |
| application | ServletContext 级别一个对象 |

------

# 五、Bean 生命周期

一个 Bean 大致经历：

1. 实例化。
2. 属性填充。
3. Aware 回调。
4. BeanPostProcessor 前置处理。
5. 初始化方法。
6. BeanPostProcessor 后置处理。
7. Bean 可用。
8. 容器关闭时销毁。

示例：

```java
@Component
public class LifeCycleDemo implements InitializingBean, DisposableBean {

    public LifeCycleDemo() {
        System.out.println("1. 构造方法：创建对象");
    }

    @PostConstruct
    public void postConstruct() {
        System.out.println("2. @PostConstruct：依赖注入完成后执行");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("3. InitializingBean：属性设置完成后执行");
    }

    @PreDestroy
    public void preDestroy() {
        System.out.println("4. @PreDestroy：销毁前执行");
    }

    @Override
    public void destroy() {
        System.out.println("5. DisposableBean：容器关闭时执行");
    }
}
```

实际项目中更推荐：

```java
@PostConstruct
public void init() {
    // 初始化资源
}

@PreDestroy
public void destroy() {
    // 释放资源
}
```

不要滥用 `InitializingBean` 和 `DisposableBean`，因为它们会让业务类和 Spring 接口产生耦合。

------

# 六、Spring 配置方式

Spring 配置经历过几个阶段：

## 1. XML 配置，老项目常见

```xml
<bean id="userService" class="com.example.UserService"/>
```

现在新项目较少使用，但维护老项目时仍需要看懂。

## 2. 注解配置，主流

```java
@Service
public class UserService {
}
@Configuration
@ComponentScan("com.example")
public class AppConfig {
}
```

## 3. Java Config，主流

```java
@Configuration
public class DataSourceConfig {

    @Bean
    public DataSource dataSource() {
        // 创建数据源
        return new HikariDataSource();
    }
}
```

## 4. Spring Boot 自动配置，最常用

Spring Boot 会根据依赖、配置文件、条件注解自动创建很多 Bean。

例如你引入：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

Spring Boot 会自动配置：

1. Tomcat。
2. Spring MVC。
3. JSON 序列化。
4. 参数转换器。
5. 异常处理基础设施。
6. 静态资源处理。
7. Web 相关 Bean。

Spring Boot 官方也强调 starter 是依赖声明的常见方式，用来简化项目搭建。([Home](https://docs.spring.io/spring-boot/docs/3.2.5/reference/htmlsingle/?utm_source=chatgpt.com))

------

# 七、Spring Boot 快速入门项目

下面给一个完整的最小 Web 项目。

## 1. `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
                             https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>spring-learning-demo</artifactId>
    <version>1.0.0</version>

    <!-- Spring Boot 父工程：统一管理依赖版本和插件 -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.5.14</version>
        <relativePath/>
    </parent>

    <properties>
        <!-- Spring Boot 3.x 通常使用 Java 17+ -->
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <!-- Web 开发：包含 Spring MVC、内嵌 Tomcat、Jackson JSON -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- 参数校验：支持 @NotBlank、@Min、@Email 等注解 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- 测试：JUnit、Mockito、Spring Test 等 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- 用于打包可执行 jar -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

## 2. 启动类

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

// @SpringBootApplication = @SpringBootConfiguration + @EnableAutoConfiguration + @ComponentScan
@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        // 启动 Spring Boot 应用，创建 Spring 容器，启动内嵌 Tomcat
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

## 3. 实体类

```java
package com.example.demo.user;

// 简单用户实体类
public class User {

    private Long id;
    private String username;
    private Integer age;

    public User() {
    }

    public User(Long id, String username, Integer age) {
        this.id = id;
        this.username = username;
        this.age = age;
    }

    public Long getId() {
        return id;
    }

    public String getUsername() {
        return username;
    }

    public Integer getAge() {
        return age;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public void setAge(Integer age) {
        this.age = age;
    }
}
```

## 4. 请求 DTO

```java
package com.example.demo.user;

import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;

// DTO：用于接收前端请求参数
public class CreateUserRequest {

    // 用户名不能为空
    @NotBlank(message = "用户名不能为空")
    private String username;

    // 年龄不能为空，且最小为 1
    @NotNull(message = "年龄不能为空")
    @Min(value = 1, message = "年龄必须大于等于 1")
    private Integer age;

    public String getUsername() {
        return username;
    }

    public Integer getAge() {
        return age;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public void setAge(Integer age) {
        this.age = age;
    }
}
```

## 5. Repository 层

这里先不用数据库，用内存 `Map` 模拟。

```java
package com.example.demo.user;

import org.springframework.stereotype.Repository;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

// @Repository 表示这是数据访问层组件，会被 Spring 扫描并注册为 Bean
@Repository
public class UserRepository {

    // 模拟数据库表
    private final Map<Long, User> database = new ConcurrentHashMap<>();

    public UserRepository() {
        // 初始化一些测试数据
        database.put(1L, new User(1L, "Tom", 18));
        database.put(2L, new User(2L, "Jerry", 20));
    }

    public List<User> findAll() {
        // 返回所有用户
        return new ArrayList<>(database.values());
    }

    public User findById(Long id) {
        // 根据 id 查询用户
        return database.get(id);
    }

    public User save(User user) {
        // 保存用户
        database.put(user.getId(), user);
        return user;
    }
}
```

## 6. Service 层

```java
package com.example.demo.user;

import org.springframework.stereotype.Service;

import java.util.List;
import java.util.concurrent.atomic.AtomicLong;

// @Service 表示这是业务层组件
@Service
public class UserService {

    private final UserRepository userRepository;

    // 用于生成自增 id
    private final AtomicLong idGenerator = new AtomicLong(100);

    // 构造器注入，推荐方式
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public List<User> listUsers() {
        // 查询所有用户
        return userRepository.findAll();
    }

    public User getUser(Long id) {
        User user = userRepository.findById(id);

        // 查询不到时抛出业务异常
        if (user == null) {
            throw new UserNotFoundException("用户不存在，id = " + id);
        }

        return user;
    }

    public User createUser(CreateUserRequest request) {
        // 生成新 id
        Long id = idGenerator.incrementAndGet();

        // 创建用户对象
        User user = new User(id, request.getUsername(), request.getAge());

        // 保存用户
        return userRepository.save(user);
    }
}
```

## 7. 自定义异常

```java
package com.example.demo.user;

// 用户不存在异常
public class UserNotFoundException extends RuntimeException {

    public UserNotFoundException(String message) {
        super(message);
    }
}
```

## 8. Controller 层

```java
package com.example.demo.user;

import jakarta.validation.Valid;
import org.springframework.web.bind.annotation.*;

import java.util.List;

// @RestController = @Controller + @ResponseBody
// 表示这个类返回的是 JSON 数据，而不是页面
@RestController
@RequestMapping("/users")
public class UserController {

    private final UserService userService;

    // 构造器注入 Service
    public UserController(UserService userService) {
        this.userService = userService;
    }

    // GET /users
    @GetMapping
    public List<User> listUsers() {
        return userService.listUsers();
    }

    // GET /users/1
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.getUser(id);
    }

    // POST /users
    @PostMapping
    public User createUser(@Valid @RequestBody CreateUserRequest request) {
        return userService.createUser(request);
    }
}
```

## 9. 全局异常处理

```java
package com.example.demo.common;

import com.example.demo.user.UserNotFoundException;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.Map;

// @RestControllerAdvice 用于统一处理 Controller 抛出的异常
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 处理用户不存在异常
    @ExceptionHandler(UserNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public Map<String, Object> handleUserNotFound(UserNotFoundException ex) {
        Map<String, Object> result = new HashMap<>();
        result.put("code", 404);
        result.put("message", ex.getMessage());
        return result;
    }

    // 处理参数校验异常
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Map<String, Object> handleValidationException(MethodArgumentNotValidException ex) {
        Map<String, Object> result = new HashMap<>();
        result.put("code", 400);

        // 获取第一个校验错误信息
        String message = ex.getBindingResult()
                .getFieldErrors()
                .get(0)
                .getDefaultMessage();

        result.put("message", message);
        return result;
    }

    // 兜底异常处理
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public Map<String, Object> handleException(Exception ex) {
        Map<String, Object> result = new HashMap<>();
        result.put("code", 500);
        result.put("message", "服务器内部错误");
        return result;
    }
}
```

启动后可以测试：

```bash
curl http://localhost:8080/users

curl http://localhost:8080/users/1

curl -X POST http://localhost:8080/users \
  -H "Content-Type: application/json" \
  -d '{"username":"Alice","age":22}'
```

------

# 八、Spring MVC 必学知识点

Spring MVC 的核心流程：

1. 浏览器发送请求。
2. 请求进入 `DispatcherServlet`。
3. `HandlerMapping` 查找对应 Controller 方法。
4. `HandlerAdapter` 调用 Controller 方法。
5. 参数解析，例如 `@RequestParam`、`@PathVariable`、`@RequestBody`。
6. 执行业务逻辑。
7. 返回结果。
8. `HttpMessageConverter` 把对象转成 JSON。
9. 响应客户端。

必须掌握的注解：

```java
@RestController
@RequestMapping
@GetMapping
@PostMapping
@PutMapping
@DeleteMapping
@PathVariable
@RequestParam
@RequestBody
@ResponseBody
@RequestHeader
@CookieValue
@Valid
@ControllerAdvice
@ExceptionHandler
```

示例：

```java
@GetMapping("/search")
public List<User> search(
        @RequestParam String keyword,
        @RequestParam(defaultValue = "1") Integer page,
        @RequestParam(defaultValue = "10") Integer size) {

    // keyword 来自 ?keyword=xxx
    // page 和 size 如果没有传，则使用默认值
    return List.of();
}
```

上传文件：

```java
@PostMapping("/upload")
public String upload(@RequestParam("file") MultipartFile file) throws IOException {
    // 获取原始文件名
    String filename = file.getOriginalFilename();

    // 获取文件字节
    byte[] bytes = file.getBytes();

    // 实际项目中应保存到对象存储或文件服务器
    return "上传成功：" + filename + "，大小：" + bytes.length;
}
```

拦截器：

```java
@Component
public class LoginInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {

        // 请求进入 Controller 前执行
        String token = request.getHeader("Authorization");

        if (token == null || token.isBlank()) {
            response.setStatus(401);
            response.getWriter().write("未登录");
            return false;
        }

        return true;
    }
}
```

注册拦截器：

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    private final LoginInterceptor loginInterceptor;

    public WebConfig(LoginInterceptor loginInterceptor) {
        this.loginInterceptor = loginInterceptor;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // 拦截所有请求，但放行登录接口
        registry.addInterceptor(loginInterceptor)
                .addPathPatterns("/**")
                .excludePathPatterns("/login");
    }
}
```

------

# 九、配置文件与环境隔离

Spring Boot 常用配置文件：

```yaml
# application.yml
server:
  port: 8080

spring:
  application:
    name: spring-learning-demo

app:
  upload-dir: /tmp/uploads
  max-size: 10485760
```

读取单个配置：

```java
@Component
public class UploadService {

    // 从配置文件读取 app.upload-dir
    @Value("${app.upload-dir}")
    private String uploadDir;
}
```

更推荐使用 `@ConfigurationProperties`：

```java
package com.example.demo.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

// 读取 app 前缀下的配置
@Component
@ConfigurationProperties(prefix = "app")
public class AppProperties {

    private String uploadDir;
    private Long maxSize;

    public String getUploadDir() {
        return uploadDir;
    }

    public Long getMaxSize() {
        return maxSize;
    }

    public void setUploadDir(String uploadDir) {
        this.uploadDir = uploadDir;
    }

    public void setMaxSize(Long maxSize) {
        this.maxSize = maxSize;
    }
}
```

使用：

```java
@Service
public class FileService {

    private final AppProperties appProperties;

    public FileService(AppProperties appProperties) {
        this.appProperties = appProperties;
    }

    public void printConfig() {
        System.out.println("上传目录：" + appProperties.getUploadDir());
        System.out.println("最大大小：" + appProperties.getMaxSize());
    }
}
```

多环境配置：

```yaml
# application.yml
spring:
  profiles:
    active: dev
# application-dev.yml
server:
  port: 8080
# application-prod.yml
server:
  port: 80
```

启动时指定环境：

```bash
java -jar app.jar --spring.profiles.active=prod
```

------

# 十、条件装配

Spring Boot 自动配置大量依赖条件注解。

常见条件注解：

```java
@ConditionalOnClass
@ConditionalOnMissingBean
@ConditionalOnBean
@ConditionalOnProperty
@ConditionalOnWebApplication
@Profile
```

示例：

```java
@Configuration
public class SmsConfig {

    // 只有 app.sms.enabled=true 时才创建这个 Bean
    @Bean
    @ConditionalOnProperty(prefix = "app.sms", name = "enabled", havingValue = "true")
    public SmsSender smsSender() {
        return new SmsSender();
    }
}
public class SmsSender {

    public void send(String phone, String message) {
        System.out.println("发送短信给：" + phone + "，内容：" + message);
    }
}
```

配置：

```yaml
app:
  sms:
    enabled: true
```

这个机制是 Spring Boot 自动配置的基础。

------

# 十一、AOP 切面编程

AOP 用来处理横切逻辑，例如：

1. 日志。
2. 权限。
3. 事务。
4. 监控。
5. 缓存。
6. 限流。
7. 参数校验。
8. 防重复提交。

核心概念：

| 概念       | 含义                           |
| ---------- | ------------------------------ |
| Join Point | 连接点，例如方法调用           |
| Pointcut   | 切点，匹配哪些方法             |
| Advice     | 通知，增强逻辑                 |
| Aspect     | 切面，切点 + 通知              |
| Weaving    | 织入，把增强逻辑应用到目标对象 |
| Proxy      | 代理对象                       |

引入依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

示例：记录接口耗时。

```java
package com.example.demo.common;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.springframework.stereotype.Component;

// @Aspect 表示这是一个切面
@Aspect
@Component
public class LogAspect {

    // 匹配 com.example.demo 包下所有类的所有方法
    @Pointcut("execution(* com.example.demo..*(..))")
    public void applicationMethods() {
    }

    // 环绕通知：方法执行前后都能增强
    @Around("applicationMethods()")
    public Object logTime(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();

        try {
            // 执行目标方法
            return joinPoint.proceed();
        } finally {
            long cost = System.currentTimeMillis() - start;

            // 获取类名和方法名
            String className = joinPoint.getTarget().getClass().getSimpleName();
            String methodName = joinPoint.getSignature().getName();

            System.out.println(className + "." + methodName + " 耗时：" + cost + "ms");
        }
    }
}
```

注意：Spring AOP 默认基于代理实现，所以它有一些限制：

1. 同一个类内部方法互相调用，可能不会经过代理。
2. 默认只增强 Spring Bean。
3. 对 final 类、final 方法有限制。
4. 复杂编译期织入需要 AspectJ。

------

# 十二、事务管理

事务用于保证一组数据库操作要么都成功，要么都失败。

四大特性 ACID：

1. Atomicity：原子性。
2. Consistency：一致性。
3. Isolation：隔离性。
4. Durability：持久性。

Spring 声明式事务：

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final AccountRepository accountRepository;

    public OrderService(OrderRepository orderRepository,
                        AccountRepository accountRepository) {
        this.orderRepository = orderRepository;
        this.accountRepository = accountRepository;
    }

    // 开启事务
    // 方法执行成功：提交事务
    // 抛出 RuntimeException：回滚事务
    @Transactional
    public void createOrder(Long userId, Long productId, Integer amount) {
        // 1. 创建订单
        orderRepository.insert(userId, productId, amount);

        // 2. 扣减余额
        accountRepository.decreaseBalance(userId, amount);

        // 如果这里抛异常，上面两个数据库操作会一起回滚
        if (amount <= 0) {
            throw new IllegalArgumentException("金额必须大于 0");
        }
    }
}
```

事务传播行为：

| 传播行为      | 含义                           |
| ------------- | ------------------------------ |
| REQUIRED      | 默认。有事务就加入，没有就新建 |
| REQUIRES_NEW  | 总是新建事务，挂起外部事务     |
| SUPPORTS      | 有事务就加入，没有就非事务执行 |
| NOT_SUPPORTED | 总是非事务执行                 |
| MANDATORY     | 必须存在事务，否则报错         |
| NEVER         | 必须没有事务，否则报错         |
| NESTED        | 嵌套事务，基于保存点           |

示例：

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void writeLog() {
    // 即使外层事务回滚，这里的新事务也可以独立提交
}
```

事务隔离级别：

| 隔离级别         | 说明                         |
| ---------------- | ---------------------------- |
| READ_UNCOMMITTED | 可能读到未提交数据           |
| READ_COMMITTED   | 只能读到已提交数据           |
| REPEATABLE_READ  | 同一事务内多次读取结果一致   |
| SERIALIZABLE     | 串行化，隔离性最高，性能最低 |

示例：

```java
@Transactional(
        isolation = Isolation.READ_COMMITTED,
        rollbackFor = Exception.class
)
public void pay() throws Exception {
    // rollbackFor = Exception.class 表示受检异常也回滚
}
```

事务常见失效场景：

1. 方法不是 public。
2. 同类内部方法调用。
3. 异常被 catch 后没有继续抛出。
4. 抛出受检异常但没有配置 `rollbackFor`。
5. 类没有被 Spring 管理。
6. 数据库引擎不支持事务。
7. 多线程中事务上下文丢失。

错误示例：

```java
@Service
public class UserService {

    public void outer() {
        // 同类内部调用，可能不会经过 Spring 代理，事务失效
        inner();
    }

    @Transactional
    public void inner() {
        // 事务逻辑
    }
}
```

正确做法之一：把事务方法放到另一个 Bean 中。

------

# 十三、数据库访问

## 1. JDBC

引入依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>

<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>
```

配置：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/demo?useSSL=false&serverTimezone=Asia/Shanghai
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver
```

使用 `JdbcTemplate`：

```java
@Repository
public class UserJdbcRepository {

    private final JdbcTemplate jdbcTemplate;

    public UserJdbcRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public List<User> findAll() {
        String sql = "select id, username, age from user";

        return jdbcTemplate.query(sql, (rs, rowNum) -> {
            User user = new User();
            user.setId(rs.getLong("id"));
            user.setUsername(rs.getString("username"));
            user.setAge(rs.getInt("age"));
            return user;
        });
    }

    public int insert(User user) {
        String sql = "insert into user(username, age) values (?, ?)";

        return jdbcTemplate.update(sql, user.getUsername(), user.getAge());
    }
}
```

## 2. Spring Data JPA

适合领域模型明确、CRUD 较多的项目。

依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

实体：

```java
import jakarta.persistence.*;

@Entity
@Table(name = "user")
public class UserEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "username", nullable = false)
    private String username;

    private Integer age;

    // getter/setter 省略
}
```

Repository：

```java
public interface UserJpaRepository extends JpaRepository<UserEntity, Long> {

    // Spring Data JPA 会根据方法名自动生成查询
    List<UserEntity> findByUsernameContaining(String keyword);

    List<UserEntity> findByAgeGreaterThan(Integer age);
}
```

使用：

```java
@Service
public class UserJpaService {

    private final UserJpaRepository userJpaRepository;

    public UserJpaService(UserJpaRepository userJpaRepository) {
        this.userJpaRepository = userJpaRepository;
    }

    public List<UserEntity> search(String keyword) {
        return userJpaRepository.findByUsernameContaining(keyword);
    }
}
```

## 3. MyBatis / MyBatis-Plus

MyBatis 适合 SQL 可控性要求高的项目。

Mapper：

```java
@Mapper
public interface UserMapper {

    @Select("select id, username, age from user where id = #{id}")
    User findById(Long id);

    @Insert("insert into user(username, age) values(#{username}, #{age})")
    int insert(User user);
}
```

XML 方式：

```xml
<select id="findById" resultType="com.example.demo.user.User">
    select id, username, age
    from user
    where id = #{id}
</select>
```

选择建议：

| 技术         | 适合场景                |
| ------------ | ----------------------- |
| JdbcTemplate | 简单 SQL、轻量项目      |
| JPA          | 对象模型清晰，CRUD 多   |
| MyBatis      | SQL 复杂，需要手写优化  |
| MyBatis-Plus | 国内常见，CRUD 快速开发 |

------

# 十四、参数校验 Validation

DTO：

```java
public class RegisterRequest {

    @NotBlank(message = "用户名不能为空")
    private String username;

    @Email(message = "邮箱格式不正确")
    private String email;

    @Size(min = 6, max = 20, message = "密码长度必须是 6 到 20 位")
    private String password;

    // getter/setter
}
```

Controller：

```java
@PostMapping("/register")
public String register(@Valid @RequestBody RegisterRequest request) {
    return "注册成功";
}
```

常用注解：

| 注解      | 含义                           |
| --------- | ------------------------------ |
| @NotNull  | 不能为 null                    |
| @NotBlank | 字符串不能为 null 且不能全空白 |
| @NotEmpty | 集合或字符串不能为空           |
| @Min      | 最小值                         |
| @Max      | 最大值                         |
| @Size     | 长度或集合大小                 |
| @Email    | 邮箱格式                       |
| @Pattern  | 正则匹配                       |
| @Past     | 必须是过去时间                 |
| @Future   | 必须是未来时间                 |

------

# 十五、统一响应结构

实际项目中一般不会直接返回实体，而是封装统一响应。

```java
public class ApiResponse<T> {

    private int code;
    private String message;
    private T data;

    public ApiResponse() {
    }

    public ApiResponse(int code, String message, T data) {
        this.code = code;
        this.message = message;
        this.data = data;
    }

    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>(200, "success", data);
    }

    public static <T> ApiResponse<T> fail(int code, String message) {
        return new ApiResponse<>(code, message, null);
    }

    // getter/setter 省略
}
```

Controller：

```java
@GetMapping("/{id}")
public ApiResponse<User> getUser(@PathVariable Long id) {
    return ApiResponse.success(userService.getUser(id));
}
```

------

# 十六、过滤器、拦截器、AOP 的区别

| 机制        | 所在层级      | 典型用途                       |
| ----------- | ------------- | ------------------------------ |
| Filter      | Servlet 层    | 跨域、编码、登录校验           |
| Interceptor | Spring MVC 层 | 权限、请求日志、接口鉴权       |
| AOP         | 方法调用层    | 业务方法日志、事务、限流、监控 |

执行顺序大致是：

```text
客户端请求
  ↓
Filter
  ↓
DispatcherServlet
  ↓
Interceptor preHandle
  ↓
Controller
  ↓
Service AOP
  ↓
Interceptor postHandle
  ↓
Interceptor afterCompletion
  ↓
Filter
  ↓
客户端响应
```

------

# 十七、Spring 事件机制

适合模块之间解耦。

事件类：

```java
public class UserRegisteredEvent {

    private final Long userId;
    private final String email;

    public UserRegisteredEvent(Long userId, String email) {
        this.userId = userId;
        this.email = email;
    }

    public Long getUserId() {
        return userId;
    }

    public String getEmail() {
        return email;
    }
}
```

发布事件：

```java
@Service
public class RegisterService {

    private final ApplicationEventPublisher eventPublisher;

    public RegisterService(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    public void register(String username, String email) {
        // 1. 保存用户
        Long userId = 1L;

        // 2. 发布用户注册事件
        eventPublisher.publishEvent(new UserRegisteredEvent(userId, email));
    }
}
```

监听事件：

```java
@Component
public class UserEventListener {

    @EventListener
    public void handleUserRegistered(UserRegisteredEvent event) {
        // 监听用户注册事件，发送欢迎邮件
        System.out.println("发送欢迎邮件：" + event.getEmail());
    }
}
```

异步事件：

```java
@EnableAsync
@Configuration
public class AsyncConfig {
}
@Component
public class UserEventListener {

    @Async
    @EventListener
    public void handleUserRegistered(UserRegisteredEvent event) {
        // 异步处理，不阻塞注册流程
        System.out.println("异步发送邮件：" + event.getEmail());
    }
}
```

------

# 十八、异步任务

开启异步：

```java
@EnableAsync
@Configuration
public class AsyncConfig {
}
```

定义异步方法：

```java
@Service
public class EmailService {

    @Async
    public void sendEmail(String email) {
        // 该方法会在线程池中执行
        System.out.println("发送邮件：" + email);
    }
}
```

自定义线程池：

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();

        // 核心线程数
        executor.setCorePoolSize(5);

        // 最大线程数
        executor.setMaxPoolSize(20);

        // 队列容量
        executor.setQueueCapacity(100);

        // 线程名前缀
        executor.setThreadNamePrefix("async-task-");

        executor.initialize();
        return executor;
    }
}
```

注意：

1. `@Async` 方法必须通过 Spring 代理调用。
2. 同类内部调用可能失效。
3. 异步方法中的异常需要单独处理。
4. 不要无脑创建过大的线程池。

------

# 十九、定时任务

开启定时任务：

```java
@EnableScheduling
@Configuration
public class ScheduleConfig {
}
```

定时任务：

```java
@Component
public class ReportTask {

    // 每 5 秒执行一次
    @Scheduled(fixedRate = 5000)
    public void generateReport() {
        System.out.println("生成报表：" + LocalDateTime.now());
    }

    // cron 表达式：每天凌晨 2 点执行
    @Scheduled(cron = "0 0 2 * * ?")
    public void cleanData() {
        System.out.println("清理数据");
    }
}
```

常用 cron：

```text
0 */5 * * * ?      每 5 分钟
0 0 0 * * ?        每天 0 点
0 0 9 ? * MON-FRI  工作日 9 点
```

------

# 二十、缓存 Cache

引入依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

开启缓存：

```java
@EnableCaching
@Configuration
public class CacheConfig {
}
```

使用缓存：

```java
@Service
public class ProductService {

    // 查询结果放入 productCache，key 为 id
    @Cacheable(cacheNames = "productCache", key = "#id")
    public Product getProduct(Long id) {
        // 模拟慢查询
        System.out.println("查询数据库");
        return new Product(id, "手机");
    }

    // 更新后清除缓存
    @CacheEvict(cacheNames = "productCache", key = "#id")
    public void updateProduct(Long id, Product product) {
        System.out.println("更新数据库");
    }
}
```

常用注解：

| 注解        | 含义                     |
| ----------- | ------------------------ |
| @Cacheable  | 查询时写入缓存           |
| @CachePut   | 方法一定执行，并更新缓存 |
| @CacheEvict | 删除缓存                 |
| @Caching    | 组合多个缓存操作         |

实际项目常配合 Redis。

------

# 二十一、Redis 整合

依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

配置：

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
```

使用：

```java
@Service
public class RedisUserService {

    private final StringRedisTemplate stringRedisTemplate;

    public RedisUserService(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    public void saveToken(Long userId, String token) {
        String key = "login:token:" + userId;

        // 保存 token，有效期 30 分钟
        stringRedisTemplate.opsForValue()
                .set(key, token, Duration.ofMinutes(30));
    }

    public String getToken(Long userId) {
        String key = "login:token:" + userId;
        return stringRedisTemplate.opsForValue().get(key);
    }
}
```

Redis 常见用途：

1. 登录 token。
2. 缓存热点数据。
3. 分布式锁。
4. 限流。
5. 排行榜。
6. 消息队列。
7. Session 共享。

------

# 二十二、Spring Security

Spring Security 是 Spring 生态中的安全框架，核心是过滤器链。

基本概念：

1. Authentication：认证，判断你是谁。
2. Authorization：授权，判断你能做什么。
3. SecurityContext：保存当前登录用户。
4. UserDetails：用户信息。
5. PasswordEncoder：密码加密。
6. SecurityFilterChain：安全过滤器链。

依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

基础配置：

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // 关闭 CSRF，前后端分离项目中常见，但生产环境要根据实际情况评估
            .csrf(csrf -> csrf.disable())

            // 配置请求授权规则
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/login", "/register").permitAll()
                .anyRequest().authenticated()
            )

            // 开启表单登录
            .formLogin(form -> form.permitAll());

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        // 使用 BCrypt 加密密码
        return new BCryptPasswordEncoder();
    }
}
```

JWT 场景通常会自定义过滤器：

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        // 1. 从请求头获取 token
        String token = request.getHeader("Authorization");

        // 2. 校验 token
        // 3. 如果合法，把用户信息放入 SecurityContext

        // 放行到下一个过滤器
        filterChain.doFilter(request, response);
    }
}
```

Spring Security 学习难点不在注解，而在理解过滤器链执行顺序。

------

# 二十三、测试

## 1. 单元测试

只测试当前类，不启动 Spring。

```java
class UserServiceTest {

    @Test
    void shouldReturnUserWhenUserExists() {
        // 手动创建依赖
        UserRepository userRepository = new UserRepository();

        // 创建被测试对象
        UserService userService = new UserService(userRepository);

        // 执行测试
        User user = userService.getUser(1L);

        // 断言
        assertEquals("Tom", user.getUsername());
    }
}
```

## 2. Spring Boot 集成测试

启动 Spring 容器。

```java
@SpringBootTest
class UserServiceIntegrationTest {

    @Autowired
    private UserService userService;

    @Test
    void shouldListUsers() {
        List<User> users = userService.listUsers();

        assertFalse(users.isEmpty());
    }
}
```

## 3. MockMvc 测试 Controller

```java
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void shouldGetUsers() throws Exception {
        mockMvc.perform(get("/users"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$").isArray());
    }
}
```

## 4. Mockito 测试 Service

```java
class UserServiceMockitoTest {

    @Test
    void shouldReturnUser() {
        // 创建 Mock 对象
        UserRepository userRepository = Mockito.mock(UserRepository.class);

        // 设置 Mock 行为
        Mockito.when(userRepository.findById(1L))
                .thenReturn(new User(1L, "Tom", 18));

        UserService userService = new UserService(userRepository);

        User user = userService.getUser(1L);

        assertEquals("Tom", user.getUsername());
    }
}
```

测试层级建议：

| 测试类型       | 是否启动 Spring | 速度 | 适用场景         |
| -------------- | --------------- | ---- | ---------------- |
| 单元测试       | 否              | 快   | 业务逻辑         |
| Mockito 测试   | 否              | 快   | 隔离依赖         |
| MockMvc 测试   | 是              | 中   | Controller       |
| 集成测试       | 是              | 慢   | 完整链路         |
| Testcontainers | 是              | 较慢 | 真实数据库/Redis |

------

# 二十四、Actuator 与生产监控

依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

配置：

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
```

常用端点：

| 端点                 | 作用            |
| -------------------- | --------------- |
| /actuator/health     | 健康检查        |
| /actuator/info       | 应用信息        |
| /actuator/metrics    | 指标            |
| /actuator/env        | 环境变量        |
| /actuator/beans      | Bean 信息       |
| /actuator/mappings   | 请求映射        |
| /actuator/prometheus | Prometheus 指标 |

生产环境注意：

1. 不要暴露所有 actuator 端点。
2. `/env`、`/beans`、`/configprops` 可能泄露敏感信息。
3. 需要配合 Spring Security 或网关鉴权。

------

# 二十五、OpenAPI / Swagger 文档

常用 `springdoc-openapi`。

依赖：

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.8.9</version>
</dependency>
```

Controller：

```java
@Operation(summary = "查询用户详情")
@GetMapping("/{id}")
public User getUser(@PathVariable Long id) {
    return userService.getUser(id);
}
```

访问：

```text
http://localhost:8080/swagger-ui.html
```

`springdoc-openapi` 的作用是根据 Spring 配置、类结构和注解生成 OpenAPI 文档。([OpenAPI 3 Library for spring-boot](https://springdoc.org/?utm_source=chatgpt.com))

------

# 二十六、WebFlux 简介

Spring MVC 是 Servlet 阻塞模型，WebFlux 是响应式非阻塞模型。

示例：

```java
@RestController
@RequestMapping("/reactive/users")
public class ReactiveUserController {

    @GetMapping("/{id}")
    public Mono<User> getUser(@PathVariable Long id) {
        // Mono 表示 0 或 1 个异步结果
        return Mono.just(new User(id, "Reactive User", 20));
    }

    @GetMapping
    public Flux<User> listUsers() {
        // Flux 表示 0 到 N 个异步结果
        return Flux.just(
                new User(1L, "A", 18),
                new User(2L, "B", 20)
        );
    }
}
```

使用场景：

1. 高并发长连接。
2. 网关。
3. I/O 密集型服务。
4. 响应式数据库驱动。
5. 流式响应。

不建议刚学 Spring 就深入 WebFlux。先把 Spring MVC 学扎实。

------

# 二十七、Spring Boot 自动配置原理

`@SpringBootApplication` 包含：

```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan
```

其中：

1. `@SpringBootConfiguration`：表示这是配置类。
2. `@ComponentScan`：扫描当前包及子包下的组件。
3. `@EnableAutoConfiguration`：开启自动配置。

自动配置的核心思想：

```text
Spring Boot 启动
  ↓
读取自动配置类
  ↓
根据条件注解判断是否生效
  ↓
如果满足条件，注册对应 Bean
  ↓
应用启动完成
```

典型自动配置条件：

```java
@Configuration
@ConditionalOnClass(DataSource.class)
@ConditionalOnMissingBean(DataSource.class)
public class DataSourceAutoConfiguration {
}
```

含义是：

1. classpath 中有 `DataSource` 类。
2. 用户没有自己定义 `DataSource`。
3. Spring Boot 才自动创建数据源。

这也是为什么你只要引入 starter，很多东西就自动可用了。

------

# 二十八、源码学习路线

不要一开始就啃全部源码。建议按这条路线：

## 1. IoC 容器源码

重点类：

```text
BeanFactory
ApplicationContext
DefaultListableBeanFactory
AbstractApplicationContext
BeanDefinition
BeanDefinitionReader
ClassPathBeanDefinitionScanner
AutowiredAnnotationBeanPostProcessor
CommonAnnotationBeanPostProcessor
```

重点方法：

```text
AnnotationConfigApplicationContext()
refresh()
obtainFreshBeanFactory()
invokeBeanFactoryPostProcessors()
registerBeanPostProcessors()
finishBeanFactoryInitialization()
getBean()
doGetBean()
createBean()
populateBean()
initializeBean()
```

可以理解为：

```java
// 伪代码：Spring 容器启动流程
public void refresh() {
    // 1. 准备环境
    prepareRefresh();

    // 2. 创建或刷新 BeanFactory
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

    // 3. 准备 BeanFactory
    prepareBeanFactory(beanFactory);

    // 4. 执行 BeanFactoryPostProcessor
    invokeBeanFactoryPostProcessors(beanFactory);

    // 5. 注册 BeanPostProcessor
    registerBeanPostProcessors(beanFactory);

    // 6. 初始化事件广播器
    initApplicationEventMulticaster();

    // 7. 初始化非懒加载单例 Bean
    finishBeanFactoryInitialization(beanFactory);

    // 8. 发布容器刷新完成事件
    finishRefresh();
}
```

## 2. Bean 创建源码

重点是 `getBean()`：

```java
// 伪代码：Bean 获取流程
public Object getBean(String beanName) {
    // 1. 先从单例池查找
    Object bean = singletonObjects.get(beanName);

    if (bean != null) {
        return bean;
    }

    // 2. 获取 BeanDefinition
    BeanDefinition bd = getBeanDefinition(beanName);

    // 3. 实例化 Bean
    Object instance = createBeanInstance(bd);

    // 4. 属性填充，也就是依赖注入
    populateBean(instance, bd);

    // 5. 初始化 Bean
    initializeBean(instance);

    // 6. 放入单例池
    singletonObjects.put(beanName, instance);

    return instance;
}
```

真实源码复杂得多，但主线就是这些。

## 3. AOP 源码

重点类：

```text
AnnotationAwareAspectJAutoProxyCreator
AbstractAutoProxyCreator
ProxyFactory
JdkDynamicAopProxy
CglibAopProxy
MethodInterceptor
```

核心流程：

```text
Bean 初始化后
  ↓
BeanPostProcessor 判断是否需要代理
  ↓
如果匹配切面，创建代理对象
  ↓
容器中保存代理对象
  ↓
调用方法时进入拦截器链
  ↓
执行增强逻辑
  ↓
调用目标方法
```

## 4. 事务源码

重点类：

```text
TransactionInterceptor
PlatformTransactionManager
DataSourceTransactionManager
TransactionAspectSupport
TransactionSynchronizationManager
```

事务本质上就是 AOP。

伪代码：

```java
public Object invoke(MethodInvocation invocation) {
    // 1. 开启事务
    TransactionStatus status = transactionManager.getTransaction(definition);

    try {
        // 2. 执行业务方法
        Object result = invocation.proceed();

        // 3. 成功提交
        transactionManager.commit(status);

        return result;
    } catch (Throwable ex) {
        // 4. 异常回滚
        transactionManager.rollback(status);
        throw ex;
    }
}
```

## 5. Spring MVC 源码

重点类：

```text
DispatcherServlet
HandlerMapping
HandlerAdapter
RequestMappingHandlerMapping
RequestMappingHandlerAdapter
HandlerMethodArgumentResolver
HttpMessageConverter
ExceptionHandlerExceptionResolver
```

请求流程：

```text
DispatcherServlet.doDispatch()
  ↓
getHandler()
  ↓
getHandlerAdapter()
  ↓
ha.handle()
  ↓
调用 Controller
  ↓
参数解析
  ↓
返回值处理
  ↓
异常解析
  ↓
响应输出
```

------

# 二十九、Spring 常见面试题

## 1. Spring Bean 是线程安全的吗？

Spring Bean 默认是单例，但单例不等于线程安全。

如果 Bean 没有可变成员变量，通常是线程安全的：

```java
@Service
public class UserService {

    public User getUser(Long id) {
        // 局部变量在线程栈中，线程安全
        User user = new User();
        return user;
    }
}
```

有共享可变状态就可能不安全：

```java
@Service
public class CounterService {

    private int count = 0;

    public void increase() {
        // 多线程下不安全
        count++;
    }
}
```

解决：

1. 避免在单例 Bean 中存请求级状态。
2. 使用局部变量。
3. 使用线程安全类。
4. 使用数据库或 Redis 保证一致性。
5. 必要时使用锁。

## 2. Spring 如何解决循环依赖？

Spring 对单例 Bean 的部分循环依赖可以通过三级缓存解决。

典型循环依赖：

```java
@Service
public class A {
    @Autowired
    private B b;
}

@Service
public class B {
    @Autowired
    private A a;
}
```

三级缓存：

```text
singletonObjects：一级缓存，完整 Bean
earlySingletonObjects：二级缓存，早期 Bean
singletonFactories：三级缓存，ObjectFactory，用于提前暴露代理对象
```

注意：

1. 构造器循环依赖通常无法解决。
2. prototype 循环依赖无法解决。
3. 涉及 AOP 时需要提前暴露代理对象。
4. 实际项目应尽量重构，避免循环依赖。

## 3. `@Autowired` 和 `@Resource` 区别？

| 注解       | 来源              | 默认匹配方式 |
| ---------- | ----------------- | ------------ |
| @Autowired | Spring            | 按类型       |
| @Resource  | Jakarta/Java 规范 | 按名称       |

`@Autowired` 可以配合：

```java
@Qualifier("userRepositoryImpl")
```

`@Resource` 可以指定：

```java
@Resource(name = "userRepositoryImpl")
```

## 4. JDK 动态代理和 CGLIB 区别？

| 代理方式     | 要求           | 原理               |
| ------------ | -------------- | ------------------ |
| JDK 动态代理 | 目标类实现接口 | 基于接口生成代理类 |
| CGLIB        | 不要求接口     | 生成目标类子类     |

限制：

1. final 类不能被 CGLIB 代理。
2. final 方法不能被重写增强。
3. JDK 动态代理只能代理接口方法。

## 5. 为什么 `@Transactional` 会失效？

常见原因：

1. 方法不是 public。
2. 自调用。
3. 异常被捕获。
4. 抛出受检异常但未配置 `rollbackFor`。
5. 没有被 Spring 容器管理。
6. 多线程调用。
7. 数据库不支持事务。

------

# 三十、建议学习顺序

第一阶段：Spring 基础

1. IoC。
2. DI。
3. Bean。
4. Bean 生命周期。
5. 注解配置。
6. Java Config。
7. AOP。
8. 事务。

第二阶段：Spring Boot

1. starter。
2. 自动配置。
3. `application.yml`。
4. profile。
5. 日志。
6. actuator。
7. 打包部署。

第三阶段：Web 开发

1. Spring MVC。
2. REST API。
3. 参数绑定。
4. 参数校验。
5. 全局异常。
6. 文件上传。
7. 拦截器。
8. CORS。
9. 统一响应。
10. 接口文档。

第四阶段：数据库

1. JDBC。
2. 连接池。
3. MyBatis。
4. JPA。
5. 事务。
6. 乐观锁。
7. 悲观锁。
8. 分页。
9. 多数据源。
10. 读写分离。

第五阶段：工程化

1. 分层架构。
2. DTO/VO/Entity。
3. 异常体系。
4. 日志规范。
5. 配置管理。
6. 测试。
7. Docker 部署。
8. CI/CD。

第六阶段：进阶

1. Spring Security。
2. Redis。
3. Cache。
4. MQ。
5. WebSocket。
6. WebFlux。
7. Spring Cloud。
8. 源码。
9. 性能调优。
10. 可观测性。

------

# 三十一、推荐做的练手项目

建议你按难度做 5 个项目。

## 项目 1：用户管理系统

功能：

1. 用户注册。
2. 用户登录。
3. 用户查询。
4. 用户修改。
5. 用户删除。
6. 参数校验。
7. 全局异常。
8. MySQL 存储。

涉及知识：

```text
Spring Boot
Spring MVC
Validation
MyBatis/JPA
事务
统一响应
异常处理
```

## 项目 2：博客系统

功能：

1. 文章发布。
2. 文章列表。
3. 文章详情。
4. 评论。
5. 标签。
6. 分类。
7. Redis 缓存。
8. 阅读数统计。

涉及知识：

```text
REST API
分页
数据库设计
Redis
缓存一致性
拦截器
```

## 项目 3：电商下单系统

功能：

1. 商品管理。
2. 购物车。
3. 下单。
4. 库存扣减。
5. 余额扣减。
6. 订单状态。
7. 事务控制。
8. 并发控制。

涉及知识：

```text
事务传播
锁
并发
数据库隔离级别
Redis
MQ
```

## 项目 4：权限管理系统

功能：

1. 登录。
2. JWT。
3. 用户。
4. 角色。
5. 菜单。
6. 权限。
7. 接口鉴权。

涉及知识：

```text
Spring Security
JWT
RBAC
过滤器链
权限注解
```

## 项目 5：微服务商城

功能：

1. 用户服务。
2. 商品服务。
3. 订单服务。
4. 支付服务。
5. 网关。
6. 注册中心。
7. 配置中心。
8. 链路追踪。

涉及知识：

```text
Spring Cloud
OpenFeign
Gateway
Nacos/Eureka
Config
Resilience4j
Micrometer
Prometheus
```

------

# 三十二、最终学习路线压缩版

你可以按这个顺序执行：

```text
1. Java 基础、Maven、Servlet
2. Spring IoC、DI、Bean、生命周期
3. Spring AOP、事务
4. Spring Boot starter、自动配置、配置文件
5. Spring MVC、REST API、参数校验、异常处理
6. MySQL、JDBC、MyBatis/JPA
7. Redis、缓存、异步、定时任务
8. Spring Security、JWT、权限系统
9. 测试、日志、Actuator、部署
10. Spring 源码：refresh、getBean、AOP、事务、DispatcherServlet
11. Spring Cloud 与微服务
```

最重要的主线不是背注解，而是理解这几个问题：

```text
Spring 如何创建对象？
Spring 如何注入依赖？
Spring 如何管理 Bean 生命周期？
Spring 如何用代理实现 AOP？
Spring 如何用 AOP 实现事务？
Spring MVC 如何把 HTTP 请求映射到 Controller？
Spring Boot 如何自动配置？
Spring 项目如何分层、测试、部署和监控？
```

只要这几个问题能讲清楚，Spring 的主体框架就已经掌握了。