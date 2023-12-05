---
title: https 浅析
description: 
date: 2023-11-30
slug: simple_https
---

https

# 1      前言

https作为已经广泛使用的

# 2      算法Algorithm

## 2.1    数字签名算法Digital Signature Algorithm (DSA)

该算法使用由**公钥**和**私钥**组成的密钥对。私钥用于生成消息的数字签名，并且可以通过使用签名者的相应公钥来验证这种签名。数字签名提供信息鉴定（接收者可以验证消息的来源），完整性（接收方可以验证消息自签名以来未被修改）和不可否认性（发送方不能错误地声称它们没有签署消息）

RSA-SHA-1

RSA-SHA-256

RSA-SHA-3

ECC-DSA

Ed-DSA

​          ![1-hash](/images/https/1-hash.png)

## 2.2    非对称加密算法 Asymmetric algorithms

在公钥加密法中使用的特别技术是不对称密钥算法，这个算法中用来加密信息的密钥与解密信息的密钥不是同一个。每个用户都有一对加密密钥 — 一个公钥和一个私钥。私钥是秘密保存，而公钥则会广泛传播。信息是使由接收方到公钥加密，且该信息只能使用对应的私钥解密。

常用的非对称加密算法有 RSA，非对称加密算法虽然可以解决密钥配送的问题，但是它的加密速度比较慢，并且无法抵御**中间人攻击。**

 ![2-aes](/images/https/2-aes.png)

## 2.3    对称加密算法 Symmetric Algorithm

![3-sas](/images/https/3-sas.png)

 

## 2.4    密钥交换算法 Diffie-Hellman Key Exchange

Diffie-Hellman 密钥交换（D-H）是可让事先彼此不了解的双方通过不稳定的通讯频道联合建立共享保密密钥的加密协议。

![4-alice-bob](/images/https/4-alice-bob.jpg) 

## 2.5    椭圆曲线 ECDH Elliptic Curve Diffie-Hellman Key Exchange

https://www.youtube.com/watch?v=yDXiDOJgxmg

 

 ![5-fuction](/images/https/5-fuction.png)

 

 ![6-function](/images/https/6-function.png)

 

 ![7-function](/images/https/7-function.png)

 

 ![8-fucntion](/images/https/8-fucntion.png)

![9-fucntion](/images/https/9-fucntion.png)

![10-fucntion](/images/https/10-fucntion.png)

![11-fucntion](/images/https/11-fucntion.png)

![12-fucntion](/images/https/12-fucntion.png)





# 3      证书文件

证书是基于道德、公信的基础，对CA/Root颁发机构的信任。

## 3.1    X.509

X.509是密码学里公钥证书的格式标准。

证书组成结构标准用ASN.1（一种标准的语言）来进行描述。

 ![13-x509](/images/https/13-x509.png)

## 3.2    Private Key And Public Key

由工具生成的密钥对 使用RSA算法(非对称加密算法)。

 ![14-rsa](/images/https/14-rsa.png)

