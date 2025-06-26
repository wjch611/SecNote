### 使用示例

1. 使用默认 SOCKS5 端口(1080)转发到本地 HTTP 代理(8080):

   ```bash
   python s2h.py --proxy-host 127.0.0.1 --proxy-port 8080
   ```

2. 监听所有网络接口并使用自定义 SOCKS5 端口:

   ```bash
   python s2h.py --socks-host 0.0.0.0 --socks-port 1081 --proxy-host proxy.example.com --proxy-port 8080
   ```

3. 使用远程 HTTP 代理:

   ```bash
   python s2h.py --proxy-host proxy.company.com --proxy-port 3128
   ```

### 缺点：

1. 不支持dns匿名，上游的socket5不要使用远程dns解析
