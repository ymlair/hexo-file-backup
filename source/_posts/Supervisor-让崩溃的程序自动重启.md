---
title: Supervisor 让崩溃的程序自动重启
date: 2018-01-31 09:22:05
tags:
---

### Supervisor 介绍
Supervisor 是一个用 Python 写的进程管理工具，可以很方便的用来启动、重启、关闭进程（不仅仅是 Python 进程）。除了对单个进程的控制，还可以同时启动、关闭多个进程，比如很不幸的服务器由于某种原因暂时 kill 掉你的应用，此时可以用 Supervisor 让你的应用自动重启，如果是多个应用被杀死，也省去了手动一个一个地敲命令重新启动。

### 安装
目前 Supervisor 只能运行在 Unix-Like 的系统上，无法运行在 Windows 上。Supervisor 官方版目前只能运行在 Python 2.4 以上版本，但是还无法运行在 Python 3 上。执行下面代码前，需要[安装 pip](https://pip.pypa.io/en/stable/installing/)：

    pip install supervisor

安装完成后，可以使用两个命令，分别是 `supervisord` 和 `supervisorctl`,如果你的系统里有两个版本的 Python，且默认的 `python` 命令版本是 Python 3,此时运行会出错，解决方式是修改两个命令使用的 Python 版本。使用 `which` 命令找到两个命令的文件地址，然后编辑文件并指定 Python 版本：
![修改 Python 版本](http://upload-images.jianshu.io/upload_images/5306603-42fbaf472b45e0dc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 编辑配置文件

将下面内容保存到 `/etc/supervisor/supervisord.conf`:
```
; 基础配置样例

[unix_http_server]
file=/var/run/supervisor.sock   ; (the path to the socket file)
chmod=0700                       ; sockef file mode (default 0700)

[supervisord]
logfile=/var/log/supervisor/supervisord.log ; (main log file;default $CWD/supervisord.log)
pidfile=/var/run/supervisord.pid ; (supervisord pidfile;default supervisord.pid)
childlogdir=/var/log/supervisor            ; ('AUTO' child log dir, default $TEMP)

; the below section must remain in the config file for RPC
; (supervisorctl/web interface) to work, additional interfaces may be
; added by defining them in separate rpcinterface: sections
[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

[supervisorctl]
serverurl=unix:///var/run/supervisor.sock ; use a unix:// URL  for a unix socket

; The [include] section can just contain the "files" setting.  This
; setting can list multiple files (separated by whitespace or
; newlines).  It can also contain wildcards.  The filenames are
; interpreted as relative to this file.  Included files *cannot*
; include files themselves.

[include]
files = /etc/supervisor/conf.d/*.conf ;加载其他配置文件

[inet_http_server]         ; inet (TCP) server disabled by default
port=*:9001                ; 通过网页可以控制子进程
;username=user              ; (default is no username (open server))
;password=123               ; (default is no password (open server))

; 进程的配置样例

; 设置进程的名称，使用 supervisorctl 来管理进程时需要使用该进程名，这里的进程名是 your_program_name
[program:your_program_name] 
;numprocs=1                 ; 进程数量，默认为1
;process_name=%(program_name)s   ; 默认为 %(program_name)s，即 [program:x] 中的 x
directory=/home/yiming ; 执行 command 之前，先切换到工作目录
command=python test.py
autostart=true ;如果设置为true，当supervisord启动的时候，进程会自动重启。
user=yiming                 ; 使用 yiming 用户来启动该进程
autorestart=true   ; 程序崩溃时自动重启，重启次数是有限制的，默认为3次
startsecs = 5        ; 启动 5 秒后没有异常退出，就当作已经正常启动了           
redirect_stderr=true        ; 错误日志重定向到标准输出
loglevel=info


```

现在以守护进程的方式启动 `test.py`：

```
supervisord -c /etc/supervisor/supervisord.conf

```
此时命令 `python test.ty` 已经被执行，因为进程配置样例中有 `autostart=true`，所以 Supervisord 服务运行后启动进程 your_program_name ，并把 your_program_name 进程作为自己的子进程，所以当进程 your_program_name 挂掉后，Supervisord 会收到通知，然后可以再次将 your_program_name 作为子进程启动。

		

### 模拟程序异常退出
如下图，名称为 echo 的进程被杀掉两次，之后都会被重新启动，右侧是 Supervisord 日志记录了  echo 进程状态的变化：

![功能演示](http://upload-images.jianshu.io/upload_images/5306603-8e9dac42e1906d30.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 使用 supervisorctl 管理进程

* 停止某一个进程，program_name 为 [program:x] 里的 x：
```
supervisorctl stop program_name
```
* 启动某个进程：
```
supervisorctl start program_name
```
* 重启某个进程：
```
supervisorctl restart program_name
```
* 停止全部进程，注：start、restart、stop 都不会载入最新的配置文件：
```
supervisorctl stop all
```
* 载入最新的配置文件，停止原有进程并按新的配置启动、管理所有进程：
```
supervisorctl reload
```
* 根据最新的配置文件，启动新配置或有改动的进程，配置没有改动的进程不会受影响而重启：
```
supervisorctl update
```

### Web 管理
![Web 管理进程](http://upload-images.jianshu.io/upload_images/5306603-4a7b683d50405b4d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Supervisor 可以在界面上管理进程，Web Server 其实是通过 XML_RPC 来实现的，可以向Supervisor 请求数据，也可以控制 Supervisor 及子进程。配置在 `[inet_http_server]` 块里面：

```
[inet_http_server]         ; inet (TCP) server disabled by default
port=*:9001                ; 通过网页可以控制子进程
;username=user              ; (default is no username (open server))
;password=123               ; (default is no password (open server))
```

### 配置参数介绍

| 参数             | 参数解释 |
| ------------- | ------------- |
| command | 启动程序使用的命令，可以是绝对路径或者相对路径 |
| process_name | 一个python字符串表达式，用来表示supervisor进程启动的这个的名称，默认值是%(program_name)s |
| numprocs | Supervisor启动这个程序的多个实例，如果numprocs>1，则process_name的表达式必须包含%(process_num)s，默认是1 |
| numprocs_start | 一个int偏移值，当启动实例的时候用来计算numprocs的值 |
| priority | 权重，可以控制程序启动和关闭时的顺序，权重越低：越早启动，越晚关闭。默认值是999 |
| autostart | 如果设置为true，当supervisord启动的时候，进程会自动重启。 
| autorestart | 值可以是false、true、unexpected。false：进程不会自动重启，unexpected：当程序退出时的退出码不是exitcodes中定义的时，进程会重启，true：进程会无条件重启当退出的时候。 |
| startsecs | 程序启动后等待多长时间后才认为程序启动成功 |
| startretries | supervisord尝试启动一个程序时尝试的次数。默认是3 |
| exitcodes | 一个预期的退出返回码，默认是0,2。 |
| stopsignal | 当收到stop请求的时候，发送信号给程序，默认是TERM信号，也可以是 HUP, INT, QUIT, KILL, USR1, or USR2。 |
| stopwaitsecs | 在操作系统给supervisord发送SIGCHILD信号时等待的时间 |
| stopasgroup | 如果设置为true，则会使supervisor发送停止信号到整个进程组 |
| killasgroup | 如果设置为true，则在给程序发送SIGKILL信号的时候，会发送到整个进程组，它的子进程也会受到影响。 |
| user | 如果supervisord以root运行，则会使用这个设置用户启动子程序 |
| redirect_stderr | 如果设置为true，进程则会把标准错误输出到supervisord后台的标准输出文件描述符。 |
| stdout_logfile | 把进程的标准输出写入文件中，如果stdout_logfile没有设置或者设置为AUTO，则supervisor会自动选择一个文件位置。 |
| stdout_logfile_maxbytes | 标准输出log文件达到多少后自动进行轮转，单位是KB、MB、GB。如果设置为0则表示不限制日志文件大小 |
| stdout_logfile_backups | 标准输出日志轮转备份的数量，默认是10，如果设置为0，则不备份 |
| stdout_capture_maxbytes | 当进程处于stderr capture mode模式的时候，写入FIFO队列的最大bytes值，单位可以是KB、MB、GB |
| stdout_events_enabled | 如果设置为true，当进程在写它的stderr到文件描述符的时候，PROCESS_LOG_STDERR事件会被触发 |
| stderr_logfile | 把进程的错误日志输出一个文件中，除非redirect_stderr参数被设置为true |
| stderr_logfile_maxbytes | 错误log文件达到多少后自动进行轮转，单位是KB、MB、GB。如果设置为0则表示不限制日志文件大小 |
| stderr_logfile_backups | 错误日志轮转备份的数量，默认是10，如果设置为0，则不备份 |
| stderr_capture_maxbytes | 当进程处于stderr capture mode模式的时候，写入FIFO队列的最大bytes值，单位可以是KB、MB、GB |
| stderr_events_enabled | 如果设置为true，当进程在写它的stderr到文件描述符的时候，PROCESS_LOG_STDERR事件会被触发 |
| environment | 一个k/v对的list列表 |
| directory | supervisord在生成子进程的时候会切换到该目录 |
| umask | 设置进程的umask |
| serverurl | 是否允许子进程和内部的HTTP服务通讯，如果设置为AUTO，supervisor会自动的构造一个url |


