---
title: "使用 YARA 检测 PHP WebShell：从规则原理到本地靶场实践"
description: "介绍 YARA 的基本概念、工作原理和使用方法，并在本地 PHPStudy 靶场中完成 PHP WebShell 检测实验。"
slug: "yara-php-webshell-detection"
date: 
image:
categories:
    - 应急响应
tags:
    - YARA
    - WebShell
    - PHP
    - 恶意代码检测
weight: 1
---

在恶意软件分析、应急响应和服务器安全检查中，我们经常需要回答一个问题：如何快速判断大量文件中是否存在具有某些特征的可疑文件？

逐个打开文件检查显然不现实。传统文件哈希只能识别完全相同的样本，一旦文件内容发生少量变化，哈希也会完全不同。YARA 提供了一种更加灵活的检测方式：安全人员可以将已知样本中的文本、字节序列和结构特征编写成规则，再使用规则批量扫描文件。

本文将介绍 YARA 的概念、工作原理、使用场景和规则结构，并结合本地 PHPStudy 靶场，演示如何使用 YARA 4.5.5 检测具有特定特征的 PHP WebShell。

## 一、YARA 是什么

YARA 是一种基于规则的模式匹配工具，主要用于识别和分类恶意软件或可疑文件。

可以把 YARA 理解为安全领域的“高级搜索引擎”。普通搜索工具通常只能查找单个关键词，而 YARA 可以同时描述多个特征，并通过逻辑条件决定文件是否命中。

例如，一段 PHP 代码中同时存在以下内容：

```text
@session_start
base64_decode($_POST
eval($payload)
getBasicsInfo
@run($data)
```

单独出现其中一个字符串，并不一定意味着文件是 WebShell。例如，`session_start` 在普通 PHP 项目中很常见。但是，当同一个文件中同时出现多个特征，尤其是解码用户输入、动态执行代码和特定函数名称的组合时，它的可疑程度就会明显提高。

YARA 的核心价值在于：将多个低置信度特征组合成一个高置信度检测条件。

## 二、YARA 的工作原理

YARA 的基本工作流程可以概括为四步：

1. 读取并解析 YARA 规则；
2. 检查规则是否存在语法错误；
3. 在目标文件中搜索规则定义的字符串或字节模式；
4. 根据 `condition` 中的逻辑表达式判断规则是否命中。

一个标准的 YARA 规则通常包含三个部分：

```yara
rule Example_Rule
{
    meta:
        description = "规则说明"
        author = "安全分析人员"

    strings:
        $s1 = "first string"
        $s2 = "second string"

    condition:
        all of them
}
```

### 1. meta：元数据

`meta` 用于记录规则的说明信息，例如：

- 规则用途；
- 作者和创建日期；
- 样本类型；
- 适用范围；
- 参考信息。

元数据不会直接参与匹配，但有助于规则维护和团队协作。

### 2. strings：检测特征

`strings` 用于定义需要搜索的内容。YARA 支持多种特征类型。

文本字符串：

```yara
$text = "base64_decode"
```

忽略大小写：

```yara
$text = "powershell" nocase
```

同时匹配 ASCII 和宽字符：

```yara
$text = "malicious" ascii wide
```

十六进制字节：

```yara
$hex = { 4D 5A 90 00 }
```

正则表达式：

```yara
$regex = /https?:\/\/[a-z0-9.-]+/i
```

### 3. condition：命中条件

`condition` 是规则的判断核心。

全部字符串命中：

```yara
condition:
    all of them
```

任意一个字符串命中：

```yara
condition:
    any of them
```

至少命中三个字符串：

```yara
condition:
    3 of them
```

也可以结合文件头和逻辑表达式：

```yara
condition:
    uint16(0) == 0x3f3c and 4 of ($s*)
```

这里表示文件开头的两个字节必须是 `<?`，并且以 `$s` 命名的字符串至少命中四个。两个条件同时成立时，规则才会触发。

## 三、为什么要使用 YARA

### 1. 比文件哈希更加灵活

MD5、SHA-1 或 SHA-256 只能识别完全相同的文件。样本只要修改一个字符，文件哈希就会改变。YARA 关注的是样本中的关键特征，即使文件哈希发生变化，只要核心特征仍然存在，规则就可能继续命中。

### 2. 可以组合多个特征

