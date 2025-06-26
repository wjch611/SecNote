# 一、知识点

### X-Forwarded-Host 的作用：

##### 工作原理

- 客户端发送请求到代理服务器，请求头中包含 Host: example.com。
- 代理服务器转发请求到后端服务器时，可能会修改 Host 头部为后端服务器的地址（例如 backend-server.local）。
- 代理服务器添加 X-Forwarded-Host: example.com 头部，告知后端服务器客户端请求的原始主机名。
- **后端应用程序**可能需要根据原始主机名**生成绝对 URL**（例如重定向链接或资源路径）。X-Forwarded-Host 提供了必要的信息，确保生成的 URL 与客户端请求的域名一致。

##### 示例

客户端请求：

```
GET /page HTTP/1.1 Host: example.com
```

代理服务器转发到后端：

```
GET /page HTTP/1.1 Host: backend-server.local X-Forwarded-Host: example.com
```

### age/mix-age、hit/miss:

![image-20250522140238284](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250522140238284.png)

如果能够在响应中看到这样的头，就太好了，因为直接给出了：

1. 这是一个缓存页面
2. 缓存每30秒刷新一次、age就是在正计时
3. 想要缓存投毒成功，你必须是缓存刷新后第一个对应缓存键的请求，也就是当返回是miss的时候，说明可能成功了，之后的时间直到再次刷新，访问对应缓存键内容，都hit第一次miss并且存入的响应。

### X-Forwarded-Scheme 的作用:

##### 工作原理

- 客户端发送请求到代理服务器，例如通过 HTTPS 协议（https://example.com）。
- 代理服务器可能终止 SSL/TLS，并以 HTTP 转发到后端服务器。
- 代理服务器添加 X-Forwarded-Scheme: https 头部，告知后端服务器客户端使用的协议。

##### 示例

客户端请求:

```
GET /page HTTP/1.1 Host: example.com
```

代理服务器转发到后端:

```
GET /page HTTP/1.1 Host: backend-server.local X-Forwarded-Scheme: https X-Forwarded-Host: example.com
```

后端服务器根据 X-Forwarded-Scheme: https 知道客户端使用 HTTPS，可以**生成重定向 URL**（如 https://example.com/redirect）

### X-Forwarded-Host与X-Forwarded-Scheme联合->控制重定向：

只有：X-Forwarded-Scheme

![image-20250522150257170](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250522150257170.png)

只有：X-Forwarded-Host

![image-20250522150400948](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250522150400948.png)

联合：

![image-20250522150859430](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250522150859430.png)

### X-Host/X-Forwarded-Host的非标准形式可能也是used&unkeyed

### 响应头：Vary：缓存键

```
HTTP/1.1 200 OK 
Content-Type: text/html 
Vary: Accept-Encoding, User-Agent
```

注意：

​	当缓存键使用了以用户唯一标识，那么如果要中毒这个用户，还需要知道他的这个标识，比如UA

### Param Miner的使用：

扫描结果：

![image-20250522160306348](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250522160306348.png)

or

![image-20250522164136161](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250522164136161.png)

### GET参数不是缓存键：

![image-20250522163502801](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250522163502801.png)

### 在许多GET参数是缓存键情况下使用param miner找到不是缓存键的参数：

![image-20250522164633110](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250522164633110.png)

### X-Cache-Key与Vary:

前者是实际的缓存键、后者是抽象的缓存键。

### 发现了一个参数，可被利用，但是keyed：

![image-20250522171901085](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250522171901085.png)

#### 一、参数隐藏

##### 如何利用：

使用param miner找一个unkeyed的参数：

![image-20250522172238438](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250522172238438.png)

##### 知识点：

1. 重复参数后端取最后面的

   ```
   GET /js/geolocate.js?callback=setCountryCookie&callback=1
   ```

2. 使用;分隔的缓存视为一个参数

   ```
   GET /js/geolocate.js?callback=setCountryCookie&utm_content=foo;callback=alert(1)
   ```

#### 二、FAT GET

![image-20250522173537669](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250522173537669.png)

原因：

​	缓存看上面的，后端看下面的

