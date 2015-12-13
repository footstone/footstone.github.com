---
layout: post
title: 线上应用FullGC导致服务不可用问题分析全过程
---

***（为避免可能存在的信息泄露，此文部分信息信息脱敏，单纯记录问题解决过程，留存备忘。）***

###一、现象

下午1:00我们负责的系统发布后，大约在14:30左右开始，该系统业务处理过程的RT陡增，导致上游系统数据下发和下游系统回传处理超时。

###二、线上分析

1. 随机登录一台主机，通过top查看，发现该主机内存所剩不足1M，其中Java进程占用内存很高。
2. jstat工具查看JVM垃圾回收状态，FullGC达到200多次，初步确认就是FullGC导致的。
3. 检查JVM内存配置项，未发现问题，内存空间足够，没有做过调整，怀疑与最新发布的代码版本有关。
4. 进一步观察jstat状态，发现FullGC的主要原因在于JVM O区增长太快。于是调整Xmn大小为2000m，给O区增加500m，并重启进程。然并卵，继续FullGC。
5. jmap导出堆内存，线下分析dump文件。
6. 为减少故障影响，决定先回退代码版本。回退版本发布后，应用服务能力正常。

###三、线下分析

<img src="http://i1371.photobucket.com/albums/ag283/njzeroc/Develop/fullgc-dump.png_zpsky7prcp2.jpeg" width="600px"/>

通过MAT分析线上导出的dump文件，分析本次内存泄露的主要原因是因为该应用通过异步的方式向消息中间件发消息，实现方式是将消息对象塞到`ArrayBlockingQueue`里，然后通过另一独立线程每隔一秒从其中取出一条消息消费发出，而该`ArrayBlockingQueue`声明时也并没有初始化其最大长度限制。

所以至此基本明确了，**<u>队列消息消费太慢，导致大量消息对象堆积，是造成本次故障的主要原因</u>**。但是诡异的是，这套代码在线上已经运行了很长一段时间，当天发布的代码版本跟该实现也并没有关系。那么为什么之前没有故障，而单单是本次代码发布后导致了这次故障呢？为了回答这个问题，主要确认了两件事情：

1. 观察前几日的内存监控数据，发现前一周的主机内存占用量在85%以上，已经很高。
2. 仔细分析本次发布的增量代码，人肉review每行代码，唯一可疑点在于一处对报文的处理逻辑:通过调用8次replaceAll函数去除报文中的特定字符。

```

sbLog.append(requestMsg.replace("\r\n", "").replaceAll("\n", "").replaceAll("\\\\n", "").replaceAll(" ", ""));
sbLog.append("|");
sbLog.append(responseMsg.replace("\r\n", "").replaceAll("\n", "").replaceAll("\\\\n", "").replaceAll(" ", ""));
sbLog.append("|");
```

根据上面两个情况猜测，前期该应用内存占用量已经很高，达到了一个临界状态（只能这么猜测，因为拿不到前几日的GC日志）。本次代码版本发布后，replace逻辑直接成为了压垮应用的最后一根稻草，成为触发本次故障的导火索。为了验证这个想法，线下针对两个代码版本分别做了压测，期望能重现问题。

###四、压测模拟
针对发布前后的3个版本各自做10分钟的压测，并发数500。观察两次压测状态下JVM内存状况，并通过jstat观察GC状态。

####_V1（发布前的代码版本）_
<img src="http://i1371.photobucket.com/albums/ag283/njzeroc/Develop/fullgc-v1_zps5ogi7sln.png" width="600px"/>

####_V2（发布后的代码版本，含8次replace）_
<img src="http://i1371.photobucket.com/albums/ag283/njzeroc/Develop/fullgc-v2_zpskg1olf4r.png" width="600px"/>

####_V3(修复后的代码版本，删除repalce、取消异步消息发送)_
<img src="http://i1371.photobucket.com/albums/ag283/njzeroc/Develop/fullgc-v3_zpsqot7kg73.png" width="600px"/>

V1版本在压测进行到第10分钟开始出现FullGC，V2在压测进行到第8分钟开始出现FullGC，且V2版本后续的FullGC状态十分频繁，而V1版本的内存上涨过程则相对平稳。V3版本内存状态极其稳定。由此，**<u>可以确认replace代码确实是压垮JVM内存的最后一根稻草</u>**。

###五、修复动作

1. 消息发送中间件逻辑由异步改为同步。
2. 去除replace代码逻辑。

###六、分析工具

- 观察JVM GC状态 `jstat -gcutil {pid} 1000 `
- JVM堆内存dump ` jmap -dump:format=b,file=heap.bin {pid}`
- MAT工具分析dump文件，有内存泄露分析功能。
- visualvm，进程开启JMX开关，远程实时监控JVM状态。

	```
	-Dcom.sun.management.jmxremote.port=1090 
	-Dcom.sun.management.jmxremote.authenticate=false 
	-Dcom.sun.management.jmxremote.ssl=false
	```
--EOF--