单个关键词容易产生误报，而 YARA 可以要求多个特征同时出现。例如：

```yara
condition:
    4 of ($s*)
```

这比单独查找 `eval` 或 `base64_decode` 更可靠。

### 3. 适合批量扫描

YARA 可以递归扫描整个目录，适用于：

- Web 服务器目录检查；
- 恶意软件样本分类；
- 应急响应排查；
- 邮件附件分析；
- 文件上传目录监控；
- 安全产品检测验证；
- 威胁狩猎。

### 4. 规则容易阅读和共享

YARA 规则接近自然语言，安全人员可以快速理解规则检测了哪些内容，也便于在团队之间共享。

需要注意的是，YARA 是检测和分类工具，并不是完整的杀毒软件。它不会自动判断文件是否绝对恶意，也不会默认删除或隔离命中的文件。

## 四、本地实验环境

本次实验使用以下环境：

```text
YARA 版本：
4.5.5

YARA 主程序：
E:\xxxxx\1\yara-4.5.5-2368-win64\yara64.exe

YARA 编译器：
E:\xxxx\1\yara-4.5.5-2368-win64\yarac64.exe

规则文件：
E:\xxxx\1\yara-4.5.5-2368-win64\new.yar

PHPStudy 靶场目录：
D:\xxxx\phpstudy\phpstudy_pro\WWW
```

本文采用只读扫描方式，不修改、删除或隔离靶场中的任何文件。

## 五、编写 PHP WebShell 检测规则

本次使用的规则如下：

```yara
rule Lab_PHP_Godzilla_Like_WebShell
{
    meta:
        description = "Detects the audited PHP XOR/Base64 session webshell"
        scope = "authorized lab"

    strings:
        $s1 = "@session_start" ascii
        $s2 = "base64_decode($_POST" ascii
        $s3 = "eval($payload)" ascii
        $s4 = "getBasicsInfo" ascii
        $s5 = "@run($data)" ascii

    condition:
        uint16(0) == 0x3f3c and 4 of ($s*)
}
```

### 规则分析

规则名称为：

```text
Lab_PHP_Godzilla_Like_WebShell
```

`meta` 明确说明该规则用于经过授权的实验环境。规则一共定义了五个字符串：

| 标识符 | 检测内容 | 含义 |
| --- | --- | --- |
| `$s1` | `@session_start` | 启动 PHP Session，并抑制错误信息 |
| `$s2` | `base64_decode($_POST` | 解码通过 POST 参数提交的数据 |
| `$s3` | `eval($payload)` | 动态执行变量中的 PHP 代码 |
| `$s4` | `getBasicsInfo` | 特定功能名称 |
| `$s5` | `@run($data)` | 调用数据处理或执行函数并抑制错误 |

最终条件为：

```yara
uint16(0) == 0x3f3c and 4 of ($s*)
```

`uint16(0)` 会从文件偏移 `0` 处读取两个字节。由于 YARA 默认按小端序解释整数，PHP 文件开头的 `<?` 对应 `0x3f3c`。因此，该规则要求目标文件以 PHP 开始标记起始，并且五个字符串中至少命中四个。

## 六、检查 YARA 版本

在 PowerShell 中执行：

```powershell
& "E:\xxxx\1\yara-4.5.5-2368-win64\yara64.exe" --version
```

预期输出：

```text
4.5.5
```

这说明 YARA 主程序可以正常运行。

## 七、检查规则语法

在扫描之前，建议先使用 `yarac64.exe` 编译规则：

```powershell
$Yarac = "E:\xxx\1\yara-4.5.5-2368-win64\yarac64.exe"
$Rule = "E:\xxxx\1\yara-4.5.5-2368-win64\new.yar"
$CompiledRule = Join-Path $env:TEMP "new.compiled.yarc"

& $Yarac $Rule $CompiledRule
Write-Host "规则编译返回码：$LASTEXITCODE"
```

预期结果：

```text
规则编译返回码：0
```

