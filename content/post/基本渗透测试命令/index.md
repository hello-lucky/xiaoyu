---
title: "基本渗透测试命令"
description: 
slug: 
date: 
image:
categories:
    - stu
tags:
    - 基本渗透测试命令
weight: 1
---



y



# 一、Linux渗透测试命令

### 查看本机正在监听的 TCP、UDP 端口

```
sudo ss -tulpn
```

用途是查看本机正在监听的 TCP、UDP 端口，以及对应的 PID 和程序名3
参数意义：
-t：显示 TCP。
-u：显示 UDP。
-l：只显示监听中的端口。
-p：显示占用端口的进程和 PID。
-n：直接显示数字 IP 和端口，不把 22 转成 ssh、80 转成 http。

### 实时查看TCP、UDP监听端口

```
sudo watch -n 1 'ss -tulpn'
```

这里：

- `watch -n 1`：每一秒刷新一次。
- `ss -tulpn`：显示 TCP、UDP、监听端口、进程和数字端口。

按 `Ctrl+C` 退出。

### `lsof` 

原意是“列出打开的文件”。在 Unix/Linux 中，网络套接字也被视为文件，所以 `-i` 会列出 Internet 网络套接字

常用写法：

```
# 所有网络套接字
sudo lsof -nP -i

# 只看已经建立的 TCP 连接
sudo lsof -nP -iTCP -sTCP:ESTABLISHED

# 查看谁在使用 8080 端口
sudo lsof -nP -i :8080

# 查看某个进程打开的网络连接
sudo lsof -nP -a -p 1234 -i
```

### 修改 MAC、IP 和 MTU

```
sudo macchanger -m 02:11:22:33:44:55 eth0
```

含义：

- `-m`：指定一个明确的 MAC 地址。
- `02:11:22:33:44:55`：新 MAC 地址。
- `eth0`：目标接口。

通常先关闭接口，再修改：

```
sudo ip link set dev eth0 down
sudo macchanger -m 02:11:22:33:44:55 eth0
sudo ip link set dev eth0 up
```

查看当前和永久 MAC：

```
macchanger -s eth0
```

恢复硬件原始 MAC：

```
sudo macchanger -p eth0
```

这通常改变的是系统当前使用的 MAC，而不是改写网卡芯片里的永久硬件地址。还应避免与同一局域网内其他设备使用相同 MAC。

### 给 `eth0` 配置 IPv4 地址

```
sudo ip addr add 192.168.2.1/24 dev eth0
sudo ip link set dev eth0 up
```

这里 `/24` 表示：

```
网络地址：192.168.2.0
常见主机范围：192.168.2.1 ～ 192.168.2.254
广播地址：192.168.2.255
子网掩码：255.255.255.0
```

`ip addr add` 是“增加地址”，不会自动删除该接口原来的其他地址。可以先检查：

```
ip addr show dev eth0
```
### 给 eth0 增加多个地址

```
sudo ip addr add 192.168.2.3/24 dev eth0
```

查看：

```
ip addr show dev eth0
```

删除：

```
sudo ip addr del 192.168.2.3/24 dev eth0
```

### Linux 允许一个接口同时拥有多个 IPv4 或 IPv6 地址

```
 ifconfig eth0 hw ether 02:11:22:33:44:55
```

查看修改结果：

```
ip link show dev eth0
```

### DNS 查询

```
dig -x 192.168.1.1
```

这是反向 DNS 查询：根据 IP 地址查询对应的主机名。

它实际查询的是 PTR 记录。例如 IPv4 地址会被转换到 `in-addr.arpa` 域下

```
host 192.168.1.1
```

输入是 IP 地址时，`host` 默认也会执行反向 DNS 查询。

它不是只有“没安装 dig 时才用”，而是另一种更简洁的 DNS 查询工具：

```
host example.com
host 8.8.8.8
host -t MX example.com
```

其中：

- `host example.com`：查询域名地址。
- `host 8.8.8.8`：反向查询主机名。
- `host -t MX example.com`：查询邮件服务器记录。

```
dig @192.168.2.2 domain.com AXFR
```

含义：

- `@192.168.2.2`：指定要询问的 DNS 服务器。
- `domain.com`：要传输的 DNS 区域。
- `AXFR`：请求完整区域传送。

```
host -l example.com ns1.example.com
```

它同样尝试列出 DNS 区域，本质上执行区域传送：

