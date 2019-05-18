## 第5篇：DDOS病毒

### 现象描述

某服务器网络资源异常,感染该木马病毒的服务器会占用网络带宽，甚至影响网络业务正常应用。

### 系统分析

针对日志服务器病毒事件排查情况：
在开机启动项/etc/rc.d/rc.local发现可疑的sh.sh脚本，进一步跟踪sh.sh脚本,这是一个检测病毒十分钟存活的脚本。

在root目录下发现存活检测脚本

![](.\image\linux-14-1.png)

解决步骤：

1. 结束进程 ps aux | grep "conf.m" | grep -v grep | awk ‘{print $2}‘| xargs kill -9 
2. 清除自动启动脚本 vim /etc/rc.local 去掉 sh /etc/chongfu.sh & 
3. 清除 脚本 rm -rf /etc/chongfu.sh /tem/chongfu.sh 
4. 修改登录密码 passwd 
5. 重启。 reboot