### 一、TUN与全局代理

简单点说：

​	tun是把内部所有流量 -> clash；

​	全局代理是clash的一个规则：规则是，clash -> 所有网站/去向，都走代理

之前发现访问：myip.ipip.net那些网站，发现ip没隐藏，是因为，这些网站在规则中走到直连

##### 最好的配置：

	1. tun + 规则 + 负载均衡 + fake-ip
	2. 规则：
	 1. 经常访问的国内网站走直连，其他走代理

### 二、添加负载均衡代理组

### 三、订阅可以合并

### 四、fake-ip的作用

#### **（1）防止 DNS 泄露**

- **问题**：某些应用可能绕过代理，直接向公共 DNS（如 `8.8.8.8`）查询域名，导致真实访问目标暴露。
- **Fake-IP 方案**：
  - 代理工具（如 Clash）在本地维护一个 Fake IP 池（如 `198.18.0.0/16`）。
  - 当应用查询域名时，代理直接返回一个 Fake IP（如 `198.18.1.1`），而非真实 IP。
  - 后续对该 IP 的请求会被代理工具拦截，并还原为原始域名，再通过代理访问。
  - **结果**：DNS 查询不会外泄，所有流量强制经过代理。

#### **（2）零延迟 DNS（Zero-Delay DNS）**

- **传统 DNS 解析**：
  - 应用发起请求 → 查询 DNS → 拿到真实 IP → 连接代理 → 代理转发。
  - 必须等待 DNS 解析完成，增加延迟。
- **Fake-IP 优化**：
  - 代理预先分配 Fake IP，应用查询域名时**立即返回假 IP**，无需等待真实 DNS 解析。
  - 后续连接 Fake IP 时，代理**并发处理 DNS 解析和连接建立**，减少延迟。

#### 关闭fake-ip:

使用直连ip有被反查的风险，如果要开启直连：

![image-20250428140242685](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250428140242685.png)

![image-20250428140311502](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250428140311502.png)

### 五、最终配置

