[TOC]

# 一、php基础

#### $_SESSION：

1. 只有在 session_start()开启了才有用

#### $_FILES：

当你在表单中用 `<input type="file" name="fileToUpload">` 上传文件时，PHP 会自动把上传的文件信息放到 `$_FILES['fileToUpload']` 这个数组中，它是一个**关联数组**，结构如下：

```
php复制编辑$_FILES["fileToUpload"] = array(
    "name" => "example.jpg",        // 用户上传的文件名
    "type" => "image/jpeg",         // 浏览器声明的 MIME 类型（不可信）
    "tmp_name" => "/tmp/php1234.tmp", // 文件临时路径（系统自动生成）
    "error" => 0,                   // 错误代码，0 表示无错
    "size" => 123456                // 文件大小（字节）
);
```

#### 防护手段：

1. php.ini配置

   ```
   安全模式 safe_mode 命令执行函数会被禁用
   *路径访问 open_basedir 限制文件操作安全（遍历等）
   *禁用函数 disable_function 升级版安全模式，自定义限制函数
   *魔术引号转义 magic_quotes_gpc 同理下面的sql过滤第一个函数
   数据库访问次数 max_connections 防止数据库爆破
   禁用远程执行 allow_url_include allow_url_fopen远程包含开关等
   *安全会话管理 session.cookie_httponly session.cookie_secure
   防止跨站脚本攻击（XSS）和中间人攻击（MITM）
   ```

2. 调用内置过滤函数

   ```
   filter_var()使用特定的过滤器过滤一个变量
   FILTER_SANITIZE_STRING 过滤器可以过滤HTML标签和特殊字符
   FILTER_SANITIZE_NUMBER_INT 过滤器可过滤非整数字符
   FILTER_SANITIZE_URL 过滤器用于过滤URL中的非法字符 
   FILTER_VALIDATE_EMAIL 过滤器来验证电子邮件地址的有效性
   ```

3. 编写并包含调用过滤文件

   ```
   1、关键内容检测（黑白名单）
   2、模仿流量检测（基于规则或AI算法）
   ```

# 二、原生安全

## 1、弱比较（in_array、array_search、==、switch）

#### in_array:

```
in_array(mixed $needle, array $haystack, bool $strict = false)

`$needle`：你要查找的值。

`$haystack`：被搜索的数组。

`$strict`（默认是 `false`）：是否启用严格类型比较（`===`）。
```

##### 案例：

in_array(7shell.php，range(1,24))；

7shell.php会强制转换为7，从而绕过文件名检测！

## 2、htmlspecialchars可被XSS伪协议绕过

