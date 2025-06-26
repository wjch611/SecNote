### 功能：

1. 命令行下指定一个脚本，走指定的socket5代理

### 万能BP代理：

1. 如果是python脚本，首先添加REQUESTS_CA_BUNDLE 环境变量，不然ssl验证不过（注意：证书需要pem格式）
2. S2H代理打开：监听socket5 A -> BP
3. 使用proxychains_win将流量转发到 socket5 A端口

### 配置：

1. 代理配置

   ```
   [ProxyList]
   socks5 localhost 1080
   ```

2. fake dns配置（也就是dns匿名配置）

   ```
   proxy_dns //启用后，DNS 请求将通过代理（通常是 SOCKS5）在远程进行解析，从而防止 DNS 泄露。
   ```

   ```
   remote_dns_subnet 224
   当 proxy_dns 启用时，proxychains 使用 指定的子网 IP 地址范围（例如 224.x.x.x）来伪装 DNS 响应的结果。
   ```

   