- `example.com`：区域名称。
- `ns1.example.com`：指定权威 DNS 服务器。
- `-l`：列出区域。

### 附加IP、终止连接和路由设置

```
sudo ip addr add 192.168.2.22/24 dev eth0
```

这是给 `eth0` 增加一个额外 IP 地址。

它是接口上的第二个或后续地址，运行下面的命令就能看到：

```
ip addr show dev eth0
```

也可以只显示 IPv4：

```
ip -4 addr show dev eth0
```

删除：

```
sudo ip addr del 192.168.2.22/24 dev eth0
```



```
sudo tcpkill -9 'host google.com'
```

`tcpkill` 是 `dsniff` 工具包中的程序，用来终止匹配过滤条件的活动 TCP 连接。

这里：

- `host google.com` 是类似 `tcpdump` 的过滤表达式。
- `-9` 是较高的强制程度，用来提高 TCP 重置包命中当前接收窗口的概率。

它会尝试用 TCP RST 终止匹配连接，但它不是持久防火墙：

- 只能处理 TCP 连接；
- 程序结束后不再继续干预；
- 运行中的应用可能重新建立连接；
- 不会处理匹配目标的 UDP 流量。

# 二、系统信息命令

### `whoami`：我当前以谁的身份运行

```
whoami
```

它显示的是当前进程的**有效用户名称**，等价于：

```
id -un
```

### `id`：查看 UID、GID 和所属组

```
id
```

可能得到：

```
uid=1000(alice) gid=1000(alice) groups=1000(alice),27(sudo)
```

其中：

- `uid=1000(alice)`：用户 ID 和用户名。
- `gid=1000(alice)`：主组 ID 和主组名称。
- `groups=...`：该用户所属的全部补充组。
- 出现 `sudo`、`wheel` 等组，通常表示该用户可能拥有提权权限，但是否能够使用 sudo 仍取决于具体策略。

常见用法：

```
# 查看指定用户
id alice

# 只显示当前有效 UID
id -u

# 显示当前用户名
id -un

# 显示全部组名
id -Gn
```

与 `whoami` 相比，`id` 提供的信息更加完整，还能查询指定用户

### `last`：查看历史登录记录

```
last
```

常用写法：

```
# 最近 10 条记录
last -n 10

# 某个用户的记录
last alice

# 系统重启记录
last reboot

# 关机、重启及运行级别等更多记录
last -x
```

`last` 通常从 `wtmp` 登录记录中读取信息，也可能显示 `reboot`、`shutdown` 等特殊记录。它是排查异常登录的辅助工具，但不能视为不可篡改的安全审计日志。

### 查看目录挂载

```
findmnt
```

或者查看某个目录所在的文件系统：

```
findmnt /
findmnt /home
```

### `df -h`：查看文件系统空间

```
df -h
```

`df` 查看的是**文件系统空间使用情况**，`-h` 表示使用更容易阅读的单位

常用扩展：

```
# 同时显示文件系统类型
df -hT

# 查看根文件系统
df -h /

# 查看 inode 使用情况
df -ih
```

### `echo "user:passwd" | chpasswd`：批量修改密码

主要用于批量更新已有用户的密码。例如：

```
sudo chpasswd <<'EOF'
alice:Password1
bob:Password2
charlie:Password3
EOF
```

只修改一个用户时，通常更适合使用交互式命令：

```
sudo passwd alice
```

### `getent passwd`：查询系统用户数据库

```
getent passwd
```

它会列出系统通过 Name Service Switch 能够查询到的用户条目

只查询一个用户：

```
getent passwd alice
```

输出可能是：

```
alice:x:1000:1000:Alice:/home/alice:/bin/bash
```

七个字段依次是：

```
用户名:密码字段:UID:GID:说明:家目录:登录Shell
```

### `strings /usr/local/bin/blah`：从二进制文件提取可打印文本

```
strings /usr/local/bin/blah
```

`strings` 会从文件中搜索连续的可打印字符。GNU `strings` 默认通常显示长度不少于 4 个字符的字符串。

它常用于快速寻找：

- 程序错误信息；
- 文件路径；
- 域名和 URL；
- 配置文件名称；
- 编译器和库信息；
- 命令行参数提示。

常用选项：

```
# 至少连续 8 个可打印字符
strings -n 8 file.bin

# 显示每个字符串在文件中的十六进制偏移
strings -t x file.bin

# 扫描整个文件
strings -a file.bin
```

### `uname -ar`：查看内核和系统信息

