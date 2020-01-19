对ClassComponen和FunctionComponent分别说明

## ClassComponent

示例如下：

![](https://img.alicdn.com/tfs/TB1uFsAukT2gK0jSZPcXXcKkpXa-476-651.jpg)

#### 介绍示例

首先有一个App的组件，里面两个button，一个调用test方法，一个调用test2方法；

* 在test方法中，setTimeout回调里面先调用this.setState改变name为jeff，然后console.log(this.state.name)， 得到的结果是更新后的结果"jeff"；

* 在test2方法中，直接调用this.setState, 然后输出this.state.name， 得到的结果是更新前的结果"clark"


#### Why?

首先打开chrome开发者工具的performance，看下两个方法的执行栈有什么不一样：

Test方法执行栈如下图

![](https://img.alicdn.com/tfs/TB10h.zueL2gK0jSZFmXXc7iXXa-1766-333.jpg)

Test2方法执行栈如下图

![](https://img.alicdn.com/tfs/TB1hDIxukL0gK0jSZFAXXcA9pXa-890-380.jpg)

这两个图的关键区别在与Component.setState在调用栈中的位置，图1，是在用户点击执行栈执行完毕后，并且TimerFired回调里面立即执行，图二是在用户点击后的执行栈中执行的。到目前为止同样是接着样执行Component.setState但是他们的执行栈不一样，外层的Scope Context自然很可能不同，两种情况都断点在setState语句上，开始调试setState会发现，内部在scheduleUpdateOnFiber方法执行时，内部有一个判断

![](https://img.alicdn.com/tfs/TB13E.zubH1gK0jSZFwXXc7aXXa-430-91.jpg)

如果此时EC是NoContext，那么会调用flushSyncCallbackQueue()，否则不会；断点调试会发现test里面的timeout触发后执行setState时，EC是0， 而test2里面直接执行setState执行到这里时EC是6，所以timeout调用会多执行一个flushSyncCallbackQueue()，同步更新数据，而test2不会，test2会在下一次渲染才会进行更新；

总结下关系, 如下图：

绿色的EC展示了值的变化，可以看到在onClick中出现了分支， 一个setTimeout，执行下setTimeout函数就结束了，而另外一个函数是在点击回调中直接执行了setState，此时EC是6，就造成了两种写法执行路径的不一致；当timer时间到了的时候，timeout中的回调才会开始执行，这时候点击的EC已经全部执行完毕，变成了0（NoWork），所以在执行setState的时候，就会调用scheduleSyncCallbackQueue同步更新数据；

![](https://img.alicdn.com/tfs/TB1kq2YuX67gK0jSZPfXXahhFXa-741-770.jpg)

## FunctionComponent

示例如下：

![](https://img.alicdn.com/tfs/TB1Xycvuoz1gK0jSZLeXXb9kVXa-461-624.jpg)

还是上面的例子，只是改成function component， 如右图, 函数型组件因为没有this的原因，相比上面要简单很多，不需要走流程，每次渲染name都从React内部Hook取一次最新的值，然后这个name就是一个普通的本地变量了，React内部是无法修改到这个name变量的，所以后续无论怎么执行，按照作用域向上取值，一定得到的是最开始得到的值，不管是setTimeout还是不setTimeout都一样；所以左边的例子很容易理解，得到的结果都是clark；
