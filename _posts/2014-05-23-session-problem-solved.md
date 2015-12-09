---
layout: post
title: 一次Session“混乱”的问题定位
---

###一次Session“混乱”的问题定位

业务部门反馈说最近在测试环境经常会有串Session的问题，明明通过A用户登录，操作日志却显示是B用户的，虽然不是必现问题，但是问题出现很频繁。

观察了下日志，在登录成功后，总是有两条日志记录的是错误的userCode，而后又恢复正常。如下面日志片段，是02BJQA36用户的操作日志，前两条01BJQA1出现得比较奇怪，而且随机出现。




> =====================reuqestPath : /CS/gridturnpage
[2014-05-19 14:36:36,949] [WebContainer-380] (01BJQA1) (SSOPopedomImpl.java:87) ERROR 
com.asiainfo.crm.sso.SSOPopedomImpl - isLogin called.........
[2014-05-19 14:36:36,950] [WebContainer-380] (01BJQA1) (SSOPopedomImpl.java:107) ERROR 
com.asiainfo.crm.sso.SSOPopedomImpl - User(200001069) prefer language  : zh_CN
[2014-05-19 14:36:36,955] [WebContainer-380] (02BJQA36 ) (DataStoreImpl.java:1004) DEBUG......


继续观察发现，同一个登录过程中，错误的userCode都是出现在以上两行代码处。上述两行日志都是在SSOPopedomImpl.isLogin中输出的。该方法是SSOFilter中的一部分逻辑，通过接口方式，交由业务应用自行实现，用于判断当前访问是否存在登录会话。如果没有，SSOFilter继续检查，如果Cookie中存在SSO认证票据，则会联系SSO服务端继续确认。否则，直接跳转到SSO登录页面。

在isLogin实现中，如果发现当前存在session，则会找到该session对应的user，并设置到SessionManager中，如以下代码片段：

```

log.error("isLogin called.........");
......
user = BaseServer.getCurUser(request);
log.error("User(" + user.getID() + ") prefer language : " + prefLanguage);
if (StringUtils.isNotBlank(prefLanguage))
	user.setPrefLanguage(prefLanguage);
SessionManager.setUser(user);
```
所以后续Session中就都会是正确的用户信息了。从这里看，这两行日志对业务操作并不会产生影响。

现在的问题是，为什么在SSOFilter刚开始判断时，SessionManager中就已经有了如此诡异的会话信息？开始怀疑是业务代码中有对SessionManager的误操作，检查半天，没有发现有用信息。最后才反应过来，SSOFilter是应用的第一个过滤器，所有请求第一步先通过SSOFilter，而这个时候就已经携带了用户信息，说明已经不是代码的问题，而很有可能Tomcat分配的处理线程的问题。

框架将sessionId与user对象的关系维护在HashMap中，同时提供SessionManager.getUser()接口，供外围获得当前用户对象。SessionManager内部基于ThreadLocal变量存储user。Tomcat这种多线程的模式下，每次请求都会从线程池中获取一个线程来处理请求，处理完毕，又将线程放回池中，所以该线程即会通过ThreadLocal携带缓存的用户信息。新的请求进来，如果从线程池中取到已经用过的线程，则会导致该线程中携带用户信息。

在原框架过滤器中，请求第一次进来时，会对线程变量作处理。即先全部置空，再根据sessionId找到的user重设。

```
//先将当前线程中的用户清空
SessionManager.setUser(null);
//先将当前线程中的语言信息清空
SessionManager.setLocale(null);
......
curUser = BaseServer.getCurUser(req);
SessionManager.setUser(curUser);
```

接入SSO后，SSOFilter配置为处理请求的第一个过滤器，而在该过滤器中也没有对线程历史信息做清理，所以就会出现以上问题。所以处理方法很简单，在第一个过滤器处理请求时，对线程变量作一次清理再赋值即可。













