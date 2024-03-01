# 前言

[blockly官网的构建自定义块工具](https://blockly-demo.appspot.com/static/demos/blockfactory/index.html)已经不可用了(估计服务器挂了)，学了其源码之后，发现，其实它就存在于demos里面。

自己clone项目，然后install安装，运行网页。但是，虽然可以添加块，但是没有预览功能，查看控制台，发现其中有个`prettify.js`的预览文件加载不了，没有梯子的话也不行，这里自己找了一个js进行替换`demos/blockfactory/index.html`里面的`prettify.js`，即可使用了。



[TOC]



# 展示

![image-20200727135057534](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200727135057534.png)



# 使用

[github项目地址](https://github.com/fly7632785/blockly) mine分支

```shell
clone https://github.com/fly7632785/blockly.git
cd 项目
cd demos
npm install --registry=https://registry.npm.taobao.org   (使用淘宝镜像)
```

然后直接浏览器打开，本地文件`demos/blockfactory/index.html`即可使用



# 自定义块

再简单介绍一些这个构建自定义块相关。

一般自定义块是自定义这几个地方：

> 1、块名字
>
> 2、输入
>
> - value input 值输入
> - statement input 块输入
> - dummy input 无输入
>
> 3、输入类型
>
> - external  外接
> - inline 内接
>
> 4、连接方式
>
> - left output左连接（输出）
> - top + bottm connections 上下连接
> - top connection上连接
> - bottom connection 下连接
>
> 5、工具提示
>
> 6、帮助提示
>
> 7、颜色 0-360

![image-20200727140144439](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200727140144439.png)

最基础的模板就是这样的，对应的json为

```json

{
  "type": "my_new_block",
  "message0": "",
  "colour": 230,
  "tooltip": "",
  "helpUrl": ""
}
```

示例：

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



## input输入

![image-20200727140525272](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200727140525272.png)

输入有三种方式：

> 1、value input 值输入（个人理解，起名）
>
> 2、statement input 块输入
>
> 3、dummy input 无输入

![image-20200727144245279](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200727144245279.png)





![image-20200727140830168](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200727140830168.png)

很简单，说明这块块需要一个输入，这个输入默认是any任意类型的值。

**一般有几个输入 `message`里面就有多少个 %1到%n**

如果说你想要给块加入文字，可以这样，直接加入一个纯文本值块即可：

![image-20200727141711509](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200727141711509.png)







## field

`field`也大致分为这几种：

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

所有`field`展示

![image-20200727142321394](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200727142321394.png)



## type

![image-20200727143058107](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200727143058107.png)



## color

![image-20200727143114637](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200727143114637.png)





## 输入类型

- 外接

![image-20200727143402353](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200727143402353.png)

- 内含

![image-20200727143434603](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200727143434603.png)





## connect连接方式

- left output左连接（输出）

![image-20200727143620428](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200727143620428.png)

- top + bottm connections 上下连接

![image-20200727143649514](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200727143649514.png)

- top connection上连接

![image-20200727143702377](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200727143702377.png)

- bottom connection 下连接

![image-20200727143714600](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200727143714600.png)







# 示例

**示例：构建颜色相等**

![image-20200727145048790](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200727145048790.png)

**示例：判等**

![image-20200727145304973](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200727145304973.png)

**示例：无限循环**

![image-20200727145355368](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200727145355368.png)

**示例：电机**

![image-20200727150041851](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200727150041851.png)

**示例：如果**

![image-20200727150628793](https://gitee.com/jafir/blogs/raw/master/2020/images/image-20200727150628793.png)





# 总结

blockly学习，最基础的就是要搞懂如何自定义块，如果你能够通过json格式学会构建规则是什么，固然是好的。但是一开始入手，完全可以通过这个构建工具，来实践不同的效果，从而更好的学习如何自定义块，摸清楚json的规则。





# 关于作者

作者是一个热爱学习、开源、分享，传播正能量，喜欢打篮球、头发还很多的程序员-。-

热烈欢迎大家关注、点赞、评论交流！

简书：https://www.jianshu.com/u/d234d1569eed

github：https://github.com/fly7632785

CSDN：https://blog.csdn.net/fly7632785

掘金：https://juejin.im/user/5efd8d205188252e58582dc7/posts