### 筛选存在http服务的域名

#### 一、使用

```
.\go_httpx.exe -l .\result.txt -threads 10 -rate-limit 30 -random-agent -timeout 5 -retries 2 -follow-redirects -o result1.txt
```

| 参数                       | 含义                             | 说明                                             |
| -------------------------- | -------------------------------- | ------------------------------------------------ |
| `.\httpx.exe`              | 执行 httpx 程序                  | 当前目录下的可执行文件                           |
| `-l .\combined_result.txt` | 读取目标列表                     | 从 `combined_result.txt` 这个文件读取 URL 或域名 |
| `-threads 10`              | 开启 20 个并发线程               | 控制并发请求数量，20 比较稳健，既快又不容易被封  |
| `-rate-limit 30`           | 限制每秒最多请求 50 次           | 防止过快发包被目标服务器封锁                     |
| `-random-agent`            | 使用随机的 User-Agent            | 模拟浏览器行为，避免被识别为工具流量             |
| `-timeout 5`               | 请求超时时间设为 10 秒           | 防止目标卡死，太短可能误判，太长会拖慢整体速度   |
| `-retries 2`               | 失败请求最多重试 2 次            | 网络不稳时容错，避免漏掉短暂掉线的站点           |
| `-follow-redirects`        | 自动跟随 301/302 跳转            | 这样能获取最终跳转页面信息，比如跳转到真实后台   |
| `-o result.txt`            | 将结果输出到 `result.txt` 文件中 |                                                  |

