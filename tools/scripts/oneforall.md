# 使用原则 ：主动 + 无需API的被动 （不要用啊！！！）

[TOC]

### 配置：

```python
# coding=utf-8
"""
OneForAll自定义配置
"""

import pathlib

# 路径设置
relative_directory = pathlib.Path(__file__).parent.parent  # OneForAll代码相对路径
data_storage_dir = relative_directory.joinpath('data')  # 数据存放目录

# OneForAll入口参数设置
enable_check_network = True  # 开启网络环境检查
enable_check_version = True  # 开启最新版本检查
enable_brute_module = True  # 使用爆破模块(默认True)
enable_dns_resolve = True  # 使用DNS解析子域(默认True)
enable_http_request = True  # 使用HTTP请求子域(默认True)
enable_finder_module = True  # 开启finder模块,开启会从响应体和JS中再次发现子域(默认True)
enable_altdns_module = True  # 开启altdns模块,开启会利用置换技术重组子域再次发现新子域(默认True)
enable_cdn_check = True  # 开启cdn检查模块(默认True)
enable_banner_identify = True  # 开启WEB指纹识别模块(默认True)
enable_takeover_check = True  # 开启子域接管风险检查(默认False)
# HTTP请求子域的端口范围 参数可选值有 'small', 'medium', 'large'
http_request_port = 'small'  # 请求端口范围(默认 'small'，表示请求子域的80,443端口)
# 参数可选值True，False分别表示导出存活，全部子域结果
result_export_alive = True  # 只导出存活的子域结果(默认False)
result_save_format = 'csv'  # 子域结果保存文件格式(默认csv)
# 参数path默认None使用OneForAll结果目录自动生成路径
result_save_path = None  # 子域结果保存文件路径(默认None)

# 收集模块设置
save_module_result = False  # 保存各模块发现结果为json文件(默认False)
enable_all_module = True  # 启用所有收集模块(默认True)
enable_partial_module = []  # 启用部分收集模块 必须禁用enable_all_module才能生效
# 只使用ask和baidu搜索引擎收集子域的示例
# enable_partial_module = ['modules.search.ask', 'modules.search.baidu']

# 爆破模块设置
brute_concurrent_num = 1000  # 爆破时并发查询数量(默认2000，最大推荐10000)
# 爆破所使用的字典路径(默认None则使用data/subdomains.txt，自定义字典请使用绝对路径)
brute_wordlist_path = "E:\SecTools\OneForAll\data\subnames_medium.txt"
use_china_nameservers = False  # 使用中国域名服务器 如果你所在网络不在中国则建议设置False
enable_recursive_brute = True  # 是否使用递归爆破(默认False)
brute_recursive_depth = 3  # 递归爆破深度(默认2层)
# 爆破下一层子域所使用的字典路径(默认None则使用data/subnames_next.txt，自定义字典请使用绝对路径)
recursive_nextlist_path = "E:\SecTools\OneForAll\data\subnames.txt"
enable_check_dict = True  # 是否开启字典配置检查提示(默认False)
delete_generated_dict = True  # 是否删除爆破时临时生成的字典(默认True)
#  是否删除爆破时massdns输出的解析结果 (默认True)
#  massdns输出的结果中包含更详细解析结果
#  注意: 当爆破的字典较大或使用递归爆破或目标域名存在泛解析时生成的文件可能会很大
delete_massdns_result = True
only_save_valid = True  # 是否在处理爆破结果时只存入解析成功的子域
check_time = 10  # 检查字典配置停留时间(默认10秒)
enable_fuzz = False  # 是否使用fuzz模式枚举域名
fuzz_place = None  # 指定爆破的位置 指定的位置用`*`表示 示例：www.*.example.com
fuzz_rule = None  # fuzz域名的正则 示例：'[a-z][0-9]' 表示第一位是字母 第二位是数字
brute_ip_blacklist = {'0.0.0.0', '0.0.0.1'}  # IP黑名单 子域解析到IP黑名单则标记为非法子域
# CNAME黑名单 子域解析到CNAME黑名单则标记为非法子域
brute_cname_blacklist = {'nonexist.sdo.com', 'shop.taobao.com'}
ip_appear_maximum = 100  # 多个子域解析到同一IP次数超过100次则标记为非法(泛解析)子域
cname_appear_maximum = 50  # 多个子域解析到同一cname次数超过50次则标记为非法(泛解析)子域

# 代理设置
enable_request_proxy = True  # 是否使用代理(全局开关)
proxy_all_module = True  # 代理所有模块
proxy_partial_module = ['GoogleQuery', 'AskSearch', 'DuckDuckGoSearch',
                        'GoogleAPISearch', 'GoogleSearch', 'YahooSearch',
                        'YandexSearch', 'CrossDomainXml',
                        'ContentSecurityPolicy']  # 代理自定义的模块
request_proxy_pool = [{'http': 'http://127.0.0.1:1234',
                       'https': 'http://127.0.0.1:1234'}]  # 代理池
# request_proxy_pool = [{'http': 'socks5h://127.0.0.1:10808',
#                        'https': 'socks5h://127.0.0.1:10808'}]  # 代理池


# 请求设置
request_thread_count = 10  # 请求线程数量(默认None，则根据情况自动设置)
request_timeout_second = (13, 27)  # 请求超时秒数(默认connect timout推荐略大于3秒)
request_ssl_verify = False  # 请求SSL验证(默认False)
request_allow_redirect = True  # 请求允许重定向(默认True)
request_redirect_limit = 10  # 请求跳转限制(默认10次)
# 默认请求头 可以在headers里添加自定义请求头
request_default_headers = {
    'Accept': 'text/html,application/xhtml+xml,'
              'application/xml;q=0.9,*/*;q=0.8',
    'Accept-Encoding': 'gzip, deflate',
    'Accept-Language': 'en-US,en;q=0.9,zh-CN;q=0.8,zh;q=0.7',
    'Cache-Control': 'max-age=0',
    'DNT': '1',
    'Referer': 'https://www.google.com/',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 '
                  '(KHTML, like Gecko) Chrome/76.0.3809.100 Safari/537.36',
    'Upgrade-Insecure-Requests': '1',
    'X-Forwarded-For': '127.0.0.1'
}
enable_random_ua = True  # 使用随机UA(默认True，开启可以覆盖request_default_headers的UA)


# 搜索模块设置
enable_recursive_search = True  # 递归搜索子域
search_recursive_times = 3  # 递归搜索层数

# 网络空间测绘引擎设置
cam_records_maximum_per_domain = 1000   # 对于单个主域名，在测绘引擎中的最多查询多少条记录，防止泛解析和CDN浪费积分，对 fofa, hunter, quake, zoomeye 生效，最低为100

# 是否从输入的数据中使用tldextract提取主域名。若设为 False，OneForAll会直接将输入的域名作为主域名，比如北京分公司的域名 beijing.10086.com 就不会被解析成母公司的域名 10086.com
use_tld_extract = True
```

