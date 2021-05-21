---
layout: article
title:  "公司2k显示器的正确打开方式（开启HiDPI）"
tags: "瞎折腾"
excerpt_type: html
article_header:
  type: overlay
  background_image:
    src: /pics/pic1.JPG
---
公司配置的2k显示器在mac下的正确打开方式（开启HiDPI让显示更清晰！）
<!--more-->

## 背景
可能有部分小伙伴比较好奇，为什么2k显示器在mac下面会显示的糊糊的，这里可以去看这篇文章：[retina原理](https://blog.csdn.net/Allenyhy/article/details/81610244)。简而言之，当你插上一个4k显示器的时候，打开设置-显示器，可以看到显示这样的画面：
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214182231598.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
实际上一般是默认，但是4k的默认和下面的第一个选项是一样的（下面的一排选项我们成为HiDPI）。这个看起来类似1920x1080是什么意思？考虑一下4k的实际分辨率是3840x2160，也就是实际上4k的像素个数是1080p的4倍，这里retina会用四个像素合成一像素使用，所以屏幕看起来大小会类似1080p，但是由于四个像素合成显示一个像素，所以会看起来比1080p清晰非常多（具体为啥会牵扯到渲染机制，这里就不展开了）。

但是公司发放的显示器是2k的，2k的分辨率是2560x1440，并不是1080p的整数倍，因此这里就存在一个问题，2k没办法进行点对点的渲染，就会导致出现2k屏幕上面字体发虚模糊等的现象。所以这里我们得想个办法来开启上面4k的同款HiDPI，通过虚拟多像素，来让屏幕可以得到更好的渲染效果。

## 教程

### 需要的软件
[PlistEdit Pro](https://www.fatcatsoftware.com/plisteditpro/)
我们只需要这个app对plsit文件进行修改（如果有xcode也可以用xcode）

### 获取显示器信息
首先我们可以打开终端，输入这条命令：
``` shell
ioreg -lw0 | grep IODisplayPrefsKey
```
这条命令会显示当前的显示器信息，比如我的显示如下：
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214189215926.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
这里如果有多个显示器需要关掉显示器来找到需要修改的（如果是一个显示器+笔记本自带显示器则可以通过红框前面的是AppleDisplay来识别，因为笔记本显示器显示的会是AppleBacklightDisplay）

这里红框里面会得到两个16进制数字：
1. 10ac，前面这个就是DisplayVendorID
2. d0c1，后面这个就是DisplayProductID

然后随便找一个在线的16进制转10进制计算器，进行进制转换，我这里用的是这个https://tool.oschina.net/hexconvert
转换：
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214191226878.jpg?x-oss-process=image/auto-orient,1/quality,q_90)

得到两个十进制的数字：
1. vendorid，4268
2. productid,53441

### 获取可修改文件
这里我们需要找到一个适合的可以用来修改的配置文件，步骤如下：
1. 打开你的Finder，然后选择前往服务器![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214192723048.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
2. 输入/System/Library/Displays/Contents/Resources/Overrides/DisplayVendorID-610 选择前往![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214193173035.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
3. 在文件夹里面找到一个叫 DisplayProductID-a033的文件![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214193679856.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
4. 将这个文件复制一份到桌面，准备修改


### 修改文件内容
1. 刚刚我们复制了一份文件到桌面，将这个文件用最开始那个PlistEdit Pro打开：![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214194684994.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
2. 找到我们刚刚上面转换的DisplayVendorID和DisplayProductID:![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214195151059.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
3. 修改为我们上面转换的十进制：![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214195914956.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
4. 修改分辨率。点开下面的选项scale-resolutions,可以看到很多选项：![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214197282198.jpg?x-oss-process=image/auto-orient,1/quality,q_90)这里我们将里面的数据清空（按住shift全选删除）：![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214197763308.jpg?x-oss-process=image/auto-orient,1/quality,q_90)，然后新建一个data类型：![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214198023150.jpg?x-oss-process=image/auto-orient,1/quality,q_90)![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214198229222.jpg?x-oss-process=image/auto-orient,1/quality,q_90)，数据填入**00000F00 00000870 00**注意空格也需要：![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214198738678.jpg?x-oss-process=image/auto-orient,1/quality,q_90),然后再新建一个输入**00001400 00000B40 00**

### 放入文件
1. 打开你的终端，输入sudo su，并输入你的密码（没有看到输入没关系，这个是unix系统特性，输入完毕回车就可以了）
2. 输入**mkdir -p /Library/Displays/Contents/Resources/Overrides/**
3. 再次在Finder里面前往**/Library/Displays/Contents/Resources/Overrides/**这个文件夹
4. 在这个文件夹里面新建一个文件夹：![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214200150938.jpg?x-oss-process=image/auto-orient,1/quality,q_90)，命名规则就是DisplayVendorID-xxxx，xxxx用你上面获取到的16进制vendorid替换
5. 进入刚刚的文件夹，然后将我们上面修改的分辨率文件保存，保存名称为DisplayProductID-xxxx,xxxx用你上面获取到的16进制productid替换，然后将文件放入该文件夹

最后，重启～然后打开设置-显示器，我们就可以看到2k显示器应该有这样的设置了：


## 前后对比：
之前的设置，同为1080p缩放：
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214203804797.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
实际看文字效果：
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214204224763.jpg?x-oss-process=image/auto-orient,1/quality,q_90)

设置完毕之后：
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214208092922.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
实际看文字效果：
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214208793525.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
看起来会更清晰（可能图片不是很明显，但是实际上观感会改善很多）