```
uname -ar
```

这里：

- `-a`：显示所有可用系统信息。
- `-r`：只表示显示内核发行号。

### `history`：查看 Bash 命令历史

```
history 20 #查看最近 20 条
```

Bash 的 `history` 命令不仅能读取记录，还能删除、清空、读写历史文件。例如：

```
# 删除某一项
history -d 1003

# 清空当前历史列表
history -c

# 将当前历史写入历史文件
history -w

# 把本次会话的新记录追加到历史文件
history -a
```

# 三、版本系列

### 用来显示基础系统的版本信息

```
cat /etc/os-release
```

典型字段包括：

```
PRETTY_NAME="Kali GNU/Linux Rolling"
NAME="Kali GNU/Linux"
VERSION_ID="2026.2"
VERSION="2026.2"
VERSION_CODENAME=kali-rolling
ID=kali
ID_LIKE=debian
HOME_URL="https://www.kali.org/"
SUPPORT_URL="https://forums.kali.org/"
BUG_REPORT_URL="https://bugs.kali.org/"
ANSI_COLOR="1;31"
```

### 列出 `dpkg` 软件包数据库里的记录

```
dpkg -l
```

它列出 `dpkg` 软件包数据库里的记录，包括包名、版本、架构、说明和状态。

输出示意：

```
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/...
||/ Name       Version       Architecture  Description
+++-==========-=============-=============-=================
ii  curl       8.x.x-...     amd64         command line tool
rc  oldpkg     1.0-...       amd64         old package
```

最左边的状态非常重要：

| 状态  | 含义                               |
| ----- | ---------------------------------- |
| `ii`  | 希望安装，并且已经正确安装         |
| `rc`  | 软件包主体已删除，但配置文件仍保留 |
| `iU`  | 已解包，但尚未配置完成             |
| `iF`  | 配置失败或处于半配置状态           |
| `iHR` | 可能需要重新安装，属于异常状态     |

常见的 `dpkg` 查询命令

查看某个包的详细状态

```
dpkg -s openvpn
```

可以看到版本、架构、依赖关系、安装状态和描述。

查看某个包安装了哪些文件

```
dpkg -L openvpn
```

例如可能列出：

```
/usr/sbin/openvpn
/usr/share/doc/openvpn
/etc/openvpn
```

查询某个文件属于哪个软件包

```
dpkg -S /usr/bin/curl
```

可能得到：

```
curl: /usr/bin/curl
```

### 查询 RPM 数据库中的所有已安装软件包

```
rpm -qa
```

参数含义：

- `-q`：query，查询。
- `-a`：all，查询全部已安装软件包。

常见 RPM 查询命令

查看详细信息：

```
rpm -qi openvpn
```

查看该包安装的文件：

```
rpm -ql openvpn
```

查询某个文件属于哪个 RPM 包：

```
rpm -qf /usr/sbin/openvpn
```

使用自定义格式输出包名、版本和架构：

```
rpm -qa --qf '%{NAME}\t%{VERSION}-%{RELEASE}\t%{ARCH}\n'
```

### 日常检查时，可以直接记住这一组：

```
# 所有发行版：查看系统版本
cat /etc/os-release

# Debian / Ubuntu：查看已安装包
dpkg -l | less
apt list --installed

# RPM 系：查看已安装包
rpm -qa | sort | less

# 查看单个包
dpkg -s openvpn       # Debian / Ubuntu
rpm -qi openvpn       # RPM 系
```

## 四、Linux用户管理

### 创建用户账户

```
sudo useradd -m -s /bin/bash new-user
sudo passwd new-user
```

参数含义：

- `-m`：创建家目录，例如 `/home/new-user`。
- `-s /bin/bash`：将登录 Shell 设置为 Bash。
- `passwd`：为新用户设置密码。

检查结果：

```
id new-user
getent passwd new-user
ls -ld /home/new-user
```

### 修改指定用户的密码

```
sudo passwd username
```

用于修改指定用户的密码。

普通用户执行：

```
passwd
```

查看密码状态可以使用：

```
sudo passwd -S username
```

### 高级用户删除工具

```
sudo deluser username
```

`deluser` 是 Debian/Ubuntu 系列常用的高级用户删除工具。默认情况下，它通常只删除用户账户，**不会删除家目录、邮件目录以及用户在其他位置创建的文件**。

同时删除家目录：

```
sudo deluser --remove-home username
```

