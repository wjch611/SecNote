## 专门针对crtsh的subdomain被动收集器

#### 一、使用

```
python .\ctfr.py -d slack.com -o urls.txt
```

### 二、缺陷

1. 结果带 *的表示，可以进一步fuzz的，需要进行处理
1. 没有dns验活