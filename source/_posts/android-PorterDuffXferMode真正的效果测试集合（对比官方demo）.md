layout: android
title: android PorterDuffXferMode真正的效果测试集合（对比官方demo）
date: 2018-01-21 12:24:37
tags: android
---

##### 前言
当时在做头像imageView，就是切圆头像
1、如果我们先画一个circle（非bitmap），然后setXfermode 为Src_In，再画一个bitmap(图片的)。成功，完美 。

![成功](http://upload-images.jianshu.io/upload_images/1311457-1d790e3f8065c18d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2、如果我们先画一个bitmap(图片的)，然后setXfermode 为Dsr_In,再画circle。粗略学习了网上那张图之后，理论上应该也是成功的，但是却出现了问题。

![不成功](http://upload-images.jianshu.io/upload_images/1311457-cc02fd317d50ee46.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

所以，我很疑惑，就开始了各种探索学习测试。（当然，至于为什么出现这种情况，文末会有解释）

到目前为止已经被PorterDuffXferMode坑了有1天时间了，网上看了无数的文章，很乱很杂，没有写得能够让我很清楚很明白且有点权威的文献，因为很多没有测试结果，并且我没有亲身实践过，所以，现在我要重新自己动手实践一下。

在网上搜罗了一大圈，在群里和很多人交流了，大概有2篇文章，个人认为说得在理。（建议大家先去看一下，不过可能你会跟我一样，看了之后就更云里雾里，但是还是需要亲身实践为好）

感谢两位作者。

[PorterDuffXferMode不正确的真正原因PorterDuffXferMode深入试验](http://m.blog.csdn.net/article/details?id=50534175)

[Android中Canvas绘图之PorterDuffXfermode使用及工作原理详解](http://blog.csdn.net/iispring/article/details/50472485)

**第一篇**主要总结：
如果想让PorterDuffXferMode按照预期Demo（或者效果图）的效果图像实现，必须满足以下条件：
**1、关闭硬件加速。（经过作者修改为 开启硬件离屏缓存）
2、两个bitmap大小尽量一样。
3、背景色为透明色。
4、如果两个bitmap位置不完全一样，可能也是预期效果，只不过你看到的效果和你自己脑补的预期效果不一致。**

**第二篇**主要总结：
**PorterDuffXfermode用于实现新绘制的像素与Canvas上对应位置已有的像素按照混合规则进行颜色混合。**


-----------------
我主要是在第一篇的结论基础上去做测试，并给大家展示测试结果。
下面高能篇幅，如果你不是真正在学习研究PorterDuffXfermode踩坑的人，如果你没有耐心阅读的，就可以别往下看了。
亦或你也想亲手尝试，那还是先去尝试一下吧。

-------

首先代码图：
```
public class TestXfermodeView extends View {
    public TestXfermodeView(Context context) {
        super(context);
    }

    public TestXfermodeView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    public TestXfermodeView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }


    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        Bitmap circle = getCircleBitmap();
        Bitmap rectangle = getRetangleBitmap();

//        int sc = canvas.saveLayer(0, 0, 400, 400, null,
//                Canvas.MATRIX_SAVE_FLAG |
//                        Canvas.CLIP_SAVE_FLAG |
//                        Canvas.HAS_ALPHA_LAYER_SAVE_FLAG |
//                        Canvas.FULL_COLOR_LAYER_SAVE_FLAG |
//                        Canvas.CLIP_TO_LAYER_SAVE_FLAG);

        /**
         * 开启硬件离屏缓存
         */
        setLayerType(LAYER_TYPE_HARDWARE, null);
        Paint paint = new Paint();
        /**
         * 画bitmap的也透明
         */
        canvas.drawARGB(0, 0, 0, 0);
//        canvas.drawCircle(100, 100, 100, paint);
        canvas.drawBitmap(rectangle, 100, 100, paint);
//        Bitmap b= BitmapFactory.decodeResource(getResources(),R.mipmap.ic_launcher);
//        Rect rect = new Rect(0, 0, 100, 100);
//        canvas.drawBitmap(b,rect, rect, paint);
//        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.SRC));
//        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.SRC_IN));
//        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.SRC_OUT));
//        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.SRC_ATOP));
//        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.SRC_OVER));

//        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.DST));
//        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.DST_IN));
//        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.DST_OUT));
//        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.DST_ATOP));
//        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.DST_OVER));

//        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.XOR));
//        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.CLEAR));
//        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.ADD));
//        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.MULTIPLY));
//        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.DARKEN));
//        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.LIGHTEN));
//        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.OVERLAY));
//        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.SCREEN));
        canvas.drawBitmap(circle, 0, 0, paint);
//        canvas.restoreToCount(sc);

    }

    @NonNull
    private Bitmap getRetangleBitmap() {
        /**
         * bm1 在bitmap上面画正方形
         */
        Bitmap rectangle = Bitmap.createBitmap(200, 200, Bitmap.Config.ARGB_8888);
        Canvas c1 = new Canvas(rectangle);
        Paint p1 = new Paint(Paint.ANTI_ALIAS_FLAG);
        p1.setColor(getResources().getColor(R.color.colorAccent));
        /**
         * 设置透明
         */
        c1.drawARGB(0, 0, 0, 0);
        c1.drawRect(0, 0, 200, 200, p1);
        return rectangle;
    }

    @NonNull
    private Bitmap getCircleBitmap() {
        /**
         * bm 在bitmap上面画圆
         */
        Bitmap circle = Bitmap.createBitmap(200, 200, Bitmap.Config.ARGB_8888);
        Canvas c = new Canvas(circle);
        /**
         * 设置透明
         */
        c.drawARGB(0, 0, 0, 0);
        Paint p = new Paint(Paint.ANTI_ALIAS_FLAG);
        p.setColor(getResources().getColor(R.color.colorPrimary));
        c.drawCircle(100, 100, 100, p);
        return circle;
    }
}
```
这俩方法 就是在2个新的bitmap上画圆和正方形
注意是跟官方demo一致 ** 先画 正方形 后画 圆**
满足条件
**2、两个bitmap大小尽量一样。
3、背景色为透明色。**

onDraw中也设置了透明和开启硬件离屏缓存
```
 setLayerType(LAYER_TYPE_HARDWARE, null);
        Paint paint = new Paint();
        /**
         * 画bitmap的也透明
         */
        canvas.drawARGB(0, 0, 0, 0);
```
满足条件
**1、开启硬件离屏缓存**（顺便说一下它的好处）
    - 1.解决xfermode黑色问题。
    - 2.效率比关闭硬件加速高3倍以上


### 好正式开始 Xfermode的条件测试：

```
paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.SRC));
```
![PorterDuff.Mode.SRC](http://upload-images.jianshu.io/upload_images/1311457-754c0b6d61b351f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看出圆的canvas背景（透明，activity本身就是白色，所以这里为白色）也显示出来并且覆盖在了正方形上面

```
 paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.DST));
```
![PorterDuff.Mode.DST](http://upload-images.jianshu.io/upload_images/1311457-0613ecff661625fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看出后画的圆已经不显示了

```
paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.SRC_IN));
```
![PorterDuff.Mode.SRC_IN](http://upload-images.jianshu.io/upload_images/1311457-2cb7351f70b49922.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看出圆的**canvas背景显示出来**， 只取了与正方形**相交的部分**（是canvas背景区域相交的部分），并且相交部分显示的是**后画**的**圆**的颜色

```
 paint.setXfermode(newPorterDuffXfermode(PorterDuff.Mode.DST_IN));
```
![PorterDuff.Mode.DST_IN](http://upload-images.jianshu.io/upload_images/1311457-969d4479ee7371f5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看出圆的**canvas背景显示出来**， 只取了与正方形**相交的部分**（是canvas背景区域相交的部分），并且相交部分显示的是**先画**的**正方形**的颜色
```
 paint.setXfermode(newPorterDuffXfermode(PorterDuff.Mode.XOR));
```
![PorterDuff.Mode.XOR](http://upload-images.jianshu.io/upload_images/1311457-354d41f24e1bafca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看出取的是**相交部分之外**，并且与官方demo效果一致
```
 paint.setXfermode(newPorterDuffXfermode(PorterDuff.Mode.CLEAR));
```
![PorterDuff.Mode.CLEAR](http://upload-images.jianshu.io/upload_images/1311457-ac3954a3a64a9bfe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看出**后画**的**圆**不见了 ，并且**相交的部分也不见了**

这前面几个是相对比较常用，也是比较重要的。

后面的我直接列出来


![PorterDuff.Mode.SRC_OUT](http://upload-images.jianshu.io/upload_images/1311457-2d9284eef84de8fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![PorterDuff.Mode.SRC_ATOP](http://upload-images.jianshu.io/upload_images/1311457-414dc08dcc4d67a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![PorterDuff.Mode.SRC_OVER](http://upload-images.jianshu.io/upload_images/1311457-1a2aa3a16595f547.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![PorterDuff.Mode.DST_OUT](http://upload-images.jianshu.io/upload_images/1311457-68b79dda92a19db6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![PorterDuff.Mode.DST_ATOP](http://upload-images.jianshu.io/upload_images/1311457-eb5219787ee38ce0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![PorterDuff.Mode.DST_OVER](http://upload-images.jianshu.io/upload_images/1311457-f3dc6a1420311882.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![PorterDuff.Mode.ADD](http://upload-images.jianshu.io/upload_images/1311457-4b0087f29d24e930.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



![PorterDuff.Mode.MUTIPLY](http://upload-images.jianshu.io/upload_images/1311457-9650b3ed79cccab9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![PorterDuff.Mode.DARKEN](http://upload-images.jianshu.io/upload_images/1311457-f5c46b9049166e04.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![PorterDuff.Mode.OVERPLAY](http://upload-images.jianshu.io/upload_images/1311457-90123e62d6385489.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![PorterDuff.Mode.SCREEN](http://upload-images.jianshu.io/upload_images/1311457-b9c3af1f7e50c5e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 总览全局

![官方demo](http://upload-images.jianshu.io/upload_images/1311457-a84859d50c0765e3?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)

![我的](http://upload-images.jianshu.io/upload_images/1311457-1758f01d16feb3b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)

![第二篇文献里面的](http://upload-images.jianshu.io/upload_images/1311457-4641819c69a2b5fa?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)

注：我的是跟官方demo一样先画正方形后画圆，第二篇文献里面的图是先画圆后画正方形，可以对比一下，体会一下。

我说说我们的demo和官方api demo的区别：
- 我的demo里面，两个bitmap大小一样，是在bitmap里面填充画满 circle和rectangle,并且在画两个Bitmap的时候，是调整的它的位置来画的；
- 官方的demo里面，是两个bitmap大小一样，circle的bitmap里面只填充了 左上角2/3，而rectangle的bitmap里面只填充了 右下角的2/3，并且在画的时候，两个bitmap的位置大小都是一样的

我的


![circle的bitmap](http://upload-images.jianshu.io/upload_images/1311457-e82d8c4b1c4e7c49.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![rectangle的bitmap](http://upload-images.jianshu.io/upload_images/1311457-683334825b5b9b60.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
canvas.drawBitmap(rectangle,**100, 100**, paint);
canvas.drawBitmap(circle, 0, 0, paint);
```

官方的
![circle的bitmap](http://upload-images.jianshu.io/upload_images/1311457-0ac2c837454b7b6f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![rectangle的bitmap](http://upload-images.jianshu.io/upload_images/1311457-9abd3d62e2f26c27.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
canvas.drawBitmap(mSrcB, **0, 0**, paint);
canvas.drawBitmap(mDstB, 0, 0, paint);
```

大家可以再去对比对比两张总结图，思考思考。
如果想要探寻为什么不同的方法会导致不同的效果，也可以去阅读页首的第二篇文章

### 总结
如果你想要做出实际效果，那么你要按照官方的那种方式，你就能够做出跟网上普遍流传的那张图一样的效果。

如果你要根据自己的实际情况来，那么你可能就要考虑我的这种方式和官方的那种方式来决定怎么做了。

至于文首的问题，**原因如下：**
因为我们的Xfermode 叠合裁剪，都是建立在不同的层级上，重新画一个bitmap会新开一层。
第一种：先画circle 在canvas那层，再画Bitmap，新开了一层，中间镶嵌Xfermode，成功。
第二种:   先画bitmap，新开了一层，再画circle，还是在bitmap那层，中间镶嵌 Xfermode,不成功。

**解决方案：**
```
 //第一种
        
        canvas.drawCircle(scaleBitmap.getWidth() / 2, scaleBitmap.getHeight() / 2, scaleBitmap.getWidth() / 2, paint);

        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.DST_IN));

        canvas.drawBitmap(scaleBitmap, rect, rect1, paint);
```

```
//第二种
        canvas.drawBitmap(scaleBitmap, rect, rect1, paint);

        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.DST_IN));
      
        Bitmap bitmap = Bitmap.createBitmap(scaleBitmap.getWidth(), scaleBitmap.getHeight(), Bitmap.Config.ARGB_8888);
        Canvas canvas1 = new Canvas(bitmap);
        Paint p = new Paint();
        canvas1.drawARGB(0, 0, 0, 0);
        canvas1.drawCircle(scaleBitmap.getWidth() / 2, scaleBitmap.getHeight() / 2, scaleBitmap.getWidth() / 2, p);
        canvas.drawBitmap(bitmap, rect, rect1, paint);
```

**ps:** 
写这篇文章呢，主要是我个人想要实践一下，顺便记录一下实验结果。最后也顺便分享出来，个人感觉这里确实有很多坑，如果你是真正在研究PorterDuffXformode的人，那你肯定会跟我一样很疑惑，希望这篇文章能够带给你一个思路。