在 RHEL、CentOS、Rocky Linux 等系统上通常使用：

```
sudo userdel username
```

连同家目录和邮件目录删除：

```
sudo userdel -r username
```

即使使用 `userdel -r`，用户在其他文件系统或其他路径中拥有的文件也不会自动全部删除，需要另外检查。

删除用户前建议先检查：

```
id username
pgrep -u username
find /home -maxdepth 2 -user username -ls
```

# 五、Linux解压缩命令

### unzip archive.zip

```
unzip archive.zip
```

把 ZIP 压缩包中的内容解压到当前目录。

先查看而不解压：

```
unzip -l archive.zip
```

解压到指定目录：

```
unzip archive.zip -d output-dir
```

测试 ZIP 是否完整：

```
unzip -t archive.zip
```

`unzip` 不带其他参数时，默认把压缩包中的全部成员解压到当前目录及其子目录。

对于来源不明的压缩包，先列出内容，再解压到新建的空目录：

```
mkdir extracted
unzip -l archive.zip
unzip archive.zip -d extracted
```

### `zipgrep` 的基本语法

```
zipgrep 搜索模式 压缩包 [包内文件名]
```

搜索 ZIP 内所有文件中的 `error`：

```
zipgrep 'error' archive.zip
```

只搜索 ZIP 内名称匹配 `*.txt` 的成员：

```
zipgrep 'error' archive.zip '*.txt'
```

这里必须把 `*.txt` 放在压缩包参数之后，并且最好加引号，防止当前 Shell 先把它展开成本地文件名。

### tar xf archive.tar

```
tar xf archive.tar
```

解开未压缩的 TAR 归档。

参数含义：

- `x`：extract，提取。
- `f`：后面的参数是归档文件名。

查看内容但不解包：

```
tar tf archive.tar
```

解压到指定目录：

```
tar xf archive.tar -C output-dir
```

### `ar xzf archive.tar.gz`

```
tar xzf archive.tar.gz
```

解开使用 gzip 压缩的 TAR 归档。

参数：

- `x`：解包。
- `z`：通过 gzip 解压。
- `f`：指定归档文件。

先查看：

```
tar tzf archive.tar.gz
```

解压到指定位置：

```
tar xzf archive.tar.gz -C output-dir
```

### `tar xjf archive.tar.bz2`

```
tar xjf archive.tar.bz2
```

解开使用 bzip2 压缩的 TAR 归档。

其中 `j` 表示 bzip2。

查看内容：

```
tar tjf archive.tar.bz2
```

这三个 TAR 命令的区别主要是：

| 文件格式   | 解包命令               |
| ---------- | ---------------------- |
| `.tar`     | `tar xf file.tar`      |
| `.tar.gz`  | `tar xzf file.tar.gz`  |
| `.tar.bz2` | `tar xjf file.tar.bz2` |

### tar -ztvf file.tar.gz | grep -F 'blah'

```
tar -ztvf file.tar.gz | grep -F 'blah'
```

作用是：

1. `tar -ztvf` 列出 `.tar.gz` 中的成员；
2. `grep` 筛选名称中包含 `blah` 的行。

参数含义：

- `t`：列出内容。
- `v`：显示详细信息。
- `z`：gzip。
- `f`：指定文件。

### `gzip -d archive.gz`

```
gzip -d archive.gz
```

解压单个 gzip 文件，等价于：

```
gunzip archive.gz
```

通常会生成：

```
archive
```

并删除原来的 `archive.gz`。

保留原压缩文件：

```
gzip -dk archive.gz
```

### `vim file.txt.gz`

```
vim file.txt.gz
```

标准 Vim 通常可以通过自带的 gzip 插件透明地读取压缩文件：

1. 打开时临时解压；
2. 在编辑器中显示普通文本；
3. 保存时重新压缩。

# 六、Linux压缩命令

### 把文件夹中文件递归压缩

```
zip -r file.zip /dir
```

或者在目录内创建只包含目录内容的 ZIP：

```
cd /dir
zip -r /tmp/file.zip .
```

`zip -r` 会递归处理目录结构，并将多个文件放入同一个 ZIP 归档

### 把 `files` 打包成 `archive.tar`

`tar cf` 只是归档，不进行压缩

```
tar cf archive.tar files
```

例如：

```
tar cf backup.tar document.txt pictures/
```

`c`：create，创建归档。

`f`：指定归档文件名。

查看：

```
tar tf backup.tar
```

