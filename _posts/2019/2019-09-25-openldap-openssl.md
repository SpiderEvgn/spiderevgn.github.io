---
layout: post
title: 在 CentOS 搭建 LDAP 服务器（三） - OpenLDAP TLS 与 OpenSSL
tags: [centos, ssl]
---

### 0. 引言

这是『在 CentOS 搭建 LDAP 服务器』系列的第三篇，[上一篇]({% post_url 2019/2019-09-22-openldap-config %})我们配置好了 OpenLDAP，这篇我们启用 TLS 加密，并且详细介绍利用 OpenSSL 证书的自签发过程。这里多解释一句，自签证书一般用作测试，商用证书一定要找权威 CA 签发（往往价格不菲），比如权威 CA 在主流浏览器都保存有根证书确保合法性，自签证书当然也可以配置到浏览器，但是，商用，你懂的。好了，我们还是专注技术本身。

### 1. 基本概念

本文将着重于介绍证书签发的具体步骤和命令，不会涉及太多关于 SSL 和密码学的基础知识，但是在实施之前还是有必要了解一些相关的概念，比如什么是 PEM、CRT、CSR 等，然后才能一步一步签出证书。关于加密的基础理论，一如既往地推荐阮一峰大神的博客：
* [密码学笔记](http://www.ruanyifeng.com/blog/2006/12/notes_on_cryptography.html)
* [SSL/TLS协议运行机制的概述](http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html)
* [数字签名是什么？](http://www.ruanyifeng.com/blog/2011/08/what_is_a_digital_signature.html)
* [RSA 算法原理（一）](http://www.ruanyifeng.com/blog/2013/06/rsa_algorithm_part_one.html)

如果对密码学和 SSL/TLS 的概念一知半解，强烈建议先读完以上的文章，然后再看本文地实操，会更有的放矢。

下面笔者大致介绍一下证书签发过程中会遇到的名词：

* **公钥私钥：** 是基于 RSA 算法的一对密钥，可以互相加解密。
* **加密、解密：** 把公钥和信息（明文）通过某种算法生成新的信息（密文），叫做加密；然后用私钥和密文通过某种算法生成原来的明文，叫做解密。
* **数字签名：** 先把信息（明文）通过某种算法生成摘要（digest），然后用私钥对摘要加密，加密后的密文就是数字签名。反过来，用私钥对数字签名解密后得到摘要，然后用同样的已知算法对信息（明文）重新生成摘要 ，比对数字签名解密出来的摘要，以此确定发信人身份。
* **CA(Certificate Authority)：** 证书颁发机构，用来认证密钥的合法性，并签发数字证书。
* **数字证书：** CA 用自己的 CA 私钥将用户的个人信息以及用户公钥加密后生成的密文，叫做数字证书。反过来，通过 CA 公开的公钥对数字证书解密后就可以确认对方的身份和公钥。
* **CA 根证书：** CA 认证中心给自己颁发的证书，是信任链的起始点，安装根证书意味着对这个 CA 认证中心的信任。
* **CSR(Certificate Signing Request)：** 用来向 CA 发起证书签发的请求，里面包含公钥和认证信息。公司、机构或个人可以包装自己的 CSR，然后向 CA 请求签发证书

### 2. OpenSSL 与证书签发

下面我们就从创建自己的 CA 开始，一步步生成自签证书

创建 CA 私钥
```
$ openssl genrsa -aes256 -out ca.key.pem
```
创建 CA 根证书
```
$ openssl req -new -x509 -days 365 -key ca.key.pem -extensions v3_ca -out ca.cert.pem
```
查看 CA 私钥内容
```
$ openssl rsa -in ca.key.pem -text -noout
```
查看 CA 根证书内容
```
$ openssl x509 -in ca.cert.pem -text -noout
```
生成自己的服务器私钥
```
$ openssl genrsa -out server.key.pem
```
用自己的服务器私钥生成 CSR，注意：Country、Location 和 OrganizationName 必须保持一致
```
$ openssl req -new -key server.key -out server.csr
```
查看 CSR
```
$ openssl req -in server.csr -text -noout
```
用自己创建的 CA 签发证书
```
$ openssl ca -keyfile ca.key.pem -cert ca.cert.pem -in server.csr -out server.crt
```

### 3. OpenLDAP server 配置开启 tls

服务端配置证书，将刚才生成的各种证书放入相应文件夹，然后编写证书配置文件：

```
$ vi certs.ldif
dn: cn=config
changetype: modify
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/openldap/certs/server.crt

dn: cn=config
changetype: modify
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/openldap/certs/server.key.pem

dn: cn=config
changetype: modify
replace: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/openldap/cacerts/ca.cert.pem
```

运行配置
```
$ ldapmodify -Y EXTERNAL -H ldapi:/// -f certs.ldif
```

查看证书情况
```
$ slapcat -b "cn=config" | egrep "olcTLSCertificateFile|olcTLSCertificateKeyFile|olcTLSCACertificateFile"
```

最后重启服务
```
$ systemctl restart slapd
```

### 4. OpenLDAP client 配置开启 tls

将『server.crt』证书放到每台 client 的 `/etc/openldap/cacerts` 文件夹底下

然后修改每台客户端的 sssd 配置文件：

```
$ vi /etc/sssd、sssd.conf

    [... other config]

    ldap_id_use_start_tls = True
    ldap_tls_cacertdir = /etc/openldap/cacerts
    ldap_tls_reqcert = allow
```

最后重启『sssd』服务即可
```
$ systemctl restart sssd
```

---

## 参考资料：

* [那些证书相关的玩意儿（SSL,X.509,PEM,DER,CRT,CER,KEY,CSR,P12等）](https://www.cnblogs.com/guogangj/p/4118605.html)
* [证书格式普及](https://blog.freessl.cn/ssl-cert-format-introduce/)
* [配置 OpenLDAP 使用 SSL/TLS 加密数据通信](https://www.ibm.com/developerworks/cn/linux/1312_zhangchao_opensslldap/)
* [SSL数字证书（三）使用 openssl 生成证书](https://blog.csdn.net/lwwl12/article/details/80909068)









