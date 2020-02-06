# Colorful


- [Colorful自述](#Colorful自述)
- [快速开始](#快速开始)
- [概览](#概览)
    - [文件目录](#文件目录)
        - [像素](#像素)
        - [像素集](#像素集)
        - [光谱分布](#光谱分布)
        - [色匹配函数、相机光谱敏感度函数](#色匹配函数、相机光谱敏感度函数)
        - [转换矩阵](#转换矩阵)
    - [数据结构](#数据结构)
    - [色彩管理](#色彩管理)
      - [Gamma校正](#Gamma校正)
      - [色彩空间转换](#色彩空间转换)
      - [例：一个完整的色彩管理](#例：一个完整的色彩管理)
  - [色彩模型变换](#色彩模型变换)
    - [光谱 -> XYZ](#光谱-->-XYZ)
    - [XYZ <-> L\*a\*b\*](#XYZ-<->-L\*a\*b\*)
    - [XYZ <-> L\*u'v' <-> L\*u\*v\*](#XYZ-<->-L\*u'v'-<->-L\*u\*v\*)
    - [L\*a\*b\*、L\*u\*v\* -> LCH](#L\*a\*b\*、L\*u\*v\*-->-LCH)
    - [LAB -> DE00](#LAB-->-DE00)
  - [绘图](#绘图)
    - [1.光谱绘图](#光谱绘图)
    - [色彩模型变换](#相机谱特性绘图)
    - [CIE1931 xy 色度图](#CIE-1931-xy色度图)
    - [CIE1976 uv 色度图](#CIE-1976-uv色度图)
  - [色彩模型变换](#LUT生成器)
    - [1DLUT](#1DLUT)
    - [3DLUT](#3DLUT)



  

# Colorful自述

写到减色法那一篇时，有朋友反映说，此时知识已经变得很抽象了，进一步用文字描述的话，很难快速看懂。所以把近一年写过的工具代码整理了一下，做成一个计算框架，起名Colorful，开源出来，链接在此。

其实已经有一些很棒的项目，比如goCIE，比如BruceLindbloom的网页计算器、表格计算器，甚至Mac自带的ColorSync工具也能做简单的色彩计算。但是一个可以在Mathematica中运行的计算框架应该更吸引人：可以批量处理数据，无需担心精度，爽快的列表操作，有现成的数据处理函数，美如画的绘图等等。

Colorful并不复杂，它更像是一个简单的函数集合。它能做 XYZ <-> RGB <-> LUV <-> LAB的互换，计算色差（DE00），还能从光谱计算颜色并转换到任意色彩空间中去。其中有一个完整的色彩管理，你可以随意定义新的色彩空间和白点，甚至把不同的基色与白点随意搭配。Colorful还可以一键做光谱、色匹配函数、xyY、Luv的绘图。再加上Mathematica丰富的函数和超棒的文档，赋能一些小想法还是可以的！但水平有限，代码不太好看，而且都是朴素实现，效率勉强能用。

另外，因为自己写这些代码的初心就是解决实际问题（如胶片校色、胶片模拟等），常用于上百M的16bit文件，Mathematica处理大文件实在太慢，所以实现了简单的LUT生成器。支持任意精度的3DLUT、1DLUT，可以随意定义变换函数（不过高精度生成的效率嘛...）。所以你可以在PS中调用LUT，快速预览实验结果。

欢迎所有意见，不过git不太会用..尽我所能地维护，希望一起把Colorful做得更好使。

# 快速开始
我的运行环境是Mathematica 12。更老的版本应该也能正常运行，还没尝试。

用Mathematica打开Colorful.nb，点选计算-计算笔记本，运行整个文件，将其载入内核。
新建一个新的笔记本。由于文件运行在同一个内核中，所以可以在新的文件中作计算了。
写入代码，Shift+Enter

    Draw1931XYZ[{{1, 1, 1}}]

这一句的意思是，将像素{X=1，Y=1，Z=1}打印在CIE 1931色度图上：

<center><img src="https://github.com/V7CN/Colorful/blob/master/img/README01.png?raw=true" width="50%" h></center>


# 概览

## 文件目录

框架下载下来是一个文件夹，目录如下：

.  
+--Colorful.nb  
+--Lut.nb  
+--Data/  
|   +--Colorimetry/  
|   +--LightSource/  
+--Workspace/

Colorful.nb所在的文件夹是根目录，运行后记录在全局变量gCSDir中。

Workspace作为程序的默认工作目录，调用Import、Export函数时，默认从Workspace中导入、导出文件。

Data是计算框架的数据源，每次都会重新调用。目前里面保存着CIE 1931 2deg色匹配函数，和一些标准光源的光谱定义。


## 数据结构

Colorful主要用列表存储数据。其中有5类数据，具体计算的数学表达可以参考专栏首文：[色彩科学学习笔记——从摄影出发](https://zhuanlan.zhihu.com/p/72587460)

### 像素
3x1列向量。

可以存储XYZ、RGB、L*a*b、L*u*v和L*u‘v’。例如

    pixel = {1, 1, 1}

往往用函数直接操作它。比如将XYZ转换到LAB：

    XYZ2LAB[pixel]

值得一提，当像素的色彩空间是XYZ或RGB，则数据尽量不要小于0，不要超过1。

### 像素集
Nx3列表。

    pixels = {{2, -1, 1},{1, .5, .2}}

在Mathematica中，该列表是Nx3的，相当于两个三维行向量——Mathematica并不支持“列向量”。所以，在做计算的时候，我们不对数据矩阵直接做矩阵乘，而是将其看作是“像素的集合”，用函数+Map处理。

在定义函数的时候，几乎都是传入像素，而不是像素集（绘图函数除外）。在操作像素集时，只需使用Map函数即可，这样就能让函数逐次处理像素集中的像素，并返回处理完的像素集。Mma提供了一个Map函数的语法糖，下面三句话等价：

    Map[XYZ2xyY, pixels]
    XYZ2xyY /@ pixels
    XYZ2xyY[#]& /@ pixels

这个技巧非常常用，应当熟练！

### 光谱分布
69x1列表

光谱计算一般涉及到函数内积。为方便计算，在Colorful中，光谱被定义为69维的列向量，波长范围为380-720nm，波长步长为5nm。

输入“LSD65”并运行，可以输出D65的光谱分布。

<center><img src="https://github.com/V7CN/Colorful/blob/master/img/README02.png?raw=true" width="90%" h></center>

Mathematica默认会把结果打印出来。如果你定义或导入一个庞大的列表，在表达式最后加一个分号，就能阻止输出结果，看起来清爽：

<center><img src="https://github.com/V7CN/Colorful/blob/master/img/README03.png?raw=true" width="40%" h></center>

### 色匹配函数、相机光谱敏感度函数
3x69列表

内置有色匹配函数CMFXYZ，可以用简单的矩阵乘将光谱转化为XYZ色彩：

    CMFXYZ.LSD65;
    Draw1976XYZ[ {%*10} ]

第一句是计算D65光源的XYZ值；

第二句是将上一句的结果（用%表示）打印到LUV坐标系中。乘10的意义是提升曝光，否则显示的色点会很暗。

### 转换矩阵
3x3列表

将转换矩阵与像素作矩阵乘，可以将XYZ数据在各色彩空间中变换。在Colorful中，内置有一些转换矩阵，比如CMXYZ2sRGB：

    CMXYZ2sRGB.{1, 1, 1}

由于“像素集”并不是列向量集，所以不能用“转换矩阵”乘“像素集”。这种运算有很大的局限性：早期我常用转换矩阵来变换色彩空间，但在做批量处理时要做大量的转置运算，所以非常不便。我计划将这种朴素的色彩管理方式淘汰掉，替代为专用色彩管理函数，如下。

## 色彩管理
——非常重要！

色彩管理相关的函数将以CM为前缀。Colorful的特色是，将Gamma、基色（Primaries）、白点（White Point）的变换分离开。

确实，色彩空间的转换应当同时包含Gamma、基色、白点的转换，但Colorful并非生产工具，而是计算工具。分开做的好处是可以带来极大的灵活性，坏处是在使用他们的时候，你必须非常明确自己在做什么！

### Gamma校正
在Mathematica中，你可以对列表直接作幂次运算。这样做的好处是，可以有基本的数学灵活性，并将处理过程明确地写到代码里。例如：

    {{.1, .2, .3}}^(2.2)
### 色彩空间转换
可以用一个万能的函数CM解决基色和白点的转换。例如：

    CM[{0.95047, 1.00000, 1.08883},
    "iPri" -> "XYZ",
    "iWP" -> "E",
    "oPri" -> "sRGB",
    "oWP" -> "D65",
    "Clip" -> True]

这句话的意思是将某个色从XYZ空间、白点E转换为sRGB、白点D65。后面几个用箭头连接的东西是“选项”。选项不填则会调用默认值，选项之间也可以交换顺序。其中：

* "iPri" ：指输入基色，默认sRGB。
* "iWP“：指输入白点，默认D65。
* "oPri" ：指输出基色，默认sRGB。
* "oWP"：指输入白点，默认D65。
* "Clip" ：在广色域转换到窄色域空间时，往往会将色域外的数据截掉（因为其小于0或大于1）。若该选项为False，则不会把数据截掉，这样在大小色域间互换时，可以保留数据完整性（慎用，可能会导致一些问题）。

另外，你可以用基色的xy值任意添加新的色彩空间。白点也是，不过大多数白点数据已经录入其中了。

### 例：一个完整的色彩管理

问题：像素用sRGB编码，请将其从sRGB转换到AdobeRGB。

回答：

    pixel = {{1, .5, .2}}
    CM[pixel^2.2,
    "iPri" -> "sRGB",
    "oPri" -> "AdobeRGB",
    "iWP" -> "D65",
    "oWP" -> "D65",
    "Clip" -> True]^(1/2.2) 

其流程是：

1. 将数据转换到线性（作2.2次方）
2. 用CM函数转换到AdobeRGB
3. 再用Gamma编码（作1/2.2次方）


## 色彩模型变换
色彩管理本质上仍是XYZ空间内的变换，是线性变换（并包含Gamma运算）。而色彩模型间的变换是XYZ、LAB、LUV、LCH坐标间的变换，是非线性变换。而从光谱计算颜色也是一种“模型变换”，所以也纳入此列。
### 光谱 -> XYZ
从光谱到色彩，有两个路径。一是用标准人眼色匹配函数CMFXYZ去乘，而标准人眼色匹配函数有很多组，常用CIE 1931 2-deg Standard Observer；二是用相机或胶片的光谱敏感度函数去乘，并用相对正确的色彩矩阵做色彩管理，[这篇文章](https://zhuanlan.zhihu.com/p/77471401)使用过第二种路径，你可以亲自尝试将相机结果与人眼结果做对比。

这里用人眼色匹配函数举例。前文提到过，LSD65是光谱分布，只需用代码：

    CMFXYZ.LSD65

即按CIE 1931 2-deg标准将光谱变换成XYZ。

### XYZ <-> xyY
两个函数：
* XYZ2xyY
* xyY2XYZ

注意：该函数无法处理Y=0的数据。举例：

    XYZ2xyY[{1, 1, 1}]

### XYZ -> L\*a\*b\*
为方便书写，在Colorful中，用LAB表示，常用函数是
* XYZ2LAB（"refWP"默认"D65"）
* LAB2XYZ（"refWP"默认"D65"）

请一定记住，
**LAB是一个白点相关的色彩模型！** 
所以两个函数都有一个重要的选项："refWP"。选项的默认值是"D65"。举例：

    XYZ2LAB[{0.5, 0.5, 0.5}, "refWP" -> "D65"];
    LAB2XYZ[%, "refWP" -> "D65"]

这是一个套娃测试，从XYZ转到LAB再转回来，结果与输入相同。

### XYZ <-> L\*u'v' <-> L\*u\*v\*
为方便书写，在Colorful中，用Luv表示白点无关的L\*u'v'，用LUV表示白点相关的L\*u\*v\*。
值得注意，大写的LAB、LUV、LCH都是白点相关的。

有以下函数：
* XYZ2Luv、Luv2XYZ
* Luv2LUV、LUV2Luv ("refWP"默认"D65")
* XYZ2LUV、LUV2XYZ（"refWP"默认"D65"）

同样的，用一个套娃测试作示例：
    XYZ2LUV[{0.5, 0.5, 0.5}, "refWP" -> "D65"];
    LUV2XYZ[%, "refWP" -> "D65"]

### L\*a\*b\*、L\*u\*v\* -> LCH
LCH实际是一个“高级版”的HSL。不同的是，HSL是关于RGB的柱坐标变换，而LCH是关于LAB或LUV的柱坐标变换。两者的“L”和“H”有一些差别。

LCH的数据源有两种，LAB或LUV，但变换算法是相同的。所以你会见到LCH(ab)、LCH(uv)这样的记法。
为了让编程者明白自己在做什么，定义了名字不同的两个函数：

* LAB2LCH（"deg"默认False）
* LUV2LCH（"deg"默认False）

两函数都有一个选项，用来设定H是否用角度单位输出。由于Mathematica三角函数都以弧度为单位，所以默认为False，即用弧度表示H。举例：

    LAB2LCH[{100,1,1},"deg"->False]

不出意外，结果是{100,$\sqrt 2$,$\frac{\pi}{4}$}。如果想输出数值结果，在语句最后加上"//N"即可。

<center><img src="https://github.com/V7CN/Colorful/blob/master/img/README04.png?raw=true" width="70%" h></center>

因为LAB、LUV已经设置参考白点，所以此转换没有参考白点的选项。但你要明白，LCH数据也是暗含参考白点的，为了不引起混乱，暂不设反转换的函数。

### LAB -> DE00

DE00是计算色偏相对较新的标准。它也暗含参考白点。所以计算DE00的唯一方式是，先转到特定白点的LAB，再计算DE00。

举例：

    DE00LAB[{100, 0.5, 0.5}, {0.3, 0, 0}]

## 绘图
### 光谱绘图
    DrawSpectrum[LSD55]
<center><img src="https://github.com/V7CN/Colorful/blob/master/img/README05.png?raw=true" width="90%" h></center>

### 相机谱特性绘图
    DrawCMF[CMFXYZ]
<center><img src="https://github.com/V7CN/Colorful/blob/master/img/README06.png?raw=true" width="90%" h></center>

### CIE 1931 xy色度图
**绘图函数输入的是“像素集”，而不是“像素”。**
该绘图函数有两个：
* Draw1931XYZ，输入XYZ像素集，打印在xy色度图上；
* Draw1931xyY，输入xyY像素集，打印在xy色度图上。

两个函数都有四个选项：
* "PointSize"：是数据点大小，默认0.01。
* "Ymax"：是决定可打印数据点的Y的最大值。Y超过该值，便不会打印出来。默认为1。
* "Ygamma"：该函数的数据点会以真实色彩显现出来，如果数据点过于暗，可以减小该值。相当于对显示亮度作gamma校正。
* "PlotRange"：打印的横纵坐标范围，默认为{0, 0.9}一般不会改变。

举例，绘制白点E：
    Draw1931XYZ[{{1, 1, 1}}, "PointSize" -> 0.02]

<center><img src="https://github.com/V7CN/Colorful/blob/master/img/README07.png?raw=true" width="90%" h></center>

该绘图函数绘制的是三维图形。垂直于屏幕的是表示亮度的Y轴，你可以用鼠标拖动旋转它。

### CIE 1976 uv色度图
该绘图函数有一个：
* Draw1976XYZ，输入XYZ像素集，打印在uv色度图上；

函数有四个选项：
* "PointSize"：是数据点大小，默认0.01。
* "Lmax"：是决定可打印数据点的L的最大值。L超过该值，便不会打印出来。默认为100。
* "Lgamma"：该函数的数据点会以真实色彩显现出来，如果数据点过于暗，可以减小该值。相当于对显示明度作gamma校正。
* "refWP"：参考白点，它不会改变数据点的位置，但会细微改变点的颜色。默认为"E"

举例，绘制白点E：
    Draw1976XYZ[{{1, 1, 1}}, "PointSize" -> 0.02]

<center><img src="https://github.com/V7CN/Colorful/blob/master/img/README08.png?raw=true" width="90%" h></center>

## 六、LUT生成器
LUT的支持单独写在LUT.nb文件中了。你需要打开LUT.nb并点击计算-计算笔记本，将其载入内核中。
该文件可独立使用。在其中，写有简单的教程和Demo。
### 1DLUT
#### 1DLUT特性
* 1DLUT仅调整每个通道的影调映射，不能作通道间的变换，如1DLUT不能作色彩空间转换。
* 正由上述原因，1DLUT可以用很小的文件实现相当高的精度，用以Gamma的变换。1DLUT的精度由LUT的大小（size）决定，常设为2的整数次方。如在16bit工作流中，设置size=65536，可以实现每个通道65536个点位，相当于每个值都被精确映射到所需值。
* PS中可以使用1DLUT。

#### LutCreator1D函数的使用方法

* 首先明确工作流的bit数、希望的精度。比如设定精度为16bit，LUT的大小为65536。
* 1DLut中的值最小为0，最大为1，所以要分别定义r、g、b各通道的变换函数，定义域、值域都是0到1。

举例：设定将gamma2.2转换成gamma1.8的函数----"gamma"

    gamma[v_] := (v^(2.2))^(1/1.8)

* 调用LutCreator1D函数。
  * 第一个参数是文件名，文件后缀设置为.cube。文件将保存到工作路径/Workspace下；
  * 第二至四个参数分别对应r、g、b通道的转换函数，这里用同样的函数"gamma"；
  * 第五个参数是LUT的大小。

举例：

    LutCreator1D["gamma2.2-1.8.cube", gamma, gamma, gamma, 65536]

可以在工作路径/Workspace下找到该lut，在PS中调用该LUT，可以发现图像的gamma变化（图像变暗）:

<center><img src="https://github.com/V7CN/Colorful/blob/master/img/README09.png?raw=true" width="90%" h></center>
验证成功。

### 3DLUT
#### 3DLUT特性

* 3DLUT在精度上往往不及1DLUT，但是其功能更多：可以进行通道间的变换，这意味着可以囊括相当复杂的色彩变换，如色彩空间的转换。
* 3DLUT非常庞大，而且文件大小相对于点位提高指数上升：典型的64个点位的LUT将占据7.8M的大小，256点位的LUT将占据百余兆的空间。所以，影调变换尽量用1DLUT，遇到复杂的色彩变换时再使用3DLUT，配合使用。
* 正是因为精度有限，所以必须注重3DLUT的使用技巧，如使用gamma提升精度等。

#### LutCreator3D函数的使用方法
* 首先明确希望的精度。快速生成往往设size=16；普通使用可以设size=64；高精度可设size=256，但需要花很长时间生成。
* 定义对"像素"的变换函数。这里就可以使用Colorful框架中的大量函数了。但是必须对流程中的每一步有正确的理解！

举例：将sRGB转换为AdobeRGB（这里需要Colorful框架支持），先将数据变成线性的，并调用基色、白点变换，最后应用gamma校正。

    sRGBtoAdobeRGB[{r_, g_, b_}] :=
    CM[{r, g, b}^2.2,
    "iPri" -> "sRGB",
    "oPri" -> "AdobeRGB",
    "iWP" -> "D65",
    "oWP" -> "D65",
    "Clip" -> True]^(1/2.2)

* 调用LutCreator3D函数。
  * 第一个参数是文件名，文件后缀设置为.cube。文件将保存到工作路径/Workspace下；
  * 第二个参数是对像素进行操作的函数。这里用示例函数"sRGBtoAdobeRGB"；
  * 第三个参数是LUT的大小，这里使用16。

举例：

    LutCreator3D["sRGB to AdobeRGB.cube", sRGBtoAdobeRGB, 16]

这时可以从PS中打开一个sRGB的文件，调用该LUT，并指定色彩空间为AdobeRGB：

<center><img src="https://github.com/V7CN/Colorful/blob/master/img/README10.png?raw=true" width="90%" h></center>

可以发现与图片刚打开时观感相同，验证成功。

自此，Colorful的功能简介结束了。