## 第3篇:Web日志分析

### ox01  Web日志

web服务器日志记录了Web服务器接收处理请求及运行时错误等各种原始信息。通过Web日志可以清楚的得知用户在什么IP、什么时间、用什么操作系统、什么浏览器的情况下访问了你网站的哪个页面，是否访问成功。通过对WEB日志进行的安全分析，可以还原攻击场景，不仅可以帮助我们追踪到攻击者，同时也可以找到了网站存在的安全漏洞并进行修复。

我们来看一条Apache的访问日志：

`127.0.0.1 - - [11/Jun/2018:12:47:22 +0800] "GET /login.html HTTP/1.1" 200 786 "-" "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.139 Safari/537.36"`

本文通过介绍Web日志安全分析时的思路和常用的一些技巧。

### 0x02 日志分析技巧

在对WEB日志进行安全分析时，一般可以按照两种思路展开，逐步深入，还原整个攻击过程。

~~~
第一种：确定入侵的时间范围，以此为线索，查找这个时间范围内可疑的日志，进一步排查，最终确定攻击者，还原攻击过程。

第二种：一般攻击者在入侵网站后，通常会上传一个后门文件，以方便自己以后访问，我们也可以以该文件为线索来展开分析。
~~~

常用分析工具：

~~~
Window下，推荐用 EmEditor 进行日志分析，支持大文本，搜索效率还不错。
Linux下，使用Shell命令组合查询分析
~~~

Shell+Linux命令实现日志分析，一般结合grep、awk等命令等实现了几个常用的日志分析统计技巧。

Apache日志分析：

~~~
1、列出当天访问次数最多的IP命令：
cut -d- -f 1 log_file|uniq -c | sort -rn | head -20

2、查看当天有多少个IP访问：
awk '{print $1}' log_file|sort|uniq|wc -l

3、查看某一个页面被访问的次数：
grep "/index.php" log_file | wc -l

4、查看每一个IP访问了多少个页面：
awk '{++S[$1]} END {for (a in S) print a,S[a]}' log_file

5、将每个IP访问的页面数进行从小到大排序：
awk '{++S[$1]} END {for (a in S) print S[a],a}' log_file | sort -n

6、查看某一个IP访问了哪些页面：
grep ^111.111.111.111 log_file| awk '{print $1,$7}'

7、去掉搜索引擎统计当天的页面：
awk '{print $12,$1}' log_file | grep ^\"Mozilla | awk '{print $2}' |sort | uniq | wc -l

8、查看2018年6月21日14时这一个小时内有多少IP访问:
awk '{print $4,$1}' log_file | grep 21/Jun/2018:14 | awk '{print $2}'| sort | uniq | wc -l	
~~~

刪除一个月前的日志：

```javascript
rm -f /www/logs/access.log.$(date -d '-1 month' +'%Y-%m')*			
```

统计爬虫：

```javascript
grep -E 'Googlebot|Baiduspider'  /www/logs/www.example.com/access.2011-02-23.log | awk '{ print $1 }' | sort | uniq
```

统计浏览器：

```javascript
cat /www/logs/example.com/access.2010-09-20.log | grep -v -E 'MSIE|Firefox|Chrome|Opera|Safari|Gecko|Maxthon' | sort | uniq -c | sort -r -n | head -n 100		
```

IP 统计：

```javascript
# grep '22/May/2012' /tmp/myid.access.log | awk '{print $1}' | awk -F'.' '{print $1"."$2"."$3"."$4}' | sort | uniq -c | sort -r -n | head -n 10
   2206 219.136.134.13
   1497 182.34.15.248
   1431 211.140.143.100
   1431 119.145.149.106
   1427 61.183.15.179
   1427 218.6.8.189
   1422 124.232.150.171
   1421 106.187.47.224
   1420 61.160.220.252
   1418 114.80.201.18			
```

统计网段：

```javascript
# cat /www/logs/www/access.2010-09-20.log | awk '{print $1}' | awk -F'.' '{print $1"."$2"."$3".0"}' | sort | uniq -c | sort -r -n | head -n 200			
```

统计域名：

```javascript
cat  /www/logs/access.2011-07-27.log |awk '{print $2}'|sort|uniq -c|sort -rn|more		
```

HTTP Status：

```javascript
# cat  /www/logs/access.2011-07-27.log |awk '{print $9}'|sort|uniq -c|sort -rn|more
5056585 304
1125579 200
   7602 400
      5 301	
```

URL 统计：

```javascript
cat  /www/logs/access.2011-07-27.log |awk '{print $7}'|sort|uniq -c|sort -rn|more			
```

文件流量统计：

```javascript
cat /www/logs/access.2011-08-03.log |awk '{sum[$7]+=$10}END{for(i in sum){print sum[i],i}}'|sort -rn|more

grep ' 200 ' /www/logs/access.2011-08-03.log |awk '{sum[$7]+=$10}END{for(i in sum){print sum[i],i}}'|sort -rn|more			
```

URL访问量统计：

```javascript
# cat www.access.log | awk '{print $7}' | egrep '\?|&' | sort | uniq -c | sort -rn | more			
```

脚本运行速度：

查出运行速度最慢的脚本

```javascript
grep -v 0$ access.2010-11-05.log | awk -F '\" ' '{print $4" " $1}' web.log | awk '{print $1" "$8}' | sort -n -k 1 -r | uniq > /tmp/slow_url.txt			
```

IP, URL 抽取：

```javascript
# tail -f /www/logs/www.365wine.com/access.2012-01-04.log | grep '/test.html' | awk '{print $1" "$7}'			
```



 参考链接：

> https://www.jb51.net/article/53954.htm
>
> https://www.jb51.net/article/58017.htm
>
> https://cloud.tencent.com/developer/article/1051427