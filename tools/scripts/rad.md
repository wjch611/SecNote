## 一、使用

```
 .\rad_windows_amd64.exe -t slackb.com -text-output result.txt
```

**原理**：rad 内部用 headless 浏览器打开目标页面，会自动点击跳转，提取其中的 JS、XHR 请求。

**优点**：

- 无需交互，全自动
- 快速拿一部分动态路径

**缺点**：

- 测不到你“特定交互”触发的 JS（比如点击评论、打开个人中心才发请求）

## 二、记录范围

| 请求类型                       | 能否被 rad 抓到？ | 说明                                                        |
| ------------------------------ | ----------------- | ----------------------------------------------------------- |
| `GET` 请求的 URL 参数          | ✅ 会被记录        | 例如 `https://example.com/page?id=1`                        |
| 动态构造的 URL（JS 拼接）      | ✅ 会记录          | 比如 `xhr.open("GET", "/api/data?id=1")`                    |
| URL 重定向跳转                 | ✅ 记录            | 前端点击或跳转触发的                                        |
| `POST` 请求的 URL（路径）      | ✅ 记录            | 例如 `POST /login`，**只记录路径**，**不记录 body**         |
| `POST` 请求的参数（body data） | ❌ **不会记录**    | POST form 表单、XHR 的 JSON body、urlencoded 等都不会被记录 |
| Header 参数（如 token）        | ❌ 不记录          | rad 专注于 URL 抓取，不分析请求头                           |

## 三、缺陷

1. 记录结果太混
   1. 指定的域名的带/不带参数的url
   2. 非指定域名的带/不带参数的url（跳转记录）

后续利用需要筛选！

