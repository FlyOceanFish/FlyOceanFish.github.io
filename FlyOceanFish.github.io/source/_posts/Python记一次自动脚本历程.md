---
title: Python记一次自动脚本历程 # 这是标题
date: 2018-03-13 14:52:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- Python # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- Python
---
# 介绍
现在我们iOS的安装包是通过企业形式的发布，比较狗血的是发布的方式，网站是一个静态网页,没有所谓的后台让你上传一个包，填写一个发布内容、版本号等一系列的信息，然后点击保存按钮。这些操作都是通过我们人工远程连接服务器去，然后备份、修复、复制等一系列的操作。由于正好学习了Python，并且介于这个场景，作者就自己写了一个Python脚本来自动化操作这一系列的操作。
# 历程
## 脚本运行的问题
说做就做，作者用了半天的时间脚本也写出来了，现在就是怎么让脚本在服务器上运行啊？难不成在服务器上安装Python环境？这样使用起来就确实很不爽。

其实大家应该第一想法Python能否打包成可执行文件的。搜索下来确实是可以的！使用`pyinstaller`即可，喜出望外啊！

作者此时兴致冲冲的赶紧安装了`pyinstaller`，马上就打了一个运行包。
>pyinstaller -F xx.py

-F 参数可以打出一个安装包
-D 默认参数，是一个文件夹
## 平台的问题
运行包打出来，运行下来很不错的样子。问题来了，我们服务器是Windows系统！！怎么运行啊？？

又搜索研究，最终看`pyinstaller`死心了。。。要想在Windows系统上运行，必须在Windows系统打包才行！

好吧，这就把虚拟机装起来，系统搞起来。（VirtualBox不错，免费的）接下来装Python环境，因为Mac上是Python2.x，所以就在Windows系统上装了Python3.x的。然后也顺利打出了.exe文件

## 服务器上运行

在自己虚拟机上运行，一切正常。满怀喜悦复制到服务器上运行（当然是自己写了一个test文件，不是正式的脚步文件）。一下子就弹出了`api-ms-win-crt-runtimel1-1-0.dll`缺少的错误！oh my god。接下来开始解决dll文件缺失的问题，服务器又不管乱安装文件，最终想找运维帮忙搞一下环境。

## 灵机一现

搞到上一步已经下班了，运维啥的一时半会也找不到，找到人家也不一定马上帮你弄，所以只能第二天再搞。这时是有多心不甘啊！回家路上睡觉之前就在想这个事情。突然灵机一现，回想了白天搜索的过程，其实发现报这个错误都在Python3.x上，会不会在Python2.x上打包就可以了呢？

第二天一到公司，马上卸载了3.x，装上了2.x。写了一个test.py脚本，然后打包在服务器运行测试。YES，成功了！

## 编写正式脚本
Python学习的时候就知道中文有编码的问题，所以早早就在文件开头写下了
>#coding=utf-d

哈哈，暗自庆幸自己很聪明。接下来开始测试脚本，然后运行。啪啪啪啪！打脸！又报错。因为服务器上文件下有个中文的文件夹，所以路径有一个是带中文的文件夹。报错显示就在这里。

然后研究。。。。已经写了编码怎么还会报错啊。头大！研究了一会，没救。就在琢磨是不是编码问题，改成了gbk。oh 居然成功了！！
```
#coding=gbk
```
## 获取当前.exe文件执行的文件夹
这个也是挺纠结的一个问题，通过一遍遍测试得出下边的结论。
>ios_dis_path = os.path.split(sys.argv[0])[0]

## 执行过程显示
.exe执行的时候，肯定想知道一些反馈信息，所以在代码过程中打印了log信息。但是有个问题，命名执行完后命令提示框一闪而过，根本看不到。
加上以上代码可以将提示框停住。
>raw_input("输入回车退出")

# 代码
代码已经脱敏处理了,因为有个问题就是发布完APP之后，要验证完之后才能正式修改sqlite数据库，所以作者加了一个输入框等待用户输入，当发布人员发布完成并且验证成功之后，输入本次升级的版本，然后按回车即可发布成功
````Python
#!/usr/bin/env python
#coding=gbk
import sys
import os
import time
import shutil
import sqlite3

database_name = 'apidb.db3'
date_dir = time.strftime("%Y%m%d", time.localtime())
root_path = 'C:\\xxx\\xx'

ios_dis_path = os.path.split(sys.argv[0])[0]
#ios_dis_path = "C:\\xx\\Desktop\\ios"
# 备份数据库
print('########开始备份数据库########\n')

back_database_path = os.path.join(root_path, 'backup/数据库/'+date_dir)
if not os.path.exists(back_database_path):
    os.mkdir(back_database_path)
shutil.copy(os.path.join(root_path, database_name), back_database_path)

# 备份xx_project_name.html
print('########  备份数据库完成！开始备份xx_project_name.html  ########\n')

index_path = root_path+'/xx_project_name.html'
back_index_path = os.path.join(root_path, 'backup/xx_project_name/'+date_dir)
if not os.path.exists(back_index_path):
    os.mkdir(back_index_path)
shutil.copy(index_path, back_index_path)
shutil.copy(os.path.join(ios_dis_path,'xx_project_name.html'), index_path)
#备份安装包
print('########  复制xx_project_name.html完成！开始备份安装包  ########\n')

project_path = os.path.join(root_path, 'download/images/ios/xx_project_name')
backup_ipa_path = os.path.join(project_path, date_dir)
if os.path.exists(backup_ipa_path):
    shutil.rmtree(backup_ipa_path)
shutil.copytree(project_path+'/latest', backup_ipa_path)

# 复制新的安装包
print('########  备份安装包完成！开始复制新的安装包  ########\n')

shutil.copy(os.path.join(ios_dis_path, 'xx_project_name.ipa'), project_path+'/latest')
shutil.copy(ios_dis_path+'/manifest.plist', project_path+'/latest')
print("########  复制新的安装包完成！开始修改数据库中App的版本########\n")

# 修改数据库

version = raw_input("请输入此次发布的版本号:")

def update_version(version):
    print("########  开始修改数据库中的版本号！########\n")
    date = time.strftime("%Y/%m/%d %H:%M:00", time.localtime())
    conn = sqlite3.connect(os.path.join(root_path, database_name))
    conn.execute('update api set version=(?),update_time=(?) where api_name = "xx_project_name"', (version, date))
    conn.commit()
    conn.close()
    print("########  修改数据库中的版本号成功！########\n")


update_version(version)

print('--------- APP全部发布成功！ -----------\n')
raw_input("输入回车退出")

````
## 效果图
![1520923800618.jpg](https://upload-images.jianshu.io/upload_images/6644906-dba402400d4357cf.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
