## 流量分析

psexec基于smb协议，利用远程主机的445端口，因此在该工具启动的时候，直接进行了TCP三次握手，尝试连接远程主机的445端口

，SMB协商使用SMBv2协议通信，进行NTLM认证

![tcp-smb-ntlm](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/another_Intranet_horizontal/tcp-smb-ntlm.png)

NTLM认证过程中能够看到请求的信息，域以及用户名

![ntlm-name](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/Psexec_stream/ntlm-name.png)

接下来psexec请求想要访问的目标资源，即ADMIN$共享

<img src="https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/Psexec_stream/request-Admin%24.png" alt="request-Admin$" style="zoom:33%;" />

创建文件请求，开始通过TCP流写入文件

![file-write](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/Psexec_stream/file-write.png)

一直到close response,写入文件的任务完成

![file-write-finish](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/Psexec/file-write-finish.png)

PSexec从资源文件中提取出了一个服务，并创建运行，在OpenSCManagerW中，攻击机开始调用svcctl协议，并打开psexecsvc服务

![svcttl](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/Psexec/svcttl.png)

随后创建四个管道

![pipe1](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/Psexec/pipe1.png)

![pipe2-3-4](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/Psexec/pipe2-3-4.png)

后面三个管道分别对应了输入，输出和错误，管道创建成功就能通过psexec连接远程主机进行使用，在关闭会话时会删除服务和进程

## 进程服务与日志

在psexec启动时通过PsExec进程来创建服务

![service-and-exe](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/Psexec_stream/service-and-exe.png)

Psexec在退出后会产生Event 4624,4628,4634的登录日志，系统日志中会7045记录了PSEXESVC的安装，7036记录PSEXESVE的服务状态

![7045](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/Psexec/7045.png)

很明显，这样的日志特征非常容易被检测，这也是为什么后续会转向WMI的使用

## 基础的免杀

最简单也是比较直接的

![pipe_name](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/Psexec/pipe_name.png)

修改管道名称，或者尝试修改输出

![printf](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/Psexec-stream/printf.png)

修改后重新编译生成exe文件，转为hex

```python
import binascii
filename = 'RemComSvc.exe'
with open(filename, 'rb') as f:
    content = f.read()
print(binascii.hexlify(content))
```

拿去替换掉项目中原来的二进制文件，或是指定-f选择自己的文件

![recomsvc](https://raw.githubusercontent.com/uu2fu3o/blog-picture/main/Psexec-stream/recomsvc.png)
