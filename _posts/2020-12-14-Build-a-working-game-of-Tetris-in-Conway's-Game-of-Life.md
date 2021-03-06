---
layout: article
title:  "Build a working game of Tetris in Conway's Game of Life"
tags: "瞎折腾"
article_header:
  type: overlay
  background_image:
    src: /pics/pic1.JPG
---
用康威来造计算机辣~~~~~
<!--more-->
## 什么是Conway's Game of Life
Conway's Game of Life，康威生命游戏，是UK科学家康威在1970年发明的细胞自动机。
在生命游戏中，有着若干规则：

> * 每个细胞有两种状态 - 存活或死亡，每个细胞与以自身为中心的周围八格细胞产生互动（如图，黑色为存活，白色为死亡）
* 当前细胞为存活状态时，当周围的存活细胞低于2个时（不包含2个），该细胞变成死亡状态。（模拟生命数量稀少）
* 当前细胞为存活状态时，当周围有2个或3个存活细胞时，该细胞保持原样。
* 当前细胞为存活状态时，当周围有超过3个存活细胞时，该细胞变成死亡状态。（模拟生命数量过多）
* 当前细胞为死亡状态时，当周围有3个存活细胞时，该细胞变成存活状态。（模拟繁殖）

看起来很简单，但是简单的规则可以生成很多类型的物体：
### 稳定状态

