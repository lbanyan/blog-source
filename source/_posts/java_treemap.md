---
title: Java：TreeMap的使用
date: 2017-06-12 20:23
tags:
    - TreeMap
---

### 使用TreeMap的例子
基于key排序：
``` java
Map<Double, String> scoreAnchor = Maps.newTreeMap(new Comparator<Double>() {
    public int compare(Double o1, Double o2) {
        // 降序排序
        return o2.compareTo(o1);
    }
});
```
基于value排序：
``` java
List<Map.Entry<String, Double>> sortGameRatio = new ArrayList<Map.Entry<String, Double>>(context.gameRatio.entrySet());
Collections.sort(sortGameRatio,new Comparator<Map.Entry<String, Double>>() {
    public int compare(Map.Entry<String, Double> o1, Map.Entry<String, Double> o2) {
        return o2.getValue().compareTo(o1.getValue());
    }
});

for (Map.Entry<String, Double> entry : sortGameRatio) {
}
```