### 先用 TAR 归档，再通过 gzip 压缩

```
tar czf project.tar.gz project/
```

详细显示处理的文件：

```
tar czvf project.tar.gz project/
```

其中 `v` 表示 verbose。

### 先归档，再使用 bzip2 压缩

```
tar cjf project.tar.bz2 project/
```

### 单个文件压缩

```
gzip file
```

把单个文件压缩为：

```
file.gz
```

默认情况下原文件会被替换。保留原文件：

```
gzip -k file
```

gzip 本身不能把整个目录变成一个单独的 `.gz` 归档。压缩目录通常使用：

```
tar czf directory.tar.gz directory/
```

# 七、Linux文件命令

### 逐行比较两个文本文件

```
diff file1 file2
```

逐行比较两个文本文件，用于显示文件或目录之间的差异，其输出也常用于生成补丁。

如果没有任何输出，表示内容相同。更容易阅读的统一格式：

```
diff -u file1 file2
```

输出示意：

```
-old line
+new line
```

递归比较两个目录：

```
diff -ru directory1 directory2
```

### 计算文件的 MD5 摘要

```
sha256sum file
```

计算文件的 MD5 摘要，例如：

```
d41d8cd98f00b204e9800998ecf8427e  file
```

它可辅助发现下载或复制过程中发生的**意外损坏**，但不适合安全敏感的真实性验证

```
sha256sum -c blah.iso.md5
```

读取校验文件，并检查其中列出的文件。

校验文件通常类似：

```
1234567890abcdef...  blah.iso
```

生成和验证：

```
sha256sum blah.iso > blah.iso.md5
sha256sum -c blah.iso.md5
```

### 尝试识别文件类型，而不只依赖扩展名

```
file blah
```

它会依次使用文件系统信息、魔数数据库和部分语言测试来分类文件。

### 把输入内容进行 Base64 编码，并写入输出文件

```
base64 input-file > output-file
```

Base64 用可打印 ASCII 字符表示二进制数据

### 解码 Base64 数据

```
base64 -d < input-file > output-file
```

`-d` 或 `--decode` 将已编码的数据恢复为原始字节

### rm -rf

```
rm -rf directory
```

参数：

- `-r`：递归删除目录及其内容。
- `-f`：强制执行，不询问，忽略不存在的目标。

它非常危险，尤其是在 root 权限、变量为空或路径写错时。

# 八、Misc命令

### 重启

```
sudo systemctl reboot
```

或者：

```
sudo reboot
```

`systemctl reboot` 会通过系统服务管理器安排重启，并向其他用户发送通知。

### 从 Linux 图形环境连接远程桌面协议服务器

```
rdesktop [选项] server[:port]
rdesktop 192.168.1.20
```

它是一个 RDP 客户端，而不是 SSH 客户端。只能连接自己拥有或明确获准使用的服务器

### 立即杀死当前 Shell

```
kill -9 $$
```

在普通交互式 Bash 中，`$$` 通常表示当前 Shell 的进程 ID，`-9` 表示发送 `SIGKILL`。

结果是立即杀死当前 Shell：

- 不执行正常退出处理；
- 不运行信号处理程序；
- 不保证保存未写入的数据；
- 可能留下子进程；
- SSH 会话可能突然断开。

它不是一种良好的“关闭当前会话”方式。

正常退出使用：

```
exit
```

# 九、Linux文件系统权限

执行：

```
ls -l file
```

可能看到：

```
-rw-r----- 1 alice developers 1234 Aug  9 10:00 file
```

权限部分可以拆成：

```
-  rw-  r--  ---
│   │    │    │
│   │    │    └─ 其他用户 other
│   │    └────── 所属组 group
│   └─────────── 所有者 user
└─────────────── 文件类型
```

第一个字符常见含义：

- `-`：普通文件；
- `d`：目录；
- `l`：符号链接；
- `c`、`b`：字符设备、块设备。

| 字符 | 数值 | 对普通文件         | 对目录                           |
| ---- | ---- | ------------------ | -------------------------------- |
| `r`  | 4    | 读取文件内容       | 列出目录中的名称                 |
| `w`  | 2    | 修改、覆盖文件内容 | 创建、删除、重命名目录项         |
| `x`  | 1    | 把文件作为程序执行 | 进入、穿越目录并访问其中已知名称 |
| `-`  | 0    | 没有对应权限       | 没有对应权限                     |

