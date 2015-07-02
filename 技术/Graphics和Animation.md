# 前言

主要包含在UIKit，CoreGraphics，QuartzCore。

## 架构

UIKit/AppKit

|||||||||||||||||||||||||||||||

Core Animation

.------------------------------.

OpenGL ES/OpenGL | Core Graphics

.------------------------------.

Graphics Hardware

其中，

1. UIKit, framework:*UIKit*, 文档[View Programming Guide](http://www.cnblogs.com/patientAndPersist/tag/iOS/default.html?page=2),
2. Core Animation, framework:*QuartzCore*, 文档:[Core Animation Programming Guide](http://www.cnblogs.com/xdream86/p/3250782.html),
3. OpenGL ES(3D绘制), framework:*GLKit and CAEAGLLayer*, 文档:[OpenGL ES Programming Guide](),
4. Core Graphics(2D绘制), framework:*CoreGraphics*, 文档[Quarzt 2D Programming Guide](http://www.cnblogs.com/xdream86/archive/2012/12/12/2814552.html)

# Graphics 和 Animation 基础

## UIView

从UIKit出发，我们可以了解到View Drawing Circle 和 The Runtime Interaction Model for Views。这也是high-level上的原理。

### View Drawing Circle

UIView 类使用一个按需绘制模型(on-demand drawing model)来呈现视图内容。 当视图第一次出现在屏幕上时，系统要求它绘制其内容。 系统捕捉该内容的快照，并使用该快照作为视图的视觉代理(visual representation)。 如果你从来没有改变视图的内容，那视图的绘制代码可能永远也不会再被调用。快照图片被视图设计的大多数操作所重用。如果你对内容做了改变，你需要通知系统告诉它视图已经改变。 然后视图重复绘制视图过程，重新捕捉新的快照。

当视图内容发生改变时，你不会直接重新绘制那些改变。你可以用 setNeedsDisplay 或 setNeedsDisplayInRect: 方法让视图无效。 这些方法告诉系统视图的内容已经发生改变，需要在下次重新绘制。 系统在直到当前运行循环(current run loop)结束，在任何绘制操作初始化之前一直等待。 该等待给你一次机会，用来无效多个视图，添加或从视图层次里删除视图，隐藏视图，调整视图大小，一次性重新定位所有视图等。所有这些你做的变化将在同一时间反映。

注意：改变视图的几个外形并不会自动导致系统重画视图内容。 视图的contentMode 属性解释了如何对视图几何外形做改变。大多数内容模式都是在视图边界里拉伸或重定位已经存在的快照，并不需要创建新视图内容。

当开始渲染视图内容时，实际绘图进程变化取决于视图和它的配置。 系统视图通常实现似有绘图方法来渲染它们的内容。 那些相同的系统视图常常暴露出你可以用来配置视图真实外观的接口。比如定制UIView 子类，你通常重载视图的drawRect: 方法，并用该方法绘制视图的内容。还有其它方法来提供一个视图的内容，比如直接设置下面的层内容，但是重载 drawRect: 方法是最通用的技术。

### The Runtime Interaction Model for Views

![](https://developer.apple.com/library/ios/documentation/WindowsViews/Conceptual/ViewPG_iPhoneOS/Art/drawing_model.jpg)

下面的步骤更进一步的分析了 图1-7 中发生的事件， 解释了每一步发生的什么，以及你可能需要你的应用程序如何响应。

1. 用户触摸了屏幕。 

2. 硬件向UIKit 框架报告了该触摸事件。

3. UIKit 框架把触摸事件打包进一个  UIEvent 对象， 并把它发派给相应的视图。(UIKit 如何传递事件到视图的详细解释，请看Event Handling Guide for iOS)

4. 视图 的事件处理代码 响应该事件。 比如，你的代码可能：
	* 改变视图或它的子视图的属性(frame , bounds, alpha, 等等)。
	* 调用 setNeedsLayout 方法来标记该视图(或其子视图)，作为一个需要更新的布局。
	* 调用setNeedsDisplay 或setNeedsDisplayInRect: 方法来标记视图(或其子视图)，作为需要被重新绘制。
	* 通知一个控制器有关一些数据块的改变。

	当然，由你决定这些事情，哪些视图需要做，哪些方法需要被调用。

5. 如果视图的几何外形因为任何原因发生了改变， UIKit 参照以下规则更新它的子框架：
	* 如果你已经给你的视图设置了自动调整大小规则， UIKit 根据那些规则调整每个视图。 更多有关自动调整大小规则是如何工作的信息，请看“Handling Layout Changes Automatically Using Autoresizing Rules.”
	* 如果视图实现了layoutSubviews 方法， UIKit 框架调用它。
	你可以在你的自定义视图里重载该方法， 并用它调整任何子视图的位置及大小。 比如， 一个提供大滚动区域的视图需要使用几个子视图作为瓦片(tiles), 而不是创建一个无论如何也不太可能适应内存的大视图。在该方法的实现中，视图可能隐藏任何现在不用显示的子视图，或者重新定位它们并使用它们去绘制新暴露的内容。作为该过程的一部分， 视图布局代码也可以让需要被重画的任何视图无效。

6. 如果任何视图的任何部分被标记为需要被重画， UIKit 要求视图重画它自己。
	对于明确定义了一个 drawRect: 方法的自定义视图， UIKit 调用那个方法。 该方法的实现应该是尽可能快的重画视图的指定区域，而不做其它任何事情。 不要在该这时候做额外的布局改变， 也不要给应用程序数据模型做任何其它改变。 该方法的目的是更新视图的可见内容。
	标准系统视图通常不会实现一个drawRect:方法， 但是会在这时候管理它们的绘图。

7. 所有视图更新会被合并到应用程序可见内容的其余部分中，并被发送给图形硬件以便显示。

8. 图形硬件在屏幕上绘制这些内容。

注意： 先前的更新模型主要应用于使用标准系统视图以及绘图技术的应用程序。通常使用OpenGL ES来绘制的应用程序都配置了一个全屏视图， 并直接在相关的OpenGL图形上下文上绘制。 在这种情况下，视图可能任然需要处理触摸事件，但是因为它是全屏的，它可能不需要布局子视图或实现一个drawRect:方法。 更多使用OpenGL ES的信息，请看OpenGL ES Programming Guide for iOS.

在前面的步骤中，你自定义视图的主要交互点是:

* 事件处理方法：
	touchesBegan:withEvent:

	touchesMoved:withEvent:

	touchesEnded:withEvent:

	touchesCancelled:withEvent:

* The layoutSubviews method

 	layoutSubviews 方法

* The drawRect: method

 	drawRect:方法

这些是视图最常重载的方法，但是你可能不需要全部重载。 如果你使用手势辨认器来处理事件，你就不需要重载任何一个事件处理方法。 同样地，如果你的视图不包含子视图或它的尺寸不会改变，也没有原因要重载layoutSubviews 方法。 最后， 当你的视图内容在运行时能发生改变时，drawRect:方法是你仅需重载的一个方法， 并且你使用本地技术，比如UIKit 或 Core Graphics 来完成绘图。

请记住这些方法是主要的交互点，但不是只有这些点才可以交互。 UIView的一些方法也可以被设计可以成为子类们的重载点(override points)。你应该在UIView Class Reference 里查看方法说明，查看它们哪些方法适合你在自定义实现里重载。

## CALayer

每个UIView都有一个CALayer(即所谓的backing layer)，其绘制还是通过CALayer来完成的。实际上这些背后关联的图层layer才是真正用来在屏幕上显示和做动画，UIView仅仅是对它的一个封装，提供了一些iOS类似于处理触摸的具体功能，以及Core Animation底层方法的高级接口。

但是为什么iOS要基于UIView和CALayer提供两个平行的层级关系呢？为什么不用一个简单的层级来处理所有事情呢？原因在于要做职责分离，这样也能避免很多重复代码。在iOS和Mac OS两个平台上，事件和用户交互有很多地方的不同，基于多点触控的用户界面和基于鼠标键盘有着本质的区别，这就是为什么iOS有UIKit和UIView，但是Mac OS有AppKit和NSView的原因。他们功能上很相似，但是在实现上有着显著的区别。

![](http://images.cnitblog.com/blog/429321/201309/10231326-77cd2c5191e04a989fe0ac56dc52c04c.png)

### Backing Image

CALayer有一个属性叫做contents, 需要是CGImage类型。还有诸如contentGravity, contentsScale, maskToBounds, contentsRect, contentsCenter。

Core Animation通过尽可能的使用图形硬件操纵缓存后的位图来避免了这种开销，从而完成相同或相似的效果。

### 自定义绘制

当然，给contents赋值并不是自定义绘制的唯一方法。还有：

* Core Graphics直接绘制backing image
* 重载UIView的drawRect方法

#### UIView的drawRect

通过调用setNeedsDisplay.

方法`drawRect:`方法没有默认的实现，因为对UIView来说，Backing Image并不是必须的，它不在意那到底是单调的颜色还是有一个图片的实例。如果UIView检测到-drawRect: 方法被调用了，它就会为视图分配一个寄宿图，这个寄宿图的像素尺寸等于视图大小乘以 contentsScale的值。

如果你不需要寄宿图，那就不要创建这个方法了，这会造成CPU资源和内存的浪费，这也是为什么苹果建议：如果没有自定义绘制的任务就不要在子类中写一个空的-drawRect:方法。

当视图在屏幕上出现的时候 -drawRect:方法就会被自动调用。-drawRect:方法里面的代码利用Core Graphics去绘制一个寄宿图，然后内容就会被缓存起来直到它需要被更新（通常是因为开发者调用了-setNeedsDisplay方法，尽管影响到表现效果的属性值被更改时，一些视图类型会被自动重绘，如bounds属性）。虽然-drawRect:方法是一个UIView方法，事实上都是底层的CALayer安排了重绘工作和保存了因此产生的图片。

#### CALayer的displayLayer

通过调用display.

CALayerDelegate中

/* If defined, called by the default implementation of the -display
 * method, in which case it should implement the entire display
 * process (typically by setting the `contents' property). */

- (void)displayLayer:(CALayer *)layer;

/* If defined, called by the default implementation of -drawInContext: */

- (void)drawLayer:(CALayer *)layer inContext:(CGContextRef)ctx;


### 更多关于CALayer

1. 各种视觉效果，可查看《iOS Core Animation Advanced Teachniques》的视觉效果 章节。包括圆角，阴影和蒙板，拉伸过滤器和组透明。
2. 变换，包括affineTransform(2D), transform(3D), sublayerTransform。图层是双面绘制的(doubleSided)，反面显示的是正面的一个镜像图片。当改变一个图层的position，你也改变了它的消亡点，做3D变换的时候要时刻记住这一点，当你视图通过调整m34来让它更加有3D效果，应该首先把它放置于屏幕中央，然后通过平移来把它移动到指定位置（而不是直接改变它的position），这样所有的3D图层都共享一个消亡点。
3. 专用图层，包括CAShapeLayer、CATextLayer、CATransformLayer、CAGradientLayer、CAReplicatorLayer、CAScrollLayer、CATiledLayer、CAEmitterLayer、CAEAGLLayer、AVPlayerLayer。


## 总结

view-based drawing techniques 经常通过调用view的`drawRect:`来改变view本身，CPU上的操作。而layer-based drawing techniques将一个静态位图（也称为backing store）和layer信息送给GPU来完成。

layer的意义，对于iOS来说view只是layer的封装，而对于Max OS X来说，layer是另外一种绘图方式的选择。这里，可以简单认为view和layer分别代表了CPU和GPU绘图机制。其中view通过drawRect(也就是使用CG框架来完成)，layer通过contents(CA)来完成。而替换view的layerClass,也相当于换了种绘图方法。

这也就能理解addSubView和drawRect的区别，前者的底层是CA，后者是CG。

## 更多

[objcio.Views](http://objccn.io/issue-3/)

# 动画

使用CA的app，有三种不同类型的layer对象:

1. model layer tree, 目标值。
2. presentation tree, 动画中的值，并最好只在动画中访问。
3. render tree，完成实际动画，并且是private的。

## 动画框架

![](http://ww1.sinaimg.cn/large/65cc0af7gw1dxlusbklpmj.jpg)

可以看到动画主要由属性动画、过渡动画、组动画以及时间系统构成：

1. 属性动画包括CABasicAnimation和CAKeyframeAnimation两个对可动画属性操作的动画。
2. 过渡动画CATransition是处理不可动画属性的东西，如view过渡。
3. 组动画CAAnimationGroup是把多个属性动画打包。
4. 时间系统CAMediaTiming是能精确控制动画执行的节点。

### 动画机制

CA 假设 视图和其他的可视元素 都可以动画，为你完成了动画所需的大部分绘制工作。基本上CA的对象就是UIView和CALayer。CA支持CATransaction 隐式/显式 的动画。

#### CATransaction 隐式和显式

CA通过事务CATransaction来管理动画。任何用指定事务去改变可以做动画的图层属性都不会立刻发生变化，而是当事务一旦提交的时候开始用一个动画过渡到新值。管理了一叠你不能访问的事务。CATransaction 没有属性或者实例方法，并且也不能用+alloc和-init方法创建它。但是可以用+begin和+commit分别来入栈或者出栈。

CATransaction也分两类,显式的和隐式的。CA在每个run loop周期中自动开始一次新的事务（run loop是iOS负责收集用户输入，处理定时器或者网络事件并且重新绘制屏幕的东西），即使你不显式的用[CATransaction begin]开始一次事务，任何在一次run loop循环中属性的改变都会被集中起来，然后做一次0.25秒的动画。如果在没有RunLoop的地方设置layer的animatable属性,则必须使用显式的事务.其中，隐式动画常见于CALayer上的属性改变后的动画。

在UIView的系统API的动画中，CATransaction的+begin和+commit方法在其内部自动调用，这样block中所有属性的改变都会被事务所包含。这样也可以避免开发者由于对+begin和+commit匹配的失误造成的风险。

#### 隐式动画原理

首先，值得注意的是UIKit禁掉 UIView关联的CALayer 的 隐式动画。你对UIView的属性操作是没有动画的。那么问题来了，隐式动画原理是？

当CALayer的属性被修改时候，它会调用-actionForKey:方法，传递属性的名称。

1. 图层首先检测它是否有委托，并且是否实现CALayerDelegate协议指定的-actionForLayer:forKey方法。如果有，直接调用并返回结果。
2. 如果没有委托，或者委托没有实现-actionForLayer:forKey方法，图层接着检查包含属性名称对应行为映射的actions字典。(action里面存储了属性名和对应的动画)
3. 如果actions字典没有包含对应的属性，那么图层接着在它的style字典接着搜索属性名。
4. 最后，如果在style里面也找不到对应的行为，那么图层将会直接调用定义了每个属性的标准行为的-defaultActionForKey:方法。图层执行由Core Animation定义的隐式动作(有得话)。

		* 1. if defined, call the delegate method -actionForLayer:forKey:
		* 2. look in the layer's `actions' dictionary
		* 3. look in any `actions' dictionaries in the `style' hierarchy
		* 4. call +defaultActionForKey: on the layer's class
	 
所以一轮完整的搜索结束之后，-actionForKey:要么返回空（这种情况下将不会有动画发生），要么是CAAction协议对应的对象，最后CALayer拿这个结果去对先前和当前的值做动画。

UIKit是如何禁用隐式动画的：每个UIView对它关联的图层都扮演了一个委托，并且提供了 -actionForLayer:forKey 的实现方法。当不在一个动画块的实现中，UIView对所有图层行为返回nil，但是在动画block范围之内，它就返回了一个非空值。actionForLayer:forKey返回的是CABasicAnimation这类。且其返回的是属性动画的同时仍是被事务进行管理。当然返回nil并不是禁用隐式动画唯一的办法，CATransacition有个方法叫做+setDisableActions:，可以用来对所有属性打开或者关闭隐式动画。

启用UIView的隐式动画

* UIView关联的图层禁用了隐式动画，对这种图层做动画的唯一办法就是使用UIView的动画函数（而不是依赖CATransaction），或者继承UIView，并覆盖-actionForLayer:forKey:方法，或者直接创建一个显式动画。
* 对于单独存在的图层，我们可以通过实现图层的-actionForLayer:forKey:委托方法，或者提供一个actions字典来控制隐式动画。

#### 显式动画的状态保留

显式动画结束后，又会变回之前的外观状态。因为动画只是改变了展示树的值。

解决这个问题的几个方法：

1. 动画之前改变属性
	将fromevalue改为属性之前的值，然后设置属性。这里仍有不少问题，如正在进行中的动画，还得从呈现树中获取fromvalue。另外还得禁用隐式动画。（虽然显式动画通常会覆盖隐式动画）
	
2. 动画之后改变属性
	CAAnimationDelegate的协议中-animationDidStop:finished:中更新，并禁掉隐式动画。但也仍有问题，多个动画不容易区分。不过你可以通过CAAnimation的KVC来完成。
	最后的结果是在调用delegate方法之前，马上返回到了原始值。

3. CAMediaTiming的fillMode属性
	
	使得动画开始前，图层保持动画开始的状态还是结束的值。因为这是动画的属性，所以还需要设置removeOnCompletion = NO
		
	kCAFillModeForwards 
	kCAFillModeBackwards 
	kCAFillModeBoth 
	kCAFillModeRemoved

### 时间系统(CAMediaTiming)

包括：beginTime、duration、speed、timeOffset、repeatCount、repeatDuration、autoreverses、fillMode。各自的含义可以直接看文档，其中：
本地时间t=(父层时间tp-beigin)*speed+timeOffset。

动画时间和它类似，每个动画和图层在时间上都有它自己的层级概念，相对于它的父亲来测量。对图层调整时间将会影响到它本身和子图层的动画，但不会影响到父图层。另一个相似点是所有的动画都被按照层级组合。

对CALayer或者CAGroupAnimation调整duration和repeatCount/repeatDuration属性并不会影响到子动画。但是beginTime，timeOffset和speed属性将会影响到子动画。然而在层级关系中，beginTime指定了父图层开始动画（或者组合关系中的父动画）和对象将要开始自己动画之间的偏移。类似的，调整CALayer和CAGroupAnimation的speed属性将会对动画以及子动画速度应用一个缩放的因子。

因此控制 self.window.layer.speed 来控制全局的动画速度。

这里的时间变化是靠动画本身驱动的。

CA有个全局时间, Mach Time（设备上的所有进程都一样，是设备自上次启动后的秒数，设备休眠的时候会暂停，也就是所有CA动画都会暂停，因此不适合闹钟）。

#### Easing（CAMediaTimingFunction）

通过CAAnimation的timingFunction来控制动画的移动加速度。通常也是贝塞尔函数来完成。

[贝塞尔参数在线生成器Cubic-bezier](http://cubic-bezier.com/#.17,.67,.83,.67)

[常见缓懂函数](http://easings.net/zh-cn)

#### 计时器NSTimer和CADisplayerLink

NSTimer的触发是和runloop的任务列表一起的。因此通常是会有延迟。有时候比屏幕重绘快，有时候慢。

CADisplayerLink则总是在屏幕完成一个更新之前启动。其概念适合1秒60帧关联的。frameInterval表示隔几帧更新一次。其也不能保证每帧都按计划执行，一些失去控制的离散任务或者事件(如资源紧张的后台程序)可能会导致动画偶尔地丢帧，CADisplayer会忽略丢失的帧，下次接着运行。而NSTimer不会。

在创建CADisplayerLink的时候，需要指定run loop和run loop mode。一般来说run loop是[NSRunLoop mainRunLoop]，而mode则要看具体的任务。

#### 暂停和恢复动画

-(void)pauseLayer:(CALayer*)layer
{
    CFTimeInterval pausedTime = [layer convertTime:CACurrentMediaTime() fromLayer:nil];
    layer.speed = 0.0;
    layer.timeOffset = pausedTime;
}

-(void)resumeLayer:(CALayer*)layer
{
    CFTimeInterval pausedTime = [layer timeOffset];
    layer.speed = 1.0;
    layer.timeOffset = 0.0;
    layer.beginTime = 0.0;
    CFTimeInterval timeSincePause = [layer convertTime:CACurrentMediaTime() fromLayer:nil] - pausedTime;
    layer.beginTime = timeSincePause;
}

## 不同类型动画的讨论

### CAKeyFrameAnimation

其不单单靠keyTimes和timingFunctions来完成定时器，即移动的变化曲线。

指定关键帧动画的定时器

关键帧动画的定时与步调比基本动画来的要复杂。以下是几个用于控制定时和步调的属性：
calculationMode属性定义了计算动画定时的算法。该属性值会影响其他与定时相关属性的使用方式。

线性和曲线动画，动画的calculationMode属性被设置为kCAAnimationLinear或CAAnimationCubic，属性值被用于提供定时器信息以生成动画。这些模式值让你最大化控制动画的定时器。

* 节奏动画，动画的calculationMode属性被设置为kCAAnimationPaced或kCAAnimationCubicPaced，这些属性值不依赖由keyTimes或timingFunctions属性提供的额外定时器值。相反，定时器值被隐式地计算以提供一个常速率动画。

* 离散动画，动画的calculationMode属性被设置为kCAAnimationDiscrete，该值将引起动画属性从一个关键帧跳到另一个没有任何补间动画的下一个关键帧。计算模式使用keyTimes属性值，但忽略timingFunctions属性。

* keyTimes属性为应用在每一关键帧指定应用到每一个关键帧上的计时器。该属性只在calculationMode属性被设置为kCAAnimationLinear，kCAAnimaitonDiscrete，kCAAnimationCubic时被使用。它不使用在节奏动画中。

* timingFunctions属性指定使用在每一个关键帧部分的定时曲线(该属性替换了继承的timingFunction属性)。

如果你想自己处理动画的定时，可以使用kCAAnimationLinear或kCAAnimaitonCubic模式与keyTimes和timingFunctions属性。keyTimes定义了应用在每一关键帧的时间点。所有中间值的定时由定时函数控制，定时函数允许你对各个部分应用缓入或缓出曲线定时。如果你不指定任何定时函数，动画将会是线性的。

## 动画效果汇总

[Timeoffset的控制](http://ronnqvi.st/controlling-animation-timing/)
[多个动画的合成](http://ronnqvi.st/multiple-animations/)
[objc.io动画](http://objccn.io/issue-12/)
<http://ronnqvi.st/clear-animation-code/>

## 动画工具

# 性能调优

## 原理

这里完全参考 《iOS-Core-Animation-Advanced-Techniques》

动画和屏幕上组合的图层实际上被一个单独的进程管理，而不是你的应用程序。这个进程就是所谓的渲染服务。在iOS5和之前的版本是SpringBoard进程（同时管理着iOS的主屏）。在iOS6之后的版本中叫做BackBoard。

当运行一段动画时候，这个过程会被四个分离的阶段被打破：

* **布局** - 这是准备你的视图/图层的层级关系，以及设置图层属性（位置，背景色，边框等等）的阶段。
* **显示** - 这是图层的寄宿图片被绘制的阶段。绘制有可能涉及你的`-drawRect:`和`-drawLayer:inContext:`方法的调用路径。
* **准备** - 这是Core Animation准备发送动画数据到渲染服务的阶段。这同时也是Core Animation将要执行一些别的事务例如解码动画过程中将要显示的图片的时间点。
* **提交** - 这是最后的阶段，Core Animation打包所有图层和动画属性，然后通过IPC（内部处理通信）发送到渲染服务进行显示。

但是这些仅仅阶段仅仅发生在你的应用程序之内，在动画在屏幕上显示之前仍然有更多的工作。一旦打包的图层和动画到达渲染服务进程，他们会被反序列化来形成另一个叫做渲染树的图层树（在第一章“图层树”中提到过）。使用这个树状结构，渲染服务对动画的每一帧做出如下工作：

* 对所有的图层属性计算中间值，设置OpenGL几何形状（纹理化的三角形）来执行渲染

* 在屏幕上渲染可见的三角形

所以一共有六个阶段；最后两个阶段在动画过程中不停地重复。前五个阶段都在软件层面处理（通过CPU），只有最后一个被GPU执行。而且，你真正只能控制前两个阶段：布局和显示。Core Animation框架在内部处理剩下的事务，你也控制不了它。

![](https://github.com/100mango/zen/raw/master/WWDC%E5%BF%83%E5%BE%97%EF%BC%9AAdvanced%20Graphics%20and%20Animations%20for%20iOS%20Apps/rendering_pass.png)

### GPU操作

宽泛的说，大多数CALayer的属性都是用GPU来绘制。比如如果你设置图层背景或者边框的颜色，那么这些可以通过着色的三角板实时绘制出来。如果对一个contents属性设置一张图片，然后裁剪它 - 它就会被纹理的三角形绘制出来，而不需要软件层面做任何绘制。

但是有一些事情会降低（基于GPU）图层绘制，比如：

* 太多的几何结构 
* 重绘
* 离屏绘制
* 过大的图片

#### 图层的性能

1. 隐式绘制
	CATextLayer和UILabel的背后机制。
	shouldRasterize机制，可以解决重叠透明图层的混合失灵问题。然后这个图像将会被缓存起来并绘制到实际图层的contents和子图层。如果有很多的子图层或者有复杂的效果应用，这样做就会比重绘所有事务的所有帧划得来得多。但是光栅化原始图像需要时间，而且还会消耗额外的内存。
2. 屏幕外渲染
	当图层属性的混合体被指定为在未预合成之前不能直接在屏幕中绘制时，屏幕外渲染就被唤起了。屏幕外渲染并不意味着软件绘制，但是它意味着图层必须在被显示之前在一个屏幕外上下文中被渲染（不论CPU还是GPU）。图层的以下属性将会触发屏幕外绘制：圆角（当和maskToBounds一起使用时）、图层蒙板、阴影。屏幕外渲染和我们启用光栅化时相似，除了它并没有像光栅化图层那么消耗大，子图层并没有被影响到，而且结果也没有被缓存，所以不会有长期的内存占用。但是，如果太多图层在屏幕外渲染依然会影响到性能。有时候我们可以把那些需要屏幕外绘制的图层开启光栅化以作为一个优化方式，前提是这些图层并不会被频繁地重绘。对于那些需要动画而且要在屏幕外渲染的图层来说，你可以用CAShapeLayer，contentsCenter或者shadowPath来获得同样的表现而且较少地影响到性能。
3. 混合和过度绘制
	GPU每一帧可以绘制的像素有一个最大限制。GPU会放弃绘制那些完全被其他图层遮挡的像素，但是要计算出一个图层是否被遮挡也是相当复杂并且会消耗处理器资源。同样，合并不同图层的透明重叠像素（即混合）消耗的资源也是相当客观的。所以为了加速处理进程，不到必须时刻不要使用透明图层。任何情况下，你应该这样做：给视图的backgroundColor属性设置一个固定的，不透明的颜色；设置opaque属性为YES。最后，明智地使用shouldRasterize属性，可以将一个固定的图层体系折叠成单张图片，这样就不需要每一帧重新合成了，也就不会有因为子图层之间的混合和过度绘制的性能问题了。
4. 减少图层数量
	* 不绘制不可见图层
	* 对象回收池（处理巨大数量的相似视图或图层时还有一个技巧就是回收他们）。
	* Core Graphics绘制，在因为图层数量而使得性能受限的情况下，软件绘制很可能提高性能呢，因为它避免了图层分配和操作问题。
	* -renderInContext: 方法

### CPU操作

* 布局计算
* 视图懒加载
* Core Graphics绘制 
* 解压图片

当图层被成功打包，发送到渲染服务器之后，CPU仍然要做如下工作：为了显示屏幕上的图层，Core Animation必须对渲染树种的每个可见图层通过OpenGL循环转换成纹理三角板。由于GPU并不知晓Core Animation图层的任何结构，所以必须要由CPU做这些事情。这里CPU涉及的工作和图层个数成正比，所以如果在你的层级关系中有太多的图层，就会导致CPU没一帧的渲染，即使这些事情不是你的应用程序可控的。

#### 高效绘图

软件绘图的代价昂贵，除非绝对必要，你应该避免重绘你的视图。提高绘制性能的秘诀就在于尽量避免去绘制。

1. 特定图层类

Core Animation为这些图形类型的绘制提供了专门的类，并给他们提供硬件支持。CAShapeLayer可以绘制多边形，直线和曲线。CATextLayer可以绘制文本。CAGradientLayer用来绘制渐变。这些总体上都比Core Graphics更快，同时他们也避免了创造一个寄宿图。

2. 重绘脏区域（DrawRect和setNeedsDisplayInRect）
3. CATiledLayer和drawsAsynchronously

#### DrawRect原理

 如果对视图实现了-drawRect:方法，或者CALayerDelegate的-drawLayer:inContext:方法，那么在绘制任何东西之前都会产生一个巨大的性能开销。为了支持对图层内容的任意绘制，Core Animation必须创建一个内存中等大小的寄宿图片。然后一旦绘制结束之后，必须把图片数据通过IPC传到渲染服务器。在此基础上，Core Graphics绘制就会变得十分缓慢，所以在一个对性能十分挑剔的场景下这样做十分不好。

### IO操作

优化从闪存或者网络中加载和显示图片。

对iPhone5来说，1/60秒要加载大概700KB左右的图片。

1. 线程加载
	在主线程上加载大图，容易卡顿。提升性能唯一的方式就是在另一个线程中加载图片。这并不能够降低实际的加载时间。为了在后台线程加载图片，我们可以使用GCD或者`NSOperationQueue`创建自定义线程，或者使用`CATiledLayer`。为了从远程网络加载图片，我们可以使用异步的`NSURLConnection`，但是对本地存储的图片，并不十分有效。
2. 延迟解压
	一旦图片文件被加载就必须要进行解码，解码过程是一个相当复杂的任务，需要消耗非常长的时间。解码后的图片将同样使用相当大的内存。当加载图片的时候，iOS通常会延迟解压图片的时间，直到加载到内存之后。这就会在准备绘制图片的时候影响性能，因为需要在绘制之前进行解压（通常是消耗时间的问题所在）。
	`+imageWithContentsOfFile:`在加载图片之后才进行解压。
	* 使用`UIImage`的`+imageNamed:`方法避免延时加载。
	* 把它设置成图层内容，或者是`UIImageView`的`image`属性。不幸的是，这又需要在主线程执行，所以不会对性能有所提升。
	* 绕过`UIKit`，使用ImageIO框架
	* 立即绘制到CGContext
3. `CATiledLayer`
	可以用来异步加载和显示大型图片，而不阻塞用户输入。使用`CATiledLayer`有几个潜在的弊端：
	* `CATiledLayer`的队列和缓存算法没有暴露出来，所以我们只能祈祷它能匹配我们的需求
	* `CATiledLayer`需要我们每次重绘图片到`CGContext`中，即使它已经解压缩，而且和我们单元格尺寸一样（因此可以直接用作图层内容，而不需要重绘）
4. 分辨率交换（为大图准备小图）
	为了做到图片交换，我们需要利用`UIScrollView`的一些实现`UIScrollViewDelegate`协议的委托方法（和其他类似于`UITableView`和`UICollectionView`基于滚动视图的控件一样）：    - (void)scrollViewDidEndDragging:(UIScrollView *)scrollView willDecelerate:(BOOL)decelerate;    - (void)scrollViewDidEndDecelerating:(UIScrollView *)scrollView;	
你可以使用这几个方法来检测传送器是否停止滚动，然后加载高分辨率的图片。只要高分辨率图片和低分辨率图片尺寸颜色保持一致，你会很难察觉到替换的过程（确保在同一台机器使用相同的图像程序或者脚本生成这些图片）。
5. 缓存
	`imageNamed:`和自定义缓存`NSCache`。
6. 文件格式
	图片加载性能取决于加载大图的时间和解压小图时间的权衡。不同文件格式适用于不同情况。诸如PNG, JPEG, 混合图片, JPEG2000, PVRTC。
	PNG图片使用的无损压缩算法可以比使用JPEG的图片做到更快地解压，但是由于闪存访问的原因，这些加载的时间并没有什么区别。
	
#### 自定义缓存

如果要写自己的图片缓存的话，那该如何实现呢？让我们来看看要涉及哪些方面：
* 选择一个合适的缓存键 - 缓存键用来做图片的唯一标识。如果实时创建图片，通常不太好生成一个字符串来区分别的图片。在我们的图片传送带例子中就很简单，我们可以用图片的文件名或者表格索引。
* 提前缓存 - 如果生成和加载数据的代价很大，你可能想当第一次需要用到的时候再去加载和缓存。提前加载的逻辑是应用内在就有的，但是在我们的例子中，这也非常好实现，因为对于一个给定的位置和滚动方向，我们就可以精确地判断出哪一张图片将会出现。
* 缓存失效 - 如果图片文件发生了变化，怎样才能通知到缓存更新呢？这是个非常困难的问题（就像菲尔 卡尔顿提到的），但是幸运的是当从程序资源加载静态图片的时候并不需要考虑这些。对用户提供的图片来说（可能会被修改或者覆盖），一个比较好的方式就是当图片缓存的时候打上一个时间戳以便当文件更新的时候作比较。
* 缓存回收 - 当内存不够的时候，如何判断哪些缓存需要清空呢？这就需要到你写一个合适的算法了。
	
## 性能瓶颈分析

* 时间分析器 - 用来测量被方法/函数打断的CPU使用情况。

* Core Animation - 用来调试各种Core Animation性能问题。

* OpenGL ES驱动 - 用来调试GPU性能问题。这个工具在编写Open GL代码的时候很有用，但有时也用来处理Core Animation的工作。

### 时间分析器

* 通过线程分离
* 隐藏系统库
* 只显示Obj-C代码 - 隐藏除了Objective-C之外的所有代码

### Core Animation

* Color Blended Layers - 这个选项基于渲染程度对屏幕中的混合区域进行绿到红的高亮（也就是多个半透明图层的叠加）。由于重绘的原因，混合对GPU性能会有影响，同时也是滑动或者动画帧率下降的罪魁祸首之一。

* ColorHitsGreenandMissesRed - 当使用shouldRasterizep属性的时候，耗时的图层绘制会被缓存，然后当做一个简单的扁平图片呈现。当缓存再生的时候这个选项就用红色对栅格化图层进行了高亮。如果缓存频繁再生的话，就意味着栅格化可能会有负面的性能影响了（更多关于使用shouldRasterize的细节见第15章“图层性能”）。

* Color Copied Images - 有时候寄宿图片的生成意味着Core Animation被强制生成一些图片，然后发送到渲染服务器，而不是简单的指向原始指针。这个选项把这些图片渲染成蓝色。复制图片对内存和CPU使用来说都是一项非常昂贵的操作，所以应该尽可能的避免。

* Color Immediately - 通常Core Animation Instruments以每毫秒10次的频率更新图层调试颜色。对某些效果来说，这显然太慢了。这个选项就可以用来设置每帧都更新（可能会影响到渲染性能，而且会导致帧率测量不准，所以不要一直都设置它）。

* Color Misaligned Images - 这里会高亮那些被缩放或者拉伸以及没有正确对齐到像素边界的图片（也就是非整型坐标）。这些中的大多数通常都会导致图片的不正常缩放，如果把一张大图当缩略图显示，或者不正确地模糊图像，那么这个选项将会帮你识别出问题所在。

* Color Offscreen-Rendered Yellow - 这里会把那些需要离屏渲染的图层高亮成黄色。这些图层很可能需要用shadowPath或者shouldRasterize来优化。

* Color OpenGL Fast Path Blue - 这个选项会对任何直接使用OpenGL绘制的图层进行高亮。如果仅仅使用UIKit或者Core Animation的API，那么不会有任何效果。如果使用GLKView或者CAEAGLLayer，那如果不显示蓝色块的话就意味着你正在强制CPU渲染额外的纹理，而不是绘制到屏幕。

* Flash Updated Regions - 这个选项会对重绘的内容高亮成黄色（也就是任何在软件层面使用Core Graphics绘制的图层）。这种绘图的速度很慢。如果频繁发生这种情况的话，这意味着有一个隐藏的bug或者说通过增加缓存或者使用替代方案会有提升性能的空间。

### OpenGL-ES

* Renderer Utilization - 如果这个值超过了~50%，就意味着你的动画可能对帧率有所限制，很可能因为离屏渲染或者是重绘导致的过度混合。

* Tiler Utilization - 如果这个值超过了~50%，就意味着你的动画可能限制于几何结构方面，也就是在屏幕上有太多的图层占用了。

## UIImageView优化流程

[iOS图片性能优化-TBImageView](http://rdc.taobao.org/?p=1437)
异步下载->强制解码、软件绘制效果并保存到本地-》缓存策略

## UIScrollView优化流程

[UIScrollView 实践经验](http://tech.glowing.com/cn/practice-in-uiscrollview/)

滚动的时候，加载已经下载的图片；不是滚动的 时候，下载并加载图片。

## UITableView优化流程

主要是有对象回收池。

## SDWebImage源码分析

# 更多

<http://ciechanowski.me/blog/2014/01/05/exploring_gpgpu_on_ios>
<https://github.com/100mango/zen/blob/master/WWDC%E5%BF%83%E5%BE%97%EF%BC%9AAdvanced%20Graphics%20and%20Animations%20for%20iOS%20Apps/Advanced%20Graphics%20and%20Animations%20for%20iOS%20Apps.md>
