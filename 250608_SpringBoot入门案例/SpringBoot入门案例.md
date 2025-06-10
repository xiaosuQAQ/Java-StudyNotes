## 引言

通过这部分内容的学习

案例参考黑马程序员的javaweb课程。

虽然之前的Servlet可以自动解析请求行，但是对每个URL的解析都需要一个类来实现（这个类继承自HttpServlet），针对不同的请求类型需要重写对应的方法（例如doGet、doPost），一个简单的模板见下。而在实际开发中会有非常多的URL，如果为每个网页都构建一个HttpServlet类，那么会导致项目结构冗余，而SpringBoot可以通过注解的方式简化功能的实现，因此成为了目前最流行的框架之一。

```java
@WebServlet(urlPatterns = "url")
public class ForwardServlet extends HttpServlet {
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) {
        // 响应Get请求
    }
}
```

## SpringBoot入门程序

在IDE中创建一个SpringBoot、选择Maven并勾选需要的依赖，会自动创建出一个标准的代码库。

![项目目录](imgs/项目目录.png)

其中会自动创建引导类SpringbootBaseApplication，而上方有 ``@SpringBootApplication``注解，这个注解表示下方的类是主程序类，用于启动SpringBoot应用。

接下来来编写一个控制类，用于处理网页根目录"/"下的请求，代码见下：

```java
@RestController
public class HelloController {
    @RequestMapping("/")
    public String hello(String name){
        System.out.println("name: " + name);
        return "hello, " + name;
    }
}
```

其中有两个新的注解，首先 ``RestController``表示类中的所有方法会直接返回对象、服务器底层序列化为JSON文件。而 ``RequestMapping``表示通用的请求映射，当访问其中指定的URL时，由注解下面的方法或者类处理。和Servlet中需要继承HttpServlet、重写doGet方法相比，SpringBoot只需要注解就可以实现相同的功。（另外Servlet开发要么需要将打包文件运行在tomcat服务器中，要么需要在IDE中引入tomcat依赖并创建tomcat对象，总之开发很复杂，而SpringBoot中由内嵌的tomcat服务器，因此只需要引入SpringBoot依赖，由于Maven的依赖传递就可以自动配置好tomcat服务器依赖并且运行，进一步简化了开发过程）。

## 案例2

这个案例参考[黑马讲义](https://heuqqdmbyk.feishu.cn/wiki/MQ95wDTtji6ob6kiRkyc9jamnwg)完成，包含前后端开发的内容，主要思路是浏览器发送静态资源的访问请求（也就是html页面），html页面中会通过钩子函数尝试访问非静态资源（可以理解为数据库），接着springbbot框架会找到这个非静态资源对应的方法，由这个方法返回JSON格式的数据，从而交由静态资源显示。

这个案例中的前端程序已经提供了，因此需要完成**服务器响应非静态资源的部分**。

和上面的 ``HelloController``类似，使用 ``@RestController``注解类、使用 ``@RequestMapping``注解方法（URL为 ``/list``），使用txt文件模拟数据库。

```java
@RestController
public class UserController {
    @RequestMapping("/list")
    public List<User> list(){
        //1.加载并读取文件
        InputStream in = this.getClass().getClassLoader().getResourceAsStream("user.txt");
        ArrayList<String> lines = IoUtil.readLines(in, StandardCharsets.UTF_8, new ArrayList<>());
        //2.解析数据，封装成对象 --> 集合
        List<User> userList = lines.stream().map(line -> {
            String[] parts = line.split(",");
            Integer id = Integer.parseInt(parts[0]);
            String username = parts[1];
            String password = parts[2];
            String name = parts[3];
            Integer age = Integer.parseInt(parts[4]);
            LocalDateTime updateTime = LocalDateTime.parse(parts[5], DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
            return new User(id, username, password, name, age, updateTime);
        }).collect(Collectors.toList());
        //3.响应数据, 直接返回userList，底层会自动转化为JSON格式
        //return JSONUtil.toJsonStr(userList, JSONConfig.create().setDateFormat("yyyy-MM-dd HH:mm:ss"));
        return userList;
    }
}
```

## 分层解耦

上面的 ``Controller``类中同时处理了数据访问、处理数据、响应返回，而实际开发中需要考虑代码的复用性，并且尽量遵循“单一指责原则”，也就是类/接口/方法尽量只完成一件事，所以要将上面的处理进行拆分。

业务中也通常将处理逻辑划分为：

- 接收请求、响应数据，Controller，BS架构中和浏览器最近；
- 逻辑处理，Service，BS架构中处于中间；
- 数据访问，Dao，BS架构中和数据库最近；

那么将上面的代码按照处理逻辑抽离，项目包的文件目录见下：

```
com.test
|	SpringbootWebApplication
|_______controller
|	|   UserController.java
|
|_______dao
|	|_______impl
|	|	| UserDaoImpl.java
|	|    UserDao.java
|
|_______pojo
|	|   User.java
|
|_______service
|	|_______impl
|	|	| UserServiceImpl.java
|	|    UserService.java
```

按照逻辑将代码拆分为3部分，其中存在依赖关系：UserController -> UserService -> UserDao，即UserController中需要UserService得到处理后的数据，UserService需要通过UserDao获取数据，朴素的做法是在类中直接new一个依赖的类的对象，但是这种方法与“高内聚、低耦合”相悖，而SpringBoot中的控制反转IoC和依赖注入DI技术可以优雅地解决这个问题。

（下面是朴素做法的展示)

