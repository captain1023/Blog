# fiber的初次渲染与更新中

本篇属于React中的`React fiber的初次创建与更新(中)`
* [X] React启动过程
* [X] React的两大工作循环
* [X] React中的对象
* [X] React fiber的初次创建与更新(上)
* [X] React fiber的初次创建与更新(中)
* [X] React fiber的初次创建与更新(下)
* [X] React fiber的渲染
* [X] React的管理员(reconciler运行循环)
* [X] react的优先级管理(Lane模型)

**React Hook原理**

* [X] React Hook原理

**其他**

* [ ] React的合成事件
* [ ] Context原理
* [ ] diff算法
* [ ] 状态与副作用

## fiber的初次渲染与更新(中)

总结放前面👏:

主要逻辑就是 `workLoop()`调用 `performSyncWorkOnRoot()`, perform函数中会调用 `beginwrok()`和 `completework()`,整个过程是个递归过程

1. beginWork根据 `React Element`创建(更新,删除)所有fiber节点,为了最大程度的复用之前的节点,beginWork调用了 `reconcileChildren()`,著名的diff算法实现
2. `completework()`,回溯阶段,向上移动副作用队列,将所有节点的副作用收集到fiber的根节点上,供commitRoot()渲染阶段消费
3. 副作用队列: 浅薄理解就是在某处定义了一个变量,有使用到这个变量的地方都可以称为副作用(不官方,建议Google查询)

react启动的过程就是创建 `hostContaine`r然后调用 `updateContainer`(legacyRenderSubtreeIntoContainer()函数)

```js
// ... 省略了部分代码
export function updateContainer(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  callback: ?Function,
): Lane {
  // 获取当前时间戳
  const current = container.current;
  const eventTime = requestEventTime();
  // 1. 创建一个优先级变量(车道模型)
  const lane = requestUpdateLane(current);

  // 2. 根据车道优先级, 创建update对象, 并加入fiber.updateQueue.pending队列
  const update = createUpdate(eventTime, lane);
  update.payload = { element };
  callback = callback === undefined ? null : callback;
  if (callback !== null) {
    update.callback = callback;
  }
  enqueueUpdate(current, update);

  // 3. 进入reconciler运作流程中的`输入`环节
  scheduleUpdateOnFiber(current, lane, eventTime);
  return lane;
}
```

最初的ReactElement对象 `<App/>`被挂载到HostRootFiber.updateQueue.shared.pending.payload.element中, 后文fiber树构造过程中会再次变动.

在scheduleUpdateOnFiber 函数中:

```js
// ...省略部分代码
export function scheduleUpdateOnFiber(
  fiber: Fiber,
  lane: Lane,
  eventTime: number,
) {
  // 标记优先级,在(对比更新中才会发挥作用),因为在初次创建时并没有与当前页面所对应的fiber树, 所以核心代码并没有执行, 最后直接返回了FiberRoot对象.
  const root = markUpdateLaneFromFiberToRoot(fiber, lane);
  if (lane === SyncLane) {
    if (
      (executionContext & LegacyUnbatchedContext) !== NoContext &&
      (executionContext & (RenderContext | CommitContext)) === NoContext
    ) {
      // 首次渲染, 直接进行`fiber构造`
      performSyncWorkOnRoot(root);
    }
    // ...
  }
}
```

可以看到, 在Legacy模式下且首次渲染时, 有 2 个函数 `markUpdateLaneFromFiberToRoot`和 `performSyncWorkOnRoot`.

其中 `markUpdateLaneFromFiberToRoot(fiber, lane)`函数在fiber树构造(对比更新)中才会发挥作用, 因为在初次创建时并没有与当前页面所对应的fiber树, 所以核心代码并没有执行, 最后直接返回了FiberRoot对象.

performSyncWorkOnRoot看起来源码很多, 初次创建中真正用到的就 2 个函数:

