\# CVE‑2016‑4437 Shiro‑550 反序列化漏洞复现



\## 漏洞说明

Apache‑Shiro是一个权限安全框架。

当开启rememberMe记住我的功能时，1.2.4版本代码内置了一个固定AES密钥。

攻击者拿到密钥就可以生成恶意序列化数据，加密放到rememberMe Cookie，服务器解密后就会执行我们构造的命令，造成远程代码执行。



漏洞编号：CVE‑2016‑4437

靶场：vulhub shiro:1.2.4



\## 环境信息

攻击机：Kali

靶机地址：http://192.168.1.81:8080



\## 复现过程

1、启动靶场

进入vulhub对应的shiro‑550文件夹，执行

```bash

docker-compose up -d



