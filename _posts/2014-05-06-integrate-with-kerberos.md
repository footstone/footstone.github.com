---
layout:post
title:基于Kerberos协议与Active Directory集成的统一认证解决方案
---
**基于Kerberos协议与Active Directory集成的统一认证解决方案
**

`Kerberos`
`Active Directory`
`jaas`


 1. 用户登录AD域之后，通过IE访问Veris（如CRM\Billing等），Veris识别到当前用户没有会话信息，将用户redirect到SSO Server。
  2. SSO Server使用 HTTP 提示响应（HTTP 401）进行响应，其中包含“Authenticate: Negotiate”头。
  3. 客户端（用户浏览器）识别 Negotiate 头（需要IE配置为支持“集成Windows认证”）。cd
  4. 客户端（用户浏览器）解析所请求的 URL，以获得主机名。
  5. 客户端（用户浏览器）从Kerberos身份验证服务（Authentication Service）请求票据（TGT）。
  6. 客户端（用户浏览器）从 SPN的Kerberos 票证授予服务（Ticket Granting Service，TGS）请求服务票证，例如HTTP/sso.ailk.telenor@TELENOR.COM。客户端（用户浏览器）解析所请求的URL获得协议和主机名，并使用客户端自己的域名作为 Kerberos 安全领域，从而构造 SPN： <protocol>/<host's FQDN>@<Client's Kerberos Security Realm> 注意：必须在配置环境时向 Kerberos 密钥分发中心（Key Distribution Center，KDC）注册 SPN。