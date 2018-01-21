layout: android
title: android OTG 读取usb设备文件并上传（问题解决）
date: 2018-01-21 12:31:46
tags: android
---
### 前言
有一个需求：**Android otg连接usb设备（一个带SD卡的可穿戴相机），并读取其内存中的文件上传到网络**

### 问题
这个需求最大的难点在于读取usb设备中的文件，并且由于我们上传的API接口要求是以File为参数上传文件。而Android提供的是很低级别的API，以bulk transfer的方式来传输数据字节流的。

### 知识要点
###### OTG 
On the Go ，主要用于usb和不同的移动设备之间的连接和数据交换，就是用于连接类似手机等移动设备和USB设备的。
###### 文件系统类型
常见的文件系统类型有fat12 、fat16、fat32、ntfs 、hfs(苹果的)等，fat16只支持2GB一下的存储，fat32支持最多32G，但是单个文件不能超过4G，ntfs就可以支持单个文件4G以上
###### MBR
主引导记录，就是相当于磁盘的一个信息梗概，包含了有多少个分区，分区的文件系统类型是啥，有大大，逻辑地址区间是多少等等，512字节，446之后的每16个为一个分区表项，每个分区表项的第5位就是表明分区的文件系统类型

![MBR组织](http://upload-images.jianshu.io/upload_images/1311457-9c5b3a11bd46d113.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)


![文件系统标志位](http://upload-images.jianshu.io/upload_images/1311457-d2cf549ae7d4939c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
###### GPT
比MBR高级的一个分区表，主要是由于MBR最大支持2TB磁盘，4个分区，而GPT就支持更多，且兼容MBR。现在多数电脑硬盘都是GPT的

### 探索
我们需要一个文件系统的支持，于是在GitHub上面搜罗了一遍，最终找到一个框架[Libaums](https://github.com/magnusja/libaums)。

###### Libaums
- 这是一个超级棒的库，并且作者也是非常棒，很有耐心和我一起测试调查问题
- 支持otg连接手机读取USB mass storage devices , 并且是在Android Usb Host API的基础上增加了对于FAT32文件系统类型设备的支持，这才使得我们可以通过访问类似文件树的文件路径来访问其文件目录。
- 以UsbFile来表明一个文件，但是没有办法从UsbFile直接获取一个实际的挂载路径（**因为Android为了各自应用的文件安全，希望我们使用FileProvider来进行应用间文件共享。并且，虽然Android是基于linux内核的，但是Android不支持直接读取挂载的usb的文件**）
- Libaums 也是基于Android Api基础写的，所以符合Android的系统要求，所以集成的时候也需要在Manifest中配置<provider>
- UsbFile 不可以直接获取文件路径来让我们操作File，而是提供一个UsbFileInputStream类来读取UsbFile的输出流，我们可以通过流来获取文件对象。


### 开发
由于Libaums和Android自身系统不支持，我们只好把Usb设备里面的文件以流的形式读取，并且写入到手机中保存为File文件（copy到手机中），然后再上传。最后，删除手机中的copy文件。

### 问题来了
一切在U盘上测试都是好好的，但是一到了Usb设备上测试，就出现问题了。
代码中表示getPartition为空，所以没法读取到设备文件。为什么partition为空呢？

### 解决
在issue上跟作者交流了很久，并且自己也不断的测试。
最后才弄明白怎么回事。
Usb设备是一个科涵公司的相机，带SD卡，拍了照也就存储到了SD卡里面了。
partition是分区的意思，Libaums支持带有MBR分区和文件系统类型为FAT32的设备读取。
此命令``` diskutil list```可以查看一下电脑上挂在的usb设备的路径

```
bin-no-MBP:Volumes bingao$ diskutil list
/dev/disk0 (internal, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *251.0 GB   disk0
   1:                        EFI EFI                     209.7 MB   disk0s1
   2:          Apple_CoreStorage Macintosh HD            250.1 GB   disk0s2
   3:                 Apple_Boot Recovery HD             650.0 MB   disk0s3

/dev/disk1 (internal, virtual):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:                  Apple_HFS Macintosh HD           +249.8 GB   disk1
                                 Logical Volume on disk0s2
                                 8ABB2546-043F-4E17-A20A-8EECAB36BE00
                                 Unlocked Encrypted

/dev/disk2 (external, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:                            NO NAME                *31.9 GB    disk2

```
而经过测试这个设备MBR分区有问题，利用linux命令 
```hexdump -C -s 446 -n 64 /dev/disk2```发现都是0

```
bin-no-MBP:Volumes bingao$ sudo hexdump -C -s 446 -n 64 /dev/disk2
000001be  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
000001fe
```

**原来**：现在有很多SD卡或者储存设备为了节约内存，所以连512的MBR sector都给去掉了。

![网上资料](http://upload-images.jianshu.io/upload_images/1311457-1c5850838685198b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)

 最后在磁盘工具中擦除之后重新format才得以解决。
![利用磁盘工具擦除数据重新format](http://upload-images.jianshu.io/upload_images/1311457-36772f6052e2d7ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)

详情可见
[Libaums Uri to file path or usbFile to File](https://github.com/magnusja/libaums/issues/70)
[Libaums Investigate on SD card issues](https://github.com/magnusja/libaums/issues/86)


### PS

![分区表项中的第5位代表其文件系统类型
这是文件类型对照表
](http://upload-images.jianshu.io/upload_images/1311457-08be5fae058040a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)

另外mac上面的第三方磁盘工具很多都是收费的，在网上搜罗了半天，找到一个很不错的工具[Hard DIsk Manager](https://www.paragon-software.com/home/hdm-mac/)，最主要是因为它可以10天免费trial，有很多的功能

![[Hard DIsk Manager](https://www.paragon-software.com/home/hdm-mac/)](http://upload-images.jianshu.io/upload_images/1311457-c6e5564b344b4b1b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 总结
此篇文章旨在整理记录我所遇到的问题，及其在调查解决问题期间收获的东西，也顺便给一些有类似需求的朋友提供一些思路。
