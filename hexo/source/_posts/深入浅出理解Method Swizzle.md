---
title: 深入浅出理解Method Swizzle
---

对于iOS开发者来说，runtime可以说是必须了解的知识，而我们使用runtime可以做到很多事情，其中就包括Method Swizzle。

那么Method Swizzle是什么呢？我们都知道，Objective-C是一门动态语言，它的方法调用使用的是消息机制，即在运行过程中确定方法的具体调用，例如我们有如下代码``[person sayHello];``，我们在实际调用的时候才能确定具体调用的方法代码。

<!--more-->

那么runtime是如何做到这点的呢？我们来进行一下讨论。

# runtime方法调用机制

首先，我们可以使用runtime的API获取到OC类中的所有方法：

```
Method * class_copyMethodList(Class cls, unsigned int * outCount);
```

该API返回一个``Method``类型的指针，即可以理解为返回``Method``数组，其中每一个``Method``都表示该类中的一个方法。

我们看一下``Method``是什么：

```
typedef struct method_t *Method;

struct method_t {
    SEL name;
    const char *types;
    IMP imp;
};

```

Method是method_t结构体的指针，而method_t结构体包含三部分，分别为``SEL``、``types``、``IMP``。

1. ``SEL``表示方法的名称，我们在OC中一般使用``@selector(numberWithObject:)``来获取。同时由该方法可以看出，OC的方法名中并没有包含任何返回值以及参数类型信息，所以对于SEL来说，``- (void)numberWithObject:(id)object``与``- (id)numberWithObject:(NSString *)string``对于的方法名是一致的。
2. ``types``表示该方法的类型，虽然SEL无法确定同名方法中返回值以及参数类型的不同，但是types可以辅助SEL完成这一点。对于OC中各个类型的表示方式，可以参考[官方文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1)。
3. ``IMP``在官方文档的描述为：An opaque type that represents a method selector。即对应Method中SEL所表示代码内存的指针。

了解完Method的构造，我们再回到runtime调用方法的机制上。

我们都知道，runtime在调用方法时，最终都会转换为

```
id objc_msgSend(id self, SEL op, ...);
```

由此我们可以看出，在进行方法调用时，会根据方法的SEL来进行调用，而可以通过对应SEL所在的Method，进而获取到对应的IMP，从而完成整个方法的动态调用。

而在该过程中，我们可以通过修改IMP的指向，进而达到修改方法实现的目的。这就是Method Swizzle。

# Method Swizzle实现

了解了Method Swizzle的概念，我们来实现一下Method Swizzle。

首先我们定义一个用来实验的Model类：

```
@interface Model : NSObject

- (void)sayHello;

- (void)sayHi;

@end

@implementation Model

- (void)sayHello {
    NSLog(@"Say Hello");
}

- (void)sayHi {
    NSLog(@"Say Hi");
}

@end

```

要实现Method Swizzle，我们可以很容易的在runtime的API中找到如下方法：

```
/** 
 * Exchanges the implementations of two methods.
 */
void method_exchangeImplementations(Method m1, Method m2);
```

我们对它进行简单的封装：

```
void exchangeMethod(Class cls, SEL originalSelector, SEL newSelector) {
    Method originalMethod = class_getInstanceMethod(cls, originalSelector);
    Method newMethod = class_getInstanceMethod(cls, newSelector);
    method_exchangeImplementations(originalMethod, newMethod);
}
```

然后使用我们封装的方法：

```
Model *model = [[Model alloc] init];
[model sayHello];
[model sayHi];

exchangeMethod([Model class], @selector(sayHello), @selector(sayHi));

[model sayHello];
[model sayHi];
```

结果如下：

```
Say Hello
Say Hi
Say Hi
Say Hello
```

我们可以发现，此时两个方法的实现已经被调换，该方法完成了我们预期想要做的事情。

那么Method Swizzle是否就如此简单呢？答案并不是。

# 进一步探讨

我们知道，子类可以继承父类中的方法，那么子类中的方法来源就有两个：自己本身以及继承父类。

现在我们定义两个类，来探索一下交换继承父类方法会造成什么影响：

```
@interface SuperModel: NSObject

- (void)sayHello;

@end

@implementation SuperModel

- (void)sayHello {
    NSLog(@"Say Hello");
}

@end

@interface SubModel: SuperModel

- (void)sayHi;

@end

@implementation SubModel

- (void)sayHi {
    NSLog(@"Say Hi");
}

@end

```

然后我们进行如下代码实验：

```
SuperModel *superModel = [[SuperModel alloc] init];
[superModel sayHello];
NSLog(@"--------------");

SubModel *subModel = [[SubModel alloc] init];
[subModel sayHello];
[subModel sayHi];

exchangeMethod([SubModel class], @selector(sayHello), @selector(sayHi));

[subModel sayHello];
[subModel sayHi];

NSLog(@"--------------");

[superModel sayHello];
```

打印结果如下：

```
Say Hello
--------------
Say Hello
Say Hi
Say Hi
Say Hello
--------------
Say Hi
```

