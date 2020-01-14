# React初印象

以下是第一遍大概过React代码的一个记录，中间只记录了一些数据变化（总结还在整理），适用于刚开始看代码的人，对React有更进一步的印象；文章包括两部分：《第一次Render过程》与《一次Update过程》，这两个是最基本的两个过程，以下内容可能本身并不解决任何问题，但是它们是比较重要的一个框架，在了解了框架后，里面的内容就比较容易补充了；

假设有如下代码：

![代码示例](https://img.alicdn.com/tfs/TB16yh.t.z1gK0jSZLeXXb9kVXa-414-610.jpg)

## 第一次Render过程

首先看这段代码生成的FiberTree是什么样子的：

* 不同颜色表示不同的tag字段
* 圈中的文字表示type字段

从代码中可以看到，目前我们有以下几种类型的节点：

* 首先FunctionComponent  - App，Parent, Child， 它们的type字段是对应的函数
* 然后内部的HostComponent - Div， P，他们的type字段是它们的标签名，如"div"
* 内部还有HostText - 比如parent's lastname is，它的type字段是null
* 看不到的有Host Root和Fiber Root两个节点，这个是内部实现；
* 在React执行过程中，还未确定是什么类型的组件时，会存在一个中间状态叫IndeterminateComponent，比如在App创建时，它是从Indeterminate状态转为FunctionComponent状态的
	
![](https://img.alicdn.com/tfs/TB1PZCdt4v1gK0jSZFFXXb0sXXa-620-278.jpg)
![](https://img.alicdn.com/tfs/TB1CMmat4n1gK0jSZKPXXXvUXXa-587-928.jpg)

可以发现它并不是多叉树；那么这个结构是如何生成的？下面讲具体过程

#### 生成FiberTree

##### createContainer

在第一次调用render时，会判断是否containerDOM节点上是否有_reactRootContainer属性，如果有，证明已经渲染过，如果没有，则标识是第一次执行；现在假设我们都是第一次，那么第一次执行的时候， 会执行createContainer方法，创建下面的结构：他们的属性如右图，目前稍微留个印象就可以了，其中FiberRootNode几个现在就用到的属性如下：
	
* Tag
* FiberRootNode.current = HostRoot
* HostRoot.StateNode = FiberRootNode
* containerInfo

FiberNode目前用到的属性如下：

* Tag
* Type
* Return
* Child
* Sibling
* stateNode， 这个stateNode如果是HostRoot，那么它的StateNode是FiberRootNode，如果是下面的函数组件节点，那么stateNode就是null，如果是Host节点，那么它的StateNode就是对应的DOM元素；

![](https://img.alicdn.com/tfs/TB1cgKstYj1gK0jSZFuXXcrHpXa-472-281.jpg)

执行完createContainer后，mount节点的MountNode._reactRootContainer._internalRoot就是FiberRootNode；


##### updateContainer

###### 初步印象

由于是第一次渲染， 就会调用

![](https://img.alicdn.com/tfs/TB11nKntYr1gK0jSZFDXXb9yVXa-443-49.jpg)

先解释unbatchedUpdates做了什么，代码比较简单，可以看到就是设置了EC为LegacyUnBatchedContext，然后执行回调updateContainer，执行完了再把状态还原回去，这种try - finally的结构在代码里非常常见；还有一句非常重要的是，如果还原后，Ec为NoWork，那么就执行flushSyncCallbackQueue；

![](https://img.alicdn.com/tfs/TB1spSnt7L0gK0jSZFtXXXQCXXa-497-203.jpg)

在执行完这部分后，会发现已经生成了完整的FiberTree（见最开始的图），并且界面也显示了结果，那么中间的过程是怎么样的？

![](https://img.alicdn.com/tfs/TB1CuOrt7T2gK0jSZPcXXcKkpXa-261-178.jpg)

关于二进制逻辑运算，这里大概补充一点，这部分与流程无关，可以跳过：

ExecutionContext（简称EC）默认为0，它有以下几种状态：
* NoContext = 0
* BatchedContext = 1
* EventContext = 2
* DiscreteEventContext = 4
* LegacyUnbatchedContext= 8
* RenderContext = 16
* CommitContext = 32

为什么不连续？因为它用二进制的每一位表示是否处于某个状态，如假设现在EC的值是6，那么它就满足:
* ExecutionContext & DiscreteEventContext !== NoWork
* ExecutionContext & EventContext !== NoWork
* ExecutionContext & (DiscreteEventContext  | EventContext) !== NoWork

6只能是4和2，4是DiscreteEventContext ，2是EventContext ，所以在&操作后，他们都不会是0（NoWork）；

可以看到用二进制表示状态，非常简洁，在业务中写代码也可以考虑参考下；另外补充一下，在代码中可能还会看到以下代码，解释如下：
* ~BatchedContext； 取反，每一位0->1, 1->0, 语义: 非BatchedContext
* executionContext &= ~BatchedContext;   语义：设置EC为非BatchedContext
* executionContext |= LegacyUnbatchedContext; 语义：设置EC为LegacyUnbatchedContext


###### 执行过程

再回到createContainer结束后的状态，updateContainer第一步会先创建一个update，结构如下图，其中payload的element是调用render传入的第一个参数的ReactElement；

![](https://img.alicdn.com/tfs/TB125uot9f2gK0jSZFPXXXsopXa-254-203.jpg)

然后会创建一个updateQueue，并将上面的update放入updateQueue，执行完enqueuUpdate后，queue与fiber的关系是：fiber.updateQueue = queue

![](https://img.alicdn.com/tfs/TB1lKyot.Y1gK0jSZFMXXaWcVXa-285-225.jpg)

最后会调用第三个比较重要的函数scheduleWork

##### scheduleWork - 开始调度

在看其他文章的时候，都知道调度分为两个阶段，RenderPhase和CommitPhase，也就是说scheduleWork中的逻辑主要是这两个工作；

RenderPhase：
* 生成完整的FiberTree
* 生成对应FiberNode的stateNode，也即HtmlElement
* 确定updateQueue, 以及EffectList，也就说要更新什么
* 调用对应的生命周期函数(~useEffect)

CommitPhase：
* 根据FiberTree去操作DOM更新视图
* 调用对应的生命周期函数(~useEffect)

scheduleWork(fiber, expirationTime) 接收两个参数，为了简化，先暂时忽略expirationTime的存在；
主要关注fiber的生成与变化；经过上面的逻辑，此时的状态如下：

![](https://img.alicdn.com/tfs/TB1ljeut8r0gK0jSZFnXXbRRXXa-414-229.jpg)

函数内部有下面的代码判断EC，因为上面说过经过unbatchedUpdates函数，会改变EC为LegacyUnbatchedContext，所以会进入performSyncWorkOnRoot

![](https://img.alicdn.com/tfs/TB1Cmqrt.Y1gK0jSZFCXXcwqXXa-637-127.jpg)

断点快照： performSyncWorkOnRoot(root)
此处传入的参数root就是上图的fiberRoot节点，在内部会执行下面的代码：
大体上这个方法做4件事情：
1. prepareFreshStack(root), 执行完这个方法后，会创建一个WIP(Working In Progress)节点，它们的关系如下图：
![](https://img.alicdn.com/tfs/TB15lqrt4v1gK0jSZFFXXb0sXXa-545-197.jpg)
2. 设置EC为RenderContext， executionContext |= RenderContext,  进入Render Phase
3. 进入workLoop, 这个workLoop就是RenderPhase，做的事情上面介绍过；
4. workLoop结束后，调用finishSyncRender， 进度Commit Phase，完成视图更新；




## 一次Update过程