```js
function performSyncWorkOnRoot(root) {
  let lanes;
  let exitStatus;
  if (
    root === workInProgressRoot &&
    includesSomeLane(root.expiredLanes, workInProgressRootRenderLanes)
  ) {
    // 初次构造时(因为root=fiberRoot, workInProgressRoot=null), 所以不会进入
  } else {
    // 1. 获取本次render的优先级, 初次构造返回 NoLanes
    lanes = getNextLanes(root, NoLanes);
    // 2. 从root节点开始, 至上而下更新
    exitStatus = renderRootSync(root, lanes);
  }

  // 将最新的fiber树挂载到root.finishedWork节点上
  const finishedWork: Fiber = (root.current.alternate: any);
  root.finishedWork = finishedWork;
  root.finishedLanes = lanes;
  // 进入commit阶段
  commitRoot(root);

  // ...后面的内容本节不讨论
}
```

1. 其中getNextLanes返回本次 render 的渲染优先级
2. renderRootSync

```js
function renderRootSync(root: FiberRoot, lanes: Lanes) {
  const prevExecutionContext = executionContext;
  executionContext |= RenderContext;
  // 如果fiberRoot变动, 或者update.lane变动, 都会刷新栈帧, 丢弃上一次渲染进度,
  //legacy模式下必进入
  if (workInProgressRoot !== root || workInProgressRootRenderLanes !== lanes) {
    // 刷新栈帧, legacy模式下都会进入
    prepareFreshStack(root, lanes);
  }
  do {
    try {
      workLoopSync();
      break;
    } catch (thrownValue) {
      handleError(root, thrownValue);
    }
  } while (true);
  executionContext = prevExecutionContext;
  // 重置全局变量, 表明render结束
  workInProgressRoot = null;
  workInProgressRootRenderLanes = NoLanes;
  return workInProgressRootExitStatus;
}
```

在renderRootSync中, 在执行fiber树构造前(workLoopSync)会先刷新栈帧 `prepareFreshStack()`.在这里创建了 `HostRootFiber.alternate`, 重置全局变量 `workInProgress`和 `workInProgressRoot`等.

#### 循环构造

```js
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}

function workLoopConcurrent() {
  // Perform work until Scheduler asks us to yield
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

可以看到workLoopConcurrent相比于Sync, 会多一个停顿机制, 这个机制实现了时间切片和可中断渲染(参考React 调度原理)

结合performUnitOfWork函数

```js
// ... 省略部分无关代码
function performUnitOfWork(unitOfWork: Fiber): void {
  // unitOfWork就是被传入的workInProgress
  const current = unitOfWork.alternate;
  let next;
  next = beginWork(current, unitOfWork, subtreeRenderLanes);
  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  if (next === null) {
    // 如果没有派生出新的节点, 则进入completeWork阶段, 传入的是当前unitOfWork
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
  }
}
```

可以明显的看出, 整个fiber树构造是一个深度优先遍历(可参考React 算法之深度优先遍历), 其中有 2 个重要的变量 `workInProgress`和 `current`:

- workInProgress和current都视为指针
- workInProgress指向当前正在构造的fiber节点
- `current = workInProgress.alternate(即fiber.alternate)`, 指向当前页面正在使用的fiber节点. 初次构造时, 页面还未渲染, 此时current = null.
  在深度优先遍历中, 每个fiber节点都会经历 2 个阶段:
- 探寻阶段 beginWork
- 回溯阶段 completeWork
  这 2 个阶段共同完成了每一个fiber节点的创建, 所有fiber节点则构成了fiber树.

```js
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  const updateLanes = workInProgress.lanes;
  if (current !== null) {
    // update逻辑, 首次render不会进入
  } else {
    didReceiveUpdate = false;
  }
  // 1. 设置workInProgress优先级为NoLanes(最高优先级)
  workInProgress.lanes = NoLanes;
  // 2. 根据workInProgress节点的类型, 用不同的方法派生出子节点
  switch (
    workInProgress.tag // 只保留了本例使用到的case
  ) {
    case ClassComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateClassComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderLanes,
      );
    }
    case HostRoot:
      return updateHostRoot(current, workInProgress, renderLanes);
    case HostComponent:
      return updateHostComponent(current, workInProgress, renderLanes);
    case HostText:
      return updateHostText(current, workInProgress);
    case Fragment:
      return updateFragment(current, workInProgress, renderLanes);
  }
}