我们发现，随着对子类中方法的置换，不仅子类中的方法发生了变化，父类中的方法也随之发生了变化。这是为什么呢？

造成上述现象，是由于在OC中，一个类所拥有的方法并不全部在自己内部保存，那些通过继承父类拥有的方法，是保存在父类的内部。如下图：

![Objective-C类结构图](/blog/MethodSwizzle/Runtime_Class.png)

在此例中，对于SubModel的sayHello方法，该方法是保存在Superclass(class)中的，而sayHi方法是保存在Subclass(class)中。

下面对该原理做一下简单验证，我们封装一下读取类中方法列表的方法，如下：

```
void getInstanceMethod(Class cls) {
    NSLog(@"---method---");
    unsigned int methodCount = 0;
    Method *methods = class_copyMethodList(cls, &methodCount);
    for (NSInteger i = 0; i < methodCount; i++) {
        Method method = methods[i];
        NSLog(@"%@", [NSString stringWithCString:sel_getName(method_getName(method)) encoding:NSUTF8StringEncoding]);
    }
    NSLog(@"---end---");
}
```

并对SuperModel以及SubModel进行读取：

```
getInstanceMethod([SuperModel class]);
getInstanceMethod([SubModel class]);
```

结果如下：

```
---method---
sayHello
---end---
---method---
sayHi
---end---
```

事实上，我们在对SubModel的方法进行交换时，实际上是将SuperModel的sayHello的SEL与SubModel的sayHi的IMP进行绑定，同时将SubMode的sayHi的SEL与SuperModel的sayHello的IMP进行绑定。

这样做虽然达到了我们想要的效果，但是不可避免的破坏了SuperModel的内容，这在通常情况下是不希望出现的。

# 优化

为了防止以上情况的出现，我们可以从runtime的其他API入手。

在选择新的API之前，我们需要做以下几点的考虑：

1. 用来交换的方法可能是该类本身就存在的或者是从父类继承而来的。
2. 用来交换的方法可能在类的继承体系中不存在，该方法可能是一个全新的方法。

我们可以找到以下两条API：

```
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char * types);
```

该API会判断当前类是否含有SEL，若含有，则返回NO；若不含有，则使用SEL与IMP添加该方法，并返回YES，同时，该方法可以为子类添加父类中的SEL，并和父类中的SEL使用同一个IMP。该API可以帮助我们解决上述第一个问题。

```
IMP class_replaceMethod(Class cls, SEL name, IMP imp, const char * types);
```

该API会有两种处理逻辑：

1. 若SEL在该类本身中不存在，那么方法会使用SEL，IMP来添加该方法，并返回一个空的IMP。
2. 若SEL在该类本身中存在，那么会使用IMP来替换掉SEL原先对应的IMP，并将SEL原先对应的IMP返回。

将此逻辑对应为代码，结果如下：

```
void swizzleMethod(Class cls, SEL originalSelector, SEL newSelector) {
    Method originalMethod = class_getInstanceMethod(cls, originalSelector);
    Method newMethod = class_getInstanceMethod(cls, newSelector);
    
    if (class_addMethod(cls,
                        originalSelector,
                        method_getImplementation(newMethod),
                        method_getTypeEncoding(newMethod))) {
        //original在本类中不存在（包括本身不存在或方法为从父类继承），此时为originalSelector绑定newSelector的IMP
        
        //此时需要为本类添加newSelector，无论本类是否已经存在newSelector
        class_replaceMethod(cls,
                            newSelector,
                            method_getImplementation(originalMethod),
                            method_getTypeEncoding(originalMethod));
    } else {
        //originalSelector在本类中已经存在，直接对originalSelector的IMP进行交换，此时为保证不干扰newSelector，故而不使用method_exchangeImplementations
        IMP imp = class_replaceMethod(cls,
                                      originalSelector,
                                      method_getImplementation(newMethod),
                                      method_getTypeEncoding(newMethod));
        //为本类添加newSelector，无论本类是否已经存在newSelector
        class_replaceMethod(cls,
                            newSelector,
                            imp,
                            method_getTypeEncoding(originalMethod));
    }
}
```

我们对该代码进行验证：

```
SuperModel *superModel = [[SuperModel alloc] init];
[superModel sayHello];
NSLog(@"--------------");

SubModel *subModel = [[SubModel alloc] init];
[subModel sayHello];
[subModel sayHi];

swizzleMethod([SubModel class], @selector(sayHello), @selector(sayHi));

[subModel sayHello];
[subModel sayHi];

NSLog(@"--------------");

[superModel sayHello];

getInstanceMethod([SuperModel class]);
getInstanceMethod([SubModel class]);

```

结果如下：

```
Say Hello
--------------
Say Hello
Say Hi
Say Hi
Say Hello
--------------
Say Hello
---method---
sayHello
---end---
---method---
sayHello
sayHi
---end---
```

通过上述结果，我们发现，SuperModel中的sayHello依然保持了它的实现，而在SubModel中，则有了两个方法：sayHello和sayHi，同时这两个方法的实现进行了互换。

至此，我们完成了我们理想中的Method Swizzle的实现。