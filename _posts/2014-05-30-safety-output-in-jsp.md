---
layout: post
title: 在JSP里使用更安全的数据输出方式
---

###在JSP里使用更安全的数据输出方式

因为JSP最终也是被编译成class运行，所以可以在JSP内通过<%%>方式直接调用Java代码。通常有以下几种场景：

1. 参数验证或业务逻辑等纯Java写法。
2. 通过<%=xxx%>直接作为HTML内容输出，如：

	```
	<%
	String message = request.getParameter("message");
	%>
	......
	<div id="msg"><%=message %></div>
	```
3. 通过<%=xxx%>将变量值赋值给Javascript对象，如：
	
	```
	<%
	String message = request.getParameter("message");
	%>
	......
	<script type="text/javascript">
	var t1 = "<%=message%>";
	document.getElementById("t1input").value = t1;
	</script>
	```
这种写法比较简单，所以在很多工程中用的比较多，抛去前后台逻辑拆分及代码可读性可维护性问题不谈。尤其后两种写法很容易引起XSS攻击。比如message参数被伪造为：
	
	```
	String message = "<script>alert();</script>";
	```
在第二种场景下，加载出的数据直接作为HTML脚本内容输出。如上脚本，浏览器会直接解释执行，从而形成XSS漏洞。

	```
	String message ="\";alert('hacked');\"";
	```
在第三种场景下，数据被赋值给js变量t1，赋值过程在script片段中进行。而如上脚本所示，t1变量赋值的过程被Hacked，而直接执行alert脚本，同样形成XSS漏洞。

Fortify对此有如下说明：

> Cross-Site Scripting (XSS) 漏洞在以下情况下发生：
> 
> 1. 数据通过一个不可信赖的数据源进入 Web 应用程序。对于 Reflected XSS，不可信赖的源通常为 Web 请求，而对于 Persisted（也称为 Stored）XSS，该源通常为数据库或其他后端数据存储。
> 在这种情况下，数据经由 index.jsp 的第 470 行进入 getParameter()。 
> 2. 未检验包含在动态内容中的数据，便将其传送给了 Web 用户。 
> 在这种情况下，数据通过 index.jsp 的第 484 行中的 print() 传送。
> 传送到 Web 浏览器的恶意内容通常采用 JavaScript 代码片段的形式，但也可能会包含一些 HTML、Flash 或者其他任意一种可以被浏览器执行的代码。基于 XSS 的攻击手段花样百出，几乎是无穷无尽的，但通常它们都会包含传输给攻击者的私人数据（如 Cookie 或者其他会话信息）。在攻击者的控制下，指引受害者进入恶意的网络内容；或者利用易受攻击的站点，对用户的机器进行其他恶意操作。

针对这种情况，相对安全的做法有如下方式：

1. 通过ajax方式与后端交互数据，ajax调用返回的数据将被认为是文本格式。
2. 针对数据直接输出到HTML的场景，可以通过JSTL标签方式（不支持对JS变量的直接赋值）。如：
	
	```
	<div id="msg"><c:out value="<%=message %>"/></div>
	```
3. 针对数据直接赋值给JS变量的场景，对输入数据作验证并encoding，如ESAPI中提供的encodingForJavascript可将需要赋值给JS变量的数据encoding为16进制格式数据（不支持在HTML中直接输出）。
	
	```
	<%
	String message = "\";alert('hacked');\"";
	message = ESAPI.encoder().encodeForJavaScript(message);
	%>
	......
	<script type="text/javascript">
	var t1 = "<%=message%>";// \x22\x3Balert\x28\x27hacked\x27\x29\x3B\x22
	document.getElementById("t1input").value = t1;
	</script>
	```
