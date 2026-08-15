---
title: 自建 CA 证书并用其签名服务器证书
date: 2026-08-15 21:34:18
tags:
  - CA
  - TLS
  - SSL
categories:
  - Development
  - Generic
---

# 自建 CA 证书

## 0. 什么是 CA 证书？

**CA（Certificate Authority，证书颁发机构）** 是一个"可信第三方"。
它用自己的**私钥**给别人的证书"盖章"（签名），从而证明"这张证书确实是某某机构发的"。

- **根 CA 证书（Root CA）**：自签名的、最顶层的证书，是信任链的起点。
  它自己给自己签名，所以也叫"自签名证书"。
- **服务器证书（Server Certificate）**：由 CA 签发的、真正用于 HTTPS 的证书。
  浏览器验证流程：`服务器证书 ← 由根 CA 签名`，而根 CA 在客户端的信任库里，于是信任成立。

**密钥对（Key Pair）** 的概念：

| 对象                | 作用           | 是否保密               |
| ------------------- | -------------- | ---------------------- |
| 私钥（`.key`）      | 签名、解密     | 必须严格保密，永不外传 |
| 公钥/证书（`.crt`） | 验证签名、加密 | 公开分发               |

> 私钥一旦泄露，等于你的 CA 被"盗号"，任何人可以伪造合法证书。

## 1. 依赖

```bash
# 检查 openssl 是否安装
openssl version
```

## 2. 生成根 CA 私钥

```bash
# -out    输出文件名
# 4096    密钥长度（位），越长越安全，2048 也够用
# genrsa  生成一对 RSA 密钥
openssl genrsa -out KeunlasRootCA.key 4096

# 设置私钥权限：仅属主可读写（安全起见）
chmod 400 KeunlasRootCA.key
```

## 3. 编写 CA 配置文件

```cnf
# KeunlasRootCA.cnf
# OpenSSL 配置文件：定义 CA 证书的身份信息和扩展属性

[ req ]
distinguished_name = req_dn    # 使用下面 [req_dn] 段的字段
prompt = no                    # 不交互式提问，直接读配置

[ req_dn ]
C  = CN                         # 国家 Country
ST = Shanghai                   # 省/州 State
L  = Shanghai                   # 城市 Locality
O  = KeunlasCA                  # 组织 Organization
CN = KeunlasRootCA              # 通用名 Common Name（必填）

[ v3_ca ]
# 以下三项是"它是一张 CA 证书"的关键标志：
basicConstraints       = critical, CA:TRUE      # 允许它签发其他证书
keyUsage               = critical, keyCertSign, cRLSign  # 用途：签证书、签吊销列表
subjectKeyIdentifier   = hash                   # 给证书算一个唯一 ID
authorityKeyIdentifier = keyid:always, issuer   # 记录"谁签发的我"
```

## 4. 生成根 CA 自签名证书

```bash
# -new              生成新证书请求
# -x509             直接输出自签名证书（而不是 .csr）
# -days 10950       有效期 10950 天 ≈ 30 年
# -nodes            私钥不加密（与已有私钥配套，无需重复设密码）
# -key              使用已有的私钥
# -config           使用我们的配置文件
# -extensions v3_ca 附加 [v3_ca] 中的扩展（CA:TRUE 等）

openssl req -new -x509 -days 10950 -nodes \
  -key KeunlasRootCA.key \
  -config KeunlasRootCA.cnf \
  -extensions v3_ca \
  -out KeunlasRootCA.crt
```

## 5. 校验根 CA 证书

```bash
# 查看证书内容
openssl x509 -in KeunlasRootCA.crt -noout -text

# 重点确认这两处：
#   1) Basic Constraints: CA:TRUE
#   2) Key Usage: Certificate Sign, CRL Sign
openssl x509 -in KeunlasRootCA.crt -noout -text | grep -E "CA:TRUE|Certificate Sign|CRL Sign"
```

# 签发服务器证书

## 1. 生成服务器私钥和 CSR

```bash
# 服务器私钥
openssl genrsa -out localhost.key 2048

# CSR（证书签名请求）：相当于"申请表"
openssl req -new \
  -key localhost.key \
  -out localhost.csr \
  -subj "/C=CN/ST=Shanghai/L=Shanghai/O=LocalhostServer/CN=localhost"
```

CSR 里包含服务器的公钥和身份信息（CN=localhost），
CA 拿到 CSR 后核验、签名，就能"盖章"发证。
CN=localhost 表示这张证书是给 localhost 域名用的。


## 2. 使用 CA 签发服务器证书


```bash
# -CA / -CAkey    用哪个 CA 及其私钥来签名
# -CAcreateserial 自动创建序列号文件 KeunlasRootCA.srl（后续签发号递增，保证每张证书序列号唯一）
# -days 825       有效期约 2.26 年（生产证书一般不超过 825 天）
# -extfile        附加扩展文件，这里写入 SAN（主题备用名）

openssl x509 -req \
  -in localhost.csr \
  -CA KeunlasRootCA.crt \
  -CAkey KeunlasRootCA.key \
  -CAcreateserial \
  -days 825 \
  -extfile <(printf "subjectAltName=DNS:localhost,DNS:*.localhost,IP:127.0.0.1") \
  -out localhost.crt
```

为什么必须加 SAN？
现代浏览器（Chrome 58+、Firefox）只认 SAN，不再认 CN。
没有 SAN 的证书，浏览器会报 ERR_CERT_COMMON_NAME_INVALID。
SAN 必须包含你要访问的实际域名/IP，例如 DNS:localhost、IP:127.0.0.1。

## 3. 校验服务器证书

```bash
# 确认签发者、SAN 是否正确
openssl x509 -in localhost.crt -noout -text | grep -E "Issuer:|Subject:|DNS:|IP Address:"

# 验证证书链是否有效（CA 与服务器证书配对）
openssl verify -CAfile KeunlasRootCA.crt localhost.crt
# 期望输出：localhost.crt: OK
```


# Diffie-Hellman（DH）密钥交换参数

过去常用于 TLS 握手时的前向保密（Perfect Forward Secrecy, PFS）

```bash
openssl dhparam -out dh4096.pem 4096
```

曾用于 TLS 1.2 时代的 DHE 密码套件，实现前向保密。TLS 1.3 / ECDHE 已取代它，一般不再需要。



