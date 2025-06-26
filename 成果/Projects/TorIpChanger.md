### 一、功能&原理

免费的ip代理池，虽然带宽很小，因为是tor的。原理就是使用stem一段时间后重新请求一下tor连接，并且访问一个ip查询网站，确认新的ip。

### 二、代码

```py
import time
import random
import requests
from stem import Signal
from stem.control import Controller
from stem.connection import AuthenticationFailure

# ========== 配置项 ==========
TOR_CONTROL_PORT = 9051
TOR_PASSWORD = "cww123"  # 明文密码，对应 tor --hash-password 生成的密码
CHECK_IP_URLS = [
    "https://checkip.amazonaws.com",
    "https://api.ipify.org",
    "http://icanhazip.com",
    "https://ident.me"
]
PROXY = {
    "http": "socks5h://127.0.0.1:9050",
    "https": "socks5h://127.0.0.1:9050",
}
MAX_RETRIES = 3  # 最大重试次数
BASE_DELAY = 5   # 基础等待时间(秒)
# ===========================

def get_ip(retry_count=0):
    """获取当前IP地址，带自动重试功能"""
    for url in CHECK_IP_URLS:
        try:
            response = requests.get(url, proxies=PROXY, timeout=10)
            if response.status_code == 200:
                return response.text.strip()
        except Exception as e:
            if retry_count < MAX_RETRIES:
                time.sleep(BASE_DELAY * (retry_count + 1))
                return get_ip(retry_count + 1)
            return f"[!] 获取IP失败(已重试{MAX_RETRIES}次): {str(e)}"
    return "[!] 所有IP检测服务均不可用"

def change_ip(retry_count=0):
    """更换Tor出口IP，带错误处理和重试"""
    try:
        with Controller.from_port(port=TOR_CONTROL_PORT) as controller:
            try:
                controller.authenticate(password=TOR_PASSWORD)
                controller.signal(Signal.NEWNYM)
                time.sleep(15)  # 必须等待足够时间让Tor重建电路
                return True
            except AuthenticationFailure:
                if retry_count < MAX_RETRIES:
                    time.sleep(BASE_DELAY * (retry_count + 1))
                    return change_ip(retry_count + 1)
                print("[!] Tor控制端口认证失败(请检查密码)")
                return False
    except Exception as e:
        if retry_count < MAX_RETRIES:
            time.sleep(BASE_DELAY * (retry_count + 1))
            return change_ip(retry_count + 1)
        print(f"[!] Tor控制连接失败: {str(e)}")
        return False

def verify_tor_connection():
    """验证Tor代理是否正常工作"""
    try:
        test_url = "https://check.torproject.org"
        response = requests.get(test_url, proxies=PROXY, timeout=15)
        return "Congratulations" in response.text
    except:
        return False

def main():
    print("=== Tor IP更换工具 ===")
    
    if not verify_tor_connection():
        print("[!] 无法连接到Tor网络，请检查:")
        print("1. Tor服务是否运行")
        print("2. 9050端口是否开放")
        print("3. 防火墙设置")
        return

    try:
        interval = int(input("输入更换间隔(秒，0=随机10-20秒): "))
        times = int(input("输入更换次数(0=无限循环): "))
    except ValueError:
        print("[!] 请输入有效数字")
        return

    count = 0
    while True:
        if times > 0 and count >= times:
            break

        print(f"\n[{count+1}] 正在更换IP...")
        if change_ip():
            new_ip = get_ip()
            print(f"当前IP: \033[92m{new_ip}\033[0m")
        else:
            print("[!] IP更换失败，跳过本次")

        count += 1
        current_interval = random.randint(10, 20) if interval == 0 else interval
        time.sleep(current_interval)

if __name__ == "__main__":
    main()
```



