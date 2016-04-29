`FBRetainCycleDetector`是Facebook新开源的一个项目。配合`FBMemoryProfiler`使用起来也是很方便。当然`FBMemoryProfiler`里面使用到了`FBAllocationTracker`。目前第一版，在测试的过程中也会遇到一些crash，相信经过使用者的修改和作者本人的自测，会越来越完善的。这篇文章的目的主要是对于FBRetainCycleDetector内部实现进行一个介绍，单单只会使用总感觉是远远不够的。

文章会分为几个模块进行介绍：

* [最简单的使用方法](#最简单的使用方法)
* [主要元素类及其辅助类的介绍](#主要元素类及其辅助类的介绍)
* [主要的查找类及其辅助类介绍](#主要的查找类及其辅助类介绍)

<p id="最简单的使用方法">
###最简单的使用方法
最简单的使用方法，不包含Configuration。单纯的去查找一个对象的引用循环

	FBRetainCycleDetector *detector = [[FBRetainCycleDetector alloc] initWithConfiguration:nil];
	[detector addCandiate:myObject];
	//- (NSSet<NSArray<FBObjectiveCGraphElement *> *> *)findRetainCycles;
	NSSet<NSArray<FBObjectiveCGraphElement *> *> *retainCycles = [detector findRetainCycles];
	NSLog(@"%@", retainCycles);
	
这里先简单的说明一下，`findRetainCycles`查询方式所使用到的算法是DFS(深度优先搜索)。

<p id="主要元素类及其辅助类的介绍">
###主要元素类及其辅助类的介绍(`FBObjectiveCGraphElement `)
`FBRetainCycleDetector`所使用到的对象类型是`FBObjectiveCGraphElement`，会在调用函数:`addCandiate`的时候内部进行初始化为该对象类型或者其子类。

####FBObjectiveCGraphElement
`FBObjectiveCGraphElement`是所有用来查找对象类型的基类。所有的查找对象都基于它实现。该类并不需要外部的调用，主要是供内部查询使用。其提供的功能主要是：

* 提供初始化方法封装`object`（即调用`addCandiate`传入的`object`）
* 获取所有该对象所持有对象`- (NSSet *)allRetainedObjects;`。基类`FBObjectiveCGraphElement `所获取的对象类型是通过associated object所持有的对象。  associated object对象的获取是通过Facebook自身的fishhook去hook原先的`objc_setAssociatedObject`和`objc_removeAssociatedObjects`来实现对象的持有标记。
* 提供过滤接口`- (NSSet *)filterObjects:(nullable NSArray *)objects;`，过滤接口主要是与`FBObjectGraphConfiguration`相结合使用，`FBObjectGraphConfiguration`会在下文介绍。
* 以及其它一些helper接口，例如：获取类、类名、地址等等。

####FBObjectGraphConfiguration
这里先介绍一下Configuration，再去介绍`FBObjectiveCGraphElement`的子类。`FBObjectGraphConfiguration `内容很少，其主要提供的是过滤的block类型`FBGraphEdgeFilterBlock`和过滤器的初始化方法：

		- (instancetype)initWithFilterBlocks:(NSArray<FBGraphEdgeFilterBlock> *)filterBlocks
                 shouldInspectTimers:(BOOL)shouldInspectTimers
即传入一个过滤block的数组，该数组会被`FBObjectiveCGraphElement `对象类型在调用`filterObjects `的时候一次调用。`shouldInspectTimers `的作用是是否检查NSTimer。  
接下来看看`FBGraphEdgeFilterBlock `的定义：

	typedef FBGraphEdgeType (^FBGraphEdgeFilterBlock)(FBObjectiveCGraphElement *_Nullable fromObject,
                                                  FBObjectiveCGraphElement *_Nullable toObject);
传入fromObject（传入的对象）和toObject（被持有的对象），根据自己需求对对象进行处理。添加到数组后进行初始化。这里可以举个例子,过滤掉所有以`UINavi `开头的对象：

	FBGraphEdgeFilterBlock filterBlock = ^(FBObjectiveCGraphElement *_Nullable fromObject,
                                               FBObjectiveCGraphElement *_Nullable toObject){
            if (![[fromObject classNameOrNull] hasPrefix:@"UINavi"]) {
                return FBGraphEdgeValid;
            }
            return FBGraphEdgeInvalid;
        };
	FBObjectGraphConfiguration *configuration = [[FBObjectGraphConfiguration alloc]
                                                     initWithFilterBlocks:@[filterBlock]
                                                     shouldInspectTimers:NO];
	FBRetainCycleDetector *detector = [[FBRetainCycleDetector alloc] initWithConfiguration:configuration];
这就是一个包含configuration的初始化过程。

####FBObjectiveCGraphElement相关子类
上面已经说了，FBObjectiveCGraphElement只提供了对associate object的持有查找。因此其它对象的持有查找是通过子类实现的，主要包含：`FBObjectiveCBlock`,`FBObjectiveCObject`,`FBObjectiveCNSCFTimer`

#####FBObjectiveCBlock实现
主要的实现内容是：重写父类方法`allRetainedObjects`,当然也是有调用`[super allRetainedObjects]`。接下来就是对于block的识别和获取引用关系。最后再封装为`FBObjectiveCBlock `对象类型。  

* block识别方法的一些细节：   
用到`FBBlockStrongLayout.h`里面的函数（用C实现）来进行判断是不是block以及获取引用。判断是不是block所用的方法是利用一个空block`^{}`，判断传入的block是不是其子类。当然这里是先转换为Class类型。  
 
* block内部引用关系获取：  
获取引用关系相对就比较发杂一点，先通过block的`size_t`大小创建相同大小的数组类型，其对象类型是`FBBlockStrongRelationDetector`，该对象主要是为了统计block内部内容是否是一个对象，会在release的时候进行标记。接下来对于每一个调用该block的`dispose_helper`,如若调用了release则证明其是对象，否则就是一些普通数据类型。记录其在block内部的位置关系，返回位置关系数组。再通过数组获取每一个对象。

* 最后当然也会调用`filterObjects`过滤掉不需要查找引用循环的对象。

#####FBObjectiveCObject实现
在重写以及调用父类方法与block是一样的。不同的地方在于对于持有对象的获取。。

* FBClassStrongLayout里面的函数辅助获取对象，同样利用到了runtime(`class_copyIvarList`)机制去获取属性列表，并且封装为`FBIvarReference`类型。如果遇到属性是struct类型还需单独进行处理。
* `FBIvarReference`的初始化方法会对不同的`Ivar`进行分类:`FBObjectType`,`FBBlockType`,`FBStructType`,`FBUnknownType`.
* 然后也是会封装成`FBObjectiveCObject `，在未过滤之前，需要判断该object的类型。因为object的有些类型在进行数据处理的时候会造成崩溃，目前fb处理了一部分，但经过测试还是会发生异常的崩溃现象，不过这个现象主要是对于系统的一些对象类型遍历所造成的。暂时没有发现自己创建的对象在查找retain cycle的时候发生崩溃。

#####FBObjectiveCNSCFTimer实现
FBObjectiveCNSCFTimer的实现内容比较少，其主要就是通过runloop去获取`CFRunLoopTimerGetContext`,再对获取到的数据进行处理即可。

<p id="主要的查找类及其辅助类介绍">
###主要的查找类及其辅助类介绍(`FBRetainCycleDetector`)
####FBRetainCycleDetector
* `FBRetainCycleDetector `主要的功能就是查找retain cycle。使用到的算法思想是深度优先搜索（DFS），因此如果在对象量比较大并且查找深度（默认为8）比较深的情况下，会比较慢。一般情况下是在异步线程执行查找。  
* `FBRetainCycleDetector `会对通过方法`addCandidate`所添加的对象都进行DFS，当然查找之前会通过`FBObjectGraphConfiguration`进行过滤。  
其中查找过程对对象会进一步的封装为`FBNodeEnumerator `类型，接下来介绍该类型。

####FBNodeEnumerator
* `FBNodeEnumerator`继承于`NSEnumerator`，`NSEnumerator`可以方便的提供`nextObject`的方法调用，只需在子类中重写该方法即可。  
* `FBNodeEnumerator`中的`nextObject`主要的处理是：通过object去获取`allRetainedObjects`（此方法是`FBObjectiveCGraphElement`提供获取过滤后的持有对象）。再获取第一个对象进行返回。
* 至于深搜的一些数据存储，这里就不进行解释。

###结论
FBRetainCycleDetector目前处于第一版本，因此会有一些bug,但并不会影响正常的使用。虽然查找算法上面有可能会导致比较大的内存消耗（毕竟如果程序够大的话，深搜也是谈不上效率的）。暂时没有对`FBMemoryProfiler `进行描述的原因是，`FBMemoryProfiler `主要还是界面的实现以及与`FBAllocationTracker `功能的结合。 `FBAllocationTracker `的功能比较简单，后面会用一篇小文章来进行概述。

