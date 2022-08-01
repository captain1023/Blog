# React源码解析之Lane优先级

本篇属于React中的优先级管理
* [X] React启动过程
* [X] React的两大工作循环
* [X] React中的对象
* [ ] React fiber的初次创建与更新
* [ ] React fiber的渲染
* [X] React的管理员(reconciler运行循环)
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

---

## React优先级管理

1. 2套优先级体系和1套转换体系
2. React内部对于优先级的管理, 贯穿运作流程的 4 个阶段(从输入到输出), 根据其功能的不同, 可以分为 3 种类型:

- fiber优先级(`LanePriority`): 位于react-reconciler包, 也就是Lane(车道模型).
- 调度优先级(`SchedulerPriority`): 位于scheduler包.
- 优先级等级(`ReactPriorityLevel`) : 位于react-reconciler包中的SchedulerWithReactIntegration.js, 负责上述 2 套优先级体系的转换.

### Lane(车道模型)

- Lane类型被定义为二进制变量, 利用了位掩码的特性, 在频繁运算的时候占用内存少, 计算速度快.
  * Lane和Lanes就是单数和复数的关系, 代表单个任务的定义为`Lane`, 代表多个任务的定义为`Lanes`
- Lane是对于expirationTime的重构, 以前使用expirationTime表示的字段, 都改为了lane

```js
  renderExpirationtime -> renderLanes
  update.expirationTime -> update.lane
  fiber.expirationTime -> fiber.lanes
  fiber.childExpirationTime -> fiber.childLanes
  root.firstPendingTime and root.lastPendingTime -> fiber.pendingLanes
```

- 使用Lanes模型相比expirationTime模型的优势:
  * Lanes把任务优先级从批量任务中分离出来, 可以更方便的判断单个任务与批量任务的优先级是否重叠.

```js
// 判断: 单task与batchTask的优先级是否重叠
//1. 通过expirationTime判断
const isTaskIncludedInBatch = priorityOfTask >= priorityOfBatch;
//2. 通过Lanes判断
const isTaskIncludedInBatch = (task & batchOfTasks) !== 0;

// 当同时处理一组任务, 该组内有多个任务, 且每个任务的优先级不一致
// 1. 如果通过expirationTime判断. 需要维护一个范围(在Lane重构之前, 源码中就是这样比较的)
const isTaskIncludedInBatch =
  taskPriority <= highestPriorityInRange &&
  taskPriority >= lowestPriorityInRange;
//2. 通过Lanes判断
const isTaskIncludedInBatch = (task & batchOfTasks) !== 0;
```

* Lanes使用单个 32 位二进制变量即可代表多个不同的任务, 也就是说一个变量即可代表一个组(group), 如果要在一个 group 中分离出单个 task, 非常容易.
  - 在expirationTime模型设计之初, react 体系中还没有Suspense 异步渲染的概念. 现在有如下场景: 有 3 个任务, 其优先级 A > B > C, 正常来讲只需要按照优先级顺序执行就可以了. 但是现在情况变了: A 和 C 任务是CPU密集型, 而 B 是IO密集型(Suspense 会调用远程 api, 算是 IO 任务), 即 A(cpu) > B(IO) > C(cpu). 此时的需求需要将任务B从 group 中分离出来, 先处理 cpu 任务A和C.

```js
// 从group中删除或增加task

//1. 通过expirationTime实现
// 0) 维护一个链表, 按照单个task的优先级顺序进行插入
// 1) 删除单个task(从链表中删除一个元素)
task.prev.next = task.next;
// 2) 增加单个task(需要对比当前task的优先级, 插入到链表正确的位置上)
let current = queue;
while (task.expirationTime >= current.expirationTime) {
current = current.next;
}
task.next = current.next;
current.next = task;
// 3) 比较task是否在group中
const isTaskIncludedInBatch =
taskPriority <= highestPriorityInRange &&
taskPriority >= lowestPriorityInRange;

// 2. 通过Lanes实现
// 1) 删除单个task
batchOfTasks &= ~task;
// 2) 增加单个task
batchOfTasks |= task;
// 3) 比较task是否在group中
const isTaskIncludedInBatch = (task & batchOfTasks) !== 0;
```

