# React初印象

以下是第一遍大概过React代码的一个记录，中间只记录了一些数据变化（总结还在整理），适用于刚开始看代码的人，对React有更进一步的印象；文章包括两部分：《第一次Render过程》与《一次Update过程》，这两个是最基本的两个过程，以下内容可能本身并不解决任何问题，但是它们是比较重要的一个框架，在了解了框架后，里面的内容就比较容易补充了；

假设有如下代码：

![代码示例](https://img.alicdn.com/tfs/TB16yh.t.z1gK0jSZLeXXb9kVXa-414-610.jpg)

## 记React第一次Render过程

首先看这段代码生成的FiberTree是什么样子的：

* 不同颜色表示不同的tag字段
* 圈中的文字表示type字段

从代码中可以看到，目前我们有以下几种类型的节点：

* 首先FunctionComponent  - App，Parent, Child， 它们的type字段是对应的函数
* 然后内部的HostComponent - Div， P，他们的type字段是它们的标签名，如"div"
* 内部还有HostText - 比如parent's lastname is，它的type字段是null
* 看不到的有Host Root和Fiber Root两个节点，这个是内部实现；
* 在React执行过程中，还未确定是什么类型的组件时，会存在一个中间状态叫IndeterminateComponent，比如在App创建时，它是从Indeterminate状态转为FunctionComponent状态的
	


## 记React的一次Update过程


