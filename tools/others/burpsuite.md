### 一、burp -> 代理出去

![image-20250417004548694](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250417004548694.png)

### 二、target/out-of scope过滤

浏览器插件的代理只能过滤不想要的：

![image-20250422133324999](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250422133324999.png)

但是可能有很多不想要的，那还不如过滤想要的。

**步骤：**

1. add scope

   ![image-20250422133433695](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250422133433695.png)

2. Show only in-scope items

   ![image-20250422133519101](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250422133519101.png)
   
3. out-of scope

   ![image-20250426122528298](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250426122528298.png)

   注意：
   
   	1. target scope是只对http history起作用，但是**out-of-scope是对全局起作用**，包括Intercept，**需要在add scope之后选择yes**
   	1. history与out-of-scpoe不冲突，开了out-of-scope，history filter依然可以配
   
   ##### 丢弃超出范围的请求：
   
   ![image-20250430140543330](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250430140543330.png)
   
   最好不要配，因为一些网站可能会用到其他网站的api。

### 三、intruder

##### 1. 四种方式

![image-20250425140016403](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250425140016403.png)

1. n个payload位置，只需设置一个payloadxm，然后一个一个跑，每次payload只在一个位置：n x m
2. n个payload位置，只需设置一个payloadxm，一阵一阵跑，每次所有payload位置都使用相同payload: m x 1
3. n个payload位置，需要设置n个payloadxm，一片一片跑，每个payload位置都按照它们自己的payload跑：m x 1
4. n x m

##### 2. 速度控制

![image-20250424223906481](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250424223906481.png)

最大一个线程，没0.2s发一次

##### 3. payload处理

1. url特殊字符的编码：在payloads在url上特别管用，要开，其他位置看情况开

![image-20250424224040173](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250424224040173.png)

2. 其他payload处理

   ![image-20250425220119278](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250425220119278.png)

### 四、replace regex rule

```
{
  "email": "wanwencheng2@gma121211133il.com",
  "password": "@QWer611814",
  "partner": null,
  "channelInfo": {
    "referrer": "https://help.syfe.com/",
    "landingPage": "/managed-portfolio",
    "queryParams": {}
  },
  "subscribeToMarketingCommunications": true,
  "raFlowVersion": 2,
  "preferredLanguage": "ENGLISH",
  "choosePortfolioPositioningVersion": "VARIANT"
}
```

wanwencheng2@gma121211133il.com是替换内容：

**配置：**

![image-20250426151457270](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250426151457270.png)

### 五、BP那个history的filter --- 默认是过滤掉图片的！

### 6、保留BP快的配置，存在磁盘，如何打开

![image-20250430160535503](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250430160535503.png)

**setting -> proxy：**

![image-20250430160505162](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250430160505162.png)

### 7、获取bp证书的两种方式

1. 直接访问获取：127.0.0.1:8080获取

   1. 前提：

      ![image-20250501140726356](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250501140726356.png)

2. 手动export

   ![image-20250501141015351](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250501141015351.png)

   注意：Regenertate不能碰，不然要重新安装证书了

### 8、宏录制

Session->Macros:

![image-20250513223812024](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250513223812024.png)

缺点：只能跑一遍组合，批量跑组合使用tuber intruder写脚本

### 9、简单并发手法

1. 创建group
2. 加一个tab到里面，control+d加倍
3. send parallel

### 10、使用tuber插件race.py并发

需要在请求头加上：X：%s

##### 优化后的race.py:

```
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                          concurrentConnections=30,
                          requestsPerConnection=100,
                          pipeline=False)

    # 指定字典文件路径（示例路径，可修改）
    dictionary_path = r"C:\Users\33940\Desktop\payloads\word.txt"

    
    # 读取字典文件内容
    with open(dictionary_path, "r") as f:
        custom_wordlist = [line.strip() for line in f.readlines()]

    # 遍历字典，替换 %s 并发送请求
    for word in custom_wordlist:
        modified_req = target.req.replace(b"%s", word.encode())
        engine.queue(modified_req, gate='race1')

    # 同时触发所有请求（Race Condition 测试）
    engine.openGate('race1')
    engine.complete(timeout=60)


def handleResponse(req, interesting):
    table.add(req)
```

注意：concurrentConnections要和字典大小匹配！

### 11、burp字体配置

##### 光标偏移：

![image-20250530142412089](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250530142412089.png)

##### 包字体：

![image-20250530142447963](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250530142447963.png)

##### 其他字体：

![image-20250530142524328](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250530142524328.png)
