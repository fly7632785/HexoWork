---
title: android-okhttp缓存真正正确的实现方式
date: 2018-03-13 11:59:27
tags:
---
### 前言
关于okhttp的缓存，网上有大量的文章，或相同，或不同，方式不一，但都八九不离十，原理都是通过CacheControl的设置策略不同来实现的。
但是，真正实践过的人会发现，好像有这样那样的问题。
比如：
- 到底是用addNetInterceptor呢还是用addInterceptor，不同的用法有不同的效果
- 什么有网的时候是maxAge，无网的时候又是maxStale等等

简直不明白。
于是乎，我个人做了很多很多的尝试，几乎把网上的方法都试了一遍，看了大量换汤不换药的文章。下面是借鉴文章出处，但是我的方法都与他们的不同。
> http://blog.csdn.net/u014614038/article/details/51210685
http://blog.csdn.net/Picasso_L/article/details/50579884
http://blog.csdn.net/briblue/article/details/52920531
https://www.jianshu.com/p/412157e236ad
https://stackoverflow.com/questions/23429046/can-retrofit-with-okhttp-use-cache-data-when-offline
https://newfivefour.com/android-retrofit2-okhttp3-cache-network-request-offline.html

#### 对于okhttp的缓存解决方案，我的需求是：
1、有网的时候也可以读取缓存，并且可以控制缓存的过期时间，这样可以减轻服务器压力
2、有网的时候不读取缓存，比如一些及时性较高的接口请求
3、无网的时候读取缓存，并且可以控制缓存过期的时间

### 正文
说了那么多，先直接上解决方案
```
    /**
     * 有网时候的缓存
     */
    final Interceptor NetCacheInterceptor = new Interceptor() {
        @Override
        public Response intercept(Chain chain) throws IOException {
            Request request = chain.request();
            Response response = chain.proceed(request);
            int onlineCacheTime = 30;//在线的时候的缓存过期时间，如果想要不缓存，直接时间设置为0
            return response.newBuilder()
                    .header("Cache-Control", "public, max-age="+onlineCacheTime)
                    .removeHeader("Pragma")
                    .build();
        }
    };
    /**
     * 没有网时候的缓存
     */
    final Interceptor OfflineCacheInterceptor = new Interceptor() {
        @Override
        public Response intercept(Chain chain) throws IOException {
            Request request = chain.request();
            if (!SystemTool.checkNet(AppContext.context)) {
                int offlineCacheTime = 60;//离线的时候的缓存的过期时间
                request = request.newBuilder()
//                        .cacheControl(new CacheControl
//                                .Builder()
//                                .maxStale(60,TimeUnit.SECONDS)
//                                .onlyIfCached()
//                                .build()
//                        ) 两种方式结果是一样的，写法不同
                        .header("Cache-Control", "public, only-if-cached, max-stale=" + offlineCacheTime)
                        .build();
            }
            return chain.proceed(request);
        }
    };

  //setup cache
    File httpCacheDirectory = new File(AppContext.context.getCacheDir(), "okhttpCache");
    int cacheSize = 10 * 1024 * 1024; // 10 MiB
    Cache cache = new Cache(httpCacheDirectory, cacheSize);
    OkHttpClient client = new OkHttpClient.Builder()
            .addNetworkInterceptor(NetCacheInterceptor)
            .addInterceptor(OfflineCacheInterceptor)
            .cache(cache)
            .connectTimeout(10, TimeUnit.SECONDS)
            .readTimeout(10, TimeUnit.SECONDS)
            .build();

```
没错最终的解决方案就是，两个interceptor，并且针对不同的网络情况进行不同的处理。（为什么是两个interceptor后面会讲到）

如果赶时间的朋友，可以直接拿去用，看到这里就行了。如果觉得用起来就问题，欢迎下面留言。

### 实践和测试的过程

首先我最初的写法就是来源于网上大多数人的写法，类似
```
public class HttpCacheInterceptor implements Interceptor {

@Override
public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    if (!NetWorkHelper.isNetConnected(MainApplication.getContext())) {
        request = request.newBuilder()
                .cacheControl(CacheControl.FORCE_CACHE)
                .build();
    }

    Response response = chain.proceed(request);

    if (NetWorkHelper.isNetConnected(MainApplication.getContext())) {
        int maxAge = 60 * 60; // read from cache for 1 minute
        response.newBuilder()
                .removeHeader("Pragma")
                .header("Cache-Control", "public, max-age=" + maxAge)
                .build();
    } else {
        int maxStale = 60 * 60 * 24 * 28; // tolerate 4-weeks stale
        response.newBuilder()
                .removeHeader("Pragma")
                .header("Cache-Control", "public, only-if-cached, max-stale=" + maxStale)
                .build();
    }
    return response;
  }
}

  //设置缓存100M
        Cache cache = new Cache(new File(MainApplication.getContext().getCacheDir(),"httpCache"),1024 * 1024 * 100);
        return new OkHttpClient.Builder()
            .cache(cache)
            .addNetworkInterceptor(new HttpCacheInterceptor())
            .build();
```
这段代码满足不了我的需求，并且会发现，有些情况离线的时候缓存过期的时间不可靠，有些情况在线的时候缓存不可用。对于我的需求不可兼得。（故网上有人还分了2种不同情况的写法来针对不同的需求，但我想，为什么不综合起来呢？）

