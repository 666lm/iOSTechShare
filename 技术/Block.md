# Block起源

主要是因为为了GCD，才增加了Blocks。

在Mac OS X10.6 和iOS 4开发，Apple为Blocks(Closures)引入了语法和runtime。其是对C扩充，并被扩展到Objective-C, C++, Objective-C++。Plausible Labs对Block实现了后向支持(PLBlocks)。

Blocks是 各种变量被绑定到automatic(stack)或者managed(heap)内存 的匿名函数。当然匿名函数并不捕获变量。还加了__block存储类型修饰符来修饰变量，以来获取能mute的变量。

Blocks最weird（也是最不同）的设计是，Blocks在函数（方法）中，字面量定义所产生的是一个stack object。这也间接造成了Blocks的使用和理解障碍。

## Blocks 是 Object

其isa指针是class_t的结构体实例。

## Blocks 是 Stack Object

### 什么是 Stack Object 和 Heap Object

C标准中对象有三种storage durations: 对于stack变量的auto(register)，静态globals和local的static，以及allocated。

比如说，定义了一个局部变量，`NSObject *obj = [[NSObject alloc]init]`。obj变量本身是在stack上，而对象他本身是指向heap的。alloc是分配了heap内存块。

stack object 是这样分配的：

	struct {
	      Class isa;
	  } fakeNSObject;
	  fakeNSObject.isa = [NSObject class];
	  
	  NSObject *obj = (NSObject *)&fakeNSObject;
	  NSLog(@"%@", [obj description]);

Stack Object的特点：
1. 速度快。
2. 简单，其owner可以认为是该scope(方法)，超出该scope就会被释放。

#### Stack Object 和 Heap Object的转换

在Objective-C(和其他许多语言中)，不可能在对象创建后再移动对象。原因是对象相关的许多指针都不能被跟踪了。这些都需要在移动后被更新，但是没什么方法去完成他。（当然一般也是可能的，许多语言也把这个当回事，经常是作为垃圾回收机制的一部分。这要比在Objective-C的runtime更智能，类型系统更严格。）

#### Stack Object in Objective-C

在引用计数管理中，每个对象都可以有多个owner，并且系统只有在没人拥有该对象的时候，才会销毁对象。Stack Object本质上只有一个owner（创建他的函数）。如果Objective-C有stack object，那么retain一把会怎么样。因为没有方法能组织函数销毁对象，所以`retain`没用。代码持有对象也会失败，其结果是悬挂引用，并会crash。

Stack Object 并不灵活。在Objective-C中常常见到 实现一个初始化器，销毁原来的值并返回一个新值。那么Stack Object怎么做呢？你真不能。大多数Objective-C的runtime灵活性依靠heap object。

### Blocks 是 Stack Object 的疑难点

在Objective-C中，只有Blocks是Stack Object，其余的都是Heap Object。其也有上述stack object的特点。其字面量表达式产生的是对Blocks的一个引用。Block的可执行代码实际上并不live在stack上，但是所有data是(包括__block变量)。
/* Blocks的layout是被语言本身来修复的，除非销毁二进制兼容，不然不会改变(can't be changed without destroying binary compatibility)。block对象可以在编译的时候被计算出来，并且整个对象代码是在编译期生成的，所以在初始化器不存在tricky的东西。*/

#### stack object的生命周期所导致的问题

blocks也有对象生命周期的问题，但并不严重。原因很简单，block这样的对象在之前根本就不存在。如果需要持有该引用的话，任何处理block的代码都知道需要copy blocks到heap(如果block不在heap的话)，然后返回一个指针指向他，而不是retain他。

#### 为什么Block需要copy，而不是retain

1. Block定义是个Stack Object，在超出scope后，会强制被销毁。哪怕是retain也没用，retain想持有对象会失败，并造成悬挂引用和crash。除非你copy了该block。
2. 而copy是将Block复制到heap上，从而持有Block。