### 解释：

#### 一、主动扫描模块（全开）

你启用了所有主控开关，以下模块 **全部已启用**：

| 模块                              | 说明                                                     |
| --------------------------------- | -------------------------------------------------------- |
| ✅ `enable_brute_module = True`    | 子域名**字典爆破**                                       |
| ✅ `enable_dns_resolve = True`     | 爆破出的子域使用 DNS 进行解析验证                        |
| ✅ `enable_recursive_brute = True` | **递归爆破**，对发现的子域再次进行爆破（递归深度为 3）   |
| ✅ `enable_http_request = True`    | 对子域发起 HTTP 请求，探测是否存活                       |
| ✅ `enable_finder_module = True`   | 从响应页面、JS 中提取新子域（**Finder 模块**）           |
| ✅ `enable_altdns_module = True`   | 使用变形方式生成新子域（**Altdns 模块**）                |
| ✅ `enable_cdn_check = True`       | 检测子域是否部署 CDN                                     |
| ✅ `enable_banner_identify = True` | 页面指纹识别（CMS、框架等）                              |
| ✅ `enable_takeover_check = True`  | 子域接管检测（例如：指向 GitHub Pages、S3 等未绑定服务） |

> **总结**：主动扫描部分你是全开的，OneForAll 会尽可能深入地爆破和验证子域。

#### 二、被动收集模块（只启用非 API 模块）

##### 你的配置：

```
python复制编辑enable_all_module = True  # 启用所有收集模块
enable_partial_module = []  # 空，不启用限制
```

虽然这里是写了 `enable_all_module = True`，**但你没有配置任何 API 密钥**，所以：

| 模块类别       | 模块名                                             | 实际效果                                                     |
| -------------- | -------------------------------------------------- | ------------------------------------------------------------ |
| ✅ 公共搜索引擎 | ask, baidu, bing, google, sogou, so, yahoo, yandex | **启用**，使用搜索引擎采集子域名                             |
| ✅ 证书透明日志 | crt.sh                                             | **启用**，查询 `crt.sh` 提取子域                             |
| ❌ 空间测绘平台 | fofa, quake, zoomeye, hunter                       | **未启用**，因未配置 API 密钥，**这些模块虽然逻辑上启用，但不会生效** |

#### 三、代理配置（Tun模式不需要开）

你开启了代理，并配置如下：

```
python复制编辑enable_request_proxy = True          # 全局启用代理
proxy_all_module = True              # 所有模块都走代理
request_proxy_pool = [{
    'http': 'http://127.0.0.1:1234',
    'https': 'http://127.0.0.1:1234'
}]
```

✅ 表示你已经正确配置了代理，本地代理监听在 `127.0.0.1:1234`。只要代理服务开启，所有请求都会通过这个代理发出。

#### 四、细节

```
use_china_nameservers = False  # 使用中国域名服务器 如果你所在网络不在中国则建议设置False

brute_recursive_depth = 3  # 递归爆破深度(默认2层)

search_recursive_times = 3  # 递归搜索层数
```

### 缺陷：

#### 一、指定FUZZ麻烦

​	oneforall的爆破可以指定爆破多深，也可以指定每一层的字典。

​	**但是只能给一级域名进行从头到尾的爆破，不能单独指定FUZZ**；

​	是有FUZZ模块，但是和爆破模块冲突，如果要使用FUZZ，需要修很多配置，很麻烦！

##### 总结：

​	若要进行单独设置子域名的FUZZ使用FFUF工具。

#### 二、不能控制爆破的延时

只能通过：

```
brute_concurrent_num = 1000  # 爆破时并发查询数量(默认2000，最大推荐10000)
```

控制爆破线程数量。

### 三、被动 + 爆破 大字典 结果 < ffuf 爆破小字典

一无是处，用了就是sb，慢的要死！

