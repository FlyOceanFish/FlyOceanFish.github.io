---
title: Python爬取美女照片 # 这是标题
date: 2018-02-06 13:22:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- Python # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- Python
---
也学了两个星期的Python了，从第一次看就有点爱上他的从动。感觉这门语言真的既简洁，同时功能也很强大。工作中遇到使用脚本的情景应该比较多，建议读者如果要学一门脚本的话，就学Python。哈哈哈哈。

学习Python，鬼使神差对网页爬虫很有兴趣，也一直在研究，路很长，我只是一个初学者。接下来介绍怎么爬取一个美女图片的网页。
# 分析
我们爬取的网址是"https://tuchong.com"其中的美女图片,第一次尝试是：
````Python
url = "https://tuchong.com/tags/%E7%BE%8E%E5%A5%B3/"
header = {"User-Agent": "Mozilla/5.0"}
res = urllib2.Request(url, headers=header)
response = urllib2.urlopen(res)
data = response.read()
````
虽然爬取到了网页，但是没有我们想要的美女图片数据。这时其实可以考虑两点：
* 我们`header`加的内容缺少了，可以通过浏览器仔细查看`header`
* 网页的数据时通过接口请求回来之后通过js加载的。

**事实证明是第二点**
接下来我们看调试面板
![1517895415261.jpg](http://upload-images.jianshu.io/upload_images/6644906-8b5ef86d44e94bd6.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
通过仔细观察我们找到了一个接口地址，而且通过'Preview'可以观察到返回的数据确实是我们想要的，而且可以看到数据返回是json

# 开始爬取

```Python
# -*- coding: utf-8 -*-

import urllib2
import json
import re

def __getgirlpicturesData(page):
    url = "https://tuchong.com/rest/tags/%E7%BE%8E%E5%A5%B3/posts?page="+str(page)+"&count=20"
    header = {"User-Agent": "Mozilla/5.0"}
    res = urllib2.Request(url, headers=header)
    response = urllib2.urlopen(res)
    data = response.read()
    josndata = json.loads(data)#通过json解析数据
    return josndata["postList"]


def getalllurls():
    urls = []
    for index in range(1, 10):#爬取9页的数据，并且获得图片的URL
        data = __getgirlpicturesData(index)
        for item in data:
            if item["type"] == "text":
                urls.append(item["title_image"]["url"])
            elif "cover_image_src" in item.keys():
                urls.append(item["cover_image_src"])
    return urls


def dowloadpicture(url):
    print(url)
    try:
        name = re.search("/\w*\.(jpeg|jpg)", url).group()#通过正则匹配图片地址最后的名字
    except:
        print("异常网址" + url)
    else:
        data = urllib2.urlopen(url).read()
        path = "/Users/xx/Desktop/picture"+name
        file = open(path, "w+")#将图片存储到本地
        file.write(data)
        file.close()


if __name__ == "__main__":
    urls = getalllurls()
    for url in urls:
        dowloadpicture(url)
```
爬取之后作者第一次尝试写了一个小脚本，将所有的图片重命名。
```Python
# -*- coding: utf-8 -*-
import os

path = raw_input("请输入地址").strip()

li = os.listdir(path)
num = 0
for file in li:
    num += 1
    name = str(num)+".jpg"
    os.rename(path+"/"+file, path+"/"+name)
print("修改成功")
```
再一次感受到了学习一门脚本语言的方便性。
# 花絮
作者尝试过二维码识别`pytesseract`,装环境花了不少时间，但是最后识别结果不是很理想；而且发现了一个比较有趣的东西`itchat`可以通过Python实现自动回复啥的，大家有兴趣可以玩玩
# 总结
今年过年前应该是最后一篇文章了，从开始搭建博客到现在已经写了48篇文章，也是有一点小成就感。很多人问：为啥要写博客，有什么用？其实好处的话，当你写的越来越多的时候自然而的就明白了，不用别人去说。这里我想说的是让我们在分享中成长吧。
