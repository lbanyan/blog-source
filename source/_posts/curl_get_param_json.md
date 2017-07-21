---
title: curl GET参数中使用json
tags:
    - curl
    - GET
    - json
---

## 举例
``` bash
curl -H "X-Auth-Token:ac477518fc0d4f7da33ca9740211bec8" -X GET https://cloudgw.yyclouds.com/api/rule/list?search={"gid":"LCTQ"}
```

## 解析
上述方式错误，服务端无法成功解析search，当然GET请求参数中使用json也是很不合标准的。

正确的请求方式：
``` bash
curl -H "X-Auth-Token:ac477518fc0d4f7da33ca9740211bec8"  -X GET https://cloudgw.yyclouds.com/api/rule/list?search=%7B%22gid%22%3A%22TEST%22%7D
```

即将json参数进行<font color='red'>URI编码</font>，可以使用在线工具：http://tool.oschina.net/encode?type=4 来编码。

如果使用Postman工具，则可无需编码，直接访问即可。
