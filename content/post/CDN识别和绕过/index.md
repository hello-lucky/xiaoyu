---
title: "CDN识别和绕过"
description: 
slug: 
date: 
image:
categories:
    - stu
tags:
    - CDN
weight: 1

---



# 一、CND核心原理

CDN核心原理是将网站的静态资源缓存到分布在各地的边缘节点上，当用户发送请求时，DNS解析会根据用户地理位置将其引导到距离最近、速度最快的缓存节点IP。

为什么要识别：

​	如果直接对 CDN 节点进行 `Nmap` 扫描，你扫描到的只是 CDN 厂商的服务器，不仅无法发现源站开放的真实端口，还极易触发 CDN 的防线导致 IP 被封禁。



如何识别网站是否使用了CDN进行了加速：

1.超级ping：

​	未开启：各地区返回的响应 IP 完全一致。

​	已开启：不同地区、不同运营商返回多个截然不同的 IP。



2.Nslookup/Dig：通过命令行查询域名的 CNAME 记录

```
dig target.com
```



​	如果输出的 `ANSWER SECTION` 中包含类似于 `*.w.kunlunsl.com`（阿里云 CDN）、`*.tl88.net`（网易云信）或 `*.cloudfront.net` 等明显的 CNAME 别名，则说明必定开启了 CDN

# 二、CDN绕过

### 1.子域名突破

​	**原理：**为了节约成本，有的网站仅为流量大的主站配置了加速而忽略了其他子域名。

​	**实操步骤：**

1.子域名爆破（OneForAll等）对域名进行全面枚举

2.过滤搜集到的二级、三级域名

3.对这些子域名进行IP解析。判断：如果发现某个子域名解析出公网IP独立的C段且运行着内部系统，那么它大概率是主站源IP。

### 2.国外节点访问（地域配置差）

​	**原理：**如果网站至配置了国内CDN加速时，当由来自海外请求时，CDN 节点没有覆盖，DNS就会直接将流量放行、直连到位于国内的源站物理机。

​	**实操步骤**：

1. 使用海外 VPS，或者利用多地 Ping 工具的海外节点。

2. 执行解析命令：

	```jboss-cli
	curl -I https://target.com --proxy http://海外代理IP:端口
	```

3. 对比国内解析结果，若海外节点返回的 IP 是国内某电信/联通机房的固定 IP，即可高度怀疑其为真实 IP。

### 3.漏洞诱导（服务器主动外联）

​	**原理：**攻击者外联时必然经过CDN代理节点，但让目标服务器主动向攻击者的服务器发起请求，流量则会绕过CDN直接暴露出真实IP。

​	**实操方法（SSRF）：**

1. 在目标网站寻找具有网络交互功能的接口，如：远程图片加载、头像上传（输入 URL 解析）、Markdown 编辑器解析远程图片、Webmail 附件预览、URL 采集器。

2. 在你控制的公网 VPS 上启动网络监听：

	```apache
	nc -lvvp 8080
	```

3. 在目标的 SSRF 漏洞输入框或图片加载位置，填入你的 VPS 地址：`http://YOUR_VPS_IP:8080/test.jpg`。

4. 触发漏洞后，查看 VPS 的 `nc` 监听日志。此时，向你的 VPS 发起 `GET` 请求的客户端 IP，就是目标 Web 服务器的真实公网 IP。

### 4.遗留文件或者敏感信息泄露

​	**原理：**开发或运维人员在部署、调试环境时，可能不小心将测试文件（如 `phpinfo.php`、`info.jsp`）或代码仓库（`.git`、`.svn`）遗留在线上环境中。

​	**实操步骤：**

1. 使用目录扫描工具（如 `Dirsearch`、御剑等）对网站目录进行深度扫描。
2. 如果幸运地扫描到了 `phpinfo.php`，直接在网页中通过 `Ctrl + F` 搜索以下关键字：
	- `_SERVER["SERVER_ADDR"]`（服务器 IP）
	- `_SERVER["LOCAL_ADDR"]`
	- `HTTP_X_FORWARDED_FOR`（查看链路信息）

### 5.邮件服务器直连

​	**前提：**邮件不能是第三方代发（如腾讯、网易、阿里云）要不然只会抓到它们的机房IP。

​	**实操方法（被动接受）：**

1. 在目标网站寻找能够触发系统自动发信的功能点，例如：**用户注册激活、密码找回、绑定邮箱、发送电子发票/订阅新闻**。

2. 填写你自己的测试邮箱（如 QQ 邮箱、Gmail），等待接收目标网站发来的邮件。

3. 收到邮件后，不要只看正文。点击邮箱自带的 **“显示邮件网页源码” / “查看原始邮件”**（View Raw）。

4. 在弹出的邮件头（Header）文本中，从下往上寻找 `Received` 字段：

	```applescript
	Received: from target.com (unknown [XXX.XXX.XXX.XXX])
	by qq.com (XXXX) with SMTP id ...
	```

	其中的 `XXX.XXX.XXX.XXX` 即为目标发送邮件的服务器真实 IP。

### 6.DNS解析历史记录

​	**原理:**网站创建初期大多没有上限CDN而是直接解析到服务器上。后期由于遭受太多DDoS攻击或者业务扩展，中途接入CDN，当往往没有更换物理服务器的公网IP，可通过查询历史DNS档案可找到“裸奔”时的真实IP。

​	接口查询：

​	ViewDNS（https://viewdns.info）

​	SecurityTrails（https://securitytrails.com）

​	Get Site IP（https://get-site-ip.com/）

​	全球CDN查询（https://tools.ipip.net/）

​	DNSDumpster（https://dnsdumpster.com/）

### 7.网络空间测绘

​	**原理**：像 FOFA、Shodan、ZoomEye 这样的空间测绘引擎，会全天候对全球全网的 IP 进行端口扫描和抓取。如果某个 CDN 厂商（如 Cloudflare）在代理源站时，源站的 80/443 端口依然对全网开放，那么测绘引擎就会绕过域名，直接通过 IP 抓取到该源站的 Web 页面特征。

​	**实操方法（FoFA）：**

1. **利用网站独有特征匹配**：假设目标网站的 HTML 源代码中包含一段独特的统计代码、备案号或者特定的 Title。

	```ini
	title="目标网站独特标题" && country="CN"
	body="icp.gov.cn/xxxxxx"
	```

2. **利用 SSL 证书指纹（极高成功率）**：如果目标网站部署了 SSL 证书，可以通过证书的序列号或 SHA256 指纹在全网搜索直接绑定该证书的真实 IP。

	```abnf
	cert="目标域名的证书名称"
	```

	在搜索结果中，排除掉属于 CDN 厂商（如 Cloudflare、Akamai）的 IP 段，剩下的独立服务器 IP 极有可能就是源站。

