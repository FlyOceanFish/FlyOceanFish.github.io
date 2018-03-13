---
title: Python基础知识分享 # 这是标题
date: 2018-03-12 13:22:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- Python # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- Python
---
# 介绍
之前也走马观花的看了一下Python的基础，但是感觉有点不扎实，所以自己又重新细细的把基础过了一遍，同时把觉着重要的记录下来。文章最末尾分享了《Python爬虫开发与项目实战》pdf书籍，此pdf是高清有目录的，有需要的朋友拿去。
# 元组

***元组内的数据不能修改和删除***

Python 表达式 | 结果|描述
---|---|--
('Hi!',) * 4 | ('Hi!', 'Hi!', 'Hi!', 'Hi!')|复制
3 in (1, 2, 3) | True|元素是否存在

>任意无符号的对象，以逗号隔开，默认为元组。例：x, y = 1, 2;

## 创建一个元素的元组
***一定要有一个逗号，要不是错误的***
```Python
tuple = ("apple",)
```
## 通过元组实现数值交换

```Python
def test2():
    x = 2
    y = 3
    x, y = y, x
    print x,y
```

# 查看帮助文档
help
```Python
help(list)
```
# 字典
```Python
dict["x"]="value"
```
如果索引x不在字典dict的key中，则会新增一条数据，反之为修改数据

# set()内置函数
set() 函数创建一个无序不重复元素集，可进行关系测试，删除重复数据，还可以计算交集、差集、并集等。
```Python
x = set(["1","2"])
y = set(["1","3","4"])
print x&y # 交集
print x|y # 并集
print x-y # 差集
zip(x) #解包为数组
```
# zip()内置函数
zip() 函数用于将可迭代的对象作为参数，将对象中对应的元素打包成一个个元组，然后返回由这些元组组成的列表。

如果各个迭代器的元素个数不一致，则返回列表长度与最短的对象相同，利用 * 号操作符，可以将元组解压为列表。

```Python
a = [1,2,3]
b = [4,5,6]
c = [4,5,6,7,8]
zipped = zip(a,b)     # 打包为元组的列表[(1, 4), (2, 5), (3, 6)]
zip(a,c) # 元素个数与最短的列表一致[(1, 4), (2, 5), (3, 6)]
zip(*zipped)  #与zip相反，可理解为解压，返回二维矩阵式
[(1, 2, 3), (4, 5, 6)]
```
# 可变参数
在函数的参数使用标识符"\*"来实现可变参数的功能。"\*"可以引用元组，把多个参会组合到一个元组中；
"\**"可以引用字典
```Python
def search(*t,**d):
    keys = d.keys()
    for arg in t:
        for key in keys:
            if arg == key:
                print ("find:",d[key])

search("a","two",a="1",b="2") #调用
```
# 时间与字符串的转换
* 时间转字符串
使用time模块中的`strftime()`函数
```Python
import time

print time.strftime("%Y-%m-%d",time.localtime())
```
* 字符串到时间
使用time模块中`strftime`和datetime模块中的`datetime()`函数
```Python
import time
import datetime

t = time.strptime("2018-3-8", "%Y-%m-%d")
y, m, d = t[0:3]

print datetime.datetime(y,m,d)
```
# 操作文件和目录操作
比如对文件重命名、删除、查找等操作

* `os`库:文件的重命名、获取路径下所有的文件等。`os.path`模块可以对路径、文件名等进行操作
```Python
files = os.listdir(".")
print type(os.path)
for filename in files:
    print os.path.splitext(filename)# 文件名和后缀分开
````

* `shutil`库：文件的复制、移动等操作
* `glob`库：glob.glob("*.txt")查找当前路径下后缀名txt所有文件

# 读取配置文件
通过`configparser`(3.x，ConfigParser（2.x）)库进行配置的文件的读取、更改、增加等操作
```Python
config = ConfigParser.ConfigParser()
config.add_section("系统")
config.set("系统", "系统名称", "iOS")
f = open("Sys.ini", "a+")
config.write(f)
f.close()
```
# 正则
`re`正则匹配查找等操作
# 类
## 属性
私有属性名字前边加"__"
```Python
class Fruits:
    price = 0               # 类属性，所有的类变量共享，对象和类均可访问。但是修改只能通过类访问进行修改

    def __init__(self):
        self.color = "red"  # 实例变量，只有对象才可以访问
        zone = "中国"        # 局部变量
        self.__weight = "12" # 私有变量，不可以直接访问，可以通过_classname__attribute进行访问


if __name__ == "__main__":
    apple = Fruits()
    print (apple._Fruits__weight) #访问私有变量
```
## 方法
* 静态方法
```Python
    @staticmethod
    def getPrice():
        print (Fruits.price)
```
* 私有方法
```Python
    def __getWeight(self):
        print self.__weight
