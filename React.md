# React

## 1. 为什么 react 虚拟 dom 树要采用逐步渲染的方式？

react 的标准数据流是单项的，即：父 state -> 子 prop -> 孙 prop，并且 react 是一个纯 runtime 的库，没有编译优化，所以每个 fiber 节点的具体状态，必须等到渲染的时候才能明确知道，也就是说，必须先把父级的状态计算好，才能知道子级能得到什么。

举个例子：比如在组件的更新渲染阶段，用户触发了 useState 的 dispatch 函数，这个 dispatch 函数会把 action 传进一个 updateQueue 中，通过 scheduleUpdateOnFiber 进行调度统一执行。也就是说，用户触发了 dispatch 之后，状态并没有更新，而是 scheduleUpdateOnFiber 触发了组件的重新渲染，进入 renderWithHooks 阶段后，才拿到了最新状态，这时才能确定他的下一层子级接受了他的 props 的组件是否需要渲染、销毁、更新。

## 2. react 和 vue 的本质区别？

vue 是有编译优化的，是响应式的，响应式的本质是通过 Object.defineProperties 或 Proxy 实现了数据与函数/视图的绑定（用户修改数据 -> vue 劫持数据 -> 修改视图/运行函数），依赖关系在编译的时候就已经确定

react 是计算驱动的，是纯运行时的，本质是纯函数，视图与数据不绑定，需要用户手动触发 dispatch，react 才能开始协调并触发视图更新，用源码上来说就是：用户触发 dispatch -> 生成 update -> 加入 updateQueue -> scheduleUpdateOnFiber 完成一次视图更新

对比的话，两种模式在心智模型上是不同的，vue 的哲学是: 视图 <==> 数据，react 的哲学是：视图 = F（数据），vue 的自动跟踪功能，上手简单；react 的函数式带来了更好的逻辑组合能力

## 3. react 的 diff 比对原理？

以更新渲染为例，当用户通过某种调用了 dispatch 后，react 会经历：

- schedule（调度阶段）
- reconcile（调和阶段）
- commit（提交阶段）

diff 发生在 reconcile 阶段，具体过程如下

- 用户触发更新，dispatch 会生成一个新的 update，通过 enqueueUpdate 将其添加到对应 hook 的 queue 队列中，然后调用 scheduleUpdateOnFiber 等待调度

- scheduleUpdateOnFiber 通过一些列优先级判断后，开始更新流程

- 先从触发更新的节点一路向上回溯到根节点 rootFiberNode，回溯的目的是确保所有的更新都能被正确处理（比如 context）

- 从根节点开始，进入 workLoop 阶段，workLoop 分为两个步骤：

  - beginWork： 向下递，创建或复用某个 fiber 的子节点，打副作用标记

  - completeWork： 向上归，生成真实 dom 保存在 stateNode 中，并向上收集副作用

- beginWork：

  - 根据 fiber 的 tag 调用不同的渲染函数，得到下一层的子节点 children

    - tag 是节点的类型，比如 ClassComponent，FunctionComponent，HostComponent，HostText 等

    - 不同的渲染函数：比如 ClassComponent 调用 instance.render，FunctionComponent 调用 renderWithHooks

  - 把 children 传入 reconcileChildren 中，开始 diff 过程：

    - 先判断当前的 fiber.alternate 是否存在，如果不存在，说明这个节点是新创建的，走 mountChildFibers 流程，生成子节点，不打Flag

    - 如果 fiber.alternate 存在，证明是节点的 update 过程，走 reconcileChildFibers 流程，判断是否复用子 fiber，判断如下：

      - 判断所有的子 fiber 的 alternate 是否存在，如果不存在，证明是新建，打 Placement Flag

      - 如果子 fiber 的 alternate 存在，尝试复用，判断如下：

        - 判断 key 和 type 是否一致（key 是用户定义的唯一 key，type 是具体的组件类型，比如 tag 是 HostComponent 时，type 为 div，p 等， tag 是 FunctionComponent 时，type 为函数的引用...）

        - 如果 key 和 type 有一个不一致，把旧的 fiber 加入 deletions 数组，给父级打 DeleteChildren Flag，创建新的 fiber 打 Placement Flag

        - 如果 key 和 type 都一致，则复用 fiber，props 不同的话，修改一下 props，打 Update Flag

      - 子 fiber 的 diff 完成，给所有子 fiber 绑定 return 指针，指向父 fiber；父 fiber 的 child 指针指向第一个子 fiber，每个子 fiber 的 sibling 指针指向自己右边的兄弟 fiber

      - 指针绑定完成，返回 child

  - reconcileChildren 完成，把 child 指针返回给 workLoop

