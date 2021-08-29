---
title: 实操理解 [自签ca证书]
date: 2021-08-20 23:49:20
author: yuxuan
img: https://image.fyxemmmm.cn/blog/fj/images/fj-17.jpg
top: false
hide: false
cover: false
coverImg: 
toc: true
mathjax: false
summary: 理解自签ca以及如何签发证书
categories: 加密与安全
tags:
  - 证书
  - ca
  - 自签
---

# 自签ca证书，及颁发证书

cd /etc/pki/CA

## 服务端生成ca证书

自签名证书，依赖自己的私钥

1. 生成私钥文件

   **(umask 077;openssl genrsa -out private/cakey.pem 4096)**

2. 直接生成自签名的证书

   **openssl req -new -x509 -key private/cakey.pem -out cacert.pem -days 3650**

   > `x509代表生成的是自签名的证书，不带x509代表是向ca生成的证书，不是自签名的` 
   >
   > **要借助私钥文件生成自签名的证书**

   ![ca证书生成](https://image.fyxemmmm.cn/blog/images/crt1.png)

3. 查看证书信息

   **openssl x509 -in cacert.pem -noout -text**

   ![ca证书](https://image.fyxemmmm.cn/blog/images/crt2.png)

## 服务端生成证书数据库

> 这里由于我们是自己的宿主机作为ca的管理、吊销及维护，所以需要一个数据库

1. **touch /etc/pki/CA/index.txt** 添加数据库文件

2. **touch /etc/pki/CA/serial** 新证书要从几开始编号，这个是记录用的，16进制数

3. **echo 0F > /etc/pki/CA/serial** 下次颁发证书的时候 就是15这个编号了

   ![颁发证书的编号](https://image.fyxemmmm.cn/blog/images/crt3.png)

## 客户端侧用户申请证书

> 证书可能是给某个服务用，比如给https服务用，https可能有自己配置数据的文件夹

1. 生成应用私钥

   **(umask 066; openssl genrsa -out app.key 1024)**

   > 注: 你申请证书的时间不是由你说了算，是颁发机构说了算

2. 生成csr文件

   **openssl req -new -key app.key -out app.csr**  `之前ca通过x509参数生成的是自签名的证书 而且可以有有效期，这里是生成证书请求文件`

   ![三项必须一致，且commonName填写服务的名称](https://image.fyxemmmm.cn/blog/images/crt4.png)

   

## 服务端颁发证书

**openssl ca -in /root/app.csr -out certs/app.crt -days 365 app.crt**是别人用私钥生成的证书请求文件，会读取服务端宿主机的配置文件，所以没指定ca的私钥，也不需要的 `这就是用服务端的ca自签名`

![](https://image.fyxemmmm.cn/blog/images/crt5.png)

如果提交的东西都对，就可以y 进行签名了

![](https://image.fyxemmmm.cn/blog/images/crt6.png)

证书申请完毕， 然后发给客户端使用 即可

---

## 其他

查看证书有效性

**openssl ca -status 0F**

查看证书的有效期

**openssl x509 -in app.crt -noout -dates**

