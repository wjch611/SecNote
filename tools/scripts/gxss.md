### 一、一句话描述

**Gxss 是一款基于上下文反射位置的 XSS 漏洞辅助探测工具，能快速发现 URL 中参数插入点的潜在反射点。**

------

### 二、Gxss 原理简述

Gxss 的核心原理可以归纳为三步：

1. **注入测试 Payload（默认是 `gxssfuzz`）**：
   - Gxss 向目标 URL 注入特定测试值（如参数中插入 `gxssfuzz`）；
   - 支持通过 `-p` 指定 Payload，常配合浏览器 UA 模拟真实访问。
2. **读取响应内容**：
   - 检查响应中是否能反射出原始 Payload；
   - 会标注出反射的位置（如 body、script、attribute、inline script 等）；
   - 这样可以初步判断是否是 XSS 注入点（但**不判断是否能成功执行脚本**，仅做“是否反射”的辅助探测）。
3. **输出反射位置和上下文信息**：
   - 帮助渗透人员判断是否值得继续 fuzz；
   - 输出结构清晰，适合批量扫描。

### 三、配合脚本

```
python .\run_gxss.py
```

| 功能                 | 实现方式                       |
| -------------------- | ------------------------------ |
| 随机选择 User-Agent  | 从 `ua.txt` 中随机抽取         |
| 批量读取 URL         | 来自 `merged_results.txt`      |
| 写入到 `urlpipe.txt` | 模拟标准输入传参               |
| 调用 `Gxss.exe`      | 使用 `subprocess.run` 启动     |
| 处理输出             | 清洗空行，只保留有效结果       |
| 实时输出 + 进度显示  | `print()` 结合百分比和当前进度 |
| 保存有效结果         | 写入 `gxss_result.txt`         |
| 资源清理             | 自动删除临时文件               |

### 四、输出

![image-20250420171325508](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250420171325508.png)

![image-20250420171445951](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250420171445951.png)

### 五、功能验证

结论：

​	可以检测反射xss、反射domxss，但是纯dom检测不出来。

区别：

​	纯dom的payload不会直接出现在返回包中，是js获得这个参数，再经过浏览器执行js，然后再插入到dom中

**例：**

```
https://0a1e00b10391f44f829862d800f800c7.web-security-academy.net/feedback?returnPath=123213
```

返回：

```
<div class="is-linkback">
    <a id="backLink">Back</a>
</div>

<script>
    $(function () {
        const params = new URLSearchParams(window.location.search);
        const returnPath = params.get('returnPath');
        $('#backLink').attr("href", returnPath);
    });
</script>
```

浏览器渲染：

![image-20250421111518349](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250421111518349.png)