对于这段代码疑惑点：
1、max-age是啥，maxStale是啥，他们的区别是啥？
2、为什么没有网络的情况下，request要cacheControl.FORCE_CACHE
3、为什么又要对response设置header的cache-control，到底request的设置跟response的设置有什么区别？
4、addNetInterceptor和addInterceptor有什么区别？

解答：
1、max-age是啥，maxStale是啥，他们的区别是啥？
>maxAge和maxStale的区别在于：
maxAge:没有超出maxAge,不管怎么样都是返回缓存数据，超过了maxAge,发起新的请求获取数据更新，请求失败返回缓存数据。
maxStale:没有超过maxStale，不管怎么样都返回缓存数据，超过了maxStale,发起请求获取更新数据，请求失败返回失败

2、为什么没有网络的情况下，request要cacheControl.FORCE_CACHE
>  public static final CacheControl FORCE_CACHE = new Builder()
      .onlyIfCached()
      .maxStale(Integer.MAX_VALUE, TimeUnit.SECONDS)
      .build();
可以看到FORCE_CACHE是设置了maxStale的最大时间为interger的最大时间，所以，意思就是无论如何，都不会超过这个时间，所以就是一直（强制）拿缓存，也是想要实现缓存的正确逻辑

> 一般控制缓存有两种方式：
1、在request里面去设置cacheControl()策略
2、在header里面去添加cache-control

>后面也会看到，在request里面设置header的cache-control和调用cacheControl方法来设置其实是一样的，我们就是通过在request里面来控制无网缓存的maxStale过期时间的

3、为什么又要对response设置header的cache-control，到底request的设置跟response的设置有什么区别？
>其实我到现在都还没有搞清楚，为啥那些人要这样写，只是后面我在测试的过程中推断出来，有网的时候和无网的时候对于interceptor的调用是不同的，产生的结果也是不同的。比如request设置的时候就对无网缓存及其时间控制有效，response就不行

4、addNetInterceptor和addInterceptor有什么区别？
> addNetInterceptor是添加网络拦截器，addInterceptor是添加应用拦截器，如果看到okhttp的流程分析的知道：应用拦截器是在网络拦截器前执行的。

>如果我使用的是addNetInterceptor：
1、有网的情况下，可以在期限内拿到缓存，而没有去请求接口（通过测试数据库的数据改动来判断的）
2、没有网的情况下，直接就ConnectException了，根本不会走到interceptor里面去了。（网上很多人都提出了这样的问题）

>如果我使用的是addInterceptor:
1、有网的情况下，明明设置的是60秒，但是每次都没有去拿缓存而都是请求的接口。（通过测试数据库的数据改动来判断的）
2、没有网的情况下，可以拿到缓存数据（猜想：可能是因为应用拦截器在网络拦截器前执行，没有网的情况下，本身就执行不到网络拦截器里面去），但是缓存过期时间是“永久”，因为FORCE_CACHE里面已经设置为了integer的最大值，21亿秒左右，堪称永久
但是，依旧没有办法控制无网时候的缓存过期时间

面对这些个问题，我也是很无奈。只得不停地尝试，不停地摸索。于是就在尝试的过程中发现了它的一些规则，于是最终写出了一个自认为“万全”的方法。

- 既然想要有网的情况下拿缓存，那么就需要addNetInterceptor，如果需要无网的情况下拿缓存，就需要addInterceptor，所以不如直接做两个interceptor吧！
- 另外，如果想要控制有网的时候不去读取缓存，可以直接通过在response里设置maxAge=0来实现。
- 这里通过大量实验发现，只有在request去设置其maxStale才能控制无网时候的缓存时间，在response里面去控制是不行的！

##### 延伸
如果无网的时候，stale缓存时间过了，会怎么样呢？
会报504错误（属于正常的逻辑）。

### 最后
欢迎留言，欢迎提问题、交流。