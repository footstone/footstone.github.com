---
layout: post
title: 一次跌宕起伏的Log4j配置验证测试
---

###一、背景

得知项目生产环境log4j的配置是直接输出到Console时，我很惊讶，以为发现了一个疏忽点，可以优化日志配置再提高点系统性能。因为在开发规范中是强烈禁止使用System.out方式输出日志或其他信息的，一方面输出到Console在生产环境时没有意义，另一方面对系统性能有影响。而Log4j的ConsoleAppender最终就是直接通过System.out（或System.err）方式直接输出的。
而我也一直想当然的以为生产配置必然是file（FileAppender）的方式，并且~~Log4j对file的处理是基于异步方式实现的~~。为了验证想法，做了这个测试。

跌宕起伏的剧情暂且略过，重点是测试结果。

###二、过程

测试过程在公司项目测试环境主机中进行。写了两个程序，分别模拟单线程和多线程并发两种场景，按不同的log4j配置，各运行4次，记录每次消耗时间。

1. 单线程，纯写日志10w次，如表中：console、file、buffered-file
2. 多线程，20个线程并发，每个线程各写5k次日志，如表中从multi-console到multi-file-async

#####测试结果

Configure         | Time1         | Time2        | Time3        | Time4
------------      | ------------- | ------------ | ------------ | ----------
console           | 41263ms	       | 98951ms      | 58342ms      | 72711ms
file              | 11028ms	       | 9268ms       | 8034ms       | 9744ms
buffered-file     | 9985ms	       | 5650ms       | 8862ms       | 5612ms
multi-console     | 62175ms	       | 23576ms      | 56032ms      | 24075ms
multi-file        | 7234ms	       | 7374ms       | 7188ms       | 7378ms
console-cronolog  | 7885ms	       | 7584ms       | 7810ms       | 7851ms 
console-nohup     | 7479ms	       | 8365ms       | 7582ms       | 7660ms
multi-file-async  | 7224ms	       | 7342ms       | 6959ms       | 7237ms

#####Log4j配置：

1.	console、multi-console、console-cronolog、console-nohup

	```
	log4j.appender.console=org.apache.log4j.ConsoleAppender
	log4j.appender.console.layout=org.apache.log4j.PatternLayout
	log4j.appender.console.layout.ConversionPattern=[%d] [%t] (%F:%L) %-5p %c - %m%n
	```
2.	file

	```
	log4j.appender.file=org.apache.log4j.DailyRollingFileAppender
	log4j.appender.file.File=/Users/xxx/Develop/logs/log.log
	log4j.appender.file.DatePattern='.'yyyy-MM-dd
	log4j.appender.file.layout=org.apache.log4j.PatternLayout
	log4j.appender.file.layout.ConversionPattern=[%d] [%t] (%F:%L) %-5p %c - %m%n
	```
3.	buffered-file、multi-file

	```
	log4j.appender.file=org.apache.log4j.DailyRollingFileAppender
	log4j.appender.file.File=/Users/footstone/Develop/logs/log.log
	log4j.appender.file.DatePattern='.'yyyy-MM-dd
	log4j.appender.file.layout=org.apache.log4j.PatternLayout
	log4j.appender.file.layout.ConversionPattern=[%d] [%t] (%F:%L) %-5p %c - %m%n
	log4j.appender.file.BufferedIO=true
	log4j.appender.file.BufferSize=8192
	```
4.	multi-file-async
	
	```
	<appender name="file" class="org.apache.log4j.DailyRollingFileAppender">  
		<param name="File" value="/Users/footstone/Develop/logs/log.log"/>
		<param name="DatePattern" value="'.'yyyy-MM-dd" />
		<param name="Append" value="true" />
	 	<param name="BufferedIO" value="true" />
	 	<param name="BufferSize" value="8192" />
		<layout class="org.apache.log4j.PatternLayout">
		<param name="ConversionPattern"	value="[%d] [%t] (%F:%L) %-5p %c - %m%n" />  
	 	</layout>
	 </appender>
	<appender name="async" class="org.apache.log4j.AsyncAppender">  
		<param name="BufferSize" value="1024" />
		<appender-ref ref="file" />
	</appender>
	```
5.	console-cronolog
	
	```
java com.ai.log.Test 2>&1 | ${BASE_HOME}/sbin/cronolog -k 3 ${BASE_HOME}/log/test_log-%Y%m%d.log &
	```
6.	console-nohup
	
	```
	nohup java com.ai.log.Test &
	```

###三、总结

1.	单线程和多线程的两个场景下，都证明log4j写文件比直接输出到console效率更高。
2.	log4j配置输出到console时，效率最低，但是当程序是后台运行时，即使用nohup或者cronolog方式输出，而不是直接占着当前终端输出时，效率会提升很多。并且占着当前终端输出时，性能起伏较大。
3.	log4j写文件的方式，会丢失日志。10w条日志平均会丢失90条左右，而使用console-nohup和console-cronolog时则很完整，不存在丢失日志的情况。
4.	使用文件记录时，设置buffered参数很重要，即BufferedIO和BufferedSize。
5.	使用异步写文件的方式，效率最高，但是在解决file方式丢失日志问题之前，还是使用console-nohup或console-cronolog方式较为靠谱。





