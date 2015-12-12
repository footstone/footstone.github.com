---
layout: post
title: Tomcat的两个编码参数
---

针对在URL中携带中文或其他ISO8859-1不能识别的字符的场景，`useBodyEncodingForURI=“true”`优先级高于URIEncoding。

当server.xml中同时配置`useBodyEncodingForURI`和`URIEncoding`时，Tomcat优先识别`useBodyEncodingForURI`

1. request中获取的String参数仍然是以ISO8859-1编码，中文或丹麦文的参数，servlet中获取参数值时需做转码： `new String(value.getBytes("ISO8859-1"),"UTF-8")`
2. `request.setCharacterEncoding(“UTF-8”)`可以使参数值自动做UTF-8转码，通过request获取时，不需要再做手工转码，但必须在当前请求过程中第一次`request.getParameter`执行之前调用才有效。

server.xml中只配置`URIEncoding=“UTF-8”`

1. request获取的String参数即是以UTF-8编码，不需要做任何转码处理
2. `request.setCharacterEncoding`，对request参数值无任何作用。

--EOF--








