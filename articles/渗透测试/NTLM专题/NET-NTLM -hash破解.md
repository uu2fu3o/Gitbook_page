## Net-NTLM v1破解

只要获取到Net-NTLM v1，都能破解为NTLM hash。

**操作**

+ 修改`Responder.conf`里面的Challenge为`1122334455667788`

![chanllege](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/chanllege.png)

+ 在responder种指定--lm参数(针对SMB协议)，其他协议需要自行修改type2，比如Http协议的话，需要修改packets.py里面的NTLM_Challenge类。原来是NegoFlags的值是`\x05\x02\x89\xa2`，改成`\x05\x02\x81\xa2`

+ 监听ntlm-v1hash

  ![nossp](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/nossp.png)

  可以发现得到的hash并没有ssp字样

+ 使用[ntlmv1-multi](https://github.com/evilmog/ntlmv1-multi)里面的ntlmv1.py转换(因为获取到的是没有ssp字样的hash)

  ![ntlmv1crack](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/ntlmv1crack.png)

  最下方给出了可以直接使用crack.sh直接破解的hash

+ 将转换好的hash放到crack.sh进行破解，不过网站好像一直都在维护

**注意**：

```shell
测试时可以通过更改注册表将NET-NTLMv2降到NET-NTLMv1开启
reg add HKLM\SYSTEM\CurrentControlSet\Control\Lsa\ /v lmcompatibilitylevel /t REG_DWORD /d 2 /f
reg add HKLM\SYSTEM\CurrentControlSet\Control\Lsa\MSV1_0\ /v NtlmMinClientSec /t REG_DWORD /d 536870912 /f
reg add HKLM\SYSTEM\CurrentControlSet\Control\Lsa\MSV1_0\ /v RestrictSendingNTLMTraffic /t REG_DWORD /d 0 /f
```

**具有ssp保护措施的ntlmv1-hash**

捕获到之后可以直接拿到hashcat进行破解

![withssp](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/pac/withssp.png)

```
hashcat -m 5500 PC$::HACK:C74588CDBA5A2DF500000000000000000000000000000000:1440E394CFEDD5139CE9E3E4900BAB542F084C7D17281F01:1122334455667788  pwd.txt
# 将监听到的hash拿去hashcat破解，pwd.txt为密码字典
```

ntlmv1已经很少使用了，如果不自己降级为ntlmv1，使用的都是v2

## Net-NTLM v2破解

暂时还没有什么破解的好方法，一般只能抓取到然后用hashcat明文离线爆破，看字典里有没有，跟我们出路ntlmv1-ssp的方式相同

