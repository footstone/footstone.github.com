---
layout: post
title: Linux跨主机大文件传输技巧
---

###一、gzip + scp
```
gzip -c /backup/mydb.myd > mytable.myd.gz

scp mytable.myd.gz root@server2:/var/lib/mysql/mydb/
```

###二、gzip + scp，减少磁盘读写，更快
```
gzip -cl /backup/mydb.myd | ssh root@server "gunzip -c - > /var/lib/mysql/mydb/mytable.myd"
```

###三、非跨网环境传输，不需要ssl加解密的情况下使用nc，更快

***从server1向server2传输，先在server2上开启监听端口12345，传输到该端口的文件都直接gunzip到指定目录***

```
nc -l -p 12345 | gunzip -c - > /var/lib/mysql/mydb/mytable.myd
```

***server1压缩后传输到server2对应端口上***

```
gzip -cl - /backup/mydb.myd | nc -q 1 server2 12345
```

***server2使用tar命令***

```
nc -l -p 12345 | tar xvzf -
```
***server1使用tar命令***

```
tar cvzf - /var/lib/mysqlmydb/mytable.myd | nc -1 1 server2 12345
```
###四、耗时对比

方法         | 		时间         
------------	| ------------- 
rsync，不使用压缩           | 71
scp，不使用压缩           | 68
nc，不使用压缩           | 67
rsync，使用压缩（-z）    | 63
gzip，scp和gunzip    | 60（44+10+6）
ssh，使用压缩    | 44
nc，使用压缩    | 42


--EOF--