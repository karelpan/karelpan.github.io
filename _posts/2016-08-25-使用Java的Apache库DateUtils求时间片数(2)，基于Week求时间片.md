---
layout: post
title: Java使用org.apache.commons.lang3.time.DateUtils求时间片计数(2)，基于Week求时间片
---

 {{ page.title }}
================

<p class="meta">25 August 2016 - Su Zhou</p>

# 得到当天是当前周的第几天（默认使用Sunday为第一天） [1,7]

```java
Date date = new Date();
Calendar cal = Calendar.getInstance();
cal.setTime(date);

int day = cal.get(Calendar.DAY_OF_WEEK); // 得到第几天

```

# 扩展

## 将一天分为上午，下午，得到一周的时间分片Index，范围[0,13]

```java
// 我们接着上面的代码

// get day fragment of one week
long dayFragment = day - 1;

// get hour fragment of one day
long hourFragment = DateUtils.getFragmentInHours(date, Calendar.DATE);
// get am|pm fragment
long amPmFragment = hourFragment < 12 ? 0 : 1;

// 得到基于 Week 的 AM|PM 时间分片Index
long amPmWeekFragment = dayFragment * 2l + amPmFragment; 

```

## 包装成一个方法吧

```java
public static final long getPMAMFragmentFromWeek(Date date) {
    // get hour fragment of one day
    long hourFragment = DateUtils.getFragmentInHours(date, Calendar.DATE);
    // get am|pm fragment of one day
    long amPmFragment = hourFragment < 12 ? 0 : 1;
    // get day fragment of one week
    Calendar cal = Calendar.getInstance();
    cal.setTime(date);

    long dayFragment = cal.get(Calendar.DAY_OF_WEEK) - 1;

    // get AM_PM fragment from a week -> 7 days 14 fragments
    return dayFragment * 2l + amPmFragment; // [0, 13]
}
```
