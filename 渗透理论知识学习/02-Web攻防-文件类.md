# Web攻防-文件类

## 1. 文件上传漏洞

网站允许用户上传图片、视频、头像等文件，恶意用户可以利用文件上传的漏洞将WebShell上传至服务器，获得网站权限。
- JS前端绕过：针对前端JS代码校验文件后缀。

    判断：看提示框是否在抓到数据包之前弹出。

    绕过：先把木马改为jpg，抓包后再改回php。
- 文件头绕过：服务器检验上传内容的前几个字节。

    1. 直接上传php，抓包修改在内容前面加上相应文件头，并修改content-type字段的值。
    2. 图片木马。使用copy命令将一张普通图片和木马合并成为图片木马。需要结合文件包含漏洞进行利用。
- 黑名单缺陷
    
    判断：根据上传后的特殊提示框来判断

    绕过：上传特殊后缀的php文件，如php3
- 00截断: 本质：让服务器解码 %00 得到 0x00（NULL 字节）

    在 php<5.3.4 版本中，存储文件时处理文件名的函数认为0x00是终止符。于是在存储文件的时候，当函数读到 0x00(%00) 时，会认为文件已经结束。

    我们上传 1.php%00.jpg 时，首先后缀名是合法的jpg格式，可以绕过前端的检测。上传到后端后，后端判断文件名后缀的函数会认为其是一个.jpg格式的文件，可以躲过白名单检测。但是在保存文件时，保存文件时处理文件名的函数在遇到%00字符认为这是终止符，于是丢弃后面的 .jpg，于是我们上传的 1.php%00.jpg 文件最终会被写入 1.php 文件中并存储在服务端。
        - 使用post方式上传时，需要提前对%00进行解码。
- htaccess绕过

    htaccess作为Apache服务器的分布式配置文件，允许目录级配置覆盖。攻击者可借此绕过安全限制，如先上传.htaccess文件强制将图片解析为PHP，再上传伪装成GIF的WebShell。

       
## 2. 文件下载漏洞
    
用户利用漏洞能够下载服务器上的敏感文件。
1. 漏洞特征
- URL中可能回存在一些特殊参数，若可控，则可以用来下载指定文件以外的文件。
    - 如：path, file, data, filepath, readfile, url, realpath 等。
2. 漏洞简单验证:不同系统中，验证方法稍有区别。

    windows可以尝试下载c:\Windows\win.ini

    linux可以尝试读取/etc/passwd、/etc/hosts等。
3. 利用方式

    如果真的存在文件下载漏洞，那么可以重点关注一些系统的敏感文件。
    ```
    Windows 系统:

    - C:\boot.ini //查看系统版本
    - C:\windows\system32\inetsrv\MetaBase.xml //IIS配置文件
    - C:\windows\repair\sam //存储Windows系统初次安装的密码
    - C:\ProgramFiles\mysql\my.ini //Mysql配置
    - C:\ProgramFiles\mysql\data\mysql\user.MYD //MySQL root密码
    - C:\windows\php.ini //php配置信息

    Linux/Unix系统:

    - /etc/password //账户信息
    - /etc/shadow //账户密码信息
    - /usr/local/app/apache2/conf/httpd.conf //Apache2默认配置文件
    - /usr/local/app/apache2/conf/extra/httpd-vhost.conf //虚拟网站配置
    - /usr/local/app/php5/lib/php.ini //PHP相关配置
    - /etc/httpd/conf/httpd.conf //Apache配置文件
    - /etc/my.conf //mysql配置文件
    ```

## 3. 文件包含漏洞

什么叫包含呢？以PHP为例，我们常常把可重复使用的函数写入到单个文件中，在使用该函数时，直接调用此文件，而无需再次编写函数，这一过程叫做包含。

A文件包含B，可以理解为A文件直接把B文件复制粘贴到自己的运行环境里运行了。

1. 本地文件包含截断：开发者在用户传入参数后面加了.php的后缀限制，需要用截断来突破。
    1. 00截断：在最后加入一个0字节（%00） 
    2. 路径长度截断：windows下目录最长256字节，linux4096字节，用`./`或奇数个`.` 填充。
2. 远程文件包含截断：目标网站在我们的文件名后面添加.xxx 
    1. ?截断：浏览器识别?及之后为查询语句
    2. \#截断：需要对\#进行编码，变为%23
3. php伪协议
    1. `file://` 用于访问本地文件系统，在CTF中通常用来读取本地文件的且不受allow_url_fopen与allow_url_include的影响

        `127.0.0.1/include.php?test=file://D:/study/apache-tomcat-8.5.33/www/phpinfo.txt`
    
    2. `php://filter` 用于读取源码并进行base64编码输出，解码后便可以看到源代码。
    3. `php://input` 可以访问请求的原始数据的只读流, 将post请求中的数据作为PHP代码执行。将参数设为php://input,同时post想设置的文件内容，php执行时会将post内容当作文件内容。从而导致任意代码执行
    4. `zip://` 可以访问压缩包里面的文件。当它与包含函数结合时，zip://流会被当作php文件执行。从而实现任意代码执行。
        - zip://中只能传入绝对路径。
        - 要用#分割压缩包和压缩包里的内容，并且#要用url编码成%23（即下述POC中#要用%23替换）
        - 只需要是zip的压缩包即可，后缀名可以任意更改。
        - 相同的类型还有zlib://和bzip2://

        `zip://[压缩包绝对路径]#[压缩包内文件]`
        
        例`?file=zip://D:\1.zip%23phpinfo.txt`
    5. `data://` 同样类似与php://input，可以让用户来控制输入流，当它与包含函数结合时，用户输入的data://流会被当作php文件执行。从而导致任意代码执行。
    
        `data://text/plain,<?php phpinfo();?>`
    
        如果此处对特殊字符进行了过滤，我们还可以通过base64编码后再输入：
        `data://text/plain;base64,PD9waHAgcGhwaW5mbygpPz4=`
    
        其中text/plain是声明后面是纯文本，绕过WAF/过滤规则
- 实战之包含图片木马getshell：上传.jpg文件，其中写入待执行的php木马，用文件包含漏洞包含执行该jpg文件。
- 实战之包含日志getshell：抓取任意请求数据，向其中写入php木马，多次请求，将访问信息（包括木马）写入目标服务器的日志中，再用文件包含漏洞包含其日志以运行木马。