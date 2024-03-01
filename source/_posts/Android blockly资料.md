# 前言

网上关于Android-blockly的资料很少，自己从事过这方面的学习和开发，所以这里整理分享一下相关的资料，希望可能对大家有所帮助。



[TOC]



# 资料文档

## 官方

[官方文档](https://developers.google.cn/blockly)

[官方github例子](https://github.com/google/blockly-samples)

[官方codelab例子](https://blocklycodelabs.dev)

- [创建一个自定义的块](https://blocklycodelabs.dev/codelabs/custom-generator/index.html?index=..%2F..index#9) （网页blockly相关）

  

## 网上文档相关：

[blockly中文教程（翻译，有几章）](https://www.cnblogs.com/scratch8/p/9637214.html)

[blockly学习（其博客下还有其他相关）](https://www.jianshu.com/p/4c1696b0bb17)

[简书博客相关](https://www.jianshu.com/p/9d62c5d2a52c)

[blockly构建自定义块及其工具](https://www.jianshu.com/p/eb1e0a8e38ee)



# 个人项目介绍

[github项目地址](https://github.com/fly7632785/blockly-android-master/)  请自行下载源码，里面多数代码含有注释，本文简单讲解开发流程及其思路等。

![ble_welcome](https://gitee.com/jafir/blogs/raw/master/2020/images/ble_welcome.png)

![ble_xunji](https://gitee.com/jafir/blogs/raw/master/2020/images/ble_xunji.png)

![ble_control](https://gitee.com/jafir/blogs/raw/master/2020/images/ble_control.png)

![ble_connect](https://gitee.com/jafir/blogs/raw/master/2020/images/ble_connect.png)

![ble_blockly_zhixing](https://gitee.com/jafir/blogs/raw/master/2020/images/ble_blockly_zhixing.png)

![ble_blockly_yundong](https://gitee.com/jafir/blogs/raw/master/2020/images/ble_blockly_yundong.png)

接下来主要给大家介绍一些源码相关及其一些重要的实现。





## app编程与硬件交互流程：

1、app生成自定义块及其基础块，及其自定义块所对应的代码指令

2、自由拼装逻辑代码块后执行

3、生成js代码，自定义代码块的代码指令转义为js自定义方法

4、利用webview的执行js代码

5、js与Android交互（代码转换为指令）

6、Android根据不同的自定义方法发送不同的指令给蓝牙串口（首先保证蓝牙已连接状态）



## blockly自定义块流程：（以src/main/assets/robot为例）

> 1、robot_blocks.json 构建块json
>
> 2、generator.js中构建块所对应的代码（或者指令）
>
> 3、toolbox_basic_robot.xml中添加自定义块 供使用



关于自定义块，还可以参看我的另一篇博客[blockly构建自定义块及其工具](https://www.jianshu.com/p/eb1e0a8e38ee)



![image-20200728110037671](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200728110037671.png)

![image-20200728110150955](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200728110150955.png)



### 几个文件介绍：

`empty.html` 用于webview加载的的空html，主要功能是webview执行js语言的代码

`generators.js` 块代码生成

`robot.js` 类似js执行器

`robot.xml` 默认工作区的块

`robot_blocks.json` 块构建

`toolbox_basic_robot.xml`  工具盒



### 关于蓝牙发送和接收数据：

连接蓝牙 ble4.0 Android系统api封装BleController   注意对应的蓝牙的BleUUID

发送：直接write发送数据（注意数据格式及其长度）

接收：注册接收监听，收到数据，则解析，保存（组合指令）

**发送和接收的数据都是byte[]数组**



### 关于工作区的diy:

`blocklylib-core`和`blocklylib-vertical`module中

1、AbstractBlocklyActivity 相关

2、blockly_unified_workspace.xml 相关

3、default_toolbox_tab.xml 相关

4、default_category_start.xml 相关

5、default_flyout_start.xml相关



### 关于默认的基础逻辑块：

1、基础构建块json

`blocklylib-core/src/main/assets/default`下面

2、基础块生成代码

`blocklylib-core/src/main/assets/javascript_compressed.js`

3、基础块模块的toolbox.xml

`blocklylib-core/src/main/assets/toolbox.xml`



### 构建块json范例：

```
{
  "type": "controls_forever",      块名字
  "message0": "无限循环 %1 ",       块上的文字 及其 参数   如果有多个则为 %1 %2 %3
  "args0": [    参数 数组 可多个
    {
      "type": "input_statement",    参数类型   有很多 常用： field_input field_number field_dropdown  input_value
      "name": "DO"   参数名
    }
  ],
  "colour": "#04B2FF",  颜色
  "inputsInline": true,  是否是输入行内联 （就是通常右边直接接一个输入块）
  "tooltip": "", tool提示
  "helpUrl": "" 帮助提示
  ....其他
  "output": "String", 输出类型   
},
```



### 关于参数类型：

- field类  常用：field_input field_number field_dropdown  field_variable  field_checkbox  field_image（配合src width height等）  内含型 属性值块

> 1、文本
>
> - 常量文本
> - 变量文本
>
> 2、数字
>
> 3、角度
>
> 4、下拉
>
> 5、checkbox
>
> 6、color 颜色
>
> 7、变量
>
> 8、外链图片

![image-20200727142139845](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200727142139845.png)

- value input  常用：

  ```
  "type": "input_value"      外联型 输入块
  ```

- statement input  常用：

  ```
  "type": "input_statement"     添加镶嵌型内部语句块
  ```

- dummy input 常用：

  ```
  "previousStatement": null,
  "nextStatement": null,
  ```

**value input \ statement input \ dummy input 的区别：**

![image-20200727144245279](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200727144245279.png)



# 总结

关于blockly，Google开源的，是少儿编程一个很重要的框架，scratch3.0也是基于blockly二次开发的。现在大多数公司也是用它来进行二次开发的，所以，学习好它是一个最基础也是最重要的。

Android iOS 的 blockly，官方已经放弃维护了，个人在开发和使用中也是发现其移动端有很多的局限性，不是特别适合用来二次开发blockly，反而web是天然优势。

**移动端劣势：**

> 1、需要与webview进行交互，交互和实现局限性较大，有很多东西实现起来较为困难，比如有些交互需要阻塞等待，双向通信等，Android和js交互必然没有web纯天然js开发方便。
>
> 2、手机端屏幕较小，很多块的操作较难，比如有些块里面有下拉或者输入，边缘只有很小的地方供焦点来拖动，操作很不方便，毕竟手指头很大，你需要放大工作区，来实现拖动，太麻烦了。
>
> 3、多端开发，较难维护，本身就是用js开发的，那使用网页来开发是最合适的，并且，现在它也不需要注重太多性能问题，嵌套webview使用就OK了。不必维护Android、ios等多端代码。



# 关于作者

作者是一个热爱学习、开源、分享，传播正能量，喜欢打篮球、头发还很多的程序员-。-

热烈欢迎大家关注、点赞、评论交流！

简书：https://www.jianshu.com/u/d234d1569eed

github：https://github.com/fly7632785

CSDN：https://blog.csdn.net/fly7632785

掘金：https://juejin.im/user/5efd8d205188252e58582dc7/posts