- completeWork：

  - 如果当前 fiber 有 sibling，就让 sibling 进入 workLoop 流程

  - 根据每个 fiber 节点不同的 Tag 类型，生成、修改、复用之前的 dom，并保存在 stateNode 中，不做真实的 domApi 操作
    
  - 把新增的fiber节点打Placement标记（这个新增指的是父fiber不存在的那种；父fiber存在的情况在beginWork阶段处理过了）

  - 将每个 fiber 的 Flag 冒泡到父 fiber 的 subtreeFlags 上，这样 commit 阶段就可以根据这个标识选择是否跳过整棵子树的遍历

- 先向下递，再向上归，最终回到 rootFiberNode，得到一棵新的 fiber 树，此时存在两棵树 current 树和 finishedWork 树

diff 阶段采取的性能优化措施如下：

- 采用链表形式，保存当前处理到的 fiber 节点的指针，让 diff 过程有了暂停，恢复的能力

- finishedWork 树在构建好之前叫 workInProgress 树，也就是说，diff 过程中同时存在 current 和 wip 两棵树，如果 diff 过程被中止了，因为 current 完全没有被修改，所以页面不会出现中间状态

- beginWork 阶段判断 key、type、props 如果相等，果断复用子树，跳过后续对比

- compeleteWork 阶段的 subtreeFlags 为 commit 阶段的性能优化做了准备，如果某个 fiber 的 flags 和 subtreeFlags 都为 NoFlag 的话，果断跳过子树遍历；如果某个节点是Placement的，证明整棵子树都是新的，就只对根做一次插入操作就可以了

## 4. Hooks的实现原理

每个fiber的数据结构上都一个memoizedState字段，在类组件中，该字段存储的是状态对象；在函数组件中，该字段存储的是组件内使用的所有的hook。

hook的存储形式是一条单项链表，链表的顺序就是组件内hook的调用顺序，fiber的memoziedState字段指向这个链表的头，因为只储存了链表头的地址，所以节省比类组件节省内存占用。

又因为是一条链表，每次渲染的时候是按顺序取值的，所以react要求不能条件调用hook，不然数量或顺序就会和上一次不同，就会导致取值错乱。

每个hook通过闭包的形式，保存了当前的fiber指向currentlyRenderingFiber，例如useState的dispatch是可以赋值给一个全局变量在组件外部使用的，因为currentlyRenderingFiber保持了fiber的指向；所以react不允许在组件外初始化hooks，因为无法绑定currentlyRenderingFiber

以最常见的useState举例，hook的数据结构上有几个关键字段：

- initialState：初始化hook的值

- baseState: 上一次hook状态更新后的值

- memoizedState: 更新过程中的状态，有可能是中间态

- queue：dispatch的更新队列

解释一下以上名词：

- 组件mount的时候，调用useState(action)，initialState赋值为action，如果action是函数，将函数的执行结果赋值到baseState，否则将action直接赋值给baseState

- 用户触发dispatch更新，可能会连续触发很多次，每次触发dispatch后：

  - dispatch内部会生成一个update
    
  - 通过enqueueUpdate加入到当前hook的queue.pending中（queue.pending是一个环形链表，pending指向链表尾部，链表尾部指向头部，这样每次添加一个update的时候，就不需要找到链表的最后一个了，直接在pending指针添加，在修改下next指向就可以了）
    
  - 通知scheduleUpdateOnFiber等待调度
    
  - 更新开始后，hook会依次执行queue.pending中的update，每次的执行结果保存在memoizedState中，上一个update的结果，是下一个update的入参，都执行完成后，清空queue.pending，把memoziedState赋值给baseState
 
从以上的描述可以看出，hook的性能好，主要体现在以下两个方面：

- 存储空间小，碎片化的hook通过指针连接起来，memoziedState只保存链表头，让函数组件比类组件轻量许多

- baseState和memoizedState的配合让状态的更新变得可暂停、可中断，契合fiber设计理念（在update按顺序执行过程中，baseState不会发生变化，memoizedState记录中间值）

## 5. setState/dispatch是同步还是异步的?

首先，setState/dispatch不能理解成传统意义上的同步异步，但从表现上看确实类似异步，原因如下：

- 调用每次dispatch会生成一个update，这个update本身是同步的

- 因为react是调度特点是并发批处理的（并发原理是快速在不同优先级之间切换，类似js的事件循环，批处理原理是scheduleUpdateOnFiber会收集一个时间切片里的所有update，安排一次渲染），所以看起来像没有立即执行一样

## 有哪几个生命周期被标记为不安全？为什么？有什么替代方案吗？

一共有三个生命周期被标记为不安全，分别是：

- componentWillMount
 
- componentWillUpdate
 
- componentWillReceiveProps
 
他们的执行时间分别是：

- componentWillMount：reconciler阶段，打Placement Flag的时候

- componentWillUpdate：reconciler阶段，打Update Flag的时候

- componentWillReceiveProps：reconciler阶段，reconcileChidFibers的更新props的时候

因为并发渲染的原因，导致这三个生命周期可能会被反复的执行，不符合fiber架构的逻辑


