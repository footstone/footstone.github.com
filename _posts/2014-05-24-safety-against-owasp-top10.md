---
layout: post
title: 基于OWASP Top10的安全编码规范
---

###基于OWASP Top10的安全编码规范

因为企业内网应用的开发背景，我（及部门团队）在开发过程中，基本没有考虑过安全相关的问题-_-。安全问题琐碎零散，通过OWASP Top10来梳理或许算得上是一种比较清晰明确的方式。根据以往经验及收集到的材料，整理了一份安全编码规范。

#### 一、注入

1. 不使用java.sql.Statement执行SQL，而是使用java.sql.PreparedStatement。
2. 对SQL参数值中的单引号要检查并过滤。
3. 对WEB请求或接口参数中的select、update、delete、truncate、drop等非法字符作拦截过滤。
4. 禁止向外暴露数据库等信息，如异常处理等，具体参考防止信息泄露章节。
5. 基于最小权限原则，减少数据库用户操作权限。
6. 使用相对路径Canonical Path，避免文件路径含有../等字符。如：
	```
File file = new File(str);
file.getAbsolutePath();//no
file.getCanonicalPath();//yes
	```
	
7. 尽量避免使用Runtime.getRuntime().exec()，如必须使用，则尽量避免传入动态参数，并注意检查特定字符。


#### 二、认证和会话管理

1. 在用户连续登陆失败的情况下，需要限制其请求，如增加验证码、锁定账号等。
3. 支持限定用户登陆的IP、MAC绑定功能，限定登陆范围。
4. 用户登出或浏览器关闭时，即立刻注销会话，删除与该会话相关的所有临时数据，并及时清理认证票据。
5. 会话时间尽量设短。	
6. Cookie票据加密存储，并设置HttpOnly、Secure属性。
7. 禁止明文存储密码。
8. 尽量减少在HttpSession对象中的存储数据。
9. 登陆成功后重新建立sessionId。
10. 用户会话中的sessionId禁止作为参数传递。


#### 三、XSS

* 对HTTP请求进行检查

    因为前台输入的字符串具有很大的随意性，且基于js的输入校验很容易被突破，所以必须在server端进行输入校验。
	* 校验原则：
	1) 基于白名单原则的校验，对于不符合格式的一律拒绝。
	2) 基于正确格式的校验，对于不符合格式的，拒绝或替换非法字符。

	* 对HttpServletRequest参数值要进行校验，校验范围包括：

```
	request.getParameter
	request.getQueryString
	request.getHeader
	request.getCookies
```

* 对HTTP响应进行编码

如果仅仅对请求进行处理，而不处理响应，那么通过其他渠道进入到系统的恶意脚本还是有可能被执行，所以必须对输出也要处理。具体方式是采用HTML编码方式对危险字符进行编码，包括：

* JSP页面中使用JSTL标签输出以避免XSS
