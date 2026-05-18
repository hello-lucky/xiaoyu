---
title: "2024蓝桥安全省赛"
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

### 一、安全知识

1.在PHP中，以下选项中哪一个运算符含义为x绝对等于y。

A. x = y

B. x == y

C. x === y

D. x !== y

 

2.文件名绝对符合允许规则” 的严格校验逻辑，其核心思想最接近以下哪种安全策略？

A. 无限制

B. 有规则的文件名

C. 白名单

D. 黑名单

 

3.读取外部传入XML文件时，XML解析器初始化过程中应该设置（）

A. 关闭DTD解析

B. 关闭XML解析

C. 打开DTD解析

D. 关键字过滤

在读取外部传入的XML文件时，默认启用的DTD（文档类型定义）解析功能，是导致XXE（XML）外部实体注入）漏洞的主要原因

 

4.代码安全审计规范中规定审计过程包括四个阶段：审计准备、审计实施、审计报告、（）。

A. 改进跟踪

B. 前瞻审计

C. 形成报告

D. 档案归档

 

5.代码安全审计准备阶段，应该了解项目的应用场景、目标客户、开发内容、（）。

A. 代码

B. 开发者遵循的标准和流程

C. 开发者编写的测试代码

D. 开发者所用的开发工具

 

6.对于可重放的敏感操作请求，需部署（）防御机制

A. SSRF

B. CSRF

C. SSTI

D. RCE

 

7.开发人员编写前端代码时，应该避免使用（），转为使用（），以此避免注入攻击。

A. eval、function

B. escape、encodeURIComponent

C. eval、JSON.parse

D. eval、console.log

 

8.在web开发过程中，与用户相关的重要操作建议采用下列哪项措施。

A. 密码要小于6位方便记忆

B. 验证码使用后不要失效

C. 接收短信验证码

D. 同一密码多次使用

 

9.以下远程运维中，不存在风险行为的是 （）。

A. 将可能存在安全风险的终端接入内部网络进行运维

B. 使用TLS协议加密传输通道

C. 仅记录运维操作日志中的操作人及操作时间

D. 使用DES算法加密敏感数据

TLS（传输层安全协议）是远程运维的标准安全配置，可对传输数据进行加密和身份认证，有效防止数据被窃听、篡改或中间人攻击。

 

10.针对防止敏感文件泄漏以下做法错误的是 （）。

A. 删除网站.git相关文件

B. 应用系统操作手册外传gitee

C. 禁止将系统开发源码上传github

D. 删除网站.svn文件



### 二、情报收集

小蓝同学在开发网站时了解到了一个***\*爬虫协议\****，该协议指网站可建立一个特别的txt文件来告诉搜索引擎哪些页面可以抓取，哪些页面不能抓取，而搜索引擎则通过读取该txt文件来识别这个页面是否允许被抓取。爬虫协议并不是一个规范，而只是约定俗成的，所以并不能保证网站的隐私。

