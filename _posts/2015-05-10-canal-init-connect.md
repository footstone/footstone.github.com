---
layout: post
title: Canal启动失败的问题定位
---


按照手册一步步操作，建立mysql用户并赋权之后，进程启动即报错。

```
CREATE USER aidc IDENTIFIED BY 'aidc';
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO aidc@'%';
FLUSH PRIVILEGES;
```

####异常内容

```

2015-04-10 15:48:25.352 [destination = sec2pos , address = /10.1.228.47:3660 , EventParser] WARN  c.a.otter.canal.parse.inbound.mysql.MysqlConnection - java.io.IOException: ErrorPacket [errorNumber=1184, fieldCount=-1, message=Aborted connection 13368360 to db: 'unconnected' user: 'aidc' host: '10.11.20.128' (init_connect command failed), sqlState=08S01, sqlStateMarker=#]

 with command: set wait_timeout=9999999
     at com.alibaba.otter.canal.parse.driver.mysql.MysqlUpdateExecutor.update(MysqlUpdateExecutor.java:49)
     at com.alibaba.otter.canal.parse.inbound.mysql.MysqlConnection.update(MysqlConnection.java:73)
     at com.alibaba.otter.canal.parse.inbound.mysql.MysqlConnection.updateSettings(MysqlConnection.java:172)
     at com.alibaba.otter.canal.parse.inbound.mysql.MysqlConnection.dump(MysqlConnection.java:105)
     at com.alibaba.otter.canal.parse.inbound.AbstractEventParser$3.run(AbstractEventParser.java:209)
     at java.lang.Thread.run(Thread.java:662)
```

分析异常，发现canal在初始化连接时居然还会设置一堆变量。

```
set wait_timeout=9999999

set net_write_timeout=1800
set net_read_timeout=1800
set names 'binary'
set @master_binlog_checksum= '@@global.binlog_checksum'
SET @mariadb_slave_capability='" + LogEvent.MARIA_SLAVE_CAPABILITY_MINE + "'
```

于是怀疑是aidc用户权限不足（不足以设置以上变量）导致。但是查了半天文档，也没发现针对变量设置的权限控制。而且这些变量都是会话期局部变量或者是用户变量，无需权限即可设置:(

最后才注意到异常中出现的一句`init_connect command failed`，才发现my.cnf中配置了`init_connect`属性，即所有连接初始化时都会执行的SQL。

```
init_connect='SET AUTOCOMMIT=0;SET NAMES utf8;insert into xxqa.xx_audit values(null,connection_id(),now(),user(),current_user());'
```

好吧，原来是需要`xxqa.xx_audit`的insert权限，增加该权限后，一切OK。

```
GRANT INSERT ON `xxqa`.* TO 'aidc'@'%';
FLUSH PRIVILEGES;
```

--EOF--