# React源码解析之React的管理员(reconciler与scheduler)

本系列行文思路如下,本篇属于`React中的React的管理员(reconciler与scheduler)`
* [X] React启动过程
* [X] React的两大工作循环
* [X] React中的对象
* [ ] React fiber的初次创建与更新
* [ ] React fiber的渲染
* [X] React的管理员(reconciler与scheduler)
* [X] react的优先级管理(Lane模型)

**React Hook原理**

* [ ] 状态与副作用
* [ ] Hook原理
* [ ] 状态Hook
* [ ] 副作用Hook

**其他**

* [ ] React的合成事件
* [ ] Context原理
* [ ] diff算法

**React源码解析之Reconciler运行循环**

## Reconciler小循环概览

![](https://raw.githubusercontent.com/captain1023/picGo/master/img/reactfiberworkloop.png)

`reconciler`包的主要作用，将主要功能分为4个方面:

1. **输入**:暴露api函数(如:scheduleUpdateOnFiber)，提供给其他包(如react包)调用.(所有的输入的必经之路)
2. **注册调度任务**:与调度中心(scheduler包)交互,注册调度任务task,等待任务回调.
3. **执行任务回调**:在内存中构造出fiber树,同时与渲染器(react-dom)交互,在内存中创建出与fiber对应的DOM节点.
4. **输出**: 与渲染器(react-dom)交互,渲染DOM节点

### 输入

在 `ReactFiberWorkLoop.js`中, 承接输入的函数只有 `scheduleUpdateOnFiber` 在 `react-reconciler`对外暴露的 api 函数中, 只要涉及到需要改变 fiber 的操作(无论是首次渲染或后续更新操作), 最后都会间接调用 `scheduleUpdateOnFiber`, 所以 `scheduleUpdateOnFiber`函数是**输入链路中的必经之路**.

```js
// 唯一接收输入信号的函数
export function scheduleUpdateOnFiber(
  fiber: Fiber,
  lane: Lane,
  eventTime: number,
) {
  // ... 省略部分无关代码
  //给fiber节点添加优先级
  const root = markUpdateLaneFromFiberToRoot(fiber, lane);
  if (lane === SyncLane) {
    if (
      (executionContext & LegacyUnbatchedContext) !== NoContext &&
      (executionContext & (RenderContext | CommitContext)) === NoContext
    ) {
      // 直接进行`fiber构造`
      // 条件是NoContext,一般是初次构造，不经过调度
      performSyncWorkOnRoot(root);
    } else {
      // 注册调度任务, 经过`Scheduler`包的调度, 间接进行`fiber构造`
      ensureRootIsScheduled(root, eventTime);
    }
  } else {
    // 注册调度任务, 经过`Scheduler`包的调度, 间接进行`fiber构造`
    // 如果同步优先级，走'Scheduler'调度
    ensureRootIsScheduled(root, eventTime);
  }
}
```

### 注册调度任务

与输入环节紧密相连, `scheduleUpdateOnFiber`函数之后, 立即进入 `ensureRootIsScheduled()`函数

```js
// ... 省略部分无关代码
function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
  // 前半部分: 判断是否需要注册新的调度,(如果当前任务已经被挂起则不需要注册新的调度)
  const existingCallbackNode = root.callbackNode;
  const nextLanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes,
  );
  const newCallbackPriority = returnNextLanesPriority();
  if (nextLanes === NoLanes) {
    return;
  }
  if (existingCallbackNode !== null) {
    const existingCallbackPriority = root.callbackPriority;
    if (existingCallbackPriority === newCallbackPriority) {
      return;
    }
    cancelCallback(existingCallbackNode);
  }

  // 后半部分: 注册调度任务
  //其实是在scheduleCallback去创建task,然后加入任务队列里面,后面有详细解释
  let newCallbackNode;
  if (newCallbackPriority === SyncLanePriority) {
    newCallbackNode = scheduleSyncCallback(
      performSyncWorkOnRoot.bind(null, root),
    );
  } else if (newCallbackPriority === SyncBatchedLanePriority) {
    newCallbackNode = scheduleCallback(
      ImmediateSchedulerPriority,
      performSyncWorkOnRoot.bind(null, root),
    );
  } else {
    // concurrent模式下注册任务,然后加入任务队列
    const schedulerPriorityLevel = lanePriorityToSchedulerPriority(
      newCallbackPriority,
    );
    newCallbackNode = scheduleCallback(
      schedulerPriorityLevel,
      performConcurrentWorkOnRoot.bind(null, root),
    );
  }
  root.callbackPriority = newCallbackPriority;
  root.callbackNode = newCallbackNode;
}
```

1. 前半部分: 判断是否需要注册新的调度(如果无需新的调度, 会退出函数)
2. 后半部分: 注册调度任务
   - `performSyncWorkOnRoo`t或`performConcurrentWorkOnRoot`被封装到了任务回调(scheduleCallback)
   - 等待调度中心执行任务, 任务运行其实就是执行`performSyncWorkOnRoot`或`performConcurrentWorkOnRoot`(其实就是把对应模式的优先级和回调放到`root.callbackPriority`和`root.callbackNode`中，等待执行`performSyncWorkOnRoot`或`performConcurrentWorkOnRoot`)

### 执行任务回调

任务回调, 实际上就是执行`performSyncWorkOnRoot`或`performConcurrentWorkOnRoot`.

**performSyncWorkOnRoot**

```js
// ... 省略部分无关代码
function performSyncWorkOnRoot(root) {
  let lanes;
  let exitStatus;

  lanes = getNextLanes(root, NoLanes);
  // 1. fiber树构造
  exitStatus = renderRootSync(root, lanes);

  // 2. 异常处理: 有可能fiber构造过程中出现异常
  if (root.tag !== LegacyRoot && exitStatus === RootErrored) {
    // ...
  }

  // 3. 输出: 渲染fiber树 TODO:在这儿把alternate换到了finishedwork傻姑娘
  const finishedWork: Fiber = (root.current.alternate: any);
  root.finishedWork = finishedWork;
  root.finishedLanes = lanes;
  commitRoot(root);

  // 退出前再次检测, 是否还有其他更新, 是否需要发起新调度,比如在ComponentDidMount中再次调用setState()
  ensureRootIsScheduled(root, now());
  return null;
}
```

**performConcurrentWorkOnRoot**

```js
// ... 省略部分无关代码
function performConcurrentWorkOnRoot(root) {

  const originalCallbackNode = root.callbackNode;

  // 1. 刷新pending状态的effects, 有可能某些effect会取消本次任务
  // 检查是否处于render过程中，是否需要恢复上一次渲染
  // 如果之前Update的优先级有改变(之前的渲染任务改变了),则直接放弃上一次的渲染结果.比如连续两次setState,但优先级不同
  const didFlushPassiveEffects = flushPassiveEffects();
  if (didFlushPassiveEffects) {
    if (root.callbackNode !== originalCallbackNode) {
      // 任务被取消, 退出调用
      return null;
    } else {
      // Current task was not canceled. Continue.
    }
  }
  // 2. 获取本次渲染的优先级
  let lanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes,
  );
  // 3. 构造fiber树
  let exitStatus = renderRootConcurrent(root, lanes);

  if (
    includesSomeLane(
      workInProgressRootIncludedLanes,
      workInProgressRootUpdatedLanes,
    )
  ) {
    // 如果在render过程中产生了新的update, 且新update的优先级与最初render的优先级有交集
    // 那么最初render无效, 丢弃最初render的结果, 等待下一次调度
    prepareFreshStack(root, NoLanes);//这里是在丢弃最初的render结果
  } else if (exitStatus !== RootIncomplete) {
    // 4. 异常处理: 有可能fiber构造过程中出现异常
    if (exitStatus === RootErrored) {
      // ...
    }.
    const finishedWork: Fiber = (root.current.alternate: any);
    root.finishedWork = finishedWork;
    root.finishedLanes = lanes;
    // 5. 输出: 渲染fiber树
    finishConcurrentRender(root, exitStatus, lanes);//对应commitRoot
  }

  // 退出前再次检测, 是否还有其他更新, 是否需要发起新调度
  // 在渲染阶段发起了新的更新
  ensureRootIsScheduled(root, now());
  if (root.callbackNode === originalCallbackNode) {
    // 渲染被阻断, 返回一个新的performConcurrentWorkOnRoot函数, 等待下一次调用
    return performConcurrentWorkOnRoot.bind(null, root);
  }
  return null;
}
```

**commitRoot**:前两个函数中对应的输出部分

```js
// ... 省略部分无关代码
function commitRootImpl(root, renderPriorityLevel) {
  // 设置局部变量
  const finishedWork = root.finishedWork;
  const lanes = root.finishedLanes;

  // 清空FiberRoot对象上的属性
  root.finishedWork = null;
  root.finishedLanes = NoLanes;
  root.callbackNode = null;

  // 提交阶段
  let firstEffect = finishedWork.firstEffect;
  if (firstEffect !== null) {
    const prevExecutionContext = executionContext;
    executionContext |= CommitContext;
    // 阶段1: dom突变之前
    nextEffect = firstEffect;
    do {
      commitBeforeMutationEffects();
    } while (nextEffect !== null);

    // 阶段2: dom突变, 界面发生改变
    nextEffect = firstEffect;
    do {
      commitMutationEffects(root, renderPriorityLevel);
    } while (nextEffect !== null);
    root.current = finishedWork;

    // 阶段3: layout阶段, 调用生命周期componentDidUpdate和回调函数等
    nextEffect = firstEffect;
    do {
      commitLayoutEffects(root, lanes);
    } while (nextEffect !== null);
    nextEffect = null;
    executionContext = prevExecutionContext;
  }
  ensureRootIsScheduled(root, now());
  return null;
}
```

本文只是对整个流程有个大概的了解,这些函数在fiber树构造和更新的过程中会有更详细的解释



## scheduler大循环调度原理
**React scheduler** react reconciler的第二部分,与scheduler的交互就是与scheduler的交互
### 内核

`scheduler.js`文件一共导出了8个函数,最核心的逻辑,就集中在了这8个函数中:

```js
export let requestHostCallback; // 请求及时回调: port.postMessage
export let cancelHostCallback; // 取消及时回调: scheduledHostCallback = null
export let requestHostTimeout; // 请求延时回调: setTimeout
export let cancelHostTimeout; // 取消延时回调: cancelTimeout
export let shouldYieldToHost; // 是否让出主线程(currentTime >= deadline && needsPaint): 让浏览器能够执行更高优先级的任务(如ui绘制, 用户输入等)
export let requestPaint; // 请求绘制: 设置 needsPaint = true
export let getCurrentTime; // 获取当前时间
export let forceFrameRate; // 强制设置 yieldInterval (让出主线程的周期). 这个函数虽然存在, 但是从源码来看, 几乎没有用到
```

我们知道 react 可以在 nodejs 环境中使用, 所以在不同的 js 执行环境中, 这些函数的实现会有区别. 下面基于普通浏览器环境, 对这 8 个函数逐一分析 :

1. requestHostCallback
2. cancelHostCallback
3. requestHostTimeout
4. cancelHostTimeout

这 4 个函数源码很简洁, 非常好理解, 它们的目的就是请求执行(或取消)回调函数. 现在重点介绍其中的及时回调(延时回调的 2 个函数暂时属于保留 api, 17.0.2 版本其实没有用上)

```js
const performWorkUntilDeadline = () => {
  // ...省略无关代码
  if (scheduledHostCallback !== null) {
    const currentTime = getCurrentTime();
    // 更新deadline
    deadline = currentTime + yieldInterval;
    // 执行callback
    //这个callback是flushWork,是一个大循环,flushWork后面有介绍.
    scheduledHostCallback(hasTimeRemaining, currentTime);
  } else {
    isMessageLoopRunning = false;
  }
};

const channel = new MessageChannel();
const port = channel.port2;
channel.port1.onmessage = performWorkUntilDeadline;

// 请求回调
requestHostCallback = function(callback) {
  // 1. 保存callback,这个callback是flushWork,是一个大循环,flushWork后面有介绍.
  scheduledHostCallback = callback;
  if (!isMessageLoopRunning) {
    isMessageLoopRunning = true;
    // 2. 通过 MessageChannel 发送消息
    port.postMessage(null);
  }
};
// 取消回调
cancelHostCallback = function() {
  scheduledHostCallback = null;
};

```

很明显,请求回调之后 `scheduledHostCallback = callback`, 然后通过MessageChannel发消息方式触发 `performWorkUntilDeadline`函数,最后执行回调 `scheduledHostCallback`.
此处需要注意:Message Channel在浏览器事件循环中属于宏任务,所以调度中心永远是**异步执行**回调函数(类似setTimeout之类的)

2. 时间切片(time slicing)相关:执行时间分割,让出主线程(把控制权归还浏览器,浏览器可以处理用户输入,UI绘制等紧急任务)
    - `getCurrentTime`: 获取当前时间
    - `shouldYieldToHost`: 是否让出主线程
   - `requestPaint`: 请求绘制
   - `forceFrameRate`: 强制设置 yieldInterval(从源码中的引用来看, 算一个保留函数, 其他地方没有用到)

```js
const localPerformance = performance;
// 获取当前时间
getCurrentTime = () => localPerformance.now();

// 时间切片周期, 默认是5ms(如果一个task运行超过该周期, 下一个task执行之前, 会把控制权归还浏览器)
let yieldInterval = 5;

let deadline = 0;
const maxYieldInterval = 300;
let needsPaint = false;
const scheduling = navigator.scheduling;
// 是否让出主线程
shouldYieldToHost = function() {
  const currentTime = getCurrentTime();
  if (currentTime >= deadline) {
    if (needsPaint || scheduling.isInputPending()) {
      // There is either a pending paint or a pending input.
      return true;
    }
    // There's no pending input. Only yield if we've reached the max
    // yield interval.
    return currentTime >= maxYieldInterval; // 在持续运行的react应用中, currentTime肯定大于300ms, 这个判断只在初始化过程中才有可能返回false
  } else {
    // There's still time left in the frame.
    return false;
  }
};

// 请求绘制
requestPaint = function() {
  needsPaint = true;
};

// 设置时间切片的周期
forceFrameRate = function(fps) {
  if (fps < 0 || fps > 125) {
    // Using console['error'] to evade Babel and ESLint
    console['error'](
      'forceFrameRate takes a positive int between 0 and 125, ' +
        'forcing frame rates higher than 125 fps is not supported',
    );
    return;
  }
  if (fps > 0) {
    yieldInterval = Math.floor(1000 / fps);
  } else {
    // reset the framerate
    yieldInterval = 5;
  }
};

```

注意`shouldYieldToHost`的判定条件:

- currentTime >= deadline: 只有时间超过deadline之后才会让出主线程(其中deadline = currentTime + yieldInterval).
- yieldInterval默认是5ms, 只能通过forceFrameRate函数来修改(事实上在 v17.0.2 源码中, 并没有使用到该函数).
- 如果一个task运行时间超过5ms, 下一个task执行之前, 会把控制权归还浏览器.
  完整的回调的实现`performWorkUntilDeadline`
- 任务创建之后会通过`messageChannel`去触发`performWorkUntilDeadline`函数(见下文)

```js
const performWorkUntilDeadline = () => {
  if (scheduledHostCallback !== null) {
    const currentTime = getCurrentTime(); // 1. 获取当前时间
    deadline = currentTime + yieldInterval; // 2. 设置deadline
    const hasTimeRemaining = true;
    try {
      // 3. 执行回调, 返回是否有还有剩余任务
      const hasMoreWork = scheduledHostCallback(hasTimeRemaining, currentTime);
      if (!hasMoreWork) {
        // 没有剩余任务, 退出
        isMessageLoopRunning = false;
        scheduledHostCallback = null;
      } else {
        port.postMessage(null); // 有剩余任务, 发起新的调度
      }
    } catch (error) {
      port.postMessage(null); // 如有异常, 重新发起调度
      throw error;
    }
  } else {
    isMessageLoopRunning = false;
  }
  needsPaint = false; // 重置开关
};
```

### 任务队列管理

通过上文的分析, 我们已经知道请求和取消调度的实现原理. 调度的目的是为了消费任务, 接下来就具体分析任务队列是如何管理与实现的.
在Scheduler.js中, 维护了一个taskQueue, 任务队列管理就是围绕这个taskQueue展开.

```js
// Tasks are stored on a min heap(最小堆)
var taskQueue = [];
var timerQueue = [];
```

源码中除了taskQueue队列之外还有一个timerQueue队列. 这个队列是预留给延时任务使用的, 在 react@17.0.2 版本里面, 从源码中的引用来看, 算一个保留功能, 没有用到.

### 创建任务

```js
// 省略部分无关代码
function unstable_scheduleCallback(priorityLevel, callback, options) {
  // 1. 获取当前时间
  var currentTime = getCurrentTime();
  var startTime;
  if (typeof options === 'object' && options !== null) {
    // 从函数调用关系来看, 在v17.0.2中,所有调用 unstable_scheduleCallback 都未传入options
    // 所以省略延时任务相关的代码
  } else {
    startTime = currentTime;
  }
  // 2. 根据传入的优先级, 设置任务的过期时间 expirationTime
  var timeout;
  switch (priorityLevel) {
    case ImmediatePriority:
      timeout = IMMEDIATE_PRIORITY_TIMEOUT;
      break;
    case UserBlockingPriority:
      timeout = USER_BLOCKING_PRIORITY_TIMEOUT;
      break;
    case IdlePriority:
      timeout = IDLE_PRIORITY_TIMEOUT;
      break;
    case LowPriority:
      timeout = LOW_PRIORITY_TIMEOUT;
      break;
    case NormalPriority:
    default:
      timeout = NORMAL_PRIORITY_TIMEOUT;
      break;
  }
  var expirationTime = startTime + timeout;
  // 3. 创建新任务
  var newTask = {
    id: taskIdCounter++,
    callback,
    priorityLevel,
    startTime,
    expirationTime,
    sortIndex: -1,
  };
  if (startTime > currentTime) {
    // 省略无关代码 v17.0.2中不会使用
  } else {
    newTask.sortIndex = expirationTime;
    // 4. 加入任务队列
    push(taskQueue, newTask);
    // 5. 请求调度
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      //上文解释过,通过message Chanel的onchange
      requestHostCallback(flushWork);
    }
  }
  return newTask;
}

```

**task对象的各个属性**

```js
var newTask = {
  id: taskIdCounter++, // id: 一个自增编号
  callback, // callback: 传入的回调函数
  priorityLevel, // priorityLevel: 优先级等级
  startTime, // startTime: 创建task时的当前时间
  expirationTime, // expirationTime: task的过期时间, 优先级越高 expirationTime = startTime + timeout 越小
  sortIndex: -1,
};
newTask.sortIndex = expirationTime; // sortIndex: 排序索引, 全等于过期时间. 保证过期时间越小, 越紧急的任务排在最前面
```

### 消费任务

创建任务后,最后请求调度 `requestHostCallback(flushWork)`(创建任务中的第5步),flushWork函数作为参数被传入调度中心内心等待回调

```js
// 省略无关代码
function flushWork(hasTimeRemaining, initialTime) {
  // 1. 做好全局标记, 表示现在已经进入调度阶段
  isHostCallbackScheduled = false;
  isPerformingWork = true;
  const previousPriorityLevel = currentPriorityLevel;
  try {
    // 2. 循环消费队列
    return workLoop(hasTimeRemaining, initialTime);
  } finally {
    // 3. 还原全局标记
    currentTask = null;
    currentPriorityLevel = previousPriorityLevel;
    isPerformingWork = false;
  }
}
```

flushWork中调用了 `workLoop()`队列消费的主要逻辑是在workLoop函数中,这就是**任务调度循环**,返回值是True Or False;

```js
// 省略部分无关代码
function workLoop(hasTimeRemaining, initialTime) {
  let currentTime = initialTime; // 保存当前时间, 用于判断任务是否过期
  currentTask = peek(taskQueue); // 获取队列中的第一个任务
  while (currentTask !== null) {
    if (
      currentTask.expirationTime > currentTime &&
      (!hasTimeRemaining || shouldYieldToHost())
    ) {
      // 虽然currentTask没有过期, 但是执行时间超过了限制(毕竟只有5ms, shouldYieldToHost()返回true). 停止继续执行, 让出主线程
      break;
    }
    const callback = currentTask.callback;
    if (typeof callback === 'function') {
      currentTask.callback = null;
      currentPriorityLevel = currentTask.priorityLevel;
      const didUserCallbackTimeout = currentTask.expirationTime <= currentTime;
      // 执行回调,这个callback是fiber的循环构造的回调函数
      const continuationCallback = callback(didUserCallbackTimeout);
      currentTime = getCurrentTime();
      // 回调完成, 判断是否还有连续(派生)回调
      if (typeof continuationCallback === 'function') {
        // 产生了连续回调(如fiber树太大, 出现了中断渲染), 保留currentTask
        currentTask.callback = continuationCallback;
      } else {
        // 把currentTask移出队列
        if (currentTask === peek(taskQueue)) {
          pop(taskQueue);
        }
      }
    } else {
      // 如果任务被取消(这时currentTask.callback = null), 将其移出队列
      pop(taskQueue);
    }
    // 更新currentTask,去执行下一个任务
    currentTask = peek(taskQueue);
  }
  if (currentTask !== null) {
    return true; // 如果task队列没有清空, 返回true. 等待调度中心下一次回调
  } else {
    return false; // task队列已经清空, 返回false.
  }
}
```

1. 此处实现了时间切片(time slicing)和fiber树的可中断渲染.这两大特性的实现,都集中于这个while循环的退出条件

- 每一次while循环的退出就是一个时间切片,深入分析while循环的退出条件
  * 如果某个task.callback(大循环)执行时间太长(如:fiber树很大,或逻辑很重)也会造成超时
  * 所以在执行task.callback过程中, 也需要一种机制检测是否超时, 如果超时了就立刻暂停task.callback的执行.

2. 时间切片原理
   消费任务队列的过程中, 可以消费1~n个 task, 甚至清空整个 queue. 但是在每一次具体执行task.callback(workLoop的大循环)之前都要进行超时检测, 如果超时可以立即退出循环并等待下一次调用.
3. 可中断渲染原理
   在时间切片的基础之上, 如果单个task.callback执行时间就很长(假设 200ms). 就需要task.callback自己能够检测是否超时, 所以在 fiber 树构造过程中, 每构造完成一个单元, 都会检测一次超时(源码链接), 如遇超时就退出fiber树构造循环, 并返回一个新的回调函数(就是此处的`continuationCallback`)并等待下一次回调继续未完成的fiber树构造.

```js
// ... 省略部分无关代码
function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
  // 前半部分: 判断是否需要注册新的调度
  const existingCallbackNode = root.callbackNode;
  const nextLanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes,
  );
  const newCallbackPriority = returnNextLanesPriority();
  if (nextLanes === NoLanes) {
    return;
  }
  // 节流防抖
  if (existingCallbackNode !== null) {
    const existingCallbackPriority = root.callbackPriority;
    if (existingCallbackPriority === newCallbackPriority) {
      return;
    }
    cancelCallback(existingCallbackNode);
  }
  // 后半部分: 注册调度任务 省略代码...

  // 更新标记
  root.callbackPriority = newCallbackPriority;
  root.callbackNode = newCallbackNode;
}
```

正常情况下, `ensureRootIsScheduled`函数会与 `scheduler`包通信, 最后注册一个task并等待回调.

1. 在task注册完成之后, 会设置fiberRoot对象上的属性(fiberRoot是 react 运行时中的重要全局对象, 可参考React 应用的启动过程), 代表现在已经处于调度进行中
2. 再次进入 `ensureRootIsSchedule`d时(比如连续 2 次setState, 第 2 次setState同样会触发reconciler运作流程中的调度阶段), 如果发现处于调度中, 则需要一些节流和防抖措施, 进而保证调度性能.

- 节流(判断条件: `existingCallbackPriority === newCallbackPriority`, 新旧更新的优先级相同, 如连续多次执行setState), 则无需注册新task(继续沿用上一个优先级相同的task), 直接退出调用.
- 防抖(判断条件: `existingCallbackPriority !== newCallbackPriority`, 新旧更新的优先级不同), 则取消旧task, 重新注册新task.

---

ReferenceList:

1. https://github.com/7kms/react-illustration-series
2. https://react.iamkasong.com/preparation/idea.html#%E6%80%BB%E7%BB%9
3. https://juejin.cn/post/708514527420035894