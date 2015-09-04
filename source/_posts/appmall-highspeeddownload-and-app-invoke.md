title: 应用市场高速下载以及网页端调起APP页面研究与实现  
date: 2014-10-27 23:05:53  
categories: Android  
tags: [Android,网页调起应用,自定义协议]  
---
好久没写博客了，好大一个坑。正好，最近刚做完应用市场的高速下载功能，便拿来填了这个坑。  
话说产品为了增加用户量，提升用户活跃度以及配合推广，更坑爹的是看到其他市场也有这些功能，等等，要求做一个捆绑下载的功能。WTF。
当然吐槽归吐槽，任务还是要完成的。  
具体要求是：
用户在手机浏览WAP站点的时候，~~***1.进入应用详情页时打开本应用(应用市场)里面的详情页面 2.点击WAP端高速下载时，如果本应用已安装，则调用本应用进行下载，否则下载本应用的捆绑包，安装完成之后，在本应用打开时去下载之前用户想下载的应用。***~~
如何实现这两个功能呢？以下逐步分析：  
# 1.功能需求  
如之前所描述。  
一般来说，完成之前的两个功能，一般都是需要应用已安装并且在后台运行(最简单的理解就是应用打开过应用，并且没有被其他程序杀死)，因此此处也默认这种情形才算应用已安装；否则，一概认为是未安装（以下没有特别说明，均为这种情况）。
# 2.竞品分析  
当然，既然竞品有了这些功能，那就先拿来主义，研究下他们是怎么做的吧。通过Chrome抓包分析，最后分析如下。  
* 百度手机助手 http://127.0.0.1:6259/***  
* 搜狗手机助手 http://127.0.0.1:12388/***
* 豌豆荚 自定义协议wandoujia://detail/app/cn.buding.martin
百度手机助手以及搜狗手机助手的方案基本是一致的，~~***采用的是访问一个手机本地服务地址127.0.0.1（回环地址）,端口地址有所不同***~~，豌豆荚采用的是自定义协议，后续让浏览器自动帮它分发给注册了wandoujia协议的Activity。
---  
1. **打开应用详情** 这个在打开浏览器相应详情页面时，百度手机助手以及搜狗手机助手同时访问本地回环地址，而豌豆荚则调用自定义协议由系统调用相关的应用（一般就是豌豆荚）。  
2. **高速下载原理** 豌豆荚没有实现在应用已安装情形下调用客户端进行下载，在点击下载时，询问用是下载捆绑包还是直接下载想要下载的应用（描述文案见文末附图，文案有诱导性）。选择下载捆绑包，则下载一个特定文件名的豌豆荚安装包。在安装完成之后，去扫描特定的目录（一般是download以及常见浏览器的下载目录），如果存在符合规则的文件，则提取相关的资源ID，后续再下载捆绑的APP。  
在应用没有安装的情形下。而百度手机与搜狗手机助手点击高速下载（名字也有诱导），则直接下载一个APP，所有APP内容一样，但是APP名字有所不同，均类似于app_捆绑id_xx.apk，比如搜狗手机助手捆绑微信的为MobileTools_8271386494777466339_71.apk，捆绑其他包的捆绑id有所不同。其他流程同豌豆荚。如果存在多个符合条件的apk，搜狗手机助手则取最新的信息进行下载，百度手机助手没有研究。  
如果手机APP已经安装。豌豆荚没有实现，而百度手机助手以及搜狗手机助手，则是浏览器网页直接请求一个回环地址(127.0.0.1:port/actionpath?parameter)，port为端口地址，百度手机助手为6259，搜狗手机助手为12388。actionpath为需要进行的操作，不同的操作，值也不一致 parameter为相关操作的参数，在这里把需要捆绑下载的APP数据传过去。手机APP通过应用内的HTTP服务器接收到相关数据后进行应用下载，同时返回相应的数据。  