```shell
-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQDfywgJ6EIKwt81
rI9r08yiq+f4Umw9mNuPS3HMopyV6KJYudqLOsICbJt6GyI6NdFAoitZ6pcUnlEr
Hw54At8rKNZffcCR9Svjka+u6gF1we8JMSE0I/GoImWZXhJ3gy7KJ+rD84rFwbta
mJE6faFbjFeU4+f3sLrf3K71acQueGsrBZKHc7Es6N6MK8GEbw1OMLFubdp2vosA
PUpCk6oUfnH0AvUX3OP/OZV+REv+apS0ktCoCZciua/6612qblSWW/8GWAJqv17b
jd2qenYnkVkhuLXMhpwDPAc8Y494yfyiWdrHJ99POdk7u8NsFaN7z/DGZGeq+NEJ
EZb5L7h1AgMBAAECggEAVH6rwlHW3YlGDVXhsKx/DswpATDdfURBYZDynnim9lKn
OSiywu6kYZXv/eJQwfmfz+9qvsA72qULsGRBaj5rVDhO+C7ajkErGPjghAIGGxfl
0GqkRrNrgje6dHV4M3dsKxd3JBTHyHKk8ke4TYUxbwdF6glCg9pONEd2J2KPl5ta
2OfrrvFwOeix9QyUcT4mmTA2fKkuii8P8BPuARcVYLq5DBYt8WiLWey4vpaCoUc1
ZTcaKhCi/tjwMAsqchWIvrdChQOaphwTbZfPE/oAf9D+ov4p1MM7nDMD08idlOJz
TsSvSWh0zPzrwil7ipLRkX/fj7CCzwb2O4rtzPKC4QKBgQDycXNm0S6iXdQJBZGh
0acTigiSrrRYawI+5gegwBEd25T7Wcb4i9P483DR5IB1EfpHD6hXpf+kha9vrY5Z
NT4h05WvQp/KtV21/C6kI5k4T25mpBsIY2b+iyiOXlkaoxHqhBYDASBHbGjmGUmY
hqI1IiFoRlY61NAC+vfqC0OLPwKBgQDsTptPerT32QEDPJ98//n4CPLnaOeovwjr
BmNeotHFqjg9VvPKztgJOwsy44TCEX/gl/Un92pm/WxsaDplE3ClGgL3p9iqfUc9
q9tpFDxShHFtRsYu/oyduSVEUogVWNKN7QS2G12beLC+1b++mCm5n8VdtsOTY+v/
xb/b3B/TSwKBgQDkNjrE278kA2JmI6HUSr8Uu2gaeu00FXaFso4XmPQDwQBaIUYU
C7s6qhzW1lq82HFYlrqF1rHvMg/T9fD6tA2KVdqeoP49F7/gYEOfKgs+YDax02PG
35rBnEhOyyzg0AM7V55IsbSqxrdvcPo/4uupTDlaKGte8ZfkVk0rN/MajQKBgChh
WlrfjhMYSvsBngNfPpjq9o8itwt38Y8v3UUrr4sGhmu88xYB+JrDMyu0A1iiYua/
MM5ukgkdXyy7NtdU1hfwdPdbAERJ+iWIu4qeQZycM0HIKU+YgfDl1X9yVvzG29wS
145C6OELY7CImCZ6nA6zRae49ny2Q3rGkP2CBRI3AoGAU/6czO8x78qEk8dZQucH
JDiq6OuksffsoRONZE9mEUmreAQr+CW+gWmzfYihq/2ubf7CA7DB3LEsLLlaxUWp
QrBV7y65rTSQv1wDvJPcbXhT2XtG8rOY+IKJSQuLxP8FSjCUCodvkQBt+XUnJi9f
ERco+1Vln3uNjUrrgH+ywFI=
-----END PRIVATE KEY-----
```



## 3.3    CSR Sertificate Signing Request (also CSR or certification request)

The PKCS#10 standard defines a binary format for encoding CSRs for use with X.509. It is expressed in ASN.1. Here is an example of how you can examine its ASN.1 structure using OpenSSL:

```shell
openssl asn1parse -i -in your_request
```

A CSR may be represented as a Base64 encoded PKCS#10; an example of which is given below:

