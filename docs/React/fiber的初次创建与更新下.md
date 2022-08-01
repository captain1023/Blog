# fiber的初次渲染与更新下

本篇属于React中的`React fiber的初次创建与更新(下)`
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

## React fiber的初次创建与更新(下)

### 更新入口

前文**React的管理员(reconciler运行循环)**中总结的 4 个阶段(从输入到输出), 其中承接输入的函数只有 `scheduleUpdateOnFibe`.在react-reconciler对外暴露的 api 函数中, 只要涉及到需要改变 fiber 的操作(无论是首次渲染或对比更新), 最后都会间接调用 `scheduleUpdateOnFiber`, `scheduleUpdateOnFiber`函数是输入链路中的必经之路.

### 3种更新方式

**如要主动发起更新, 有 3 种常见方式**

1. Class组件中调用setState.
2. Function组件中调用hook对象暴露出的dispatchAction.
3. 在container节点上重复调用render

##### setState

在Component对象的原型上挂载有setState:

```js
Component.prototype.setState = function(partialState, callback) {
  this.updater.enqueueSetState(this, partialState, callback, 'setState');
};
```

在fiber 树构造(初次创建)中的 `beginWork`阶段, class 类型的组件初始化完成之后, this.updater对象如下:

```js
const classComponentUpdater = {
  isMounted,
  enqueueSetState(inst, payload, callback) {
    // 1. 获取class实例对应的fiber节点
    const fiber = getInstance(inst);
    // 2. 创建update对象
    const eventTime = requestEventTime();
    const lane = requestUpdateLane(fiber); // 确定当前update对象的优先级
    const update = createUpdate(eventTime, lane);
    update.payload = payload;
    if (callback !== undefined && callback !== null) {
      update.callback = callback;
    }
    // 3. 将update对象添加到当前Fiber节点的updateQueue队列当中
    enqueueUpdate(fiber, update);
    // 4. 进入reconciler运作流程中的`输入`环节
    scheduleUpdateOnFiber(fiber, lane, eventTime); // 传入的lane是update优先级
  },
};

```

第二种方式:Function组件中调用hook对象暴露出的dispatchAction

```js
function dispatchAction<S, A>(
  fiber: Fiber,
  queue: UpdateQueue<S, A>,
  action: A,
) {
  // 1. 创建update对象
  const eventTime = requestEventTime();
  const lane = requestUpdateLane(fiber); // 确定当前update对象的优先级
  const update: Update<S, A> = {
    lane,
    action,
    eagerReducer: null,
    eagerState: null,
    next: (null: any),
  };
  // 2. 将update对象添加到当前Hook对象的updateQueue队列当中
  const pending = queue.pending;
  if (pending === null) {
    update.next = update;
  } else {
    update.next = pending.next;
    pending.next = update;
  }
  queue.pending = update;
  // 3. 请求调度, 进入reconciler运作流程中的`输入`环节.
  scheduleUpdateOnFiber(fiber, lane, eventTime); // 传入的lane是update优先级
}
```

第三种方式:重复调用render

```js
import ReactDOM from 'react-dom';
function tick() {
  const element = (
    <div>
      <h1>Hello, world!</h1>
      <h2>It is {new Date().toLocaleTimeString()}.</h2>
    </div>
  );
  ReactDOM.render(element, document.getElementById('root'));
}
setInterval(tick, 1000);
```

对于重复render, 在React 应用的启动过程中已有说明, 调用路径包含updateContainer-->scheduleUpdateOnFiber

故无论从哪个入口进行更新, 最终都会进入 `scheduleUpdateOnFiber`.所以 `scheduleUpdateOnFiber()`是reconciler包的唯一入口.

#### 构造阶段

逻辑来到scheduleUpdateOnFiber函数:

```js
// ...省略部分代码
export function scheduleUpdateOnFiber(
  fiber: Fiber, // fiber表示被更新的节点
  lane: Lane, // lane表示update优先级
  eventTime: number,
) {
  const root = markUpdateLaneFromFiberToRoot(fiber, lane);//对比节点生效
  if (lane === SyncLane) {
    if (
      (executionContext & LegacyUnbatchedContext) !== NoContext &&
      (executionContext & (RenderContext | CommitContext)) === NoContext
    ) {
      // 初次渲染
      performSyncWorkOnRoot(root);
    } else {
      // 对比更新
      ensureRootIsScheduled(root, eventTime);
    }
  }
  mostRecentlyUpdatedRoot = root;
}
```

