---
title: GIF文件学习（一）：GIF文件结构学习
---

本文对GIF文件的存储方式进行分析与讲解，本文参考文章地址为：[gif 格式图片详细解析](https://blog.csdn.net/wzy198852/article/details/17266507)

#一.概述

GIF图像是有CompuServe公司开发的图形文件格式，GIF图像是基于颜色列表的（即GIF文件中存储的数据为该点的颜色在颜色列表中的索引值），同时GIF最多支持8位（即256色）的颜色展示。

#二.GIF文件结构

##基础知识

GIF文件内部是按块来进行划分的，其中包括**控制块（Control Block）**和**数据块（Data Sub-blcoks）**两种。

* 控制块是用来控制数据块行为的，不同的控制块一般包含不同的控制参数。

* 数据块是用来记载数据的，数据块是一组8-bit组成的数据流，其中每个数据块的大小最大为**256字节**，其中第一个字节用于指出该数据块所含字节数，即一个数据块记录数据所用字节最多为**255字节**。以下是数据块的结构：
<table>
	<tr>
		<th>BYTE</th>
		<th>7</th>
		<th>6</th>
		<th>5</th>
		<th>4</th>
		<th>3</th>
		<th>2</th>
		<th>1</th>
		<th>0</th>
		<th>BIT</th>
	</tr>
	<tr>
		<th>0</th>
		<th colspan = 8 bgcolor="green">块大小</th>
		<th>Blcok Size - 块大小，不包含这个字节</th>
	</tr>
	<tr>
		<th>1</th>
		<th colspan = 8 bgcolor="green"></th>
		<th rowspan = 5>Data Values - 块数据，8-bit的字符串</th>
	</tr>
	<tr>
		<th>2</th>
		<th colspan = 8 bgcolor="green"></th>
	</tr>
	<tr>
		<th>...</th>
		<th colspan = 8 bgcolor="green"></th>
	</tr>
	<tr>
		<th>254</th>
		<th colspan = 8 bgcolor="green"></th>
	</tr>
	<tr>
		<th>255</th>
		<th colspan = 8 bgcolor="green"></th>
	</tr>
</table>


##GIF结构

GIF文件的结构为：**文件头（File Header）**、**GIF数据流（GIF Data Stream）**和**文件终结器（Trailer）**三部分。

接下来详细讲解每部分的构成。

#三.GIF各部分组成

##1.文件头

文件头包含**文件署名（Signature）**和**版本号（Version）**两部分。

###(1)文件署名

文件署名表示该文件是GIF文件，这部分由"GIF"三个字符组成：

<table>
	<tr>
		<th>BYTE</th>
		<th>7</th>
		<th>6</th>
		<th>5</th>
		<th>4</th>
		<th>3</th>
		<th>2</th>
		<th>1</th>
		<th>0</th>
		<th>BIT</th>
	</tr>
	<tr>
		<th>0</th>
		<th colspan = 8 bgcolor="green">'G'(47)</th>
		<th rowspan = 3>文件署名</th>
	</tr>
	<tr>
		<th>1</th>
		<th colspan = 8 bgcolor="green">'I'(49)</th>
	</tr>
	<tr>
		<th>2</th>
		<th colspan = 8 bgcolor="green">"F"(46)</th>
	</tr>
</table>

###(2)版本号

版本号表示GIF版本号，有"87a"和"89a"两种：

<table>
	<tr>
		<th>BYTE</th>
		<th>7</th>
		<th>6</th>
		<th>5</th>
		<th>4</th>
		<th>3</th>
		<th>2</th>
		<th>1</th>
		<th>0</th>
		<th>BIT</th>
	</tr>
	<tr>
		<th>0</th>
		<th colspan = 8 bgcolor="green">'8'(38)</th>
		<th rowspan = 3>版本号</th>
	</tr>
	<tr>
		<th>1</th>
		<th colspan = 8 bgcolor="green">'7'(39)或'9'(40)</th>
	</tr>
	<tr>
		<th>2</th>
		<th colspan = 8 bgcolor="green">"a"(61)</th>
	</tr>
</table>


##2.GIF数据流

GIF数据流包含**逻辑屏幕标识符（Logical Screen Descriptor）**、**全局颜色列表（Global Color Table）**和**图像块（Image Block）**三个基本部分以及一些扩展部分组成，扩展部分包括：**图形控制扩展（Graphic Control Extension）**、**注释扩展（Comment Extension）**、**图形文本扩展（Plain Text Extension）**以及**应用程序扩展（Application Extension）**四部分。

###(1)逻辑屏幕标识符

该部分由**7个字节**组成，结构为：

<table>
	<tr>
		<th>BYTE</th>
		<th>7</th>
		<th>6</th>
		<th>5</th>
		<th>4</th>
		<th>3</th>
		<th>2</th>
		<th>1</th>
		<th>0</th>
		<th>BIT</th>
	</tr>
	<tr>
		<th>0</th>
		<th colspan = 8 rowspan = 2 bgcolor="green">逻辑屏幕宽度</th>
		<th rowspan = 2>像素数，定义GIF图像宽度</th>
	</tr>
	<tr>
		<th>1</th>
	</tr>
	<tr>
		<th>2</th>
		<th colspan = 8 rowspan = 2 bgcolor="green">逻辑屏幕高度</th>
		<th rowspan = 2>像素数，定义GIF图像高度</th>
	</tr>
	<tr>
		<th>3</th>
	</tr>
	<tr>
		<th>4</th>
		<th bgcolor="green">m</th>
		<th colspan = 3 bgcolor="green">cr</th>
		<th bgcolor="green">s</th>
		<th colspan = 3 bgcolor="green">pixel</th>
		<th>具体描述见下</th>
	</tr>
	<tr>
		<th>5</th>
		<th colspan = 8 bgcolor="green">背景色</th>
		<th>背景颜色（在全局颜色列表中的索引，如果没有全局颜色列表，则该值没有意义）</th>
	</tr>
	<tr>
		<th>6</th>
		<th colspan = 8 bgcolor="green">像素宽高比</th>
		<th>像素宽高比</th>
	</tr>
</table>

m：全局颜色列表标志，当置位时表示有全局颜色列表，pixel值有意义
cr：颜色深度，cr+1确定图像的颜色深度
s：分类标志，如果置位表示全局颜色列表分类排序
pixel：全局颜色列表大小，pixel+1确定颜色列表的索引数（2的pixel+1次方）