# 3.最后方案  
**调起详情页面**，采用HTTP服务器以及自定义协议两种都可以，而且自定义协议不用一直在后台跑一个承载HTTP服务器的Service，会比较省电。  
后续的**高速下载**，豌豆荚是没有了，既然之前说到自定义协议省电，能否考虑呢？再结合需求，看下，应用未安装的情形下，直接下载捆绑包，两种都没有问题；应用已安装的情形，调起客户端进行下载。自定义协议可以做到么？如何知道应用是否安装了呢？浏览器有相应的API么？显然没有，就算有，也只可能部分有，但是要支持所有的浏览器。唯一的办法就是通过访问HTTP服务器，同时设置超时时间，一定时候内没有响应则认为应用未安装，下载相应的捆绑包。而且，通过HTTP服务器后续扩展也好，可以通过网页服务器与Web进行双向互动，比如百度手机助手可以通过web获得位置信息。而自定义协议就显然做不到这些。当然除了省电。  
高速下载的调起解决了，那如果应用没有安装，在本应用安装完成之后，如何知道之前用户需要下载的是哪个应用呢？也就是**捆绑应用的识别**。
应用市场最开始的方案是在应用里面（asset文件夹下）打入一个文件，存放有捆绑应用的id号。但是，针对少量应用可以（最开始为了测试捆绑效果时用过），目前来说，针对大量应用肯定不行，需要打大量的包，而且需要不少存储空间。  
参考竞品的方案，确实很完美，但是，考虑目前大部分的手机安全或者清理软件，都会在应用安装成功后提示删除安装包，同时，浏览器的下载目录可能会会变。那这里能扫描整个SD卡，去寻找符合条件的文件么？肯定不行，太消耗时间了。  
那在应用没有安装或者没有在后台运行的情形下，如何知道捆绑的应用id呢。以上两个方案都不完美，如何解决。与后端彦飞探讨过，在高速下载apk时，网页记录手机下载的捆绑app id以及packagename以及该设备的唯一id（uuid），然后在打开市场时，请求相应的接口并把uuid传入，获得数据，从而下载相应的应用。想法超赞，后来发现没有实用价值，uuid暂时没有一个合理的方案。关键在于两端生成的都要唯一，mac，ip？如何生成？  
后来吃饭与国畅探讨后，考虑是不是可以调用本地代码去执行相关任务？但仔细思考过后，发现这个一般是WebView与js进行交互。而目前的使用场景根本就无法做到。这个Webview必须要是自己应用内的才可以。  
考虑多方面因素，最后决定，参考百度手机助手以及搜狗手机助手的方案，尽管有缺陷。
关于自定义协议模式，可参考此文章，[android自定义协议和html加载时自动尝试调用本地APP](http://www.oschina.net/code/snippet_256033_35330)，以及[Android 注册监听自定义协议](http://www.pocketdigi.com/20101002/117.html)，本文不做过多介绍。  
~~***最后的方案就是手机端后台通过Service运行一个HttpServer，监听12307端口。***~~
## 调起客户端
### 1.高速下载  
网页端点击高速下载时，访问http://127.0.0.1:12307/appdown?downid=1&packagename=com.sogou.map  
如果成功，返回json串形式为  
``` bash
{
  "status":1,
  "message":"OK"
}  
```
status为0表示失败，status为1表示成功。message为具体描述。  
如果在一定时间内手机没有响应，则默认应用没有安装，下载捆绑APK包。  
如果手机端响应了，则网页端不做任何处理，手机端获得相应的downid以及packagename，先判断应用是否已经安装，如果安装，则提示应用已安装，如果未安装，则提示用户应用正在下载，然后加入下载列表。  
### 2.打开详情页面  
http://127.0.0.1:12307/godetail?downid=1&packagename=com.sogou.map&backtohome=ture
backtohome为true时，按下手机端的返回按钮回到应用的主页，为false时返回浏览器。（可参考豌豆荚，比如[http://www.wandoujia.com/apps/com.tencent.mm](http://www.wandoujia.com/apps/com.tencent.mm)必须手机端访问，User-Agent与PC访问不同）  

## 安装包应用捆绑  
如何识别捆绑的是哪个应用呢？  
后续参考竞品的实现方式，通过下载文件名不同（文件名有规则）的文件，在用户安装应用并且打开后去读取下载文件夹，判断是否存在符合规则的apk。如果存在，提取相应的id，去进行相应的下载。只要服务器提供接口让浏览器在下载同一个文件显示为不同的文件名即可。  
比如搜狗手机助手的一个微信详情地址为[http://wap.sogou.com/app/apkdetail.jsp?ppid=34&cid=49&docid=8271386494777466339&e=1394](http://wap.sogou.com/app/apkdetail.jsp?ppid=34&cid=49&docid=8271386494777466339&e=1394)，高速下载安装包的地址是[http://download.zhushou.sogou.com/zhushou/android/MobileTools.apk?dn=MobileTools_8271386494777466339_71.apk](http://download.zhushou.sogou.com/zhushou/android/MobileTools.apk?dn=MobileTools_8271386494777466339_71.apk)
下载下来的apk名称为MobileTools_8271386494777466339_71.apk。捆绑的应用id为8271386494777466339，应用为微信。通过设置HTTP返回数据的headers的`Content-Disposition`字段为`attachment;filename="MobileTools_8271386494777466339_71.apk"`就可以实现，一般来说，主流浏览器都支持该属性。  

因此，最终确定，APK的命名方式为XXXX_binding_downid_9.apk，判断应用为这种格式（可通过正则匹配）就会提取downid（downid必须为整数）。应用未安装时，点击高速下载，下载以上规则命名的apk到文件夹。用户安装应用并打开应用时，去扫描常用的下载文件夹（常见的/download,/downloads,以及浏览器下载文件夹等）里面的apk，有符合规则的则提取某一个文件中的id，提取完成之后，进行下载，同时把安装包删除（防止对之后进行干扰）。考虑到同一个文件夹可能有多个符合规则的安装包，则按照时间顺序取当前文件夹符合规则的时间最近的两个，其他的文件夹的忽略，等下次启动再处理（一般这种情况也比较少）。当然还有其他一些细节需要处理，此处不再详谈。  
# 4.性能优化与展望  
程序优化，是有追求的攻城师必须做的事情，因此，当然要优化喽。  
为了省电，指定了以下两条策略。  
1. 在网络变化时，如果没有网络，则关闭服务，有网络，则打开服务。解屏以及锁屏时分别打开以及关闭服务。  
2.  有网，且屏幕打开是触发服务开启的必要条件。可以大部分降低应用在锁屏状态下有网络变化而导致的耗电问题。  
以上两个策略可叠加使用，可能会有些问题，比如有些时候条件都符合，但是服务没有开启（网络十分频繁的切换可能会导致）。  
设定为以上条件是因为，后续的操作都是有网才能完成，其次，锁屏状态下，一般用户是没办法触发相关的流程的，除非程序自动触发（即时通讯软件接收信息比如，或者自动下载，但是这太流氓了）才需要在锁屏时在后台跑。

HTTP服务器是用的Github上开源的[Nanohttpd](https://github.com/NanoHttpd/nanohttpd)，http服务器除了高速下载，也可以用在，比如传输数据到电脑，应用与网页交互等等很多方面。  

# 5.最终实现  
可访问[http://app.sogou.com/m](http://app.sogou.com/m) 查看最终效果。  
大致的思路就是以上描述。希望对大家有所启发，起个抛砖引玉的作用。  
目前暂时未上线，估计等到11月5号。  

# 6.相关工具以及附录  
（1）Chrome 开发者工具+设备模式（数据抓包）  
（2）三个市场的微信下载的演示地址：[百度手机助手](http://m.baidu.com/app?action=content&uid=0&from=844b&ssid=0&bd_page_type=1&pu=sz@1320_224#docid=7106818&ala=se@v@1@3@title&f=aladdin@%E5%BE%AE%E4%BF%A1)，[搜狗手机助手](http://wap.sogou.com/app/apkdetail.jsp?ppid=34&cid=49&docid=8271386494777466339&e=1394)，[豌豆荚](http://www.wandoujia.com/apps/com.tencent.mm)  
当时（2014-10-27的页面截图如下）  
![百度手机助手-搜狗手机助手-豌豆荚__微信捆绑下载图](https://raw.githubusercontent.com/waylife/blogimage/master/2014/10/binding_app_baidu_sogouzhushou_wandou.jpg)
（3）常见浏览器的APK下载路径（#表示该行是注释，每行一个目录，目前不支持遍历子目录）  
``` bash
#this is the binding apk file path
#this is  a comment
#some system default, Chrome, Opera, Opera mini, Sogou Broswer, ES File Explorer,Dolphin Browser
download
#meizu browser, some system default
download/apk
downloads
#vivo
下载
下载/App
#uc browser
UCDownloads
#QQ Broswer
QQBrowser/安装包
QQBrowser/下载
qqbrowser/download
#QQ Broswer HD
QQBrowser
#baidu broswer
baidu/flyflow/downloads
#baidu app
baidu/searchbox/downloads
#Maxthon Broswer
MxBrowser/Downloads
TTDownload/installapk
Application
ThunderDownload
#liebao broswer
kbrowser_fast/download/App
#360 Broswer
360Browser/download
#360 Express Broswer
360ExpressBrowser/download
#2345 broswer
Download/2345浏览器下载/安装包
#hao123
hao123/down/apk
DolphinBrowserCN/download
UCDLFiles
QCDownload
LXDOWNLOAD/DOWNLOAD
apc/ApcBrowser/downloads
#YueDong Broswer,Ignore,The apk file name is changed by the broswer,the same with 4G Explorer(do not support header's Content-Disposition attribute)
#ydBrowser/download
#4G-explorer/apks
```

哈哈哈哈，完美解决所有问题。之后还可以扩展。  
![ ](https://github.com/waylife/blogimage/raw/master/common/quick_run.gif)  
![](https://github.com/waylife/blogimage/raw/master/common/simle_1.gif)

# 7.后续
后续发现知乎上有一篇系统介绍文章http://zhuanlan.zhihu.com/andlib/19848910  
自定义协议一个完整的例子：http://rincliu.com/blog/2013/08/06/webapp/  


本文来自[RxRead's Blog](http://waylife.github.io),欢迎转载，转载请注明。  
