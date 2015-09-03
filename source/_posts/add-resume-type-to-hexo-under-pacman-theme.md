title: Pacman主题下给Hexo增加简历类型  
layout: post  
date: 2015-01-02 23:23:59
categories: Web
tags: [Web,网页]
---

#背景
虽然暂时不找工作，但是想着简历也是个向别人推销自己的好东西。然后也想着折腾点新的东西，如此，这般，便想着研究起写个简历了。  
形式不限，但是必须是在线的，最好是很简洁的。

#分析
既然是在线的，那就干脆直接用博客呗，直接放在上面。  
写博客既然用Markdown，那简历也直接用Markdown，一个是可以在线渲染，另外一个是生成PDF的工具也很多，Github一搜就有好几个不错的。  
由于用的主题是[Pacman](https://github.com/A-limon/pacman),那就在它的基础上直接改。虽然网页相关基础只是了解一些，但是并不妨碍修改。  
先看下最终的效果吧。[简历效果](/resume)。  是不是很简洁，达到了预期的效果。嘿嘿。  

#实现
hexo的主题是放在themes目录下的。先从[Pacman](https://github.com/A-limon/pacman)下载Pacman主题。下载完成后参考官方说明设置主题为它。  
接着就开始动工了。

既然是简历嘛，那就不希望它出现在首页的列表里面。也就是md文件不要放在/source/_posts文件夹。建一个about的文件夹，放在/source/about里面。  
然后新建一个resume.md文件。  
注意，这个地方我遇到了一个坑，由于md是自己写的，不是hexo自动生成的，导致date后面那行的"---"（三个-）没写，然后后面就坑爹了，死活生成不了正常的页面，同时顶部的键值对冒号之间注意要加空格，很多这种坑。  
然后一切按照正常的流程走，把简历也写好。

``` bash
title: Resume  
layout: post  
date: 2015-01-02 23:23:59
---
# 王小二(wangxiaoer#gmail.com)

##个人信息

 - 本科/XXX大学(20XX.9-20XX.7)/计算机科学与技术
 - 工作年限：2年
 - 技术博客：http://xxxxx.com
 - 地点：北京

##工作经历

###五道口宇宙中心
####XXXX项目（2013.10-至今）
 - XXXXXXXXXXXXXXXXX
 - XXXXXXXXXXXXXXXXXX
 - XXXXXXX
 - XXXXXXXXXXXXXXXXXXXXXX


##技能列表
熟悉：Android/Java
了解：C#/WP，Python，HTML, Markdown等
```
现在看起来跟Pacman主题下任何普通的博文样式一样。
现在怎么去掉首部，以及尾部，以及侧边栏等等一切不要的东西，不要捉急，慢慢来，一个一个砍，一步一步实现咱们想要的效果。  
看到md首部有个layout属性，是不是可以从这里开刀呢？  
再联想到Pacman配置搜索页面需要设置layout属性为search。OK，就以search为关键词在pacman中搜索，然后在/pacman/layout/layout.ejs里面找到了它。这种关键词查找突破法在很多地方可以用到。之前研究别人APK实现原理的时候就经常用，屡试不爽，下次写文章讲解APK的逆向研究。  
OK，找到了。发现里面大致的一段代码如下：
``` bash
<% } else if(page.layout=='search'){ %>
 <%- partial('_partial/head') %>
    <body>
      <header>
        <%- partial('_partial/header') %>
      </header>
      <div id="container">
        <%- partial('_partial/search')%>
      </div>
      <%- partial('_partial/after_footer') %>
    </body>
   </html>
```
想起之前学习php时也会有这种混用，html代码与php代码混在在一起，这里应该也是差不多的。暂时先这样认为，当然后续也证实了这是没错的。
既然pacman可以自定义search属性，那咱们也可以自定义一个resume属性。   那里面要放什么内容呢？ 跟search节点一样？  
然后查看hexo主题属性layout相关的文档[http://hexo.io/docs/themes.html](http://hexo.io/docs/themes.html#Layout)，里面说一个布局必须包含"<%- body %> "才能显示内容。
常用的是layout值是post或者page，先看下page果然有"<%- body %> "，那就考虑直接复制过来，用了再说。post与page的区别通过测试可以发现post右边有Sidebar，样式也有一定区别。鉴于实际情况，此处使用page的内容。  
改好之后就直接运行看效果了，尼玛，背景是灰色的啊，不能忍啊，太tm难看了。page类型的就是白色的，然后使用Chrome审查元素，看到内容的class id为resume，page的为page，应该跟这个有关系，然后搜索，找到了/pacman/source/css/_partial/article.styl。第一行加上,.resume，一切搞定。  
接着就是怎么搞定首部和尾部了。看到layout文件里里面有个page.ejs文件，先拷贝一份为resume.ejs（处理页面数据）,/_partial文件夹里面的也类似（这个用来处理往首页列表加摘要）。这个文件里面使用到的/_partial/post/article.ejs也拷贝一份为resume.js。修改/layout/resume.ejs文件里面的指向。  
由于不需要评论以及尾部，删除/_partial/post/article.ejs底部的一些数据，最终代码如下：
``` bash
<div id="main" class="<%= item.layout %>" itemscope itemprop="blogPost">
  <article itemprop="articleBody">
    <%- partial('header') %>
  <div class="article-content">
    <%- partial('gallery') %>
    <% if( table&&(item.toc !== false) && theme.toc.article){ %>
    <div id="toc" class="toc-article">
      <strong class="toc-title"><%= __('contents') %></strong>
    <%- toc(item.content) %>
    </div>
    <% } %>
    <%- item.content %>
  </div>
  </article>
</div>
```
然后回到/_partial/post/article.ejs里面的resume分支，之前复制的是page代码，我们需要删掉一些数据，考虑只留下"<%- body %> "，运行一下，卧槽，怎么会有乱码。应该是头部有编码信息，要考虑保留，尾部由于也有统计信息，也要考虑保留，最后经过不断的"删除->查看效果->复原"，确定删除的部分，最终保留代码如下：
``` bash  
<% } else if(page.layout=='resume'){ %>
  <% if(page.source.match(/\.md$/)){ %>
   <%- partial('_partial/head') %>
     <body>
       <div id="container">
         <%- body %>
       </div>
      <div id="footer"><br/></div>
       <%- partial('_partial/after_footer') %>
     </body>
    </html>
    <% }else{ %>
   <%- page.content %>
 <% } %>
<% } else if(page.layout=='page'){ %>
```
最后运行一下，就是之前的效果了。Perfect！
最后编写简历时把layout设置为resume就可以了。同时，最好不要把简历的md放在_posts目录下，如果放在里面，在首页列表里面会包含有它，而且样式有一定问题。/about/resume.md最后生成的路径是 http://yourwebsite/about/resume.html ，然后在首页的顶部导航栏增加这个路径就可以了。

#总结
不仅仅可以增加resume属性，也可以增加其他的来扩展更多的自定义页面，比如404页面。
花了一晚上时间研究，虽然在---三个-处耽误了不少时间，还是蛮有趣的。  
有任何问题欢迎向我反馈。  
源码我已经放在[Github](https://github.com/waylife/pacman_with_resume)上了，欢迎star以及fork。  