| 权限  | 字符形式    | 对普通文件的典型含义               | 对目录的典型含义                         |
| ----- | ----------- | ---------------------------------- | ---------------------------------------- |
| `777` | `rwxrwxrwx` | 所有人都能读、改和执行             | 所有人都能浏览、进入、创建和删除内容     |
| `755` | `rwxr-xr-x` | 所有者可修改执行，其他人可读和执行 | 所有者可管理，其他人可浏览和进入         |
| `700` | `rwx------` | 只有所有者能读、改和执行           | 只有所有者能浏览和进入                   |
| `666` | `rw-rw-rw-` | 所有人可读写，不可执行             | 因为没有 `x`，通常无法正常进入和访问内容 |
| `644` | `rw-r--r--` | 所有者可读写，其他人只读           | 不适合作为普通目录权限，因为没有 `x`     |
| `600` | `rw-------` | 只有所有者可读写                   | 不适合作为目录权限，私有目录通常用 `700` |

```
chmod 777 dir/file
```

# 十、Linux根目录结构

| 目录          | 更准确的作用                                                 |
| ------------- | ------------------------------------------------------------ |
| `/`           | 根目录，整个文件系统命名空间的起点                           |
| `/bin`        | 基本用户命令；现代系统上可能链接到 `/usr/bin`                |
| `/boot`       | 引导加载器、内核、initramfs 等启动文件                       |
| `/dev`        | 设备节点和伪设备，如磁盘、终端、`/dev/null`                  |
| `/etc`        | 当前主机的系统级配置文件                                     |
| `/home`       | 普通用户家目录的常见位置                                     |
| `/lib`        | 基本程序所需共享库及部分内核模块；可能链接到 `/usr/lib`      |
| `/lost+found` | ext2/ext3/ext4 文件系统检查后放置恢复出的孤立文件            |
| `/mnt`        | 管理员临时或手动挂载文件系统的常用位置                       |
| `/media`      | 可移动介质的挂载点，如 U 盘、光盘                            |
| `/net`        | 某些 autofs 配置用作远程文件系统入口，并非所有 Linux 的标准目录 |
| `/opt`        | 附加或第三方软件包                                           |
| `/proc`       | 虚拟文件系统，提供进程和内核运行信息                         |
| `/root`       | root 用户的家目录                                            |
| `/sbin`       | 基本系统管理命令；并不表示普通用户绝对不能执行               |
| `/tmp`        | 临时文件；可能在启动时或定期清理，但不能假定必然重启即清空   |
| `/usr`        | 发行版提供的程序、库、头文件和共享数据，不是用户家目录       |
| `/var`        | 经常变化的数据，如日志、缓存、队列、数据库和服务状态         |

# 十一、常见系统文件和目录

|   账户数据库   | /etc/passwd             | 保存本地账户的基本资料，不保存现代系统中的实际密码散列。     |
| :------------: | ----------------------- | ------------------------------------------------------------ |
|                | /etc/shadow             | 保存本地账户的：密码散列或锁定标记； 上次修改时间； 密码有效期； 过期和警告策略。 |
|                | /etc/group              | 保存本地组信息                                               |
| 启动和主机配置 | /etc/init.d/            | 传统 SysV init 服务脚本目录                                  |
|  hostnamectl   | /etc/hostname           | 保存系统的静态主机名                                         |
|                | /etc/profile            | 系统范围的 Shell 启动文件，主要供兼容的**登录 Shell**读取    |
|    网络配置    | /etc/network/interfaces | Debian `ifupdown` 体系的网络接口配置文件                     |
|                | /etc/resolv.conf        | 配置的是本机名称解析器                                       |
|   Bash 历史    | ~/.bash_history         | Bash 默认常用的持久历史文件                                  |
|    日志目录    | /var/log/               | 传统 Linux 日志目录                                          |
|                | /var/adm/               | 历史上的 UNIX 管理和日志目录                                 |
|   /etc/fstab   |                         | 静态文件系统挂载配置                                         |

SSH 文件

| 文件                           | 含义                       |
| ------------------------------ | -------------------------- |
| `id_ed25519`、`id_rsa`         | 私钥，必须保密             |
| `id_ed25519.pub`、`id_rsa.pub` | 公钥，一般不需要保密       |
| `authorized_keys`              | 服务器允许登录的公钥列表   |
| `known_hosts`                  | 已连接服务器的主机公钥记录 |
| `config`                       | 当前用户的 SSH 客户端配置  |
