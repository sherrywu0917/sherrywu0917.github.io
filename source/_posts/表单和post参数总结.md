---
title: 表单和post参数总结
date: 2019-11-26 16:50:01
tags:
---

post请求下的content-type类型:
### application/json
```
Request Payload:
{layouts: ["1", "2", "5"]}
```

### application/x-www-form-urlencoded
```
Query String Parameters
sourceUuid:np_c3sEKMgU3Xau_Tl0fmsqyReIdKH9OXQpby_FSNhz__ppcy0EKQ
articleUuid:np_eXoyZJwfsX-q8j5OLm0pyEbYGK0
forceFollow:0
isContinue:0
forceAutoPurchase:1
```

### multipart/form-data
使用表单上传文件时，必须让 form 的 enctype 等于这个值。
```
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryBJIFBCE2CA2Fwbyl
```

### text/xml
例如XML-RPC请求。
``` xml
<?xml version="1.0"?>
<methodCall>
    <methodName>examples.getStateName</methodName>
    <params>
        <param>
            <value><i4>41</i4></value>
        </param>
    </params>
</methodCall>
```

get请求的content-type只有application/x-www-form-urlencoded这一种。


form表单中action带有参数
1. get表单请求
``` html
<form method="get" action="/somePage.html?param1=foo&param2=foo">
  <input name="param2">bar</input>
  <input name="param3">bar</input>
</form>
```
最终得到的参数是：
Query String Parameters
param2: bar
param3: bar
get请求的表单在拼接参数的时候，会丢掉action指定的URL中带上的参数。

2. post表单请求：
enctype属性默认值是'application/x-www-form-urlencoded'，还支持'multipart/form-data'和'text/plain'
#### application/x-www-form-urlencoded
``` html
<form method="post" action="/somePage.html?param1=foo&param2=foo">
    <input name="param2" value="bar"/>
    <input name="param3" value="baz"/>
</form>
```
最终得到的参数是：
Query String Parameters
param1: foo
param2: foo

Form Data
param2: bar
param3: baz

#### text/plain
``` html
<form method="post" action="/somePage.html?param1=foo&param2=foo" enctype="text/plain">
    <input name="param2" value="bar"/>
    <input name="param3" value="baz"/>
</form>
```
最终得到的参数是：
Query String Parameters
param1: foo
param2: foo

Request Payload
param2=bar
param3=baz


### fetch的get、post请求参数
- get，拼接到url上
- post，参数放到body中`{body: JSON.stringify({p1: 1})}`
