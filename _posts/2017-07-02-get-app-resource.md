---
layout: post
title: 获取其他应用的资源和函数
date: 2017-07-02
categories: blog
tags: [android,资源]
description: 当前应用获取其他应用的资源和函数的方法

---

### 关于获取其他应用的资源的用途
用途就如其名字一样：获取其他应用的资源。典型的用途如我司的主题，更换某些界面的图片，颜色等。
获取到另一个应用的方法之后可以用反射调用其方法，用途有主包使用主题包的方法完成各个主题不一样的动画等。

### 获取资源的方法
获取的方法有两个，其实也相当于是一个
1. 通过Resource得到想对应的资源
```java
    Resources themeResources = null;
    PackageManager pm = getPackageManager();
    int id = 0;
    themeResources = pm.getResourcesForApplication("com.gomo.calculator");
    id = themeResources.getIdentifier("ic_launcher", "mipmap", "com.gomo.calculator");
    iv.setImageDrawable(themeResources.getDrawable(id));
```
1. 通过Uri获取到资源
```java
    Resources themeResources = null;
    PackageManager pm = getPackageManager();
    int id = 0;
    uri = Uri.parse("android.resource://" + "com.gomo.calculator" + "/" + themeResources.getIdentifier("history_to_calculator", "drawable", "com.gomo.calculator"));
    iv1.setImageURI(uri);
```
核心思路是使用PackageManager提供的方法getResourcesForApplication来获取对应应用的Resources来得到其资源对应的id然后通过这个id获取到具体的资源，方法一个和方法二的区别只是得到id之后获取对应资源的方法不一样而已，所以说这
可以说其实是一种方法。
### 获取资源的demo和源码
先贴图：
![获取其他应用的资源](http://oogbkd3ln.bkt.clouddn.com/resource_demo.jpg)
绿色的圈是源码中的iv，图片是另一个应用的图标；红色的是iv1，图片是一个资源图。

源码：
```java
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Resources themeResources = null;
        PackageManager pm = getPackageManager();
        int id = 0;
        Uri uri = null;
        try {
            themeResources = pm.getResourcesForApplication("com.gomo.calculator");
            id = themeResources.getIdentifier("ic_launcher", "mipmap", "com.gomo.calculator");

            uri = Uri.parse("android.resource://" + "com.gomo.calculator" + "/" + themeResources.getIdentifier("history_to_calculator", "drawable", "com.gomo.calculator"));
        } catch (PackageManager.NameNotFoundException e) {

            e.printStackTrace();
        }
        ImageView iv = (ImageView) findViewById(R.id.iv_resource);
        iv.setImageDrawable(themeResources.getDrawable(id));

        ImageView iv1 = (ImageView) findViewById(R.id.iv_resource1);
        iv1.setImageURI(uri);
    }
```
### 获取其他应用的方法并调用
先看代码吧：
```java
try {
    Context c = createPackageContext("com.gomo.calculator", Context.CONTEXT_INCLUDE_CODE | Context.CONTEXT_IGNORE_SECURITY);

    Class clazz = c.getClassLoader().loadClass("com.jiubang.commerce.utils.ToastUtils");
    Method method = clazz.getMethod("makeEventToast", Context.class, String.class, boolean.class);
    method.invoke(null, c, "textToast", true);
} catch (Exception e) {
    e.printStackTrace();
}
```
核心思路是用到了Context的一个方法createPackageContext来获取对应应用的Context，然后使用反射的方法调用其函数

另一个应用对应的代码：
```java
public class ToastUtils {
    public ToastUtils() {
    }

    public static void makeEventToast(Context context, String text, boolean isLongToast) {
        Toast toast = null;
        if(isLongToast) {
            toast = Toast.makeText(context, text, 1);
        } else {
            toast = Toast.makeText(context, text, 0);
        }

        if(toast != null) {
            toast.show();
        }

    }
}
```
代码很简单，不需要分析。就是弹出一个Toast。结果如预测
