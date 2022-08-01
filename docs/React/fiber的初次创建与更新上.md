# fiber的初次渲染与更新上

本篇属于React中的 `React fiber的初次创建与更新(上)`
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

## fiber 树构造(基础准备)

本文行文思路将根据fiber的初次创建和对比更新来详细说明

1. 初次创建:在React应用首次启动时, 界面还没有渲染, 此时并不会进入对比过程, 相当于直接构造一棵全新的树.
2. 对比更新: React应用启动后, 界面已经渲染. 如果再次发生更新, 创建新fiber之前需要和旧fiber进行对比. 最后构造的 fiber 树有可能是全新的, 也可能是部分更新的.

### scheduler流程回顾(详细请查看系列文章之React中的对象)

![](https://raw.githubusercontent.com/captain1023/picGo/master/img/React%E4%B8%A4%E5%A4%A7%E5%B7%A5%E4%BD%9C%E5%BE%AA%E7%8E%AF.png)

### ReactElement, Fiber, DOM 三者的关系

- 转换流程:ReactElement(JSX)->Fiber->DOM
- 开发人员能够控制的是JSX, 也就是ReactElement对象.
- fiber树是通过ReactElement生成的, 如果脱离了ReactElement,fiber树也无从谈起. 所以是ReactElement树(不是严格的树结构, 为了方便也称为树)驱动fiber树.
- fiber树是DOM树的数据模型, fiber树驱动DOM树

#### 全局变量

从React工作循环的角度来看,整个构造过程被包裹在fiber树构造循环中(源码文件:ReactFiberWorkLoop.js)
在React运行时, ReactFiberWorkLoop.js闭包中的全局变量会随着fiber树构造循环的进行而变化, 现在查看其中重要的全局变量:

```js
// 当前React的执行栈(执行上下文)
let executionContext: ExecutionContext = NoContext;

// 当前root节点
let workInProgressRoot: FiberRoot | null = null;
// 正在处理中的fiber节点
let workInProgress: Fiber | null = null;
// 正在渲染的车道(复数)
let workInProgressRootRenderLanes: Lanes = NoLanes;

// 包含所有子节点的优先级, 是workInProgressRootRenderLanes的超集
// 大多数情况下: 在工作循环整体层面会使用workInProgressRootRenderLanes, 在begin/complete阶段层面会使用 subtreeRenderLanes
let subtreeRenderLanes: Lanes = NoLanes;
// 一个栈结构: 专门存储当前节点的 subtreeRenderLanes
const subtreeRenderLanesCursor: StackCursor<Lanes> = createCursor(NoLanes);

// fiber构造完后, root节点的状态: completed, errored, suspended等
let workInProgressRootExitStatus: RootExitStatus = RootIncomplete;
// 重大错误
let workInProgressRootFatalError: mixed = null;
// 整个render期间所使用到的所有lanes
let workInProgressRootIncludedLanes: Lanes = NoLanes;
// 在render期间被跳过(由于优先级不够)的lanes: 只包括未处理的updates, 不包括被复用的fiber节点
let workInProgressRootSkippedLanes: Lanes = NoLanes;
// 在render期间被修改过的lanes
let workInProgressRootUpdatedLanes: Lanes = NoLanes;

// 防止无限循环和嵌套更新
const NESTED_UPDATE_LIMIT = 50;
let nestedUpdateCount: number = 0;
let rootWithNestedUpdates: FiberRoot | null = null;

const NESTED_PASSIVE_UPDATE_LIMIT = 50;
let nestedPassiveUpdateCount: number = 0;

// 发起更新的时间
let currentEventTime: number = NoTimestamp;
let currentEventWipLanes: Lanes = NoLanes;
let currentEventPendingLanes: Lanes = NoLanes;
```

#### 执行上下文

在全局变量中 `executionContext`,代表渲染期间的执行栈(或叫做执行上下文), 它也是一个二进制表示的变量, 通过位运算进行操作. 在源码中一共定义了 8 种执行栈:

```js
type ExecutionContext = number;
export const NoContext = /*             */ 0b0000000;
const BatchedContext = /*               */ 0b0000001;
const EventContext = /*                 */ 0b0000010;
const DiscreteEventContext = /*         */ 0b0000100;
const LegacyUnbatchedContext = /*       */ 0b0001000;
const RenderContext = /*                */ 0b0010000;
const CommitContext = /*                */ 0b0100000;
```

在 render 过程中, 每一个阶段都会改变 `executionContext`(render 之前, 会设置 `executionContext |= RenderContext; `commit 之前, 会设置 `executionContext |= CommitContext)`, 假设在render过程中再次发起更新(如在UNSAFE_componentWillReceiveProps生命周期中调用setState)则可通过executionContext来判断当前的render状态.

#### 双缓冲技术(double buffering)

在全局变量中有workInProgress, 还有不少以workInProgress来命名的变量. workInProgress的应用实际上就是React的双缓冲技术(double buffering).

在上文我们梳理了ReactElement, Fiber, DOM三者的关系, fiber树的构造过程, 就是把ReactElement转换成fiber树的过程. 在这个过程中, 内存里会同时存在 2 棵fiber树:

- 其一: 代表当前界面的fiber树(已经被展示出来, 挂载到fiberRoot.current上). 如果是初次构造(初始化渲染), 页面还没有渲染, 此时界面对应的 fiber 树为空(fiberRoot.current = null).
- 其二: 正在构造的fiber树(即将展示出来, 挂载到HostRootFiber.alternate上, 正在构造的节点称为workInProgress). 当构造完成之后, 重新渲染页面, 最后切换fiberRoot.current = workInProgress, 使得fiberRoot.current重新指向代表当前界面的fiber树.
- ![](https://raw.githubusercontent.com/captain1023/picGo/master/img/fibertreecreate1-progress.png)

### 优先级

在整个react-reconciler包中, Lane的应用可以分为 3 个方面:

##### udpat优先级

在React 应用中的高频对象一文中, 介绍过update对象, 它是一个**环形链表**. 对于单个update对象来讲, update.lane代表它的优先级, 称之为update优先级.

优先级是由外界传入的

```js
export function createUpdate(eventTime: number, lane: Lane): Update<*> {
  const update: Update<*> = {
    eventTime,
    lane,
    tag: UpdateState,
    payload: null,
    callback: null, //这儿的callback是什么
    next: null,
  };
  return update;
}
```

在React体系中,有2种情况会创建update对象:

1. 应用初始化: 在react-reconciler包中的updateContainer函数中(源码)

```js
export function updateContainer(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  callback: ?Function,
): Lane {
  const current = container.current;
  const eventTime = requestEventTime();
  const lane = requestUpdateLane(current); // 根据当前时间, 创建一个update优先级
  const update = createUpdate(eventTime, lane); // lane被用于创建update对象
  update.payload = { element };
  enqueueUpdate(current, update);
  scheduleUpdateOnFiber(current, lane, eventTime);
  return lane;
}
```

2. 发起组建更新:假设在class组件中调用setState

```js
const classComponentUpdater = {
  isMounted,
  enqueueSetState(inst, payload, callback) {
    const fiber = getInstance(inst);
    const eventTime = requestEventTime(); // 根据当前时间, 创建一个update优先级
    const lane = requestUpdateLane(fiber); // lane被用于创建update对象
    const update = createUpdate(eventTime, lane);
    update.payload = payload;
    enqueueUpdate(fiber, update);
    scheduleUpdateOnFiber(fiber, lane, eventTime);
  },
};
```

可以看到, 无论是应用初始化或者发起组件更新, 创建update.lane的逻辑都是一样的, 都是根据当前时间, 创建一个 update 优先级.

如何获取优先级?requestUpdateLane

```js
export function requestUpdateLane(fiber: Fiber): Lane {
  // Special cases(个人觉得是第一次渲染的时候,NoMode)
  const mode = fiber.mode;
  if ((mode & BlockingMode) === NoMode) {
    // legacy 模式
    return (SyncLane: Lane);
  } else if ((mode & ConcurrentMode) === NoMode) {
    // blocking模式
    return getCurrentPriorityLevel() === ImmediateSchedulerPriority
      ? (SyncLane: Lane)
      : (SyncBatchedLane: Lane);
  }
  // concurrent模式
  if (currentEventWipLanes === NoLanes) {
    currentEventWipLanes = workInProgressRootIncludedLanes;
  }
  const isTransition = requestCurrentTransition() !== NoTransition;
  if (isTransition) {
    // 特殊情况, 处于suspense过程中
    if (currentEventPendingLanes !== NoLanes) {
      currentEventPendingLanes =
        mostRecentlyUpdatedRoot !== null
          ? mostRecentlyUpdatedRoot.pendingLanes
          : NoLanes;
    }
    return findTransitionLane(currentEventWipLanes, currentEventPendingLanes);
  }
  // 正常情况, 获取调度优先级
  const schedulerPriority = getCurrentPriorityLevel();
  let lane;
  if (
    (executionContext & DiscreteEventContext) !== NoContext &&
    schedulerPriority === UserBlockingSchedulerPriority
  ) {
    // executionContext 存在输入事件. 且调度优先级是用户阻塞性质
    lane = findUpdateLane(InputDiscreteLanePriority, currentEventWipLanes);
  } else {
    // 调度优先级转换为车道模型
    const schedulerLanePriority = schedulerPriorityToLanePriority(
      schedulerPriority,
    );
    lane = findUpdateLane(schedulerLanePriority, currentEventWipLanes);
  }
  return lane;
}
```

1. legacy 模式: 返回SyncLane
2. blocking 模式: 返回SyncLane
3. concurrent 模式:

- 正常情况下, 根据当前的调度优先级来生成一个lane.
- 特殊情况下(处于 suspense 过程中), 会优先选择TransitionLanes通道中的空闲通道(如果所有TransitionLanes通道都被占用, 就取最高优先级. 源码).

最后通过scheduleUpdateOnFiber(current, lane, eventTime);函数, 把update.lane正式带入到了输入阶段.

scheduleUpdateOnFiber是输入阶段的必经函数, 在本系列的文章中已经多次提到, 此处以update.lane的视角分析:

```js
export function scheduleUpdateOnFiber(
  fiber: Fiber,
  lane: Lane,
  eventTime: number,
) {
  if (lane === SyncLane) {
    // legacy或blocking模式
    if (
      (executionContext & LegacyUnbatchedContext) !== NoContext &&
      (executionContext & (RenderContext | CommitContext)) === NoContext
    ) {
      //首次渲染,不走scheduler
      performSyncWorkOnRoot(root);
    } else {
      ensureRootIsScheduled(root, eventTime); // 注册回调任务
      if (executionContext === NoContext) {
        flushSyncCallbackQueue(); // 取消schedule调度 ,主动刷新回调队列,
      }
    }
  } else {
    // concurrent模式
    ensureRootIsScheduled(root, eventTime);
  }
}
```

当lane === SyncLane也就是 legacy 或 blocking 模式中, 注册完回调任务之后(ensureRootIsScheduled(root, eventTime)), 如果执行上下文为空, 会取消 schedule 调度, 主动刷新回调队列flushSyncCallbackQueue().

这里包含了一个热点问题(setState到底是同步还是异步)的标准答案:
如果逻辑进入flushSyncCallbackQueue(executionContext === NoContext), 则会主动取消调度, 并刷新回调, 立即进入fiber树构造过程. 当执行setState下一行代码时, fiber树已经重新渲染了, 故setState体现为同步.
正常情况下, 不会取消schedule调度. 由于schedule调度是通过MessageChannel触发(宏任务), 故体现为异步.

### 总结

update因为页面变动创建的对象,储存在fiberNode的updateQueue当中.Task指的是通过scheduler调度封装的任务.

双缓冲技术也是Hook在更新时得以保全之前状态的根本原因

ReferenceList:

1. https://github.com/7kms/react-illustration-series
2. https://react.iamkasong.com/preparation/idea.html#%E6%80%BB%E7%BB%9
3. https://juejin.cn/post/7085145274200358949