![-c140](https://upload.wikimedia.org/wikipedia/commons/thumb/f/f4/Game_of_life_loaf.svg/120px-Game_of_life_loaf.svg.png)
<center><font size=2 color=#009688>面包</font></center>

这种类型的生物在外界条件不变时可以一直维持初始形态。
### 振荡器(oscillator)

![-c140](https://pic1.zhimg.com/v2-b2e23527db501d3cafb6ee9078f8f566_b.webp)
<center><font size=2 color=#009688>信号灯(周期=2)</font></center>

![-c140](https://upload.wikimedia.org/wikipedia/commons/thumb/0/07/Game_of_life_pulsar.gif/120px-Game_of_life_pulsar.gif)
<center><font size=2 color=#009688>脉冲星(周期=3)</font></center>

振荡器从初始状态开始，在有限图形之间切换，周而复始。
### 可移动振荡器

![-c140](https://upload.wikimedia.org/wikipedia/commons/3/37/Game_of_life_animated_LWSS.gif)
<center><font size=2 color=#009688>太空船(周期=4)</font></center>

图示是一个从初始状态开始在有限图形之间切换的振荡器，和上面的不同的是，这个振荡器是可以移动的。

游戏的基本物体就只有这几种，但是人们后面发现了更多更为复杂的变化：

![-c140](https://upload.wikimedia.org/wikipedia/commons/e/e5/Gospers_glider_gun.gif)
<center><font size=2 color=#009688>康威生命游戏中的一种可持续繁殖模式：“高斯帕机枪”不断制造“滑翔机”</font></center>

上面这个范例给我们展现了康威游戏的更多可能性：可持续繁殖的物体，或者说，生命。
事实上，在GUN Emacs编辑器中就包含了一个生命游戏，我们可以任意设定任意格子的细胞的状态，从而观察该格子状态改变对于全局的状态影响。
但是简单的该格子用于制造一个可靠的可运行硬件显然是不太可能的，这时候我们就需要用到前人的智慧-[OTCA metapixel](https://www.conwaylife.com/wiki/OTCA_metapixel)

## OTCA metapixel
OTCA metapixel看名字可能会让人很疑惑是什么东西。但是就像你在平时编写代码一样，我们不会直接使用硬件相关的底层api来编写我们的应用程序，我们使用别人的封装高层代码来进行我们的编码工作。这里的OTCA就是这样一个高等级的封装。
![-c500](https://picb.zhimg.com/80/v2-196608519c26fd05d41b54d823743cb9_720w.jpg)
<center><font size=2 color=#009688>死亡和存活的OTCA metapixel</font></center>
图片可能很模糊，我们将视角放到左下角看看关键点
![-c500](media/15977422036557/15979092311410.jpg)
<center><font size=2 color=#009688>OTCA左下角</font></center>

通过ON-OFF我们可以设置该细胞的生存还是死亡，而右边有个框，S和B都对应0-8一共9个开关，这些开关是什么意思呢？

原生的康威生命游戏主要由两条规则构建：
1. 死亡细胞周围有3个存活细胞时就诞生为新细胞
2. 存活细胞周围有2或3个存活细胞时就保持存活，否则死亡。

这两条规则可以简记为B3/S23（其中，B代表出生，S代表存活）。通过修改B或S 后面的数字，就可以创建出不同的生命游戏。例如，把B1、B2、S3这三个开关打开，我们就创建了规则B12/S3，这意味着：死亡细胞周围有1或2个存活细胞时才诞生为新细胞，而存活细胞周围有3个存活细胞时才保持存活。

使用OTCA metapixel时，我们可以通过小尾巴形状的图形开关（开和关的状态在微观结构上只相差一个细胞）设置好具体规则，然后把一组OTCA metapixel单元细胞平铺成矩阵样的网格。实际运行时，每个单元细胞会自动检查周围8个邻接单元细胞的死亡或存活状态，然后根据规则开关的设定来决定自己是生是死。

看起来，每个OTCA单元细胞都是一个可以配置的细胞自动机，那么，实际运行起来时，每一个OTCA细胞是怎么监听邻接的细胞的状态的呢？
![-c500](media/15977422036557/15979103408608.jpg)
<center><font size=2 color=#009688>OTCA metapixel各功能区域</font></center>

OTCA细胞有八个输入输出用的IO口（每个边2个，分别用于输入和输出）。通过这些接口，利用上文描述过的滑翔机作为传递的消息载体，将自己的状态通知给8个邻接细胞。

左侧的规则控制面板旁边，有一个纵向的消息信道。这其实是一个小型的，带同步功能的指令分发器和接收器。

围绕整个细胞一圈的，是一条空旷的的正方形轨道。在细胞运行时，有九个滑翔机（LWSS）绕着整个细胞一圈。LWSS的作用是依次查询八个接口，查看周边细胞的存活情况，并将计数与控制面板上预先设置的条件相对比，用以激活后续指令。

OTCA的上边框和右边框内部有着两条状态射线枪阵列，一旦接受到激活指令，射线枪阵列就会从上面和右边两个方向发射枪弹，将整个细胞填满。

OTCA metapixel将康威生命游戏的尺度从单个1x1的微观细胞扩展到了2058x2058的宏观单元细胞，同时还支持游戏规则的灵活定义。在二维平面平铺开来的OTCA metapixel单元细胞矩阵，运行起来后，就是一个宏观层面的生命游戏。这的确是生命游戏中的生命游戏！

同时，通过使用OTCA细胞，我们可以方便的自定义自己的硬件条件，避免了使用微观细胞的复杂繁琐。

## VarLife
为了适配自己想要的需求，即做出一个可以游玩俄罗斯方块的计算机，我们设计了一个多规则生命游戏-VarLife。

多规则生命游戏的特点是每个细胞遵循的邻接计数规则可以不同。游戏中，我们用不同颜色来区分规则不同的细胞。具体规则如下：

* B/S规则(black/white)，type1：无论何时背景中都不会诞生新的存活细胞。一般用于充当buffer
* B1/S规则(blue/cyan)，type2：蓝色死亡细胞周围有1个任意颜色存活细胞时即诞生为青色存活细胞；任何青色存活细胞总会在下一次迭代时死亡。用于传递信号的主要细胞类型。
* B2/S规则(green/yellow)，type3：绿色死亡细胞周围有2个任意颜色存活细胞时即诞生为黄色存活细胞；任何黄色存活细胞总会在下一次迭代时死亡。一般用于信号控制，确保不会回传。
* B12/S1规则(red/green)，type4：红色死亡细胞周围有1个或2个任意颜色存活细胞时即诞生为橙色存活细胞；橙色存活细胞周围有1个任意颜色存活细胞时即保持存活，否则死亡。在一些特殊情况下被使用，例如转发信号&存储数据

### VarLife Base基础设计
#### 导线设计
想要设计出计算机，最基本的组件就是在不同组成部分之间传递信息的导线。由于这里的最小单位是正方形的细胞，因此不同的角度的传播都需要设计不同的导线来传递信号

##### 直接传输&&90˚传输
![-c200](https://i.stack.imgur.com/peFh7.gif)
<center><font size=2 color=#009688>直线&直角导线</font></center>

![-c](media/15977422036557/15979788852020.jpg)
<center><font size=2 color=#009688>直线导线传播</font></center>

直线传输采用了中间为type2，外面包裹一层type3，当信号从左边传输来（即中间变为青色存活时），右边的蓝色死亡type2细胞周围只有一个细胞存活，则自己变为青色。而左边的蓝色死亡type2细胞，由于外围type3有滞后两位的黄色存活type3，导致该细胞周围有三个存活细胞，则不会变为青色，保证了传输的单向性。

<center><img src="media/15977422036557/15979798850708.jpg" width="150"/><img src="media/15977422036557/15979815445842.jpg" width="150"><img src="media/15977422036557/15979919927009.jpg" width="150"></center>
<center><font size=2 color=#009688>直角传输的三个过程</font></center>

直角传输则是在到达直角后，原有的青色细胞自动消亡，外围的滞后两格的黄色存活type3激发了直角另一边的中间type2由蓝色死亡转变为青色存活。当然这里在青色存活传播一格之后会发生一次反向传播，但是信号并不会影响到其他的部分，因为周围的type3每次需要接受2个存活细胞才会从绿色死亡转变为黄色存活，而同时滞后2格的黄色存活type3确保了反向传播出去的信号不会发生二次传播。

##### 其他类型的传播
![-c150](https://i.stack.imgur.com/88oxH.gif)
<center><font size=2 color=#009688>信号分离</font></center>

![-c150](https://i.stack.imgur.com/0UKff.gif)
<center><font size=2 color=#009688>45˚传播</font></center>

#### 门
除开传播信号必备的导线外，为了实现复杂的逻辑，我们还需要用OTCA来实现必备的逻辑门。

由于门类型比较多，我们这里只选用一个基础的门作为例子：
![-c150](https://i.stack.imgur.com/Llur8.gif)
<center><font size=2 color=#009688>AND-NOT门</font></center>

AND-NOT门，ANT门，只有在B没有传来信号时，A才可以通过

#### 延迟器
可能大家会注意到，这里的信号传输是恒定速度的，我们称为一个tick。
为了使传输的信号符合我们的预期，需要有一定周期的延迟期来对信号同步性进行管理，使得整个系统运行符合预期。所以这里我们设计了一系列的延迟器，这里仅列举一个例子
![-c200](https://i.stack.imgur.com/OaGIt.gif)
<center><font size=2 color=#009688>一个周期为4的延迟器</font></center>

## 硬件设计
首先看一下总览：
![-c500](https://i.stack.imgur.com/JXmnf.png)
这个设计图即是我们这次的硬件架构，下面对于其中不同的部件进行介绍。
### Demultiplexer
Demultiplexer，即分用器，是Ram，Rom，ALU等模块的必须组件。下面是一个典型的分用器：
![](media/15977422036557/15985093204093.jpg)
从图中可以清楚的看到分用器的作用，即将一个输入的信号转化为多个输出信号，或者说，将单路输入信号转化为多路输出信号。一个分用器一般由三部分构成：从串行到并行的多路转换器、信号检查器和时钟信号分离器。怎么用OTCA设计一个分用器呢？

首先我们先着眼于第一个部分：将串行数据转化为并行数据。来看我们的实际实现：
![-c600](https://imgur.com/v6iX5d9.png)
还记得上面有介绍的延迟器吗，这里就用到了延迟器。通过一个11tick的延迟器（上图右下），我们可以将信号按照11个tick的间隔分离开来，配合上时间信号对数据进行转换。

然后，需要的就是在每个点时间信号和数据信号的判断，即这个点到底要不要输出信号。为了实现这点，我们使用到了ANT和AND门（两个信号相遇输出0或1），为了这样的目的可以设计出这样的比较器：
![-c400](https://imgur.com/KAtnrKI.png)

最终，我们将始终信号按照11tick的周期进行分离，将很多的信号比较器进行堆叠，即可获得一个multiplexer：
![-c400](https://imgur.com/hpgUufI.png)

### ROM
ROM，即可以理解为电脑的磁盘。在传统的计算机架构里面，我们可以用任意一个地址来从ROM中取得对应地址的数据。

在上面我们实现了一个分用器和多路复用器，这里我们使用多路复用器来将时钟信号定位到指令之一。接下来我们需要使用导线和OR门来生成信号。这些交叉的导线让时钟信号可以沿着指令的58位一直向下传播，同时允许生成的信号向下并行传播来输出数据。

![-c400](https://imgur.com/Nlj8B2F.png)

然后简单的通过将并行的数据转换为串行数据，ROM即完成搭建：
![-c500](https://imgur.com/rwF6CL9.png)

### SRL,SL,SRA
上面简单介绍了使用基本的门和简单的延迟即可构造的分用器、多路复用器和ROM。但是计算机里面想要的数据进行计算的话，还需要使用到位操作。为了实现位操作，我们需要实现更加复杂的逻辑门。

对于SL(shift left logical)和SRL(shift right logical)来说，我们需要实现两点：
1. 确保12个最高有效不打开（否则会一直输出0）
2. 根据4个最低有效位将数据延迟正确的数量

上面这两点可以通过AND/ANT门和一个多路复用器来实现：
![-c500](https://imgur.com/wtAkNw1.png)

SRA(shift right arithmetic)稍微有点不同，因为我们需要在移位的过程中复制符号位。为此，我们将时钟信号与符号位进行“与”运算，然后使用分线器和“或”门将输出复制：
![-c500](https://imgur.com/GwH8oTJ.png)

### Set-Reset(SR) latch
处理器很多部分的功能都依赖存储数据的功能。使用如图的功能我们可以简单的制作一个SR锁存器：
![-c500](https://imgur.com/W7eNmfr.png)

### Synchronizer
通过上面的分用器将串行数据转化为并行数据，然后通过SR锁存器的设置，我们可以存储数据的所有位。然后，如果要再读取数据，我们可以读取然后延迟再通过多路复用器合成数据。这使得我们可以在等待一个数据的时候存储其他多个数据，从而让不同时间到达的数据进行同步：
![-c500](https://imgur.com/fRgFuAR.png)

### Read Counter
我们的设备需要保持计算我们读写了多少次RAM，这里就需要用到Read Counter了。我们构造一个类似于SR锁存器的结构：T flip flop（T触发器）。当T触发器收到一个输入时，会改变自己的状态。当发生一次从on到off的转换时，会发送一次脉冲输出，将会触发另一个T触发器来形成一个2bit的计数器。
![-c300](https://imgur.com/ayN556Y.png)
为了制作读取计数器，我们需要使用两个ANT门将计数器设置为适当的寻址模式，并使用计数器的输出信号来决定将时钟信号定向到何处：ALU或RAM。
![-c400](https://imgur.com/Zf8t5PH.png)

### Read Queue
读取队列需要跟踪哪个读取计数器向RAM发送了输入，以便可以将RAM的输出发送到正确的位置。 为此，我们使用一些SR锁存器：每个输入一个锁存器。 当信号从读取计数器发送到RAM时，时钟信号将被分割并设置计数器的SR锁存器。 然后，RAM的输出与SR锁存器进行AND，RAM的时钟信号将SR锁存器复位。
![-c800](https://imgur.com/EkqUHae.png)

### ALU
在CPU的实际硬件当中，所有的运算都有ALU来承担。
![-c800](https://imgur.com/mC6tMoL.png)
如图，最下面标红的分别是ALU的不同op，ALU会根据传入的opcode来调用不同的指令，即使用不同的opcode对应的SR锁存器。接下来，将第一个和第二个自变量的值与SR锁存器进行AND，然后传递到逻辑电路。时钟信号在锁存器通过时将其重置，以便可以再次使用ALU。

### RAM
RAM，random access memory,可以参考当前计算机体系的内存，当断电时其中存储信息将全部丢失。
RAM是整个系统里面最为复杂的部分，我们需要对里面的每个SR锁存器进行非常具体的控制。为了读取，地址被发送到多路复用器中并发送到RAM单元。RAM单元将它们并行存储的数据输出，然后转换为串行并输出。为了进行写入，将地址发送到不同的多路复用器中，将要写入的数据从串行转换为并行，并且RAM单元在整个RAM中传播信号。

每个22x22元像素RAM单元都具有以下基本结构：
![-c300](https://imgur.com/zmjUg6p.png)
将这些单元组合到一起：
![-c500](https://imgur.com/ytVtD1k.png)

### 硬件组合
最后简单的组合一起就可以了(只需要注意亿点点细节就可以了)：
![-c700](https://imgur.com/SRnISYL.png)