```shell
-----BEGIN CERTIFICATE REQUEST-----
MIICzDCCAbQCAQAwgYYxCzAJBgNVBAYTAkVOMQ0wCwYDVQQIDARub25lMQ0wCwYD
VQQHDARub25lMRIwEAYDVQQKDAlXaWtpcGVkaWExDTALBgNVBAsMBG5vbmUxGDAW
BgNVBAMMDyoud2lraXBlZGlhLm9yZzEcMBoGCSqGSIb3DQEJARYNbm9uZUBub25l
LmNvbTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAMP/U8RlcCD6E8AL
PT8LLUR9ygyygPCaSmIEC8zXGJung3ykElXFRz/Jc/bu0hxCxi2YDz5IjxBBOpB/
kieG83HsSmZZtR+drZIQ6vOsr/ucvpnB9z4XzKuabNGZ5ZiTSQ9L7Mx8FzvUTq5y
/ArIuM+FBeuno/IV8zvwAe/VRa8i0QjFXT9vBBp35aeatdnJ2ds50yKCsHHcjvtr
9/8zPVqqmhl2XFS3Qdqlsprzbgksom67OobJGjaV+fNHNQ0o/rzP//Pl3i7vvaEG
7Ff8tQhEwR9nJUR1T6Z7ln7S6cOr23YozgWVkEJ/dSr6LAopb+cZ88FzW5NszU6i
57HhA7ECAwEAAaAAMA0GCSqGSIb3DQEBBAUAA4IBAQBn8OCVOIx+n0AS6WbEmYDR
SspR9xOCoOwYfamB+2Bpmt82R01zJ/kaqzUtZUjaGvQvAaz5lUwoMdaO0X7I5Xfl
sllMFDaYoGD4Rru4s8gz2qG/QHWA8uPXzJVAj6X0olbIdLTEqTKsnBj4Zr1AJCNy
/YcG4ouLJr140o26MhwBpoCRpPjAgdYMH60BYfnc4/DILxMVqR9xqK1s98d6Ob/+
3wHFK+S7BRWrJQXcM8veAexXuk9lHQ+FgGfD0eSYGz0kyP26Qa2pLTwumjt+nBPl
rfJxaLHwTQ/1988G0H35ED0f9Md5fzoKi5evU1wG5WRxdEUPyt3QUXxdQ69i0C+7
-----END CERTIFICATE REQUEST-----
```

 

## 3.4    CRT Signed Certificate

### 3.4.1  Base64 Certificate Example:

```shell
-----BEGIN CERTIFICATE-----
MIIDizCCAnOgAwIBAgIBETANBgkqhkiG9w0BAQsFADByMQswCQYDVQQGEwJDTjES
MBAGA1UECAwJR3Vhbmdkb25nMR0wGwYDVQQKDBRHbG9iYWwgR29vZ2xlIENBIElu
YzEXMBUGA1UECwwOR29vZ2xlIDIwMTkgQ0ExFzAVBgNVBAMMDkdvb2dsZSAyMDE5
IENBMCAXDTIyMDQxNzAxMzEzM1oYDzIxMjIwMzI0MDEzMTMzWjBaMQswCQYDVQQG
EwJDTjESMBAGA1UECAwJQ2hvbmdRaW5nMQ0wCwYDVQQKDARTQUFTMRIwEAYDVQQL
DAl0ZXN0MS5kZXYxFDASBgNVBAMMCyoudGVzdDEuZGV2MIIBIjANBgkqhkiG9w0B
AQEFAAOCAQ8AMIIBCgKCAQEAxZDmJcwta8k70I9N90EPqBMS2+1fgsvtT+KO5GVG
7ON6sbVPE41KSHPakva9kdOmdxzHpElaFY0G2QLPuTaTXtOvTxaOCB3OrR61ZhHA
SPShO5I37QXaaeA1zYmLLJQh90tvZ13eleF8WHfjNNqbKKhc7QcqF4y8sjdS9GR6
0eKif5MEdB5XEEU+S4y4TtFTHj+1/o648FfYBLGL5eBNHG34c20rRyDtMgyOfHWN
R17F4GUVfguwwi1fZiM6xQxqf9YwaQzDTlKSRsVR0ebKOy2zvbP0SSuHFjh3l1Ox
JE09Ey18+EOuT99PyoDJj1tT77tOukl4DDU4RzvX5UkKyQIDAQABo0IwQDAJBgNV
HRMEAjAAMDMGA1UdEQQsMCqCCyoudGVzdDEuZGV2ggl0ZXN0MS5kZXaHBMCoCoCH
BMCoCoWHBMCoCoEwDQYJKoZIhvcNAQELBQADggEBAF3BUTdiHFGZXdidL9b2ugqn
tIfZrenywR2u3Y8BcY/c0+kQxYlUmOqaPuFUpyvwjAzDD8CVh9bLImN/fz+k5v0X
Y6gpkn2jHFRpxshE+EziXeP51fDphtQYovTAB0ip1xG1re4WWQePzgva/ZeYLRm7
NZ/Ja4vf89Ed7RHnufoFkr8BuTxeRnvr0OpD7g2VM1kncqW3+rlSqsxQf9m1ozDX
jG7AiKRQxkh6ej915gzHqvMPHiNohGkEq1wNDaPU1m5tGnhm5sAEyHyPdnxD8X19
h89bvx6MSWVtGHywB8C0hlVRUKWN1wlRYbMl6EZDZL9e2WchCvDrP0G8uVthoNs=
-----END CERTIFICATE-----
```

 

