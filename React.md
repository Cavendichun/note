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

- 用户触发更新，dispatch 计算完成后会生成一个新的 update，通过 enqueueUpdate 将其添加到对应 hook 的 queue 队列中，然后调用 scheduleUpdateOnFiber 等待调度

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

    - 先判断当前的 fiber.alternate 是否存在，如果不存在，说明这个节点是新创建的，走 mountChildFibers 流程，生成子节点，不打 Flag

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
  - 把新增的 fiber 节点打 Placement 标记（这个新增指的是父 fiber 不存在的那种；父 fiber 存在的情况在 beginWork 阶段处理过了）

  - 将每个 fiber 的 Flag 冒泡到父 fiber 的 subtreeFlags 上，这样 commit 阶段就可以根据这个标识选择是否跳过整棵子树的遍历

- 先向下递，再向上归，最终回到 rootFiberNode，得到一棵新的 fiber 树，此时存在两棵树 current 树和 finishedWork 树

diff 阶段采取的性能优化措施如下：

- 采用链表形式，保存当前处理到的 fiber 节点的指针，让 diff 过程有了暂停，恢复的能力

- finishedWork 树在构建好之前叫 workInProgress 树，也就是说，diff 过程中同时存在 current 和 wip 两棵树，如果 diff 过程被中止了，因为 current 完全没有被修改，所以页面不会出现中间状态

- beginWork 阶段判断 key、type、props 如果相等，果断复用子树，跳过后续对比

- compeleteWork 阶段的 subtreeFlags 为 commit 阶段的性能优化做了准备，如果某个 fiber 的 flags 和 subtreeFlags 都为 NoFlag 的话，果断跳过子树遍历；如果某个节点是 Placement 的，证明整棵子树都是新的，就只对根做一次插入操作就可以了

## 4. Hooks 的实现原理

每个 fiber 的数据结构上都一个 memoizedState 字段，在类组件中，该字段存储的是状态对象；在函数组件中，该字段存储的是组件内使用的所有的 hook。

hook 的存储形式是一条单向链表，链表的顺序就是组件内 hook 的调用顺序，fiber 的 memoziedState 字段指向这个链表的头，因为只储存了链表头的地址，所以节省比类组件节省内存占用。

又因为是一条链表，每次渲染的时候是按顺序取值的，所以 react 要求不能条件调用 hook，不然数量或顺序就会和上一次不同，就会导致取值错乱。

每个 hook 通过闭包的形式，保存了当前的 fiber 指向 currentlyRenderingFiber，例如 useState 的 dispatch 是可以赋值给一个全局变量在组件外部使用的，因为 currentlyRenderingFiber 保持了 fiber 的指向；所以 react 不允许在组件外初始化 hooks，因为无法绑定 currentlyRenderingFiber

以最常见的 useState 举例，hook 的数据结构上有几个关键字段：

- initialState：初始化 hook 的值

- baseState: update 过程中的临时值

- memoizedState: 所有 update 执行完成后的最终值

- updateQueue：dispatch 的更新队列

解释一下以上名词：

- 组件 mount 的时候，调用 useState(action)，initialState 赋值为 action，如果 action 是函数，将函数的执行结果赋值到 memoziedState，否则将 action 直接赋值给 memoziedState

- 用户触发 dispatch 更新，可能会连续触发很多次，每次触发 dispatch 后：

  - dispatch 内部会生成一个 update
  - 通过 enqueueUpdate 加入到当前 hook 的 updateQueue.pending 中（queue.pending 是一个环形链表，pending 指向链表尾部，链表尾部指向头部，这样每次添加一个 update 的时候，就不需要找到链表的最后一个了，直接在 pending 指针添加，在修改下 next 指向就可以了）
  - 通知 scheduleUpdateOnFiber 等待调度
  - 更新开始后，hook 会依次执行 updateQueue 中的 update，每次的执行结果保存在 baseState 中，上一个 update 的结果，是下一个 update 的入参，都执行完成后，清空 updateQueue，把最终结果赋值给 memoizedState

从以上的描述可以看出，hook 的性能好，主要体现在以下两个方面：

- 存储空间小，碎片化的 hook 通过指针连接起来，memoziedState 只保存链表头，让函数组件比类组件轻量许多

- baseState 和 memoizedState 的配合让状态的更新变得可暂停、可中断，契合 fiber 设计理念（在 update 按顺序执行过程中，memoziedState 不会发生变化，baseState 记录中间值）

## 5. setState/dispatch 是同步还是异步的?

首先，setState/dispatch 不能理解成传统意义上的同步异步，但从表现上看确实类似异步，原因如下：

- 调用每次 dispatch 会生成一个 update 加入 hook 的 updateQueue 中，这个 update 本身是同步的

- 因为 react 是调度特点是并发批处理的，（并发类似 js 的事件循环，不同优先级之间切换）调用 scheduleUpdateOnFiber 后不会立即进行 reconcile，所以看起来像是异步

## 6. 为什么调用 useState 的 dispatch 后，立刻打印 state，得到的还是上一次的值？

因为调用 dispatch 只是将一个 update 加入了 hook 的 updateQueue 中，还没有立即执行，需要等待调度，下一次 renderWithHook 的时候，才会计算出最新值，在这之前，memoizedState 都是旧值。

## 7. 为什么更新渲染阶段，useState 的值不会被还原？

react 执行 useState 的时候会判断组件是不是初次渲染，判断的依据是 fiber.alternate 是否为空。组件初次渲染和更新渲染，useState 看似调用了同一个方法，实际在内部是两套不同的实现：

初次渲染的时候，useState 做了这些事：

- 调用mountState

- 判断 initial 是否为函数，如果函数，memoziedState 赋值为运行结果，否则赋值为 action 的值

- 创建一个 dispatch 方法，用闭包的形式保存了当前的 fiber 和 hook

- 返回[memoizedState, dispatch]

用户触发了更新后，hook 的 updateQueue 中会有 update，下一次更新渲染的时候，做了这些事：

- 调用updateState

- 取出 hook 的 updateQueue，通过 processUpdateQueue，用 memoizedState 作为初始值，计算出最终结果，赋值给 memoizedState

- 返回[memoizedState, dispatch]

## 有哪几个生命周期被标记为不安全？为什么？有什么替代方案吗？

一共有三个生命周期被标记为不安全，分别是：

- componentWillMount

- componentWillUpdate

- componentWillReceiveProps

他们的执行时间分别是：

- componentWillMount：reconciler 阶段，打 Placement Flag 的时候

- componentWillUpdate：reconciler 阶段，打 Update Flag 的时候

- componentWillReceiveProps：reconciler 阶段，reconcileChidFibers 的更新 props 的时候

因为并发渲染的原因，导致这三个生命周期可能会被反复的执行，不符合 fiber 架构的逻辑
