---
title: Jwt
tion bypass via unverified signature

### 通过未经验证的签名绕过 JWT 身份验证

抓包发现cookie位置为jwt验证，将用户部分改为administrator即可进入管理员界面，删除calors即可

## JWT authentication bypass via flawed signature verification 

### 通过有缺陷的签名验证绕过 JWT 身份验证

将签名加密方式改为none,并修改用户名部分，删除签名后访问admin,注意保留最后一部分的分隔符.

## JWT authentication bypass via weak signing key 

### 通过弱签名密钥绕过 JWT 身份验证

通过hashcat能成功爆破出jwt key,使用该key对修改后的jwt进行签名即可

```
hashcat -a 0 -m 16500 jwt jwt.secrets.list
```

可得到密钥为secret1

## JWT authentication bypass via jwk header injection 

### 通过 JWK 标头注入绕过 JWT 身份验证

实验使用基于 JWT 的机制来处理会话。服务器支持 JWT 标头中的 `jwk` 参数。这有时用于将正确的验证密钥直接嵌入到令牌中。但是，它无法检查提供的密钥是否来自受信任的来源。

通过使用自己的 RSA 私钥对修改后的 JWT 进行签名，然后在 `jwk` 标头中嵌入匹配的公钥来利用此行为。

使用burp的jwt editor生成新的jwk，通过web token注入到参数中即可成功访问/admin

![jwk](E:\笔记软件\笔记\渗透测试\brup labs\jwk.png)

## Injecting self-signed JWTs via the jku parameter

###  通过 jku 参数注入自签名 JWT

https://www.anquanke.com/post/id/236830

jku是什么以及如何利用

labs本身提供了上传jku的地方，我们可以直接使用

大致流程

生成自身的RS密钥-->生成公钥上传至恶意服务器-->在原有jwt中添加jku以及更改kid值并使用密钥进行签名-->进行攻击

## JWT authentication bypass via kid header path traversal

### 通过 kid 标头路径遍历绕过 JWT 身份验证

将参数 `kid` 指向标准文件 `/dev/null` 。实际上，可以将 `kid` 参数指向具有可预测内容的任何文件

jwt通过验证kid参数来对token进行验证，通过将kid指向文件的操作，服务器使用指向文件来进行token的验证

```
"kid": "../../../../../../../../dev/null"
```

将生成的对称密钥中k参数改为AA==并对jwt进行签名，发现能过绕过token的验证

## JWT authentication bypass via algorithm confusion 

### 通过算法混淆绕过 JWT 身份验证

在解题之前我们需要知道什么是算法混淆

https://portswigger.net/web-security/jwt/algorithm-confusion

利用泄露的公钥，我门可以自行进行签名，首先将泄露的公钥导成RSA密钥，再以PEM的形式导出进行base64编码，再导入为对称key

通过将RS256改为HS256，再使用该对称密钥进行签名即可

## JWT authentication bypass via algorithm confusion with no exposed key 

### 通过算法混淆绕过 JWT 身份验证，没有公开密钥

通过使用现有的python脚本猜测后台pem密钥，通过请求目标来验证是否猜测正确，当拿到猜测正确的jwt之后，使用对应的base64编码后的public key对猜测的jwt进行签名，请求即可访问admin