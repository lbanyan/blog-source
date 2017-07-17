---
title: JAVA：包含有某key的Map无法得到该key对应的value值
---

## 代码示例
``` java
Map<String, String> map = Maps.newHashMap();
map.put("107693789", "s1");
map.put("500140553", "1");
map.put("938098493", "s1");
String value = map.get(938098493);
System.out.println(value);
```

## 问题分析
容易发现，是由于map.get时，误传入了long类型的值。
此时编译期未报错的原因是map.get接收的是Object类型的参数。
