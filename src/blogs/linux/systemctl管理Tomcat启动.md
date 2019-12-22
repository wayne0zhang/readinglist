# systemctl管理Tomcat启动、停止、重启、开机启动
## 1. 创建服务
用service来管理服务的时候，是在/etc/init.d/目录中创建一个脚本文件，来管理服务的启动和停止，在systemctl中，也类似，文件目录有所不同，在/lib/systemd/system目录下创建一个脚本文件tomcat，里面的内容如下：
```properties
[Unit]
Description=Tomcat
After=network.target

[Service]
Type=forking
PIDFile=/usr/local/tomcat/pid
ExecStart=/usr/local/tomcat/bin/catalina.sh start
ExecReload=/usr/local/tomcat/bin/catalina.sh restart
ExecStop=/usr/local/tomcat/bin/catalina.sh stop

[Install]
WantedBy=multi-user.target
```

- [Unit] 表示这是基础信息\
    Description 是描述\
    After 是在那个服务后面启动，一般是网络服务启动后启动
- [Service] 表示这里是服务信息\
    Type 是服务类型\
    PIDFile 是服务的pid文件路径， 开启后，必须在tomcat的bin/catalina.sh中加入CATALINA_PID参数\
    ExecStart 是启动服务的命令\
    ExecReload 是重启服务的命令\
    ExecStop 是停止服务的指令
- [Install] 表示这是是安装相关信息\
    WantedBy 是以哪种方式启动：multi-user.target表明当系统以多用户方式（默认的运行级别）启动时，这个服务需要被自动运行。\
    更详细的说明请访问：[这里](https://www.csdn.net/article/2015-02-27/2824034)

```
tomcat的bin/catalina.sh中加入CATALINA_PID参数时，需要在# OS specific support.上加入

CATALINA_PID=/usr/local/tomcat/pid

# OS specific support.  $var _must_ be set to either true or false.

cygwin=false
....略..
```

## 2. 创建软链接
创建软链接是为了下一步系统初始化时自动启动服务

```
ln -s /lib/systemd/system/tomcat.service /etc/systemd/system/multi-user.target.wants/tomcat.service
```

创建软链接就好比Windows下的快捷方式
ln -s 是创建软链接
ln -s 原文件 目标文件（快捷方式的决定地址）

如果创建软连接的时候出现异常，不要担心，看看/etc/systemd/system/multi-user.target.wants/ 目录是否正常创建软链接为准，有时候报错只是提示一下，其实成功了。

```
$ ll /etc/systemd/system/multi-user.target.wants/
total 8
drwxr-xr-x  2 root root 4096 Mar 30 15:46 ./
drwxr-xr-x 13 root root 4096 Mar 13 14:18 ../
lrwxrwxrwx  1 root root   31 Nov 23 14:43 tomcat.service -> /lib/systemd/system/tomcat.service
...略...
```

## 3. 刷新配置
刚刚配置的服务需要让systemctl能识别，就必须刷新配置

```
$ systemctl daemon-reload
```

如果没有权限可以使用sudo
```
$ sudo systemctl daemon-reload
```

## 4. 启动、重启、停止
启动tomcat
```
$ systemctl start tomcat
```
重启tomcat
```
$ systemctl restart tomcat
```
停止tomcat
```
$ systemctl stop tomcat
```

## 5. 开机自启动
tomcat服务加入开机启动
```
$ systemctl enable tomcat
```
禁止开机启动
```
$ systemctl disable tomcat
```
## 6. 查看状态
查看状态
```
$ systemctl status tomcat
```
————————————————
版权声明：本文为CSDN博主「大大的微笑」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/chwshuang/article/details/68489699