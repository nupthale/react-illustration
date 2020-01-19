这两个方法是Render过程中非常重要的两个方法, 并且都有一大块的switchCase，下面以HostComponent为例，说明这两个方法大概做了什么事情：

## beginWork

![](https://img.alicdn.com/tfs/TB1GE3uueL2gK0jSZPhXXahvXXa-479-617.jpg)

这个方法里有两大块switchCase，其中上面处理Context忽略，下面的根据WIP的tag做判断处理，以HostComponent为例：

![](https://img.alicdn.com/tfs/TB1_mIAueH2gK0jSZJnXXaT1FXa-460-51.jpg)

可以看到只是调用了updateHostComponent方法， 这个方法；

#### updateHostComponent

![](https://img.alicdn.com/tfs/TB1hJZxueH2gK0jSZFEXXcqMpXa-666-306.jpg)

里面只是判断props是否有变化， 没变化什么都不做， 变化了生成updateQueue，然后markUpdate，至于update这里不讨论；

## completeWork(current, workInProgress, renderExpirationTime)

* current - wip的alternate
* workInProgress
* renderExpirationTime - 忽略

![](https://img.alicdn.com/tfs/TB1uCgruaL7gK0jSZFBXXXZZpXa-408-630.jpg)

可以看到completeWork就是根据fiber.tag做不同的处理，这里我们选一个复杂度适中的HostComponent进行了解：
一样先忽略Context和ExpirationTime；

![](https://img.alicdn.com/tfs/TB1uZZvuhD1gK0jSZFsXXbldVXa-475-605.jpg)

从上面的核心过程看出来， 我们只需要关注createInstance, appendAllChildren, updateHostComponent做了什么即可；updateHostComponent上面介绍过，此处跳过；

#### createInstance

![](https://img.alicdn.com/tfs/TB1uBsvulv0gK0jSZKbXXbK2FXa-710-325.jpg)

上面括号是___DEV__忽略，其中createElement, 内部会调用document.createElement创建domElement并返回；
precacheFiberNode和updateFiberProps都是在这个domElement加了__reactInternalInstance$和__reactEventHandlers$属性；加属性做什么，暂时忽略；

#### appendAllChildren

![](https://img.alicdn.com/tfs/TB1zmAuuoz1gK0jSZLeXXb9kVXa-670-464.jpg)

比较好理解，重点每次返回sibling一直到没有sibling，找到HostComponent或者HostText就把自己的stateNode append到parent上去；所以最外层的App的stateNode是一个最完整的全局DOM；其中还一个循环就是sibling找不到的时候，如果已经走到头了或者走到了WIP，就什么都不做，否则会走node return；

## 总结

1. 可以看到beginWork重点是标记更新

2. completeWork的重点是生成stateNode