* Lanes是一个不透明的类型, 只能在`ReactFiberLane.js`这个模块中维护. 如果要在其他文件中使用, 只能通过`ReactFiberLane.js`中提供的工具函数来使用.
* 可以使用的比特位一共有 31 位
* 共定义了18 种车道(Lane/Lanes)变量, 每一个变量占有 1 个或多个比特位, 分别定义为Lane和Lanes类型.
* 每一种车道(Lane/Lanes)都有对应的优先级, 所以源码中定义了 18 种优先级(LanePriority).
* 占有低位比特位的Lane变量对应的优先级越高
* 最高优先级为SyncLanePriority对应的车道为SyncLane = 0b0000000000000000000000000000001.
* 最低优先级为OffscreenLanePriority对应的车道为OffscreenLane = 0b1000000000000000000000000000000.

### 优先级区别和联系

1. `LanePriority`和`SchedulerPriority`从命名上看, 它们代表的是优先级
   `ReactPriorityLevel`从命名上看, 它代表的是等级而不是优先级, 它用于衡量`LanePriority`和`SchedulerPriority`的等级.

### Lane Priority

LanePriority: 属于react-reconciler包, 定义于ReactFiberLane.js.

```js
export const SyncLanePriority: LanePriority = 15;
export const SyncBatchedLanePriority: LanePriority = 14;

const InputDiscreteHydrationLanePriority: LanePriority = 13;
export const InputDiscreteLanePriority: LanePriority = 12;

// .....

const OffscreenLanePriority: LanePriority = 1;
export const NoLanePriority: LanePriority = 0;
```

与fiber构造过程相关的优先级(如fiber.updateQueue,fiber.lanes)都使用LanePriority.
关于本文重点介绍优先级体系以及它们的转换关系, 关于Lane(车道模型)在fiber树构造时的具体使用, 在fiber 树构造中详细解读.

### SchedulerPriority

SchedulerPriority, 属于scheduler包, 定义于SchedulerPriorities.js中.

```js
export const ImmediatePriority = 1;
export const UserBlockingPriority = 2;
export const NormalPriority = 3;
export const LowPriority = 4;
export const IdlePriority = 5;
```

与scheduler调度中心相关的优先级使用SchedulerPriority.

### ReactPriorityLevel

reactPriorityLevel, 属于react-reconciler包, 定义于SchedulerWithReactIntegration.js中(见源码).

```js
export const ImmediatePriority: ReactPriorityLevel = 99;
export const UserBlockingPriority: ReactPriorityLevel = 98;
export const NormalPriority: ReactPriorityLevel = 97;
export const LowPriority: ReactPriorityLevel = 96;
export const IdlePriority: ReactPriorityLevel = 95;
// NoPriority is the absence of priority. Also React-only.
export const NoPriority: ReactPriorityLevel = 90;
```

### 转换关系

为了能协同调度中心(scheduler包)和 fiber 树构造(react-reconciler包)中对优先级的使用, 则需要转换`SchedulerPriority`和`LanePriority`, 转换的桥梁正是`ReactPriorityLevel`.

在`SchedulerWithReactIntegration.js`中, 可以互转`SchedulerPriority` 和 `ReactPriorityLevel`:

```js
// 把 SchedulerPriority 转换成 ReactPriorityLevel
export function getCurrentPriorityLevel(): ReactPriorityLevel {
  switch (Scheduler_getCurrentPriorityLevel()) {
    case Scheduler_ImmediatePriority:
      return ImmediatePriority;
    case Scheduler_UserBlockingPriority:
      return UserBlockingPriority;
    case Scheduler_NormalPriority:
      return NormalPriority;
    case Scheduler_LowPriority:
      return LowPriority;
    case Scheduler_IdlePriority:
      return IdlePriority;
    default:
      invariant(false, 'Unknown priority level.');
  }
}

// 把 ReactPriorityLevel 转换成 SchedulerPriority
function reactPriorityToSchedulerPriority(reactPriorityLevel) {
  switch (reactPriorityLevel) {
    case ImmediatePriority:
      return Scheduler_ImmediatePriority;
    case UserBlockingPriority:
      return Scheduler_UserBlockingPriority;
    case NormalPriority:
      return Scheduler_NormalPriority;
    case LowPriority:
      return Scheduler_LowPriority;
    case IdlePriority:
      return Scheduler_IdlePriority;
    default:
      invariant(false, 'Unknown priority level.');
  }
}
```

