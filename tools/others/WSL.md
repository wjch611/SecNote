### 1、获取win的ip

```
ip route | grep default | awk '{print $3}'
```

