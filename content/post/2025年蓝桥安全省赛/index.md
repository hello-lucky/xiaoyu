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

EVTX文件是Windows操作系统生成的事件日志文件，用于记录系统、应用程序和安全事件。
（本题需要选手找出攻击者访问成功的一个敏感文件，提交格式为flag{文件名}，其中文件名不包含文件路径，
且包含文件后缀）
用Windows自带的事件查看器打开，点击右侧搜索按钮，搜索文件关键字：|

![](https://img.xiaoyuwell.top/PicGo/20260527153512442.png)

点击级别进行排序就可以看到异常文件

