title: Android桌面快捷方式那些事与那些坑  
layout: post  
date: 2015-03-14 16:22:45
categories: Android
tags: [Android,shortcut,快捷方式]
---

将近二个多月没写博客了。  
之前一段时间一直在搞红包助手，就没抽时间写博客，但写这个真的是很好玩。没想到居然在Android上实现模拟点击，从而实现自动抢红包，有兴趣的同学可以参考https://github.com/waylife/RedEnvelopeAssistant ,代码已经开源。
红包助手还有一些问题，但是现在基本的抢红包基本没问题了。目前正在对它进行优化以及较低版本的一些适配，还有项目的国际化工作。
废话不多说了，下面是Andrioid开发过程中快捷方式相关的事与坑。
数据来源于上次组内自己的CodeReview总结。
#背景
一般情况下，为了让用户更方便的打开应用，程序会在桌面上生成一些快捷方式。
本来呢，如果是原生的桌面，其实是十分简单，直接调用系统相关的API就行了。但是众多的系统厂商以及众多第三方自己定制的桌面（Launcher），导致在适配、兼容方面存在很多问题。  
比如，有些桌面无法删除快捷方式（比如小米），有些桌面无法生成快捷方式（比如锤子），有些系统无法更新桌面图标（比如华为荣耀6）。  
在升级、降级的时候快捷方式发生变化；比如，全部变成应用的主图标，升级、降级后点击快捷方式没有反应，删除应用后无法删除快捷方式。  
很多问题都是需要解决的，虽然有些由于系统限制，没有办法搞定所有的，但是仍然需要寻求一个最优的方案。这也就是本文需要讨论的问题。
本文说指的快捷方式是指应用桌面快捷方式，不包含长按弹出的生成快捷方式。  
快捷方式所有信息都是存在于launcher的favorite表。一般需要用到的字段为_id,title,intent,iconResource,icon，分别表示 快捷方式名称，快捷方式intent，快捷方式图标（本地），快捷方式图标（data二进制压缩数据）。
两个intent数据如下
![ ](https://github.com/waylife/blogimage/raw/master/2015/03/14/shorcut_intent.png)
数据可以通过SQLite Editor查看，需要已经ROOT的手机  
#实现
##增加快捷方式
在AndroidManifest.xml增加权限
``` bash
<uses-permission android:name="com.android.launcher.permission.INSTALL_SHORTCUT" />
```
同时，根据Intent是隐式还是显示在相关的Activity声明相关的intent-filter。
相关代码：  
![ ](https://github.com/waylife/blogimage/raw/master/2015/03/14/shortcut_add.jpg)
![ ](https://github.com/waylife/blogimage/raw/master/2015/03/14/shortcut_add_2.jpg)

##删除快捷方式
跟增加快捷方式一样，也是需要增加权限的。加上
``` bash
<uses-permission android:name="com.android.launcher.permission.UNINSTALL_SHORTCUT" />
```
相关代码：
![ ](https://github.com/waylife/blogimage/raw/master/2015/03/14/shortcut_delete.jpg)

##快捷方式修改
需要增加权限
``` bash
    <uses-permission android:name="com.android.launcher.permission.READ_SETTINGS" />
    <uses-permission android:name="com.android.launcher.permission.WRITE_SETTINGS" />
```
如果适配所有桌面，请添加附录中第二条所列出的权限。  
系统并没有提供API去更改桌面快捷方式。只能通过其他猥琐的办法了，可行的的办法之一就是通过ContentProvider去更改数据库相关的信息。当然有人会说了，先删掉快捷方式，再重新创建不就行了？这是个办法。但是有些系统是无法删除快捷方式的；另外，删除快捷方式与创建快捷方式都是通过广播实现的，这个地方需要控制两者的时间间隔。权衡之后，选用第一种办法相对稳妥。  
废话不多少，上代码。  
``` java
/**
  * 更新桌面快捷方式图标，不一定所有图标都有效<br/>
  * 如果快捷方式不存在，则不更新<br/>.
  */
 public static void updateShortcutIcon(Context context, String title, Intent intent,Bitmap bitmap) {
  if(bitmap==null){
   XLog.i(TAG, "update shortcut icon,bitmap empty");
   return;
  }
  try{
   final ContentResolver cr = context.getContentResolver();
   StringBuilder uriStr = new StringBuilder();
   String urlTemp="";
   String authority = LauncherUtil.getAuthorityFromPermissionDefault(context);
   if(authority==null||authority.trim().equals("")){
    authority = LauncherUtil.getAuthorityFromPermission(context,LauncherUtil.getCurrentLauncherPackageName(context)+".permission.READ_SETTINGS");
   }
   uriStr.append("content://");
   if (TextUtils.isEmpty(authority)) {
    int sdkInt = android.os.Build.VERSION.SDK_INT;
    if (sdkInt < 8) { // Android 2.1.x(API 7)以及以下的
     uriStr.append("com.android.launcher.settings");
    } else if (sdkInt < 19) {// Android 4.4以下
     uriStr.append("com.android.launcher2.settings");
    } else {// 4.4以及以上
     uriStr.append("com.android.launcher3.settings");
    }
   } else {
    uriStr.append(authority);
   }
   urlTemp=uriStr.toString();
   uriStr.append("/favorites?notify=true");
   Uri uri = Uri.parse(uriStr.toString());
   Cursor c = cr.query(uri, new String[] {"_id", "title", "intent" },
     "title=?  and intent=? ",
     new String[] { title, intent.toUri(0) }, null);
   int index=-1;
   if (c != null && c.getCount() > 0) {
    c.moveToFirst();
    index=c.getInt(0);//获得图标索引
    ContentValues cv=new ContentValues();
    cv.put("icon", flattenBitmap(bitmap));
    Uri uri2=Uri.parse(urlTemp+"/favorites/"+index+"?notify=true");
    int i=context.getContentResolver().update(uri2, cv, null,null);
    context.getContentResolver().notifyChange(uri,null);//此处不能用uri2，是个坑
    XLog.i(TAG, "update ok: affected "+i+" rows,index is"+index);
   }else{
    XLog.i(TAG, "update result failed");
   }
   if (c != null && !c.isClosed()) {
    c.close();
   }
  }catch(Exception ex){
   ex.printStackTrace();
   XLog.i(TAG, "update shortcut icon,get errors:"+ex.getMessage());
  }
 }


 private static byte[] flattenBitmap(Bitmap bitmap) {
  // Try go guesstimate how much space the icon will take when serialized
  // to avoid unnecessary allocations/copies during the write.
  int size = bitmap.getWidth() * bitmap.getHeight() * 4;
  ByteArrayOutputStream out = new ByteArrayOutputStream(size);
  try {
   bitmap.compress(Bitmap.CompressFormat.PNG, 100, out);
   out.flush();
   out.close();
   return out.toByteArray();
  } catch (IOException e) {
   XLog.w(TAG, "Could not write icon");
   return null;
  }
 }
```

##快捷方式存在判断
需要增加的权限同修改快捷方式  
虽然说通过SharePreference来保证快捷方式不会重复创建，以及通过shortcutIntent.putExtra("duplicate", false)也可以确保，但是为了万无一失，还是可以通过去查询数据判断快捷方式是否存在，来避免重复创建。   代码如下：
``` java
/**
  * 检查快捷方式是否存在 <br/>
  * <font color=red>注意：</font> 有些手机无法判断是否已经创建过快捷方式<br/>
  * 因此，在创建快捷方式时，请添加<br/>
  * shortcutIntent.putExtra("duplicate", false);// 不允许重复创建<br/>
  * 最好使用{@link #isShortCutExist(Context, String, Intent)}
  * 进行判断，因为可能有些应用生成的快捷方式名称是一样的的<br/>
  * 此处需要在AndroidManifest.xml中配置相关的桌面权限信息<br/>
  * 错误信息已捕获<br/>
  */
 public static boolean isShortCutExist(Context context, String title) {
  boolean result = false;
  try {
   final ContentResolver cr = context.getContentResolver();
   StringBuilder uriStr = new StringBuilder();
   String authority = LauncherUtil.getAuthorityFromPermissionDefault(context);
   if(authority==null||authority.trim().equals("")){
    authority = LauncherUtil.getAuthorityFromPermission(context,LauncherUtil.getCurrentLauncherPackageName(context)+".permission.READ_SETTINGS");
   }
   uriStr.append("content://");
   if (TextUtils.isEmpty(authority)) {
    int sdkInt = android.os.Build.VERSION.SDK_INT;
    if (sdkInt < 8) { // Android 2.1.x(API 7)以及以下的
     uriStr.append("com.android.launcher.settings");
    } else if (sdkInt < 19) {// Android 4.4以下
     uriStr.append("com.android.launcher2.settings");
    } else {// 4.4以及以上
     uriStr.append("com.android.launcher3.settings");
    }
   } else {
    uriStr.append(authority);
   }
   uriStr.append("/favorites?notify=true");
   Uri uri = Uri.parse(uriStr.toString());
   Cursor c = cr.query(uri, new String[] { "title" },
     "title=? ",
     new String[] { title }, null);
   if (c != null && c.getCount() > 0) {
    result = true;
   }
   if (c != null && !c.isClosed()) {
    c.close();
   }
  } catch (Exception e) {
   e.printStackTrace();
   result=false;
  }
  return result;
 }

 /**
  * 不一定所有的手机都有效，因为国内大部分手机的桌面不是系统原生的<br/>
  * 更多请参考{@link #isShortCutExist(Context, String)}<br/>
  * 桌面有两种，系统桌面(ROM自带)与第三方桌面，一般只考虑系统自带<br/>
  * 第三方桌面如果没有实现系统响应的方法是无法判断的，比如GO桌面<br/>
  * 此处需要在AndroidManifest.xml中配置相关的桌面权限信息<br/>
  * 错误信息已捕获<br/>
  */
 public static boolean isShortCutExist(Context context, String title, Intent intent) {
  boolean result = false;
  try{
   final ContentResolver cr = context.getContentResolver();
   StringBuilder uriStr = new StringBuilder();
   String authority = LauncherUtil.getAuthorityFromPermissionDefault(context);
   if(authority==null||authority.trim().equals("")){
    authority = LauncherUtil.getAuthorityFromPermission(context,LauncherUtil.getCurrentLauncherPackageName(context)+".permission.READ_SETTINGS");
   }
   uriStr.append("content://");
   if (TextUtils.isEmpty(authority)) {
    int sdkInt = android.os.Build.VERSION.SDK_INT;
    if (sdkInt < 8) { // Android 2.1.x(API 7)以及以下的
     uriStr.append("com.android.launcher.settings");
    } else if (sdkInt < 19) {// Android 4.4以下
     uriStr.append("com.android.launcher2.settings");
    } else {// 4.4以及以上
     uriStr.append("com.android.launcher3.settings");
    }
   } else {
    uriStr.append(authority);
   }
   uriStr.append("/favorites?notify=true");
   Uri uri = Uri.parse(uriStr.toString());
   Cursor c = cr.query(uri, new String[] { "title", "intent" },
     "title=?  and intent=?",
     new String[] { title, intent.toUri(0) }, null);
   if (c != null && c.getCount() > 0) {
    result = true;
   }
   if (c != null && !c.isClosed()) {
    c.close();
   }
  }catch(Exception ex){
   result=false;
   ex.printStackTrace();
  }
  return result;
 }
```
#兼容与注意事项
##兼容
所有的快捷方式Intent如果不是之前版本的存在很大问题，绝对不要改变参数，否则升级或者降级时快捷方式会出现问题；
同时，尽可能的采用隐式调用，自定义CATEGORY，而不是自定义ACTION，ACTION参数一定要为ACTION_MAIN，否则有些手机在卸载时无法删除快捷方式（WTF）。
##注意事项
- 【所有】activity路径的变更导致老版本升级之后快捷方式无法使用
--> 1.一旦使用确定了activity的包路径，之后就不要再变更；  
  --> 2.尽可能使用隐式调用，但是如果之前已经发出去的版本，为了兼容性，就必须一直使用老的方式，新版本的尽可能的不要更改方式，如果用户降级，老版本快捷方式会无法使用。  
- 【部分】多个快捷方式指向一个activity导致部分手机（三星SII）升级时图标变成应用图标  
--> 尽可能的避免多个快捷方式指向同一个activity，可能通过多个activity再跳转过去  
- 【部分】应用删除时无法删除快捷方式。与系统桌面Launcher实现有关。  
--> 为了适配所有Launcher，Intent Action使用Intent.ACTION_MAIN。如果是隐式调用，尽可能自定义CATEGORY，而不是自定义ACTION。  
- 【部分】应用升级时需要删除老版本部分快捷方式，但是部分手机无法删除  
--> 无解  
- 【部分】第三方桌面无法生成、删除、更新快捷方式  
--> 呵呵，一般来说生成没有问题，但是删除，更新大部分桌面会有问题。尽可能避免这些操作。或者专门适配该桌面，成本较高。  
- 【部分】部分桌面无法实时更新图标，需要重启  
--> 无解，尝试过重启Launcher，但是结果是之前的快捷方式也消失了。只有重启手机，按理来说应该是有方式触发Launcher进行刷新的。
以上【所有】【部分】，分别表示必定出现，部分出现。  

  
#参考
1.  http://grepcode.com/search/?query=InstallShortcutReceiver
2.  http://developer.android.com/index.html


#附录
1. 完整代码可参考
https://gist.github.com/waylife/437a3d98a84f245b9582
2. 通用更新快捷方式权限列表
``` bash
<uses-permission android:name="com.android.launcher.permission.READ_SETTINGS" />
    <uses-permission android:name="com.android.launcher.permission.WRITE_SETTINGS" />
    <uses-permission android:name="com.android.launcher2.permission.READ_SETTINGS" />
    <uses-permission android:name="com.android.launcher2.permission.WRITE_SETTINGS" />
    <uses-permission android:name="com.android.launcher3.permission.READ_SETTINGS" />
    <uses-permission android:name="com.android.launcher3.permission.WRITE_SETTINGS" />
    <uses-permission android:name="org.adw.launcher.permission.READ_SETTINGS" />
    <uses-permission android:name="org.adw.launcher.permission.WRITE_SETTINGS" />
    <uses-permission android:name="com.htc.launcher.permission.READ_SETTINGS" />
    <uses-permission android:name="com.htc.launcher.permission.WRITE_SETTINGS" />
    <uses-permission android:name="com.qihoo360.launcher.permission.READ_SETTINGS" />
    <uses-permission android:name="com.qihoo360.launcher.permission.WRITE_SETTINGS" />
    <uses-permission android:name="com.lge.launcher.permission.READ_SETTINGS" />
    <uses-permission android:name="com.lge.launcher.permission.WRITE_SETTINGS" />
    <uses-permission android:name="net.qihoo.launcher.permission.READ_SETTINGS" />
    <uses-permission android:name="net.qihoo.launcher.permission.WRITE_SETTINGS" />
    <uses-permission android:name="org.adwfreak.launcher.permission.READ_SETTINGS" />
    <uses-permission android:name="org.adwfreak.launcher.permission.WRITE_SETTINGS" />
    <uses-permission android:name="org.adw.launcher_donut.permission.READ_SETTINGS" />
    <uses-permission android:name="org.adw.launcher_donut.permission.WRITE_SETTINGS" />
    <uses-permission android:name="com.huawei.launcher3.permission.READ_SETTINGS" />
    <uses-permission android:name="com.huawei.launcher3.permission.WRITE_SETTINGS" />
    <uses-permission android:name="com.fede.launcher.permission.READ_SETTINGS" />
    <uses-permission android:name="com.fede.launcher.permission.WRITE_SETTINGS" />
    <uses-permission android:name="com.sec.android.app.twlauncher.settings.READ_SETTINGS" />
    <uses-permission android:name="com.sec.android.app.twlauncher.settings.WRITE_SETTINGS" />
    <uses-permission android:name="com.anddoes.launcher.permission.READ_SETTINGS" />
    <uses-permission android:name="com.anddoes.launcher.permission.WRITE_SETTINGS" />
    <uses-permission android:name="com.tencent.qqlauncher.permission.READ_SETTINGS" />
    <uses-permission android:name="com.tencent.qqlauncher.permission.WRITE_SETTINGS" />
    <uses-permission android:name="com.huawei.launcher2.permission.READ_SETTINGS" />
    <uses-permission android:name="com.huawei.launcher2.permission.WRITE_SETTINGS" />
    <uses-permission android:name="com.android.mylauncher.permission.READ_SETTINGS" />
    <uses-permission android:name="com.android.mylauncher.permission.WRITE_SETTINGS" />
    <uses-permission android:name="com.ebproductions.android.launcher.permission.READ_SETTINGS" />
    <uses-permission android:name="com.ebproductions.android.launcher.permission.WRITE_SETTINGS" />
    <uses-permission android:name="com.oppo.launcher.permission.READ_SETTINGS" />
    <uses-permission android:name="com.oppo.launcher.permission.WRITE_SETTINGS" />
    <uses-permission android:name="com.huawei.android.launcher.permission.READ_SETTINGS" />
    <uses-permission android:name="com.huawei.android.launcher.permission.WRITE_SETTINGS" />
    <uses-permission android:name="telecom.mdesk.permission.READ_SETTINGS" />
    <uses-permission android:name="telecom.mdesk.permission.WRITE_SETTINGS" />
    <uses-permission android:name="dianxin.permission.ACCESS_LAUNCHER_DATA" />
```
