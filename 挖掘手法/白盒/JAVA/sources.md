



[TOC]

# 一、原生

## 0、目录

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

## 1、Listen、Filter、Servlet

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



