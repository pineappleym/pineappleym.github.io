---
layout: article
title:  "iOS CoreAnimation阅读"
tags: "iOS"
excerpt_type: html
article_header:
  type: overlay
  background_image:
    src: /pics/pic1.JPG
---
iOS CoreAnimation阅读的一些笔记
<!--more-->

## 前言
建议看之前应该有一点视图的基本知识（指传统的UIView之类的，和CALayer还是有很多的共通之处的）

## 图层树
CALayer可以理解为仅仅负责展示内容的图层，和UIView不同的是CALayer并没有响应事件的能力，所以并不在系统的响应链里面。

## 寄宿图
### CALayer设置图片
CALayer除了可以显示颜色之外，当然也可以用来展示图片
``` Objective-c
UIImage *image = [UIImage imageNamed:@"test.jpeg"];
    
CALayer *blueLayer = [CALayer layer];
blueLayer.frame = CGRectMake(100, 200, 200, 200);
blueLayer.contents = (__bridge id)image.CGImage;
[self.view.layer addSublayer:blueLayer];
```
需要注意的是由于历史原因这里CALayer.contents的属性是id类型，所以这里需要用Toll-free Bridge做一个转换。UIImage.CGImage类型是CGImageRef,不是一个OC对象所以不是id类型，需要转换。

