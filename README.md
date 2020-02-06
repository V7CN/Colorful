# Colorful


- [Colorful自述](#Colorful自述)
- [快速开始](#快速开始)
- [概览](#概览)
    - [文件目录](#一、文件目录)
        - [像素](#1.像素)
        - [像素集](#2.像素集)
        - [光谱分布](#3.光谱分布)
        - [色匹配函数、相机光谱敏感度函数](#4.色匹配函数、相机光谱敏感度函数)
        - [转换矩阵](#5.转换矩阵)
    - [数据结构](#二、数据结构)
    - [色彩管理](#三、色彩管理)
      - [Gamma校正](#1.Gamma校正)
      - [色彩空间转换](#2.色彩空间转换)
      - [例：一个完整的色彩管理](#3.例：一个完整的色彩管理)
  - [色彩模型变换](#四、色彩模型变换)
    - [光谱 $\longrightarrow$ XYZ](#1.光谱-$\longrightarrow$-XYZ)
    - [XYZ $\longleftrightarrow$ L\*a\*b\*](#2.2.XYZ-$\longleftrightarrow$-L\*a\*b\*)
    - [XYZv $\longleftrightarrow$ L\*u'v' $\longleftrightarrow$ L\*u\*v\*](#3.XYZ-$\longleftrightarrow$-L\*u'v'-$\longleftrightarrow$-L\*u\*v\*)
    - [L\*a\*b\*、L\*u\*v\* $\longleftrightarrow$ LCH](#4.L\*a\*b\*、L\*u\*v\*-$\longleftrightarrow$-LCH)
    - [LAB $\longrightarrow$ DE00](#5.LAB-$\longrightarrow$-DE00)
  - [绘图](#五、绘图)
    - [1.光谱绘图](#1.光谱绘图)
    - [色彩模型变换](#2.相机谱特性绘图)
    - [CIE1931 xy 色度图](#3.CIE-1931-xy色度图)
    - [CIE1976 uv 色度图](#4.CIE-1976-uv色度图)
  - [色彩模型变换](#六、LUT生成器)
    - [1DLUT](#1.1DLUT)
    - [3DLUT](#2.3DLUT)



  

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

## 一、文件目录

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


## 二、数据结构

Colorful主要用列表存储数据。其中有5类数据，具体计算的数学表达可以参考专栏首文：[色彩科学学习笔记——从摄影出发](https://zhuanlan.zhihu.com/p/72587460)

### 1.像素
3x1列向量。

可以存储XYZ、RGB、L*a*b、L*u*v和L*u‘v’。例如

    pixel = {1, 1, 1}

往往用函数直接操作它。比如将XYZ转换到LAB：

    XYZ2LAB[pixel]

值得一提，当像素的色彩空间是XYZ或RGB，则数据尽量不要小于0，不要超过1。

### 2.像素集
Nx3列表。

    pixels = {{2, -1, 1},{1, .5, .2}}

在Mathematica中，该列表是Nx3的，相当于两个三维行向量——Mathematica并不支持“列向量”。所以，在做计算的时候，我们不对数据矩阵直接做矩阵乘，而是将其看作是“像素的集合”，用函数+Map处理。

在定义函数的时候，几乎都是传入像素，而不是像素集（绘图函数除外）。在操作像素集时，只需使用Map函数即可，这样就能让函数逐次处理像素集中的像素，并返回处理完的像素集。Mma提供了一个Map函数的语法糖，下面三句话等价：

    Map[XYZ2xyY, pixels]
    XYZ2xyY /@ pixels
    XYZ2xyY[#]& /@ pixels

这个技巧非常常用，应当熟练！

### 3.光谱分布
69x1列表

光谱计算一般涉及到函数内积。为方便计算，在Colorful中，光谱被定义为69维的列向量，波长范围为380-720nm，波长步长为5nm。

输入“LSD65”并运行，可以输出D65的光谱分布。

<center><img src="https://github.com/V7CN/Colorful/blob/master/img/README02.png?raw=true" width="90%" h></center>

Mathematica默认会把结果打印出来。如果你定义或导入一个庞大的列表，在表达式最后加一个分号，就能阻止输出结果，看起来清爽：

<center><img src="https://github.com/V7CN/Colorful/blob/master/img/README03.png?raw=true" width="40%" h></center>

### 4.色匹配函数、相机光谱敏感度函数
3x69列表

内置有色匹配函数CMFXYZ，可以用简单的矩阵乘将光谱转化为XYZ色彩：

    CMFXYZ.LSD65;
    Draw1976XYZ[ {%*10} ]

第一句是计算D65光源的XYZ值；

第二句是将上一句的结果（用%表示）打印到LUV坐标系中。乘10的意义是提升曝光，否则显示的色点会很暗。

### 5.转换矩阵
3x3列表

将转换矩阵与像素作矩阵乘，可以将XYZ数据在各色彩空间中变换。在Colorful中，内置有一些转换矩阵，比如CMXYZ2sRGB：

    CMXYZ2sRGB.{1, 1, 1}

由于“像素集”并不是列向量集，所以不能用“转换矩阵”乘“像素集”。这种运算有很大的局限性：早期我常用转换矩阵来变换色彩空间，但在做批量处理时要做大量的转置运算，所以非常不便。我计划将这种朴素的色彩管理方式淘汰掉，替代为专用色彩管理函数，如下。

## 三、色彩管理
——非常重要！

色彩管理相关的函数将以CM为前缀。Colorful的特色是，将Gamma、基色（Primaries）、白点（White Point）的变换分离开。

确实，色彩空间的转换应当同时包含Gamma、基色、白点的转换，但Colorful并非生产工具，而是计算工具。分开做的好处是可以带来极大的灵活性，坏处是在使用他们的时候，你必须非常明确自己在做什么！

### 1.Gamma校正
在Mathematica中，你可以对列表直接作幂次运算。这样做的好处是，可以有基本的数学灵活性，并将处理过程明确地写到代码里。例如：

    {{.1, .2, .3}}^(2.2)
### 2.色彩空间转换
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

### 3.例：一个完整的色彩管理

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


## 四、色彩模型变换
色彩管理本质上仍是XYZ空间内的变换，是线性变换（并包含Gamma运算）。而色彩模型间的变换是XYZ、LAB、LUV、LCH坐标间的变换，是非线性变换。而从光谱计算颜色也是一种“模型变换”，所以也纳入此列。
### 1.光谱 $\longrightarrow$ XYZ
从光谱到色彩，有两个路径。一是用标准人眼色匹配函数CMFXYZ去乘，而标准人眼色匹配函数有很多组，常用CIE 1931 2-deg Standard Observer；二是用相机或胶片的光谱敏感度函数去乘，并用相对正确的色彩矩阵做色彩管理，[这篇文章](https://zhuanlan.zhihu.com/p/77471401)使用过第二种路径，你可以亲自尝试将相机结果与人眼结果做对比。

这里用人眼色匹配函数举例。前文提到过，LSD65是光谱分布，只需用代码：

    CMFXYZ.LSD65

即按CIE 1931 2-deg标准将光谱变换成XYZ。

### 2.XYZ $\longleftrightarrow$ L\*a\*b\*
为方便书写，在Colorful中，用LAB表示，常用函数是
* XYZ2LAB
* LAB2XYZ

请一定记住，
**LAB是一个白点相关的色彩模型！** 
所以两个函数都有一个重要的选项："refWP"。选项的默认值是"D65"。举例：

    XYZ2LAB[{0.5, 0.5, 0.5}, "refWP" -> "D65"];
    LAB2XYZ[%, "refWP" -> "D65"]

这是一个套娃测试，从XYZ转到LAB再转回来，结果与输入相同。

### 3.XYZ $\longleftrightarrow$ L\*u'v' $\longleftrightarrow$ L\*u\*v\*

### 4.L\*a\*b\*、L\*u\*v\* $\longleftrightarrow$ LCH
### 5.LAB $\longrightarrow$ DE00

## 五、绘图
### 1.光谱绘图
### 2.相机谱特性绘图
### 3.CIE 1931 xy色度图
### 4.CIE 1976 uv色度图
## 六、LUT生成器
### 1.1DLUT
### 2.3DLUT
