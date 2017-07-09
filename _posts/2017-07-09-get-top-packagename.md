---
layout: post
title: 获取activity栈顶信息
date: 2017-07-09
categories: blog
tags: [android,栈顶]
description: 获取到手机上当前的栈顶的信息

---
### activity栈顶是什么？
简单的讲就是用户手机当前最上面的activity。如果没有其他视图遮挡就是当前可见的activity

### 获取的方法分为两种
由于Android对权限的控制越来越严格，获取栈顶的方法大致可以分为两种，一种是5.0以前直接获取。还有一种是5.0以后需要申请android.permission.PACKAGE_USAGE_STATS这个权限，这个权限需要引导用户去设置里手动开启，不是只在manifest里申请就有的。

### Android5.0以前获取的方法
国际惯例，先贴代码：
```java
private ComponentName getTopApp(Context context) {
    ActivityManager activityManager = (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
    List<ActivityManager.RunningTaskInfo> appTasks = activityManager.getRunningTasks(1);
    if (null != appTasks && !appTasks.isEmpty()) {
        return appTasks.get(0).topActivity;
    }
    return null;
}
```
代码很简单，在5.0以下也很好用，但是5.0以上就不好用了。

### Android5.0以上的方法
上面说到了，5.0以上想和5.0以前一样获取栈顶必须要申请PACKAGE_USAGE_STATS这个权限

先看如何申请权限：
1. 首先AndroidManifest同样需要申请：
```java
 < uses-permission android:name="android.permission.PACKAGE_USAGE_STATS" / >
```
1. 检测是否已经有权限：
```java
UsageStatsManager usm = (UsageStatsManager) getSystemService(Context.USAGE_STATS_SERVICE);
Calendar calendar = Calendar.getInstance();
long endTime = calendar.getTimeInMillis();
calendar.add(Calendar.YEAR, -1);
long startTime = calendar.getTimeInMillis();
List<UsageStats> usageStatsList = usm.queryUsageStats(UsageStatsManager.INTERVAL_DAILY, startTime, endTime);
// usageStatsList返回的size不为0则认为有权限，为0则认为没有权限
```
1. 申请权限跳转到设置的代码：
```java
Intent intent = new Intent(Settings.ACTION_USAGE_ACCESS_SETTINGS);
context.startActivity(intent);
```

获取栈顶的代码：
```java
private ComponentName getTopApp(Context context) {
    try {
        String packageName = "";
        String className = "";
        long endTime = System.currentTimeMillis();
        long beginTime = endTime - 10000;
        UsageStatsManager sUsageStatsManager = (UsageStatsManager) context.getSystemService(Context.USAGE_STATS_SERVICE);
        UsageEvents.Event event = new UsageEvents.Event();
        UsageEvents usageEvents = sUsageStatsManager.queryEvents(beginTime, endTime);
        while (usageEvents.hasNextEvent()) {
            usageEvents.getNextEvent(event);
            if (event.getEventType() == UsageEvents.Event.MOVE_TO_FOREGROUND) {
                packageName = event.getPackageName();
                className = event.getClassName();
            }
        }

        if (!android.text.TextUtils.isEmpty(packageName)) {
            return new ComponentName(packageName, className);
        }
    } catch (Throwable e) {
        e.printStackTrace();
    }
    return null;
}
```
