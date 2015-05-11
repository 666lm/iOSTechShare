精通Runtime指南
----

# 前言(Runtime本身的特点)

Objective-C Runtime是Runtime库，基本上是用C和汇编，使C具有了面向对象的能力。这也使得Objective-C语言成为了一个动态语言。这动态语言和传统的面向过程语言(C语言)、面向对象语言(C++, OOP)有比较大的区别：函数调用在编译期不能确定内存地址，只有到了运行时才能做出跳转。也决定了Objective-C语言将 决定 尽可能的从编译和链接时推迟到运行时。并且只要有可能，Objective-C总是用动态的方式来解决问题。Objective-C是基于Runtime完成了面向对象和动态的特性，这里Runtime系统扮演的角色类似于Objective-C语言的操作系统。

这里概括为一句话，基于C语言的Objective-C Runtime系统造就了Objective-C这门[面向对象语言](http://zh.wikipedia.org/wiki/%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%A8%8B%E5%BA%8F%E8%AE%BE%E8%AE%A1)，同时也是[动态语言](http://zh.wikipedia.org/wiki/%E5%8A%A8%E6%80%81%E8%AF%AD%E8%A8%80)。[为什么Objective-C是动态语言?](http://www.zhihu.com/question/19970471)

OOP基本理论包括 类、对象、消息传递、继承、封装性、多态、抽象性等。动态语言则是一类在运行时可以改变其结构的语言。因此我们通过这些特点来了解runtime系统的内容。

## Runtime优缺点

1. 执行效率相对慢，但算法复杂度上一样；
2. 安全性差，因为反编译出来的代码近似源码；安全相关最好还是C,C++。

# Runtime系统内容(OOP内容，对象模型和消息转发)

Runtime系统内容包括语言本身、ARC、Block等相关内容。这里语言本身主要包含了：对象模型和类载入、消息和动态方法解析、元编程等。

[libobjec.order](http://www.opensource.apple.com/source/objc4/objc4-551.1/libobjc.order)

## 对象模型和类载入

[Non-pointer isa](http://www.sealiesoftware.com/blog/archive/2013/09/24/objc_explain_Non-pointer_isa.html)

### 对象模型

对象模型中最需要理解的是 实例对象－类－metaclass 之间的关系。可以简单地认为实例方法调用是搜素类的方法列表，而类方法的调用是搜索metaclass的方法列表。三者之间通过isa来联系。众多关于这之间关系的描述，基本上都源自于[Classes and metaclasses](http://www.sealiesoftware.com/blog/archive/2009/04/14/objc_explain_Classes_and_metaclasses.html)。这里需要特别注意地是，metaclass的isa都是root metaclass；root metaclass继承于root class。

相关对象模型的文章就很多了：

1. [Objective-C对象模型及应用](http://blog.devtang.com/blog/2013/10/15/objective-c-object-model/)
2. [刨根问底Objective－C Runtime（2）－ Object & Class & Meta Class](http://chun.tips/blog/2014/11/05/bao-gen-wen-di-objective%5Bnil%5Dc-runtime-(2)%5Bnil%5D-object-and-class-and-meta-class/)
3. 深入浅出Cocoa 之 类与对象

这里也冒出了一个关于self 和 super区别的讨论。self调用的是objc_msgSend，而objc_msgSendSuper(其receiver的指针仍是self，而class是superclass)。你可以在[这里](http://www.cnblogs.com/kimimaro/archive/2011/05/04/2036878.html)获得更多更详细的讨论。另[刨根问底Objective－C Runtime（1）－ Self & Super]<http://chun.tips/blog/2014/11/05/bao-gen-wen-di-objective%5Bnil%5Dc-runtime(1)%5Bnil%5D-self-and-super/>。

### 类初始化和载入

初始化的争论，[The How and Why of Cocoa Initializers](https://www.mikeash.com/pyblog/the-how-and-why-of-cocoa-initializers.html)。

同时也有两个重要的方法来完成类载入和首次被使用，分别是`+(void)load`和`+(void)initialize`。[Objective C类方法load和initialize的区别](http://www.cnblogs.com/ider/archive/2012/09/29/objective_c_load_vs_initialize.html)，[Friday Q&A 2009-05-22: Objective-C Class Loading and Initialization](https://www.mikeash.com/pyblog/friday-qa-2009-05-22-objective-c-class-loading-and-initialization.html)。

1. `+(void)load`

1). 写在应用或者framework中，`main()`运行之前调用。写在loadable bundle中，调用在bundle载入处理中。2). 而此时，App（或者框架或者插件）中的C++ static initiaizers还没有run，如果你依靠这些的话，可能会crash。但是你连接的框架（framework）在这时候已经完全载入了，所以你可以使用framework类。这时候superclass也被保证完全载入了，所以他们也能安全使用。3).仍有一点你需要注意的是，通常在loading的时候，这里没有自动释放池，所以如果你调用了Objective-C的东西，你需要将你的代码封装到一个地方。4). category里面的load 也可以被一起调用。

2. `+(void)initialize`

至少`main()`运行之后调用。其调用符合消息转发的逻辑，如果你在子类未实现，则会调用父类的方法。

	 id objc_msgSend(id self, SEL _cmd, ...)
	    {
	        if(!self->class->initialized)
	            [self->class initialize];
	        ...send the message...
	    }

### 类的memory layout

需要以后再研究底层的看看类编译后，category、protocal、方法列表等的内存分布。

## 消息传递和动态方法解析

### 消息

概念的话，可以参看 <Objective-C 2.0 运行时系统编程指南>

### 消息转发 和 动态方法解析

通常消息转发和动态方法解析是互不相干的。

下面的机制可参看[Friday Q&A 2009-03-27: Objective-C Message Forwarding](https://www.mikeash.com/pyblog/friday-qa-2009-03-27-objective-c-message-forwarding.html)

1. 在进入消息转发机制之前，(`respondsToSelector:`和`instancesRespondToSelector:`（会lookUpImpOrNil）会首先被调用，来判断是否能，不能的话执行2)，查找该类及其父类 的 cahce 和方法分发表，在找不到的情况下执行2.
2. Lazy method resolution。找替代方法。执行`+ (BOOL) resolveInstanceMethod:(SEL)aSEL` 或者`+ (BOOL)resolveClassMethod:(SEL)sel`。返回NO的话，则进行后续的消息转发。否则再次检查是否有方法，有的话重新进行消息发送，否则报错并进入后续的消息转发。(使用场景，知道方法名，但是不知道返回类型。)
3. Fast forwarding path。找替代对象。接下来调用 `– (id)forwardingTargetForSelector:(SEL)aSelector` 方法，以来更换转发对象。如果返回值不是nil或者self，那么将给新的target发送该消息。
4. Normal forwarding path。调用`methodSignatureForSelector`，看有没有methodSignature，没有的话runtime发送`doesNotRecognizeSelector`。有的话，runtime创建`NSInvocation`，然后给对象发了个forwardInvocation，即5。(使用场景，[HigherOrderMessaging](http://cocoadev.com/HigherOrderMessaging)，(HOM) is the use of a message as the argument for another message.)
5. 最后调用`– (void)forwardInvocation:(NSInvocation *)anInvocation` 这个消息转发方法。NSInvocation 其实就是一条消息的封装。（除了）


## 元编程

Objective-C程序有三种途径和运行时系统交互:

+ Objective-C 源代码(当编译Objective-C类和方法时，编译器为实现语言动态特性将自动创建一些数据结构和函数。)
+ Foundation框架中类NSObject的方法
+ 直接调用运行时系统的函数，[Runtime系统参考库](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/ObjCRuntimeRef/index.html)

[Friday Q&A 2010-11-6: Creating Classes at Runtime in Objective-C](https://www.mikeash.com/pyblog/friday-qa-2010-11-6-creating-classes-at-runtime-in-objective-c.html) 里面包含了给类增加各种属性的方法。

## 创建类

[Friday Q&A 2010-11-6: Creating Classes at Runtime in Objective-C](https://www.mikeash.com/pyblog/friday-qa-2010-11-6-creating-classes-at-runtime-in-objective-c.html)

[Friday Q&A 2010-11-19: Creating Classes at Runtime for Fun and Profit](https://www.mikeash.com/pyblog/friday-qa-2010-11-19-creating-classes-at-runtime-for-fun-and-profit.html)

# Runtime源码分析

## objc_msgSend

[Friday Q&A 2012-11-16: Let's Build objc_msgSend](https://www.mikeash.com/pyblog/friday-qa-2012-11-16-lets-build-objc_msgsend.html)

## imp_implementationWithBlock()的实现

[Implementing imp_implementationWithBlock()](http://landonf.bikemonkey.org/code/objc/imp_implementationWithBlock.20110413.html)

## Method Signature Mismatches

[Friday Q&A 2011-08-05: Method Signature Mismatches](https://www.mikeash.com/pyblog/friday-qa-2011-08-05-method-signature-mismatches.html)

[A big weakness in Objective-C's weak typing](http://www.cocoawithlove.com/2011/06/big-weakness-of-objective-c-weak-typing.html)，[翻译](http://select.yeeyan.org/view/213582/209416)

## TypeCoding 和 编译指令

[TypeCoding](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)

[编译指令](http://husbandman.diandian.com/post/2012-08-09/40033093614)

@defs指令返回一个Objective-C类的布局(类的本质是一种C struct，返回的就是struct的布局形式)，它允许你使用与Objective-C类相同的布局创建一个C结构(struct)。

@package - 声明在@package指令下面的实例变量在定义类的框架(Framework)内是可见的，定义类的框架外是私有的。这些程序仅用在64位系统，在32位系统把@package与@public同等对待

## vTable

vTable好像是早期的时候，用来加速消息dispatch的。他是IMPs的一个数组。可以查看[understanding-objective-c-runtime](http://cocoasamurai.blogspot.sg/2010/01/understanding-objective-c-runtime.html)。

# 相关开源和应用

## trampoline class

also you can think hom([HigherOrderMessaging]([Friday Q&A 2009-03-27: Objective-C Message Forwarding](https://www.mikeash.com/pyblog/friday-qa-2009-03-27-objective-c-message-forwarding.html)))。常使用NSProxy来做trampoline class。

[objc与鸭子对象（下）](http://blog.sunnyxx.com/2014/08/26/objc-duck-advanced/)

## 增加实例变量

在新旧版本中，新版本最显著的特性就是实例变量是“健壮”(non-fragile)的:

+ 在早期版本中，如果改变类中实例变量的布局，必须重新编译该类的所有子类。
+ 在现行版本中，如果改变类中实例变量的布局，无需重新编译该类的任何子类。

现行版本也支持property的synthesis属性。这里fragile ivars是指 地址计算是 [对象地址 ＋ ivar偏移字节], non-fragile是指 地址计算是 [对象地址 ＋ 基类大小 + ivar偏移字节]。这样就能保证在运行时能计算出ivar的地址，而不用通过重新编译。

Runtime增加变量是用该方法，是因为class_addIvar方法需要程序员来管理内存。而[Associated Objects](http://nshipster.cn/associated-objects/)不用。

使用中需要注意，objc_removeAssociatedObjects会删除所有，最好objc_setAssociatedObject来置为nil。

同时，并不支持增加Properties。

## Method Replacement

### Posing

[ClassPosing](http://cocoadev.com/ClassPosing)，通过方法poseAs来让某类来替换某类。但是该方法在iPhone和64-bit的mac上不再支持。

Posing (扮演)和Categories(类目)的区别是：对于子类override父类方法的情况，Categories 不能再调用父类的被重写的方法了；而Posing 可以通过“[super 方法];”方式来调用父类被重写的方法。

### Category

这个问题主要是方法override的时机不固定。

### Swizzling

#### Method Swizzling

主要使用runtime的method_exchangeImplementations来完成。可以使用[jrswizzle](https://github.com/rentzsch/jrswizzle)。因为method swizzling还是有挺多坑的。见[Friday Q&A 2010-01-29: Method Replacement for Fun and Profit](https://www.mikeash.com/pyblog/friday-qa-2010-01-29-method-replacement-for-fun-and-profit.html)

[NSHispter: Method Swizzling](http://nshipster.com/method-swizzling/)

#### isa-swizzling 和 KVO的实现

KVO的实现是通过继承xxx生成新的类NSKVONotifying_xxx，isa指向新的类，而class方法返回原来的类，并提供了__NSSetXXXAndNotify函数来重写被观察的属性。

但其也没有支持所有类型，比如long dobule或_Bool都没有。甚至没有为通用指针类型(generic pointer type)提供方法。

1. [Friday Q&A 2009-01-23, KVO实现](https://www.mikeash.com/pyblog/friday-qa-2009-01-23.html)，[翻译](http://limboy.me/ios/2013/08/05/internal-implementation-of-kvo.html)
2. [详解键值观察（KVO）及其实现机理](http://blog.csdn.net/kesalin/article/details/8194240)
3. [Friday Q&A 2012-03-02: Key-Value Observing Done Right: Take 2](https://www.mikeash.com/pyblog/friday-qa-2012-03-02-key-value-observing-done-right-take-2.html)
4. [Key-Value Observing Done Right](https://www.mikeash.com/pyblog/key-value-observing-done-right.html)

### 直接重载

直接在`+ (void)load`将原来的方法置换。

# 更多

[By your _cmd](http://www.sicpers.info/2013/12/by-your-_cmd)
[A Story About Swizzling "the Right Way™" and Touch Forwarding](http://petersteinberger.com/blog/2014/a-story-about-swizzling-the-right-way-and-touch-forwarding)
<http://vombat.tumblr.com/post/83074022251/synchronizing-around-a-class>
