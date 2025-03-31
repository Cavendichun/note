# React

## 1. 为什么react虚拟dom树要采用逐步渲染的方式？

react的标准数据流是单项的，即：父state -> 子prop -> 孙prop，并且react是一个纯runtime的库，没有编译优化，所以每个fiber节点的具体状态，必须等到渲染的时候才能明确知道，也就是说，必须先把父级的状态计算好，才能知道子级能得到什么。

举个例子：比如在组件的更新渲染阶段，用户触发了useState的dispatch函数，这个dispatch函数会把action传进一个updateQueue中，通过scheduleUpdateOnFiber进行调度统一执行。也就是说，用户触发了dispatch之后，状态并没有更新，而是scheduleUpdateOnFiber触发了组件的重新渲染，进入renderWithHooks阶段后，才拿到了最新状态，这时才能确定他的下一层子级接受了他的props的组件是否需要渲染、销毁、更新。


## 2. react和vue的本质区别？

vue是有编译优化的，是响应式的，响应式的本质是通过Object.defineProperties或Proxy实现了数据与函数/视图的绑定（用户修改数据 -> vue劫持数据 -> 修改视图/运行函数），依赖关系在编译的时候就已经确定

react是计算驱动的，是纯运行时的，本质是纯函数，视图与数据不绑定，需要用户手动触发dispatch，react才能开始协调并触发视图更新，用源码上来说就是：用户触发dispatch -> 生成update -> 加入updateQueue -> scheduleUpdateOnFiber 完成一次视图更新

对比的话，两种模式在心智模型上是不同的，vue的哲学是: 视图 <==> 数据，react的哲学是：视图 = F（数据），vue的自动跟踪功能，上手简单；react的函数式带来了更好的逻辑组合能力