对比更新与初次渲染的不同的点

1. `markUpdateLaneFromFiberToRoot`函数, 只在对比更新阶段才发挥出它的作用, 它找出了fiber树中受到本次update影响的所有节点, 并设置这些节点的fiber.lanes或fiber.childLanes(在legacy模式下为SyncLane)以备fiber树构造阶段(perform阶段)使用.代码如下

```js
function markUpdateLaneFromFiberToRoot(
  sourceFiber: Fiber, // sourceFiber表示被更新的节点
  lane: Lane, // lane表示update优先级
): FiberRoot | null {
  // 1. 将update优先级设置到sourceFiber.lanes
  sourceFiber.lanes = mergeLanes(sourceFiber.lanes, lane);
  let alternate = sourceFiber.alternate;
  if (alternate !== null) {
    // 同时设置sourceFiber.alternate的优先级
    alternate.lanes = mergeLanes(alternate.lanes, lane);
  }
  // 2. 从sourceFiber开始, 向上遍历所有节点, 直到HostRoot. 设置沿途所有节点(包括alternate)的childLanes
  let node = sourceFiber;
  let parent = sourceFiber.return;
  while (parent !== null) {
    parent.childLanes = mergeLanes(parent.childLanes, lane);
    alternate = parent.alternate;
    if (alternate !== null) {
      alternate.childLanes = mergeLanes(alternate.childLanes, lane);
    }
    node = parent;
    parent = parent.return;
  }
  if (node.tag === HostRoot) {
    const root: FiberRoot = node.stateNode;
    return root;
  } else {
    return null;
  }
}
```

* 以sourceFiber为起点, 设置起点的fiber.lanes
* 从起点开始, 直到HostRootFiber, 设置父路径上所有节点(也包括fiber.alternate)的fiber.childLanes.
* 通过设置fiber.lanes和fiber.childLanes就可以辅助判断子树是否需要更新(在下文循环构造中详细说明).

2. 对比更新没有直接调用performSyncWorkOnRoot, 而是通过调度中心来处理, 由于本示例是在Legacy模式下进行, 最后会同步执行performSyncWorkOnRoot.(详细原理可以参考React 调度原理(scheduler)). 所以其调用链路performSyncWorkOnRoot--->renderRootSync--->workLoopSync与初次构造中的一致.

**renderRootSync**

```js
function renderRootSync(root: FiberRoot, lanes: Lanes) {
  const prevExecutionContext = executionContext;
  executionContext |= RenderContext;
  // 如果fiberRoot变动, 或者update.lane变动, 都会刷新栈帧, 丢弃上一次渲染进度
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

**注意😭**

- fiberRoot.current指向与当前页面对应的fiber树, workInProgress指向正在构造的fiber树.
- 刷新栈帧会调用createWorkInProgress(), 使得workInProgress.flags和workInProgress.effects都已经被重置. 且workInProgress.child = current.child. 所以在进入循环构造之前, HostRootFiber与HostRootFiber.alternate共用一个child(这里是fiber(`<App/>`)).

#### 循环构造

回顾一下fiber 树构造(初次创建)中的介绍. 整个fiber树构造是一个深度优先遍历, 其中有 2 个重要的变量workInProgress和current(详见系列文章**React的管理员**)

- workInProgress和current都视为指针
- workInProgress指向当前正在构造的fiber节点
- current = workInProgress.alternate(即fiber.alternate), 指向当前页面正在使用的fiber节点.
  在深度优先遍历中, 每个fiber节点都会经历 2 个阶段:

1. 探寻阶段 beginWork:创建(更新)fiber节点,并打tag
2. 回溯阶段 completeWork:处理beginwork传来的fiber节点,并生成dom节点
   这 2 个阶段共同完成了每一个fiber节点的创建(或更新), 所有fiber节点则构成了fiber树.

```js
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}