在`ReactFiberLane.js`中, 可以互转`LanePriority` 和 `ReactPriorityLevel`:

```js
export function schedulerPriorityToLanePriority(
  schedulerPriorityLevel: ReactPriorityLevel,
): LanePriority {
  switch (schedulerPriorityLevel) {
    case ImmediateSchedulerPriority:
      return SyncLanePriority;
    // ... 省略部分代码
    default:
      return NoLanePriority;
  }
}

export function lanePriorityToSchedulerPriority(
  lanePriority: LanePriority,
): ReactPriorityLevel {
  switch (lanePriority) {
    case SyncLanePriority:
    case SyncBatchedLanePriority:
      return ImmediateSchedulerPriority;
    // ... 省略部分代码
    default:
      invariant(
        false,
        'Invalid update priority: %s. This is a bug in React.',
        lanePriority,
      );
  }
}
```



创建任务后,最后请求调度requestHostCallback(flushWork)(创建任务中的第5步),flushWork函数作为参数被传入调度中心内心等待回调

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

flushWork中调用了workLoop(因为使用了(),所以返回的是函数运行的结果,如果直接返回函数名,就是返回这个函数).队列消费的主要逻辑是在workLoop函数中,这就是**任务调度循环**,返回值是True Or False;

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
      // 执行回调
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
      // TODO:什么情况下任务会被取消啊?
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
  * 如果某个task.callback(大循环呗)执行时间太长(如:fiber树很大,或逻辑很重)也会造成超时
  * 所以在执行task.callback过程中, 也需要一种机制检测是否超时, 如果超时了就立刻暂停task.callback的执行.

2. 时间切片原理
   消费任务队列的过程中, 可以消费1~n个 task, 甚至清空整个 queue. 但是在每一次具体执行task.callback(workLoop的大循环)之前都要进行超时检测, 如果超时可以立即退出循环并等待下一次调用.
3. 可中断渲染原理
   在时间切片的基础之上, 如果单个task.callback执行时间就很长(假设 200ms). 就需要task.callback自己能够检测是否超时, 所以在 fiber 树构造过程中, 每构造完成一个单元, 都会检测一次超时(源码链接), 如遇超时就退出fiber树构造循环, 并返回一个新的回调函数(就是此处的continuationCallback)并等待下一次回调继续未完成的fiber树构造.

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

正常情况下, ensureRootIsScheduled函数会与scheduler包通信, 最后注册一个task并等待回调.

1. 在task注册完成之后, 会设置fiberRoot对象上的属性(fiberRoot是 react 运行时中的重要全局对象, 可参考React 应用的启动过程), 代表现在已经处于调度进行中
2. 再次进入ensureRootIsScheduled时(比如连续 2 次setState, 第 2 次setState同样会触发reconciler运作流程中的调度阶段), 如果发现处于调度中, 则需要一些节流和防抖措施, 进而保证调度性能.

- 节流(判断条件: existingCallbackPriority === newCallbackPriority, 新旧更新的优先级相同, 如连续多次执行setState), 则无需注册新task(继续沿用上一个优先级相同的task), 直接退出调用.
- 防抖(判断条件: existingCallbackPriority !== newCallbackPriority, 新旧更新的优先级不同), 则取消旧task, 重新注册新task.


---

ReferenceList:

1. https://github.com/7kms/react-illustration-series
2. https://react.iamkasong.com/preparation/idea.html#%E6%80%BB%E7%BB%9
3. https://juejin.cn/post/7085145274200358949