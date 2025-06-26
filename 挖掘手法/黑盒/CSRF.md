# 一、自我理解

1. 鉴权看有没有samesite
2. 看敏感操作有没有二次验证
3. 看有请求体中没有X-CSRF-token
4. 看是不是**简单请求**
   1. 表单只能发送简单请求
   2. 非简单请求受COSR策略影响CSRF的结果
5. 攻击看看有没有referer校验
   1. meta空referer绕过一下
   2. 利用同站的url重定向绝对可绕过referer
6. Origin校验