[**htmlspecialchars** ](http://php.net/manual/zh/function.htmlspecialchars.php)：(PHP 4, PHP 5, PHP 7)

**功能** ：将特殊字符转换为 HTML 实体

**定义** ：string **htmlspecialchars** ( string `$string` [, int `$flags` = ENT_COMPAT | ENT_HTML401 [, string`$encoding` = ini_get("default_charset") [, bool `$double_encode` = **TRUE** ]]] )

```
& (& 符号)  ===============  &amp;
" (双引号)  ===============  &quot;
' (单引号)  ===============  &apos;
< (小于号)  ===============  &lt;
> (大于号)  ===============  &gt;
```

## 3、文件

#### 包含：class_exists()->__autoload、include/include_once、require/require_once

鸡肋处：

1. 只能包含存在的文件，因此还需要借助文件写入、文件上传等漏洞照成rce（**即使包含txt，但是里面包含php代码，依然能执行**！）
1. 与上传图片马的配合，只要上传到可执行目录，并且可以包含这个就行

#### 删除：unlink 

#### 读取：file_get_contents

#### 下载：readfile

#### 写入：file_put_contents 

#### 上传：move_uploaded_file

## 4、parse_str、$$、$_GLOABLS、extract-> 变量覆盖

#### 一、parse_str

**功能** ：parse_str的作用就是解析字符串并且注册成变量，它在注册变量之前不会验证当前变量是否存在，所以会直接覆盖掉当前作用域中原有的变量。

**定义** ：`void parse_str( string $encoded_string [, array &$result ] )`

如果 **encoded_string** 是 URL 传入的查询字符串（query string），则将它解析为变量并设置到当前作用域（如果提供了 result 则会设置到该数组里 ）。

#### 二、extract的EXTR_OVERWRITE

`extract()` 函数的作用是：
**将关联数组的键作为变量名，值作为变量值，导入到当前的符号表中**

```
$array = ['name' => 'John', 'age' => 30];
extract($array);

echo $name; // 输出: John
echo $age;  // 输出: 30
```

```
extract(array $array, int $flags = EXTR_OVERWRITE, string $prefix = ""): int
```

1. **`$array`**：必需，要处理的关联数组
2. **`$flags`**：可选，处理冲突的方式（见下文）
3. **`$prefix`**：可选，为变量名添加前缀

| 常量             | 值   | 说明                         |
| :--------------- | :--- | :--------------------------- |
| `EXTR_OVERWRITE` | 0    | 默认，如有冲突则覆盖已有变量 |

## 5、代码执行（eval，assert，preg_replace(php<5.6)，create_function、文件包含函数)

> [**preg_replace**](http://php.net/manual/zh/function.preg-replace.php)：(PHP 5.5)
>
> **功能** ： 函数执行一个正则表达式的搜索和替换
>
> **定义** ： `mixed preg_replace ( mixed $pattern , mixed $replacement , mixed $subject [, int $limit = -1 [, int &$count ]] )`
>
> 搜索 **subject** 中匹配 **pattern** 的部分， 如果匹配成功以 **replacement** 进行替换

- **$pattern** 存在 **/e** 模式修正符，允许代码执行
- **/e** 模式修正符，是 **preg_replace() ** 将 **$replacement** 当做php代码来执行

## 6、命令执行（exec，system，passthru，popen，shell_exec，pcntl_exec）

## 7、双写绕过（preg_replace、str_replace）

[str_replace ](http://php.net/manual/zh/function.str-replace.php)：(PHP 4, PHP 5, PHP 7)

**功能** ：子字符串替换

**定义** ： `mixed str_replace ( mixed $search , mixed $replace , mixed $subject [, int &$count ] )`

该函数返回一个字符串或者数组。如下：

str_replace(字符串1，字符串2，字符串3)：将字符串3中出现的所有字符串1换成字符串2。

str_replace(数组1，字符串1，字符串2)：将字符串2中出现的所有数组1中的值，换成字符串1。

str_replace(数组1，数组2，字符串1)：将字符串1中出现的所有数组1一一对应，替换成数组2的值，多余的替换成空字符串。

![image-20250626183646859](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250626183646859.png)

例如攻击者使用payload：**....//** 或者 **..././** ，在经过程序的 **str_replace** 函数处理后，都会变成 **../**

## 8、继续执行漏洞

![image-20250626185209758](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250626185209758.png)

关键在header，它不会阻止程序继续执行到，11行。

## 10、htmlentities 绕过过滤

> [htmlentities](http://php.net/manual/zh/function.htmlentities.php) — 将字符转换为 HTML 转义字符
>
> ```
> string htmlentities ( string $string [, int $flags = ENT_COMPAT | ENT_HTML401 [, string $encoding = ini_get("default_charset") [, bool $double_encode = true ]]] )
> ```
>
> 作用：在写PHP代码时，不能在字符串中直接写实体字符，PHP提供了一个将HTML特殊字符转换成实体字符的函数 htmlentities()。

注：**htmlentities()** 并不能转换所有的特殊字符，是转换除了空格之外的特殊字符，且单引号和双引号需要单独控制（通过第二个参数）。第2个参数取值有3种，分别如下：

- ENT_COMPAT（默认值）：只转换双引号。
- ENT_QUOTES：两种引号都转换。
- ENT_NOQUOTES：两种引号都不转换。

## 11、addslashes（配合反斜杠截断）

[addslashes](http://php.net/manual/zh/function.addslashes.php) — 使用反斜线引用字符串

```
string addslashes ( string $str )
```

作用：在单引号（'）、双引号（"）、反斜线（\）与 NUL（ **NULL** 字符）字符之前加上反斜线。

##### 绕过思路：

1. 字符型sql，之前存在n个输入字符截断，让它截断\后的"/'
2. 后续还存在url解码，并且继续深入过滤，并且存在使用\绕过的过滤，使用这个\ + 双url编码

## 12、$_SERVER['PHP_SELF']

`$_SERVER['PHP_SELF']` 是一个 **超级全局变量**，表示当前执行脚本的路径（相对于网站根目录）。

例如：

```
<?php echo $_SERVER['PHP_SELF']; ?>
```

- 访问：http://www.test.com/index.php
   输出：`/index.php`
- 访问：http://www.test.com/index.php/foo/bar
   输出：`/index.php/foo/bar`

这就是关键 —— 当访问路径中包含 **PATH_INFO**，`PHP_SELF` 会连同 PATH_INFO 一起输出！

##### 修复：

$_SERVER['SCRIPT_NAME']` 始终只返回 `/index.php

## 13、is_array+implode -> sqli

![image-20250627104235094](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250627104235094.png)

## 14、反序列化

#### 序列化数据识别：

```
<?php
class Example {
    public $public = "public data";
    protected $protected = "protected data";
    private $private = "private data";
}

$obj = new Example();
echo serialize($obj);
?>
```

```
O:7:"Example":3:{s:6:"public";s:11:"public data";s:12:"\0*\0protected";s:14:"protected data";s:19:"\0Example\0private";s:13:"private data";}

注意：
	序列化数据中，对于对象是三者属性，规则是不同的，如果在php7.1+，可以忽视，全部按照public来写，不然需要按照各个属性对应的来写，不然反序列化解析不了
```

#### Source:

```
原生:
	1. unserialize
	2. Phar（类似jar包）
```

![image-20250713164024119](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250713164024119.png)

#### Jump:（都是对象中的函数）

```
原生:
	无条件：
		1. __wakeup
			-> 当使用 unserialize() 时被调用，用于反序列化后的初始化操作
		2. __destruct
			-> 销毁对象或脚本结束时被调用
	有条件：
            __call	调用不可访问或不存在的方法时被调用
            __callStatic	调用不可访问或不存在的静态方法时被调用
            __clone	进行对象 clone 时被调用，用来调整对象的克隆行为
            __construct	构建对象时被调用
            __debugInfo	当调用 var_dump() 打印对象时被调用（控制输出内容）适用于 PHP 5.6+ 版本
            __get	读取不可访问或不存在属性时被调用
            __invoke	当以函数方式调用对象时被调用 ($obj())
            __isset	对不可访问或不存在的属性调用 isset() 或 empty() 时被调用
            __set	当给不可访问或不存在属性赋值时被调用
            __set_state	当调用 var_export() 导出类时被调用，返回值作为导出结果（静态方法）
            __sleep	当使用 serialize() 时被调用，用于指定要序列化的属性
            __toString	当对象被当作字符串使用时被调用 (echo $obj)
            __unset	对不可访问或不存在的属性进行 unset 时被调用
```

#### Sink：

```
__call	调用不可访问或不存在的方法时被调用
call_user_func	一般 PHP 代码执行都会选择这里
call_user_func_array	一般 PHP 代码执行都会选择这里
```

#### 反序列化链：

```
PHP原生类：（与PHP版本无关，主要看php.ini是否开启了对应类）
	1. Error/Exception类
		入口:	unserialize
		jump: __toString
		Sink: XSS
	2. SoapClient
		入口:	unserialize
		jump: __Call
		Sink: SSRF
```

## 15、SSRF

当 `allow_url_fopen=On`（默认开启）时，这些函数可通过 `http://`、`ftp://` 等协议发起请求：

- **`file_get_contents()`**

  ```
  $content = file_get_contents($_GET['url']); // 用户输入直接控制URL
  ```

- **`fopen()` / `fread()` / `fgets()`**

  ```
  $handle = fopen($_POST['url'], 'r');
  ```

# 三、TP框架

### 1、如何判断版本

```
根目录/composer.lock
```

### 2、配置文件：application/config.php、application/database.php

```
#调试开关

'app_debug'              => true,
// 应用Trace
'app_trace'              => true,
```

```
#视图层模板配置

'template'               => [
    // 模板引擎类型 支持 php think 支持扩展
    'type'         => 'Think',
    // 默认模板渲染规则 1 解析为小写+下划线 2 全部转换小写
    'auto_rule'    => 1,
    // 模板路径
    'view_path'    => '',
    // 模板后缀
    'view_suffix'  => 'html',
    
重要：
1. 模板路径（
	1. 如果固定目录，比如/tamples，那模板文件就需要放在这下面
	2. 如果留空，在调用渲染模板的代码对应的controller层的同层View目录下对应控制器名目录下的对应渲染的文件
2. 模板后缀（也就是渲染的目标文件类型）
```

### 3、路由关系

```
http://localhost/index.php/Index/Blog/read
=http://localhost/index.php?s=/Index/Blog/read
旧版本TP支持：(3.*)
http://localhost/index.php?s=Index&m=Blog&a=read

其中：/Index/Blog/read = PATH_INFO
```

- Index： 模块，在application下的目录名


- Blog：控制器，在控制器目录下的文件名/在该文件下的class名


- read：方法，在控制器类下的方法名

### 4、控制器层：application\index\controller

### 5、视图层：application\index\view

1. 配置config.php的模板

2. 在控制层编写调用渲染模板

   ```php
   use think\Controller;
   
   class Index extends \think\Controller
   {
       public function index()
       {
           // 模板变量赋值
           $this->assign('name','ThinkPHP');
           $this->assign('email','thinkphp@qq.com');
           // 或者批量赋值
           $this->assign([
               'name'  => 'ThinkPHP',
               'email' => 'thinkphp@qq.com'
           ]);
           // 模板输出
           return $this->fetch('index');
       }
   }
   ```

3. 找到对应模板目录，创建fetch的模板文件，类型参考配置文件（**如果fetch没有指定，那么，里面的默认是方法名**）

   ![image-20250628164627272](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250628164627272.png)

4. 执行fetch最终效果是，执行到文件包含函数

### 6、内置方法


#### 获取参数差异:

1. 使用助手函数

   ```
   input('get.id');
   ```

2. 使用封装的Request:instance函数

   ```
   Request::instance()->get('id'); // 获取某个get变量
   ```

#### 过滤差异:

```
input('get.id/d');
input('post.name/s');
input('post.ids/a');
Request::instance()->get('id/d');
```

| 修饰符 | 作用                 |
| :----- | :------------------- |
| s      | 强制转换为字符串类型 |
| d      | 强制转换为整型类型   |
| b      | 强制转换为布尔类型   |
| a      | 强制转换为数组类型   |
| f      | 强制转换为浮点类型   |

#### 内置安全写法:

1. TP写法，防护SQLI

   ```
   Db::table('think_user')->where('info$.email','thinkphp@qq.com')->find();
   ```

## 7、框架安全

# 四、Laravel框架

### 1、如何判断版本

```
根目录/composer.lock
```

### 2、配置文件

````
根目录/config/
````

### 3、控制器层：app\Http\Controllers

### 4、视图层：resources\views

### 5、路由关系：routes

### 6、内置方法

---





