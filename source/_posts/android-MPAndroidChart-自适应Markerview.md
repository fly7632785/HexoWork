---
title: Android MPAndroidChart 自适应Markerview
date: 2018-01-21 12:32:52
tags: android
---
### 前言
Android里面只要用过图表的应该都知道[MPAndroidChart](https://github.com/PhilJay/MPAndroidChart)这个库。这个库在iOS里面也有对应[Charts](https://github.com/danielgindi/Charts)，所以一般移动端做图表，Android和iOS两端都要实现同样的效果，他们是不错的一个选择。
但是，对于图表这种包含的情况非常复杂的东西，很难满足大家各种各样的需求，所以很多都需要自定义。下面就是给大家分享一下自己写的自适应MarkerView。
#### 先上图

![正常居中显示](http://upload-images.jianshu.io/upload_images/1311457-d5d185af3b7037e2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![超过左边界](http://upload-images.jianshu.io/upload_images/1311457-84f1f978ac4211b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![超过上边界](http://upload-images.jianshu.io/upload_images/1311457-faa4dd3a11715f51.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![超过右边界](http://upload-images.jianshu.io/upload_images/1311457-e2e240b518251db1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![超过上边界和右边界](http://upload-images.jianshu.io/upload_images/1311457-6dcbf615e7957d05.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 正文
要实现这样的效果，大家先想想怎么做？
这里的逻辑步骤分为：
1、创建新类继承自MarkerView
2、inflate layout进去
3、重写getOffsetForDrawingAtPoint ，分别处理各种边界情况的偏移
4、重写draw，绘制底色，根据不同的情况，绘制带箭头的对话框

##### 直接上代码
```
public class XYMarkerView extends MarkerView {
    public static final int ARROW_SIZE = 40; // 箭头的大小
    private static final float CIRCLE_OFFSET = 10;//因为我这里的折点是圆圈，所以要偏移，防止直接指向了圆心
    private static final float STOKE_WIDTH = 5;//这里对于stroke_width的宽度也要做一定偏移
    private final TextView tvContent;
    private final RoundImageView avatar;
    private final TextView name;
    private final List<StepListModel> stepListModels;
    private int index;
    private int oldIndex = -1;

    public XYMarkerView(Context context, List<StepListModel> stepListModels) {
        super(context, R.layout.custom_marker_view);
        tvContent = (TextView) findViewById(R.id.tvContent);
        avatar = (RoundImageView) findViewById(R.id.avatar);
        name = (TextView) findViewById(R.id.name);
        this.stepListModels = stepListModels;
    }

    @Override
    public void refreshContent(Entry e, Highlight highlight) {
        super.refreshContent(e, highlight);
        index = highlight.getDataSetIndex();//这个方法用于获得折线是哪根
        tvContent.setText((int) e.getY() + "");
//            StepListModel stepListModel = stepListModels.get(highlight.getDataSetIndex() % Constants.battleUsersCount);
//            name.setText(stepListModel.getNickNm());
//            Glide.with(getContext())
//                    .load(GlideUtil.getGlideUrl(stepListModel.getIconUrl()))
//                    .into(avatar);
        Glide.with(getContext())
                .asBitmap()
                .load(avatars[index % avatars.length])
                .listener(new RequestListener<Bitmap>() {
                    @Override
                    public boolean onLoadFailed(@Nullable GlideException e, Object model, Target<Bitmap> target, boolean isFirstResource) {
                        return false;
                    }

                    @Override
                    public boolean onResourceReady(Bitmap resource, Object model, Target<Bitmap> target, DataSource dataSource, boolean isFirstResource) {
                        if (resource != null) {
                            if (oldIndex != index) {
                                XYMarkerView.this.getChartView().invalidate();
                                oldIndex = index;
                            }
                            avatar.setImageBitmap(resource);
                        }
                        return false;
                    }
                })
                .into(avatar);
        name.setText(highlight.getDataSetIndex() + "");
        tvContent.setTextColor(getResources().getColor(ColorUtil.colors[highlight.getDataSetIndex() % ColorUtil.colors.length]));
        LogUtil.m("getDataSetIndex" + highlight.getDataSetIndex());

    }

    @Override
    public MPPointF getOffsetForDrawingAtPoint(float posX, float posY) {
        MPPointF offset = getOffset();
        Chart chart = getChartView();
        float width = getWidth();
        float height = getHeight();
// posY \posX 指的是markerView左上角点在图表上面的位置
//处理Y方向
        if (posY <= height + ARROW_SIZE) {// 如果点y坐标小于markerView的高度，如果不处理会超出上边界，处理了之后这时候箭头是向上的，我们需要把图标下移一个箭头的大小
            offset.y = ARROW_SIZE;
        } else {//否则属于正常情况，因为我们默认是箭头朝下，然后正常偏移就是，需要向上偏移markerView高度和arrow size，再加一个stroke的宽度，因为你需要看到对话框的上面的边框
            offset.y = -height - ARROW_SIZE - STOKE_WIDTH; // 40 arrow height   5 stroke width
        }
//处理X方向，分为3种情况，1、在图表左边 2、在图表中间 3、在图表右边
//
        if (posX > chart.getWidth() - width) {//如果超过右边界，则向左偏移markerView的宽度
            offset.x = -width;
        } else {//默认情况，不偏移（因为是点是在左上角）
            offset.x = 0;
            if (posX > width / 2) {//如果大于markerView的一半，说明箭头在中间，所以向右偏移一半宽度
                offset.x = -(width / 2);
            }
        }
        return offset;
    }

    @Override
    public void draw(Canvas canvas, float posX, float posY) {
        Paint paint = new Paint();//绘制边框的画笔
        paint.setStrokeWidth(STOKE_WIDTH);
        paint.setStyle(Paint.Style.STROKE);
        paint.setStrokeJoin(Paint.Join.ROUND);
        paint.setColor(getResources().getColor(ColorUtil.colors[index % ColorUtil.colors.length]));

        Paint whitePaint = new Paint();//绘制底色白色的画笔
        whitePaint.setStyle(Paint.Style.FILL);
        whitePaint.setColor(Color.WHITE);

        Chart chart = getChartView();
        float width = getWidth();
        float height = getHeight();

        MPPointF offset = getOffsetForDrawingAtPoint(posX, posY);
        int saveId = canvas.save();

        Path path = new Path();
        if (posY < height + ARROW_SIZE) {//处理超过上边界
            path = new Path();
            path.moveTo(0, 0);
            if (posX > chart.getWidth() - width) {//超过右边界
                path.lineTo(width - ARROW_SIZE, 0);
                path.lineTo(width, -ARROW_SIZE + CIRCLE_OFFSET);
                path.lineTo(width, 0);
            } else {
                if (posX > width / 2) {//在图表中间
                    path.lineTo(width / 2 - ARROW_SIZE / 2, 0);
                    path.lineTo(width / 2, -ARROW_SIZE + CIRCLE_OFFSET);
                    path.lineTo(width / 2 + ARROW_SIZE / 2, 0);
                } else {//超过左边界
                    path.lineTo(0, -ARROW_SIZE + CIRCLE_OFFSET);
                    path.lineTo(0 + ARROW_SIZE, 0);
                }
            }
            path.lineTo(0 + width, 0);
            path.lineTo(0 + width, 0 + height);
            path.lineTo(0, 0 + height);
            path.lineTo(0, 0);
            path.offset(posX + offset.x, posY + offset.y);
        } else {//没有超过上边界
            path = new Path();
            path.moveTo(0, 0);
            path.lineTo(0 + width, 0);
            path.lineTo(0 + width, 0 + height);
            if (posX > chart.getWidth() - width) {
                path.lineTo(width, height + ARROW_SIZE - CIRCLE_OFFSET);
                path.lineTo(width - ARROW_SIZE, 0 + height);
                path.lineTo(0, 0 + height);
            } else {
                if (posX > width / 2) {
                    path.lineTo(width / 2 + ARROW_SIZE / 2, 0 + height);
                    path.lineTo(width / 2, height + ARROW_SIZE - CIRCLE_OFFSET);
                    path.lineTo(width / 2 - ARROW_SIZE / 2, 0 + height);
                    path.lineTo(0, 0 + height);
                } else {
                    path.lineTo(0 + ARROW_SIZE, 0 + height);
                    path.lineTo(0, height + ARROW_SIZE - CIRCLE_OFFSET);
                    path.lineTo(0, 0 + height);
                }
            }
            path.lineTo(0, 0);
            path.offset(posX + offset.x, posY + offset.y);
        }

        // translate to the correct position and draw
        canvas.drawPath(path, whitePaint);
        canvas.drawPath(path, paint);
        canvas.translate(posX + offset.x, posY + offset.y);
        draw(canvas);
        canvas.restoreToCount(saveId);
    }
}
```


### 详细

#####  不同颜色、头像等处理
我这里的需求是根据不同的折线，显示不同的人物头像、名字和对应的值，并且对话框的颜色、字体都要对应折线的颜色。
所以，我这里根据```highlight.getDataSetIndex() ```就能处理不同颜色、头像等信息的情况。
```
//类似这个
 name.setText(highlight.getDataSetIndex() + "");
 tvContent.setTextColor(getResources().getColor(ColorUtil.colors[highlight.getDataSetIndex() % ColorUtil.colors.length]));
```
注意：
这里有个地方要注意一下，这里如果你使用Glide来加载图片到ImageView里去，会出现第一次加载不出来头像，第二次点击的时候就加载出来了。
原因 ：是因为其实glide已经加载出来了，然后只是异步加载出来之后，已经是在layout绘制完成之后了，并没有进行invalidate的刷新。所以直到第二次的时候，其实glide已经加载过了，有缓存，所以直接就显示了，发生在draw方法之前，因为refreshContent就在draw之前调用
```

            // callbacks to update the content
            mMarker.refreshContent(e, highlight);

            // draw the marker
            mMarker.draw(canvas, pos[0], pos[1]);
```
我这里是直接在glide的onResourceReady里面调用chart的invalidate，但是这样的话会一直循环刷新了，卡死！
所以，这里使用了一个oldIndex是否等于index来判断是否是已经invalidate，这样的话就只有点击不同的折线的时候会invalidate一次，之后就不会了。
```
  private int index;
  private int oldIndex = -1;
  ...
  Glide.with(getContext())
                .asBitmap()
                .load(avatars[index % avatars.length])
                .listener(new RequestListener<Bitmap>() {
                    @Override
                    public boolean onLoadFailed(@Nullable GlideException e, Object model, Target<Bitmap> target, boolean isFirstResource) {
                        return false;
                    }

                    @Override
                    public boolean onResourceReady(Bitmap resource, Object model, Target<Bitmap> target, DataSource dataSource, boolean isFirstResource) {
                        if (resource != null) {
                            if (oldIndex != index) {
                                XYMarkerView.this.getChartView().invalidate();
                                oldIndex = index;
                            }
                            avatar.setImageBitmap(resource);
                        }
                        return false;
                    }
                })
                .into(avatar);
```

##### 边界情况的处理
MarkerView的draw方法，里面有posX、posY，这个点坐标代表的是markerView的左上角的坐标（如果不偏移的情况下，官方的默认也是做了偏移处理的，但是还有不够完善）。


![默认是在左上角的，像这样](http://upload-images.jianshu.io/upload_images/1311457-0fb26dedb2ee3beb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```getOffsetForDrawingAtPoint```就是处理偏移的方法，最终应用是在draw里面，进行canvas的translate变换
```
 @Override
    public void draw(Canvas canvas, float posX, float posY) {

        MPPointF offset = getOffsetForDrawingAtPoint(posX, posY);

        int saveId = canvas.save();
        // translate to the correct position and draw
        canvas.translate(posX + offset.x, posY + offset.y);
        draw(canvas);
        canvas.restoreToCount(saveId);
    }
```
下面是我对各种边界情况的处理
```
     @Override
    public MPPointF getOffsetForDrawingAtPoint(float posX, float posY) {
        MPPointF offset = getOffset();
        Chart chart = getChartView();
        float width = getWidth();
        float height = getHeight();
// posY \posX 指的是markerView左上角点在图表上面的位置
//处理Y方向
        if (posY <= height + ARROW_SIZE) {// 如果点y坐标小于markerView的高度，如果不处理会超出上边界，处理了之后这时候箭头是向上的，我们需要把图标下移一个箭头的大小
            offset.y = ARROW_SIZE;
        } else {//否则属于正常情况，因为我们默认是箭头朝下，然后正常偏移就是，需要向上偏移markerView高度和arrow size，再加一个stroke的宽度，因为你需要看到对话框的上面的边框
            offset.y = -height - ARROW_SIZE - STOKE_WIDTH; // 40 arrow height   5 stroke width
        }
//处理X方向，分为3种情况，1、在图表左边 2、在图表中间 3、在图表右边
//
        if (posX > chart.getWidth() - width) {//如果超过右边界，则向左偏移markerView的宽度
            offset.x = -width;
        } else {//默认情况，不偏移（因为是点是在左上角）
            offset.x = 0;
            if (posX > width / 2) {//如果大于markerView的一半，说明箭头在中间，所以向右偏移一半宽度
                offset.x = -(width / 2);
            }
        }
        return offset;
    }
```

![处理左边界](http://upload-images.jianshu.io/upload_images/1311457-a9f533f780593dcd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![处理一般情况](http://upload-images.jianshu.io/upload_images/1311457-c786c8d1e97c52ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![处理上边界](http://upload-images.jianshu.io/upload_images/1311457-ca258b7c1f605dfc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![处理右边界](http://upload-images.jianshu.io/upload_images/1311457-61b9d9ab08f5a30e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 对于各个箭头的处理
实现这个对话框，主要是通过绘制path来实现的，箭头的处理也是根据不同的情况，进行不同的path绘制实现的。
```
 @Override
    public void draw(Canvas canvas, float posX, float posY) {
        Paint paint = new Paint();//绘制边框的画笔
        paint.setStrokeWidth(STOKE_WIDTH);
        paint.setStyle(Paint.Style.STROKE);
        paint.setStrokeJoin(Paint.Join.ROUND);
        paint.setColor(getResources().getColor(ColorUtil.colors[index % ColorUtil.colors.length]));

        Paint whitePaint = new Paint();//绘制底色白色的画笔
        whitePaint.setStyle(Paint.Style.FILL);
        whitePaint.setColor(Color.WHITE);

        Chart chart = getChartView();
        float width = getWidth();
        float height = getHeight();

        MPPointF offset = getOffsetForDrawingAtPoint(posX, posY);
        int saveId = canvas.save();

        Path path = new Path();
        if (posY < height + ARROW_SIZE) {//处理超过上边界
            path = new Path();
            path.moveTo(0, 0);
            if (posX > chart.getWidth() - width) {//超过右边界
                path.lineTo(width - ARROW_SIZE, 0);
                path.lineTo(width, -ARROW_SIZE + CIRCLE_OFFSET);
                path.lineTo(width, 0);
            } else {
                if (posX > width / 2) {//在图表中间
                    path.lineTo(width / 2 - ARROW_SIZE / 2, 0);
                    path.lineTo(width / 2, -ARROW_SIZE + CIRCLE_OFFSET);
                    path.lineTo(width / 2 + ARROW_SIZE / 2, 0);
                } else {//超过左边界
                    path.lineTo(0, -ARROW_SIZE + CIRCLE_OFFSET);
                    path.lineTo(0 + ARROW_SIZE, 0);
                }
            }
            path.lineTo(0 + width, 0);
            path.lineTo(0 + width, 0 + height);
            path.lineTo(0, 0 + height);
            path.lineTo(0, 0);
            path.offset(posX + offset.x, posY + offset.y);
        } else {//没有超过上边界
            path = new Path();
            path.moveTo(0, 0);
            path.lineTo(0 + width, 0);
            path.lineTo(0 + width, 0 + height);
            if (posX > chart.getWidth() - width) {
                path.lineTo(width, height + ARROW_SIZE - CIRCLE_OFFSET);
                path.lineTo(width - ARROW_SIZE, 0 + height);
                path.lineTo(0, 0 + height);
            } else {
                if (posX > width / 2) {
                    path.lineTo(width / 2 + ARROW_SIZE / 2, 0 + height);
                    path.lineTo(width / 2, height + ARROW_SIZE - CIRCLE_OFFSET);
                    path.lineTo(width / 2 - ARROW_SIZE / 2, 0 + height);
                    path.lineTo(0, 0 + height);
                } else {
                    path.lineTo(0 + ARROW_SIZE, 0 + height);
                    path.lineTo(0, height + ARROW_SIZE - CIRCLE_OFFSET);
                    path.lineTo(0, 0 + height);
                }
            }
            path.lineTo(0, 0);
            path.offset(posX + offset.x, posY + offset.y);
        }

        // translate to the correct position and draw
        canvas.drawPath(path, whitePaint);
        canvas.drawPath(path, paint);
        canvas.translate(posX + offset.x, posY + offset.y);
        draw(canvas);
        canvas.restoreToCount(saveId);
    }
```
绘制MarkerView分为两部分，一部分是绘制对话框，一部分是绘制传入的R.layout.xx的view。


![类似这样](http://upload-images.jianshu.io/upload_images/1311457-6955c8a84ac1ddb2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这里有一个问题，就是需要填充对话框白色，并且还要有边框。
一个画笔是不够的，所以这里有两个画笔，一个是填充的，一个是画边框的。
并且，是必须要先绘制对话框的底色，绘制对话框，再绘制传入的view
```
   Paint paint = new Paint();//绘制边框的画笔
        paint.setStrokeWidth(STOKE_WIDTH);
        paint.setStyle(Paint.Style.STROKE);
        paint.setStrokeJoin(Paint.Join.ROUND);
        paint.setColor(getResources().getColor(ColorUtil.colors[index % ColorUtil.colors.length]));

        Paint whitePaint = new Paint();//绘制底色白色的画笔
        whitePaint.setStyle(Paint.Style.FILL);
        whitePaint.setColor(Color.WHITE);
```
```
// 第一步        canvas.drawPath(path, whitePaint); 
// 第二步        canvas.drawPath(path, paint); 
                canvas.translate(posX + offset.x, posY + offset.y); 
// 第三步        draw(canvas);
```
因为我们的markerView偏移是对于canvas的偏移，但是我们的对话框的path并没有偏移，所以我们也要对path进行同样的偏移处理。很简单，直接获取偏移值，然后偏移就好。
```
   MPPointF offset = getOffsetForDrawingAtPoint(posX, posY);
  path.offset(posX + offset.x, posY + offset.y);
```
### 最后

这样的最终效果就是传入的layout 被对话框包裹在里面了。

![](http://upload-images.jianshu.io/upload_images/1311457-bfbfff71ab2aeb8f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

箭头的绘制处理也比较简单，就是根据不同的情况来进行绘制就好，看代码吧。