```
mixed-port: 7890
allow-lan: true
bind-address: '*'
mode: rule
log-level: info
external-controller: '127.0.0.1:9090'
dns:
    enable: true
    ipv6: false
    default-nameserver: [223.5.5.5, 119.29.29.29]
    enhanced-mode: fake-ip
    fake-ip-range: 198.18.0.1/16
    use-hosts: true
    nameserver: ['https://doh.pub/dns-query', 'https://dns.alidns.com/dns-query']
    fallback: ['https://doh.dns.sb/dns-query', 'https://dns.cloudflare.com/dns-query', 'https://dns.twnic.tw/dns-query', 'tls://8.8.4.4:853']
    fallback-filter: { geoip: true, ipcidr: [240.0.0.0/4, 0.0.0.0/32] }
proxies:
    - { name: '剩余流量：9.46 GB', type: trojan, server: hk01t.goodyun.buzz, port: 22331, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: hk01t.goodyun.buzz }
    - { name: '距离下次重置剩余：3 天', type: trojan, server: hk01t.goodyun.buzz, port: 22331, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: hk01t.goodyun.buzz }
    - { name: 套餐到期：2026-01-30, type: trojan, server: hk01t.goodyun.buzz, port: 22331, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: hk01t.goodyun.buzz }
    - { name: '🇭🇰[HK]HongKong01-GPT优化', type: trojan, server: hk01t.goodyun.buzz, port: 22331, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: hk01t.goodyun.buzz }
    - { name: '🇭🇰[HK]HongKong02-GPT优化', type: trojan, server: hk02t.goodyun.buzz, port: 22332, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: hk02t.goodyun.buzz }
    - { name: '🇭🇰[HK]HongKong03-GPT优化', type: trojan, server: hk03t.goodyun.buzz, port: 22333, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: hk03t.goodyun.buzz }
    - { name: '🇭🇰[HK]HongKong04-GPT优化', type: trojan, server: hk04t.goodyun.buzz, port: 22334, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: hk04t.goodyun.buzz }
    - { name: '🇭🇰[HK]HongKong05-GPT优化', type: trojan, server: hk05t.goodyun.buzz, port: 22335, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: hk05t.goodyun.buzz }
    - { name: '🇭🇰[HK]HongKong06-GPT优化', type: trojan, server: hk06t.goodyun.buzz, port: 22336, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: hk06t.goodyun.buzz }
    - { name: '🇭🇰[HK]HongKong07-GPT优化', type: trojan, server: hk07t.goodyun.buzz, port: 22337, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: hk07t.goodyun.buzz }
    - { name: '🇭🇰[HK]HongKong08-GPT优化', type: trojan, server: hk08t.goodyun.buzz, port: 22338, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: hk08t.goodyun.buzz }
    - { name: '🇭🇰[HK]HongKong09-GPT优化', type: trojan, server: hk09t.goodyun.buzz, port: 22339, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: hk09t.goodyun.buzz }
    - { name: '🇭🇰[HK]HongKong10-GPT优化', type: trojan, server: hk10t.goodyun.buzz, port: 22340, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: hk10t.goodyun.buzz }
    - { name: '🇨🇳[CN]HK专线01-【5倍率】测试', type: trojan, server: hkiepl01.goodyun.buzz, port: 22350, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: hkiepl01.goodyun.buzz }
    - { name: '🇨🇳[CN]HK专线02-【5倍率】测试', type: trojan, server: hkiepl02.goodyun.buzz, port: 22351, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: hkiepl02.goodyun.buzz }
    - { name: '🇨🇳[CN]SG专线01-【5倍率】测试', type: trojan, server: sgiepl001.goodyun.buzz, port: 26219, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: sgiepl001.goodyun.buzz }
    - { name: '🇨🇳[CN]TW专线01-【5倍率】测试', type: trojan, server: twiepl01.goodyun.buzz, port: 33922, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: twiepl01.goodyun.buzz }
    - { name: '🇯🇵[JP]Tokyo01', type: trojan, server: jp01t.goodyun.buzz, port: 35001, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: jp01t.goodyun.buzz }
    - { name: '🇯🇵[JP]Tokyo02', type: trojan, server: jp02t.goodyun.buzz, port: 35002, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: jp02t.goodyun.buzz }
    - { name: '🇯🇵[JP]Tokyo03', type: trojan, server: jp03t.goodyun.buzz, port: 35003, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: jp03t.goodyun.buzz }
    - { name: '🇯🇵[JP]Tokyo04', type: trojan, server: jp04t.goodyun.buzz, port: 35004, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: jp04t.goodyun.buzz }
    - { name: '🇯🇵[JP]Tokyo06', type: trojan, server: jp06t.goodyun.buzz, port: 35006, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: jp06t.goodyun.buzz }
    - { name: '🇯🇵[JP]Tokyo07', type: trojan, server: jp07t.goodyun.buzz, port: 35007, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: jp07t.goodyun.buzz }
    - { name: '🇯🇵[JP]Tokyo08', type: trojan, server: jp08t.goodyun.buzz, port: 38088, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: jp08t.goodyun.buzz }
    - { name: '🇯🇵[JP]Tokyo09', type: trojan, server: jp09t.goodyun.buzz, port: 35009, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: jp09t.goodyun.buzz }
    - { name: '🇯🇵[JP]Tokyo10', type: trojan, server: jp10t.goodyun.buzz, port: 35010, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: jp10t.goodyun.buzz }
    - { name: '🇯🇵[JP]Tokyo11', type: trojan, server: jp11t.goodyun.buzz, port: 35011, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: jp11t.goodyun.buzz }
    - { name: '🇸🇬[SG]Singapore01', type: trojan, server: sg01t.goodyun.buzz, port: 26201, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: sg01t.goodyun.buzz }
    - { name: '🇸🇬[SG]Singapore02', type: trojan, server: sg02t.goodyun.buzz, port: 26202, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: sg02t.goodyun.buzz }
    - { name: '🇸🇬[SG]Singapore03', type: trojan, server: sg03t.goodyun.buzz, port: 26203, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: sg03t.goodyun.buzz }
    - { name: '🇸🇬[SG]Singapore04', type: trojan, server: sg04t.goodyun.buzz, port: 26211, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: sg04t.goodyun.buzz }
    - { name: '🇸🇬[SG]Singapore05', type: trojan, server: sg05t.goodyun.buzz, port: 26205, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: sg05t.goodyun.buzz }
    - { name: '🇸🇬[SG]Singapore06', type: trojan, server: sg06t.goodyun.buzz, port: 26216, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: sg06t.goodyun.buzz }
    - { name: '🇸🇬[SG]Singapore07', type: trojan, server: sg07t.goodyun.buzz, port: 26207, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: sg07t.goodyun.buzz }
    - { name: '🇸🇬[SG]Singapore08', type: trojan, server: sg08t.goodyun.buzz, port: 26208, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: sg08t.goodyun.buzz }
    - { name: '🇸🇬[SG]Singapore09', type: trojan, server: sg09t.goodyun.buzz, port: 26209, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: sg09t.goodyun.buzz }
    - { name: '🇸🇬[SG]Singapore10', type: trojan, server: sg10t.goodyun.buzz, port: 26210, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: sg10t.goodyun.buzz }
    - { name: '🇺🇸[US]美国Los Angeles01-GPT优化', type: trojan, server: us01t.goodyun.buzz, port: 33801, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: us01t.goodyun.buzz }
    - { name: '🇺🇸[US]美国Los Angeles02-GPT优化', type: trojan, server: us02t.goodyun.buzz, port: 33802, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: us02t.goodyun.buzz }
    - { name: '🇺🇸[US]美国Los Angeles03-GPT优化', type: trojan, server: us03t.goodyun.buzz, port: 33803, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: us03t.goodyun.buzz }
    - { name: '🇺🇸[US]美国Los Angeles04-GPT优化', type: trojan, server: us04t.goodyun.buzz, port: 33805, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: us04t.goodyun.buzz }
    - { name: '🇺🇸[US]美国Santa Clara05-GPT优化', type: trojan, server: us05t.goodyun.buzz, port: 33806, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: us05t.goodyun.buzz }
    - { name: '🇺🇸[US]美国Santa Clara06-GPT优化', type: trojan, server: us06t.goodyun.buzz, port: 33807, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: us06t.goodyun.buzz }
    - { name: '🇺🇸[US]美国San Jose07-GPT优化', type: trojan, server: us07t.goodyun.buzz, port: 33808, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: us07t.goodyun.buzz }
    - { name: '🇺🇸[US]美国San Jose08-GPT优化', type: trojan, server: us08t.goodyun.buzz, port: 33809, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: us08t.goodyun.buzz }
    - { name: '🇹🇼[TW]TaiPei01-GPT优化', type: trojan, server: tw01t.goodyun.buzz, port: 33901, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: tw01t.goodyun.buzz }
    - { name: '🇹🇼[TW]TaiPei02-GPT优化', type: trojan, server: tw02t.goodyun.buzz, port: 33902, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: tw02t.goodyun.buzz }
    - { name: '🇹🇼[TW]TaiPei03-GPT优化', type: trojan, server: tw03t.goodyun.buzz, port: 33903, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: tw03t.goodyun.buzz }
    - { name: '🇹🇼[TW]TaiPei04-GPT优化', type: trojan, server: tw04t.goodyun.buzz, port: 33904, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: tw04t.goodyun.buzz }
    - { name: '🇹🇼[TW]TaiPei05-GPT优化', type: trojan, server: tw05t.goodyun.buzz, port: 33905, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: tw05t.goodyun.buzz }
    - { name: '🇹🇼[TW]TaiPei06-GPT优化', type: trojan, server: tw06t.goodyun.buzz, port: 33906, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: tw06t.goodyun.buzz }
    - { name: '🇫🇷[FR]法国-Paris01', type: trojan, server: fr01t.goodyun.buzz, port: 11001, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: fr01t.goodyun.buzz }
    - { name: '🇫🇷[FR]法国-Paris02', type: trojan, server: fr02t.goodyun.buzz, port: 11002, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: fr02t.goodyun.buzz }
    - { name: '🇬🇧[UK]英国Coventry01-BBC优化', type: trojan, server: uk01t.goodyun.buzz, port: 11003, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: uk01t.goodyun.buzz }
    - { name: '🇬🇧[UK]英国Coventry02-BBC优化', type: trojan, server: uk02t.goodyun.buzz, port: 11005, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: uk02t.goodyun.buzz }
    - { name: '🇩🇪[DE]德国-Frankfurt01', type: trojan, server: de01t.goodyun.buzz, port: 11006, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: de01t.goodyun.buzz }
    - { name: '🇩🇪[DE]德国-Frankfurt02', type: trojan, server: de02t.goodyun.buzz, port: 11007, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: de02t.goodyun.buzz }
    - { name: '🇨🇦[CA]加拿大-Toronto', type: trojan, server: ca01t.goodyun.buzz, port: 11008, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: ca01t.goodyun.buzz }
    - { name: '🇳🇱[NL]荷兰-Amsterdam', type: trojan, server: nl01t.goodyun.buzz, port: 11009, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: nl01t.goodyun.buzz }
    - { name: '🇷🇺[RU]俄罗斯-Moscow', type: trojan, server: ru01t.goodyun.buzz, port: 11010, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: ru01t.goodyun.buzz }
    - { name: '🇹🇷[TR]土耳其-Istanbul', type: trojan, server: tr01t.goodyun.buzz, port: 11011, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: tr01t.goodyun.buzz }
    - { name: '🇮🇳[IN]印度-bangalore', type: trojan, server: in01t.goodyun.buzz, port: 11012, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: in01t.goodyun.buzz }
    - { name: '🇰🇷[KR]韩国-Seoul', type: trojan, server: kr01t.goodyun.buzz, port: 11013, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: kr01t.goodyun.buzz }
    - { name: '🇻🇳[VN]越南-HoChiMinh', type: trojan, server: vn01t.goodyun.buzz, port: 11015, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: vn01t.goodyun.buzz }
    - { name: '🇦🇺[AU]澳大利亚-Sydney', type: trojan, server: au01t.goodyun.buzz, port: 11016, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: au01t.goodyun.buzz }
    - { name: '🇳🇬[NG]尼日利亚-Lagos', type: trojan, server: ng01t.goodyun.buzz, port: 11017, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: ng01t.goodyun.buzz }
    - { name: '🇦🇷[AR]阿根廷-BuenosAires', type: trojan, server: ar01t.goodyun.buzz, port: 11018, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: ar01t.goodyun.buzz }
    - { name: '🇲🇰[MK]马其顿-Macedonia', type: trojan, server: mk01t.goodyun.buzz, port: 11019, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: mk01t.goodyun.buzz }
    - { name: '🇦🇪[AE]阿联酋-ArabEmirates', type: trojan, server: ae01t.goodyun.buzz, port: 11028, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true, sni: ae01t.goodyun.buzz }
    - { name: '[境外用户专用]GPT01', type: ss, server: hkhw01.goodyun.buzz, port: 19723, cipher: aes-256-gcm, password: 7420f733-2d9d-46ba-b3ee-d138b69c477b, udp: true }
    - { name: 建议每次使用前都更新一下订阅, type: ss, server: hzhz1.sssyun.xyz, port: 29527, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇭🇰测试节点, type: ss, server: hzhz1.sssyun.xyz, port: 18008, cipher: aes-128-gcm, password: 7b072c12-2801-4fa8-8fe2-2bd8a20f481b, udp: true }
    - { name: 🇭🇰香港1, server: hk1.jueduibupao.top, port: 50001, sni: hk1.jueduibupao.top, up: 100, down: 300, skip-cert-verify: true, type: hysteria2, password: 7b072c12-2801-4fa8-8fe2-2bd8a20f481b }
    - { name: 🇭🇰香港2, server: hk2.jueduibupao.top, port: 43599, sni: hk2.jueduibupao.top, up: 100, down: 300, skip-cert-verify: true, type: hysteria2, password: 7b072c12-2801-4fa8-8fe2-2bd8a20f481b, ports: 40000-50000 }
    - { name: 🇸🇬新加坡1, server: linsg1.jueduibupao.top, port: 49371, sni: linsg1.jueduibupao.top, up: 300, down: 300, skip-cert-verify: true, type: hysteria2, password: 7b072c12-2801-4fa8-8fe2-2bd8a20f481b, ports: 40000-50000 }
    - { name: 🇸🇬新加坡2, server: linsg2.jueduibupao.top, port: 47949, sni: linsg2.jueduibupao.top, up: 300, down: 300, skip-cert-verify: true, type: hysteria2, password: 7b072c12-2801-4fa8-8fe2-2bd8a20f481b, ports: 40000-50000 }
    - { name: 🇨🇳台湾1-Gpt, server: tw2.jueduibupao.top, port: 30078, sni: tw2.jueduibupao.top, up: 200, down: 200, skip-cert-verify: true, type: hysteria2, password: 7b072c12-2801-4fa8-8fe2-2bd8a20f481b }
    - { name: 🇨🇳台湾2-Gpt, server: tw22.jueduibupao.top, port: 30104, sni: tw22.jueduibupao.top, up: 200, down: 200, skip-cert-verify: true, type: hysteria2, password: 7b072c12-2801-4fa8-8fe2-2bd8a20f481b }
    - { name: 🇯🇵日本1, server: linjp1.jueduibupao.top, port: 47237, sni: linjp1.jueduibupao.top, up: 100, down: 300, skip-cert-verify: true, type: hysteria2, password: 7b072c12-2801-4fa8-8fe2-2bd8a20f481b, ports: 40000-50000 }
    - { name: 🇯🇵日本2, server: linjp2.jueduibupao.top, port: 44353, sni: linjp2.jueduibupao.top, up: 300, down: 300, skip-cert-verify: true, type: hysteria2, password: 7b072c12-2801-4fa8-8fe2-2bd8a20f481b, ports: 40000-50000 }
    - { name: 🇰🇷韩国-家宽, server: 6bkr1.6bnw.top, port: 50750, sni: 6bkr1.6bnw.top, up: 100, down: 100, skip-cert-verify: true, type: hysteria2, password: 7b072c12-2801-4fa8-8fe2-2bd8a20f481b }
    - { name: 🇺🇸拉斯维加斯, server: lvs.jueduibupao.top, port: 50001, sni: lvs.jueduibupao.top, up: 300, down: 300, skip-cert-verify: true, type: hysteria2, password: 7b072c12-2801-4fa8-8fe2-2bd8a20f481b }
    - { name: 🇺🇸西雅图, server: ny.jueduibupao.top, port: 50001, sni: ny.jueduibupao.top, up: 300, down: 300, skip-cert-verify: true, type: hysteria2, password: 7b072c12-2801-4fa8-8fe2-2bd8a20f481b }
    - { name: 🇺🇸圣何塞, server: azcus1.jueduibupao.top, port: 20101, sni: azcus1.jueduibupao.top, up: 300, down: 300, skip-cert-verify: true, type: hysteria2, password: 7b072c12-2801-4fa8-8fe2-2bd8a20f481b }
    - { name: 🇩🇪德国, server: de1.jueduibupao.top, port: 43385, sni: de1.jueduibupao.top, up: 300, down: 300, skip-cert-verify: true, type: hysteria2, password: 7b072c12-2801-4fa8-8fe2-2bd8a20f481b, ports: 40000-50000 }
    - { name: 🇬🇧英国, server: uk1.jueduibupao.top, port: 45729, sni: uk1.jueduibupao.top, up: 300, down: 300, skip-cert-verify: true, type: hysteria2, password: 7b072c12-2801-4fa8-8fe2-2bd8a20f481b, ports: 40000-50000 }
    - { name: 测试-trojan香港1, type: trojan, server: hzhz1.sssyun.xyz, port: 22240, password: 7b072c12-2801-4fa8-8fe2-2bd8a20f481b, udp: true, sni: trhk1.6bnw.top, skip-cert-verify: true }
    - { name: 测试-trojan香港2, type: trojan, server: hzhz2.sssyun.xyz, port: 22241, password: 7b072c12-2801-4fa8-8fe2-2bd8a20f481b, udp: true, sni: trhk1.6bnw.top, skip-cert-verify: true }
    - { name: 🇭🇰香港01-IPv6, type: ss, server: hzhz1.sssyun.xyz, port: 29527, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇭🇰香港02-IPv6, type: ss, server: hzhz2.sssyun.xyz, port: 40010, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇭🇰香港03-IPv6, type: ss, server: hzhz1.sssyun.xyz, port: 19527, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇭🇰香港04-IPv6, type: ss, server: hzhz2.sssyun.xyz, port: 40023, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇭🇰香港05-流媒体解锁, type: ss, server: hzhz1.sssyun.xyz, port: 39301, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇭🇰香港06-流媒体解锁, type: ss, server: hzhz2.sssyun.xyz, port: 40011, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇸🇬新加坡01-流媒体解锁, type: ss, server: hzhz1.sssyun.xyz, port: 39204, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇸🇬新加坡02-流媒体解锁, type: ss, server: hzhz2.sssyun.xyz, port: 40013, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇸🇬新加坡03, type: ss, server: hzhz1.sssyun.xyz, port: 39505, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇸🇬新加坡04, type: ss, server: hzhz2.sssyun.xyz, port: 44014, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇰🇷韩国-家宽03, type: ss, server: hzhz1.sssyun.xyz, port: 39206, cipher: 2022-blake3-aes-128-gcm, password: 'NjVhMWE2YmVmNzg4MTcwMA==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇰🇷韩国-家宽04, type: ss, server: hzhz2.sssyun.xyz, port: 20116, cipher: 2022-blake3-aes-128-gcm, password: 'NjVhMWE2YmVmNzg4MTcwMA==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇨🇳台湾01-Gpt, type: ss, server: hzhz1.sssyun.xyz, port: 32207, cipher: 2022-blake3-aes-128-gcm, password: 'NjVhMWE2YmVmNzg4MTcwMA==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇨🇳台湾02-Gpt, type: ss, server: hzhz2.sssyun.xyz, port: 40017, cipher: 2022-blake3-aes-128-gcm, password: 'NjVhMWE2YmVmNzg4MTcwMA==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇨🇳台湾03-Gpt, type: ss, server: hzhz1.sssyun.xyz, port: 39201, cipher: 2022-blake3-aes-128-gcm, password: 'NjVhMWE2YmVmNzg4MTcwMA==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇨🇳台湾04-Gpt, type: ss, server: hzhz2.sssyun.xyz, port: 40018, cipher: 2022-blake3-aes-128-gcm, password: 'NjVhMWE2YmVmNzg4MTcwMA==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇯🇵日本01, type: ss, server: hzhz1.sssyun.xyz, port: 19529, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇯🇵日本02, type: ss, server: hzhz2.sssyun.xyz, port: 40019, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇯🇵日本03, type: ss, server: hzhz1.sssyun.xyz, port: 49207, cipher: 2022-blake3-aes-256-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNjE5OWQyNDYxZjM1MjA0OTg=:N2IwNzJjMTItMjgwMS00ZmE4LThmZTItMmJkOGEyMGY=', udp: true }
    - { name: 🇯🇵日本04, type: ss, server: hzhz2.sssyun.xyz, port: 40020, cipher: 2022-blake3-aes-256-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNjE5OWQyNDYxZjM1MjA0OTg=:N2IwNzJjMTItMjgwMS00ZmE4LThmZTItMmJkOGEyMGY=', udp: true }
    - { name: 🇨🇦加拿大01, type: ss, server: hzhz1.sssyun.xyz, port: 39209, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇨🇦加拿大02, type: ss, server: hzhz2.sssyun.xyz, port: 40021, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇺🇸美国西雅图01, type: ss, server: hzhz1.sssyun.xyz, port: 12980, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇺🇸美国西雅图02, type: ss, server: hzhz2.sssyun.xyz, port: 12981, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇺🇸美国拉斯维加斯01, type: ss, server: hzhz1.sssyun.xyz, port: 12983, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇺🇸美国拉斯维加斯02, type: ss, server: hzhz2.sssyun.xyz, port: 12984, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇺🇸美国圣何塞01, type: ss, server: hzhz1.sssyun.xyz, port: 39203, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇺🇸美国圣何塞02, type: ss, server: hzhz2.sssyun.xyz, port: 40022, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇩🇪德国01, type: ss, server: hzhz1.sssyun.xyz, port: 39987, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇩🇪德国02, type: ss, server: hzhz2.sssyun.xyz, port: 39981, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇬🇧英国01, type: ss, server: hzhz1.sssyun.xyz, port: 32988, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: 🇬🇧英国02, type: ss, server: hzhz2.sssyun.xyz, port: 32989, cipher: 2022-blake3-aes-128-gcm, password: 'MDhhMTdjZDI0MjI2ZWRlNg==:N2IwNzJjMTItMjgwMS00Zg==', udp: true }
    - { name: '🇺🇸拉斯维加斯01-下载&防失联-0.1X', type: vmess, server: mugen11.6bnw.top, port: 8080, uuid: 7b072c12-2801-4fa8-8fe2-2bd8a20f481b, alterId: 0, cipher: auto, udp: true, network: ws, ws-opts: { path: '/?ed=2560', headers: { Host: buyvmus2.6bnw.top } }, ws-path: '/?ed=2560', ws-headers: { Host: buyvmus2.6bnw.top } }
    - { name: '🇺🇸拉斯维加斯02-下载&防失联-0.1X', type: vmess, server: mugen22.6bnw.top, port: 8080, uuid: 7b072c12-2801-4fa8-8fe2-2bd8a20f481b, alterId: 0, cipher: auto, udp: true, network: ws, ws-opts: { path: '/?ed=2560', headers: { Host: buyvmus2.6bnw.top } }, ws-path: '/?ed=2560', ws-headers: { Host: buyvmus2.6bnw.top } }
    - { name: '🇺🇸拉斯维加斯03-下载&防失联-0.1X', type: vmess, server: mugen33.6bnw.top, port: 8080, uuid: 7b072c12-2801-4fa8-8fe2-2bd8a20f481b, alterId: 0, cipher: auto, udp: true, network: ws, ws-opts: { path: '/?ed=2560', headers: { Host: buyvmus2.6bnw.top } }, ws-path: '/?ed=2560', ws-headers: { Host: buyvmus2.6bnw.top } }
    - { name: '🇺🇸拉斯维加斯04-下载&防失联-0.1X', type: vmess, server: mugen44.6bnw.top, port: 8080, uuid: 7b072c12-2801-4fa8-8fe2-2bd8a20f481b, alterId: 0, cipher: auto, udp: true, network: ws, ws-opts: { path: '/?ed=2560', headers: { Host: buyvmus2.6bnw.top } }, ws-path: '/?ed=2560', ws-headers: { Host: buyvmus2.6bnw.top } }
proxy-groups:
    - { name: TaiShan Net, type: select, proxies: [负载均衡, 自动选择(不含IPLC), 故障转移, '剩余流量：9.46 GB', '距离下次重置剩余：3 天', 套餐到期：2026-01-30, '🇭🇰[HK]HongKong01-GPT优化', '🇭🇰[HK]HongKong02-GPT优化', '🇭🇰[HK]HongKong03-GPT优化', '🇭🇰[HK]HongKong04-GPT优化', '🇭🇰[HK]HongKong05-GPT优化', '🇭🇰[HK]HongKong06-GPT优化', '🇭🇰[HK]HongKong07-GPT优化', '🇭🇰[HK]HongKong08-GPT优化', '🇭🇰[HK]HongKong09-GPT优化', '🇭🇰[HK]HongKong10-GPT优化', '🇨🇳[CN]HK专线01-【5倍率】测试', '🇨🇳[CN]HK专线02-【5倍率】测试', '🇨🇳[CN]SG专线01-【5倍率】测试', '🇨🇳[CN]TW专线01-【5倍率】测试', '🇯🇵[JP]Tokyo01', '🇯🇵[JP]Tokyo02', '🇯🇵[JP]Tokyo03', '🇯🇵[JP]Tokyo04', '🇯🇵[JP]Tokyo06', '🇯🇵[JP]Tokyo07', '🇯🇵[JP]Tokyo08', '🇯🇵[JP]Tokyo09', '🇯🇵[JP]Tokyo10', '🇯🇵[JP]Tokyo11', '🇸🇬[SG]Singapore01', '🇸🇬[SG]Singapore02', '🇸🇬[SG]Singapore03', '🇸🇬[SG]Singapore04', '🇸🇬[SG]Singapore05', '🇸🇬[SG]Singapore06', '🇸🇬[SG]Singapore07', '🇸🇬[SG]Singapore08', '🇸🇬[SG]Singapore09', '🇸🇬[SG]Singapore10', '🇺🇸[US]美国Los Angeles01-GPT优化', '🇺🇸[US]美国Los Angeles02-GPT优化', '🇺🇸[US]美国Los Angeles03-GPT优化', '🇺🇸[US]美国Los Angeles04-GPT优化', '🇺🇸[US]美国Santa Clara05-GPT优化', '🇺🇸[US]美国Santa Clara06-GPT优化', '🇺🇸[US]美国San Jose07-GPT优化', '🇺🇸[US]美国San Jose08-GPT优化', '🇹🇼[TW]TaiPei01-GPT优化', '🇹🇼[TW]TaiPei02-GPT优化', '🇹🇼[TW]TaiPei03-GPT优化', '🇹🇼[TW]TaiPei04-GPT优化', '🇹🇼[TW]TaiPei05-GPT优化', '🇹🇼[TW]TaiPei06-GPT优化', '🇫🇷[FR]法国-Paris01', '🇫🇷[FR]法国-Paris02', '🇬🇧[UK]英国Coventry01-BBC优化', '🇬🇧[UK]英国Coventry02-BBC优化', '🇩🇪[DE]德国-Frankfurt01', '🇩🇪[DE]德国-Frankfurt02', '🇨🇦[CA]加拿大-Toronto', '🇳🇱[NL]荷兰-Amsterdam', '🇷🇺[RU]俄罗斯-Moscow', '🇹🇷[TR]土耳其-Istanbul', '🇮🇳[IN]印度-bangalore', '🇰🇷[KR]韩国-Seoul', '🇻🇳[VN]越南-HoChiMinh', '🇦🇺[AU]澳大利亚-Sydney', '🇳🇬[NG]尼日利亚-Lagos', '🇦🇷[AR]阿根廷-BuenosAires', '🇲🇰[MK]马其顿-Macedonia', '🇦🇪[AE]阿联酋-ArabEmirates', '[境外用户专用]GPT01'] }
    - { name: 自动选择(不含IPLC), type: url-test, proxies: ['🇭🇰[HK]HongKong01-GPT优化', '🇭🇰[HK]HongKong02-GPT优化', '🇭🇰[HK]HongKong03-GPT优化', '🇭🇰[HK]HongKong04-GPT优化', '🇭🇰[HK]HongKong05-GPT优化', '🇭🇰[HK]HongKong06-GPT优化', '🇭🇰[HK]HongKong07-GPT优化', '🇭🇰[HK]HongKong08-GPT优化', '🇭🇰[HK]HongKong09-GPT优化', '🇭🇰[HK]HongKong10-GPT优化', '🇨🇳[CN]HK专线01-【5倍率】测试', '🇨🇳[CN]HK专线02-【5倍率】测试', '🇨🇳[CN]SG专线01-【5倍率】测试', '🇨🇳[CN]TW专线01-【5倍率】测试', '🇯🇵[JP]Tokyo01', '🇯🇵[JP]Tokyo02', '🇯🇵[JP]Tokyo03', '🇯🇵[JP]Tokyo04', '🇯🇵[JP]Tokyo06', '🇯🇵[JP]Tokyo07', '🇯🇵[JP]Tokyo08', '🇯🇵[JP]Tokyo09', '🇯🇵[JP]Tokyo10', '🇯🇵[JP]Tokyo11', '🇸🇬[SG]Singapore01', '🇸🇬[SG]Singapore02', '🇸🇬[SG]Singapore03', '🇸🇬[SG]Singapore04', '🇸🇬[SG]Singapore05', '🇸🇬[SG]Singapore06', '🇸🇬[SG]Singapore07', '🇸🇬[SG]Singapore08', '🇸🇬[SG]Singapore09', '🇸🇬[SG]Singapore10', '🇺🇸[US]美国Los Angeles01-GPT优化', '🇺🇸[US]美国Los Angeles02-GPT优化', '🇺🇸[US]美国Los Angeles03-GPT优化', '🇺🇸[US]美国Los Angeles04-GPT优化', '🇺🇸[US]美国Santa Clara05-GPT优化', '🇺🇸[US]美国Santa Clara06-GPT优化', '🇺🇸[US]美国San Jose07-GPT优化', '🇺🇸[US]美国San Jose08-GPT优化', '🇹🇼[TW]TaiPei01-GPT优化', '🇹🇼[TW]TaiPei02-GPT优化', '🇹🇼[TW]TaiPei03-GPT优化', '🇹🇼[TW]TaiPei04-GPT优化', '🇹🇼[TW]TaiPei05-GPT优化', '🇹🇼[TW]TaiPei06-GPT优化', '🇫🇷[FR]法国-Paris01', '🇫🇷[FR]法国-Paris02', '🇬🇧[UK]英国Coventry01-BBC优化', '🇬🇧[UK]英国Coventry02-BBC优化', '🇩🇪[DE]德国-Frankfurt01', '🇩🇪[DE]德国-Frankfurt02'], url: 'http://www.gstatic.com/generate_204', interval: 86400 }
    - { name: 负载均衡, type: load-balance, proxies: ['🇭🇰[HK]HongKong01-GPT优化', '🇭🇰[HK]HongKong02-GPT优化', '🇭🇰[HK]HongKong03-GPT优化', '🇭🇰[HK]HongKong04-GPT优化', '🇭🇰[HK]HongKong05-GPT优化', '🇭🇰[HK]HongKong06-GPT优化', '🇭🇰[HK]HongKong07-GPT优化', '🇭🇰[HK]HongKong08-GPT优化', '🇭🇰[HK]HongKong09-GPT优化', '🇭🇰[HK]HongKong10-GPT优化', '🇨🇳[CN]HK专线01-【5倍率】测试', '🇨🇳[CN]HK专线02-【5倍率】测试', '🇨🇳[CN]SG专线01-【5倍率】测试', '🇨🇳[CN]TW专线01-【5倍率】测试', '🇯🇵[JP]Tokyo01', '🇯🇵[JP]Tokyo02', '🇯🇵[JP]Tokyo03', '🇯🇵[JP]Tokyo04', '🇯🇵[JP]Tokyo06', '🇯🇵[JP]Tokyo07', '🇯🇵[JP]Tokyo08', '🇯🇵[JP]Tokyo09', '🇯🇵[JP]Tokyo10', '🇯🇵[JP]Tokyo11', '🇸🇬[SG]Singapore01', '🇸🇬[SG]Singapore02', '🇸🇬[SG]Singapore03', '🇸🇬[SG]Singapore04', '🇸🇬[SG]Singapore05', '🇸🇬[SG]Singapore06', '🇸🇬[SG]Singapore07', '🇸🇬[SG]Singapore08', '🇸🇬[SG]Singapore09', '🇸🇬[SG]Singapore10', '🇺🇸[US]美国Los Angeles01-GPT优化', '🇺🇸[US]美国Los Angeles02-GPT优化', '🇺🇸[US]美国Los Angeles03-GPT优化', '🇺🇸[US]美国Los Angeles04-GPT优化', '🇺🇸[US]美国Santa Clara05-GPT优化', '🇺🇸[US]美国Santa Clara06-GPT优化', '🇺🇸[US]美国San Jose07-GPT优化', '🇺🇸[US]美国San Jose08-GPT优化', '🇹🇼[TW]TaiPei01-GPT优化', '🇹🇼[TW]TaiPei02-GPT优化', '🇹🇼[TW]TaiPei03-GPT优化', '🇹🇼[TW]TaiPei04-GPT优化', '🇹🇼[TW]TaiPei05-GPT优化', '🇹🇼[TW]TaiPei06-GPT优化', '🇫🇷[FR]法国-Paris01', '🇫🇷[FR]法国-Paris02', '🇬🇧[UK]英国Coventry01-BBC优化', '🇬🇧[UK]英国Coventry02-BBC优化', '🇩🇪[DE]德国-Frankfurt01', '🇩🇪[DE]德国-Frankfurt02', 🇭🇰测试节点, 🇭🇰香港1, 🇭🇰香港2, 🇸🇬新加坡1, 🇸🇬新加坡2, 🇨🇳台湾1-Gpt, 🇨🇳台湾2-Gpt, 🇯🇵日本1, 🇯🇵日本2, 🇰🇷韩国-家宽, 🇺🇸拉斯维加斯, 🇺🇸西雅图, 🇺🇸圣何塞, 🇩🇪德国, 🇬🇧英国, 测试-trojan香港1, 测试-trojan香港2, 🇭🇰香港01-IPv6, 🇭🇰香港02-IPv6, 🇭🇰香港03-IPv6, 🇭🇰香港04-IPv6, 🇭🇰香港05-流媒体解锁, 🇭🇰香港06-流媒体解锁, 🇸🇬新加坡01-流媒体解锁, 🇸🇬新加坡02-流媒体解锁, 🇸🇬新加坡03, 🇸🇬新加坡04, 🇰🇷韩国-家宽03, 🇰🇷韩国-家宽04, 🇨🇳台湾01-Gpt, 🇨🇳台湾02-Gpt, 🇨🇳台湾03-Gpt, 🇨🇳台湾04-Gpt, 🇯🇵日本01, 🇯🇵日本02, 🇯🇵日本03, 🇯🇵日本04, 🇨🇦加拿大01, 🇨🇦加拿大02, 🇺🇸美国西雅图01, 🇺🇸美国西雅图02, 🇺🇸美国拉斯维加斯01, 🇺🇸美国拉斯维加斯02, 🇺🇸美国圣何塞01, 🇺🇸美国圣何塞02, 🇩🇪德国01, 🇩🇪德国02, 🇬🇧英国01, 🇬🇧英国02, '🇺🇸拉斯维加斯01-下载&防失联-0.1X', '🇺🇸拉斯维加斯02-下载&防失联-0.1X', '🇺🇸拉斯维加斯03-下载&防失联-0.1X', '🇺🇸拉斯维加斯04-下载&防失联-0.1X'], strategy: round-robin , url: 'http://www.gstatic.com/generate_204', interval: 3 }
    - { name: 故障转移, type: fallback, proxies: ['剩余流量：9.46 GB', '距离下次重置剩余：3 天', 套餐到期：2026-01-30, '🇭🇰[HK]HongKong01-GPT优化', '🇭🇰[HK]HongKong02-GPT优化', '🇭🇰[HK]HongKong03-GPT优化', '🇭🇰[HK]HongKong04-GPT优化', '🇭🇰[HK]HongKong05-GPT优化', '🇭🇰[HK]HongKong06-GPT优化', '🇭🇰[HK]HongKong07-GPT优化', '🇭🇰[HK]HongKong08-GPT优化', '🇭🇰[HK]HongKong09-GPT优化', '🇭🇰[HK]HongKong10-GPT优化', '🇨🇳[CN]HK专线01-【5倍率】测试', '🇨🇳[CN]HK专线02-【5倍率】测试', '🇨🇳[CN]SG专线01-【5倍率】测试', '🇨🇳[CN]TW专线01-【5倍率】测试', '🇯🇵[JP]Tokyo01', '🇯🇵[JP]Tokyo02', '🇯🇵[JP]Tokyo03', '🇯🇵[JP]Tokyo04', '🇯🇵[JP]Tokyo06', '🇯🇵[JP]Tokyo07', '🇯🇵[JP]Tokyo08', '🇯🇵[JP]Tokyo09', '🇯🇵[JP]Tokyo10', '🇯🇵[JP]Tokyo11', '🇸🇬[SG]Singapore01', '🇸🇬[SG]Singapore02', '🇸🇬[SG]Singapore03', '🇸🇬[SG]Singapore04', '🇸🇬[SG]Singapore05', '🇸🇬[SG]Singapore06', '🇸🇬[SG]Singapore07', '🇸🇬[SG]Singapore08', '🇸🇬[SG]Singapore09', '🇸🇬[SG]Singapore10', '🇺🇸[US]美国Los Angeles01-GPT优化', '🇺🇸[US]美国Los Angeles02-GPT优化', '🇺🇸[US]美国Los Angeles03-GPT优化', '🇺🇸[US]美国Los Angeles04-GPT优化', '🇺🇸[US]美国Santa Clara05-GPT优化', '🇺🇸[US]美国Santa Clara06-GPT优化', '🇺🇸[US]美国San Jose07-GPT优化', '🇺🇸[US]美国San Jose08-GPT优化', '🇹🇼[TW]TaiPei01-GPT优化', '🇹🇼[TW]TaiPei02-GPT优化', '🇹🇼[TW]TaiPei03-GPT优化', '🇹🇼[TW]TaiPei04-GPT优化', '🇹🇼[TW]TaiPei05-GPT优化', '🇹🇼[TW]TaiPei06-GPT优化', '🇫🇷[FR]法国-Paris01', '🇫🇷[FR]法国-Paris02', '🇬🇧[UK]英国Coventry01-BBC优化', '🇬🇧[UK]英国Coventry02-BBC优化', '🇩🇪[DE]德国-Frankfurt01', '🇩🇪[DE]德国-Frankfurt02', '🇨🇦[CA]加拿大-Toronto', '🇳🇱[NL]荷兰-Amsterdam', '🇷🇺[RU]俄罗斯-Moscow', '🇹🇷[TR]土耳其-Istanbul', '🇮🇳[IN]印度-bangalore', '🇰🇷[KR]韩国-Seoul', '🇻🇳[VN]越南-HoChiMinh', '🇦🇺[AU]澳大利亚-Sydney', '🇳🇬[NG]尼日利亚-Lagos', '🇦🇷[AR]阿根廷-BuenosAires', '🇲🇰[MK]马其顿-Macedonia', '🇦🇪[AE]阿联酋-ArabEmirates', '[境外用户专用]GPT01'], url: 'http://www.gstatic.com/generate_204', interval: 7200 }
rules:
    - 'DOMAIN-SUFFIX,deepseek.com,DIRECT'
    - 'DOMAIN-SUFFIX,cn,DIRECT' //cn一般是中国域名

    - 'MATCH,TaiShan Net'
```

