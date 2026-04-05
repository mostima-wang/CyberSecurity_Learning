# Meterpreter
Meterpreter 是 Metasploit 框架中的一种攻击载荷（Payload），它通过许多极具价值的组件为渗透测试过程提供支持。Meterpreter 运行在目标系统上，在“命令与控制”（C2）架构中充当代理（agent）角色。你可以利用 Meterpreter 的专门命令，与目标的操作系统及文件进行交互。

根据目标系统的不同，Meterpreter 提供了多种版本，各版本具备不同的功能。

# Meterpreter 的工作原理
Meterpreter 运行在目标系统上，但并不会“安装”在上面。它直接在内存中运行，不会将自身写入目标系统的磁盘。这一特性旨在规避杀毒软件（AV）的扫描检测。默认情况下，大多数杀毒软件会扫描磁盘上的新文件（例如当你从互联网下载文件时），而 Meterpreter 运行在内存（RAM）中，从而避免产生必须写入磁盘的文件（例如 meterpreter.exe）。通过这种方式，Meterpreter 在目标系统中会被视为一个进程，而不是一个文件。

此外，Meterpreter 还旨在通过与运行 Metasploit 的服务器（通常是你的攻击机）进行加密通信，来躲避基于网络的入侵防御系统（IPS）和入侵检测系统（IDS）。如果目标组织不对进出本地网络的加密流量（如 HTTPS）进行解密和审查，IPS 和 IDS 方案将无法检测到其活动。

虽然主流杀毒软件现在已经能识别 Meterpreter，但上述特性依然提供了一定程度的隐蔽性。
# mpt版本
使用命令`msfvenom --list payload | grep meterpreter`可以查看可用的mpt。

决定使用哪个版本的 Meterpreter，主要基于以下三个因素：

1. 目标操作系统：
    目标系统是 Linux 还是 Windows？是 Mac 设备吗？还是 Android 手机？等等。

2. 目标系统上的可用组件：
    系统里是否安装了 Python？这是一个 PHP 网站吗？等等。

3. 你可以与目标系统建立的网络连接类型：
    目标网络允许原始 TCP 连接吗？你是否只能建立 HTTPS 反向连接？IPv6 地址的监控是否没有 IPv4 那么严格？等等。

4. 你的利用方式：
    如果是使用msfvenom生成，那版本随便挑。如果是直接在 Metasploit 里用 exploit 模块去。这种时候，有些漏洞程序已经帮你配好了最稳、最兼容的mpt版本，并且可能不支持某些mpt版本。

# 命令
- help：打开命令帮助菜单。每个版本mpt的命令选项各不相同，因此运行 help 命令始终是明智之举。
## 核心命令
- background：当前会话的背景
- exit：终止 Meterpreter 会话
- guid：获取会话 GUID（全局唯一标识符）
- info：显示有关Post模块的信息
- irb：在当前会话中打开一个交互式 Ruby shell
- load：加载一个或多个 Meterpreter 扩展
    - load kiwi：加载著名的 mimikatz（用于抓取 Windows 纯文本密码）。
    - load sniffer：加载嗅探器，抓取网络流量。
    - load python：在目标机上运行 Python 插件。
- migrate：允许您迁移mpt进入另一个过程，后跟目标进程的 PID。

    *例如，如果您看到目标系统上运行着文字处理器（例如 word.exe、notepad.exe 等），您可以迁移到该进程并开始捕获用户发送给该进程的按键。一般来说，进入后首先要迁入一个稳定的进程，比如lsass.exe*
- run：执行 Meterpreter 脚本或 Post 模块
- sessions：快速切换到另一个会话
这些是 Meterpreter 中最常用的 文件系统操作命令。如果你熟悉 Linux 终端或者 Windows 的 CMD，你会发现它们非常相似：

## 文件系统命令
- cd：切换目录 (Change directory)。
- ls：列出当前目录下的文件（使用 dir 命令也可以）。
- pwd：显示当前所在的工作目录 (Print working directory)。
- edit：编辑文件。
- cat：在屏幕上查看/打印文件内容。
- rm：删除指定的文件。
- search：搜索文件。类似于locate
- upload：上传文件或目录（从你的攻击机到目标机）。
- download：下载文件或目录（从目标机到你的攻击机）。

## 网络相关命令
- arp：显示主机的 ARP（地址解析协议）缓存表。用途： 发现同一局域网内的其他设备 IP 和 MAC 地址。

- ifconfig：显示目标系统上可用的网络接口信息（网卡、IP 地址等）。
- netstat：显示当前的网络连接状态。用途： 查看目标机器正在和谁通信，开启了哪些端口。

- portfwd（Port Forward）：将本地端口转发到远程服务。

    用途： 极度重要！如果你打进了一台内网机器，可以用它把内网里其他由于防火墙挡住、你原本访问不到的端口（比如 3389 远程桌面）转发到你的攻击机上。
- route：允许你查看和修改路由表。
    用途： 用于设置“内网穿透”，告诉 Metasploit 如何通过这台已控制的机器去访问更深层的内网网段。

## 系统相关命令 
- clearev：清除事件日志 (Clear Event Logs)，毁尸灭迹。
- execute：执行一个命令。用途：在目标机器上运行某个程序（比如运行一个你上传的 .exe）。
- getpid：显示当前的进程 ID (Process Identifier)。
- getuid：显示 Meterpreter 正在以哪个用户身份运行。
- ps：列出所有正在运行的进程。
- kill：通过进程 ID (PID)结束一个进程。
- pkill：按进程名称结束进程。如 pkill notepad.exe
- reboot：重启远程计算机。
- shell：切换到系统命令行。直接进入目标的 CMD (Windows) 或 Bash (Linux)
- shutdown：关闭远程计算机。
- sysinfo：获取远程系统的基本信息（如操作系统版本、架构等）。

## 其他高级功能命令 
1. 监控与间谍功能 (Monitoring)
- idletime：返回远程用户已**闲置（没动电脑）**的秒数。
- keyscan_start：开始捕获键盘记录。
- keyscan_dump：导出并查看已捕获的键盘记录（可以看到对方输入的密码、聊天记录等）。
- keyscan_stop：停止捕获键盘记录。
- screenshot：抓取远程桌面的一张截图。
- screenshare：实时监控/看对方的桌面（类似远程桌面看直播）。

2. 多媒体监控 (Media Control)
- record_mic：从默认麦克风录音 X 秒。
- webcam_list：列出目标机器上的所有摄像头。
- webcam_snap：用指定的摄像头拍一张照片。
- webcam_stream：从指定的摄像头开启视频直播流。
- webcam_chat：启动视频聊天（这通常会弹出窗口，极易暴露，慎用）。

3. 权限与核心数据 (Post-Exploitation)
- getsystem：尝试提权。

    *用途： 核心命令！尝试从普通用户权限升级为系统的最高权限（Local System）。*

- hashdump：导出 SAM 数据库的内容。

    *用途： 核心命令！直接获取系统所有用户的密码哈希值（拿到后可以去离线破解密码）。*

# 后渗透
