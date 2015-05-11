精通ARC指南
---

首先你可能还要考虑[是否应该使用ARC?](http://blog.devtang.com/blog/2013/03/27/should-we-use-arc/)。如果要使用的话，@onevcat[手把手教你ARC——iOS/Mac开发ARC入门和使用](http://onevcat.com/2012/06/arc-hand-by-hand/)。基本上这样就能满足日常开发所需了。

只需要牢记内存管理的思考方式:

1. 自己生成的对象，自己持有；alloc/new/copy/mutablecopy 
2. 非自己生成的对象，自己也能持有；
3. 不再需要自己持有的对象时释放；
4. 非自己持有的对象无法释放。

MRC是基于引用计数进行内存管理的，这点在ARC(Automatic reference counting)也并没有变化。正如Automatic所示，ARC技术是自动帮助我们处理『引用计数』的相关部分。

基于引用计数的内存管理有两个重要的概念:Reference count和Ownership。

# 内存管理

/*
首先，回顾下 错误的内存管理，往往包括两类：

1. 释放或者覆盖正在使用中的数据。
2. 不用的数据却不释放，从而导致内存泄露。
*/

在本章节，首先描述了iOS内存管理几个概念的来龙去脉，然后给出基于这几个概念所采取的内存管理策略，最后讨论了相关actions的实现。

## 来龙去脉

对于内存管理常用的方法，如GC、引用计数，为什么会有这些机制，这里就不讨论了，你可以看看《编译原理》。

这里着重对内存管理的两个概念(Reference count和Ownership)进行讨论，不描述基础。

iOS内存管理，是基于引用计数，通过 Reference count 和 Ownership 的结合来完成的。

从引用计数出发

***Reference count***

Reference count，引用计数。

从Reference count 这里，衍生出相关的action:

* retain
* release
* dealloc

然后，在实际的实现中，如果只有上述引用计数的几个方法的话，会出现一种情况：
eg.1，一个方法返回了一个需要在方法外进行内存管理的对象，但是你在方法调用后，忘了对该对象进行释放或者 压根就忘了该方法返回的对象需要释放。这里就会出现内存泄露。

因此，为了辨别哪些是需要调用者释放，哪些不需要，在内存管理中采用了Ownership策略。(以上纯属我猜测)// FIXME 

***Ownership***

Ownership, 这里翻译成 持有，也有些翻译成 所有。

从Ownership这里，衍生出一个概念Method families，按以下的方法开头(驼峰的书写方法)，包括:// FIXME : 具体的规范

* alloc
* new
* copy
* mutableCopy

但是，我们发现仅仅从命名规范，还是会存在上述出现eg.1的问题。其实我们更希望方法本身的scope内就能对这样的对象进行内存管理，而不是让调用方法的人在方法scope之外来进行内存管理。这样的话，既能避免 人主观上会出的错误，也能让方法命名更为规范(如，不需要调用的方法进行内存管理的，可以不按上述的命名规则)。(以上纯属我猜测)// FIXME 

因此，在内存管理中增加了Autorelease Pool来彻底解决这类问题。

***Autorelease Pool***

Autorelease Pool，自动释放池。同时也衍生出一个action:

* autorelease

autorelease可以让你立刻放弃 持有，交给自动释放池来释放。

---

你也可以简单地认为内存管理由 alloc/retain/release/dealloc/autorelease 五个方法组成，其中alloc来自持有，retain/release/dealloc来自 引用计数，autorelease来自于实际需要。

注意:
在实践过程中，不要总是考虑内存管理的实现细节，也就是不用引用计数来理解内存管理，因为这样会让你理解起内存管理来非常困难。你应该用 持有(Ownership) 和 对象图(Object Graph) 来理解。
// FIXME : 引用计数 的实现，好像是用表。

## 基于Retain Counts的Ownership策略

上述衍生出来的actions，实际上也对应着NSObject中的方法。

基于上述的概念和actions，已经能完成iOS的内存管理，这里采用了基于Retain Counts的Ownership策略。每个对象都有一个retain count。

Action			 | Operation		 | Retain Count
------------ 	 | ------------ 	 | ------------
alloc/new/copy/mutableCopy等 | 生成并持有对象 | +1
retain			 | 持有对象		    | +1
release		 | 释放对象			 | -1
autorelease 	 | 对象加入到自动释放池 | 未来某个时候-1
retain count = 0 | 废弃对象 | dealloc(这个是例外，是当retain count = 0的时候，自动调用dealloc方法)

基本上Ownership策略的规则，如上所述。实现方法有时候不同版本有些不同。

## Actions的实现

接下来，通过了解NSObject的alloc/retain/release/delloc实现，来理解内存管理。

其中相关的开源实现有:

1. Foundation框架所使用的Core Foundation框架源代码
2. 调用NSObject.mm类进行内存管理
3. Cocoa的互换框架GNUStep(以来观察NSObject的实现)

未开源的有:

1. Foundation框架，NSObject的源码

相关网站[Apple Open Source](http://opensource.apple.com/), [GNUStep](http://gnustep.org/)

这里就不描述代码了，可以自行查看。

另外你也可以查看 mikeash的相关教育性实现，包括retain count, NSAutoreleasePool：

1. [Friday Q&A 2011-09-16: Let's Build Reference Counting](https://www.mikeash.com/pyblog/friday-qa-2011-09-16-lets-build-reference-counting.html)

2. [Friday Q&A 2011-09-02: Let's Build NSAutoreleasePool](https://www.mikeash.com/pyblog/friday-qa-2011-09-02-lets-build-nsautoreleasepool.html)

### Reference Counting(Retain Count)/Alloc/Retain/Release/Dealloc

因为历史原因，NSObject的引用计数实现并没有存储在对象本身。而是存储一张外部表中。并有得到、设置引用计数值的函数，以及增加和减少引用计数的函数。retain是调用了增加引用计数的函数，而release则是减少函数并在count为0时调用dealloc。

其中增加和减少引用计数，加了锁(OSSpinLock)。

另外GNUStep里面将引用计数存储在对象头部。

引用计数表的好处:
1. 对象用内存块的分配无需考虑内存块头部。
2. 引用计数表各记录表中有内存块地址，可从各个记录追溯到各对象的内存块。

内存块头部管理引用计数:
1. 少量代码即可完成。
2. 能够统一管理引用计数用内存块与对象内存块。

引用计数表在调试中，第二条特性非常重要，检查内存分配也很方便。

通过lldb调试和结合CFRunTime，可以看到整个Apple的实现。

### NSAutoreleasePool

当发生autorelease的时候，对象被加入到当前线程的([NSTread currentThread])的某个共享对象里(如threadDictionary)(该对象存储了当前的自动释放池栈)的自动释放池栈栈顶的自动释放池中。

自动释放池dealloc的时候，会释放自该自动释放池及栈向上的所有的自动释放池及其所有对象。

其中使用了CF数组而不是NS数组，是因为CF可以兼容ARC。如果在ARC下使用NS的话，其ownership会有比较乱的问题。

当然，现在的版本相对于mikeash的教育版本有所不同，是基于runtime
的thin wrapper，你可以查看[AutoreleasePoolPage](http://opensource.apple.com/source/objc4/objc4-493.9/runtime/objc-arr.mm)。

GNUStep中使用了IMP Caching。加速了`addObject`的调用速度。

/*
mikeash

Conceptually, as far as "autorelease pools are arrays of stuff that get released later", everything is the same. But the details have changed a great deal. In particular, there's not really an NSAutoreleasePool class anymore. The class still exists, but is now a thin wrapper around deeper runtime functionality. That functionality is, I think, directly accessed by the -autorelease method. In short, there's just enough left that you can still use NSAutoreleasePool objects in your code (when not using ARC), but you can't subclass or override things anymore and expect it to work. 

I'd suggest ditching the custom class, at least on iOS 5. The new autorelease pool implementation is extremely fast, probably very difficult to improve upon, and I'd wager faster than what you have. 

In particular, it uses a single array for the entire thread. Individual pools are just pointers into that array, and when popping a pool, that pointer is just used as a fence to know when to stop. Page-granular allocation and clever recycling means that no copying is needed when expanding the array (which doesn't need to be contiguous). For details, check out AutoreleasePoolPage and related code here: http://opensource.apple.com/source/objc4/objc4-493.9/runtime/objc-arr.mm
*/


## 自动释放池 实战

相对于显式的release的操作而言，自动释放池所产生的行为是隐式的，我这里理解为隐式的release(程序员无法完全控制release时机)。这也是内存管理中最好玩的一个玩意儿。我们需要挖掘出其内在的机制，以写出更为明白的代码。

我们知道main.m文件写有个全局的自动释放池，/*另外Cocoa程序在每个线程都维护一个自己的自动释放池的栈*/。除了自己加入的自动释放池外，AppKit和UIKit框架自动在每个消息循环的开始创建一个池(比如鼠标按下事件、触摸事件)，并在结尾处销毁该池。这里我理解为Apple是因为，这两个Kit很容易让UI交互带来了大量的autorelease变量，因此在消息循环创建一个池能快速清除这些变量。

/*
 一个使用run loop的线程可能在每次运行完一次循环的时候，创建并释放该自动释放池。
*/

因此，你往往不怎么需要主动书写自动释放池，除了下述的几个情况你可能需要:

1. 不是基于UI framework的程序，比如命令行；
2. 产生大量临时对象的循环；
3. spawn一个secondary线程。在线程开始的时候，需要创建自动释放池。

注意：

1. NSAutoreleasePool不能调用autorelease，因为已经被重载过，会出现异常；
2. Autorelease池必须以inline的方式使用，你永远不需要把一个这样的池作为对象成员的变量来处理。(在cocoa中，成员变量生成的时候，会被自动加入到自动释放池栈，那么你就无法控制释放时机)
3. 基于Foundation的程序或者detach 一个thread的话，你需要自己创建自动释放池。但是如果detached thread 没有cocoa call的话，就没必要创建自动释放池。在Cocoa应用中，每个线程维持着自动释放池栈。// FIXME : Thread in Cocoa，关于自动释放池

在Cocoa框架中，相当于程序主循环的NSRunLoop或者在其他程序可运行的地方，对NSAutoreleasePool对象进行生成、持有和废弃处理。因此，应用程序开发者不一定非得使用NSAutoreleasePool对象进行开发工作。

/*
我理解，在Cocoa程序中，UIKit主动给每个线程加上自动释放池，但是Foundation没有。同时自动释放池是Cocoa里面的东西，也就是NSObject的子类，所有其他非cocoa是没有的，非cocoa不需要这样管理内存。所以你在Foundation的程序或者detach一个thread的时候，需要自己创建。而没有cocoa call的时候，也就不需要创建自动释放池，因为你不需要用自动释放池进行管理。而为什么在secondary线程，需要创建自动释放池。线程本身不会自己加上自动释放池的，是cocoa框架主动加上的。
*/

# ARC

在使用ARC的时候，常常简单到我们不用考虑内存管理，除了循环引用问题。主要是通过ownership qualification、runtime和clang的属性修饰词管理retainable指针，并做了一些(针对对象生命周期)优化和特殊处理。

## Retainable object pointer

ARC仍是基于原来的内存管理。他的管理范围限制在Retainable object pointers(Retainable 对象指针，也简称为retainable指针)，其是retainable对象指针类型(也简称为retainable类型)的值。也可以认为是能发送`retain/release/auorelease`的对象。retainable类型有三种：

1. block指针 (^开头的函数类型)
2. Objective-C对象指针(id, Class , NSFoo*, 等)
3. 用`__attributes((NSObject))`标志的typedefs

像其他指针类型，比如`int*`和`CFStringRef`，都不在ARC语义和限制的范围内。

type system必须能知道哪些是retainable指针，从而进行管理。

### 当retainable指针是操作元或者参数

一般地，当在一个表达式中简单使用一个retainable指针(比如当操作元或者参数)时，ARC不会去做retain或者release。这包括:

* 从一个不是weak拥有权的对象中load一个retainable对象指针,
* retainable指针作为一个参数传递给一个函数或者方法
* 接收到一个retainable指针作为函数或者方法调用的结果

### 当retainable指针是返回值

分为两类，retained return value 和 Unretained return values

1. retained return value

像`init`这类的method family。

2. Unretained return values

些方法或者函数，它返回了一个retainable对象类型，但是没有返回一个retained值，这种情况必须要保证对象在跨出返回边界后，仍是可用的。

当从这样的方法或者函数返回，ARC在返回statement评估的某个点上retain了该值，然后离开整个本地scope，在保证了值在跨出调用边界后仍是活着的之后，balance out该retain.在最糟糕的情况下，这可能调用了`autorelease`，但是调用者不能假定值实际上在自动释放池中。

虽然ARC可能选择做某些事情来缩短返回值的生命周期，但是在调用者这边不做任何强制工作，也使得存在了优化的可能。

## 语义

## 使用

主要来自于『Objective-C 高级编程』和Clang的arc文档。

ARC满足内存管理的思考方式:

1. 自己生成的对象，自己持有；alloc/new/copy/mutablecopy 
2. 非自己生成的对象，自己也能持有；默认的__strong
3. 不再需要自己持有的对象时释放；比如方法名不是alloc这类时，自动将返回值加到自动释放池
4. 非自己持有的对象无法释放。

### 隐藏的点

1. __weak 变量，为了防止访问该变量时被废弃，做了一个__autorelease的局部变量，将其注册到自动释放池。
2. id的指针或者对象指针 默认是__autorelease。如NSError的使用。同时赋值给对象指针时，所有权修饰符必须一致。这也是因为这样的指针不在arc的管理范围。
3. 当变量传递的时候，往往会根据方法的需要，增加些中间变量来适应方法的所有权。
4. 最重要的一点，除了alloc/new/copy/mutablecopy是自己生成并持有，其他的情况是 取得非自己生成并持有 的对象。变量需要加上__autoreleasing，才能作为对象的取得参数。
5. 正确使用toll-free bridege。
6. Block的最佳实践是copy，所以arc默认是copy。对于id类型的block，是不会自动的copy的，除了是`dispatch_blockt_t`。
7. 本质上，自动释放池是和当前线程、scope绑定在一起的，可以让临时对象(实例变量)是autoreleased 对象，而不能让ARC提供任何形式的安全保证。因此最好不要用`__autoreleasing`修饰非自动变量，block也最好不要捕获`__autoreleasing`修饰的变量，当然也有例外。
8. Class，默认__unsafe_unretained，因为有些Class没有实现retain。

### 规则

在ARC有效的情况下编译源代码，必须遵守一定的规则:

1. 不能使用retain/release/retainCount/autorelease
2. 不能使用NSAllocateObject/NSDeallocateObject
3. 须遵守内存管理的方法命名规则
4. 不要显式调用dealloc
5. 使用@autorelease块替代NSAutoreleasePool
6. 不能使用区域(NSZone)
7. 对象型变量不能作为C语言结构体的成员(一定要使用的话，需要转换)
8. 显式转换『id』和『void *』

### owner inference

#### objects

1. 如果一个对象声明为retainable对象拥有类型，但是没有显示的拥有权修饰词，那么隐式调整为`__strong`修饰。
2. 作为一个特例，如果对象的基础类型是`Class`(可能是protocol-qualified)，那么类型是`__unsafe_unretained`修饰。[dont retain that class](http://arigrant.com/blog/2014/3/14/dont-retain-that-class)

#### Indirect parameters

如果一个函数或者方法的参数有类型`T*`，这里的`T*`是一个没有被`owner-unqualified`retainable的对象指针类型，那么:

* 如果 `T`是`const-qualified`或者Class, 那么隐式修饰为`__unsafe_unretained`；
* 其他情况，隐式修饰为 `__autoreleasing`。

更多的owner inference，可以查看clang的arc文档。

### 优化


### 总结

ARC还隐藏了很多的细节和内存管理的规则。要使用好ARC，是比较复杂的。

如，在属性声明的地方所写的叫modifiers(如strong, retain等)，这等同于指针被ownership qualifier(如__strong)修饰，并保存在metadata里面。modifiers跟实例变量的ownership qualifier的关系也是一个点。

在clang的arc文档中，使得arc能更加自由，如`__attribute((ns_consumed))`、`__attribute((ns_returns_retained))`、`__attribue__((NSObject))`，禁用weak等等。

## 实现

[Friday Q&A 2011-09-30: Automatic Reference Counting](https://www.mikeash.com/pyblog/friday-qa-2011-09-30-automatic-reference-counting.html)

### __strong、__unsafe_unretained和__autorelease

这三个ownership qualifier基本是经过arc compiler的编译，通过runtime来完成整个管理工作。

[When an Autorelease Isn't](https://www.mikeash.com/pyblog/friday-qa-2014-05-09-when-an-autorelease-isnt.html)

### __weak

[Zeroing Weak References in Objective-C](https://www.mikeash.com/pyblog/friday-qa-2010-07-16-zeroing-weak-references-in-objective-c.html)

[Introducing MAZeroingWeakRef](https://www.mikeash.com/pyblog/introducing-mazeroingweakref.html)

[Zeroing Weak References to CoreFoundation Objects](https://www.mikeash.com/pyblog/friday-qa-2010-07-30-zeroing-weak-references-to-corefoundation-objects.html)

### Toll-free Bridge

[Toll Free Bridging Internals](https://www.mikeash.com/pyblog/friday-qa-2010-01-22-toll-free-bridging-internals.html)

[一个简介](http://ridiculousfish.com/blog/posts/bridge.html)

CFInternal.h

每个能桥接的类实际上都是类簇。比如NSString是一个抽象接口，CFString的isa指针是NSCFString，其是NSString的子类。所以从CF到Objc的话是最简单的，要么通过CF本身的函数，要么是实现了对应的方法。另外从Objc到CF相对复杂，首先用宏检查是不是CF对象，不是的话调用对应的objc方法，是的话调用CF方法。

CF_OBJC_FUNCDISPATCH0

if (__builtin_expect(CF_IS_OBJC(typeID, obj), 0))

### Block

还需要再研究`__weak`，自动释放池，retainable指针、对象指针类型以及生命周期，runtime。

# 更多

<http://conradstoll.com/blog/2013/1/19/blocks-operations-and-retain-cycles.html>