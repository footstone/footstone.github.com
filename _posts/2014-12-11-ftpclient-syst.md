---
layout: post
title: FTPClient的syst问题
---

###问题jar包
apache-commons-net-1.4.1.jar

###问题说明
`FTPClient.listFiles`函数需要根据FTP服务器的操作系统类型，创建对应的`FTPFileEntryParser`,确定FTP服务器`SystemType`的方法是使用FTP协议中的`syst`命令。

```

$ ftp release_ftp@10.11.20.108

Connected to 10.11.20.108.
220 (vsFTPd 2.0.5)
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> syst
215 UNIX Type: L8
ftp>
```

在某些情况下，通过该命令获取`SystemType`会存在问题，导致`getSystemName()`返回值为`null`，从而`FTPFileEntryParser`对象创建失败，即会导致异常。

```
[2014-12-05 12:09:31,521] [TaskFrameWork_Worker-5] () (KOBFileImportTask.java:142) ERROR com.asiainfo.crm.cm.bi2oneframe.exe.task.ImportUserTask - Task:10030013 Import FileFailed….
         org.apache.commons.net.ftp.parser.ParserInitializationException: Error initializing parser
        at org.apache.commons.net.ftp.parser.DefaultFTPFileEntryParserFactory.createFileEntryParser(DefaultFTPFileEntryParserFactory.java:129)
        at org.apache.commons.net.ftp.FTPClient.initiateListParsing(FTPClient.java:2358)
        at org.apache.commons.net.ftp.FTPClient.listFiles(FTPClient.java:2141)
        at com.ai.common.util.FtpUtil.getFtpFile(FtpUtil.java:285)
        at com.asiainfo.crm.cm.dk.exe.task.KOBFileImportTask.doTask(KOBFileImportTask.java:136)
        at com.asiainfo.appframe.ext.exeframe.task.TaskJob.execute(TaskJob.java:98)
        at org.quartz.core.JobRunShell.run(JobRunShell.java:202)
```

在2.2版本之后，`getSystemName()`改为`getSystemType()`，并且会避免返回null，在syst无效后，会取启动参数的配置值`-DFTP_SYSTEM_TYPE_DEFAULT`

所以遇到该问题的修复办法可以是,升级版本，并增加启动参数`-DFTP_SYSTEM_TYPE_DEFAULT=UNIX`

###LINKS
- http://commons.apache.org/proper/commons-net/changes-report.html
- https://issues.apache.org/jira/browse/NET-338
- https://issues.apache.org/jira/browse/NET-440

--EOF--