layout: android
title: android scheme链接打开本地应用
date: 2018-01-21 12:29:19
tags: android 
---
大家都知道，如果我们想要打开手机本地的其他应用，我们可以通过**intent的隐式**启动，添加相关界面activity的包路径，来打开对应的应用和其界面。但这并不是万能的，因为**一来别人家的APP的包路径，不通过逆向手段是无法获知的，二来如果其打开界面本就需要传递一些参数才能正常开启，字段key你也不知道**。

所以，Android系统也知道这种情况，所以才有了**scheme**这样的东西来**方便**我们的应用提供一个入口供别人开启。

废话不多说先上[demo地址](https://github.com/fly7632785/SchemeDemo)

![](http://upload-images.jianshu.io/upload_images/1311457-e32a9849d8b9ae35.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/840)
![](http://upload-images.jianshu.io/upload_images/1311457-b96e0bee3f01d5e2.gif?imageMogr2/auto-orient/strip)

这篇文章主要讲述
一、提供scheme供别人打开自己的应用
二、webview中的网页链接打开应用
三、有哪些现成的scheme url
四、相关注意事项


---------
### 一、通过scheme提供开启自己应用的入口：
方法很简单：
在AndroidManifest.xml文件中的activity标签中添加intent-filter,并且添加data的scheme、host等，如图

![](http://upload-images.jianshu.io/upload_images/1311457-1a561a637d421c90.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/840)

##### 注意：在MainActivity中要新写一个intent-filter，如果直接写在launcher打开的filter中则会app图标消失（因为category被覆盖为DEAFAULT）

这样就可以通过以下代码打开我们自己的app了
```
String url = "jafir://main.app" 
 Intent in = new Intent(Intent.ACTION_VIEW, Uri.parse(url));
                    startActivity(in);
```


**我们还可以通过这个url链接传递参数**
传递参数有啥用呢？比如打开QQ的与某人的聊天界面，你肯定要传递一个qq号
所以打开qq的链接一般为：
**mqqwpa://im/chat?chat_type=wpa&uin=522648467**
```
String url = "jafir://main.app?key=传递的参数" 
 Intent in = new Intent(Intent.ACTION_VIEW, Uri.parse(url));
                    startActivity(in);
```
然后在MainActivity中通过如下代码获得
```
        Intent intent = getIntent();
        String scheme = intent.getScheme();
        Uri uri = intent.getData();
        System.out.println("scheme:"+scheme);
        if (uri != null) {
            String host = uri.getHost();
            String dataString = intent.getDataString();
            //获得参数值
            String key1 = uri.getQueryParameter("key1");
      }  
```
效果图：

![](http://upload-images.jianshu.io/upload_images/1311457-b96e0bee3f01d5e2.gif?imageMogr2/auto-orient/strip)



### 二、 webview中链接打开应用
我们都知道现在的网页web一般开启三方应用，比如开启迅雷，开启QQ临时对话等等，这些都是网页链接，但是网页链接确可以打开Android本地应用（前提是浏览器支持）。一个网页链接可以做到在电脑浏览器上打开PC软件，在手机浏览器上又是打开app，这是怎么回事呢？原因稍后解释

在Android的webview中本是不支持直接打开本地应用的，所以我们就要自己来处理。


```
 String url = "http://wpa.qq.com/msgrd?v=3&uin=522648467&site=qq&menu=yes";
 WebViewClient webViewClient = new WebViewClient() {
            @Override
            public boolean shouldOverrideUrlLoading(final WebView view, String url) {
                if (url.startsWith("http") || url.startsWith("https")) { //http和https协议开头的执行正常的流程
                    return false;
                } else {  //其他的URL则会开启一个Acitity然后去调用原生APP
                    Intent in = new Intent(Intent.ACTION_VIEW, Uri.parse(url));
                    if (in.resolveActivity(getPackageManager()) == null) {
                        //说明系统中不存在这个activity
                        view.post(new Runnable() {
                            @Override
                            public void run() {
                                Toast.makeText(MainActivity.this, "应用未安装", Toast.LENGTH_SHORT).show();
                                view.loadUrl(failUrl);
                            }
                        });

                    } else {
                        in.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED);
                        startActivity(in);
                        //如果想要加载成功跳转可以 这样
                        view.post(new Runnable() {
                            @Override
                            public void run() {
                                view.loadUrl(successUrl);
                            }
                        });
                    }
                    return true;
                }
            }
        };
        webView = (WebView) findViewById(R.id.webview);
        webView.getSettings().setJavaScriptEnabled(true);
        webView.setWebViewClient(webViewClient);
        webView.loadUrl(url);
```
网上的方法，普遍为这种：
**通过在重定向的时候判断是否是普通的网页链接，如果不是则为scheme调用的这种，则我们自己来处理为intent跳转开启**

**tips:**
可以在成功的时候再``` view.post(new Runnable() {
                            @Override
                            public void run() {
                            view.loadUrl(successUrl);
                            }
                        });```然后就会自动跳转到成功的界面。successUrl是我本地的assets里面写的一个成功的html界面
同理 可以在应用未安装失败的时候再``` //说明系统中不存在这个activity
                        view.post(new Runnable() {
                            @Override
                            public void run() {
                                Toast.makeText(MainActivity.this,"应用未安装",Toast.LENGTH_SHORT).show();
                                view.loadUrl(failUrl);
                            }
                        });```然后就会自动跳转到失败的提示界面。failUrl是我本地的assets里面写的一个失败提示的html界面

##### 测试通过网页链接打开QQ：
###### 注意：
我们在打印重定向url的时候发现，qq的url重定向了几次，最终在手机浏览器上呈现的形式就是scheme的形式：**mqqwpa://im/chat?chat_type=wpa&uin=522648467**
为什么要重定向呢？就回到了之前上面提的那个问题，网页上使用的url：**http://wpa.qq.com/msgrd?v=3&uin=793563805&site=qq&menu=yes**这个在电脑浏览器上可以直接打开PC的qq，但是在手机上，重定向就是在判断和甄别是否是手机浏览器，如果是的话就会把url重定向为scheme形式的链接。

并且这里要**注意**：**如果shouldOverrideUrlLoading返回的不是true，而是super，那么你会惊奇的发现，qq直接重定向到它自己的应用宝去了，哈哈，有点恶心**

测试：
我们在本地assets里面建一个html
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <title>JavaScript HTML App</title>
</head>
<body>
    <!--<h1>已经打开APP啦！！！！</h1>-->
 <a target="_blank" href=" http://wpa.qq.com/msgrd?v=3&uin=522648467&site=qq&menu=yes">打开qq</a>
</body>
</html>
```

然后直接打开
```
webview.loadUrl("file:///android_asset/index.html");
```
模拟webview中点击链接打开qq应用

#### 注意：
1、我们要想打开指定qq的临时聊天界面，需要那个qq已经开通了临时聊天界面，不然会报错。
[在这里开启](http://shang.qq.com/v3/index.html)的**推广工具**中免费开通，不然的话它重定向的url就不是mqqwpa://chat... 而是 tencent://message...去了，直接调用intent就找不对对应的activity然后crash，所以要**确保你的对应qq号是已经开通了临时聊天的（推广）**。
2、如果你想要点击了开启应用然后跳转到别的网页链接可以这么做
```
//在shouldInterceptRequest中重新加载
view.post(new Runnable() {
                        @Override
                        public void run() {
                            view.loadUrl(localUrl);
                        }
                    });
```
**注意**：因为webview的拦截线程不在主线程，所以可以用
```webview.post(new runnable(){})```
来实现

效果图：由于模拟器上面没有x86 qq 所以，效果的话，你自己运行[demo](https://github.com/fly7632785/SchemeDemo)然后在手机上试吧

### 有哪些应用的scheme url

我这里整理了一下，有点多，没有全部测试过(网上搜罗而来，并非原创)
```
QQ的url是 mqq:// 
微信是weixin:// 
淘宝taobao:// 
点评dianping://
 dianping://search
 微博 sinaweibo:// 
名片全能王camcard:// 
weico微博weico:// 
支付宝alipay:// 
豆瓣fm：doubanradio:// 
微盘 sinavdisk:// 
网易公开课ntesopen://
美团 imeituan:// 
京冬openapp.jdmoble:// 
人人renren://
 我查查 wcc:// 
1号店wccbyihaodian:// 
有道词典yddictproapp:// 
知乎zhihu://
优酷 youku://
```


![](http://upload-images.jianshu.io/upload_images/1311457-62f9e36a5656ab3e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![](http://upload-images.jianshu.io/upload_images/1311457-6bd04773176feacf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




![](http://upload-images.jianshu.io/upload_images/1311457-51e2ca8fbebd771e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![](http://upload-images.jianshu.io/upload_images/1311457-7b33516711742ec4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![](http://upload-images.jianshu.io/upload_images/1311457-e6e56fe463a39b09.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 四、注意事项
其实写到最后发现，那些注意事项基本都是写到各项里面去了，觉得那样会好点。
这里就写写思路吧：

我们的应用如果需要提供给别人数据，那么用的是**contentProvider内容提供者**，如果是供别人打开的话用的是**scheme url intent隐式打开**，这里就就涉及到相关的一些被别人开启和使用的知识，联想到比如**launchMode**，提供给别人使用的界面一般情况下可能是用singleTask 或者某些特殊的会使用singleInstance，再者再深一点的还有**taskAffinity**相关，哪些提供出来的独立的界面在自己的任务栈还是在指定的任务栈里面，这些统统都是跟 **提供** 有关联。

### 最后
总之，我在写博客的之前并没有想到我要写这篇博客，但是在一步一步实践探索那些知识的时候确实就是踩了一些坑，所以还是想要记录下来分享，又想不能做的太水，于是要求内容稍微饱满，就这样逐渐逐渐思维发散，然后拼接整理，最终落下此文。

ps:如果文中有错误，请指正，谢谢。也欢迎广提意见，思维发散。
