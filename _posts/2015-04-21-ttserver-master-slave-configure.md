---
layout: post
title: ttserver主辅同步配置说明
---

###ttserver主辅同步配置说明

####一、应用场景
业务系统中数据路由信息通过ttserver存取，运行期有大量的读写。为保证高可用，应用中写数据是同时写到所有ttserver节点中（但不存在事务控制）。仅仅采用这种方式并不靠谱，在某server网络中断或异常一段时间之后，即可能出现数据版本不一致。
ttserver本身可以支持主辅同步，配置主辅同步后，ttserver节点之间可以同步数据，可以保证各节点间数据一致。


####二、主辅配置说明
配置两台ttserver节点情况下，配置server1为server2的备用，server2为server1的备用，如果有超过两台节点，也保持这样的关系，形成一个环形结构。

```
server1: 10.11.20.128:48920
server2: 10.11.20.107:49920
```

主辅模式下，各节点启动脚本中必须增加如下参数：

```
-sid 128 #server节点标识ID，各server唯一
-mhost 10.11.20.107 #主辅模式下，主server节点host
-mport 49920 #主辅模式下，主server节点端口
-ulog /home/wangdong/aicau/app/cau/ulog/ #指定存放更新日志（update log）的目录，
-ulim 128m #每个数据同步日志文件的大小
-rts $PWD2/ttserver.rts #主辅模式下时间戳存放文件
```

启动脚本

```
bin/ttserver -port 48920 -thnum 12 -dmn -pid logs/ttsrv.pid -log logs/ttsrv.log **-sid 128 -mhost 10.11.20.107 -mport 49920 -ulog ulog/ -ulim 128m -rts ttserver.rts **
-le data/database.tch#bnum=10000#xmsiz=67108864
```

####附：ttserer参数说明

```
-host name :指明服务器的hostname或者ip地址。默认服务器的所有地址都会被绑定。比如：指定127.0.0.1这样的ip，就只是本地可以访问了。
-port num : 指定服务启动的端口. 默认1978.如果要启动多个数据库实例，端口需要不一样。
-thnum num : 指定服务工作的线程数。默认8.
-tout num : 指定每个会话的超时时间。默认永不超时。
-dmn : 以守护进程方式运行。
-pid path : 输出进程IP到指定的文件。
-log path : 输出日志信息到指定文件。
-ld : 日志中记录debug信息。
-le :日志中只记录错误信息。
-ulog path : 指定存放更新日志（update log）的目录.可以用来备份恢复数据库，主从库之间的同步。
-ulim num : 指定每个更新日志文件的大小限制.
-uas :使用异步IO记录更新日志。（使用此项可以减少写入日志的IO开销，但是在服务器意外关机，进程被kill时可能会丢失数据。根据经验，一般可以不使用）。
-sid num : 指定服务的ID号。主从复制的时候通过不同的ID号来识别。
-mhost name : 指定主从复制模式下的主服务器的IP或域名。
-mport num : 指定主从模式下主服务器的端口号.
-rts path : 指定用于主从复制的时间戳存放文件.
-ext path : 指定扩展脚本语言文件。
-extpc name period : 指定被周期调用的函数名和间隔时间.
-mask expr : 指定被禁止的命令名（比如可以禁止使用清空vanish）.
-unmask expr : 指定被允许的命令名.
```