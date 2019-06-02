## 第2篇:Linux日志分析

### 0x00 前言

Linux系统拥有非常灵活和强大的日志功能，可以保存几乎所有的操作记录，并可以从中检索出我们需要的信息。 本文简介一下Linux系统日志及日志分析技巧。

### 0x01 日志简介

日志默认存放位置：/var/log/

查看日志配置情况：more /etc/rsyslog.conf

|   日志文件    |             说明             |
| :-----------: | :--------------------------: |
| /var/log/cron | 记录了系统定时任务相关的日志 |
| /var/log/cups |      记录打印信息的日志      |
| /var/log/dmesg |      记录了系统在开机时内核自检的信息，也可以使用dmesg命令直接查看内核自检信息      |
| /var/log/mailog |      记录邮件信息    |
| /var/log/message |      记录系统重要信息的日志。这个日志文件中会记录Linux系统的绝大多数重要信息，如果系统出现问题时，首先要检查的就应该是这个日志文件   |
| /var/log/btmp |     记录错误登录日志，这个文件是二进制文件，不能直接vi查看，而要使用lastb命令查看      |
| /var/log/lastlog |      记录系统中所有用户最后一次登录时间的日志，这个文件是二进制文件，不能直接vi，而要使用lastlog命令查看      |
| /var/log/wtmp |      永久记录所有用户的登录、注销信息，同时记录系统的启动、重启、关机事件。同样这个文件也是一个二进制文件，不能直接vi，而需要使用last命令来查看    |
| /var/log/utmp |      记录当前已经登录的用户信息，这个文件会随着用户的登录和注销不断变化，只记录当前登录用户的信息。同样这个文件不能直接vi，而要使用w,who,users等命令来查询    |
| /var/log/secure |      记录验证和授权方面的信息，只要涉及账号和密码的程序都会记录，比如SSH登录，su切换用户，sudo授权，甚至添加用户和修改用户密码都会记录在这个日志文件中      |

比较重要的几个日志：
	登录失败记录：/var/log/btmp     //lastb
	最后一次登录：/var/log/lastlog  //lastlog
	登录成功记录: /var/log/wtmp     //last
	登录日志记录：/var/log/secure   

​	目前登录用户信息：/var/run/utmp  //w、who、users

​	历史命令记录：history
​	仅清理当前用户： history -c


### 0x02 日志分析技巧

#### A、常用的shell命令

Linux下常用的shell命令如：find、grep 、egrep、awk、sed

小技巧：

1、grep显示前后几行信息:

```

​	标准unix/linux下的grep通过下面參数控制上下文：
​	grep -C 5 foo file 显示file文件里匹配foo字串那行以及上下5行
​	grep -B 5 foo file 显示foo及前5行
​	grep -A 5 foo file 显示foo及后5行
​	查看grep版本号的方法是
​	grep -V
```

2、grep 查找含有某字符串的所有文件

```
	grep -rn "hello,world!" 
	* : 表示当前目录所有文件，也可以是某个文件名
	-r 是递归查找
	-n 是显示行号
	-R 查找所有文件包含子目录
	-i 忽略大小写
```

3、如何显示一个文件的某几行：

```
	cat input_file | tail -n +1000 | head -n 2000
	#从第1000行开始，显示2000行。即显示1000~2999行
```


4、find /etc -name init 

	//在目录/etc中查找文件init

5、只是显示/etc/passwd的账户

	`cat /etc/passwd |awk  -F ':'  '{print $1}'`  
	//awk -F指定域分隔符为':'，将记录按指定的域分隔符划分域，填充域，​$0则表示所有域,$1表示第一个域,​$n表示第n个域。

6、sed -i '153,$d' .bash_history

	删除历史操作记录，只保留前153行

#### B、日志分析技巧


**A、/var/log/secure**

~~~
1、定位有多少IP在爆破主机的root帐号：    
grep "Failed password for root" /var/log/secure | awk '{print $11}' | sort | uniq -c | sort -nr | more

定位有哪些IP在爆破：
grep "Failed password" /var/log/secure|grep -E -o "(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)"|uniq -c

爆破用户名字典是什么？
 grep "Failed password" /var/log/secure|perl -e 'while($_=<>){ /for(.*?) from/; print "$1\n";}'|uniq -c|sort -nr
 
2、登录成功的IP有哪些： 	
grep "Accepted " /var/log/secure | awk '{print $11}' | sort | uniq -c | sort -nr | more

登录成功的日期、用户名、IP：
grep "Accepted " /var/log/secure | awk '{print $1,$2,$3,$9,$11}' 

3、增加一个用户kali日志：
Jul 10 00:12:15 localhost useradd[2382]: new group: name=kali, GID=1001
Jul 10 00:12:15 localhost useradd[2382]: new user: name=kali, UID=1001, GID=1001, home=/home/kali
, shell=/bin/bash
Jul 10 00:12:58 localhost passwd: pam_unix(passwd:chauthtok): password changed for kali
#grep "useradd" /var/log/secure 

4、删除用户kali日志：
Jul 10 00:14:17 localhost userdel[2393]: delete user 'kali'
Jul 10 00:14:17 localhost userdel[2393]: removed group 'kali' owned by 'kali'
Jul 10 00:14:17 localhost userdel[2393]: removed shadow group 'kali' owned by 'kali'
# grep "userdel" /var/log/secure

5、su切换用户：
Jul 10 00:38:13 localhost su: pam_unix(su-l:session): session opened for user good by root(uid=0)

sudo授权执行:
sudo -l
Jul 10 00:43:09 localhost sudo:    good : TTY=pts/4 ; PWD=/home/good ; USER=root ; COMMAND=/sbin/shutdown -r now
~~~

**2、/var/log/yum.log**

软件安装升级卸载日志：

~~~yum install gcc
yum install gcc

[root@bogon ~]# more /var/log/yum.log

Jul 10 00:18:23 Updated: cpp-4.8.5-28.el7_5.1.x86_64
Jul 10 00:18:24 Updated: libgcc-4.8.5-28.el7_5.1.x86_64
Jul 10 00:18:24 Updated: libgomp-4.8.5-28.el7_5.1.x86_64
Jul 10 00:18:28 Updated: gcc-4.8.5-28.el7_5.1.x86_64
Jul 10 00:18:28 Updated: libgcc-4.8.5-28.el7_5.1.i686
~~~