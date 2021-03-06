# 业务协议 vul 整合笔记

---

## 免责声明

`本人撰写的手册,仅供学习和研究使用,请勿使用文中的技术源码用于非法用途,任何人造成的任何负面影响,与本人无关。`

---

# dns
**CVE-1999-0532** dns 域传送漏洞
```bash
dig @dns.xxx.edu.cn axfr xxx.edu.cn
@ 指定域名服务器；axfr 为域传送指令；xxx.edu.cn 表示要查询的域名；
```

**子域接管漏洞**
- 文章/案例
    - [EdOverflow/can-i-take-over-xyz](https://github.com/EdOverflow/can-i-take-over-xyz/)
    - [HackerOne 漏洞案例 | 子域名劫持漏洞的挖掘指南](https://www.freebuf.com/articles/web/183254.html)
    - [A Guide To Subdomain Takeovers | HackerOne](https://www.hackerone.com/blog/Guide-Subdomain-Takeovers)

---

# ipv6
**文章**
- [浅谈 IPv6 的入侵与防御](https://www.freebuf.com/articles/web/202901.html)

**工具**
- [vanhauser-thc/thc-ipv6](https://github.com/vanhauser-thc/thc-ipv6)
- [fgont/ipv6toolkit](https://github.com/fgont/ipv6toolkit)

---

# SMTP

SMTP 为邮件协讫，默认端口 25。经常用来邮箱伪造，钓鱼攻击。

**文章**
- [SMTP 协议 25 端口渗透测试记录](https://www.sqlsec.com/2017/08/smtp.html)
- [SMTP 用户枚举原理简介及相关工具](https://www.freebuf.com/articles/web/182746.html)

**邮件检测**
- SPF : http://spf.myisp.ch/
- MX : https://toolbox.googleapps.com/apps/checkmx
- DMARC : https://www.agari.com/insights/tools/DMARC

**枚举用户**
```
MAIL FROM   : 指定发件人地址
RCPT TO : 指定单个的邮件接收人；可有多个 RCPT TO；常在 MAIL FROM命令之后
VRFY    : 用于验证指定的用户/邮箱是否存在；由于安全原因，服务器常禁止此命令
EXPN    : 验证给定的邮箱列表是否存在，也常被禁用
```
```
SMTP 返回码
500 	格式错误，命令不可识别（此错误也包括命令行过长）
501 	参数格式错误
502 	命令不可实现
503 	错误的命令序列
504 	命令参数不可实现
211 	系统状态或系统帮助响应
214 	帮助信息
220 	服务就绪
221 	服务关闭传输信道
421 	服务未就绪，关闭传输信道（当必须关闭时，此应答可以作为对任何命令的响应）
250 	要求的邮件操作完成
251 	用户非本地，将转发向
450 	要求的邮件操作未完成，邮箱不可用（例如，邮箱忙）
550 	要求的邮件操作未完成，邮箱不可用（例如，邮箱未找到，或不可访问）
451 	放弃要求的操作；处理过程中出错
551 	用户非本地，请尝试
452 	系统存储不足，要求的操作未执行
552 	过量的存储分配，要求的操作未执行
553 	邮箱名不可用，要求的操作未执行（例如邮箱格式错误）
354 	开始邮件输入，以.结束
554 	操作失败
535 	用户验证失败
235 	用户验证成功
334 	等待用户输入验证信息
```

可以通过 Telnet 连接，在未禁用上述 SMTP 命令的服务器上，使用上述命令手动枚举用户名。
```
$ telnet xxx.xxx.xxx.xxx 25
Trying xxx.xxx.xxx.xxx...
Connected to xxx.xxx.xxx.xxx.
Escape character is '^]'.
220 mxt.xxx.xxx.cn ESMTP Postfix
VRFY root
252 2.0.0 root
VRFY bin
252 2.0.0 bin
VRFY admin
550 5.1.1 <admin>: Recipient address rejected: User unknown in local recipient table

$ telnet xxx.xxx.xxx.xxx 25
Trying xxx.xxx.xxx.xxx...
Connected to xxx.xxx.xxx.xxx.
Escape character is '^]'.
220 mxt.xxx.xxx.cn ESMTP Postfix
MAIL FROM:root
250 2.1.0 Ok
RCPT TO:root
250 2.1.5 Ok
RCPT TO:bin
250 2.1.5 Ok
RCPT TO:admin
550 5.1.1 <admin>: Recipient address rejected: User unknown in local recipient table
```
可以看到两种方式均返回 root、bin 用户是存在的，admin 用户不存在。


- smtp-user-enum - kali 自带
    ```
    smtp-user-enum -M VRFY -u root -t xxx.xxx.xxx.xxx
    smtp-user-enum -M RCPT -u bin -t xxx.xxx.xxx.xxx
    smtp-user-enum -M EXPN -u bin -t xxx.xxx.xxx.xxx
    ```

- MSF 模块
    ```
    use auxiliary/scanner/smtp/smtp_enum
    set rhosts xxx.xxx.xxx.xxx
    run
    ```

- smtp-enum-users - nmap 脚本
    ```
    nmap -p 25 --script smtp-enum-users.nse xxx.xxx.xxx.xxx
    ```

**邮件伪造**
```
telnet xxx.xxx.xxx.xxx 25
HELO xxxx.com   # 向邮件服务器提供连接的域名，也就是邮件将从哪台服务器发来。
MAIL FROM:admin@xxxx.com  # 伪造管理员身份来发邮件
RCPT TO:admin@xxxxx.com # 验证邮件地址是否存在。如果查询的是一个真实的 Email 地址，邮件服务器就会返回250状态码。验证邮箱存在的话，还可以给这个接受者邮箱发送邮件。
DATA    # 使用DATA命令来伪造邮箱内容,客户端告诉服务器自己准备发送邮件正文
354 go ahead, end data with CRLF.CRLF   # 服务器返回354，表示自己已经作好接受邮件的准备
"这是一个邮件伪造测试"  # 用英文状态的双引号来修饰正文，正文结束后，发送结束符.表明正文的结束。
.
QUIT
```

---

# SNMP
**文章**
- [SNMP 协议攻击](https://xz.aliyun.com/t/1562)
- [渗透测试中如何收集利用 SNMP 协议数据](https://www.freebuf.com/articles/network/104522.html)

**工具**
- [trailofbits/onesixtyone](https://github.com/trailofbits/onesixtyone)

**snmp 弱口令**
- 文章
    - [烂泥：使用 snmpwalk 采集设备的 OID 信息](https://www.ilanni.com/?p=8408)
    - [snmp 弱口令引起的信息泄漏](http://drops.xmd5.com/static/drops/tips-409.html)

- 利用

    安装 `yum -y install net-snmp-utils`

    Usage
    ```bash
    snmpwalk -v 2c -c public 192.168.xxx.xxx
    snmp-check 192.168.xxx.xxx -c public -v 2c
    ```

---

# SSL
**文章**
- [SSL＆TLS 安全性测试](https://www.mottoin.com/detail/1214.html)

**工具**
- [MassBleed](https://github.com/1N3/MassBleed)
- [sslscan](https://github.com/rbsec/sslscan)
- [testssl](https://github.com/drwetter/testssl.sh)

**CVE-2014-3566 SSL 3.0 POODLE 攻击信息泄露漏洞**
- 文章
    - [CVE-2014-3566 SSLv3 POODLE 原理分析](http://drops.xmd5.com/static/drops/papers-3194.html)

- POC | Payload | exp
    - [mpgn/poodle-PoC](https://github.com/mpgn/poodle-PoC)

- MSF 模块
    ```
    use auxiliary/scanner/http/ssl_version
    ```

**OpenSSL**
- **CVE-2014-0160 Heartbleed**
    - 文章
        - [心脏滴血 HeartBleed 漏洞研究及其 POC](https://www.cnblogs.com/KevinGeorge/p/8029947.html)
        - [OpenSSL 心脏出血漏洞全回顾](https://www.freebuf.com/articles/network/32171.html)
        - [Heartbleed 实战：一个影响无数网站的缓冲区溢出漏洞](https://onebitbug.me/reproduced/2014/04/09/heartbleed-poc/)
        - [Heartbleed 漏洞的复现与利用](https://blog.csdn.net/biziwaiwai/article/details/79323334)
        - [Heartbleed 心脏出血漏洞靶场搭建](https://blog.csdn.net/yaofeino1/article/details/54377537)

    - 复现实验
        - [心脏出血环境搭建实验](../运维/Linux/实验/心脏出血环境搭建实验.md)

    - POC | Payload | exp
        - [OpenSSL TLS Heartbeat Extension - 'Heartbleed' Memory Disclosure](https://www.exploit-db.com/exploits/32745)