```java
@RestController
public class UserController {
    private UserService userService = new UserServiceImpl();
    @RequestMapping("/list")
    public List<User> list(){
        //1.调用Service
        List<User> userList = userService.findAll();
        //2.响应数据
        return userList;
    }
}
```

### 控制反转IoC和依赖注入DI使用

顾名思义将对象的控制权进行反转，由程序交付给容器，程序中不再显式声明对象，而是存储在容器中；在目标类上添加注解 ``@Component``实现。

使用时通过依赖注入，从容器中获取指定的对象；通过 ``@Autowired``注解，告诉代码从容器中自动找到对应的对象。

上面的代码可以改写为：

```java
@Component    // 交给容器管理
public class UserServiceImpl implements UserService {
    @Autowired
    private UserDao userDao;
    @Override
    public List<User> findAll() {
        List<String> lines = userDao.findAll();
        List<User> userList = lines.stream().map(line -> {
            String[] parts = line.split(",");
            Integer id = Integer.parseInt(parts[0]);
            String username = parts[1];
            String password = parts[2];
            String name = parts[3];
            Integer age = Integer.parseInt(parts[4]);
            LocalDateTime updateTime = LocalDateTime.parse(parts[5], DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
            return new User(id, username, password, name, age, updateTime);
        }).collect(Collectors.toList());
        return userList;
    }
}
```

```java
@RestController
public class UserController {
    @Autowired     // 依赖注入，自动从容器中获得对象
    private UserService userService;
    @RequestMapping("/list")
    public List<User> list(){
        List<User> userList = userService.findAll();
        return userList;
    }
}
```

### 控制反转、依赖注入的原理

除了Component之外，还有在三个逻辑层中特有的3个注解，Controller、Service、Repository。

上面的注解指定将类交给容器管理，但是还需要 ``@ComponentScan``扫描才可以生效（``@SpringBootApplication``启动注解中已经包含了这个扫描，默认作用范围是启动类所在包及子包）

依赖注入的注解可以放置到多个位置中（属性、构造方法、setter方法），注意如果容器中管理多个相同的类对象，要么指定默认的，要么在依赖注入时指定需要的，否则会报错）

- Primary注解，指定默认
- 添加Qualifier("xxxImpl")指定注入的Bean，spring提供的，按照类型
- 使用Resource(name="xxxImpl")指定，JDK提供的，根据名称注入

## SpringBoot常用注解

|            注解            | 作用                                                 | 说明                                                    |
| :------------------------: | ---------------------------------------------------- | ------------------------------------------------------- |
| ``@SpringBootApplication`` | 标记主程序**类**，用于启动Spring Boot应用      | 包含了 ``@Component``                                   |
|    ``@RestController``    | 表示**类**中方法都直接返回对象、并序列化为JSON | `@Controller` + `@ResponseBody`                     |
|    ``@RequestMapping``    | 通用的请求映射，类/方法                              | ``@RequestMapping("url")``                              |
|      ``@GetMapping``      | 只处理get请求                                        | 等价于 `@RequestMapping(method = RequestMethod.GET)`  |
|      ``@PostMapping``      | 只处理post请求                                       | 等价于 `@RequestMapping(method = RequestMethod.POST)` |
|       ``@Component``       | 标记交给容器处理                                     |                                                         |
|       ``@Autowired``       | 自动装配，放置位置3种                                |                                                         |
