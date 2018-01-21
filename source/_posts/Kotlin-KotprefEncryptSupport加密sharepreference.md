---
title: Kotlin KotprefEncryptSupport加密sharepreference
date: 2018-01-21 12:38:51
tags: kotlin
---
### 前言
最近在学习kotlin，发现一个比较不错的sharepreference库[kotpref](https://github.com/chibatching/Kotpref)。
它是利用kotlin的扩展和代理来实现的，使用起来也方便快捷。
但是，就是还差一个想要的功能，就是加密。
然后自己写了个[KotprefEncryptSupport](https://github.com/fly7632785/KotprefEncryptSupport)来支持一下。

### 安装

```groovy
allprojects {
		repositories {
			maven { url 'https://jitpack.io' }
		}
	}
```
```groovy
dependencies {
    compile 'com.github.fly7632785:KotprefEncryptSupport:1.0.1'
}
```

### 初始化

它已经包含了gson-support，所以，如果使用了这个库，就不用再接入gson-support了
```kotlin
class SampleApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        Kotpref.init(applicationContext)
        // add Encrypt Support 
        Kotpref.gson = Gson()
        Kotpref.cipherAdapter = SharedPrefCipherAdapter(applicationContext)
    }
}
```
### 申明使用
```
    var password by ecStringPref("jafirPass")
    var code1 by ecNullableStringPref()
    var isMan by ecBooleanPref(true)
    var age1 by ecIntPref(23)
    var highScore1 by ecLongPref(1111111111L)
    var rate1 by ecFloatPref(0.5555f)
    var person1 by ecGsonPref(Person("g jafir", 21))
    var avatar21 by ecGsonPref(Avatar())
    var avatar22 by ecGsonNullablePref(Avatar())
```
支持 Int,String,Boolean,Long,Float,Gson
###  高级

如果你想自定义加密规则，也是可以的。
只需要自己实现一下CipherAdapter，然后实现一下encrypt和decrypt两个方法就可以了。例如

```
class SharedPrefCipherAdapter @Throws(Exception::class)
constructor(context: Context) : CipherAdapter {
    private val secretKey: SecretKey

    init {
        this.secretKey = AESUtil.generateKey(context)
    }

    override fun encrypt(raw: String): String {
        return AESUtil.execEncrypted(secretKey, raw)
    }

    override fun decrypt(encode: String): String {
        return AESUtil.execDecrypted(secretKey, encode)
    }
}
```
更多的细节可以看源码


#### 默认的加密
库中已经集成了一个默认的加密adapter：SharedPrefCipherAdapter
采用的是AES、PBE混合的加密方式，AES加密内容，然后PBE加密secret key
#### Xml
```xml
<map>
    <long name="highScore" value="3901" />
    <float name="rate" value="0.4" />
    <string name="password">WUb7wV8SS18d9hEvUt8kPg==&#10;    </string>
    <string name="age1">vELsGwmt5Bhz1WkAuasEHA==&#10;    </string>
    <string name="avatar22">rN29eRFNgIlf6yIAV9cptoyabAkqmDqDtf6S4ElzPWIVS1YRMXw2avvYbyJseOZEOBqVE9kAAARV&#10;T4MpZ31fAw==&#10;    </string>
    <string name="avatar1">null</string>
    <string name="avatar21">rN29eRFNgIlf6yIAV9cpttcgywAfWQ9P21mqhkpLjhty0xyusdIZtGLibaD5gzdExLQhyLF2BbIR&#10;Vz7hM0a0KA==&#10;    </string>
    <string name="avatar">{&quot;icon&quot;:&quot;lion&quot;,&quot;updated_at&quot;:&quot;Dec 19, 2017 11:13:28 PM&quot;}</string>
    <string name="person1">gA4aAC4rCqoo9Vz3VCBgVtnerDMep/WhMsoUK736qJ4=&#10;    </string>
    <string name="code">451B65F6-EF95-4C2C-AE76-D34535F51B3B</string>
    <string name="isMan">gi8S6qu7Sklcx0oiYDGnsw==&#10;    </string>
    <set name="prizes">
        <string>New Born</string>
    </set>
    <int name="age" value="2" />
    <string name="highScore1">L5E9zu5NO5AQGFVZBmmHTA==&#10;    </string>
    <string name="name">chibatching Jr</string>
    <string name="rate1">Pa0WPZj6Of7DV8S4LYfp2g==&#10;    </string>
    <string name="code1">FZbcxuspUwL0HUdYvuQ6ltT/nWL+e5d3ZXtfTfPXGcccThyKavFb+7iB1bR8PGF6&#10;    </string>
    <string name="gameLevel">EASY</string>
</map>

```
