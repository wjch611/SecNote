##  一、原理

其实它并不是传统意义上爬网页的那种“爬虫”（比如 Scrapy/BeautifulSoup 那种点击爬页面）。

### 它更像是一种 **“被动收集器”**，数据来源主要有两种：

#### ✅ 1. 使用在线搜索引擎 + GitHub dorks

它通过调用一些搜索引擎（比如 Bing、Google、GitHub、Wayback Machine）来搜索目标站点的历史 URL：

```
site:example.com inurl:?   // 查找带参数的链接
```

这些搜索语法和“Google hacking dork”类似，所以你会发现：

> 它并不是爬页面，而是抓搜索引擎查到的历史 URL。

------

#### ✅ 2. 利用历史归档网站，比如 Wayback Machine（web.archive.org）

- 调取目标网站的历史快照；
- 分析其中出现过的 URL；
- 从中提取参数形式的链接。

## 二、缺点

它只能找出前端能访问到的参数，也就是 GET 请求的 URL 形式；**对于 POST 请求参数、隐藏参数（JS生成或 URL 加密）无能为力**

| 类型              | 能抓到？   | 说明                       |
| ----------------- | ---------- | -------------------------- |
| GET 请求参数      | ✅ 会抓到   | 从 URL 提取                |
| POST 请求参数     | ❌ 不会抓到 | ParamSpider 不看请求 body  |
| JS 动态构造的 URL | ❌ 抓不到   | 除非页面源码里明文写出     |
| 本地触发的 XHR    | ❌ 抓不到   | ParamSpider 不实际加载页面 |

- 结果可能带有html编码、url编码，导致后续的Gxss执行错误，需要脚本进行解码一次.

![image-20250422101738467](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250422101738467.png)

### 三、使用

```
paramspider -d example.com
```

