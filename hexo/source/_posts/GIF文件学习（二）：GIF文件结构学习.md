---
title: GIF文件学习（二）：GIF文件结构学习
---

本文在对GIF文件结构学习之后，在iOS平台上进行一些应用。

最近在开发的项目中，有一个需求如下：对GIF文件进行调速。

* 最开始由于项目中存在多种素材（视频、图片组、GIF以及Live Photo）转为GIF的需求，使用的是开源的FFmpeg来进行格式的转换来最终生成GIF。此方法需要额外导入FFmpeg库，造成安装包大小的增大，同时FFmpeg更多的是注重于各种格式图像素材的转换，而非对各种素材进行调速等处理（虽然也能很容易达到，但是极其耗时）。
* 之后技术实现换为FFmpeg负责格式的转换，而对GIF文件数据流进行直接操作覆写达到调速功能。（虽然最终整体换为iOS平台的ImageIO框架，但是还是有必要记录一下该思路的）。

<!--more-->

# 实现思路

通过上一篇文章的学习，我们可以知道，对于GIF调速，本质上就是修改GIF中每张图片的持续时间，该持续时间存储在**图形控制扩展**的第4、5字节中，所以我们只需直接修改对应位置的数据即可，同时为了保证速度，我们可以对GIF中其余控制块与数据块进行跳过处理。

# 方法入口

首先我们提供两种修改方式：（1）直接修改持续时间；（2）在原有基础上按倍数修改持续时间

```
typedef enum : NSUInteger {
    GIFChangeSpeedTypeMultiple,
    GIFChangeSpeedTypeValue
} GIFChangeSpeedType;
```

```
/**
 修改源文件调整GIF帧间距
 @param gifFilePath GIF所在路径
 @param changeSpeedType 调整方式
 @param value 若为multiple方式，则是原先的倍数；若为value方式，则是帧间隔，单位s
 
 @return 调整结果
 */
+ (BOOL)changeGIFSpeedWithGIFFilePath:(NSString *)gifFilePath changeSpeedType:(GIFChangeSpeedType)changeSpeedType value:(CGFloat)value;
```

# 辅助方法

在处理数据过程中，我们需要一些辅助方法：

## 获取某字节数据的十六位表达

```
///获取对应data的数据
+ (BOOL)p_getUint8WithData:(NSData *)data result:(uint8_t *)result {
    if (data.length == 0 || result == NULL) {
        return NO;
    }
    CFDataRef dataRef = (__bridge CFDataRef)data;
    const char *bytes = (char *)CFDataGetBytePtr(dataRef);
    *result = *((uint8_t *)bytes);
    return YES;
}
```

# 具体实现

## 判断文件是否存在

我们首先需要判断指定的文件是否存在：

```
//判断文件是否存在
if (gifFilePath == nil || ![[NSFileManager defaultManager] fileExistsAtPath:gifFilePath]) {
    return NO;
}
```

## 判断文件是否可以正常打开

```
NSFileHandle *fileHandle = [NSFileHandle fileHandleForUpdatingAtPath:gifFilePath];

//判断是否正常打开文件
if (fileHandle == nil) {
    return NO;
}
```

## 判断指定文件是否为GIF文件

对于GIF文件的判断，我们可以从GIF文件结构入手，GIF文件开头为**GIF**三个字符：

```
///检查是否为GIF
+ (BOOL)p_checkGIF:(NSFileHandle *)fileHandle {
    if (fileHandle == nil) {
        return NO;
    }
    [fileHandle seekToFileOffset:0];
    uint8_t g;
    uint8_t i;
    uint8_t f;
    if ([self p_getUint8WithData:[fileHandle readDataOfLength:1] result:&g] == YES && g == 0x47 &&
        [self p_getUint8WithData:[fileHandle readDataOfLength:1] result:&i] == YES && i == 0x49 &&
        [self p_getUint8WithData:[fileHandle readDataOfLength:1] result:&f] == YES && f == 0x46) {
        return YES;
    }
    return NO;
}
```

## 判断是否有全局颜色列表

