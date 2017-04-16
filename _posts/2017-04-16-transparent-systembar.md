---
layout: post
title: 简单了解android设置透明状态栏
date: 2017-04-16
categories: blog
tags: [android,状态栏]
description: 使用模板改的第一篇文章

---
说在前面:android允许设置透明状态栏是在android4.4以后,所以这里主要讨论的是在android4.4以后设置透明状态栏以后的布局与android4.4之前手机上的布局的兼容问题.

首先看下如何设置状态栏透明:

在AndroidMainfest.xml中的Application里设置一个主题例如

``` java
android:theme="@style/AppTheme"

```
(如果不是想给整个app都设置这个属性也可以给某个Activity设置)

然后分版本给这个style设置属性:

建立一个value-v19的文件夹,然后在这个文件夹里建立一个style.xml文件,再到这个xml里设置一个属性,例如
``` java
<style name="AppTheme" parent="@android:style/Theme.Holo.Light.NoActionBar">
 <item name="android:windowTranslucentStatus">true</item>
</style>
```
而value文件夹里的style.xml则不设置这个属性,因为4.4以下是不支持设置透明状态栏的

看下效果:

![布局到状态栏下](http://oogbkd3ln.bkt.clouddn.com/trans_systembartrans_bar.png)

PS:是不是觉得透明状态栏怎么不透明?

(何止不透明?android4.4还是从上到下由黑色到透明的渐变.PS:一些ROM里5.0上也是这个效果,例如sony)

原因是material design里就是这样设计的,状态栏的颜色比ActionBar的颜色深一点.个人猜测原因是android没有像ios那样给状态栏上字体个图标变黑色的机制有关

但是你非要设置成完全透明也是有办法的,但是只能在android5.0以上才会生效,4.4不会生效.
``` java
getWindow().getDecorView().setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_STABLE

                | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION

                | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN

                | 0x00000800);

                getWindow().addFlags(0x04000000); //透明的状态栏

                getWindow().addFlags(0x08000000); //透明的虚拟栏
```
回到主题:

这里就看到了这个在android4.4以上的手机上的状态栏是透明的,但是问题就来了,布局到状态栏下面了,所以在android4.4以上得给他设置一个状态栏高度的pading

因为在xml里无法直接获取到状态栏高度,一个方法是通过java代码通过反射获取到状态栏高度,这显然不是一个好的办法.而且通过这个办法有可能因为极个别厂商rom的设定得到的值不准

所以google设置了一个属性
``` java
android:fitsSystemWindows="true"
```
只需要在布局文件里设置这个属性就可以自动适配是否设置了透明状态栏,如图:
![自动适应状态栏](http://oogbkd3ln.bkt.clouddn.com/trans_systembartrans_bar_fit.png)

似乎得到了理想的效果,操作简单而且兼容低版本,但是这个属性的的局限性太多了:

下面一点一点罗列发现的局限性,首先是google官方对这个属性的解释:

Boolean internal attribute to adjust view layout based on system windows such as the status bar. If true, adjusts the padding of this view to leave space for the system windows. Will only take effect if this view is in a non-embedded activity.

明确说明的这个属性只在non-embedded activity里起作用.

这个non-embedded activity从字面意思来看是 非嵌入式Activity,但是何谓非嵌入式Activity呢?

An embedded Activity is an Activity which is hosted inside a parent Activity. The common example being the TabHost/TabActivity design. In particular, embedded Acitvities reside in the host's LocalActivityManager, which is conceptually similar to the FragmentManager which allows you to display one Activity inside another.


这里说像TabHost/TabActivity 这样的组件里不会起作用.似乎现在的应用里不会用到tabhost这样过时的组件,影响不大,实则不然.

我这里实验过系统提供的ActionBar是会自动的排列在状态栏的下面,而且设置这个属性的view回自动布局在actionbar的下面:

![自动适应actionbar](http://oogbkd3ln.bkt.clouddn.com/trans_systembartrans_actionbar.png)
效果还不错.

但是在Toolbar的显示很诡异

会布局到状态栏下面,使用这个属性之后文字也会居中,但是Toolbar的高度不会增加,如图(我没有将hello world 布局到toolbar的下面

![不适应toolbar](http://oogbkd3ln.bkt.clouddn.com/trans_systembartrans_tools_bar.png)

造成这个的原因不是这个属性对ToolBar不起作用,只是ToolBar有些特别,ToolBar有一个属性

android:minHeight="?attr/actionBarSize" 然后再设置 android:layout_height="wrap_content"

不能直接设置 android:layout_height="?attr/actionBarSize" 不然就会出现上图的那样的情况

在fragment里同样起作用,似乎完美,别高兴得太早,下面开始讲这个属性的局限性.

首先,这个属性在AppCompatActivity里是不起作用的,使用了AppCompatActivity之后状态栏就不会变透明,即使不改变状态栏的颜色,状态栏也不会变透明.

然后是,用了这个属性以后,这个view的pading就会不起作用:

![不能使用padding](http://oogbkd3ln.bkt.clouddn.com/trans_systembartrans_setPadding.png)

顺便提及一下,在style里设置虚键件透明以后,这个属性会在下面也设置一个虚拟键高度的padding:

![不能使用padding](http://oogbkd3ln.bkt.clouddn.com/trans_systembartrans_no_action_bar_fits_system_window.png)

似乎这个属性是给布满整个屏幕的view用的,还是似乎完美,但是转折点就在这里:

这个属性在整个Activity里只会在一个地方生效,什么意思呢?就像下面这个:
![只能生效一次](http://oogbkd3ln.bkt.clouddn.com/trans_systembartrans_screenshot.png)

这里完美是自己做一个类似ActionBar的View出来,你把这个属性设到这个假ActionBar上以后,后面那个有水波纹的view再设置就不会生效了.

这个属性生效的顺序是在xml布局里从上到下第一个设置这个属性的地方,包括设置一个framLayout以后再在Activity里将这个framLayout替换成你想要的fragment,就像这样:

``` xml
<FrameLayout
 android:id="@+id/container"
 android:layout_width="match_parent"
 android:layout_height="300dp"/>

<fragment
 android:layout_width="match_parent"
 android:layout_height="300dp"
 android:layout_below="@id/container"
 android:fitsSystemWindows="true"
 class="com.example.caijun.statusbarapplication.MyFragment" />
 ```
 在Activity里将这个FramLayout通过FragmentTransaction将他替换成

com.example.caijun.statusbarapplication.MyFragment而这个Fragment的布局里也设置了这个属性,那生效的也是替换掉的那个Fragment





PPPPPPPS:在java代码里获取状态栏的高度:

``` java
Class<?> c = null;
Object obj = null;
Field field = null;
int x = 0;
try {
    c = Class.forName("com.android.internal.R$dimen");
    obj = c.newInstance();
    field = c.getField("status_bar_height");
    x = Integer.parseInt(field.get(obj).toString());
    sStatusBarHeight = context.getResources().getDimensionPixelSize(x);
  } catch (Exception e1) {
    e1.printStackTrace();
}
```
