---
title: 当iOS遇到malloc/free
---

最近在用8叉树为图片生成配色板，又回到了C语言的天下。

美滋滋的写完代码、运行代码，嗯，配色板出来了，而且符合预期。

But，这个程序是运行在iOS平台上的，顺带看了一下跑代码时候的内存，哎，爆炸

<!--more-->

![内存](/blog/iOS_malloc/memory.png)

接下来就是反思解决问题了：

1. 代码中无任何多余的Objective-C代码，所以由于Objective-C内存管理引起的问题不存在。
2. 代码中与8叉树生成调色板相关代码为C语言代码，在其中有多次的malloc申请内存。
3. 检查malloc申请的内存，确认都使用free进行了释放，不存在未释放情况。

既然如此，那么为何还会有如此高的内存？

以前在使用malloc与free时，是在Windows系统下，在free之后内存会释放，由于此时是在iOS系统之下，是否与这有关系呢？

查阅资料后发现，确实与系统有关：

* Windows 平台调用free，内存会马上降回来。
* Linux 平台调用free，内存不会释放回OS，而是释放回系统的内存缓冲池，进程退出时才释放回OS。
* iOS 平台调用 free 后，也只是释放到系统的内存缓冲池里，进程退出才释放回OS。

原因找到了，仅仅是简单的malloc/free并不能及时的释放内存，那么该如何解决呢。

记得以前看过，iOS在ARC之后管理内存其实是针对于**malloc_default_zone**进行管理的，那么是否有可以自定义**malloc_zone**来进行操作呢？

很幸运的是，我在系统的**malloc/malloc.h**中发现了关于**malloc_zone_t**的定义，看来可以自定义zone来进行内存区域管理了。

在头文件中，发现了以下有关函数：

```
/* Creates a new zone with default behavior and registers it */
extern malloc_zone_t *malloc_create_zone(vm_size_t start_size, unsigned flags);
```

```
/* Destroys zone and everything it allocated */
extern void malloc_destroy_zone(malloc_zone_t *zone);
```

```
/* Allocates a new pointer of size size; zone must be non-NULL */
extern void *malloc_zone_malloc(malloc_zone_t *zone, size_t size);
```

```
/* Frees pointer in zone; zone must be non-NULL */
extern void malloc_zone_free(malloc_zone_t *zone, void *ptr);
```

由此看来，我们可以在执行算法之前，手动申请一个新的**malloc_zone_t**，然后使用**malloc_zone_malloc/malloc_zone_free**方法进行正常的内存申请/释放，在最后可以使用**malloc_destory_zone**来对**malloc_zone_t**进行销毁。

然后就是对代码进行改动，改完之后的效果如下：

![内存](/blog/iOS_malloc/after_memory.png)

并且，在每次调用**malloc_zone_free**时，内存都得到了及时的释放，而并不是在销毁**malloc_zone_t**时才会释放相关内存。