// ... 省略部分无关代码
function performUnitOfWork(unitOfWork: Fiber): void {
  // unitOfWork就是被传入的workInProgress
  const current = unitOfWork.alternate;
  let next;
  //重点:在对比更新过程中current = unitOfWork.alternate;不为null, 后续的调用逻辑中会大量使用此处传入的current.
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

注意:在对比更新过程中current = unitOfWork.alternate;不为null, 后续的调用逻辑中会大量使用此处传入的current.

```js
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  const updateLanes = workInProgress.lanes;
  if (current !== null) {
    // 进入对比
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;
    if (
      oldProps !== newProps ||
      hasLegacyContextChanged() ||
      (__DEV__ ? workInProgress.type !== current.type : false)
    ) {
      didReceiveUpdate = true;
    } else if (!includesSomeLane(renderLanes, updateLanes)) {
      // 当前渲染优先级renderLanes不包括fiber.lanes, 表明当前fiber节点无需更新,
      didReceiveUpdate = false;
      switch (
        workInProgress.tag
        // switch 语句中包括 context相关逻辑, 本节暂不讨论(不影响分析fiber树构造)
      ) {
      }
      // 当前fiber节点无需更新, 调用bailoutOnAlreadyFinishedWork循环检测子节点是否需要更新
      //需要更新的子节点会去走updateXXX函数,updateXXX中会调用reconcileChildren()函数

      return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
    }
  }
  // 余下逻辑与初次创建共用
  // 1. 设置workInProgress优先级为NoLanes(最高优先级)
  workInProgress.lanes = NoLanes;
  // 2. 根据workInProgress节点的类型, 用不同的方法派生出子节点
  switch (
    workInProgress.tag // 只列出部分case
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

#### bailout逻辑

与初次创建不同, 在对比更新过程中, 如果是老节点, 那么current !== null, 需要进行对比, 然后决定是否复用老节点及其子树(即bailout逻辑).

- !includesSomeLane(renderLanes, updateLanes)这个判断分支, 包含了渲染优先级和update优先级的比较(**fiber初次创建与更新(上)**), 如果当前节点无需更新, 则会进入bailout逻辑.
- 最后会调用bailoutOnAlreadyFinishedWork:
  如果同时满足!includesSomeLane(renderLanes, workInProgress.childLanes), 表明该 fiber 节点及其子树都无需更新, 可直接进入回溯阶段(completeUnitOfWork)
  如果不满足!includesSomeLane(renderLanes, workInProgress.childLanes), 意味着子节点需要更新, clone并返回子节点.

```js
// 省略部分无关代码
function bailoutOnAlreadyFinishedWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  if (!includesSomeLane(renderLanes, workInProgress.childLanes)) {
    // 渲染优先级不包括 workInProgress.childLanes, 表明子节点也无需更新. 返回null, 直接进入回溯阶段.
    return null;
  } else {
    // 本fiber虽然不用更新, 但是子节点需要更新. clone并返回子节点
    cloneChildFibers(current, workInProgress);
    return workInProgress.child;
  }
}
```

注意: cloneChildFibers内部调用createWorkInProgress, 在构造fiber节点时会优先复用workInProgress.alternate(不开辟新的内存空间), 否则才会创建新的fiber对象.

#### updateXX函数

updateXXX函数(如: updateHostRoot, updateClassComponent 等)的主干逻辑与初次构造过程完全一致, 总的目的是为了向下生成子节点, 并在这个过程中调用**reconcileChildren**调和函数, 只要fiber节点有副作用, 就会把特殊操作设置到fiber.flags(如:节点ref,class组件的生命周期,function组件的hook,节点删除等).

**对比更新**过程的不同支持:

1. **bailoutOnAlreadyFinishedWork**

- 对比更新时如果遇到当前节点无需更新(如:class类型的节点且shouldComponentUpdate返回false),会再次进入bailout逻辑(clone子节点并返回,重走beginwork流程)

2. **reconcileChildren调和函数**

- 调和函数的作用是向下生成子节点, 并设置fiber.flags.
- 初次创建时fiber节点没有比较对象, 所以在向下生成子节点的时候没有任何多余的逻辑, 只管创建就行.
- 对比更新时需要把ReactElement对象与旧fiber对象进行比较, 来判断是否需要复用旧fiber对象.

#### 调和函数目的

- 给新增,移动,和删除节点设置fiber.flags(新增,移动: Placement, 删除: Deletion)
- 如果是需要删除的fiber, 除了自身打上Deletion之外, 还要将其添加到父节点的effects链表中(正常副作用队列的处理是在completeWork函数, 但是该节点(被删除)会**脱离fiber树**, **不会**再进入completeWork阶段, 所以**在beginWork阶段**s提前加入副作用队列).

#### 回溯阶段

completeUnitOfWork(unitOfWork)函数在初次创建和对比更新逻辑一致, 都是处理beginWork 阶段已经创建出来的 fiber 节点, 最后创建(更新)DOM 对象, 并上移副作用队列.

在这里我们重点关注completeWork函数中, current !== null的情况:

```js
// ...省略无关代码
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  const newProps = workInProgress.pendingProps;
  switch (workInProgress.tag) {
    case HostComponent: {
      // 非文本节点
      popHostContext(workInProgress);
      const rootContainerInstance = getRootHostContainer();
      const type = workInProgress.type;
      if (current !== null && workInProgress.stateNode != null) {
        // 处理改动
        updateHostComponent(
          current,
          workInProgress,
          type,
          newProps,
          rootContainerInstance,
        );
        if (current.ref !== workInProgress.ref) {
          markRef(workInProgress);
        }
      } else {
        // ...省略无关代码
      }
      return null;
    }
    case HostText: {
      // 文本节点
      const newText = newProps;
      if (current && workInProgress.stateNode != null) {
        const oldText = current.memoizedProps;
        // 处理改动
        updateHostText(current, workInProgress, oldText, newText);
      } else {
        // ...省略无关代码
      }
      return null;
    }
  }
}

