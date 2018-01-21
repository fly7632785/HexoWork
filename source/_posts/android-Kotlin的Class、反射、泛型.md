layout: android
title: Kotlin的Class、反射、泛型
date: 2018-01-21 12:36:14
tags: kotlin
---
### 前言
最近在学习kotlin的反射的时候遇到了一些问题，特地记录一下。
### 正题
在Java中使用Class很常见的就是，xxx类.class，比如我们在startActivity的时候
```java
startActivity(new Intent(this, OtherActivity.class));
```
这里接收的就是CLass<?> cls参数。
那么在java中获取Class的方法有哪些呢？
```
1、Class c = person.getClass(); //对象获取
2、Class cc =Person.class;//类获取
```

而我们来看看kotlin
```
//对象获取
person.javaClass// javaClass
person::class.java // javaClass
//类获取
Person::class// kClass
 person.javaClass.kotlin// kClass
(Person::class as Any).javaClass// javaClass
Person::class.java // javaClass
```
哇，这么多种，他们是不是一样的，有没有什么区别？
log看看他们到底是不是相同的Class
```
println(person.javaClass == person::class.java) //true
println(person.javaClass == Person::class.java)//true
println(person::class.java == Person::class.java)//true
 //person.javaClass == person::class.java == Person::class.java
println(person.javaClass == Person::class)//false
println(person.javaClass.kotlin == Person::class)//true
println(person::class == Person::class)//true
```
从log来看，
```
person.javaClass == person::class.java == Person::class.java
```

三者是相同的。但是
```
person.javaClass == Person::class
```
却是不同的。为什么呢？

原因是在kotlin中的Class与Java不同，kotlin中有一个自己的Class叫做KClass, ```person::class``` 和```Person::class```都是获取kotlin的KClass，所以```println(person::class == Person::class)``` 为true。
我们可以从kotlin的KClass获取到java的Class,```person::class.java```就是如此，先获取到kotlin的KClass然后再获取javaClass。
**object/class->kClass->Class**
同样也可以通过java的Class获取kotlin的KClass，```person.javaClass.kotlin```就是先获取javaClass然后再获取kotlin的KClass
**object/class->Class->KClass**
那么KClass都有些什么用呢？Find Usages 可以看到
![](http://upload-images.jianshu.io/upload_images/1311457-001add4c4ed4964f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
几乎多数跟Reflect相关，而用的最多的也是在**KClasses**里面
KClasses扩展了许多跟反射相关的方法，算的上是kotlin的反射中类主力输出。
![](http://upload-images.jianshu.io/upload_images/1311457-c3f64f8dac28f56e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如果要使用kotlin的反射类的话，要加入
```
  compile "org.jetbrains.kotlin:kotlin-reflect:$kotlin_version"
```
![](http://upload-images.jianshu.io/upload_images/1311457-e7500755a2a391e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 举个例子
我们要把一个类的所有字段通过反射给打印出来，调用java的方法来实现是这样
```
 println (Person::class.java.declaredFields.map {
            it.isAccessible = true
            "${it.name}: ${it.get(person)}"
        }.joinToString(","))
```
我们通过Person::class拿到KClass，然后.java拿到java的Class<?>，再获取declaredFields，最后通过map，然后把获取的Field获取到值打印出来

##### 问题
而使用kotlin，我们还有别的做法
```
 println (Person::class.memberProperties.map {
            it.isAccessible = true
            "${it.name}: ${it.get(person)}"
        }.joinToString(","))
```
通过Person::class拿到KClass，直接调用KClass的memberProperties来拿到KProperty的Collection集合，然后进行操作。当然，这里KProperty也是kotlin的反射类中的，也类似于Java的Field。
那么既然可以这样，理论上这样也应该是可以的
```
//person是对象不是Person类
 println (person::class.memberProperties.map {
            it.isAccessible = true
            "${it.name}: ${it.get(person)}"
        }.joinToString(","))
```
这下却出错了，真的奇怪！
```
Error:(73, 31) Out-projected type 'KProperty1<out OtherActivity.Person, Any?>' prohibits the use of 'public abstract fun get(receiver: T): R defined in kotlin.reflect.KProperty1'
```
为什么会这样？
查看it发现

![](http://upload-images.jianshu.io/upload_images/1311457-2db8ddb0a9a501e9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

kotlin的property的Person为out逆变的，R只能作为输出，不能作为get的参数传入，所报错了。
这里的kotlin泛型还是有点小坑的需要你踩一踩，[网上有一篇文章解释说明了一番](https://www.qcloud.com/community/article/704805)


那么怎么改呢？
三种办法：
-
```
//扩展KProperty的get方法为getUnsafed，其实就是去掉了out
   fun <T, R> KProperty1<T, R>.getUnsafed(receiver: Any): R {
        return get(receiver as T)
    }
//然后
println(person::class.memberProperties.map {
            it.isAccessible = true
            "${it.name}: ${it.getUnsafed(person)}"
        }.joinToString(","))
```
-
```
//强转一下
 println(person::class.memberProperties.map {
            it.isAccessible = true
            it as KProperty1<Person, Any>
            "${it.name}: ${it.get(person)}"
        }.joinToString(","))
```
-
```
//这一种就涉及到kotlin中的获取KClass的方式, 先获取java的Class再获取kotlin的KClass
//神奇的是，这种获取到的it的类型没有out，而是我们期望的 KProperty1<Person, Any>
 println(person.javaClass.kotlin.memberProperties.map {
            it.isAccessible = true
            "${it.name}: ${it.get(person)}"
        }.joinToString(","))
```

### 最后
在kotlin学习和探索的过程中发现了这些问题，非常地好奇，想要探知其原理。所以摸索着前进，希望能带给其他人一些学习的思路和兴趣。
另外，还有一个问题就是：
为什么有时候map的it是带out的有时候不带？这个跟class是java的class还是kotlin的class是否有关系？其间的原理和过程是怎么样的，我还在继续探索，希望明白的同学可以分享一下。