```
* 类方法
```Python
    @classmethod
    def getPrice2(cls):
        print (cls.price)
```
## 动态增加方法
Python作为动态脚本语言，编写的程序也具有很强的动态性。

>class_name.method_name = function_name
## 类的继续
并且支持多重继承

格式：
>class class_name(super_class1,super_class2):
### 抽象方法
```Python
    @abstractmethod
    def grow(self):
        pass
```
## 运算符的重载
Python将运算符和类的内置方法关联起来,每个运算符对应1个函数。例如__add__()表示加好运算符;__gt__()表示大于运算符

通过重载运算符我们可以实现对象的加减或者比较等操作。

# 异常
捕获异常
>try: except:finally:
## 抛出异常
raise语言抛出异常
## 断言
>assert len(t)==1

# 文件持久化
## `shelve`本地建库
`shelve`模块提供了本地数据化存储的方法
```Pyton
addresses = shelve.open("addresses") # 如果没有本地会创建
addresses["city"] = "北京"
addresses["pro"] = "广东"
addresses.close()
```
## cPickle 序列化
`cPickle`和`pickle`两个模块都是来实现序列号的，前者是C语言编写的，效率比较高

序列化：
```Python
import cPickle as pickle
str = "我需要序列化"
f = open("serial.txt", "wb")
pickle.dump(str, f)
f.close()
```
反序列化:
```Python
f = open("serial.txt","rb")
str = pickle.load(f)
f.close()
```
## json文件存储
Python内置了json模块用于json数据的操作
* 序列号到本地
```Python
import json
new_str = [{'a': 1}, {'b': 2}]
f = open('json.txt', 'w')
json.dump(new_str, f,ensure_ascii=False)
f.close()
```
* 从本地读取
```Python
import json
f = open('json.txt', 'r')
str = json.load(f)
print str
f.close()
```
# 线程
## threading模块
>class threading.Thread(group=None, target=None, name=None, args=(), kwargs={}, *, daemon=None)
## 线程和queue
```Python
# -*- coding:UTF-8 -*-

import threading
import Queue

class MyJob(threading.Thread):
    def __init__(self):
        threading.Thread.__init__(self, name="aa")

    def run(self):
        print threading.currentThread()

        while not q.empty():
            a = q.get()
            print("我的%d"%a)
            print "我的线程"
            q.task_done()


def job(a, b):
    print a+b
    print threading.activeCount()
    print "多线程"


thread = threading.Thread(target=job, args=(2, 4), name="mythread")
q = Queue.Queue()
if __name__ == "__main__":
    myjob = MyJob()
    for i in range(100):
        q.put(i)
    myjob.start()
    q.join() #每个昨晚的任何必须调用task_done()，要不主线程会挂起
```
# 进程
`multiprocessing`中 `Process`可以创建进程，通过`Pool`进程池可以对进程进行管理
```Python
from multiprocessing import Process
import os

def run_pro(name):
    print 'process %s(%s)' % (os.getpid(),name)

if __name__ == "__main__":
    print 'parent process %s' % os.getpid()
    for i in range(5):
        p = Process(target=run_pro, args=(str(i)))
        p.start()
```
# 爬虫
## 爬取数据
* urllib2/urllib Python内置的，可以实现爬虫，比较常用
```Python
import urllib2
response = urllib2.urlopen('http://www.baidu.com')
html = response.read()
print html

try:
    request = urllib2.Request('http://www.google.com')
    response = urllib2.urlopen(request,timeout=5)
    html = response.read()
    print html
except urllib2.URLError as e:
    if hasattr(e, 'code'):
        print 'error code:',e.code
    print e
```
* Requests 第三方比较人性化的框架
```Python
import requests
r = requests.get('http://www.baidu.com')
print r.content
print r.url
print r.headers
```
## 解析爬取的数据
通过`BeautifulSoup`来解析html数据，Python标准库（html.parser）容错比较差，一般使用第三方的`lxml`,性能、容错等比较好。
# hash算法库
## hashlib介绍
hashlib 是一个提供了一些流行的hash算法的 Python 标准库．其中所包括的算法有 md5, sha1, sha224, sha256, sha384, sha512. 另外，模块中所定义的 new(name, string=”) 方法可通过指定系统所支持的hash算法来构造相应的hash对象
# 比较好的资料
* [Python入门教程](https://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/001431658427513eef3d9dd9f7c48599116735806328e81000)
* [Python爬虫（听鬼哥说故事的博客）](http://blog.csdn.net/guiguzi1110/article/list/1)

# 《Python爬虫开发与项目实战》pdf书籍

链接: [Python爬虫开发与项目实战](https://pan.baidu.com/s/1hpVUaqcb2eAhVz6VNLtRgg) 密码: g19d
