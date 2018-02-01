---
title: Python爬虫脚本 # 这是标题
date: 2018-02-01 13:16:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- Python # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- Python
---
# Python
Python (英国发音：/ˈpaɪθən/ 美国发音：/ˈpaɪθɑːn/), 是一种面向对象的解释型计算机程序设计语言。Python语法简洁清晰，特色之一是强制用空白符(white space)作为语句缩进。

# 学习
目前Python有两个版本2.x和3.x，市场上2.x用的是比较多的。所以还是建议先学2.x比较好一些。因为作者对Swift比较熟悉，所以在看Python的过程中上手很快，花了一天时间就把《零基础学Python》看了一遍。话说起来一直以来，大家都觉着Swift跟js语法很像。看完Python之后，我觉着数组、字符串、遍历这些与Python基本一样，就是换枪不换药。甚至有些特性Swift还没有，估计以后可能会有吧。哈哈

大家可能有点疑问为啥要学习Python呢？难不成要转行做大数据分析啥的吗？Python能做的确实很多，作者考虑的主要是工作中，很多时候要用到脚本帮助我们做一些看似简单但是挺繁琐费时间的事情，这时候如果用一个脚本那就是分分钟钟的事情啦。

目前苹果系统自带Python，版本是2.x的，所以大家不必安装，可以直接使用的。

Python工具的话Atom等都能写，不过作者推荐使用PyCharm。这个工具用起来很简单，而且还有很多提示啥的功能，非常方便。
# 实践
学习一门语言，要想学好，只有通过实践才能获得快速的成长。所以作者就实践了通过Python抓包的功能
## 环境安装
抓包之前我们得装以下2个插件，第二个不装也可以，用Python自带的，不过这个第三方插件速度等方面优异与Python自带的包，而且`Beautiful Soup`是推荐安装的
* [Beautiful Soup](https://www.crummy.com/software/BeautifulSoup/)
Beautiful Soup 是一个可以从HTML或XML文件中提取数据的Python库.它能够通过你喜欢的转换器实现惯用的文档导航,查找,修改文档的方式.Beautiful Soup会帮你节省数小时甚至数天的工作时间
* lxml

Beautiful Soup这个插件有个很大的好处就是有中文文档，方便了广大学友们。哈哈！！[文档地址传送门](https://www.crummy.com/software/BeautifulSoup/bs4/doc.zh/index.html)

如果使用的是PyCharm作为开发工具的话，大家可以看一下这个文档[PyCharm安装第三方模块Request、BeautifulSoup](http://blog.csdn.net/huatian5/article/details/74502687)
## 代码实践
我们抓取的是嗅事百科网站上的资料地址是:https://www.qiushibaike.com/8hr/page/1/

```Python
# -*- coding: utf-8 -*-

import urllib2
from bs4 import BeautifulSoup
num = 0
for page in range(1, 10):
    url = "https://www.qiushibaike.com/8hr/page/"+str(page)
    heads = {"User-Agent": "Mozilla/5.0"}
    res = urllib2.Request(url, headers=heads)
    response = urllib2.urlopen(res)
    html = response.read()
    soup = BeautifulSoup(html, "lxml")
    data = soup.select("div .content span")
    for some in data:
        num = num + 1
        print num
        unicode_string = unicode(some.string)
        if unicode_string != "None":
            print unicode_string + "\n"

```
> \# -*- coding: utf-8 -*-

这个主要是为了防止中文乱码，2.x需要加入的；3.x的话就不需要加了。

>data = soup.select("div .content span")

这一句是最关键的一句，首先我们要去看`BeautifulSoup`的官方文档`select`方法的使用。其次是我们如何拿到"div .content span"这个参数的呢？

1、在浏览器打开嗅事百科
2、在文字出右击，选择`检查`
![1517464304757.jpg](http://upload-images.jianshu.io/upload_images/6644906-7b00d9fc7c7d685f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3、出现以下界面
![1517464462211.jpg](http://upload-images.jianshu.io/upload_images/6644906-d6b593725f9c7ffc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过以上图我们可以看到`div`、`.content`、`span`这三项
>我们在写 CSS 时，标签名不加任何修饰，类名前加点，id名前加 #，在这里我们也可以利用类似的方法来筛选元素，用到的方法是 soup.select()，返回类型是 list

# 总结
大家如果有不明白的可以留言，作者也是初学者，可以相互学习共同进步。我的博客[FlyOceanFish](http://flyoceanfish.top/)
