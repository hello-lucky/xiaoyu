---
title: "2026蓝桥安全省赛"
description: 
slug: 
date: 
image: 
categories:
    - ctf
tags:
    - 
weight: 1
---

### 1.情报收集

#### map_tracer

![](https://img.xiaoyuwell.top/PicGo/20260529183533300.png)

访问首页，看到一个地图展示页面。页面底部的 `<script>` 标签引用了 `/app.js`。

![](https://img.xiaoyuwell.top/PicGo/20260529183658543.png)

![](https://img.xiaoyuwell.top/PicGo/20260529183716191.png)

下载 app.js.map，恢复源代码

![](https://img.xiaoyuwell.top/PicGo/20260529141323070.png)

发现三个关键信息：

\- 内部接口地址：`/api/trace/internal/list`

\- 签名盐值：`trace_dev_2026`

\- 签名算法：`MD5(路径 + 时间戳 + 盐值)`

![](https://img.xiaoyuwell.top/PicGo/20260529183935536.png)

测试接口行为

```
https://eci-2ze8tkq3ewbz717k6p4r.cloudeci1.ichunqiu.com:5000/api/trace/internal/list
```

![](https://img.xiaoyuwell.top/PicGo/20260529184020146.png)

构造合法请求

```
ts=$(date +%s)

sign=$(echo -n "/api/trace/internal/list${ts}trace_dev_2026" | md5sum | cut -d' ' -f1)

curl -sk "https://eci-...:5000/api/trace/internal/list?ts=${ts}&sign=${sign}"
```

![](https://img.xiaoyuwell.top/PicGo/20260529211200584.png)

base64解码

```
echo "ZmxhZ3tiZDYxOGJmOS0yNjMzLTQxOTAtYTljNy01ODY0NWQ1MTU1ZTh9" | base64 -d
```

![](https://img.xiaoyuwell.top/PicGo/20260529211247437.png)

### 2.密码破解

#### 2.1Double_sign

```
题目内容：
An approval system adds a digital signature to each allowed message. You have two different messages and their signatures. The system also asks you to submit a valid admin signature. It seems like a simple "check signature" task, but the real issue may be with the person who signs.
容器地址： nc 47.94.241.66 24647
┌──(root㉿kali)-[~]
└─# nc 47.94.241.66 25028
curve=secp256k1
pubkey_x=115748035156900906057048383724950485999004508295225809154867934087472956284658
pubkey_y=22045152941370687011016904436510325215459396959148228503113922275633614538645
msg1=user=guest&action=view
msg2=user=ops&action=approve
target=role=admin&action=read_flag
z1=35803318665405666032048798908400774075259419311739592886871963585749689690594
z2=39793284188129639924973014131124626058788493396861207842790576967009843273958
r=58206291912047493001270768302978838281743794200134829962960013685718525623357
s1=50299823274630071383103819756925571651617257809878759631948693535171140512049
s2=55520076861554262741677107652353499562906081920501068121345625361837220482936
```

连接服务器，查看数据

```
nc 47.94.221.20 32777
```

服务器返回：

```
curve=secp256k1
pubkey_x=115748035156900906057048383724950485999004508295225809154867934087472956284658
pubkey_y=22045152941370687011016904436510325215459396959148228503113922275633614538645
msg1=user=guest&action=view
msg2=user=ops&action=approve
target=role=admin&action=read_flag
z1=35803318665405666032048798908400774075259419311739592886871963585749689690594
z2=39793284188129639924973014131124626058788493396861207842790576967009843273958
r=58206291912047493001270768302978838281743794200134829962960013685718525623357
s1=50299823274630071383103819756925571651617257809878759631948693535171140512049
s2=55520076861554262741677107652353499562906081920501068121345625361837220482936
```

然后提示输入 `r` 和 `s`

识别漏洞

```
注意 `msg1` 和 `msg2` 的签名共享**完全相同的 `r` 值**。
在 ECDSA 中，`r = (k·G).x mod n`。相同的 `r` 意味着两条签名用了**相同的随机数 `k`**。这就是 **nonce 重用攻击**。
```

恢复随机数 k

```
ECDSA 签名公式：s = k⁻¹ · (z + r · d)  (mod n)
```

对 msg1 和 msg2 分别有：

```
s1 = k⁻¹(z1 + r·d)  (mod n)
s2 = k⁻¹(z2 + r·d)  (mod n)
```

两式相减，消去 `r·d`：

```
s1 - s2 = k⁻¹(z1 - z2)  (mod n)
k = (z1 - z2) · (s1 - s2)⁻¹  (mod n)
```

```
k = 80603388271681002506170327227935537999727656800110082452995699209478981
```

恢复私钥 d：

```
d = (s1·k - z1) · r⁻¹  (mod n)
d = 514631507721405306716547086874154376734225971237942892825540282682087242401
```

伪造 admin 签名

用私钥 `d` 为 `"role=admin&action=read_flag"` 生成合法签名：

```
1. 计算 `zt = SHA256("role=admin&action=read_flag")`
2. 生成一个新的随机数 `k'`
3. 计算 `R' = k'·G`（secp256k1 曲线上的标量乘法）
4. `r' = R'.x mod n`
5. `s' = k'⁻¹ · (zt + r'·d) mod n`
```

提交签名，获取 flag

```python
import socket

r2 = 71350514616271635994595008056558825849248310992606662620792588957828347483804
s2f = 4932523255167955078554250428720383984681186407166419481814761285670644397299

sock = socket.socket()
sock.connect(("47.94.221.20", 32777))
# 读取提示
sock.recv(4096)
# 发送 r
sock.sendall(f"{r2}\n".encode())
sock.recv(4096)
# 发送 s
sock.sendall(f"{s2f}\n".encode())
# 读取 flag
print(sock.recv(4096).decode())
```

exp：

```
import hashlib, socket, random

n = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141
p = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F
Gx, Gy = 0x79BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798, 0x483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A68554199C47D08FFB10D4B8

z1, z2 = 35803318665405666032048798908400774075259419311739592886871963585749689690594, 39793284188129639924973014131124626058788493396861207842790576967009843273958
r, s1, s2 = 58206291912047493001270768302978838281743794200134829962960013685718525623357, 50299823274630071383103819756925571651617257809878759631948693535171140512049, 55520076861554262741677107652353499562906081920501068121345625361837220482936

# 恢复 k 和私钥 d
k = ((z1 - z2) * pow(s1 - s2, -1, n)) % n
d = ((s1 * k - z1) * pow(r, -1, n)) % n

# 计算目标消息哈希
zt = int(hashlib.sha256(b"role=admin&action=read_flag").hexdigest(), 16)

# ECC 标量乘法
def ecc_mul(kk, P):
    def add(P1, P2):
        if P1 is None: return P2
        if P2 is None: return P1
        x1,y1=P1; x2,y2=P2
        if x1==x2 and y1==(-y2)%p: return None
        s = (3*x1*x1)*pow(2*y1,-1,p)%p if x1==x2 else (y2-y1)*pow(x2-x1,-1,p)%p
        x3=(s*s-x1-x2)%p; y3=(s*(x1-x3)-y1)%p
        return (x3,y3)
    rv=None; ap=P
    while kk:
        if kk&1: rv=add(rv,ap)
        ap=add(ap,ap); kk>>=1
    return rv

# 伪造签名
k2 = random.randint(1, n-1)
R = ecc_mul(k2, (Gx, Gy))
r2, s2f = R[0] % n, (pow(k2, -1, n) * (zt + R[0] % n * d)) % n

# 提交
sock = socket.socket(); sock.settimeout(10)
sock.connect(("47.94.221.20", 32777))
sock.recv(4096)
sock.sendall(f"{r2}\n".encode())
sock.recv(4096)
sock.sendall(f"{s2f}\n".encode())
print(sock.recv(4096).decode())
sock.close()
```



#### 2.2Seed_receipt

receipts.txt

```
time=1715071200
order_id=2024042701
check_code=936992
cipher=246a346dc207c5508810015b1a727ca2050e17490903ae3f91988ac56020c90e4e937d9240c03a2e723a
```

generator.py

```python
import random

def build_check_code(ts, order_id):
    random.seed(ts ^ order_id)
    return "".join(str(random.randint(0, 9)) for _ in range(6))

def obfuscate(secret, ts, order_id):
    random.seed(ts ^ order_id)
    _ = build_check_code(ts, order_id)
    mask = bytes(random.randint(0, 255) for _ in range(len(secret)))
    return bytes(ch ^ mask[i] for i, ch in enumerate(secret))

```

1. 识别漏洞：`random.seed(ts ^ order_id)` 在 `build_check_code` 和 `obfuscate` 中被重复调用，种子完全相同。Python 的 `random` 模块是确定性的，相同种子产出相同序列。

2. 恢复种子：`ts` 和 `order_id` 已知，直接计算出 `ts ^ order_id`。

3. 重放 RNG 状态：用相同种子播种，跳过前 6 次 `randint(0,9)` 调用（即生成 `build_check_code` 的那部分），此时 RNG 状态恰好与 `obfuscate` 中生成 XOR mask 时同步。

4. 生成 mask 并解密：用同步后的 RNG 生成 XOR mask，与 cipher 异或得到明文。

exp:

```python
import random

time = 1715071200
order_id = 2024042701
cipher_hex = "246a346dc207c5508810015b1a727ca2050e17490903ae3f91988ac56020c90e4e937d9240c03a2e723a"

random.seed(time ^ order_id)

# 跳过 build_check_code 的 6 次 randint 调用
for _ in range(6):
    random.randint(0, 9)

# 此时 RNG 状态与 obfuscate 内部一致
cipher = bytes.fromhex(cipher_hex)
mask = bytes(random.randint(0, 255) for _ in range(len(cipher)))
plain = bytes(c ^ m for c, m in zip(cipher, mask))
print(plain.decode())
```



#### 2.3faulty_stamp

output.txt(已知)

```
n=4376391623420422090093125321247193997606178746835986896757980039875406307457754575765912346918411063231436257346706147995337640299058377914239903698206529
e=65537
m=1976694502555205789395915483014500
s_good=1215462546937178480989928955032329371876376937587696844835636102409613045816885299683930703380803778980920223250158896812933428862730662189869680440962991
s_fault=3528092528175974455947733812329818046193185775378825376160301555808018696615021261487903570154559785404162102106766399756539357297019413781923298085052060
cipher=2893682879964766743522320790449865549540632243943763005715261841817782270701768093468232119779163352459853082410444548419721052039891196401139541267716111

```

task.py

```
from math import gcd
def crt_sign(message, dp, dq, p, q, inject_fault=False):
    sp = pow(message, dp, p)
    sq = pow(message, dq, q)
    if inject_fault:
        sq = (sq + 1) % q
    qinv = pow(q, -1, p)
    return sq + ((qinv * (sp - sq)) % p) * q

```

1. 理解 RSA-CRT 原理：签名拆分为模 `p` 和模 `q` 两部分：

  \- `s_p = m^dp mod p`

  \- `s_q = m^dq mod q`

  \- 通过 CRT 重组：`s = s_q + q_inv * (s_p - s_q) * q`

2. 定位故障影响：`inject_fault=True` 时，只篡改了 `sq`（加 1），`sp` 完全正确。所以：

  \- 模 `p` 部分正确：`s_fault ≡ s_good (mod p)`

  \- 模 `q` 部分错误：`s_fault ≠ s_good (mod q)`

3. GCD 分解：由于 `s_fault ≡ s_good (mod p)`，即 `p | (s_fault - s_good)`，而 `p` 也是 `n` 的因子：

```
 p = gcd(n, s_fault - s_good)

  q = n // p
```

4. 解密 cipher：有了 `p` 和 `q`，计算 `φ(n) = (p-1)(q-1)`，进而得到 `d = e⁻¹ mod φ(n)`，最后解密。

exp：

```python
from math import gcd

p = gcd(n, s_fault - s_good)
q = n // p
phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)

plain_int = pow(cipher, d, n)
plain_hex = hex(plain_int)[2:]
if len(plain_hex) % 2:
    plain_hex = '0' + plain_hex
print(bytes.fromhex(plain_hex).decode())
```

