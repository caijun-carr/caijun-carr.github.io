---
layout: post
title: 三星手机使用系统闹钟类时爆出Too many alarms的问题解决
date: 2017-09-20
categories: blog
tags: [android,三星,alarm]
description: 三星手机全系Alarm类出现的坑

---
### 三星手机上常出现的坑
在三星手机中使用系统类Alarm时时常会爆出Too many alarms (500) registered的错误。错误日志看起来也无从查起。在其他手机上也不会出现类似的错误。

### 解锁方法
经测试，在三星手机中使用了FLAG_CANCEL_CURRENT配上ELAPSED_REALTIME或ELAPSED_REALTIME_WAKEUP组合后，
手机就会爆Too many alarms (500) registered，其它手机不会爆。三星手机的其它组合的也不会爆。

因此避免在三星手机上使用上述组合即可避免此问题。
