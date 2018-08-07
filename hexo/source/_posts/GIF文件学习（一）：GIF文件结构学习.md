---
title: GIF文件学习（一）：GIF文件结构学习
---

本文对GIF文件的存储方式进行分析与讲解，本文参考文章地址为：[gif 格式图片详细解析](https://blog.csdn.net/wzy198852/article/details/17266507)

# 一.概述

GIF图像是有CompuServe公司开发的图形文件格式，GIF图像是基于颜色列表的（即GIF文件中存储的数据为该点的颜色在颜色列表中的索引值），同时GIF最多支持8位（即256色）的颜色展示。

<!--more-->

# 二.GIF文件结构

## 基础知识

GIF文件内部是按块来进行划分的，其中包括**控制块（Control Block）**和**数据块（Data Sub-blcoks）**两种。

* 控制块是用来控制数据块行为的，不同的控制块一般包含不同的控制参数。

* 数据块是用来记载数据的，数据块是一组8-bit组成的数据流，其中每个数据块的大小最大为**256字节**，其中第一个字节用于指出该数据块所含字节数，即一个数据块记录数据所用字节最多为**255字节**。以下是数据块的结构：
<table><tr><th>BYTE</th><th>7</th><th>6</th><th>5</th><th>4</th><th>3</th><th>2</th><th>1</th><th>0</th><th>BIT</th></tr><tr><th>0</th><th colspan = 8 bgcolor="green">块大小</th><th>Blcok Size - 块大小，不包含这个字节</th></tr><tr><th>1</th><th colspan = 8 bgcolor="green"></th><th rowspan = 5>Data Values - 块数据，8-bit的字符串</th></tr><tr><th>2</th><th colspan = 8 bgcolor="green"></th></tr><tr><th>...</th><th colspan = 8 bgcolor="green"></th></tr><tr><th>254</th><th colspan = 8 bgcolor="green"></th></tr><tr><th>255</th><th colspan = 8 bgcolor="green"></th></tr></table>


## GIF结构

GIF文件的结构为：**文件头（File Header）**、**GIF数据流（GIF Data Stream）**和**文件终结器（Trailer）**三部分。

接下来详细讲解每部分的构成。

# 三.GIF各部分组成

## 1.文件头

文件头包含**文件署名（Signature）**和**版本号（Version）**两部分。

### (1)文件署名

文件署名表示该文件是GIF文件，这部分由"GIF"三个字符组成：

<table><tr><th>BYTE</th><th>7</th><th>6</th><th>5</th><th>4</th><th>3</th><th>2</th><th>1</th><th>0</th><th>BIT</th></tr><tr><th>0</th><th colspan = 8 bgcolor="green">'G'(0x47)</th><th rowspan = 3>文件署名</th></tr><tr><th>1</th><th colspan = 8 bgcolor="green">'I'(0x49)</th></tr><tr><th>2</th><th colspan = 8 bgcolor="green">"F"(0x46)</th></tr></table>

### (2)版本号

版本号表示GIF版本号，有"87a"和"89a"两种：

<table><tr><th>BYTE</th><th>7</th><th>6</th><th>5</th><th>4</th><th>3</th><th>2</th><th>1</th><th>0</th><th>BIT</th></tr><tr><th>0</th><th colspan = 8 bgcolor="green">'8'(0x38)</th><th rowspan = 3>版本号</th></tr><tr><th>1</th><th colspan = 8 bgcolor="green">'7'(0x39)或'9'(0x40)</th></tr><tr><th>2</th><th colspan = 8 bgcolor="green">"a"(0x61)</th></tr></table>


## 2.GIF数据流

GIF数据流包含**逻辑屏幕标识符（Logical Screen Descriptor）**、**全局颜色列表（Global Color Table）**和**图像块（Image Block）**三个基本部分以及一些扩展部分组成，扩展部分包括：**图形控制扩展（Graphic Control Extension）**、**注释扩展（Comment Extension）**、**图形文本扩展（Plain Text Extension）**以及**应用程序扩展（Application Extension）**四部分。

### (1)逻辑屏幕标识符

该部分由**7个字节**组成，结构为：

<table><tr><th>BYTE</th><th>7</th><th>6</th><th>5</th><th>4</th><th>3</th><th>2</th><th>1</th><th>0</th><th>BIT</th></tr><tr><th>0</th><th colspan = 8 rowspan = 2 bgcolor="green">逻辑屏幕宽度</th><th rowspan = 2>像素数，定义GIF图像宽度</th></tr><tr><th>1</th></tr><tr><th>2</th><th colspan = 8 rowspan = 2 bgcolor="green">逻辑屏幕高度</th><th rowspan = 2>像素数，定义GIF图像高度</th></tr><tr><th>3</th></tr><tr><th>4</th><th bgcolor="green">m</th><th colspan = 3 bgcolor="green">cr</th><th bgcolor="green">s</th><th colspan = 3 bgcolor="green">pixel</th><th>具体描述见下</th></tr><tr><th>5</th><th colspan = 8 bgcolor="green">背景色</th><th>背景颜色（在全局颜色列表中的索引，如果没有全局颜色列表，则该值没有意义）</th></tr><tr><th>6</th><th colspan = 8 bgcolor="green">像素宽高比</th><th>像素宽高比</th></tr></table>

* m：全局颜色列表标志，当置位时表示有全局颜色列表，pixel值有意义
* cr：颜色深度，cr+1确定图像的颜色深度
* s：分类标志，如果置位表示全局颜色列表分类排序
* pixel：全局颜色列表大小，pixel+1确定颜色列表的索引数（2的pixel+1次方）

### (2)全局颜色列表

全局颜色列表需要紧跟在逻辑屏幕标识符后，颜色列表中保存颜色的索引，其中每个索引又由三个字节（代表R、G、B）组成：

<table><tr><th>BYTE</th><th>7</th><th>6</th><th>5</th><th>4</th><th>3</th><th>2</th><th>1</th><th>0</th></tr><tr><th>0</th><th colspan = 8 bgcolor="green">索引1的R值</th></tr><tr><th>1</th><th colspan = 8 bgcolor="green">索引1的G值</th></tr><tr><th>2</th><th colspan = 8 bgcolor="green">索引1的B值</th></tr></tr><tr><th>3</th><th colspan = 8 bgcolor="green">索引2的R值</th></tr><tr><th>4</th><th colspan = 8 bgcolor="green">索引2的G值</th></tr><tr><th>5</th><th colspan = 8 bgcolor="green">索引2的B值</th></tr></tr><tr><th>6</th><th colspan = 8 bgcolor="green">...</th></tr></tr></table>

### (3)图像块

图像块由**图像标识符（Image Descriptor）**、**局部颜色列表（Local Color Table）**和**基于颜色列表的图像数据（Table-Based Image Data）**组成。

#### ①图像标识符

一个GIF中可以包含多张图片，新的图像以图像标识符开始，该部分由**10个字节**组成：

<table><tr><th>BYTE</th><th>7</th><th>6</th><th>5</th><th>4</th><th>3</th><th>2</th><th>1</th><th>0</th><th>BIT</th></tr><tr><th>0</th><th bgcolor="green">0</th><th bgcolor="green">0</th><th bgcolor="green">1</th><th bgcolor="green">0</th><th bgcolor="green">1</th><th bgcolor="green">1</th><th bgcolor="green">0</th><th bgcolor="green">0</th><th>图像标识符开始，固定值',' (0x2C)</th></tr><tr><th>1</th><th colspan = 8 rowspan = 2 bgcolor="green">X方向偏移量</th><th rowspan = 8>必须限制在逻辑屏幕尺寸范围内</th></tr><tr><th>2</th></tr><tr><th>3</th><th colspan = 8 rowspan = 2 bgcolor="green">Y方向偏移量</th></tr><tr><th>4</th></tr><tr><th>5</th><th colspan = 8 rowspan = 2 bgcolor="green">图像宽度</th></tr><tr><th>6</th></tr><tr><th>7</th><th colspan = 8 rowspan = 2 bgcolor="green">图像高度</th></tr><tr><th>8</th></tr><tr><th>10</th><th bgcolor="green">m</th><th bgcolor="green">i</th><th bgcolor="green">s</th><th colspan = 2 bgcolor="green">r</th><th colspan = 3 bgcolor="green">pixel</th><th>具体描述见下</th></tr></table>

* m：局部颜色列表标志，当置位时表示图像标识符后有一个局部颜色列表，pixel值有意义
* i：交织标志，置位时图像数据使用交织方式排列，否则使用顺序排列
* s：分类标志，如果置位表示局部颜色列表分类排序
* r：保留，必须初始化为0
* pixel：局部颜色列表大小，pixel+1确定颜色列表的索引数（2的pixel+1次方）

#### ②局部颜色列表

结构格式与全部颜色列表一致：

<table><tr><th>BYTE</th><th>7</th><th>6</th><th>5</th><th>4</th><th>3</th><th>2</th><th>1</th><th>0</th></tr><tr><th>0</th><th colspan = 8 bgcolor="green">索引1的R值</th></tr><tr><th>1</th><th colspan = 8 bgcolor="green">索引1的G值</th></tr><tr><th>2</th><th colspan = 8 bgcolor="green">索引1的B值</th></tr></tr><tr><th>3</th><th colspan = 8 bgcolor="green">索引2的R值</th></tr><tr><th>4</th><th colspan = 8 bgcolor="green">索引2的G值</th></tr><tr><th>5</th><th colspan = 8 bgcolor="green">索引2的B值</th></tr></tr><tr><th>6</th><th colspan = 8 bgcolor="green">...</th></tr></tr></table>

#### ③基于颜色列表的图像数据

图像数据是由**LZW编码长度**和**N个数据块**组成的：

#### LZW编码长度

<table><tr><th>BYTE</th><th>7</th><th>6</th><th>5</th><th>4</th><th>3</th><th>2</th><th>1</th><th>0</th><th>BIT</th></tr><tr><th>0</th><th colspan = 8 bgcolor="green">LZW编码长度</th><th>LZW编码初始码表大小的位数</th></tr></table>

#### N个数据块
连续的N个数据块，其中最后一个数据块是由长度为0的数据块组成，即：

<table><tr><th>BYTE</th><th>7</th><th>6</th><th>5</th><th>4</th><th>3</th><th>2</th><th>1</th><th>0</th><th>BIT</th></tr><tr><th></th><th colspan = 8 bgcolor="green">...</th><th rowspan = 3>N组数据图像</th></tr><tr><th></th><th colspan = 8 bgcolor="green">数据块</th></tr><tr><th></th><th colspan = 8 bgcolor="green">...</th></tr><tr><th></th><th colspan = 8 bgcolor="green">0x00</th><th>数据块大小为0，作为一组数据块的收尾</th></tr></table>

### (4)图形控制扩展

该部分需要89a版本，可以放在一个*图像块*或*文本扩展块*前面，用来控制后面的图像或文本的渲染形式，结构如下：

<table><tr><th>BYTE</th><th>7</th><th>6</th><th>5</th><th>4</th><th>3</th><th>2</th><th>1</th><th>0</th><th>BIT</th></tr><tr><th>0</th><th colspan = 8 bgcolor="green">扩展块标识</th><th>扩展块开始标识，固定值0x21</th></tr><tr><th>1</th><th colspan = 8 bgcolor="green">图形控制扩展标识</th><th>图形控制扩展标识，固定值0xF9</th></tr><tr><th>2</th><th colspan = 8 bgcolor="green">块大小</th><th>不包含块终结器，固定值0x04</th></tr><tr><th>3</th><th colspan = 3 bgcolor="green">保留</th><th colspan = 3 bgcolor="green">处置方式</th><th bgcolor="green">i</th><th bgcolor="green">t</th><th>具体描述如下</th></tr><tr><th>4</th><th colspan = 8 rowspan = 2 bgcolor="green">延迟时间</th><th rowspan = 2>单位1/100秒，如果值不为1，表示暂停规定时间后再继续处理下面数据流</th></tr><tr><th>5</th></tr><tr><th>6</th><th colspan = 8 bgcolor="green">透明色索引</th><th>透明色索引值</th></tr><tr><th>7</th><th colspan = 8 bgcolor="green">块终结器</th><th>标识块终结，固定值0x00</th></tr></table>

* 处置方法：指出处置图形的方法，当值为：0 - 不使用处置方法；1 - 不处置图形，把图形从当前位置移去；2 - 回复到背景色；3 - 回复到先前状态；4-7 - 自定义
* i：用户输入标志，指出是否期待用户有输入之后才继续进行下去，置位表示期待，值否表示不期待。用户输入可以是按回车键、鼠标点击等，可以和延迟时间一起使用，在设置的延迟时间内用户有输入则马上继续进行，或者没有输入直到延迟时间到达而继续
* t：透明颜色标志，置位表示使用透明颜色

### (5)注释扩展

该部分需要89a版本，可以用来记录图形、版权、描述等任何的非图形和控制的纯文本数据(7-bit ASCII字符)，注释扩展并不影响对图象数据流的处理，解码器完全可以忽略它。存放位置可以是数据流的任何地方，最好不要妨碍控制和数据块，推荐放在数据流的开始或结尾，结构如下：

<table><tr><th>BYTE</th><th>7</th><th>6</th><th>5</th><th>4</th><th>3</th><th>2</th><th>1</th><th>0</th><th>BIT</th></tr><tr><th>0</th><th colspan = 8 bgcolor="green">扩展块标识</th><th>扩展块开始标识，固定值0x21</th></tr><tr><th>1</th><th colspan = 8 bgcolor="green">注释扩展标识</th><th>注释控制扩展标识，固定值0xFE</th></tr><tr><th></th><th colspan = 8 bgcolor="green">...</th><th rowspan = 3>N组数据块</th></tr><tr><th></th><th colspan = 8 bgcolor="green">注释块</th></tr><tr><th></th><th colspan = 8 bgcolor="green">...</th></tr><tr><th></th><th colspan = 8 bgcolor="green">0x00</th><th>数据块大小为0，作为一组数据块的收尾</th></tr></table>

### (6)图形文本扩展

该部分需要89a版本，用来绘制一个简单的文本图象，这一部分由用来绘制的纯文本数据（7-bit ASCII字符）和控制绘制的参数等组成。绘制文本借助于一个文本框（Text Grid）来定义边界，在文本框中划分多个单元格，每个字符占用一个单元，绘制时按从左到右、从上到下的顺序依次进行，直到最后一个字符或者占满整个文本框（之后的字符将被忽略，因此定义文本框的大小时应该注意到是否可以容纳整个文本），绘制文本的颜色索引使用全局颜色列表，没有则可以使用一个已经保存的前一个颜色列表。另外，图形文本扩展块也属于图形块(Graphic Rendering Block)，可以在它前面定义图形控制扩展对它的表现形式进一步修改，结构如下：

<table><tr><th>BYTE</th><th>7</th><th>6</th><th>5</th><th>4</th><th>3</th><th>2</th><th>1</th><th>0</th><th>BIT</th></tr><tr><th>0</th><th colspan = 8 bgcolor="green">扩展块标识</th><th>扩展块开始标识，固定值0x21</th></tr><tr><th>1</th><th colspan = 8 bgcolor="green">注释扩展标识</th><th>注释控制扩展标识，固定值0x01</th></tr><tr><th>2</th><th colspan = 8 bgcolor="green">块大小</th><th>块大小，固定值0x0C</th></tr><tr><th>3</th><th colspan = 8 rowspan = 2 bgcolor="green">文本框左边界位置</th><th rowspan = 2>像素值，文本框离逻辑屏幕的左边界距离</th></tr><tr><th>4</th></tr><tr><th>5</th><th colspan = 8 rowspan = 2 bgcolor="green">文本框上边界位置</th><th rowspan = 2>像素值，文本框离逻辑屏幕的上边界距离</th></tr><tr><th>6</th></tr><tr><th>7</th><th colspan = 8 rowspan = 2 bgcolor="green">文本框宽度</th><th rowspan = 2>像素值</th></tr><tr><th>8</th></tr><tr><th>9</th><th colspan = 8 rowspan = 2 bgcolor="green">文本框高度</th><th rowspan = 2>像素值</th></tr><tr><th>10</th></tr><tr><th>11</th><th colspan = 8 bgcolor="green">字符单元格宽度</th><th>像素值，单个单元格宽度</th></tr><tr><th>12</th><th colspan = 8 bgcolor="green">字符单元格高度</th><th>像素值，单个单元格高度</th></tr><tr><th>13</th><th colspan = 8 bgcolor="green">文本前景色索引</th><th>前景色在全局颜色列表中的索引</th></tr><tr><th>14</th><th colspan = 8 bgcolor="green">文本背景色索引</th><th>背景色在全局颜色列表中索引</th></tr><tr><th></th><th colspan = 8 bgcolor="green">...</th><th rowspan = 3>N组数据块</th></tr><tr><th></th><th colspan = 8 bgcolor="green">文本数据块</th></tr><tr><th></th><th colspan = 8 bgcolor="green">...</th></tr><tr><th></th><th colspan = 8 bgcolor="green">0x00</th><th>数据块大小为0，作为一组数据块的收尾</th></tr></table>

### (7)应用程序扩展

该部分需要89a版本，应用程序可以在这里定义自己的标识、信息等，结构如下：

<table><tr><th>BYTE</th><th>7</th><th>6</th><th>5</th><th>4</th><th>3</th><th>2</th><th>1</th><th>0</th><th>BIT</th></tr><tr><th>0</th><th colspan = 8 bgcolor="green">扩展块标识</th><th>扩展块开始标识，固定值0x21</th></tr><tr><th>1</th><th colspan = 8 bgcolor="green">应用程序扩展标识</th><th>应用程序扩展标识，固定值0xFF</th></tr><tr><th>2</th><th colspan = 8 bgcolor="green">块大小</th><th>块大小，固定值0x0B</th></tr><tr><th>3</th><th colspan = 8 rowspan = 8 bgcolor="green">应用程序标识符</th><th rowspan = 8>用来鉴别应用程序自身的标识（8个连续的ASCII字符）</th></tr><tr><th>4</th></tr><tr><th>5</th></tr><tr><th>6</th></tr><tr><th>7</th></tr><tr><th>8</th></tr><tr><th>9</th></tr><tr><th>10</th></tr><tr><th>11</th><th colspan = 8 rowspan = 3 bgcolor="green">应用程序鉴别码</th><th rowspan = 3>应用程序定义的特殊标识码（3个连续ASCII字符）</th></tr><tr><th>12</th></tr><tr><th>13</th></tr><tr><th></th><th colspan = 8 bgcolor="green">...</th><th rowspan = 3>N组数据块</th></tr><tr><th></th><th colspan = 8 bgcolor="green">应用程序数据</th></tr><tr><th></th><th colspan = 8 bgcolor="green">...</th></tr><tr><th></th><th colspan = 8 bgcolor="green">0x00</th><th>数据块大小为0，作为一组数据块的收尾</th></tr></table>

其中，现在最多使用的是网景公司定义的应用程序控制标识，该标识中定义了GIF播放的循环次数，结构如下：

<table><tr><th>BYTE</th><th>7</th><th>6</th><th>5</th><th>4</th><th>3</th><th>2</th><th>1</th><th>0</th><th colspan = 2>BIT</th></tr><tr><th>0</th><th colspan = 8 bgcolor="green">扩展块标识</th><th colspan = 2>扩展块开始标识，固定值0x21</th></tr><tr><th>1</th><th colspan = 8 bgcolor="green">应用程序扩展标识</th><th colspan = 2>应用程序扩展标识，固定值0xFF</th></tr><tr><th>2</th><th colspan = 8 bgcolor="green">块大小</th><th colspan = 2>块大小，固定值0x0B</th></tr><tr><th>3</th><th colspan = 8 bgcolor="green">0x4E('N')</th><th colspan = 2 rowspan = 8>NETSCAPE</th></tr><tr><th>4</th><th colspan = 8 bgcolor="green">0x45('E')</th></tr><tr><th>5</th><th colspan = 8 bgcolor="green">0x54('T')</th></tr><tr><th>6</th><th colspan = 8 bgcolor="green">0x53('S')</th></tr><tr><th>7</th><th colspan = 8 bgcolor="green">0x43('C')</th></tr><tr><th>8</th><th colspan = 8 bgcolor="green">0x41('A')</th></tr><tr><th>9</th><th colspan = 8 bgcolor="green">0x50('P')</th></tr><tr><th>10</th><th colspan = 8 bgcolor="green">0x45('E')</th></tr><tr><th>11</th><th colspan = 8 bgcolor="green">0x32('2')</th><th colspan = 2 rowspan = 3>2.0</th></tr><tr><th>12</th><th colspan = 8 bgcolor="green">0x2E('.')</th></tr><tr><th>13</th><th colspan = 8 bgcolor="green">0x30('0')</th></tr><tr><th>14</th><th colspan = 8 bgcolor="green">0x03</th><th>该数据块长度为3字节(不包含该字节)</th><th rowspan = 4>数据块</th></tr><tr><th>15</th><th colspan = 8 bgcolor="green">0x01</th><th>数据块id</th></tr><tr><th>16</th><th colspan = 8 rowspan = 2 bgcolor="green">循环次数</th><th rowspan = 2>循环次数</th></tr><tr><th>17</th></tr><tr><th>18</th><th colspan = 8 bgcolor="green">0x00</th><th colspan = 2>数据块大小为0，作为一组数据块的收尾</th></tr></table>


以上就是GIF文件的结构，下一篇文章将在iOS平台上对GIF文件直接操作进而达到对GIF调速的功能。