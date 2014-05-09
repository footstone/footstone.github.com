---
layout: post
title: 诡异问题记录之ActiveXObject(“Microsoft.XMLHTTP”)获取后端中文报文乱码
---

###诡异问题记录之ActiveXObject(“Microsoft.XMLHTTP”)获取后端中文报文乱码


开发平台中的前端公用组件，获取xmlHttpRequest对象时通过代码片断1获取，该代码在各种测试环境下都没有出现问题，包括IE6\IE7\IE8\IE9\360安全浏览器，但在部分客户终端上（WinXP+IE7.0573..此环境下也只是部分终端存在问题）版本上获取后端数据时，如果后端返回报文中有中文，则中文全部乱码，导致后续报文解析出错。统一使用代码片断2获取时，则可以解决该问题。具体原因不详。

* 片断1：

```
function g_GetXMLHTTPRequest() {
  var xRequest=null;
  if (typeof ActiveXObject != "undefined"){
    //Internet Explorer
    try{
      xRequest = new ActiveXObject("Msxml2.XMLHTTP.4.0");
    }catch(e1){
    try{
      xRequest = new ActiveXObject("Msxml2.XMLHTTP");
    } catch(e2){
    xRequest = new ActiveXObject("Microsoft.XMLHTTP");
  }
}
```

* 片断2：

```
function g_GetXMLHTTPRequest() {
  var xRequest=null;
  if (typeof ActiveXObject != "undefined"){
    //Internet Explorer
    xRequest=new ActiveXObject("Microsoft.XMLHTTP");
  }
}
```
