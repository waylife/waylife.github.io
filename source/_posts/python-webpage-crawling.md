title: Python抓取单个网页中所有的PDF文档  
date: 2014-11-11 21:49:53  
categories: Python  
tags: [Python,技术,爬虫,网页抓取]  
---

# 1.背景  
最近发现算法以及数据结构落下了不少（其实还是大学没怎么好好学，囧rz），考虑到最近的项目结构越来越复杂了，用它来练练思路，就打算复习下数据结构与算法。结合最近在学英语，然后干脆就用英文喽。然后选定一本参考书籍《Data Structures and Algorithms in Java》。
刚开始看还是蛮吃力的，慢慢来。由于之前有翻录书籍附录的习惯，于是就去书籍附带的官网看了下，发现http://ww0.java4.datastructures.net/handouts/ 里面附带的PDF文档居然不错，图文并茂，作为理解是个不错的材料，果断要下载啊。但是，尼玛，结果发现，好多个，这一个一个另存为真是要命，想想还是用什么办法下载下来吧。

# 2.实现  
考虑目前学过的了解的所有语言，可以用来实现的，排列一下程度：
1. Java/Android  熟悉
2. C# 熟悉
3. Python 了解语法
4. Javascript 了解一些  
5. C/C++ 了解语法

为了实现这个，当然是最简单最快最好了。考虑到大学一直用C#，要不用它？但发现OSX平台只能用Mono了，还得重新熟悉。Java实现也不快，从需要的时间考虑。Javascript不熟，貌似可以用node.js去写(atom就是用的它)。不熟。C/C++好多年没用过了，而且，实现起来代码一大堆，特别麻烦。再考虑之前一段时间正好在Codecademy学过语法，就拿它来练手吧。  
OK，确定了用Python。后续就是怎么去请求网络了，解析网页html标签，提取下载链接，下载文件了。虽然不懂这些在Python里面是怎么实现的，但是流程是确定的，按照流程去网站找现成的，此处不研究原理，实现功能即可。
接下来就是各种搜索引擎搜索东西了，Google可，百度亦可（不同引擎侧重不一样）。不要忘了目的是什么，搜索相关的资料。  
好了，搜索之后，确定请求网络下载网页用requests，解析html用BeautifulSoup，提取下载链接BeautifulSoup，下载文档（stackoverflow中找到了一段下载文件的代码）。  
然后就是把她们一起组合了。组合之后的代码如下：
``` python
  #file-name: pdf_download.py
  __author__ = 'rxread'
  import requests
  from bs4 import BeautifulSoup


  def download_file(url, index):
      local_filename = index+"-"+url.split('/')[-1]
      # NOTE the stream=True parameter
      r = requests.get(url, stream=True)
      with open(local_filename, 'wb') as f:
          for chunk in r.iter_content(chunk_size=1024):
              if chunk: # filter out keep-alive new chunks
                  f.write(chunk)
                  f.flush()
      return local_filename

  #http://ww0.java4.datastructures.net/handouts/
  root_link="http://ww0.java4.datastructures.net/handouts/"
  r=requests.get(root_link)
  if r.status_code==200:
      soup=BeautifulSoup(r.text)
      # print soup.prettify()
      index=1
      for link in soup.find_all('a'):
          new_link=root_link+link.get('href')
          if new_link.endswith(".pdf"):
              file_path=download_file(new_link,str(index))
              print "downloading:"+new_link+" -> "+file_path
              index+=1
      print "all download finished"
  else:
      print "errors occur."
```
运行
``` bash
python pdf_download.py
```
便可以把所有的pdf文档下载到本地。

# 3.优化  
30多行代码，全部搞定，真是简洁明了，果然做Python用来一些脚本任务还是不错的。利用它下载了41个文档。
最开始下载下来的文档没有序号，这样看的时候就不知道先后，于是我给文件名前面加了个序号。
其他的优化部分可以参考如下：
1. 考虑现在函数的一些异常出错没有处理，后续需要处理。
2. 函数没有完全封装，下载的文件类型支持不多，这个后续可以根据自己的需求进行扩展。  
3. 下载的文件少的时候可能这样就行了，但是文件多的话，是有必要使用多个线程（适量的数量）或者线程池去下载，从而加快下载速度。  
4. 有些写法可能不符合python语法规范，当然写了与没写已经是0和1的区别了。
5. 其他细节，比如pdf有可能是大写的PDF。  

# 4.附录  
1. 《Data Structures and Algorithms in Java》(Michael T. Goodrich, Roberto Tamassia)下载 http://bookzz.org/ 或者 http://it-ebooks.info/
以下两个网站都是不错的书籍下载网站，有条件还是买本正版书籍支持一下作者吧。
一般我会先下载电子书看下，合适就买纸质版。  
2. Python语法入门 http://www.codecademy.com/zh/tracks/python  

以上，便是如此了。  

本文来自[RxRead’s Blog]("http://waylife.github.io"),欢迎转载，转载请注明。
欢迎一起交流探讨。
