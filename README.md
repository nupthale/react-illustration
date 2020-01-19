# 图说React

以下是第一遍大概过React代码的一个记录，中间只记录了一些数据变化（总结还在整理），适用于刚开始看代码的人，对React有更进一步的印象；文章包括两部分：《第一次Render过程》与《一次Update过程》，这两个是最基本的两个过程，以下内容可能本身并不解决任何问题，但是它们是比较重要的一个框架，在了解了框架后，里面的内容就比较容易补充了；

更多其他细节可参考：

* [以HostComponent说明beginWork与completeWork](https://github.com/nupthale/react-illustration/blob/master/beginWork%26completeWork.md)
* [Why setState之后不会更新](https://github.com/nupthale/react-illustration/blob/master/setState.md)
* [React ExpirationTime](https://github.com/nupthale/react-illustration/blob/master/expirationTime.md)

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

下面分针对RenderPhase与CommitPhase分别介绍；

###### Render Phase

先看下workLoop函数里面是什么, 就是一个while循环，结合workInProgress，很容易联系到它会遍历一遍FiberTree；直到workInProgress为null；

![](https://img.alicdn.com/tfs/TB1U.9ot7P2gK0jSZPxXXacQpXa-490-84.jpg)

这个performUnitOfWork内部也是有两个分叉，如果workInProgress.child不为null就一直beginWork，如果为null了就会执行一次completeUnitOfWork，

![](https://img.alicdn.com/tfs/TB1WQqot8v0gK0jSZKbXXbK2FXa-472-614.jpg)

可能看到这个图还是没什么感觉，那么再用快照的方法看下performUnitOfWork和beginWork做了什么事情：下面的图是在performUnitOfWork最后处断点的快照；从上面的状态继续，当前状态如下：

![](https://img.alicdn.com/tfs/TB17aGot1L2gK0jSZPhXXahvXXa-1174-498.jpg)

当执行了第一轮performUnitOfWOrk&beginWork后：

![](https://img.alicdn.com/tfs/TB1dv5ntVT7gK0jSZFpXXaTkpXa-1121-429.jpg)

第二轮performUnitOfWork&beginWork后：

![](https://img.alicdn.com/tfs/TB11g1tt7T2gK0jSZFkXXcIQFXa-1144-640.jpg)

剩余全部performUnitOfWork&beginWork执行后， 我们可以发现它的执行顺序如下：

![](https://img.alicdn.com/tfs/TB1AJKttYj1gK0jSZFOXXc7GpXa-760-622.jpg)

最后提一下，FiberTree的结构并不是二叉树，每个节点就3个分支，一个child，一个sibling，一个return（省略）；所以整体如下图：

![](https://img.alicdn.com/tfs/TB1sjent1T2gK0jSZFvXXXnFXXa-426-295.jpg)

上面看了performUnitOfWork和beginWork，但是在workLoop中我们知道其实这个过程中还执行了completeUnitOfWork&completeWork，它们在workLoop中是交织在一起的，
结合workLoop和上面介绍了performUnitOfWork, 我们知道每当沿着child走到头了，也就是Next= null的时候，就要执行completeUnitOfWork，当本次completeUnitOfWork执行结束后
就又回到performUnitOfWork&beginWork；关系如下图：

![](https://img.alicdn.com/tfs/TB1fIert7L0gK0jSZFxXXXWHVXa-734-1070.jpg)

那么completeUnitOfWork做了什么，如果将断点在completeUnitOfWork的doWhile里的尾部，那么会得到如下的一系列快照：

第一次completeUnitOfWork

![](https://img.alicdn.com/tfs/TB1ieStt4D1gK0jSZFyXXciOVXa-756-402.jpg)

第二次completeUnitOfWork

![](https://img.alicdn.com/tfs/TB1eiGqt1H2gK0jSZFEXXcqMpXa-734-362.jpg)

第三次以及后续completeUnitOfWork，从步骤7开始

![](https://img.alicdn.com/tfs/TB1ibettYY1gK0jSZTEXXXDQVXa-741-436.jpg)

Render阶段至此结束，总结下：

这里其实具体代码是怎么调用的，我们并不是很关心，而是想了解经过render，我们得到了什么结果（输出/变化）；

经过第一遍整理，简单的列了比较明显的几个变化（目前不全面）：

1. 生成了完整的FiberTree
2. 确定了updateQueue

![](https://img.alicdn.com/tfs/TB1_rett.Y1gK0jSZFCXXcwqXXa-823-367.jpg)

3. 确定了每个fiber的stateNode，如下图：

![](https://img.alicdn.com/tfs/TB1KtqptVP7gK0jSZFjXXc5aXXa-568-231.jpg)

###### Commit Phase

当执行完上面的过程后， 就会调用commitRoot方法，进入Commit Phase；在开始时FiberTree的状态如下：

![](https://img.alicdn.com/tfs/TB1fF5tt7T2gK0jSZPcXXcKkpXa-631-314.jpg)

CommitPhase内部又分为以下几个Phase：

![](https://img.alicdn.com/tfs/TB12Wytt7T2gK0jSZPcXXcKkpXa-815-530.jpg)

以我们的例子说明，我们会进入commitMutationEffects，在这个方法中，会开始找HostComponent和HostText，做DOM操作，其中DOM操作调用的方法都是从HostConfig中传入的，

![](https://img.alicdn.com/tfs/TB1sLattYY1gK0jSZTEXXXDQVXa-645-495.jpg)

经过上面的过程，完成了一次完整的render，其中只是重点关注了数据的变化，至于优先级、调度、Suspend、Context、Hook等都直接跳过，后续针对这些具体问题可以再重点关注这些内容；


## 一次Update过程

还是上面的例子，介绍当用户点击了按钮Click Me的时候的执行逻辑

![代码示例](https://img.alicdn.com/tfs/TB16yh.t.z1gK0jSZLeXXb9kVXa-414-610.jpg)

##### dispatchDiscreteEvent

当点击按钮的时候， 触发更新,   这个过程并不是简单地调用onClick，然后setName这么简单，这个过程的入口也不在onClick方法上，而是与react内部实现的event机制有关系（目前先忽略这个机制）；我们需要了解的是当用户点击button时，会调用dispatchDiscreteEvent，为什么会调用？这里也大概说一下，是因为在Render过程中，React对内部的element绑定了事件，它会分别调用以下方法，最终当element点击时的回调会设置未dispatchDiscreteEvent，大体如下图：

![](https://img.alicdn.com/tfs/TB133_5t7T2gK0jSZFkXXcIQFXa-483-193.jpg)

当调用dispatchDiscreteEvent时，里面会调用dispatch， 触发onClick，执行setName方法，然后还会调用flushSyncCallbackQueue，这两个步骤是相互关联的，我们先看这个setName做了什么，然后再看flushSyncCallbackQueue；


##### dispatchAction(fiber, queue, action)

如果你在setName处断点，那么进入方法后， 你会发现其实你调用的是dispatch方法，并且有如下代码：

![](https://img.alicdn.com/tfs/TB1Itn0t4D1gK0jSZFsXXbldVXa-555-267.jpg)

可以看到通过在调用setName的时候，传入的参数会作为第三个参数，也就是action传入(前两个参数分别是
currentRenderingFiber$1和queue)，并且通过mountWorkInProgessHook中的实现可以看到，它们的关系如下：

currentRenderingFiber$1.memorizedState = hook

Hook.queue = 上面创建的queue对象

连起来就是:  currentRenderingFiber$1.memorizedState.queue = 上图的queue；结合我们的例子，这里的currentRenderingFiber$1其实就是App节点，因为onClick是在App组件内调用的；

当调用dispatchAction时，会做下面几件事情：

1. 创建一个update对象, 其中action和eagerState都为setName设置的值

![](https://img.alicdn.com/tfs/TB1DEv4tYY1gK0jSZTEXXXDQVXa-391-180.jpg)

2. 更新currentRenderingFiber$1.memorizedState.queue.last为update对象

3. 更新update对象的eagerReducer和eagerState的值，最终结构如下：

![](https://img.alicdn.com/tfs/TB1pLr1t4n1gK0jSZKPXXXvUXXa-684-521.jpg)

4. 判断eagerState和currentState是否相同，如果相同就证明不需要更新，返回什么都不做；如果不相同，那么就调用scheduleWork调度任务；

下面看下在更新时scheduleWork都做了什么

##### scheduleWork(fiber, expirationTime)

我们的例子传入的fiber是App，从App开始，分别会调用如下方法：

1. 当时第一次渲染是满足LegacyUnbatcheCtx | RenderCtx | CommitCtx的，所以调用了performSyncWorkOnRoot，此时走了不同的路径，这个路径就是ensureRootIsScheduled，从代码里可以看到第一次会调用performSyncWorkOnRoot，其他的都会先调用的ensureRootIsScheduled方法，在方法内部，将performSyncWorkOnRoot放到schedule中的队列中
	
2. 每次调用都只执行一个task，当一个task执行完后，会再次调用ensureRootIsScheduled，再执行下一个task, 这里是个递归调用；

我们发现ensureRootIsScheduled是另外一个很重要的方法；
	
![](https://img.alicdn.com/tfs/TB1sfb4t4v1gK0jSZFFXXb0sXXa-669-420.jpg)

##### ensureRootIsScheduled

因为我们是通过Click点击的，Click事件的update的优先级是Sync级别，所以会接着调用scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root))，这个scheduleSyncCallback做两件事情：

1. 把performSyncWorkOnRoot放到syncQueue数组里

2. 通过Scheduler_scheduleCallback把flushSyncCallbackQueueImpl放到Scheduler的taskQueue中；

如下图：（先忽略那个WIP是否是null的判断， 这个判断的意思是在执行performSyncWorkOnRoot时，如果WIP不是null的话， 还会再次调用ensureRootIsScheduled方法）

![](https://img.alicdn.com/tfs/TB1ogf6t1H2gK0jSZJnXXaT1FXa-700-391.jpg)

目前看到的只是把task放到了queue里面，什么时候执行？其实在第二步放入scheduler的时候，在最后会调用
requestHostCallback方法，这个方法会使用messageChannel的postMessage方法，这个网上很多解释，暂且理解为requestIdleCallback即可，也就是说postMessage的回调performWorkUntilDeadline会在下一个有Idle空闲时间的帧时执行；如下图：

![](https://img.alicdn.com/tfs/TB1gHr5t7T2gK0jSZPcXXcKkpXa-710-307.jpg)

其中scheduledHostCallback其实是Scheduler内部的flushWork方法，这个方法会调用Scheduler的workLoop，这个workLoop会从taskQueue这个优先级队列中每次取出一个task执行，直到没有剩余时间或者调用shouldYieldToHost，将执行权让步给Host，就会break循环，并返回是否hasMoreWork；

上面说了Scheduler的workLoop，从上面的queue图片可以看到我们的taskQueue中放入的就是flushSyncCallbackQueueImpl方法，也就是这个workLoop会执行flushSyncCallbackQueueImpl方法，这个方法很清晰：遍历syncQueue，并且调用syncQueue里面的方法， 前面提到过我们的syncQueue中是performSyncWorkOnRoot;

![](https://img.alicdn.com/tfs/TB1s.6otV67gK0jSZPfXXahhFXa-709-521.jpg)

注意这里并不是同步就清空syncQueue，而是等下一次Idle时才执行；到目前为止，我们说了setName的路径的执行过程，再回到discreteUpdates，之前提到过：当调用dispatchDiscreteEvent时，里面会调用dispatch， 触发onClick，执行setName方法，然后还会调用flushSyncCallbackQueue；上面介绍了setName，setName执行过后， 还会同步调用flushSyncCallbackQueue，这个方法在setName过程中也出现过，所以不再重复说明，总结下如图：

![](https://img.alicdn.com/tfs/TB1L3n1t8v0gK0jSZKbXXbK2FXa-536-906.jpg)

最后这里有个疑问就是flushSyncCallbackQueue出现了两次，两次都会执行SyncQueue，一个是同步， 一个是下一次Idle时执行，为什么要这么做不是很理解；

到目前为止，大体框架基本走了一遍，内部很多细节还需要填充下；
