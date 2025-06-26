### 一、安装独立tor服务

专家包

### 二、配置torrc

解压：

![image-20250429173731023](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250429173731023.png)

data下存在torrc：

```
# 网桥
Bridge snowflake 192.0.2.3:80 2B280B23E1107BB62ABFC40DDCC8824814F80A72 fingerprint=2B280B23E1107BB62ABFC40DDCC8824814F80A72 url=https://1098762253.rsc.cdn77.org/ fronts=www.cdn77.com,www.phpmyadmin.net ice=stun:stun.antisip.com:3478,stun:stun.epygi.com:3478,stun:stun.uls.co.za:3478,stun:stun.voipgate.com:3478,stun:stun.mixvoip.com:3478,stun:stun.nextcloud.com:3478,stun:stun.bethesda.net:3478,stun:stun.nextcloud.com:443 utls-imitate=hellorandomizedalpn
Bridge snowflake 192.0.2.4:80 8838024498816A039FCBBAB14E6F40A0843051FA fingerprint=8838024498816A039FCBBAB14E6F40A0843051FA url=https://1098762253.rsc.cdn77.org/ fronts=www.cdn77.com,www.phpmyadmin.net ice=stun:stun.antisip.com:3478,stun:stun.epygi.com:3478,stun:stun.uls.co.za:3478,stun:stun.voipgate.com:3478,stun:stun.mixvoip.com:3478,stun:stun.nextcloud.com:3478,stun:stun.bethesda.net:3478,stun:stun.nextcloud.com:443 utls-imitate=hellorandomizedalpn

# 开启控制端口
ControlPort 9051

# 使用明文密码认证（推荐用 hash 更安全，见下文）
HashedControlPassword 16:46A3A2757B963B2D6091E54399674B7BD30B07B059DCCD640759C9FBEB

# 开启 SOCKS 代理端口（默认）
SocksPort 9050

# 允许通过 stem 控制
CookieAuthentication 0


# 加速的配置
# Try for at most NUM seconds when building circuits. If the circuit isn't
# open in that time, give up on it. (Default: 1 minute.)
CircuitBuildTimeout 5
# Send a padding cell every N seconds to keep firewalls from closing our
# connections while Tor is not in use.
KeepalivePeriod 60
# Force Tor to consider whether to build a new circuit every NUM seconds.
NewCircuitPeriod 15
# How many entry guards should we keep at a time?
NumEntryGuards 8
```

### 三、结果

1. 不用访问外网，也能快速连上tor，不够带宽很小。
2. 需要挂socket5代理9050端口

![image-20250429173942564](C:\Users\33940\AppData\Roaming\Typora\typora-user-images\image-20250429173942564.png)

命令：

```
tor -f D:\tor-expert-bundle-windows-x86_64-14.5\data\torrc
```

