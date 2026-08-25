# Shiro-550 反序列化 RCE 复现（CVE-2016-4437）

---

## 漏洞原理

Apache Shiro ≤ 1.2.4 中，`rememberMe` 功能使用了一个硬编码的 AES 密钥（`kPH+bIxk5D2deZiIxcaaaA==`）。攻击者可利用该密钥构造恶意序列化对象，加密后作为 Cookie 发送。服务端解密并反序列化时触发任意代码执行。

---

## 环境信息

- 攻击机：Kali Linux
- 靶场：Vulhub / shiro 1.2.4
- 靶机地址：`http://192.168.1.81:8080`
- 工具：Burp Suite / ysoserial / Python

---

## 复现过程

### 1. 启动靶场

```bash
cd vulhub/shiro/CVE-2016-4437
docker-compose up -d