### 3.4.2  证书内容

```shell
$ openssl x509 -in acs.cdroutertest.com.pem -text
```

 ![15-cert-in](/images/https/15-cert-in.png)

 

# 4      HTTPS HTTP over TLS/SSL

Hypertext Transfer Protocol Secure

## 4.1    发展

 ![16-https](/images/https/16-https.png)

## 4.2    协议层 OSI Level

 ![17-https-osi](/images/https/17-https-osi.png)

 

## 4.3    HTTPS大概流程

使用非对称加密算法  à 解密证书的签名

使用数字签名算法  à 计算签名和解密的对比验证

使用非对称加密算法 à 加密报文完成[对称加密算法密钥协商]

使用对称加密算法 à 数据传输

| NO   | 过程               | 作用                               |
| ---- | ------------------ | ---------------------------------- |
| 1    | 使用非对称加密算法 | 解密证书的签名                     |
| 2    | 使用数字签名算法   | 计算签名和解密的对比验证           |
| 3    | 使用非对称加密算法 | 加密报文完成[对称加密算法密钥协商] |
| 4    | 使用对称加密算法   | 数据传输                           |

## 4.4    为什么两种加密算法结合使用

### 4.4.1  RSA缺点

密钥尺寸大，加解密速度慢，一般用来加密少量数据。速度太慢，由于RSA 的分组长度太大，为保证安全性，n 至少也要 600 bits以上，使运算代价很高，尤其是速度较慢，较对称密码算法慢几个数量级；且随着大数分解技术的发展，这个长度还在增加，不利于数据格式的标准化不具有向前的安全特性，私钥丢失就不安全了。

### 4.4.2  DH缺点

无法抵御中间人攻击。造成中间人攻击得逞的原因是：DH密钥交换算法不进行认证对方。利用数字签名可以解决中间人攻击的缺陷。具有向前的安全特性(每次生成的密钥不同)

## 4.5    证书签名与验证

### 4.5.1  证书链

 ![18-cert-chain](/images/https/18-cert-chain.png)

### 4.5.2  证书链验证

 ![19-cert-chain](/images/https/19-cert-chain.png)

### 4.5.3  证书详解签名与验证

 ![20-cert-verify](/images/https/20-cert-verify.png)

## 4.6    TLS 1.2 Handshake

### 4.6.1  RSA Handshake

 ![21-ssl-handshake](/images/https/21-ssl-handshake.jpg)

### 4.6.2  DH Handshake

 ![21-ssl-handshake-dh](/images/https/21-ssl-handshake-dh.jpg)

## 4.7    TLS 1.3 Handshake

 ![22-tls13-handshake](/images/https/22-tls13-handshake.png)

# 5      SNI CDN与多域名证书

## 5.1    背景

随着 IPv4 地址的短缺，为了让多个域名复用同一个 IP 地址，HTTP 服务引入了虚拟主机的概念。即服务器可以根据客户端不同的 Host 请求，将请求分配给不同的域名处理。一个 IP 地址对应多个域名。

但是在一个被多个域名所共享的 IP 服务（HTTPS）上，由于在握手建立之前服务器无法知道客户端请求的是哪个 Host，此时就出现了无法将请求交给特定的虚拟主机。所以就出现了上面的问题，服务端返回了缺省证书，但是该证书所对应域名并非客户端请求的 Host。

 ![23-sni](/images/https/23-sni.png)

## 5.2    HTTPS 为什么不行

基于Host的虚拟主机，Http没有问题，但是Https不行，为什么呢？因为Https使用SSL/TLS，需要SSL证书才能建立连接，但是同一个nginx上配置了多个SSL证书，使用哪个呢？根据Host选择？这是不可能的，因为Host是Http协议的内容，而这一切发生在发送Http请求前，此时Host还没发送呢