updateHostComponent = function(
  current: Fiber,
  workInProgress: Fiber,
  type: Type,
  newProps: Props,
  rootContainerInstance: Container,
) {
  const oldProps = current.memoizedProps;
  if (oldProps === newProps) {
    return;
  }
  const instance: Instance = workInProgress.stateNode;
  const currentHostContext = getHostContext();
  const updatePayload = prepareUpdate(
    instance,
    type,
    oldProps,
    newProps,
    rootContainerInstance,
    currentHostContext,
  );
  workInProgress.updateQueue = (updatePayload: any);
  // 如果有属性变动, 设置fiber.flags |= Update, 等待`commit`阶段的处理
  if (updatePayload) {
    markUpdate(workInProgress);
  }
};
updateHostText = function(
  current: Fiber,
  workInProgress: Fiber,
  oldText: string,
  newText: string,
) {
  // 如果有属性变动, 设置fiber.flags |= Update, 等待`commit`阶段的处理
  if (oldText !== newText) {
    markUpdate(workInProgress);
  }
};
```

可以看到在更新过程中, 如果 DOM 属性有变化, 不会再次新建 DOM 对象, 而是设置fiber.flags |= Update, 等待commit阶段处

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

1. 调用**completeWork**
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

1. **begin阶段**给节点打tag,complete节点识别beginWork阶段设置的fiber.flags, 判断当前 fiber 是否有副作用(增,删,改), 如果有, 需要将当前 fiber 加入到父节点的effects队列, 等待commit阶段处理.
2. **complete阶段**创建DOM树
3. 初次创建:在React应用首次启动时, 界面还没有渲染, 此时并不会进入对比过程, 相当于直接构造一棵全新的树.
4. 对比更新: React应用启动后, 界面已经渲染. 如果再次发生更新, 创建新fiber之前需要和旧fiber进行对比. 最后构造的 fiber 树有可能是全新的, 也可能是部分更新的.

ReferenceList:

1. https://github.com/7kms/react-illustration-series
2. https://react.iamkasong.com/preparation/idea.html#%E6%80%BB%E7%BB%9
3. https://juejin.cn/post/7085145274200358949
