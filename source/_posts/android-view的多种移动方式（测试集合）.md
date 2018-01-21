layout: android
title: android view的多种移动方式（测试集合）
date: 2018-01-21 12:28:00
tags: android
---
### 前言
由于最近在写一个涉及移动方面的自定义View，在做移动的时候用到了类似offsetTopAndBottom 、setTranslationY、scrollTo、scrollBy等方法，对于他们的使用，有一些不太清晰的地方。比如，view的getX/Y、getSrollX/Y、getTranslationX/Y、getLeft/top/right/bottom、点击事件触发区域等等 是否会受到影响改变，由哪些所影响。

因为View的属性有点多，所以，很多时候你不知道哪些属性受哪些方法影响，并且多种方法联合使用的时候，效果又是如何，影响又是如何。

### 正题
于是我就开始测试，实践来检验结果。

目前为止大致有这几种方法可以移动view:
**1、setTranslationX/Y
2、scrollTo/scrollBy
3、offsetTopAndBottom/offsetLeftAndRight
4、平移动画
5、设置margin
**

主要是验证一些属性：
**1、getX()、getY()
2、getScrollX() 、getScrollY()
3、getTranslationX() 、getTranslationY()
4、getLeft()、 getTop()、 getRight()、 getBottom()（坐标位置是否改变）
5、点击事件触发区域是否改变
6、是否会影响同层级的其他view的位置
7、超过父View是否绘制**


现在主要把他们用一张表列出来：



![](http://upload-images.jianshu.io/upload_images/1311457-48e8976caad5939c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 稍微整理一下他们各自特点：
###### setTranslationX/Y
- getX getY 会变
- getTranslationXY会变 
- 点击事件的位置也变了但是不会超过父布局
- 会超过边界到同级View的区域去（被覆盖或者覆盖别人）
- 这个方法的底层实现主要是通过metrix矩阵变换来的，坐标位置没有改变（跟offset不同，它是通过坐标位置改变）

###### scrollTo/scrollBy
- getScrollXY 会变 
- 点击事件还是在原位置 （跟动画类似）
- 但是内容区域变了（如果超出自己的区域 就显示不出来）
- 它只是内容区域的移动，本身view是不移动的
- scrollBy的x y 是相对移动的值
- scrollTo的x y 是绝对移动的值

###### offsetTopAndBottom/offsetLeftAndRight
- 上下左右坐标会变 （主要是通过坐标位置的改变产生移动效果）
- getXY会变 
- 点击事件的位置也变了
- 会超过边界到别人的区域去（被覆盖或者覆盖别人）
- 它的offY是相对移动的值

###### 平移动画
- 点击事件还是在原位置
- 如果setFillAfter位置保留 但是其他任何坐标位置没有改变 再次点击从原位置重新开始移动

###### 设置margin
- 如果父View为wrap的话，设置margin可以移动，但是可能会对同级view造成影响（比如在linear中或者relative中有关联关系）


#### 下面是验证过程：（前方高能，多图预警！！！！！最重要的东西都罗列在前面了，没时间不用往下看了）

![](http://upload-images.jianshu.io/upload_images/1311457-ff7b4f4cbb30765d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)


![默认情况log](http://upload-images.jianshu.io/upload_images/1311457-860436f7fd2ec6b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)


##### setTranslationXY:

![](http://upload-images.jianshu.io/upload_images/1311457-49e1cf0edfa63775.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)

![](http://upload-images.jianshu.io/upload_images/1311457-d95b41e650dd9bc2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)


![指定了父布局](http://upload-images.jianshu.io/upload_images/1311457-0abd4457d3be1261.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)


![不能超过父布局，会显示不出来](http://upload-images.jianshu.io/upload_images/1311457-e9e76e59fe4a8458.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)


![会超过边界到同层view的区域去（被覆盖或者覆盖别人）](http://upload-images.jianshu.io/upload_images/1311457-95fb206b842006dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)

- getX getY 会变
- getTranslationXY会变 
- 点击事件的位置也变了但是不会超过父布局
- 会超过边界到同层view的区域去（被覆盖或者覆盖别人--取决于先后顺序）


##### scrollBy:

![](http://upload-images.jianshu.io/upload_images/1311457-52ba8ec93da93e22.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)

![](http://upload-images.jianshu.io/upload_images/1311457-c2db3a444192d12e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)

##### offsetTopAndBottom offsetLeftAndRight:


![](http://upload-images.jianshu.io/upload_images/1311457-2f3913f3dbde5552.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)


![](http://upload-images.jianshu.io/upload_images/1311457-5435c616ee7e5d92.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)

##### 动画+setFillAfter(true):


![](http://upload-images.jianshu.io/upload_images/1311457-90d34cbd7e59343f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)

![](http://upload-images.jianshu.io/upload_images/1311457-5435c616ee7e5d92.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)



##### margin:

![](http://upload-images.jianshu.io/upload_images/1311457-d9c9b0b400b88616.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)
![](http://upload-images.jianshu.io/upload_images/1311457-5435c616ee7e5d92.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)

## 组合
---
#### 比如先多点几次 offset ，然后再margin会立马回到（原位置+margin）后的状态 

![](http://upload-images.jianshu.io/upload_images/1311457-fed0c848512981f2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)

![](http://upload-images.jianshu.io/upload_images/1311457-2e3045b1227ece07.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)

 **说明：**
 margin的平移效果是以view在父View中的位置和margin值决定的，是父View通过计算margin值之后，重新给你排的位置，实现的移动。当我们设置margin之后，会触发requestLayout,所以父VIew又重新给它排了位置。

----

#### 如果，我先offset几次，然后再点击动画，动画会在原来的基础上，继续平移。

  **说明：**
  动画不是根据位置来移动的，可能是根据一个metrix的矩阵变换来实现平移的（请指正）

#### 如果，先scrollBy，然后再动画、offset和其他移动方法，

![](http://upload-images.jianshu.io/upload_images/1311457-d45bbde6af792a2f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)


![](http://upload-images.jianshu.io/upload_images/1311457-2855f1a5a6aaf40f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)
  **说明：**
  其他的平移方法，都是对于view本身在做移动，而不像scrollBy只是对其内容进行平移




## 总结
好了，差多就这些了，其他更多复杂的组合使用，只要你能逐一弄清楚他们各自起作用的属性和对象，你就能大致摸索出来。
剩下的大家可以去[demo](https://github.com/fly7632785/MyBookExplore)看看，然后自己试一试。