## 5.3    SNI 介绍

SNI (Server Name Indication) 是用来改善服务器与客户端 SSL (Secure Socket Layer) 和 TLS (Transport Layer Security) 的一个扩展。

主要解决多证书多域名的情况，随着服务器对虚拟主机的支持，一个服务器上可以为多个域名多个证书提供服务，因此SNI必须得到支持才能满足需求

## 5.4    SNI 的工作原理

它的工作原理是：在客户端与服务器建立 SSL 连接之前，Client Hello 阶段会通过 SNI 扩展，将要访问站点的域名（Hostname）提前告诉服务器，这样服务器会根据这个域名匹配对应的证书返回给客户端，以完成校验过程。

目前，大多数操作系统和浏览器都已经很好地支持SNI扩展

 ![24-sni-pcap](/images/https/24-sni-pcap.png)

 

## 5.5    SNI问题

对于多域名多证书的WEB前端，如果TLS Client Hello中没有包含SNI扩展(server_name)

服务器可能根据不同的厂商有几种可能:

l 返回默认证书(ali CDN)

l 返回错误Internal ERROR拒绝握手(akamai)

# 6      安全性

## 6.1    SNI - 域名裸奔

由于Client Hello报文明文传输server_name主机名，所有都可以监控你访问的域名。

因此，即便是 HTTPS，访问的域名信息也是裸奔状态。你上班期间访问小电影网站，都留下了痕迹，若接入了公司网络，就自然而然被抓个正着。

除了域名是裸奔外，其实还有更严重的风险，那就是中间人攻击

## 6.2    DNS - 请求裸奔

对域名发起请求前，要知道域名的IP地址，就要访问DNS服务器，公司只需网络中指定DNS服务器，截获不加密的 DNS 报文分分钟就了解“摸鱼”的情况

想要上网不被监控，除了上文中提到的 HTTPS，不要忘了安全的DNS，DoH(DNS over HTTPs, DoH)或DoT(DNS over TLS, DoT)。

目前比较好的方式还是，自己搭建DoH的DNS server，连上网络后设置DNS服务器为你的server IP，原生Android甚至在设置里提供了“私人DNS”选项。

当然如果还能跑一个代理服务，前面提到的SNI泄露访问域名的问题也一起解决了。抓包只能发现你一直在访问你自己的server。为了再真一点，甚至可以在你的server上再搭一个web server，放上一些内容，这样就算有人追踪到这个IP，打开看也是一个正常。

# 7      中间人攻击 MITM Man-in-the-middle attack

 ![25-mitm-1](/images/https/25-mitm-1.png)

 ![25-mitm-2](/images/https/25-mitm-2.png)

 

 

# 8      SSH协议

## 8.1    Private Key And Public Key

sshd_config

 ![26-ssh-1](/images/https/26-ssh-1.png)

 ![26-ssh-2](/images/https/26-ssh-2.png)

 ![26-ssh-3](/images/https/26-ssh-3.png)

 

## 8.2    ECDH

 ![27-ecdh](/images/https/27-ecdh.png)

## 8.3    SSH互信

认证的时候直接读取.ssh/authorized_keys进行匹配。

复制本机.ssh/id_rsa.pub  到对方  .ssh/authorized_keys

```shell
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC1sN62Y3NeUK9+8vfD+6BqecohPG1XG81Y4SDzGFDWhV7Nb9on2oTzj8Wg3ZHAiCSQUyVddd7P6zlKGvWaxgQodQqYWTNgvFXlvwjmdxevv3A42euRSxBNW7pORvhbWcKIvBxCTCrK/eOybpqXHadAZDKaGR8orHpcF+z72kgXrMEregS4Zjvef/kvbIQGu2dkRhcV3v1ZnvxQaZQwilLH2pHSgEIGOVGlcpjXLktzKFwORVHVVWy5KqKlBNK/n7Z9QRDDf+Zb3F6XpM0+SuSCgkCKWMlhfdu93kYT9KyI08XlY9UCj9SG96ZHt1n/khS9Tdc802raT8M1TKPzlbmB root@rhel
```