```

#### 探寻阶段(探寻阶段 beginWork)

`beginWork(current, unitOfWork, subtreeRenderLanes)`针对所有的 Fiber 类型, 其中的每一个 case 处理一种 Fiber 类型. updateXXX函数(如: updateHostRoot, updateClassComponent 等)的主要逻辑:

- 根据 ReactElement对象创建所有的fiber节点, 最终构造出fiber树形结构(设置return和sibling指针)
- 设置fiber.flags(二进制形式变量, 用来标记 fiber节点 的增,删,改状态, 等待completeWork阶段处理)
- 设置fiber.stateNode局部状态(如Class类型节点: fiber.stateNode=new Class())

```js
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  const updateLanes = workInProgress.lanes;
  if (current !== null) {
    // update逻辑, 首次render不会进入
  } else {
    didReceiveUpdate = false;
  }
  // 1. 设置workInProgress优先级为NoLanes(最高优先级)
  workInProgress.lanes = NoLanes;
  // 2. 根据workInProgress节点的类型, 用不同的方法派生出子节点
  switch (
    workInProgress.tag // 只保留了本例使用到的case
  ) {
    case ClassComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateClassComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderLanes,
      );
    }
    case HostRoot:
      return updateHostRoot(current, workInProgress, renderLanes);
    case HostComponent:
      return updateHostComponent(current, workInProgress, renderLanes);
    case HostText:
      return updateHostText(current, workInProgress);
    case Fragment:
      return updateFragment(current, workInProgress, renderLanes);
  }
}
```

基本逻辑:

1. 根据fiber.pendingProps,fiber.updateQueue等输入数据状态,计算fiber.memoizedState作为输出状态
2. 获取下级ReactElement对象

- class类型的fiber节点
  * 构建React.Component实例
  * 把新的实例挂载到fiber.stateNode上
  * 执行render之前的生命周期函数
  * 执行render方法,获取下级reactElement
  * 根据实际情况,设置fiber.flags
- funciton类型的fiber节点
  * 执行function,获取下级reactElement
  * 根据实际情况,设置fiber.flags

3. 根据ReactElement对象,调用 `reconcileChildren`生成Fiber子节点(只生成次级子节点)

- 根据实际情况, 设置fiber.flags
  不同的updateXXX函数处理的fiber节点类型不同, 总的目的是为了向下生成子节点. 在这个过程中把一些需要持久化的数据挂载到fiber节点上(如fiber.stateNode,fiber.memoizedState等); 把fiber节点的特殊操作设置到fiber.flags(如:节点ref,class组件的生命周期,function组件的hook,节点删除等).

这里列出updateHostRoot, updateHostComponent的代码, 对于其他常用 case 的分析(如class类型, function类型), 在状态组件章节中进行探讨.

fiber树的根节点是HostRootFiber节点, 所以第一次进入beginWork会调用updateHostRoot(current, workInProgress, renderLanes)

```js
// 省略与本节无关代码
function updateHostRoot(current, workInProgress, renderLanes) {
  // 1. 状态计算, 更新整合到 workInProgress.memoizedState中来
  const updateQueue = workInProgress.updateQueue;
  const nextProps = workInProgress.pendingProps;
  const prevState = workInProgress.memoizedState;
  const prevChildren = prevState !== null ? prevState.element : null;
  cloneUpdateQueue(current, workInProgress);
  // 遍历updateQueue.shared.pending, 提取有足够优先级的update对象, 计算出最终的状态 workInProgress.memoizedState
  processUpdateQueue(workInProgress, nextProps, null, renderLanes);
  const nextState = workInProgress.memoizedState;
  // 2. 获取下级`ReactElement`对象
  const nextChildren = nextState.element;
  const root: FiberRoot = workInProgress.stateNode;
  if (root.hydrate && enterHydrationState(workInProgress)) {
    // ...服务端渲染相关, 此处省略
  } else {
    // 3. 根据`ReactElement`对象, 调用`reconcileChildren`生成`Fiber`子节点(只生成`次级子节点`)
    reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  }
  return workInProgress.child;
}
```

普通 DOM 标签类型的节点(如div,span,p),会进入updateHostComponent:

```js
// ...省略部分无关代码
function updateHostComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
) {
  // 1. 状态计算, 由于HostComponent是无状态组件, 所以只需要收集 nextProps即可, 它没有 memoizedState
  const type = workInProgress.type;
  const nextProps = workInProgress.pendingProps;
  const prevProps = current !== null ? current.memoizedProps : null;
  // 2. 获取下级`ReactElement`对象
  let nextChildren = nextProps.children;
  const isDirectTextChild = shouldSetTextContent(type, nextProps);

  if (isDirectTextChild) {
    // 如果子节点只有一个文本节点, 不用再创建一个HostText类型的fiber
    nextChildren = null;
  } else if (prevProps !== null && shouldSetTextContent(type, prevProps)) {
    // 特殊操作需要设置fiber.flags
    workInProgress.flags |= ContentReset;
  }
  // 特殊操作需要设置fiber.flags
  markRef(current, workInProgress);
  // 3. 根据`ReactElement`对象, 调用`reconcileChildren`生成`Fiber`子节点(只生成`次级子节点`)
  reconcileChildren(current, workInProgress, nextChildren, renderLanes);(diff算法)
  return workInProgress.child;
}
```

#### 回溯阶段completeWork

`completeUnitOfWork(unitOfWork)`, 处理 beginWork 阶段已经创建出来的 fiber 节点, 核心逻辑:

1. 调用completeWork
   给fiber节点(tag=HostComponent, HostText)创建 DOM 实例, 设置fiber.stateNode局部状态(如tag=HostComponent, HostText节点: fiber.stateNode 指向这个 DOM 实例).
   为 DOM 节点设置属性, 绑定事件(这里先说明有这个步骤, 详细的事件处理流程, 在合成事件原理中详细说明).
   设置fiber.flags标记
2. 把当前 fiber 对象的副作用队列(firstEffect和lastEffect)添加到父节点的副作用队列之后, 更新父节点的firstEffect和lastEffect指针.**effect队列会一直上移**
3. 识别beginWork阶段设置的fiber.flags, 判断当前 fiber 是否有副作用(增,删,改), 如果有, 需要将当前 fiber 加入到父节点的effects队列, 等待commit阶段处理.

```js
function completeUnitOfWork(unitOfWork: Fiber): void {
  let completedWork = unitOfWork;
  // 外层循环控制并移动指针(`workInProgress`,`completedWork`等)
  do {
    const current = completedWork.alternate;
    const returnFiber = completedWork.return;
    if ((completedWork.flags & Incomplete) === NoFlags) {
      let next;
      // 1. 处理Fiber节点, 会调用渲染器(调用react-dom包, 关联Fiber节点和dom对象, 绑定事件等)
      next = completeWork(current, completedWork, subtreeRenderLanes); // 处理单个节点
      if (next !== null) {
        // 如果派生出其他的子节点, 则回到`beginWork`阶段进行处理
        workInProgress = next;
        return;
      }
      // 重置子节点的优先级
      resetChildLanes(completedWork);
      if (
        returnFiber !== null &&
        (returnFiber.flags & Incomplete) === NoFlags
      ) {
        // 2. 收集当前Fiber节点以及其子树的副作用effects
        // 2.1 把子节点的副作用队列添加到父节点上
        if (returnFiber.firstEffect === null) {
          returnFiber.firstEffect = completedWork.firstEffect;
        }
        if (completedWork.lastEffect !== null) {
          if (returnFiber.lastEffect !== null) {
            returnFiber.lastEffect.nextEffect = completedWork.firstEffect;
          }
          returnFiber.lastEffect = completedWork.lastEffect;
        }
        // 2.2 如果当前fiber节点有副作用, 将其添加到子节点的副作用队列之后.
        const flags = completedWork.flags;
        if (flags > PerformedWork) {
          // PerformedWork是提供给 React DevTools读取的, 所以略过PerformedWork
          if (returnFiber.lastEffect !== null) {
            returnFiber.lastEffect.nextEffect = completedWork;
          } else {
            returnFiber.firstEffect = completedWork;
          }
          returnFiber.lastEffect = completedWork;
        }
      }
    } else {
      // 异常处理, 本节不讨论
    }

    const siblingFiber = completedWork.sibling;
    if (siblingFiber !== null) {
      // 如果有兄弟节点, 返回之后再次进入`beginWork`阶段
      workInProgress = siblingFiber;
      return;
    }
    // 移动指针, 指向下一个节点
    completedWork = returnFiber;
    workInProgress = completedWork;
  } while (completedWork !== null);
  // 已回溯到根节点, 设置workInProgressRootExitStatus = RootCompleted
  if (workInProgressRootExitStatus === RootIncomplete) {
    workInProgressRootExitStatus = RootCompleted;
  }
}
```

接下来分析fiber处理函数completeWork

```js
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  const newProps = workInProgress.pendingProps;
  switch (workInProgress.tag) {
    case ClassComponent: {
      // Class类型不做处理
      return null;
    }
    case HostRoot: {
      const fiberRoot = (workInProgress.stateNode: FiberRoot);
      if (fiberRoot.pendingContext) {
        fiberRoot.context = fiberRoot.pendingContext;
        fiberRoot.pendingContext = null;
      }
      if (current === null || current.child === null) {
         // 设置fiber.flags标记
         workInProgress.flags |= Snapshot;
      }
      return null;
    }
    case HostComponent: {
      popHostContext(workInProgress);
      const rootContainerInstance = getRootHostContainer();
      const type = workInProgress.type;
      if (current !== null && workInProgress.stateNode != null) {
        // update逻辑, 初次render不会进入
      } else {
        const currentHostContext = getHostContext();
        // 1. 创建DOM对象
        const instance = createInstance(
          type,
          newProps,
          rootContainerInstance,
          currentHostContext,
          workInProgress,
        );
        // 2. 把子树中的DOM对象append到本节点的DOM对象之后
        appendAllChildren(instance, workInProgress, false, false);
        // 设置stateNode属性, 指向DOM对象
        workInProgress.stateNode = instance;
        if (
          // 3. 设置DOM对象的属性, 绑定事件等
          finalizeInitialChildren(
            instance,
            type,
            newProps,
            rootContainerInstance,
            currentHostContext,
          )
        ) {
          // 设置fiber.flags标记(Update)
          markUpdate(workInProgress);
        }
        if (workInProgress.ref !== null) {
          // 设置fiber.flags标记(Ref)
          markRef(workInProgress);
        }
        return null;
    }
  }
}
```

**结尾全面总结一下**:

1. begin阶段给节点打tag,complete节点识别beginWork阶段设置的fiber.flags, 判断当前 fiber 是否有副作用(增,删,改), 如果有, 需要将当前 fiber 加入到父节点的effects队列, 等待commit阶段处理.
2. complete阶段还会创建DOM对
3. 初次创建:在React应用首次启动时, 界面还没有渲染, 此时并不会进入对比过程, 相当于直接构造一棵全新的树.
4. 对比更新: React应用启动后, 界面已经渲染. 如果再次发生更新, 创建新fiber之前需要和旧fiber进行对比. 最后构造的 fiber 树有可能是全新的, 也可能是部分更新的.

ReferenceList:

1. https://github.com/7kms/react-illustration-series
2. https://react.iamkasong.com/preparation/idea.html#%E6%80%BB%E7%BB%9
3. https://juejin.cn/post/7085145274200358949
