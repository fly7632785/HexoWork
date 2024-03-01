layout: android
title: 写文件生成器的时候换行请用\r\n
date: 2018-01-21 12:34:19
tags:
---
### 前言
最近在学习研究写文件生成器，中间遇到一个问题。
就是，在生成了Activity之后，在Manifest文件中增加一个<activity>的标签，如果我要删除文件，也要从Manifest中清除这个标签。
做法很简单，就是读取清单文件，然后往Application标签中插入一段新增的Activity标签。如果要删除，就读取清单文件，然后把里面这段Activity标签给移除掉。
具体代码描述就是：
1、生成的时候，读取file文件内容，然后indexOf Application标签，然后substring(0 - index） + activity标签代码 + sub（ index - length），然后再写入，这样就增加了代码
2、删除的时候，读取file文件内容，然后直接replace 原来那段Activity标签代码为"" 空字符串
### 问题
逻辑很简单，但是却发现无论如何都没有达到replace的效果，测试contains，也依旧返回false，百思不得其解，调试N次，检查代码以及Activity标签代码，都无误。Android studio用对比工具进行比对也没有任何问题一模一样
![对比1.png](http://upload-images.jianshu.io/upload_images/1311457-1887b413628bdc1f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
最终无奈，只得自己亲自比对string(其实是比如charsequence,一个字符一个字符比对)，这才发现了端倪
![对比2.png](http://upload-images.jianshu.io/upload_images/1311457-edd4441ba799a7d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**原来是Android studio里面的所有文件的回车并不是单单一个"\n"而是"\r\n"**
而我们如果是单单从文件里面copy一段代码出来，然后直接放到string对象里面，转化后的换行就是**\n**
![copy.png](http://upload-images.jianshu.io/upload_images/1311457-6bad0cc02cd0e293.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
所以这种情况，你需要手动加一个**\r**才能解决问题
### 总结
这里再说一下\r 和\n的区别
>\r 回车是将光标移到一行的前面，
>n 是移到下一行,相当于换行。

出个简单的题：
```
        System.out.print("第一行")
        System.out.print("第1段\r")
        System.out.print("1")
        System.out.print("第二行")
        System.out.print("第2段\r\n")
        System.out.print("第三行")
```
打印出来是什么？
直接说答案
```
1第二行第2段
第三行
```
对，你没有看错。
出现这种效果的原因是，/r是回到这行的前面，那么“第二行第二段”就从第一行开始，覆盖了原来的第一行了。
所以，请大家以后记住，对文件的操作，如果是直接copy的，那么记得回车是**\r\n**！
