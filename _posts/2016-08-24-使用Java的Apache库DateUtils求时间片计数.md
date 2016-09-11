---
layout: post
title: Java使用org.apache.commons.lang3.time.DateUtils求时间片计数
uuid: 20b67400-77b9-11e6-955d-005056359bbd
---

{{ page.title }}
================

<p class="meta">24 August 2016 - Su Zhou</p>

- 求时间片样例，比如：求出一份数据中，一年内小区车辆出入的时间点分布(以小时来制作分布图，以天为分布采样基准)，进和出分别算一次记录

```java
List<String> dateMarkLsit = getDateMarkListFromYear("2015");
int[] hourFlagmentCounts = new int[]{
    0,0,0,0,0,
    0,0,0,0,0,
    0,0,0,0,0,
    0,0,0,0,0,
    0,0,0,0
};
for(String dateStr : dateMarkList) {
    Date date = DateTimeUtil.parseDate(dateStr, "yyyy-MM-dd HH:mm:ss.SSS")
    // range from 0 to 23 -> [0,23]
    long hourFragment = DateUtils.getFragmentInHours(date, Calendar.DATE);
    hourFlagmentCounts[hourFragment]++;
}

// 将数据进行渲染
renderToChartUI(hourFlagmentCounts);
```
<p>注意： DateUtils 并不支持所有的Calendar-Fragemnt， 想知道支持的信息， 可查看对应版本的 DateUtils
源码中方法getFragment(Calendar calendar, int fragment, int unit)的实现，可和Java核心库的Calendar类对比查看。</p>
        