### CALayer显示图片的参数
#### contentGravity
图片在设置过程中会有很多种的自适应方式（因为绝大多数情况下你不能确保图片的长宽比例是一样的），比如在我用例子里面的方式设置自己的图片时，得到的是下面的情况：
![-w493](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/18/16211510525243.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
明显不符合我们的需求，图片被拉伸了，因此这里我们可以指定content.contentsGravity为kCGGravityResizeAspect来进行自适应的变换：
![-w493](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/18/16211512173166.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
可以看到图片自适应的进行了变换。具体的选项可以去CoreAnimation的定义里面去查看。

#### contentScale
这里的变换是定义了像素尺寸和你的视图大小的比例，默认为1，也就是这里会以一个物理像素来渲染一个图片像素。可能有人会对这里比较懵，实际上就是retina屏幕的一个特性。retina屏幕会用多个像素渲染一个像素，比如在mac外接4k显示屏的情况下，它会显示看起来像1080p：
![-w417](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/18/16211514747378.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
但是实际上4k的像素个数是1080p的4倍，这里是用四个像素合成一个像素进行渲染，也就是contentScale为4。

这边设置这个是为了保证点对像素的对应，因为你不知道设备是retina还是非retina的，所以需要做一个设置来保证不会出现问题：
``` Objective-c
layer.contentsScale = [UIScreen mainScreen].scale;
```

#### maskToBounds
这个简单来说就是CALayer允许绘制的图层超过视图的边界，这边可以通过设置这个为True来保证内容处于边界。

#### contentsRect
这个看到rect可以知道其表示的是一个显示区域。但是这个需要注意的点在于，这个是按照比例来对图片进行裁切的。
举个例子，这个的默认值是{0,0,1,1}，表示默认的坐标起始点是{0,0},默认的比例是x，y都取100%，即显示整个图片。
在某些场景下，我们可能会把一张图片裁切成四个区域进行显示，这里即可设置contentsRect。比如我想获取一张图片的左上角四分之一，可以设置{0,0,0.5,0.5}即可。

#### contentsCenter
看到这个概念可能有人会联想到rect的center概念，但是这个和那个完全无关。原文说的也有点晦涩，这里就自己画个图说明下。
首先这个概念本身的值也是一个rect，默认值为{0,0,1,1}，也就是以{0,0}为原点，x和y比例都是1，表示整个区域都在覆盖内。那么当我们将这个值修改为{0.25,0.25,0.5,0.5},也就是以{0.25,0.25}为原点，x,y比例都是0.5，会发生什么呢,我们看下面这张图：
![-w276](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/18/16211620845070.jpg?x-oss-process=image/auto-orient,1/quality,q_90)

我们按照比例划分好了这张图，注意看四个角落都有正方形，这里划分的意思就是除开四个角落的正方形区域，当发生拉伸或者收缩的时候，会优先改变非这些区域。下面是对比图：
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/18/16211621636200.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
这里就是当我们进行变换的时候，注意除开四个角落都发生了变化。这张图可以很好的体现出这里的contentsCenter的意义。

### Custom Drawing
UIView有一个drawRect方法可以被实现，当我们实现这个方法时，会在视图在屏幕上面展示的时候调用进行绘制。

当我们在已经展示过后需要进行修改时，需要注意的是如果这里我们修改了视图的大小，会对该视图进行重新计算绘制，如果没有修改的话可能需要手动调用setNeedsDisplay方法，为了保持这里的行为的一致性，建议修改完均手动调用刷新一下。

CALayer存在一个delegate-CALayerDelegate，虽然不是一个正式的协议，但是当你实现里面的方法就会优先调用。

CALayerDelegate里面包含的方法如下：
![-w284](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/18/16213069661486.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
具体每个方法可以参考UIView的类似生命周期delegate。

## 图层几何学
这一章主要讲述图层之间的关系和图层的布局的自动调整

### 布局
回想一下UIView用于表示坐标或者说布局的几个常用属性是：
1. frame，表示其相对于superView的坐标和大小
2. bounds，表示其内部坐标
3. center，表示中心点相对于superView的anchorPoint的所在位置

关于这几个属性的具体定义可以参考这个文章：[bounds frame cneter定义](https://blog.csdn.net/chy305chy/article/details/45485587)

回到CALayer，为了和UIView进行区分，其中心位置从center变为了position，其余没什么区别。
另外需要注意的是frame一般都是一个计算出来的值，我们对frame的修改对应到transform和bounds，会重新计算一个值来对应修改。

### 锚点(AnchorPoint)
这个锚点也是一个百分比概念，可以说是相对于自己应该所处的位置的一个偏移值，具体来说就是自己的应该所处的中点相对于自己目前的左上角点的一个偏移。如果说的不够清楚的话看下面几张图：
![-w1075](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/18/16213107108754.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
![-w1076](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/18/16213107655249.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
（蓝色和图片是红色图层的subLayer)
可以看到这里当我们设定y为0.25（相对于默认值0.5少了0.25），其向下偏移了0.25个相对距离。这里就是左上角的y相对于默认接近了0.25.

AnchorPoint相对于Position的一句话区别也很好理解：Position是相对于superLayer来说的，AnchorPoint是相对于本身来说的。这里可以顺便看一下当我们修改Position的时候Position的值：![-w877](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/18/16213182873471.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
可以发现这里的值仍然是默认值（视图大小为300x300），也就是相对于superLayer仍然是处于中心的

### 坐标系
#### 坐标的转换
一般来说坐标都是相对于superView来说的，但是这里存在需求就是当我获取到一个point或者是rect时，我可能需要知道其相对于屏幕的绝对坐标，而这里只提供了一个相对于superView的相对坐标，因此需要做转换：

``` Objective-c
- (CGPoint)convertPoint:(CGPoint)point fromLayer:(CALayer *)layer;
- (CGPoint)convertPoint:(CGPoint)point toLayer:(CALayer *)layer;
- (CGRect)convertRect:(CGRect)rect fromLayer:(CALayer *)layer;
- (CGRect)convertRect:(CGRect)rect toLayer:(CALayer *)layer;
```

#### mac和iOS
从上面的知识我们知道position在iOS上的计算是相对于superLayer的左上角点的。但是在MacOS上面计算会以左下角为标准。所以这里在处理这种兼容性情况时需要进行一点适配，适配的办法就是通过geometryFlipped来设定是否允许垂直翻转（翻转过后参考点就从左上角变成了左下角），这是一个BOOL值，默认False

#### Z轴
手机屏幕是一个平面，所以我们之前的坐标都是在平面上的，但是CALayer也有Z轴的概念。这里z轴实际上用途很少，但是有一个比较常用的用法就是用于改变显示顺序。还是上面展示的例子，这里我们将两个CALayer都设置为同一个CALayer的子layer，然后改变他们的z轴值：
![-w978](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/18/16213304110163.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
可以看到红色layer被设置了一个较小的值，蓝色的更大，所以蓝色图层叠到了红色图层上面。也就是说，z轴值越大，会更靠近用户。

### Hit Testing
CALayer并不属于传统的响应链，但是仍然提供了方法来做点击检测。在看这个之前先介绍一下传统的UIView的点击测试。

当我们点击屏幕时，数据会被打包通过SpringBoard通过mach转发消息给当前前台app，然后app会去根据数据做一个叫hit test的操作，这个操作会逐层向下转发消息，直到找到哪个UIView应该来响应当前这次事件。

因此这里类比UIView，CALayer也提供了两个方法来响应当前这次事件：

``` Objective-c
- (BOOL)containsPoint:(CGPoint)p;
- (nullable __kindof CALayer *)hitTest:(CGPoint)p;
```

通过这两个方法即可返回响应的图层。

## 视觉效果
这一个Chapter将会讨论一些可以使用CALayer属性实现的视觉效果～～～

### 圆角
当我们使用iPhone的时候，可以看到整个屏幕上面充斥着很多圆角元素。圆角是app在设计视觉效果绕不开的一步，设置该属性的属性是cornerRadius。

需要注意的是，这里我们设置该属性只会影响本身：
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/18/16213350833420.jpg?x-oss-process=image/auto-orient,1/quality,q_90)

当我们设置完毕之后，可以看到layer本身确实被上了一个圆角，这个时候让我们再添加一个图片：
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/18/16213352332389.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
可以看到图片内容并没有圆角，也就是这里的图层内容并不会被干扰。如果想要其也要圆角，需要使用maskToBounds属性：
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/18/16213352971359.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
可以看到图片也被添加了圆角。
上面的这个属性也可以用于一些比较复杂的视图，比如我想要某个视图的一个角是圆角其余的不是，则可以用这个属性叠加视图来完成。

### 图层边框
这个就比较简单了，通过设置borderWidth(float)即可给图层添加边框：
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16213929303605.jpg?x-oss-process=image/auto-orient,1/quality,q_90)

### 阴影
首先来点简单的，我们直接用shadowOpacity(float,[0,1])给layer加一点简单的阴影：
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214066213186.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
可以看到这里的layer已经被挂了一层阴影，这就是阴影的最简单应用。当我们在layer的类里面搜索shadow的时候，可以看到还存在如下属性：
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214075299255.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
除开shadowOpacity我们上面已经使用过了之外，这里介绍一下其他属性。

首先是shadowColor(CGImageRef)，这个很好理解，我们可以通过这个属性来控制阴影的颜色，默认就是灰黑色，当然你也可以换成绿的（我想把这玩意染成绿的.jpg）：

![-w400](http://wx1.sinaimg.cn/large/415f82b9gy1fi2753iaklj20qo0placn.jpg)

![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214076843692.jpg?x-oss-process=image/auto-orient,1/quality,q_90)

然后是shadowOffset(CGSize)，从这个名称可以看出来这个属性控制着阴影的方向和距离，可以先看一下默认的值：
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214080864822.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
也就是对于默认值来说是{0,-3}，分别用于控制宽度和高度。可能会对于宽度和高度会比较难以理解这个值对应什么，我们可以修改一下值来看一下阴影会发生什么变化（为了容易区分将阴影颜色设置为红色）
默认{0,3}:
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214087486465.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
改变y到一个更大的负值{0,-10}:
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214090584859.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
改变y到一个正值{0,10}：
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214091746653.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
改变x的值{10,10}:
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214093684193.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
可以看到x,y都是在不同的方向上控制阴影宽度，正值右下，负值左上。

需要补充一点，就是这个阴影向上的由来。上面说过mac和iOS会有y轴翻转的问题，最开始在mac设计的是向下，所以在iOS上默认阴影向下～

然后是shadowRadius，这个用于控制阴影的圆角～

最后介绍一下shadowPath。其实从这个名称中可以比较简单的看出这个和CGPath有关，这是一个CGPathRef类型的参数。CGPath是一个用于指定矢量图形的对象，通过制定这个对象我们可以创建出来一个非紧贴着layer的奇形怪状阴影。

#### 阴影裁剪
有时候我们想要一个贴合图层上面显示的图片的边框（这种很常见），比如我加载了一个twitter的图标：
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214117826564.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
但是觉得没有阴影这个图标不够生动不够突出，我希望这里的推特图标有阴影，随着轮廓的阴影，所以我们可以直接添加阴影上去：
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214128225469.jpg?x-oss-process=image/auto-orient,1/quality,q_90)

但是这里会存在一个问题，如果我们在添加图片的时候选择了开启maskToBounds属性，这里图层之外的渲染都会被裁剪掉导致失效，为了解决这个问题，在这种情况下，我们可以选择使用两个layer，一个用于显示裁剪内容，一个专门用来显示阴影。

### 图层蒙版
图层蒙版可以理解为我在一张图片上面叠加了另一张图来实现一个复杂的视觉效果。比如上面我们利用到了两张图，一张可爱的妹子图和一张推特logo，现在有一个产品经理对你说想要一个推特形状的妹子，这个需求很简单，怎么实现她不管，这里你就可以用到图层蒙版了：
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/19/16214149218100.jpg?x-oss-process=image/auto-orient,1/quality,q_90)

通过设置layer的mask属性，我们可以较为轻松的裁切出来以mask为轮廓的图。

同时，mask可以不仅仅是我们读取的一张图片资源，也可以是一张我们通过代码实时创建的蒙版，所以这里可以动态的去创建图形然后首先一个动图效果～

### 拉伸过滤
当我们在显示一张图片的时候，大多数时候我们是希望像素和屏幕一一对应的，但是在其他情况下我们可能希望对于图片做一个拉伸或者是缩小，比如我们希望用同一张图来当头像和背景，那么必然要做一次拉伸或者是缩小才能符合我们的需求。

为了满足这个需求，CALayer提供了两个属性：minificationFilter（缩小用的）& magnificationFiliter（放大用的）。这两个属性都拥有三个可选过滤器：
* kCAFilterLinear
* kCAFilterNearest
* kCAFilterTrilinear

可以理解三个过滤器都对应着不同的算法，默认过滤器的话系统选择的是第一个。这个选项的算法翻译过来就是双线性滤波算法。这里考虑一下如果我们放大一张图片，肯定有很多的区块会是空的，我们需要一种算法去填补这些内容。而双线性滤波算法实际上就是将原有的像素（像素实际上就是一个矩阵）过一下一个滤波矩阵，得到的结果填入空白区域，我们就可以获取一张较为大的图片（或者是缩小的图片）。具体的细节可以参考这篇文章：[bilinear interpolation](https://en.wikipedia.org/wiki/Bilinear_interpolation)

在实际表现上，第一和第三个滤波器采用的算法比较接近，效果也比较好，可以优先采用这两种。

第二种滤波器从名称nearest也可以看出，这种算法会直接取最近的单个像素点，算起来比较快，但是会导致还原度比较差。

当然也不是绝对的，对于差异比较明显或者是很小的图，很少斜线的图，采用第二种滤波器会让图片特征得到较好的保留（感觉有点像边缘检测）。所以说具体情况具体看待，来选择较好的滤波器～

### 组透明
说到透明效果，看过UIView的应该都知道有一个α属性，对应的，CALayer有一个opacity属性。

这边要说到的组透明和多个控件叠加有关。假设我们这里有两个50%透明度的组件，相互叠加会出现比较尴尬的问题：
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/20/16214967638985.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
会出现两个50%透明度叠加导致右图的情况。但是很明显我们这里是希望组件的透明度是50%，所以这里我们可以选择启用CALayer的shouldRasterize属性，这个属性会让整个图层和子图层在应用透明度之前将其作为一个整体处理，然后整体施加一个50%的透明度。


## 变换
这个部分主要会介绍对于图层的旋转扭曲等操作，统称为变换

### 仿射变换
说到这个之前，先回想一下上面提到的知识，默认来说，某个图层在屏幕上会有三个坐标，分别是x,y,z，这三个可以用来形容一个在三维空间的点，或者说，一个在三维空间的以原点为起始点的三维空间向量。为了对这个向量做一个变换，按照我贫瘠的线性代数知识，我们需要一个3x3的矩阵，也就是：
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/20/16215032184398.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
然后我们之前其实还有提到，z轴实际上是用于管理层级的展现关系的，因此这里并不需要z的转换，所以这里我们可以将矩阵缩减为：
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/05/20/16215034556849.jpg?x-oss-process=image/auto-orient,1/quality,q_90)

通过这样的两列的矩阵即可对于一个视图进行一次仿射变换。

当然有些同学可能对于矩阵变换比较懵逼，这里系统也提供了比较便捷的办法。

CALayer本身存在一个属性：
``` Objective-c
/* A transform applied to the layer relative to the anchor point of its
 * bounds rect. Defaults to the identity transform. Animatable. */

@property CATransform3D transform;
```
CATransform3D是一个比较复杂的类型：
``` cpp
/* Homogeneous three-dimensional transforms. */

struct CATransform3D
{
  CGFloat m11, m12, m13, m14;
  CGFloat m21, m22, m23, m24;
  CGFloat m31, m32, m33, m34;
  CGFloat m41, m42, m43, m44;
};
```
这里可以看到是一个四维矩阵。而苹果提供了这样的一个接口来作为快捷的