```
/读取是否含有全局颜色列表
uint8_t hasGlobalColorTable = 0x00;
if ([self p_getUint8WithData:[fileHandle readDataOfLength:1] result:&hasGlobalColorTable] == NO) {
    return NO;
}

if (hasGlobalColorTable >= 0x80) {
    //存在全局颜色列表，跳过全局颜色列表
    NSUInteger pixel = (NSUInteger)(((uint8_t)(hasGlobalColorTable << 5)) >> 5);
    //全局颜色列表大小为 3 * 2的(pixel + 1)次方
    [fileHandle seekToFileOffset:fileHandle.offsetInFile + (3 * pow(2, pixel + 1))];
}
```

## 循环处理控制块与数据块

```
while (YES) {
    //标识
    uint8_t identifier;
    if ([self p_getUint8WithData:[fileHandle readDataOfLength:1] result:&identifier] == NO) {  //读取完毕
        [fileHandle synchronizeFile];
        [fileHandle closeFile];
        return YES;
    }
    if (identifier == 0x21) {   //处理可能为标识块的数据
        [self p_handleIdentifierBlockWithFileHandle:fileHandle changeSpeedType:changeSpeedType value:value];
    } else if (identifier == 0x2c) {    //处理图像
        [self p_handleImageDataWithFileHandle:fileHandle];
    }
}
```

### 处理控制块
其中标识为**0x21**为控制块

```
///处理可能为标识块的数据
+ (void)p_handleIdentifierBlockWithFileHandle:(NSFileHandle *)fileHandle changeSpeedType:(GIFChangeSpeedType)changeSpeedType value:(CGFloat)value {
    uint8_t subIdentifier = 0x00;
    if ([self p_getUint8WithData:[fileHandle readDataOfLength:1] result:&subIdentifier]) {
        if (subIdentifier == 0xf9) {    //可能为图形控制拓展
            [self p_handleGraphicControlExtensionWithFileHandle:fileHandle changeSpeedType:changeSpeedType value:value];
        } else if (subIdentifier == 0xfe) { //确定为注释拓展
            [self p_handleCommentExtensionWithFileHandle:fileHandle];
        } else if (subIdentifier == 0x01) { //可能为图形文本拓展
            [self p_handlePlainTextExtensionWithFileHandle:fileHandle];
        } else if (subIdentifier == 0xff) { //可能为应用程序扩展
            [self p_handleApplicationExtensionWithFileHandle:fileHandle];
        }
    }
}
```

#### 处理图形控制扩展

```
///处理可能为图形控制拓展
+ (void)p_handleGraphicControlExtensionWithFileHandle:(NSFileHandle *)fileHandle changeSpeedType:(GIFChangeSpeedType)changeSpeedType value:(CGFloat)value {
    uint8_t subSubIdentifier = 0x00;
    if ([self p_getUint8WithData:[fileHandle readDataOfLength:1] result:&subSubIdentifier]) {
        if (subSubIdentifier == 0x04) { //确认为图形控制拓展
            //跳过一个字节的信息
            [fileHandle seekToFileOffset:fileHandle.offsetInFile + 1];
            
            //查看是否有标识块终结
            [fileHandle seekToFileOffset:fileHandle.offsetInFile + 3];
            
            //Block Terminator - 标识块终结，固定值0
            uint8_t blockTerminator;
            if ([self p_getUint8WithData:[fileHandle readDataOfLength:1] result:&blockTerminator] == YES && blockTerminator ==  0x00) {
                [fileHandle seekToFileOffset:fileHandle.offsetInFile - 4];
                
                uint8_t low = 0x00;
                uint8_t high = 0x00;
                if ([self p_getUint8WithData:[fileHandle readDataOfLength:1] result:&low] &&
                    [self p_getUint8WithData:[fileHandle readDataOfLength:1] result:&high]) {
//                    NSLog(@"%02x %02x", low, high);
                    [self p_changeSpeedWithLow:low high:high fileHandle:fileHandle changeSpeedType:changeSpeedType value:value];
                }
            }
        }
    }
}
```

在图形控制扩展中，我们可以直接修改读取到的持续时间：