![](https://img.xiaoyuwell.top/PicGo/20260727154012586.png)

返回码 `0` 表示规则编译成功，没有发现语法错误。测试结束后，可以删除临时编译文件：

```powershell
Remove-Item -LiteralPath $CompiledRule -Force
```

## 八、递归扫描 PHPStudy 靶场

首先定义相关路径：

```powershell
$Yara = "E:\xxxx\1\yara-4.5.5-2368-win64\yara64.exe"
$Rule = "E:\xxxx\1\yara-4.5.5-2368-win64\new.yar"
$Target = "D:\xxxx\phpstudy\phpstudy_pro\WWW"
```

执行递归扫描：

```powershell
& $Yara -r $Rule $Target
```

其中：

- `-r` 表示递归扫描目标目录及其所有子目录；
- `$Rule` 是规则文件；
- `$Target` 是靶场网站根目录。

本次实验命中了两个文件：

```text
Lab_PHP_Godzilla_Like_WebShell D:\xxxx\phpstudy\phpstudy_pro\WWW\pikachu-master\vul\unsafeupload\uploads\test1.php
Lab_PHP_Godzilla_Like_WebShell D:\xxxx\phpstudy\phpstudy_pro\WWW\test.php
```

YARA 的默认输出格式为“规则名称 + 文件路径”。这说明两个文件均满足规则中定义的检测条件。

## 九、查看具体命中特征

只看到规则命中还不够，分析人员通常还需要知道文件具体命中了哪些字符串。可以使用 `-s` 参数：

```powershell
& $Yara -r -s $Rule $Target
```

本次扫描结果显示，两个文件均命中了全部五个特征。其中一个文件的结果类似：

```text
Lab_PHP_Godzilla_Like_WebShell D:\xxxx\phpstudy\phpstudy_pro\WWW\test.php
0x7:$s1: @session_start
0x149:$s2: base64_decode($_POST
0x242:$s3: eval($payload)
0x1ed:$s4: getBasicsInfo
0x306:$s4: getBasicsInfo
0x2a1:$s5: @run($data)
```

以这行结果为例：

```text
0x149:$s2: base64_decode($_POST
```

它表示：

- `0x149`：字符串在文件中的十六进制偏移；
- `$s2`：命中的规则字符串标识符；
- `base64_decode($_POST`：实际命中的内容。

虽然规则只要求至少命中四个特征，但本次两个样本均命中了全部五个特征，因此检测结果与预期一致。

![](https://img.xiaoyuwell.top/PicGo/20260727154210384.png)

## 十、保存扫描报告

为了便于后续分析，可以将扫描输出保存为文本报告：

```powershell
$Report = Join-Path $env:TEMP "yara_scan_result.txt"

& $Yara -r -s $Rule $Target 2>&1 |
    Tee-Object -FilePath $Report

Write-Host "扫描报告：$Report"
```

`Tee-Object` 会同时在终端显示扫描结果，并将结果写入报告文件。

需要注意，YARA 的退出码 `0` 表示程序正常完成扫描，并不等同于“一定发现了目标”。是否命中规则，应以输出中是否出现规则名称为准。

## 十一、规则优化

如果需要将规则用于真实环境，可以从以下方向改进：

- 使用正则表达式兼容空格和换行；
- 降低对固定变量名的依赖；
- 增加 PHP 文件类型和大小限制；
- 加入更加稳定的家族特征；
- 分离高置信度和低置信度规则；
- 使用正常文件集与恶意样本集测试误报和漏报；
- 在 `meta` 中加入版本、日期、作者和参考信息；
- 在部署前先于隔离测试环境中评估规则。

优化时不能一味放宽条件。规则越宽泛，覆盖率可能越高，但误报也可能随之增加。好的 YARA 规则需要在检出率和误报率之间取得平衡。

## 十二、总结

通过本次本地靶场实验，可以确认：

- YARA 4.5.5 能够正常加载和执行规则；
- `new.yar` 通过了实际编译检查；
- 规则能够递归扫描 PHPStudy 网站目录；
- 规则成功检测到 `test.php` 和 `test1.php`；
- 两个文件均命中了规则定义的全部五个特征；
- 整个扫描过程只读取文件，没有修改靶场内容。

YARA 的优势并不只是搜索某个敏感关键词，而是能够把文件格式、文本、字节模式和逻辑条件组合成可重复使用的检测规则。

在恶意软件分析、WebShell 排查和应急响应中，YARA 非常适合作为快速筛选工具。不过，YARA 命中只是调查的开始，最终结论仍应结合文件内容、服务器日志和运行行为综合判断。
