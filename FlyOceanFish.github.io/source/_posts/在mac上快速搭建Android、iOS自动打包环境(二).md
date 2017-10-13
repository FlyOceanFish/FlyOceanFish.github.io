---
title: 在mac上快速搭建Android、iOS自动打包环境(二) # 这是标题
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
接上一篇文章，环境已经搭建好，紧接着就是开始创建项目，自动编译工程了。
#自动打包iOS项目
* 点击下图的新建项目

![5E049239-E505-47E3-817F-8FBC8736B5D4.png](http://upload-images.jianshu.io/upload_images/6644906-9726bf015d2cde77.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 输入一个名字，选择构建一个自由风格的项目，然后点击OK

![E8218566-45F0-4B98-86CD-DCB2EBC6CABF.png](http://upload-images.jianshu.io/upload_images/6644906-195089799f629836.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 接下来就是出现如下图，配置git或svn账号，配置Excute shell
     1.　***配置git账号和打包分支***
 　　首先配置的是git账号，由于作者用的gitlab管理源代码，所以选择了git，然后填写Repository URL和Credentials。Credentials点击Add之后选择Username with password,填写自己的账号;最后填写Branches to build。这里我选择了develop分支，这里填写其实非常灵活，有兴趣的可以自行研究一下。
![D607C119-F9D7-497B-9522-E7A582598ABE.png](http://upload-images.jianshu.io/upload_images/6644906-a99870397b8753c0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
最终配置图如下：

![11E5C43A-EB0B-4333-8883-227DBEDC5002.png](http://upload-images.jianshu.io/upload_images/6644906-d98bc56830c552ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
　　2.　***脚本设置***
　　这一步主要用来打包 ipa 并上传到蒲公英。我们点击“增加构建步骤”，选择 "Execute Shell"。输入下列脚本:

    IPANAME="jinkens-myapp"
    fastlane gym --export_method ad-hoc --                     output_name ${IPANAME}
    curl -F "file=@${IPANAME}.ipa" -F    "uKey=USER_KEY" -F "_api_key=API_KEY"    https://qiniu-storage.pgyer.com/apiv1/app/upload

注意：
    　　其中，USER_KEY 和 API_KEY 可以在蒲公英的「账户设置」中找到，之后进行相应替换。
export_method 可以根据打包类型进行相应设置。可选的值有：app-store、ad-hoc、　development、enterprise。对于 Xcode 8.3 以下的版本，则不需要设置 export_method。
设置好之后，类似界面如下所示：
![42B25C98-F734-4401-81DC-A2379AA4B046.png](http://upload-images.jianshu.io/upload_images/6644906-68e35d3aea302a09.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3.　***配置邮箱插件，自动打包之后能够自动发邮件通知***
　　点击系统管理->系统设置 找到“Extended E-mail Notification”，填写相应的信息之后。（注意:“系统管理员邮件地址”要填写发送邮件的账号）


![B8BE5A3A-BAE3-467B-A15E-90996B566480.png](http://upload-images.jianshu.io/upload_images/6644906-ed5de6fe69a856c7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

4.　***配置项目的邮件通知***
　　　点击“增加构建后操作步骤”->"Editable Email Notification"，填写相应的信息如下图:


![F2A2F314-9A91-4A96-9509-7CCAF7348EAC.png](http://upload-images.jianshu.io/upload_images/6644906-e52f63285c4590e8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

　　最后选择Triggers 选择sucees和failure两个即可，之后选择相应的发送人员就好了。
　　保存点击立即构建，见证奇迹的时刻到了。几分钟后则会收到邮件

##邮件表达式详解：##
${BUILD_LOG, maxLines, escapeHtml} -显示最终构建日志。
maxLines – 显示该日志最多显示的行数，默认250行。
escapeHtml -如果为true，格式化HTML。默认false。
${BUILD_LOG_REGEX, regex, linesBefore, linesAfter, maxMatches, showTruncatedLines, substText, escapeHtml, matchedLineHtmlStyle} -按正则表达式匹配显示构建日志的行数。
匹配符合该正则表达式的行数。参阅java.util.regex.Pattern，默认“(?i)\b(error|exception|fatal|fail(ed|ure)|un(defined|resolved))\b”。
linesBefore -包含在匹配行之前的行编号。行数会与当前的另一个行匹配或者linesAfter重叠，默认0。
linesAfter -包含在匹配行之后的行编号。行数会与当前的另一个行匹配或者linesBefore重叠，默认0。
maxMatches -匹配的最大数量，如果为0，则包含所有匹配。默认为0。
showTruncatedLines -如果为true，包含[...truncated ### lines...]行。默认为true。
substText -如果非空，把这部分文字插入该邮件，而不是整行。默认为空。
escapeHtml -如果为true，格式化HTML。默认false。
matchedLineHtmlStyle -如果非空，输出HTML。匹配的行数将变为<b style=”your-style-value”> html escaped matched line </b>格式。默认为空。
${BUILD_NUMBER} -显示当前构建的编号。
${BUILD_STATUS} -显示当前构建的状态(失败、成功等等)
${BUILD_URL} -显示当前构建的URL地址。
${CHANGES, showPaths, format, pathFormat} -显示上一次构建之后的变化。
showPaths – 如果为 true,显示提交修改后的地址。默认false。
format – 遍历提交信息，一个包含%X的字符串，其中%a表示作者，%d表示日期，%m表示消息，%p表示路径，%r表示版本。注意，并不是所有的版本系统都支持%d和%r。如果指定showPaths将被忽略。默认“[%a] %m\n”。
pathFormat -一个包含“%p”的字符串，用来标示怎么打印字符串。
${CHANGES_SINCE_LAST_SUCCESS, reverse, format, showPaths, changesFormat, pathFormat} -显示上一次成功构建之后的变化。
reverse -在顶部标示新近的构建。默认false。
format -遍历构建信息，一个包含%X的字符串，其中%c为所有的改变，%n为构建编号。默认”Changes for Build #%n\n%c\n”。
showPaths, changesFormat, pathFormat – 分别定义如${CHANGES}的showPaths、format和pathFormat参数。
${CHANGES_SINCE_LAST_UNSTABLE, reverse, format, showPaths, changesFormat, pathFormat} -显示显示上一次不稳固或者成功的构建之后的变化。
reverse -在顶部标示新近的构建。默认false。
format -遍历构建信息，一个包含%X的字符串，其中%c为所有的改变，%n为构建编号。默认”Changes for Build #%n\n%c\n”。
showPaths, changesFormat, pathFormat -分别定义如${CHANGES}的showPaths、format和pathFormat参数。
${ENV, var} – 显示一个环境变量。
var – 显示该环境变量的名称。如果为空，显示所有，默认为空。
${FAILED_TESTS} -如果有失败的测试，显示这些失败的单元测试信息。
${JENKINS_URL} -显示Jenkins服务器的地址。(你能在“系统配置”页改变它)。
${HUDSON_URL} -不推荐，请使用$JENKINS_URL
${PROJECT_NAME} -显示项目的名称。
${PROJECT_URL} -显示项目的URL。
${SVN_REVISION} -显示SVN的版本号。
${CAUSE} -显示谁、通过什么渠道触发这次构建。
${JELLY_SCRIPT, template} -从一个Jelly脚本模板中自定义消息内容。有两种模板可供配置：HTML和TEXT。你可以在$JENKINS_HOME/email-templates下自定义替换它。当使用自动义模板时，”template”参数的名称不包含“.jelly”。
template -模板名称，默认”html”。
${FILE, path} -包含一个指定文件的内容
path -文件路径，注意，是工作区目录的相对路径。
${TEST_COUNTS, var} -显示测试的数量。
var – 默认“total”。
total -所有测试的数量。
fail -失败测试的数量。
skip -跳过测试的数量。

下一篇将详细介绍Android的自动打包过程，Android的打包比iOS的打包稍微麻烦一些。