```
///调速
+ (void)p_changeSpeedWithLow:(uint8_t)low high:(uint8_t)high fileHandle:(NSFileHandle *)fileHandle changeSpeedType:(GIFChangeSpeedType)changeSpeedType value:(CGFloat)value {
    NSUInteger speed = 0;
    if (changeSpeedType == GIFChangeSpeedTypeMultiple) {
        speed = (NSUInteger)((uint16_t)(high << 8 | low));
        speed *= value;
    } else {
        //文件中存储的时间单位为 1/ 100 s
        speed = value * 100;
    }

    uint16_t newSpeed = (uint16_t)speed;
    uint8_t newLow = newSpeed & 0x00FF;
    uint8_t newHigh = (newSpeed & 0xFF00) >> 8;
    
    NSData *data = nil;
    [fileHandle seekToFileOffset:fileHandle.offsetInFile - 2];
    
    data = [NSData dataWithBytes:(const void *)&newLow length:1];
    [fileHandle writeData:data];
    
    data = [NSData dataWithBytes:(const void *)&newHigh length:1];
    [fileHandle writeData:data];
}
```

#### 处理注释扩展

```
///处理注释拓展
+ (void)p_handleCommentExtensionWithFileHandle:(NSFileHandle *)fileHandle {
    //在判断注释拓展时已经处理了拓展块标识和注释块标签，直接处理注释块和块终结器就好
    [self p_handleDataBlockWithFileHandle:fileHandle];
}
```

#### 处理图形文本扩展

```
///处理可能为图形文本拓展
+ (void)p_handlePlainTextExtensionWithFileHandle:(NSFileHandle *)fileHandle {
    uint8_t subSubIdentifier = 0x00;
    if ([self p_getUint8WithData:[fileHandle readDataOfLength:1] result:&subSubIdentifier]) {
        if (subSubIdentifier == 0x0c) { //确认为图形文本拓展
            //跳过2字节文本框左边界位置，2字节文本框上边界位置，2字节文本框高度，2字节文本框高度，1字节字符单元格宽度，1字节字符单元格高度，1字节文本前景色索引，1字节文本背景色索引
            [fileHandle seekToFileOffset:fileHandle.offsetInFile + 12];
            
            //跳过文本数据块
            [self p_handleDataBlockWithFileHandle:fileHandle];
        }
    }
}
```

#### 处理应用持续扩展

```
///处理可能为应用程序扩展
+ (void)p_handleApplicationExtensionWithFileHandle:(NSFileHandle *)fileHandle {
    uint8_t subSubIdentifier = 0x00;
    if ([self p_getUint8WithData:[fileHandle readDataOfLength:1] result:&subSubIdentifier]) {
        if (subSubIdentifier == 0x0b) { //确定为应用程序扩展
            //跳过8字节的应用程序标识符，3字节的应用程序鉴别码
            [fileHandle seekToFileOffset:fileHandle.offsetInFile + 11];
            
            //跳过应用程序数据数据块
            [self p_handleDataBlockWithFileHandle:fileHandle];
        }
    }
}
```

### 处理数据块

数据块的长度在每个数据块第一个字节指明了，在数据块组结束时，末尾会有一个长度为0x00的数据块为结束标志。

```
///跳过数据块
+ (void)p_handleDataBlockWithFileHandle:(NSFileHandle *)fileHandle {
    //读取块大小
    uint8_t blockSize = 0x00;
    if ([self p_getUint8WithData:[fileHandle readDataOfLength:1] result:&blockSize]) {
        if (blockSize == 0x00) {    //读取完毕(如果某个数据子块的第一个字节数值为0，即该数据子块中没有包含任何有用数据，则该子块称为块终结符，用来标识数据子块到此结束)
            return;
        } else {
            [fileHandle seekToFileOffset:fileHandle.offsetInFile + (NSUInteger)blockSize];
            //继续处理
            [self p_handleDataBlockWithFileHandle:fileHandle];
        }
    }
}
```

以上就是iOS对于GIF文件数据流直接处理达到调速功能，[Demo地址](https://github.com/mademao/ChangeGifSpeed)