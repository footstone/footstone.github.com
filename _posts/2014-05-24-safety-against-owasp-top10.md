---
layout: post
title: 基于OWASP Top10的安全编码规范
---

###基于OWASP Top10的安全编码规范

因为企业内网应用的开发背景，我（及部门团队）在开发过程中，基本没有考虑过安全相关的问题-_-。安全问题琐碎零散，通过OWASP Top10来梳理或许算得上是一种比较清晰明确的方式。根据以往经验及收集到的材料，整理了一份安全编码规范。

*(本规范将长期维护)*

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
		
7. 尽量避免使用Runtime.getRuntime().exec()，如必须使用，则尽量避免传入动态参数，并注意检查特定字符,如：


#### 二、认证和会话管理

1. 在用户连续登陆失败的情况下，需要限制其请求，如增加验证码、锁定账号等。
2. 支持限定用户登陆的IP、MAC绑定功能，限定登陆范围。
3. 用户登出或浏览器关闭时，即立刻注销会话，删除与该会话相关的所有临时数据，并及时清理认证票据。
4. 会话时间尽量设短。	
5. Cookie票据加密存储，并设置HttpOnly、Secure属性。
6. 禁止明文存储密码。
7. 尽量减少在HttpSession对象中的存储数据。
8. 登陆成功后重新建立sessionId。
9. 用户会话中的sessionId禁止作为参数传递。

#### 三、XSS

* 对HTTP请求进行检查

因为前台输入的字符串具有很大的随意性，且基于js的输入校验很容易被突破，所以必须在server端进行输入校验。校验原则：

1) 基于白名单原则的校验，对于不符合格式的一律拒绝。
2) 基于正确格式的校验，对于不符合格式的，拒绝或替换非法字符。

对HttpServletRequest参数值要进行校验，校验范围包括：
	
```
	request.getParameter
	request.getQueryString
	request.getHeader
	request.getCookies
```

* 对HTTP响应进行编码

如果仅仅对请求进行处理，而不处理响应，那么通过其他渠道进入到系统的恶意脚本还是有可能被执行，所以必须对输出也要处理。具体方式是采用HTML编码方式对危险字符进行编码。

* JSP页面中使用JSTL标签输出以避免XSS

#### 四、不安全的直接对象引用

一个已经授权的用户，通过更改访问时的一个参数，从而访问到了原本其并没有得到授权的对象。Web应用往往在生成Web页面时会用它的真实名字，且并不会对所有的目标对象访问时来检查用户权限，所以这就造成了不安全的对象直接引用的漏洞。例如开发者将数据库记录主键ID作为URL的一部分暴露给用户，可能被恶意用户利用进行恶意操作

* 使用基于用户或者会话的间接对象引用
	
这样能防止攻击者直接攻击未授权资源。例如，一个下拉列表包含6个授权给当前用户的资源，它可以使用数字1-6来指示哪个是用户选择的值，而不是使用资源的数据库关键字来表示。在服务器端，应用程序需要将每个用户的间接引用映射到实际的数据库关键字。

* ID稀疏
	
不使用自增ID，而使用随机字符串作数据记录ID的方式，减少被用户恶意操纵的可能。如：

```
http://www.xxx.com/getCustomerInfo.jsp?id=xcdfuwolqpox
```

* 检查访问

任何来自不可信源的直接对象引用都必须通过访问权限检查，确保该用户对请求的对象有访问权限。

#### 五、安全配置错误

这个问题可能存在于Web应用的各个层次，譬如平台、Web服务器、应用服务器，系统框架，甚至是代码中。开发人员需要和网络管理人员共同确保所有层次都合理配置，如错误的安全配置，缺省用户是否存在，不必要的服务等等。

1. 主机配置时，通过统一的脚本配置环境，关闭ip6tables\NetworkManager等不必要的服务。
2. 安装中间件时，关闭资源目录的列目录功能。
3. 对于服务器，数据库等，开发、测试、生产等各环境下均使用不同的账号密码，并定期修改密码。
4. 对于操作系统、中间件、第三方软件产品等，及时安装最新补丁。
5. 数据库、FTP等资源账号密码不得明文存储。

#### 六、防止敏感信息泄露

1. 异常内容不包含敏感信息
	
	在异常处理的时候删除敏感信息，抛给前台的异常信息不能包含：服务器IP地址、端口、文件名、路径等信息。如：
	
	```
	try {
		......
		throw new Exception(“IP地址：192.168.1.1访问不到”);
	}catch(Exception ex) {
		log.error(ex);
		throw new Exception(“IP访问异常”);
	}		
	```