但是Blocks的stack本质也有pitfalls，即其生命周期是enclosing scope。超出就会失效，并该指针仍有可能存在。(可以看[stack and heap objects in objective-c](https://www.mikeash.com/pyblog/friday-qa-2010-01-15-stack-and-heap-objects-in-objective-c.html)的例子)

#### 那么问题来了，为什么Block是Stack Object，而不是Heap Object？

主要原因是：(n1370)
1. 在缺省GC的时候，通过API来完成创建和转移ownership是极为容易出错的。
2. 每个线程池积累大量的临时对象后，其之间的合作也是个问题。(GCD)
3. malloc的花费大，并且很多同步使用的时候比较划算。

# Block实践

Blocks通常是通过字面量来定义或者声明的（[字面量](https://en.wikipedia.org/wiki/Literal_(computer_programming)是能在编译期建立的一个值，跟字符串等类似，[ObjectiveCLiterals](http://clang.llvm.org/docs/ObjectiveCLiterals.html)）。

Blocks和匿名函数在语法上只有一点不同，`*`用`^`代替。另外的不同点是，是能捕获数据。

其引用的不透明数据可能驻留在automatic(static)内存，global内存，或者heap内存。Blocks也根据所引用的不透明数据被分为三类，global，stack, malloc。

## 语法

1. <http://fuckingblocksyntax.com/>
2. 遵守BNF
3. Block没有参数的话必须指定void，不然则是variadic。参看K&R。
4. Block的引用，强制换换 和 解引用问题。其size不能在编译期确定。如[Checking Objective-C block type?](http://stackoverflow.com/questions/9048305/checking-objective-c-block-type)
5. return type可以根据表达式推断，如果有多个return表达式，其return type得保持一致。

invoke的一个细节

	int (^x)(char);
	void (^z)(void);
	int (^(*y))(char) = &x;
	
	x('a'); 
	(*y)('a');
	(true ? x : *y)('a')
	
## 内存管理(及变量捕获)

复杂表达式体在他的父体中建立了一个新的lexical scope。在这个scope中的变量按normal方式受限于Block，除了那些在automaic(stack)storage。

Block中按五类方式分。...【Blocks Programming Topics】-【Blocks and Variables】-【Type of Variable】。

1. local automatic(stack)变量 会被当做const copy（比__block花费更少）。该capture(binding)是在Block字面量表达式评估的时候完成的。static和global变量除外。
2. 如果编译器知道在实际评估的时候，该变量没有被引用，那么他将不要求捕获该变量。另外你也能强制在Block开头强制引用一个变量，如：`(void) foo;`。
3. Nest Block。Block字面量表达式可能出现在Block字面量表达式里面(nest)，并且被任何内嵌的Block捕获的变量也会被他们enclosing Blocks所隐式捕获。这点上不推荐使用Nest Block，或者你可以使用GCD的[`dispatch_group_t`](http://stackoverflow.com/questions/10643797/wait-until-multiple-operations-executed-including-completion-block-afnetworki)。Block本身的引用拷贝，会在需要的时候被拷贝，即拷贝整个目录树（从顶部开始）。其内存管理，目前我也不是非常清楚。
4. 并不截获C语言的字面量数组，但可以截获指针型的数组。这是因为截获函数没实现相关的方法。Block其更遵循C语言规范。譬如C语言的字面量数组也不能直接相互赋值。
5. 如果一个实例变量被block引用，那么就会捕获self（作为const copy），该实例变量是mutate的。如果是通过值来访问一个实例变量，那么变量会被retain。
6. 关于CF对象，CF对象在Block中也是需要被retain和release的，虽然他也是Objective-C对象，但是编译并不识别他（除非加了`__attribute__((NSObject))`）。尽量避免不正确的内存管理问题，要么避免这类问题，或者使用toll-free bridge，或者保证CF对象的生命周期（跟Block一样长）。比如会有这样的问题[blocks and retaining CoreFoundation objects](http://www.cocoabuilder.com/archive/cocoa/307097-blocks-and-retaining-corefoundation-objects.html)。`__attribute__((NSObject))`被应用到CF对象，以让这样的结构体对象能在Block被autoretain。也一般建议先将CF对象进行toll-free bridge。
7. ARC知道对Block进行Copy，而不是retain。但是其类型不能是`id`类型。

Blocks实际上是Objective-C对象。即使你从C++库创建了Blocks，文档也会说每个Block必须有Objective-C对象的内存布局(memory layout)。然后runtime为他们增加了 Objective-C 方法，这允许他们存储在collections，和properties使用，一般上就能在你期望的任何地方work。这些方法包括：

- -`[Block copy]`: 和`Block_copy()`一样。retain所有引用的automatic storage变量。
- -`[Block retain]`: `Block_retain()`。对于在stack上的block并不能retain，需要将其move 到heap上，即copy。
- -`[Block release]`: `Block_release()`。
- -`[Block autorelease]`: blocks也能自动释放。
- Blocks继承于NSObject。

Objective-C中的Blocks跟C中的Blocks最重要的一个区别是在控制引用对象的变量。所有局部对象在被引用后，都是自动retain的。如果你在方法中的Block引用了一个实例变量，隐式地被修饰为引用了self，他会retain self，就像你隐式地做了self->theIvar。net effect是实例变量是mutated。

__block修饰的对象，其对象指针不能被autoretained(Blocks的auto-retain)。

### ARC

当处于ARC的时候，ARC能正确判断block的生命周期，如当做函数返回值等。但仍有不能正确判断的，如：

	向方法或者函数的参数中传递Block时。当然如果方法或者函数中适当地赋值了传递过来的参数，那么就不必在调用该方法或者函数前手动复制了。以下方法函数不用手动复制。
	Cocoa框架的方法且方法名中含有usingBlock等时。GCD的API。

（一般ARC的数据可以分为操作元，参数和返回值）

Block赋值到堆的时机：
1. 调用Block的copy实例方法
2. Block作为函数返回值返回时
3. 将Block赋值给附有__strong修饰符id类型的类或Block类型成员变量
4. 含有usingBlock的Cocoa框架方法和GCD的API。

## __block storage qualifier

`__block`是一个storage type modifier。

`__block`和已经存在的local storage qualifier auto, register, 和static是互斥的。这样的变量被假定为驻留在allocated storage，Block也一样。这样的指针发生retain和release。

个人觉得也尽量避免使用__block，这会显著地增加花费。

__block的两个限制，不能是可变长数组，并且不能是包含C99可变长度的数组变量的数据结构。

__block to __weak，__unsafe__unretained，__strong，跟平常使用习惯类似。

__block 和 __autoreleasing，没设定同时使用的方法，编译会出错。

## 调试

	gcc $ invoke-block (inv)

[Block Debugging](http://realmacsoftware.com/blog/block-debugging)

更多的详细可查看Advanced Mac OS X Programming -> 3.Blocks -> For the More Curious: Blocks Internals -> Debugging./ Dumping runtime information.

## Block + category(用途)

1. KVO + Block，[kvoblocknotificationcenter](http://github.com/schwa/KVO-Notification-Manager), [Andy Matuschak](http://blog.andymatuschak.org/post/156229939/kvo-blocks-block-callbacks-for-cocoa-observers)，NSNotification + Block
2. [FunctionalKit](https://github.com/mogeneration/functionalkit/tree/blocks), [Simplifying JSON Parsing Using FunctionalKit
](http://adams.id.au/blog/2009/04/simplifying-json-parsing-using-functionalkit/)
3. [GCD的重新实现](Matt Wright, Wigan Wallgate)
4. [MABlockClosure](https://www.mikeash.com/pyblog/friday-qa-2011-05-06-a-tour-of-mablockclosure.html)

打开，关闭file；对数据进行map、过滤，主线程同步，延迟操作，并发枚举，线程混合等。

对代码块进行封装：
@autoreleasing
"run this code immediately after returning to the runloop"RunAfterDelay
write is a critical section of code protected by a lock. 

UIActionSheet+Block
NSURLConnection+Block

gcd

直接调用
作为函数的参数
作为方法的参数

## mikeash的geek玩法

[Creating a Blocks-Based Object System](https://www.mikeash.com/pyblog/friday-qa-2009-10-16-creating-a-blocks-based-object-system.html)

[Trampolining Blocks with Mutable Code](https://www.mikeash.com/pyblog/friday-qa-2010-02-12-trampolining-blocks-with-mutable-code.html)

[Friday Q&A 2010-02-26: Futures](https://www.mikeash.com/pyblog/friday-qa-2010-02-26-futures.html)

[Friday Q&A 2010-03-05: Compound Futures](https://www.mikeash.com/pyblog/friday-qa-2010-03-05-compound-futures.html)

[Link: Implementing imp_implementationWithBloc](https://www.mikeash.com/pyblog/link-implementing-imp_implementationwithblock.html)

[A Tour of MABlockClosure](https://www.mikeash.com/pyblog/friday-qa-2011-05-06-a-tour-of-mablockclosure.html)

[Objective-C Blocks vs. C++0x Lambdas: Fight!](https://www.mikeash.com/pyblog/friday-qa-2011-06-03-objective-c-blocks-vs-c0x-lambdas-fight.html)

[ Generic Block Proxying](https://www.mikeash.com/pyblog/friday-qa-2011-10-28-generic-block-proxying.html)

[Building a Memoizing Block Proxy](https://www.mikeash.com/pyblog/friday-qa-2011-11-11-building-a-memoizing-block-proxy.html)

# Block实现

Block也是由编译器和runtime函数一起支持完成的。其实际上是作为极普通的C语言源代码来处理的。通过支持Block的编译器，含有Block语法的源代码转换为一般C语言编译器能够处理的源代码，并作为极为普通的C语言源代码被编译。

	$ clang -rewrite-objc 源代码文件名


定义一个Block字面量会让编译器生成两个structure，以及至少一个函数。这两个structure是block literal（block holder）和 block descriptor。

## Block Layout

即为__xxx_block_impl_0 和 __xxx_block_desc_0。

Block字面量表达式初始化的时候，分为两类*&_NSConcreteStackBlock* 或者 *&_NSConcreteGlobalBlock*。

### isa

isa的取值为data.c中的, 相关描述可见【Advanced Mac OS X Programming】-【3. Blocks】-【For the More Curious:Blocks Internals】- 【Block Literals】

	void * _NSConcreteStackBlock[32] = { 0 }; block在stack上；数据存在stack
	void * _NSConcreteMallocBlock[32] = { 0 }; block是在heap上。数据存在heap
	void * _NSConcreteAutoBlock[32] = { 0 }; block在collectable memory上。当在gc下，并且没引用被拷贝到heap的C++对象。
	void * _NSConcreteFinalizingBlock[32] = { 0 }; block在collectable memory上并必须有...即
	void * _NSConcreteGlobalBlock[32] = { 0 }; block是在global storage。如果是 static 或者 file level Block literal. 数据在data区，包括不捕获数据的block变量或者全局block变量。
	void * _NSConcreteWeakBlockVariable[32] = { 0 }; 变量是被__weak和__block所修饰，当他被拷贝到heap上的时候，Block_byref的isa为这个。

其并不仅仅只有三个。都继承于`_NSAbstractBlock`。该抽象类提供了被Block使用的内存相关方法实现。collectable memory 可能和GC，ARC相关。

### flags

flags在Block_private.h开头

	// Values for Block_layout->flags to describe block objects
	enum {
	    BLOCK_DEALLOCATING =      (0x0001),  // runtime
	    BLOCK_REFCOUNT_MASK =     (0xfffe),  // runtime
	    BLOCK_NEEDS_FREE =        (1 << 24), // runtime
	    BLOCK_HAS_COPY_DISPOSE =  (1 << 25), // compiler
	    BLOCK_HAS_CTOR =          (1 << 26), // compiler: helpers have C++ code
	    BLOCK_IS_GC =             (1 << 27), // runtime
	    BLOCK_IS_GLOBAL =         (1 << 28), // compiler
	    BLOCK_USE_STRET =         (1 << 29), // compiler: undefined if !BLOCK_HAS_SIGNATURE
	    BLOCK_HAS_SIGNATURE  =    (1 << 30), // compiler
	    BLOCK_HAS_EXTENDED_LAYOUT=(1 << 31)  // compiler
	};

## 变量引入(Imported Variables) 和 __block

`auto`存储类变量是const copy。`__block`存储类变量是作为enclosing数据结构的指针。global只是简单的引用，被没被import。

总结，标量(scalars), 结构体(structures), unions, 和函数指针 都是const copy，并不需要helper functions。
Block引用的const copy是需要helper functions。还可以给struct等加上__attribute__((NSObject))，以来增加helper functions。但也都只是在__xxx_block_impl_0中过一个const字段。

Helper fuctions一般是`__block_copy_xxx`和`__block_dispose_xx`，分别调用`_Block_object_assign`和`_Block_object_dispose`。用于被引用的对象(Object)的copy和dispose，可以简单的认为是retainable指针对象。

### __block

__block变量会被编译成一个Block_byref structure（因为所有的__block变量都是by reference来访问的，所以命名加上byref），如果有必要的话，会多一个辅助函数。其也是通过forwarding指针来访问(能让多个block使用一份__block所生成的structure)。当__block指针从stack被拷贝到heap上的时候，on-stack和in-heap的forwarding指针都会被更新为in-heap structure。

Block_byref的isa总是以NULL开始。当变量是被__weak和__block所修饰，当他被拷贝到heap上的时候，Block_byref的isa为`&NSConcreteWeakBlockVariable`。并且如果是（可能是）retainable指针对象（Block引用，Objective-C对象，C++对象），那么编译器会合成helper functions。

下述的两个议题在Advanced Max OS X Programming.

## 实现的进化 

跟早期相比，flags字段不断在被扩展，reserved字段被重新规划，新的字段也可被加在各种structure的后面。增加变量signature字段来消除helper fuctions。

## 编译器生成的名字

在debug的时候，有时候可以用来猜猜生成的block的名字。

一般是 所调用的block，__函数名_block_invoke_序号 (gcc和clang，在序号上不一样)。序号递增。

## Runtime支持

_Block_object_assign 和 _Block_object_dispose

# 更多

<http://macoscope.com/blog/calling-blocks-inline>