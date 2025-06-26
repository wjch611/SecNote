### 一、FUZZ指定域名

```
Get-Content .\ua.txt | Sort-Object {Get-Random} | .\ffuf.exe -w -:UA -w .\subnames_next.txt:SUB -H "User-Agent: UA" -u "https://SUB.slack.com" -t 10 -rate 20 -timeout 5 -p 0.3 -o result.txt -of json -mc all
```

| 参数                         | 类型                      | 含义                                  | 示例值/说明                  |
| ---------------------------- | ------------------------- | ------------------------------------- | ---------------------------- |
| `Get-Content .\ua.txt`       | PowerShell                | 读取 `ua.txt` 中所有 User-Agent       | 一行一个 UA                  |
| `                            | Sort-Object {Get-Random}` | PowerShell                            | 打乱顺序（模拟 `shuf`）      ||
| `-w -:UA`                    | ffuf                      | 从标准输入读取 fuzz 字典，别名为 `UA` | `-` 表示从管道读             |
| `-w .\subnames_next.txt:SUB` | ffuf                      | 指定子域名字典文件，别名为 `SUB`      | 如 `cdn`, `app`, `mail` 等   |
| `-H "User-Agent: UA"`        | ffuf                      | 自定义请求头，把变量 `UA` 作为值      | 每个请求使用随机 UA          |
| `-u "https://SUB.slack.com"` | ffuf                      | 要测试的 URL 模板，将 `SUB` 替换      | 例如最终变为 `cdn.slack.com` |
| `-t 5`                       | ffuf                      | 设置并发线程数为 5                    | 控制扫描速度，防止过载       |
| `-rate 10`                   | ffuf                      | 每秒最多请求数为 10                   | 限速，有助绕过封锁           |
| `-timeout 5`                 | ffuf                      | 单个请求超时时间为 5 秒               | 避免卡死或长时间等待         |
| `-p 0.3`                     | ffuf                      | 每个请求附加 0~0.3 秒随机延迟         | 用于降低请求特征             |
| `-o result.csv`              | ffuf                      | 输出文件名                            | 保存扫描结果                 |
| `-of csv`                    | ffuf                      | 输出格式为 CSV                        | 适合进一步处理               |

### 注意：

1. 一定要https，否则，http -> https 那全都是 301，但是https可能是 404 或者 noalive
2. 默认是过滤掉40x的，但是40x也不能疏忽，-mc指定显示，all显示所有存在http的

### 二、FUZZ目录/路径

```
Get-Content .\ua.txt | Sort-Object {Get-Random} | .\ffuf.exe -w -:UA -w .\path_dict\dsstorewordlist.txt:PATH -H "User-Agent: UA" -u "https://slack-imgs.com/PATH" -t 10 -rate 20 -timeout 5 -p 0.3 -o path_result.txt -of json -mc all -fc 400 404
```

- 扫描路径不像扫域名，需要过滤一些，不存在的