![](https://xiaoyuwell.top/PicGo/20260518202838749.png)

尝试访问robots

![](https://xiaoyuwell.top/PicGo/20260518202928735.png)

![](https://xiaoyuwell.top/PicGo/20260518202947032.png)

![](https://xiaoyuwell.top/PicGo/20260518203006011.png)



### 三、数据分析

#### 1.数据包分析

打开数据包，过滤http，搜索flag，发现一个请求cat flag

![](https://xiaoyuwell.top/PicGo/20260518203100711.png)

数据流追踪，找到返回内容是一个base64编码

![](https://xiaoyuwell.top/PicGo/20260518203125447.png)

解码

![](https://xiaoyuwell.top/PicGo/20260518203141935.png)

#### 2.缺失的数据（dwt盲水印）

![](https://xiaoyuwell.top/PicGo/20260518203215458.png)

1)：破解orign压缩包：打开压缩包发现压缩包里面的serect.txt是密码字典，未加密，可以解压出来，然后用serect.txt字典破解压缩包orign.zip，得到a.png也就是原始图片。

2)：打开lose.py分析发现为DWT盲水印代码：

```python
class WaterMarkDWT:
    def __init__(self, origin: str, watermark: str, key: int, weight: list):
        self.key = key
        self.img = cv2.imread(origin)
        self.mark = cv2.imread(watermark)
        self.coef = weight
 
 
    def arnold(self, img):
        r, c = img.shape
        p = np.zeros((r, c), np.uint8)
 
        a, b = 1, 1
        for k in range(self.key):
            for i in range(r):
                for j in range(c):  
                    x = (i + b * j) % r
                    y = (a * i + (a * b + 1) * j) % c
                    p[x, y] = img[i, j]
        return p
 
    def deArnold(self, img):
        r, c = img.shape
        p = np.zeros((r, c), np.uint8)
 
        a, b = 1, 1
        for k in range(self.key):
            for i in range(r):
                for j in range(c): 
                        x = ((a * b + 1) * i - b * j) % r
                        y = (-a * i + j) % c
                    p[x, y] = img[i, j]
        return p
 
 
 
    def get(self, size: tuple = (1200, 1200), flag: int = None):
        img = cv2.resize(self.img, size)
 
        img1 = cv2.cvtColor(img, cv2.COLOR_RGB2GRAY)
        img2 = cv2.cvtColor(self.mark, cv2.COLOR_RGB2GRAY)
 
        c = pywt.wavedec2(img2, 'db2', level=3)
        [cl, (cH3, cV3, cD3), (cH2, cV2, cD2), (cH1, cV1, cD1)] = c
 
        d = pywt.wavedec2(img1, 'db2', level=3)
        [dl, (dH3, dV3, dD3), (dH2, dV2, dD2), (dH1, dV1, dD1)] = d
 
        a1, a2, a3, a4 = self.coef
 
        ca1 = (cl - dl) * a1
        ch1 = (cH3 - dH3) * a2
        cv1 = (cV3 - dV3) * a3
        cd1 = (cD3 - dD3) * a4
 
        waterImg = pywt.waverec2([ca1, (ch1, cv1, cd1)], 'db2')
        waterImg = np.array(waterImg, np.uint8)
 
        waterImg = self.deArnold(waterImg)
 
        kernel = np.ones((3, 3), np.uint8)
        if flag == 0:
            waterImg = cv2.erode(waterImg, kernel)
        elif flag == 1:
            waterImg = cv2.dilate(waterImg, kernel)
 
        cv2.imwrite('水印.png', waterImg)
        return waterImg
 
 
if __name__ == '__main__':
    img = 'a.png'
    k = 20
    xs = [0.2, 0.2, 0.5, 0.4]
W1 = WaterMarkDWT(img, waterImg, k, xs)
```

3) ：编写exp.py得到flag.png：

```python
import cv2
import pywt
import numpy as np
class WaterMarkDWT:
    def __init__(self, origin: str, watermark: str, key: int, weight: list):
        self.key = key
        self.img = cv2.imread(origin)
        self.mark = cv2.imread(watermark)
        self.coef = weight
 
 
    def arnold(self, img):
        r, c = img.shape
        p = np.zeros((r, c), np.uint8)
 
        a, b = 1, 1
        for k in range(self.key):
            for i in range(r):
                for j in range(c):  
                    x = (i + b * j) % r
                    y = (a * i + (a * b + 1) * j) % c
                    p[x, y] = img[i, j]
        return p
 
    def deArnold(self, img):
        r, c = img.shape
        p = np.zeros((r, c), np.uint8)
 
        a, b = 1, 1
        for k in range(self.key):
            for i in range(r):
                for j in range(c): 
                        x = ((a * b + 1) * i - b * j) % r
                        y = (-a * i + j) % c
                        p[x, y] = img[i, j]
        return p
 
 
 
    def get(self, size: tuple = (1200, 1200), flag: int = None):
        img = cv2.resize(self.img, size)
 
        img1 = cv2.cvtColor(img, cv2.COLOR_RGB2GRAY)
        img2 = cv2.cvtColor(self.mark, cv2.COLOR_RGB2GRAY)
 
        c = pywt.wavedec2(img2, 'db2', level=3)
        [cl, (cH3, cV3, cD3), (cH2, cV2, cD2), (cH1, cV1, cD1)] = c
 
        d = pywt.wavedec2(img1, 'db2', level=3)
        [dl, (dH3, dV3, dD3), (dH2, dV2, dD2), (dH1, dV1, dD1)] = d
 
        a1, a2, a3, a4 = self.coef
 
        ca1 = (cl - dl) * a1
        ch1 = (cH3 - dH3) * a2
        cv1 = (cV3 - dV3) * a3
        cd1 = (cD3 - dD3) * a4
 
        waterImg = pywt.waverec2([ca1, (ch1, cv1, cd1)], 'db2')
        waterImg = np.array(waterImg, np.uint8)
 
        waterImg = self.deArnold(waterImg)
 
        kernel = np.ones((3, 3), np.uint8)
        if flag == 0:
            waterImg = cv2.erode(waterImg, kernel)
        elif flag == 1:
            waterImg = cv2.dilate(waterImg, kernel)
 
        cv2.imwrite('水印.png', waterImg)
        return waterImg
 
 
if __name__ == '__main__':
    img = 'a.png'
    newImg='newImg.png'
    k = 20
    #xs = [0.2, 0.2, 0.5, 0.4]
    coef=[5,5,2,2.5]
    #waterImg='flag.png'
    W1 = WaterMarkDWT(img, newImg, k, coef)
    waterimg=W1.get()
```



### 四、密码破解

#### 1.cc

CyberChef是一个用于加密、编码、压缩和数据分析的网络应用程序，被称为“网络版瑞士军刀”，旨在使技术和非技术分析人员能够以复杂的方式操作数据，而无需处理复杂的工具或算法。

![](https://xiaoyuwell.top/PicGo/20260518203430664.png)

![](https://xiaoyuwell.top/PicGo/20260518203442528.png)

#### 2.Theorem

![](https://xiaoyuwell.top/PicGo/20260518203520417.png)

![](https://xiaoyuwell.top/PicGo/20260518203533152.png)

```python
#task.py
from Crypto.Util.number import *
from gmpy2 import *
flag = b'xxx'
m =  bytes_to_long(flag)  # 把 flag 字节串 转换成 大整数 m（明文 m）
p = getPrime(512)	# 生成一个 512 位的大素数 p
q = next_prime(p)	# 生成 q：q 是 **比 p 大的下一个素数**
# 这是关键！p 和 q 非常接近，这是这题的漏洞点
e = 65537               #公钥的一部分
n = p * q                #n公钥的一部分，后续使用n--->p,q
phi = (p - 1) * (q - 1)       # 欧拉函数 φ(n) = (p-1)*(q-1)
d = inverse(e, phi)	# 求私钥 d：d 是 e 在模 φ(n) 下的逆元
# d * e ≡ 1 mod φ(n)
d1 = d % q               #私钥d的碎片，d1+d2--->d
d2 = d % p
c = pow(m, e, n)          #m^e mod n ---->c(密文)

print(n)
print(d1)
print(d2)
print(c)
# 94581028682900113123648734937784634645486813867065294159875516514520556881461611966096883566806571691879115766917833117123695776131443081658364855087575006641022211136751071900710589699171982563753011439999297865781908255529833932820965169382130385236359802696280004495552191520878864368741633686036192501791
# 4218387668018915625720266396593862419917073471510522718205354605765842130260156168132376152403329034145938741283222306099114824746204800218811277063324566
# 9600627113582853774131075212313403348273644858279673841760714353580493485117716382652419880115319186763984899736188607228846934836782353387850747253170850
# 36423517465893675519815622861961872192784685202298519340922692662559402449554596309518386263035128551037586034375613936036935256444185038640625700728791201299960866688949056632874866621825012134973285965672502404517179243752689740766636653543223559495428281042737266438408338914031484466542505299050233075829
```

\## 解题思路

这是 RSA 密码破解 题目，利用了 q = next_prime(p) 的特性（p 和 q 非常接近）：

![](https://xiaoyuwell.top/PicGo/20260518203729543.png)

1. 恢复 p 和 q ：由于 p ≈ q，可以通过对 n 开平方得到近似值，然后微调找到真正的 p
2. 计算 φ(n) ：φ = (p-1)(q-1)
3. 计算私钥 d ：d = e⁻¹ mod φ(n)
4. 解密 ：m = c^d mod n
5. 还原 flag ：bytes_from_long(m)

Yafu 获取p，q

pq非常接近时使用yafu

![](https://xiaoyuwell.top/PicGo/20260518203804184.png)

```python
#分解 n = p*q
#solve.py
from Crypto.Util.number import *
from gmpy2 import *
import libnum

n = 94581028682900113123648734937784634645486813867065294159875516514520556881461611966096883566806571691879115766917833117123695776131443081658364855087575006641022211136751071900710589699171982563753011439999297865781908255529833932820965169382130385236359802696280004495552191520878864368741633686036192501791
d1 = 4218387668018915625720266396593862419917073471510522718205354605765842130260156168132376152403329034145938741283222306099114824746204800218811277063324566
d2 = 9600627113582853774131075212313403348273644858279673841760714353580493485117716382652419880115319186763984899736188607228846934836782353387850747253170850
c = 36423517465893675519815622861961872192784685202298519340922692662559402449554596309518386263035128551037586034375613936036935256444185038640625700728791201299960866688949056632874866621825012134973285965672502404517179243752689740766636653543223559495428281042737266438408338914031484466542505299050233075829
e = 65537
p = iroot(n, 2)[0]
print(f"sqrt(n) ≈ p: {p}")

for offset in range(1000):
    candidate_p = p + offset
    if is_prime(candidate_p) and n % candidate_p == 0:
        p = candidate_p
        q = n // p
        print(f"Found p: {p}")
        print(f"Found q: {q}")
        break
        
#计算私钥 d
phi = (p - 1) * (q - 1)	# 计算欧拉函数 φ(n) = (p-1)*(q-1)
d = inverse(e, phi)	# 求私钥 d：d 是 e 在模 φ(n) 下的逆元
# d * e ≡ 1 mod φ(n)
#RSA 解密
m = pow(c, d, n)                 # 解密公式：m = c^d mod n
flag = long_to_bytes(m)	# 把大整数 m 转回字节串 → flag
print(f"Flag: {flag}")
#flag{5f00e1b9-2933-42ad-b4e1-069f6aa98e9a}
```



#### 4.Signature

椭圆曲线数字签名算法，它利用椭圆曲线密码学（ECC）对数字签名算法（DSA）进行模拟，其安全性基于椭圆曲线离散对数问题。但是当某些数值相同时会出现一些安全问题。

椭圆曲线数字签名算法(ECDSA）=椭圆曲线密码（ECC）+数字签名算法（DSA）

两个核心操作：

1. 签名（私钥签）
2. 烟钱（公钥验）



四个基础：

1. 曲线参数n：

secp256k1 曲线的阶，可以理解为 “模数”，所有运算都对 n 取模。

2. 私钥d：

一个 0 ~ n 之间的随机数

自己保管，不能泄露

3. 公钥Q：

由 Q = d * G 计算

G 是固定基点

可以公开

4. 随机数K：

每次签名必须重新生成！

一旦重复使用，私钥直接泄露

 

ECDSA签名过程：最终签名（r，s）

1. 随机数K：k = random(0, n)
2. 计算点P=K*G：取 P 的 x 坐标，记作 r，r = P.x mod n
3. 计算消息哈希h：h = sha256(消息)
4. 计算s：s = k-1 * (h + d * r)  mod n

 

为什么当某些数值相同时会出现一些安全问题：

如：重复K会泄露私钥

签名1：s1 = k-1 · (h1 + d · r) mod n

签名2：s2 = k-1 · (h2 + d · r) mod n

1-2：k = (h1 − h2) / (s1 − s2)  mod n ---->K被算出来了---->s1 = k-1 (h1 + d r)--->私钥d = (s1·k − h1) / r  mod n

总结：

ECDSA 签名 = (r, s)

r 由随机数 k 决定

s 由 k、私钥 d、消息哈希 h 决定

k 绝不能重复

一旦重复，可以直接算出私钥d

```python
#task.py
import ecdsa
import random

def ecdsa_test(dA,k):

    sk = ecdsa.SigningKey.from_secret_exponent(
        secexp=dA,
        curve=ecdsa.SECP256k1
    )
    sig1 = sk.sign(data=b'Hi.', k=k).hex()
    sig2 = sk.sign(data=b'hello.', k=k).hex()

    r1 = int(sig1[:64], 16)
    s1 = int(sig1[64:], 16)
    s2 = int(sig2[64:], 16)
    return r1,s1,s2

if __name__ == '__main__':
    n = 0xfffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364141
    a = random.randint(0,n)
    flag = 'flag{' + str(a) + "}"
    b = random.randint(0,n)
    print(ecdsa_test(a,b))
#r1,s1,s2
# (4690192503304946823926998585663150874421527890534303129755098666293734606680, 111157363347893999914897601390136910031659525525419989250638426589503279490788, 74486305819584508240056247318325239805160339288252987178597122489325719901254)

#solve.py
from Crypto.Util.number import *
import hashlib

n = 0xfffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364141

r = 4690192503304946823926998585663150874421527890534303129755098666293734606680
s1 = 111157363347893999914897601390136910031659525525419989250638426589503279490788
s2 = 74486305819584508240056247318325239805160339288252987178597122489325719901254

h1 = int(hashlib.sha1(b'Hi.').hexdigest(), 16)
h2 = int(hashlib.sha1(b'hello.').hexdigest(), 16)

print(f"h1 = {h1}")
print(f"h2 = {h2}")

k = (h1 - h2) * inverse(s1 - s2, n) % n
print(f"k = {k}")

dA = (s1 * k - h1) * inverse(r, n) % n
print(f"dA (private key) = {dA}")
flag = f"flag{{{dA}}}"
print(f"Flag: {flag}")
```



### 五、逆向分析

#### 1.欢乐时光（XXTEA）

```python
#main
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int i; // [rsp+Ch] [rbp-F4h]
  _DWORD v5[4]; // [rsp+10h] [rbp-F0h] BYREF
  _DWORD v6[12]; // [rsp+20h] [rbp-E0h]
  _DWORD buf[42]; // [rsp+50h] [rbp-B0h] BYREF
  unsigned __int64 v8; // [rsp+F8h] [rbp-8h]

  v8 = __readfsqword(0x28u);
  setvbuf(stdin, 0, 2, 0);
  setvbuf(stdout, 0, 2, 0);
  setvbuf(stderr, 0, 2, 0);
#密钥
  v5[0] = 2036950869;
  v5[1] = 1731489644;
  v5[2] = 1763906097;
  v5[3] = 1600602673;
#密文
  v6[0] = 1208664588;
  v6[1] = -829409294;
  v6[2] = -1943986152;
  v6[3] = 244490637;
  v6[4] = -1543286156;
  v6[5] = 611560113;
  v6[6] = -1443898536;
  v6[7] = -1523792440;
  v6[8] = -466433199;
  v6[9] = -800157149;
  v6[10] = 1875931283;
  printf(format);
#读取用户输入
  read(0, buf, 0x2Au);
#加密
  cry(buf, 11, (__int64)v5);
  for ( i = 0; i <= 10; ++i )
  {
    if ( buf[i] != v6[i] )
    {
      printf("a little error");
      exit(0);
    }
  }
  printf("yeah,this cup will definitely make you feel better!");
  return 0;
}
加密后 buf 必须等于 v6 → 输入就是 flag

#cry
a1：明文 / 密文（42 字节 → 11 个 32 位整数）
a2=11：分组数（11 组，每组 4 字节）
a3：密钥（4 个 32 位整数）
__int64 __fastcall cry(_DWORD *a1, int a2, __int64 a3)
{
  unsigned int *v3; // rax
  _DWORD *v4; // rax
  __int64 result; // rax
  unsigned int v6; // [rsp+20h] [rbp-18h]
  unsigned int v7; // [rsp+24h] [rbp-14h]
  unsigned int i; // [rsp+28h] [rbp-10h]
  int v9; // [rsp+2Ch] [rbp-Ch]
  int v10; // [rsp+30h] [rbp-8h]
  unsigned int v11; // [rsp+34h] [rbp-4h]

  v9 = 415 / a2 + 114;  #加密轮数=151
  v7 = 0;
  v6 = a1[a2 - 1];	#最后一组明文
  do
  {
    v7 -= 1640531527;
    v10 = (v7 >> 2) & 3;
    for ( i = 0; i < a2 - 1; ++i )
    {
      v11 = a1[i + 1];
      v3 = &a1[i];
      *v3 += ((v11 ^ v7) + (v6 ^ *(_DWORD *)(4LL * (v10 ^ i & 3) + a3)))
           ^ (((4 * v11) ^ (v6 >> 5)) + ((v11 >> 3) ^ (16 * v6)));
      v6 = *v3;
}

    v4 = &a1[a2 - 1];
    *v4 += ((*a1 ^ v7) + (v6 ^ *(_DWORD *)(4LL * (v10 ^ i & 3) + a3)))
         ^ (((4 * *a1) ^ (v6 >> 5)) + ((*a1 >> 3) ^ (16 * v6)));
    result = (unsigned int)*v4;
    v6 = result;
    --v9;
  }
  while ( v9 );
  return result;
}

#resolve，密文+密钥
#include <stdbool.h>
#include <stdio.h>
#define MX (((z >> 5) ^ (y << 2)) + ((y >> 3) ^ (z << 4)) ^ (sum ^ y) + (k[(p & 3) ^ e] ^ z))
#V:数据（明或密），n：正数=加密，负数=解密，k=密钥
bool btea(unsigned int *v, int n, unsigned int *k)
{
    unsigned int z = v[n - 1], y = v[0], sum = 0, e, DELTA = 0x61C88647;
unsigned int p, q;
#加密部分
    if (n > 1)
    { /* enCoding Part */
        q = 415 / n + 114;     #计算轮数
        while (q-- > 0)
        {
            sum += DELTA;
            e = (sum >> 2) & 3;
            for (p = 0; p < (n - 1); p++)
            {
                y = v[p + 1];
                z = v[p] += MX;    #加密：+=MX
            }
 
            y = v[0];
            z = v[n - 1] += MX;
        }
        return 0;
    }
    else if (n < -1)
    { /* Decoding Part */
        n = -n;
        q = 415 / n + 114;
        sum = -q * DELTA;
        while (sum != 0)
        {
            e = (sum >> 2) & 3;
            for (p = n - 1; p > 0; p--)
            {
                z = v[p - 1];
                y = v[p] -= MX;
            }
 
            z = v[n - 1];
            y = v[0] -= MX;
            sum += DELTA;
        }
        return 0;
    }
    return 1;
}
 
int main()
{
    unsigned int v[11] = {0x480AC20C, 0xCE9037F2, 0x8C212018, 0xE92A18D, 0xA4035274, 0x2473AAB1, 0xA9EFDB58, 0xA52CC5C8, 0xE432CB51, 0xD04E9223, 0x6FD07093}, key[4] = {0x79696755, 0x67346F6C, 0x69231231, 0x5F674231};
    int n = 11;       // n为要加密的数据个数
    btea(v, -n, key); // 取正为加密，取负为解密
    char *p = (char *)v;
    for (int i = 0; i < 44; i++)
    {
        printf("%c", *p);
        p++;
    }
    return 0;
}
//flag{efccf8f0-0c97-12ec-82e0-0c9d9242e335}
```



#### 2.rc4

一种流密码（stream cipher）

简单（代码简短），速度快，加解密使用同一个算法（对称加密）

密文=密钥流 异或 明文

明文=密钥流 异或 密文

总结：

RC4 就是用密钥生成一串随机数（密钥流），然后和数据逐字节异或。

 

核心步骤：

1. KSA（密钥调度算法）

用你的密钥打乱一个0—255的盒子（S盒）

2. PRGA（伪随机生成算法）

用打乱后的盒子，生成无限长的随机密钥流

然后使用密钥流和数据逐字节异或，完成加/解密

 

想象你有一副256 张的扑克牌（编号 0~255）。

第一步 KSA：洗牌

你用密钥当洗牌规则

把 256 张牌彻底打乱

→ 得到一个乱序的 S 盒

第二步 PRGA：发牌加密

从乱序牌里不断拿随机数

每拿一个随机数，就和你的数据异或一下

→ 得到加密结果

 

流程：

1. 初始化S盒：S = [0,1,2,3,...,255]
2. KSA洗牌（使用密钥打乱S盒）

j = 0

for i in 0~255:

  j = (j + S[i] + key[i % 密钥长度]) % 256

  交换 S[i] 和 S[j]

3. PRGA生成密钥流+加密

i = j = 0

for 每个数据字节:

  i = (i+1) % 256

  j = (j + S[i]) % 256

  交换 S[i] 和 S[j]

  t = (S[i] + S[j]) % 256

  密钥流字节 = S[t]

结果字节 = 数据字节 ^ 密钥流字节

密文+密钥

```python
int __cdecl main_0(int argc, const char **argv, const char **envp)
{
  size_t v4; // [esp+50h] [ebp-3Ch]
  _BYTE v5[44]; // [esp+54h] [ebp-38h] BYREF
  char Str[12]; // [esp+80h] [ebp-Ch] BYREF

  strcpy(Str, "gamelab@");  #密钥  
  v5[0] = -74;
  v5[1] = 66;
  v5[2] = -73;
  v5[3] = -4;
  v5[4] = -16;
  v5[5] = -94;
  v5[6] = 94;
  v5[7] = -87;
  v5[8] = 61;
  v5[9] = 41;
  v5[10] = 54;
  v5[11] = 31;
  v5[12] = 84;
  v5[13] = 41;
  v5[14] = 114;
  v5[15] = -88;
  v5[16] = 99;
  v5[17] = 50;
  v5[18] = -14;
  v5[19] = 68;
  v5[20] = -117;
  v5[21] = -123;
  v5[22] = -20;
  v5[23] = 13;
  v5[24] = -83;
  v5[25] = 63;
  v5[26] = -109;
  v5[27] = -93;
  v5[28] = -110;
  v5[29] = 116;
  v5[30] = -127;
  v5[31] = 101;
  v5[32] = 105;
  v5[33] = -20;
  v5[34] = -28;
  v5[35] = 57;
  v5[36] = -123;
  v5[37] = -87;
  v5[38] = -54;
  v5[39] = -81;
  v5[40] = -78;
  v5[41] = -58;
  v4 = strlen(Str);
  sub_401005((int)Str, v4, (int)v5, 42);
  printf("%s\n", Str);     #输出解密后的flag
  return 0;
}

#a1 = 密钥地址
#a2 = 密钥长度
#a3 = 数据地址（明文 / 密文）
#a4 = 数据长度
int __cdecl sub_401020(int a1, unsigned int a2, int a3, unsigned int a4)
{
  int result; // eax
  unsigned int k; // [esp+4Ch] [ebp-210h]
  char v6; // [esp+50h] [ebp-20Ch]
  char v7; // [esp+50h] [ebp-20Ch]
  unsigned int v8; // [esp+54h] [ebp-208h]
  unsigned int v9; // [esp+54h] [ebp-208h]
  unsigned int i; // [esp+58h] [ebp-204h]
  unsigned int j; // [esp+58h] [ebp-204h]
  unsigned int v12; // [esp+58h] [ebp-204h]
  _BYTE v13[512]; // [esp+5Ch] [ebp-200h]
#1.S盒初始化
  for ( i = 0; i < 256; ++i )
  {
    v13[i + 256] = i;
    v13[i] = *(_BYTE *)(a1 + i % a2);
  }
#2.KSA密钥调度（打乱S盒）
  v8 = 0;
  for ( j = 0; j < 256; ++j )
  {
    v8 = ((unsigned __int8)v13[j] + (unsigned __int8)v13[j + 256] + v8) % 256;
    v6 = v13[j + 256];
    v13[j + 256] = v13[v8 + 256];
    v13[v8 + 256] = v6;
  }
 #第三段：PRGA 生成密钥流 + 异或加密
 v9 = 0;
  result = 0;
  v12 = 0;
  for ( k = 0; k < a4; ++k )
  {
    v12 = (v12 + 1) % 256;
    v9 = (v9 + (unsigned __int8)v13[v12 + 256]) % 256;
    v7 = v13[v12 + 256];
    v13[v12 + 256] = v13[v9 + 256];
    v13[v9 + 256] = v7;
    result = (unsigned __int8)v13[v9 + 256];
    LOBYTE(result) = v13[(result + (unsigned __int8)v13[v12 + 256]) % 256 + 256] ^ *(_BYTE *)(k + a3);
    *(_BYTE *)(k + a3) = result;
  }
  return result;
}
```

```python
#解密脚本：
def rc4_decrypt(key, data):
    # 1. S盒初始化
    S = list(range(256))
    k = [key[i % len(key)] for i in range(256)]
    
    # 2. 密钥调度 KSA
    j = 0
    for i in range(256):
        j = (k[i] + S[i] + j) % 256
        S[i], S[j] = S[j], S[i]
    
    # 3. 伪随机生成 + 解密 PRGA
    i = j = 0
    result = []
    for byte in data:
        i = (i + 1) % 256
        j = (j + S[i]) % 256
        S[i], S[j] = S[j], S[i]
        t = (S[i] + S[j]) % 256
        key_stream = S[t]
        # 异或解密
        result.append(byte ^ key_stream)
    
    return bytes(result)

# ===================== 题目数据 =====================
# 密钥
key = b"gamelab@"

# 密文（main里v5数组的42个字节）
cipher = [
    -74, 66, -73, -4, -16, -94, 94, -87, 61, 41, 54, 31,
    84, 41, 114, -88, 99, 50, -14, 68, -117, -123, -20, 13,
    -83, 63, -109, -93, -110, 116, -127, 101, 105, -20, -28, 57,
    -123, -87, -54, -81, -78, -58
]

# 转无符号字节，-74--->0xB6
cipher_bytes = bytes([b & 0xFF for b in cipher])

# 解密
flag = rc4_decrypt(key, cipher_bytes)
print("解密结果：", flag.decode('utf-8'))

```