2. 采用异常配置避免向外透露信息
	
	使用error-page减少对外发布异常信息，可在应用web.xml作如下配置。而在error.jsp中只显示ex.getMessage()信息，不输出stacktrace。
	
	```
	<error-page>
  		<exception-type>java.lang.Throwable</exception-type>
  		<location>/error.jsp</location>
	</error-page>
	<error-page>
  		<error-code>500</error-code>
  		<location>/error.jsp</location>
	<error-page>
	```	
	此外，404、403等错误也可按照以上方式统一配置。

3. 防止网站文件被列表展示
	
	如果不做设置，网站内所有的文件可能被列表访问，导致信息泄露。采用tomca部署应用时，应该在conf/web.xml中作如下配置。
	
	```
	<servlet> 
		<servlet-name>default</servlet-name> 
		<servlet-class>org.apache.catalina.servlets.DefaultServlet</servlet-class> 
		<init-param> 
			<param-name>listings</param-name> 
			<param-value>false</param-value> 
		</init-param> 
	</servlet>
	```
	
4.	保护JSP页面，以免被直接访问，绕过请求检查
	
	* 通过web.xml配置保护，在web.xml中作如下配置，防止jsp被直接访问：
	
	```
	<web-app> 
		<security-constraint> 
			<web-resource-collection> 
     			<web-resource-name>no_access</web-resource-name> 
     			<url-pattern>*.jsp</url-pattern>
				</web-resource-collection>
			<auth-constraint/>
		</security-constraint> 
	</web-app>
	```
	* 通过改变存放位置保护JSP，将所有JSP存放在WEB-INF/jsp目录下。
	
5. 日志脱敏

	记录日志时，应避免存入敏感数据，如密码等。

6. 内存中的密码要及时删除

	如果在session对象中保存了用户密码信息，在使用完成后要立即删除密码信息，如果用户密码还需要继续使用，则在内存中加密存放。

7. 模糊化敏感数据展示

	针对某些敏感数据信息需要对数据进行模糊化处理。

8. 其他

	* 对于敏感页面，使用HTTPS方式传输，防止敏感数据被监听窃取。
	* 对于测试环境中的敏感数据，应该尽量清除或作脱敏操作。
	* 确保使用了合适的加密算法和强大的秘钥，并且秘钥管理到位。
	* 提醒用户禁用浏览器的自动完成，防止敏感数据收集。
	* 包含敏感数据的缓存页面禁止缓存。

#### 七、功能级访问控制

攻击者通过修改URL来进行未授权的访问。应用程序需要在每个功能被访问时在服务端执行相同的访问控制检查。

1. 基于白名单策略，除了白名单以外的功能不允许访问，缺省禁止所有权限。
2. 对所有访问控制、授权操作等作日志记录。
3. 每个对功能的访问都应该根据当时的状态进行权限检查，而不仅仅是隐藏功能链接。如WEB页面上的功能通过隐藏按钮来实现对用户的不可见，但在服务端接到请求后，仍应该继续验证权限。
4. 访问受保护的WEB界面需要经过权限检查，没有权限的即不允许查看。
5. 针对系统菜单里的嵌入URL（非系统菜单）访问，也需要经过权限校验，没有权限则不允许查看。

#### 八、跨站访问请求伪造（CSRF）

CSRF通过伪装来自受信任用户的请求来利用受信任的网站。典型攻击方式：让受害者在已经登录某站点的情况下点击某个链接。

1. 对关键性的表单设置临时TOKEN或者验证码、二次验证等手段。
2. 减少会话有效期。
3. referer检查。

#### 九、使用含有已知漏洞的组件

1. 尽量使用已有的功能组件，减少引入新组件的可能。
2. 标识正在使用的所有组件及其版本，并及时了解关注这些组件的安全信息。
3. 在合适的情况下，考虑增加对组件的安全封装，去掉不使用的功能或安全薄弱的或易受攻击的方面。

#### 十、未经验证的重定向和转发

通常重定向所引向的目的是带有用户输入参数的目的URL，而如果这些重定向未被验证，那么攻击者就可以引导用户访问他们所要用户访问的站点。

1. 如果使用了重定向和转发，不要在URL中携带用户参数。
2. 如果使用目标参数无法避免，建议把这种目标的参数做成一个映射值，而不是真的URL或其中的一部分，然后由服务器端代码将映射值转换成目标URL。
3. 可以使用ESAPI重写sendRedirect()方法来确保所有重定向的目的地是安全的。

#### *链接*

[OWASP Top10 (2013)](http://www.owasp.org.cn/owasp-project/download/OWASPTop102013V1.2.pdf)

[OWASP-ESAPI-JAVA](https://code.google.com/p/owasp-esapi-java/)
