---
layout: post
title: AWS培训记录
---
###一、印象

#####1. API方式控制服务

所有传统上硬件、中间件、系统、运行参数等，需要手动去安装、配置的设备都可以通过API的方式实现，并且比较快速。API支持多种语言。

#####2. 不可靠

EC2作为虚拟机，虽然类似于传统上硬件主机的概念，但崇尚于团队战术，不推崇单兵作战能力。单个EC2随时可以挂掉，也随时可以切换，所以核心数据不能只存储在EC2上。

#####3. 弹性

所有的服务都支持动态的资源调整，按需使用，按使用付费。可以通过监控平台获取服务资源的使用情况，根据资源使用率动态调整。如ELB可根据访问流量动态调整配置。

#####4. 自动化配置

可以通过定义服务（EC2、数据库、缓存等）配置模板、快照等，比较快速的搭建新的环境。减少了传统上的主机、网络、中间件等安装配置的重复劳动过程。

**个人觉得AWS提供的很多虚拟化服务，确实简化了很多传统设备（诸如硬件、操作系统、中间件、数据库、网络设备等等）的维护管理工作，如安装部署、节点增删、集群切换、纵向扩展等。而同时AWS(包括其他的云化平台)对于应用系统的分布式设计要求也是更加彻底的，更倾向于通过横向扩展的方式增强运算能力，对于各设备的使用也是很灵活。如此，对于需要迁移到云平台的应用，很多设计和实现也是需要重新考虑评估。比如由于EC2的不可靠，可能需要重新考虑应用会话状态的保持。
**

###二、笔记

#####区域和数据中心
AWS提供的服务在从物理范围上讲，包括以下概念：

   * Region：最大的涵盖范围，比如美国、新加坡等，每个Region可提供的服务有区别。北京Region还没对外公开。
   * AZ：可用区，在Region范围下，属于逻辑概念。每一个Region下至少有两个可用区(支持灾备)
   * 数据中心：在AZ下，有多个数据中心。


#####EC2(虚拟主机)

   * 思想上需要认定EC2不可靠，重要数据不应存储在EC2上，因为EC2是逻辑实例，每次启动后其所对应的物理磁盘地址是不固定的。
   * 可以通过UserData设定在EC2启动时做一些事情，比如设定IP、启动某些服务等。
   * 可以根据业务场景选择不同的EC2配置（CPU、内存等），AWS根据场景提供了多种EC2类型，如计算型、
   * 操作系统支持Linux、Windows
   * 可以定义Alarm，触发EC2自动伸缩调整（纵向）。
   * 可建立AMI(快照)，之后按快照建立和启动EC2，原来部署的程序和配置都会保留，建立和启动EC2的时间大概在几秒到几分钟之间。
   * 按小时收费。


#####存储

   * 支持S3、EBS、Glacier三种类型
   * S3适用于一次写、N次读，Region级别，会在多个可用区中备份，支持网络（如http）访问。
   * Bucket是S3的逻辑存储容器，每个账号限定100个Bucket，每个Bucket存储容量在0B-5TB之间。Dropbox目前用了两个Bucket。
   * 快照存储在S3中。
   * EBS：块级存储卷，独立于EC2，也可作为EC2的文件系统，但不受EC2使用寿命影响。AZ(可用区)级别，多个EC2可共用一个EBS，也可不与EC2连接，单独作为存储服务。
   * EBS可用于创建RAID配置，RAID 0或RAID 10
   * EBS支持SSD。
   * EBS不能通过网络访问。
   * Glacier适用于存储不常使用的数据，价格很低。读取需要4到5小时。
   * 各存储类型均支持导入、导出。
   * 支持数据加密，密钥由用户自己管理（CloudHSM），丢失就没了。


#####安全

   * 建立EC2时可选择建立安全组（防火墙）以保障EC2安全，具体安全组可以做哪些事情，没有得到细节确认。其他服务也都有安全组件，可能每个安全组件能做的事情也是不一样的。
   * 支持SSL创建安全通信会话（HTTPS）。
   * CloudWatch通过监控流量和异常操作等保障安全。
   * 对于应用层的攻击防范，如XSS\SQL注入等，AWS没有提供支持。
   * 不同账号的资源完全隔离，某账号出现安全问题之后不会影响到其他账号。
   * IAM账号管理。
   * Trusted Advisor，监控安全配置漏洞，如密码强度、端口开放过、IAM使用不合理等。
   * 私有子网。

#####ElastiCache(缓存)

   * 支持Memcached和Redis两种方式，可在控制台或通过API创建节点。
   * AWS对两种缓存未作任何二次包装。
   * ElastiCache可自动检测和更换出故障的节点。
   * 包含安全组件，限定访问网络等。

#####联网

   * 分别支持建立私有子网和公有子网，私有子网可通过建立VPN Gateway访问。
   * Route53，DNS，管理域名。
   * ELB，负载均衡，根据流量动态动态扩展，支持状态检查。
   * 支持CDN，CloudFront

#####IAM（身份认证及权限管理）

   * 注册账号为根用户，类似于linux系统中的root用户，可以根据需要自建多个子账号。
   * 可以管理用户组、角色，并定义policy，管理账号权限。
   * 可以与企业现有活动目录集成，而不需要为现有用户再创建IAM用户。
   * 可以为用户分配安全证书以支持用户对AWS服务API的访问权限（适用于程序中对API的访问）。

#####数据库

   * 支持关系型数据库（RDS）和NoSQL数据库（DynamoDB），其他Nosql，如MongoDB可以自己在EC2上安装。
   * RDS支持Oracle、MySQL等四种数据库，License自备。
   * 调优参数通过在管理控制台上设置。
   * 主备节点，保持数据一致是通过同步写的方式。
   * 提供完整的数据库功能，未作包装和控制，但是在数据库用户权限上有限制。

#####CloudFormation

   * 支持定义架构模板（包含实例配置、数据库、缓存、启动参数等），也可以从运行环境导出。根据该模板自动创建所有节点，对于快速搭建运行环境很有帮助。JSON格式。

#####其他

   * 支持Hadoop、MapReduce、数据仓库（Redshift），按需收费，用完即关。
   * 支持日志集中分析。
   * CloudWatch，监控。

#####资源链接

   * qwikLAB：[https://aws.qwiklab.com](https://aws.qwiklab.com)
   * AWS培训资源：[http://aws.amazon.com/cn/training](http://aws.amazon.com/cn/training)
   * AWS认证：[http://aws.amazon.com/cn/certification](http://aws.amazon.com/cn/certification)
   * AWS实践实验室：[http://aws.amazon.com/cn/training/self-paced-labs](http://aws.amazon.com/cn/training/self-paced-labs)
   * whitepapers: [http://aws.amazon.com/cn/whitepapers/](http://aws.amazon.com/cn/whitepapers/)
   * AWS文档：[http://aws.amazon.com/cn/documentation/](http://aws.amazon.com/cn/documentation/)
   * 案例研究：[http://aws.amazon.com/solutions/case-studies](http://aws.amazon.com/solutions/case-studies)
   * AWS博客：[http://aws.typepad.com](http://aws.typepad.com)
   * AWS安全：[http://aws.amazon.com/cn/security/](http://aws.amazon.com/cn/security/)
   * 架构：[http://aws.amazon.com/cn/architecture](http://aws.amazon.com/cn/architecture)

