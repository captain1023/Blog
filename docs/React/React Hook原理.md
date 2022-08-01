# React Hook原理

本篇属于React中`React Hook`原理

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

### 状态

与状态相关有 4 个属性:

- **fiber.pendingProps**: 输入属性, 从ReactElement对象传入的 props. 它和fiber.memoizedProps比较可以得出属性是否变动.
- **fiber.memoizedProps**: 上一次生成子节点时用到的属性, 生成子节点之后保持在内存中. 向下生成子节点之前叫做pendingProps, 生成子节点之后会把pendingProps赋值给memoizedProps用于下一次比较.pendingProps和memoizedProps比较可以得出属性是否变动.
- **fiber.updateQueue**: 存储update更新对象的队列, 每一次发起更新, 都需要在该队列上创建一个update对象.
- **fiber.memoizedState**: 上一次生成子节点之后保持在内存中的局部状态.
  它们的作用只局限于fiber树构造阶段, 直接影响子节点的生成.

### 副作用

与副作用相关有 4 个属性:

1. **fiber.flags**: 标志位, 表明该fiber节点有副作用(在 v17.0.2 中共定义了28 种副作用).
2. **fiber.nextEffect**: 单向链表, 指向下一个副作用 fiber节点.
3. **fiber.firstEffect**: 单向链表, 指向第一个副作用 fiber 节点.
4. **fiber.lastEffect**: 单向链表, 指向最后一个副作用 fiber 节点.

另外, 副作用的设计可以理解为对状态功能不足的补充.

1. 状态是一个静态的功能, 它只能为子节点提供数据源.
2. 而副作用是一个动态功能, 由于它的调用时机是在fiber树渲染阶段, 故它拥有更多的能力, 能轻松获取突变前快照, 突变后的DOM节点等. 甚至通过调用api发起新的一轮fiber树构造, 进而改变更多的状态, 引发更多的副作用.

### 外部api

fiber对象的这 2 类属性, 可以影响到渲染结果, 但是fiber结构始终是一个内核中的结构, 对于外部来讲是无感知的, 对于调用方来讲, 甚至都无需知道fiber结构的存在. 所以正常只有通过暴露api来直接或间接的修改这 2 类属性.

从react包暴露出的api来归纳, 只有 2 类组件支持修改:

#### class组件

```js
class App extends React.Component {
  constructor() {
    this.state = {
      // 初始状态
      a: 1,
    };
  }
  changeState = () => {
    this.setState({ a: ++this.state.a }); // 进入reconciler流程
  };

  // 生命周期函数: 状态相关
  static getDerivedStateFromProps(nextProps, prevState) {
    console.log('getDerivedStateFromProps');
    return prevState;
  }

  // 生命周期函数: 状态相关
  shouldComponentUpdate(newProps, newState, nextContext) {
    console.log('shouldComponentUpdate');
    return true;
  }

  // 生命周期函数: 副作用相关 fiber.flags |= Update
  componentDidMount() {
    console.log('componentDidMount');
  }

  // 生命周期函数: 副作用相关 fiber.flags |= Snapshot
  getSnapshotBeforeUpdate(prevProps, prevState) {
    console.log('getSnapshotBeforeUpdate');
  }

  // 生命周期函数: 副作用相关 fiber.flags |= Update
  componentDidUpdate() {
    console.log('componentDidUpdate');
  }

  render() {
    // 返回下级ReactElement对象
    return <button onClick={this.changeState}>{this.state.a}</button>;
  }
}
```

1. 状态相关：fiber树构造节点

* 构造函数: constructor实例化时执行, 可以设置初始 state, 只执行一次.
* 生命周期: getDerivedStateFromProps在fiber树构造阶段(renderRootSync[Concurrent])执行, 可以修改 state
* 生命周期: shouldComponentUpdate在, fiber树构造阶段(renderRootSync[Concurrent])执行, 返回值决定是否执行 render

2. 副作用相关：fiber树渲染阶段

* 生命周期: getSnapshotBeforeUpdate在fiber树渲染阶段(commitRoot->commitBeforeMutationEffects->commitBeforeMutationEffectOnFiber)执行.
* 生命周期: componentDidMount在fiber树渲染阶段(commitRoot->commitLayoutEffects->commitLayoutEffectOnFiber)执行
* 生命周期: componentDidUpdate在fiber树渲染阶段(commitRoot->commitLayoutEffects->commitLayoutEffectOnFiber)执行.

#### function组件

注: function组件与class组件最大的不同是: class组件会实例化一个instance所以拥有独立的局部状态; 而function组件不会实例化, 它只是被直接调用, 故无法维护一份独立的局部状态, 只能依靠Hook对象间接实现局部状态.

```js
function App() {
  // 状态相关: 初始状态
  const [a, setA] = useState(1);
  const changeState = () => {
    setA(++a); // 进入reconciler流程
  };

  // 副作用相关: fiber.flags |= Update | Passive;
  useEffect(() => {
    console.log(`useEffect`);
  }, []);

  // 副作用相关: fiber.flags |= Update;
  useLayoutEffect(() => {
    console.log(`useLayoutEffect`);
  }, []);

  // 返回下级ReactElement对象
  return <button onClick={changeState}>{a}</button>;
}
```

1. 状态相关: fiber树构造阶段.

* seState在fiber树构造阶段(renderRootSync[Concurrent])执行, 可以修改Hook.memoizedState.

2. 副作用相关: fiber树渲染阶段.

* useEffect在fiber树渲染阶段(commitRoot->commitBeforeMutationEffects->commitBeforeMutationEffectOnFiber)执行(注意是异步执行).
* useLayoutEffect在fiber树渲染阶段(commitRoot->commitLayoutEffects->commitLayoutEffectOnFiber->commitHookEffectListMount)执行(同步执行, 链接).

#### 细节

1. useEffect(function(){}, [])中的函数是异步执行, 因为它经过了调度中心
2. useLayoutEffect和Class组件中的componentDidMount,componentDidUpdate从调用时机上来讲是等价的, 因为他们都在commitRoot->commitLayoutEffects函数中被调用.
3. 

## HOOK原理(概览)

### Hook数据结构

```js
type Update<S, A> = {|
  lane: Lane,
  action: A,
  eagerReducer: ((S, A) => S) | null,
  eagerState: S | null,
  next: Update<S, A>,
  priority?: ReactPriorityLevel,
|};

type UpdateQueue<S, A> = {|
  pending: Update<S, A> | null,
  dispatch: (A => mixed) | null,
  lastRenderedReducer: ((S, A) => S) | null,
  lastRenderedState: S | null,
|};

export type Hook = {|
  memoizedState: any, // 当前状态
  baseState: any, // 基状态
  baseQueue: Update<any, any> | null, // 基队列
  queue: UpdateQueue<any, any> | null, // 更新队列
  next: Hook | null, // next指针
|};

```

从定义来看, Hook对象共有 5 个属性

1. **hook.memoizedState**: 保持在内存中的局部状态.
2. **hook.baseState**: hook.baseQueue中所有update对象合并之后的状态.
3. **hook.baseQueue**: 存储update对象的环形链表, 只包括高于本次渲染优先级的update对象.
4. **hook.queue**: 存储update对象的环形链表, 包括所有优先级的update对象.
5. **hook.next: next指针, 指向链表中的下一个hook.
   所以Hook是一个链表, 单个Hook拥有自己的状态hook.memoizedState和自己的更新队列hook.queue.

### Hook分类

```js
export type HookType =
  | 'useState'
  | 'useReducer'
  | 'useContext'
  | 'useRef'
  | 'useEffect'
  | 'useLayoutEffect'
  | 'useCallback'
  | 'useMemo'
  | 'useImperativeHandle'
  | 'useDebugValue'
  | 'useDeferredValue'
  | 'useTransition'
  | 'useMutableSource'
  | 'useOpaqueIdentifier';
```

#### 状态Hook

狭义上讲, useState, useReducer可以在function组件添加内部的state, 且useState实际上是useReducer的简易封装, 是一个最特殊(简单)的useReducer. 所以将useState, useReducer称为状态Hook.

广义上讲, 只要能实现数据持久化且没有副作用的Hook, 均可以视为状态Hook, 所以还包括useContext, useRef, useCallback, useMemo等. 这类Hook内部没有使用useState/useReducer, 但是它们也能实现多次render时, 保持其初始值不变(即数据持久化)且没有任何副作用.

得益于双缓冲技术(double buffering), 在多次render时, 以fiber为载体, 保证复用同一个Hook对象, 进而实现数据持久化.

#### 副作用Hook

回到fiber视角, 状态Hook实现了状态持久化(等同于class组件维护fiber.memoizedState), 那么副作用Hook则会修改fiber.flags. (通过前文fiber树构造系列的解读, 我们知道在performUnitOfWork->completeWork阶段, 所有存在副作用的fiber节点, 都会被添加到父节点的副作用队列后, 最后在commitRoot阶段处理这些副作用节点.)

另外, 副作用Hook还提供了副作用回调(类似于class组件的生命周期回调), 比如:

```js
// 使用useEffect时, 需要传入一个副作用回调函数
// 在fiber树构造完成之后, commitRoot阶段会处理这些副作用回调
useEffect(() => {
  console.log('这是一个副作用回调函数');
}, []);
```

在react内部, useEffect就是最标准的副作用Hook. 其他比如useLayoutEffect以及自定义Hook, 如果要实现副作用, 必须直接或间接的调用useEffect.

#### 双缓冲技术(double buffering)

详细解释见**React fiber的初次创建与更新(上)**
在全局变量中有workInProgress, 还有不少以workInProgress来命名的变量. workInProgress的应用实际上就是React的双缓冲技术(double buffering).

在上文我们梳理了ReactElement, Fiber, DOM三者的关系, fiber树的构造过程, 就是把ReactElement转换成fiber树的过程. 在这个过程中, 内存里会同时存在 2 棵fiber树:

- 其一: 代表当前界面的fiber树(已经被展示出来, 挂载到fiberRoot.current上). 如果是初次构造(初始化渲染), 页面还没有渲染, 此时界面对应的 fiber 树为空(fiberRoot.current = null).
- 其二: 正在构造的fiber树(即将展示出来, 挂载到HostRootFiber.alternate上, 正在构造的节点称为workInProgress). 当构造完成之后, 重新渲染页面, 最后切换fiberRoot.current = workInProgress, 使得fiberRoot.current重新指向代表当前界面的fiber树.
  ![](https://raw.githubusercontent.com/captain1023/picGo/master/img/fibertreecreate1-progress.png)

#### 处理函数

从fiber树构造的视角来看, 不同的fiber类型, 只需要调用不同的处理函数返回fiber子节点. 所以在performUnitOfWork->beginWork函数中, 调用了多种处理函数. 从调用方来讲, 无需关心处理函数的内部实现(比如updateFunctionComponent内部使用了Hook对象, updateClassComponent内部使用了class实例).

以workInProgress命名的元素都是内存中的缓存,current命名的都是我当前dom

本节讨论Hook, 所以列出其中的updateFunctionComponent函数:

```js
// 只保留FunctionComponent相关:
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  const updateLanes = workInProgress.lanes;
  switch (workInProgress.tag) {
    case FunctionComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateFunctionComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderLanes,
      );
    }
  }
}

function updateFunctionComponent(
  current,
  workInProgress,
  Component,
  nextProps: any,
  renderLanes,
) {
  // ...省略无关代码
  let context;
  let nextChildren;
  //重置全局变量,重置变量在下面列出
  prepareToReadContext(workInProgress, renderLanes);

  // 进入Hooks相关逻辑, 最后返回下级ReactElement对象
  nextChildren = renderWithHooks(
    current,
    workInProgress,
    Component,
    nextProps,
    context,
    renderLanes,
  );
  // 进入reconcile函数, 生成下级fiber节点
  reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  // 返回下级fiber节点
  return workInProgress.child;
}
```

#### 全局变量

```js
// 渲染优先级
let renderLanes: Lanes = NoLanes;

// 当前正在构造的fiber, 等同于 workInProgress, 为了和当前hook区分, 所以将其改名
let currentlyRenderingFiber: Fiber = (null: any);

// Hooks被存储在fiber.memoizedState 链表上
let currentHook: Hook | null = null; // currentHook = fiber(current).memoizedState

let workInProgressHook: Hook | null = null; // workInProgressHook = fiber(workInProgress).memoizedState

// 在function的执行过程中, 是否再次发起了更新. 只有function被完全执行之后才会重置.
// 当render异常时, 通过该变量可以决定是否清除render过程中的更新.
let didScheduleRenderPhaseUpdate: boolean = false;

// 在本次function的执行过程中, 是否再次发起了更新. 每一次调用function都会被重置
let didScheduleRenderPhaseUpdateDuringThisPass: boolean = false;

// 在本次function的执行过程中, 重新发起更新的最大次数
const RE_RENDER_LIMIT = 25;
```

1. currentlyRenderingFiber: 当前正在构造的 fiber, 等同于 workInProgress
2. currentHook 与 workInProgressHook: 分别指向current.memoizedState和workInProgress.memoizedState

#### renderWithHooks 函数

```js
// ...省略无关代码
export function renderWithHooks<Props, SecondArg>(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: (p: Props, arg: SecondArg) => any,
  props: Props,
  secondArg: SecondArg,
  nextRenderLanes: Lanes,
): any {
  // --------------- 1. 设置全局变量 -------------------
  renderLanes = nextRenderLanes; // 当前渲染优先级
  currentlyRenderingFiber = workInProgress; // 当前fiber节点, 也就是function组件对应的fiber节点

  // 清除当前fiber的遗留状态
  workInProgress.memoizedState = null;
  workInProgress.updateQueue = null;
  workInProgress.lanes = NoLanes;

  // --------------- 2. 调用function,生成子级ReactElement对象 -------------------
  // 指定dispatcher, 区分mount和update
  ReactCurrentDispatcher.current =
    current === null || current.memoizedState === null
      ? HooksDispatcherOnMount
      : HooksDispatcherOnUpdate;
  // 执行function函数, 其中进行分析Hooks的使用
  let children = Component(props, secondArg);

  // --------------- 3. 重置全局变量,并返回 -------------------
  // 执行function之后, 还原被修改的全局变量, 不影响下一次调用
  renderLanes = NoLanes;
  currentlyRenderingFiber = (null: any);

  currentHook = null;
  workInProgressHook = null;
  didScheduleRenderPhaseUpdate = false;

  return children;
}
```

1. useState, useEffect在fiber初次构造时分别对应mountState和mountEffect->mountEffectImpl

```js
function mountState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  const hook = mountWorkInProgressHook();
  // ...
  return [hook.memoizedState, dispatch];
}

function mountEffectImpl(fiberFlags, hookFlags, create, deps): void {
  const hook = mountWorkInProgressHook();
  // ...
```

无论useState, useEffect, 内部都通过mountWorkInProgressHook创建一个 hook.

#### 链表存储

```js
function mountWorkInProgressHook(): Hook {
  const hook: Hook = {
    memoizedState: null,
    baseState: null,
    baseQueue: null,
    queue: null,
    next: null,
  };

  if (workInProgressHook === null) {
    // 链表中首个hook
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {
    // 将hook添加到链表末尾
    workInProgressHook = workInProgressHook.next = hook;
  }
  return workInProgressHook;
}
```

#### 顺序克隆

fiber树构造(对比更新)阶段, 执行**updateFunctionComponent->renderWithHooks**时再次调用function
注意: 在renderWithHooks函数中已经设置了workInProgress.memoizedState = null, 等待调用function时重新设置.
useState, useEffect在fiber对比更新时分别对应updateState->updateReducer和updateEffect->updateEffectImpl

```js
// ----- 状态Hook --------
function updateReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
  const hook = updateWorkInProgressHook();
  // ...省略部分本节不讨论
}

// ----- 副作用Hook --------
function updateEffectImpl(fiberFlags, hookFlags, create, deps): void {
  const hook = updateWorkInProgressHook();
  // ...省略部分本节不讨论
}
```

无论useState, useEffect, 内部调用updateWorkInProgressHook获取一个 hook.

```js
function updateWorkInProgressHook(): Hook {
  // 1. 移动currentHook指针
  let nextCurrentHook: null | Hook;
  if (currentHook === null) {
    const current = currentlyRenderingFiber.alternate;
    if (current !== null) {
      nextCurrentHook = current.memoizedState;
    } else {
      nextCurrentHook = null;
    }
  } else {
    nextCurrentHook = currentHook.next;
  }

  // 2. 移动workInProgressHook指针
  let nextWorkInProgressHook: null | Hook;
  if (workInProgressHook === null) {
    nextWorkInProgressHook = currentlyRenderingFiber.memoizedState;
  } else {
    nextWorkInProgressHook = workInProgressHook.next;
  }

  if (nextWorkInProgressHook !== null) {
    // 渲染时更新: 本节不讨论
  } else {
    currentHook = nextCurrentHook;
    // 3. 克隆currentHook作为新的workInProgressHook.
    // 随后逻辑与mountWorkInProgressHook一致
    const newHook: Hook = {
      memoizedState: currentHook.memoizedState,

      baseState: currentHook.baseState,
      baseQueue: currentHook.baseQueue,
      queue: currentHook.queue,

      next: null, // 注意next指针是null
    };
    if (workInProgressHook === null) {
      currentlyRenderingFiber.memoizedState = workInProgressHook = newHook;
    } else {
      workInProgressHook = workInProgressHook.next = newHook;
    }
  }
  return workInProgressHook;
}
```

updateWorkInProgressHook函数逻辑简单: 目的是为了让currentHook和workInProgressHook两个指针同时向后移动.

1. 由于renderWithHooks函数设置了workInProgress.memoizedState=null, 所以workInProgressHook初始值必然为null, 只能从currentHook克隆.
2. 而从currentHook克隆而来的newHook.next=null, 进而导致workInProgressHook链表需要完全重建.
3. 这个克隆是按照原顺来Hook链表的顺序克隆的,这也是为什么hook可以保存更新后的状态.

## Hook原理(状态Hook)

首先回顾一下前文Hook 原理(概览), 其主要内容有:

1. function类型的fiber节点, 它的处理函数是updateFunctionComponent, 其中再通过renderWithHooks调用function.
2. 在function中, 通过Hook Api(如: useState, useEffect)创建Hook对象.

* 状态Hook实现了状态持久化(等同于class组件维护fiber.memoizedState).
* 副作用Hook则实现了维护fiber.flags,并提供副作用回调(类似于class组件的生命周期回调)
  多个【Hook对象构成一个链表结构, 并挂载到fiber.memoizedState之上.
  fiber树更新阶段, 把current.memoizedState链表上的所有Hook按照顺序克隆到workInProgress.memoizedState上, 实现数据的持久化.
  在此基础之上, 本节将深入分析状态Hook的特性和实现原理.

#### 创建Hook

在fiber初次构造阶段, useState对应源码mountState, useReducer对应源码mountReducer

```js
function mountState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  // 1. 创建hook
  const hook = mountWorkInProgressHook();
  if (typeof initialState === 'function') {
    initialState = initialState();
  }
  // 2. 初始化hook的属性
  // 2.1 设置 hook.memoizedState/hook.baseState
  // 2.2 设置 hook.queue
  hook.memoizedState = hook.baseState = initialState;
  const queue = (hook.queue = {
    pending: null,
    dispatch: null,
    // queue.lastRenderedReducer是内置函数
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: (initialState: any),
  });
  // 2.3 设置 hook.dispatch
  const dispatch: Dispatch<
    BasicStateAction<S>,
  > = (queue.dispatch = (dispatchAction.bind(
    null,
    currentlyRenderingFiber,
    queue,
  ): any));

  // 3. 返回[当前状态, dispatch函数]
  return [hook.memoizedState, dispatch];
}
```

**mount Reducer**

```js
function mountReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
  // 1. 创建hook
  const hook = mountWorkInProgressHook();
  let initialState;
  if (init !== undefined) {
    initialState = init(initialArg);
  } else {
    initialState = ((initialArg: any): S);
  }
  // 2. 初始化hook的属性
  // 2.1 设置 hook.memoizedState/hook.baseState
  hook.memoizedState = hook.baseState = initialState;
  // 2.2 设置 hook.queue
  const queue = (hook.queue = {
    pending: null,
    dispatch: null,
    // queue.lastRenderedReducer是由外传入
    lastRenderedReducer: reducer,
    lastRenderedState: (initialState: any),
  });
  // 2.3 设置 hook.dispatch
  const dispatch: Dispatch<A> = (queue.dispatch = (dispatchAction.bind(
    null,
    currentlyRenderingFiber,
    queue,
  ): any));

  // 3. 返回[当前状态, dispatch函数]
  return [hook.memoizedState, dispatch];
}
```

mountState和mountReducer逻辑简单: 主要负责创建hook, 初始化hook的属性, 最后返回[当前状态, dispatch函数].

唯一不同点是hook.queue.lastRenderedReducer:

1. mountState使用的是内置的**basicStateReducer**

```js
  function basicStateReducer<S>(state: S, action: BasicStateAction<S>): S {
    return typeof action === 'function' ? action(state) : action;
  }
```

2. mountReducer使用的是外部传入自定义reducer
   可见mountState是mountReducer的一种特殊情况, 即useState也是useReducer的一种特殊情况, 也是最简单的情况.
   useState可以转换成useReducer:

```js
  const [state, dispatch] = useState({ count: 0 });

  // 等价于
  const [state, dispatch] = useReducer(
    function basicStateReducer(state, action) {
      return typeof action === 'function' ? action(state) : action;
    },
    { count: 0 },
  );

  // 当需要更新state时, 有2种方式
  dispatch({ count: 1 }); // 1.直接设置
  dispatch(state => ({ count: state.count + 1 })); // 2.通过回调函数设置

```

  **userReducer的官网示例**

```js
  const [state, dispatch] = useReducer(
    function reducer(state, action) {
      switch (action.type) {
        case 'increment':
          return { count: state.count + 1 };
        case 'decrement':
          return { count: state.count - 1 };
        default:
          throw new Error();
      }
    },
    { count: 0 },
  );

  // 当需要更新state时, 只有1种方式
  dispatch({ type: 'decrement' });
```

  可见, useState就是对useReducer的基本封装, 内置了一个特殊的reducer(后文不再区分useState, useReducer, 都以useState为例).创建hook之后返回值[hook.memoizedState, dispatch]中的dispatch实际上会调用reducer函数.

#### 状态初始化

  在useState(initialState)函数内部, 设置hook.memoizedState = hook.baseState = initialState;, 初始状态被同时保存到了hook.baseState,hook.memoizedState中.

1. hook.memoizedState: 当前状态
2. hook.baseState: 基础状态, 作为合并hook.baseQueue的初始值(下文介绍).
   最后返回[hook.memoizedState, dispatch], 所以在function中使用的是hook.memoizedState.

#### 状态更新(dispatchAction = 上文返回的dispatch)

```js
  function dispatchAction<S, A>(
    fiber: Fiber,
    queue: UpdateQueue<S, A>,
    action: A,
  ) {
    // 1. 创建update对象
    const eventTime = requestEventTime();
    const lane = requestUpdateLane(fiber); // Legacy模式返回SyncLane
    const update: Update<S, A> = {
      lane,
      action,
      eagerReducer: null,
      eagerState: null,
      next: (null: any),
    };

    // 2. 将update对象添加到hook.queue.pending队列
    const pending = queue.pending;
    if (pending === null) {
      // 首个update, 创建一个环形链表
      update.next = update;
    } else {
      update.next = pending.next;
      pending.next = update;
    }
    queue.pending = update;

    const alternate = fiber.alternate;
    if (
      fiber === currentlyRenderingFiber ||
      (alternate !== null && alternate === currentlyRenderingFiber)
    ) {
      // 渲染时更新, 做好全局标记
      didScheduleRenderPhaseUpdateDuringThisPass = didScheduleRenderPhaseUpdate = true;
    } else {
      // ...省略性能优化部分, 下文介绍

      // 3. 发起调度更新, 进入`reconciler 运作流程`中的输入阶段.
      scheduleUpdateOnFiber(fiber, lane, eventTime);
    }
  }
```

逻辑十分清晰：

1. 创建update对象, 其中update.lane代表优先级
2. 将update对象添加到hook.queue.pending环形链表.

- 环形链表的特征: 为了方便添加新元素和快速拿到队首元素(都是O(1)), 所以pending指针指向了链表中最后一个元素.

3. 发起调度更新: 调用scheduleUpdateOnFiber, 进入reconciler 运作流程中的输入阶段.

在fiber树构造(对比更新)过程中，再次调用function，这时useState对应的函数是updateState

```js
function updateState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  return updateReducer(basicStateReducer, (initialState: any));
}
```

实际调用updateReducer

```js
function updateReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
  // 1. 获取workInProgressHook对象
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;
  queue.lastRenderedReducer = reducer;
  const current: Hook = (currentHook: any);
  let baseQueue = current.baseQueue;

  // 2. 链表拼接: 将 hook.queue.pending 拼接到 current.baseQueue
  const pendingQueue = queue.pending;
  if (pendingQueue !== null) {
    if (baseQueue !== null) {
      const baseFirst = baseQueue.next;
      const pendingFirst = pendingQueue.next;
      baseQueue.next = pendingFirst;
      pendingQueue.next = baseFirst;
    }
    current.baseQueue = baseQueue = pendingQueue;
    queue.pending = null;
  }
  // 3. 状态计算
  if (baseQueue !== null) {
    const first = baseQueue.next;
    let newState = current.baseState;

    let newBaseState = null;
    let newBaseQueueFirst = null;
    let newBaseQueueLast = null;
    let update = first;

    do {
      const updateLane = update.lane;
      // 3.1 优先级提取update
      if (!isSubsetOfLanes(renderLanes, updateLane)) {
        // 优先级不够: 加入到baseQueue中, 等待下一次render
        const clone: Update<S, A> = {
          lane: updateLane,
          action: update.action,
          eagerReducer: update.eagerReducer,
          eagerState: update.eagerState,
          next: (null: any),
        };
        if (newBaseQueueLast === null) {
          newBaseQueueFirst = newBaseQueueLast = clone;
          newBaseState = newState;
        } else {
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }
        currentlyRenderingFiber.lanes = mergeLanes(
          currentlyRenderingFiber.lanes,
          updateLane,
        );
        markSkippedUpdateLanes(updateLane);
      } else {
        // 优先级足够: 状态合并
        if (newBaseQueueLast !== null) {
          // 更新baseQueue
          const clone: Update<S, A> = {
            lane: NoLane,
            action: update.action,
            eagerReducer: update.eagerReducer,
            eagerState: update.eagerState,
            next: (null: any),
          };
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }
        if (update.eagerReducer === reducer) {
          // 性能优化: 如果存在 update.eagerReducer, 直接使用update.eagerState.避免重复调用reducer
          newState = ((update.eagerState: any): S);
        } else {
          const action = update.action;
          // 调用reducer获取最新状态
          newState = reducer(newState, action);
        }
      }
      update = update.next;
    } while (update !== null && update !== first);

    // 3.2. 更新属性
    if (newBaseQueueLast === null) {
      newBaseState = newState;
    } else {
      newBaseQueueLast.next = (newBaseQueueFirst: any);
    }
    if (!is(newState, hook.memoizedState)) {
      markWorkInProgressReceivedUpdate();
    }
    // 把计算之后的结果更新到workInProgressHook上
    hook.memoizedState = newState;
    hook.baseState = newBaseState;
    hook.baseQueue = newBaseQueueLast;
    queue.lastRenderedState = newState;
  }

  const dispatch: Dispatch<A> = (queue.dispatch: any);
  return [hook.memoizedState, dispatch];
}

```

**updateReducer**函数逻辑如下

1. 调用updateWorkInProgressHook获取workInProgressHook对象
2. 链表拼接: 将 hook.queue.pending 拼接到 current.baseQueue
3. 状态计算**优先级只有在开启了Concurrent模式下才会计算,关于concurrent相关,请查看系列文章**
   update优先级不够：加入到baseQueue中，等待下一次render
   update优先级足够：状态合并更新属性

#### 性能优化

dispatchAction函数中, 在调用scheduleUpdateOnFiber之前, 针对update对象做了性能优化.

1. queue.pending中只包含当前update时(只有一个udpate对象)，即当前update是queue.pending中的第一个update，直接调用queue.lastRenderReducer，计算出update之后的state记为eagerState
2. 如果eagerState与currentState相同, 则直接退出, 不用发起调度更新.
3. 已经被挂载到queue.pending上的update会在下一次render时再次合并.

```js
function dispatchAction<S, A>(
  fiber: Fiber,
  queue: UpdateQueue<S, A>,
  action: A,
) {
  // ...省略无关代码 ...只保留性能优化部分代码:

  // 下面这个if判断, 能保证当前创建的update, 是`queue.pending`中第一个`update`. 为什么? 发起更新之后fiber.lanes会被改动(可以回顾`fiber 树构造(对比更新)`章节), 如果`fiber.lanes && alternate.lanes`没有被改动, 自然就是首个update
  if (
    fiber.lanes === NoLanes &&
    (alternate === null || alternate.lanes === NoLanes)
  ) {
    const lastRenderedReducer = queue.lastRenderedReducer;
    if (lastRenderedReducer !== null) {
      let prevDispatcher;
      const currentState: S = (queue.lastRenderedState: any);
      const eagerState = lastRenderedReducer(currentState, action);
      // 暂存`eagerReducer`和`eagerState`, 如果在render阶段reducer==update.eagerReducer, 则可以直接使用无需再次计算
      update.eagerReducer = lastRenderedReducer;
      update.eagerState = eagerState;
      if (is(eagerState, currentState)) {
        // 快速通道, eagerState与currentState相同, 无需调度更新
        // 注: update已经被添加到了queue.pending, 并没有丢弃. 之后需要更新的时候, 此update还是会起作用
        return;
      }
    }
  }
  // 发起调度更新, 进入`reconciler 运作流程`中的输入阶段.
  scheduleUpdateOnFiber(fiber, lane, eventTime);
}
```

#### 异步更新

1. Legacy模式下，均为同步更新，所以update对象会被全量合并,hook.baseQeue和hook.baseState并没有起到实质性作用
2. 17.x 版本中没有Concurrent模式的入口，即将发布的v18.x版本将全面进入异步时代

## Hook原理(副作用Hook)

重点讨论useEffect, useLayoutEffect等标准的副作用Hook.

### 创建Hook

在fiber初次构造阶段,useEffect对应远吗mountEffect,useLayoutEffect对应源码mountLayoutEffect

**mountEffect**

```js
function mountEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  return mountEffectImpl(
    UpdateEffect | PassiveEffect, // fiberFlags
    HookPassive, // hookFlags
    create,
    deps,
  );
}
```

**mountLayoutEffect**

```js
function mountLayoutEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  return mountEffectImpl(
    UpdateEffect, // fiberFlags
    HookLayout, // hookFlags
    create,
    deps,
  );
}

```

可见mountEffect和mountLayoutEffect内部都直接调用mountEffectImpl, 只是参数不同.

```js
function mountEffectImpl(fiberFlags, hookFlags, create, deps): void {
  // 1. 创建hook
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  // 2. 设置workInProgress的副作用标记
  currentlyRenderingFiber.flags |= fiberFlags; // fiberFlags 被标记到workInProgress
  // 2. 创建Effect, 挂载到hook.memoizedState上
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags, // hookFlags用于创建effect
    create,
    undefined,
    nextDeps,
  );
}
```

**mountEffectImpl逻辑**:

1. 创建hook
2. 设置workInProgress的副作用标记: flags |= fiberFlags
3. 创建effect(在pushEffect中), 挂载到hook.memoizedState上, 即 hook.memoizedState = effect
   注意: 状态Hook中hook.memoizedState = state

#### 创建Effect

**pushEffect**

```javascript
function pushEffect(tag, create, destroy, deps) {
  // 1. 创建effect对象
  const effect: Effect = {
    tag,
    create,
    destroy,
    deps,
    next: (null: any),
  };
  // 2. 把effect对象添加到环形链表末尾
  let componentUpdateQueue: null | FunctionComponentUpdateQueue = (currentlyRenderingFiber.updateQueue: any);
  if (componentUpdateQueue === null) {
    // 新建 workInProgress.updateQueue 用于挂载effect对象
    componentUpdateQueue = createFunctionComponentUpdateQueue();
    currentlyRenderingFiber.updateQueue = (componentUpdateQueue: any);
    // updateQueue.lastEffect是一个环形链表
    componentUpdateQueue.lastEffect = effect.next = effect;
  } else {
    const lastEffect = componentUpdateQueue.lastEffect;
    if (lastEffect === null) {
      componentUpdateQueue.lastEffect = effect.next = effect;
    } else {
      const firstEffect = lastEffect.next;
      lastEffect.next = effect;
      effect.next = firstEffect;
      componentUpdateQueue.lastEffect = effect;
    }
  }
  // 3. 返回effect
  return effect;
}
```

**pushEffect逻辑**

1. 创建effect
2. 把effect对象添加到环形链表末尾
3. 返回effect

effect数据结构

```js
export type Effect = {|
  tag: HookFlags,
  create: () => (() => void) | void,
  destroy: (() => void) | void,
  deps: Array<mixed> | null,
  next: Effect,
|};

//effect.tag:二进制属性，代表effect的类型
export const NoFlags = /*  */ 0b000;
export const HasEffect = /* */ 0b001; // 有副作用, 可以被触发
export const Layout = /*    */ 0b010; // Layout, dom突变后同步触发
export const Passive = /*   */ 0b100; // Passive, dom突变前异步触发
```

* effect.create:实际上就是通过useEffect()所传入的函数
* effect.deps:依赖项，如果依赖项变动，会创建新的effect.
  现在workInProgress.flags被打上了标记, 最后会在fiber树渲染阶段的commitRoot函数中处理.

#### useEffect & useLayoutEffect

站在fiber,hook,effect的视角, 无需关心这个hook是通过useEffect还是useLayoutEffect创建的. 只需要关心内部fiber.flags,effect.tag的状态.

所以useEffect与useLayoutEffect的区别如下:

1. fiber.flags不同

* 使用useEffect时: fiber.flags = UpdateEffect | PassiveEffect.
* 使用useLayoutEffect时: fiber.flags = UpdateEffect.

2. effect.tag不同

* 使用useEffect时: effect.tag = HookHasEffect | HookPassive.
* 使用useLayoutEffect时: effect.tag = HookHasEffect | HookLayout.

#### 处理Effect回调

完成fiber树构造后, 逻辑会进入渲染阶段. 通过fiber 树渲染中的介绍, 在commitRootImpl函数中, 整个渲染过程被 3 个函数分布实现:

1. commitBeforeMutationEffects
2. commitMutationEffects
3. commitLayoutEffects
   这 3 个函数会处理fiber.flags, 也会根据情况处理fiber.updateQueue.lastEffect

**commitBeforeMutationEffects**
第一阶段: dom 变更之前, 处理副作用队列中带有Passive标记的fiber节点.

```js
function commitBeforeMutationEffects() {
  while (nextEffect !== null) {
    // ...省略无关代码, 只保留Hook相关

    // 处理`Passive`标记
    const flags = nextEffect.flags;
    if ((flags & Passive) !== NoFlags) {
      if (!rootDoesHavePassiveEffects) {
        rootDoesHavePassiveEffects = true;
        scheduleCallback(NormalSchedulerPriority, () => {
          flushPassiveEffects();
          return null;
        });
      }
    }
    nextEffect = nextEffect.nextEffect;
  }
}
```

注意: 由于flushPassiveEffects被包裹在scheduleCallback回调中, 由调度中心来处理, 且参数是NormalSchedulerPriority, 故这是一个异步回调
由于scheduleCallback(NormalSchedulerPriority,callback)是异步的, flushPassiveEffects并不会立即执行. 此处先跳过flushPassiveEffects的分析, 继续跟进commitRoot.

**commitMutationEffects**
第二阶段: dom 变更, 界面得到更新.

```js
function commitMutationEffects(
  root: FiberRoot,
  renderPriorityLevel: ReactPriorityLevel,
) {
  // ...省略无关代码, 只保留Hook相关
  while (nextEffect !== null) {
    const flags = nextEffect.flags;
    const primaryFlags = flags & (Placement | Update | Deletion | Hydrating);
    switch (primaryFlags) {
      case Update: {
        // useEffect,useLayoutEffect都会设置Update标记
        // 更新节点
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
    }
    nextEffect = nextEffect.nextEffect;
  }
}

function commitWork(current: Fiber | null, finishedWork: Fiber): void {
  // ...省略无关代码, 只保留Hook相关
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case MemoComponent:
    case SimpleMemoComponent:
    case Block: {
      // 在突变阶段调用销毁函数, 保证所有的effect.destroy函数都会在effect.create之前执行
      commitHookEffectListUnmount(HookLayout | HookHasEffect, finishedWork);
      return;
    }
  }
}
// 依次执行: effect.destroy
function commitHookEffectListUnmount(tag: number, finishedWork: Fiber) {
  const updateQueue: FunctionComponentUpdateQueue | null = (finishedWork.updateQueue: any);
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      if ((effect.tag & tag) === tag) {
        // 根据传入的tag过滤 effect链表.
        const destroy = effect.destroy;
        effect.destroy = undefined;
        if (destroy !== undefined) {
          destroy();
        }
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}

```

1. 调用关系: commitMutationEffects->commitWork->commitHookEffectListUnmount.
2. 注意在调用**commitMutationEffects(HookLayout | HookHasEffect, finishedWork)**时, 参数是HookLayout | HookHasEffect, 所以**只处理由useLayoutEffect()创建的effect**.
3. 同步调用effect.destroy().

Note：function组件删除也在commitMuatationEffects阶段处理

1. 当function组件被销毁时, fiber节点必然会被打上Deletion标记, 即fiber.flags |= Deletion. 带有Deletion标记的fiber在commitMutationEffects被处理
2. 在commitDeletion函数之后, 继续调用unmountHostComponents->commitUnmount, 在commitUnmount中, 执行effect.destroy(), 结束整个闭环.

#### commitLayoutEffects

第三阶段: dom 变更后

```js
function commitLayoutEffects(root: FiberRoot, committedLanes: Lanes) {
  // ...省略无关代码, 只保留Hook相关
  while (nextEffect !== null) {
    const flags = nextEffect.flags;
    if (flags & (Update | Callback)) {
      // useEffect,useLayoutEffect都会设置Update标记
      const current = nextEffect.alternate;
      commitLayoutEffectOnFiber(root, current, nextEffect, committedLanes);
    }
    nextEffect = nextEffect.nextEffect;
  }
}

function commitLifeCycles(
  finishedRoot: FiberRoot,
  current: Fiber | null,
  finishedWork: Fiber,
  committedLanes: Lanes,
): void {
  // ...省略无关代码, 只保留Hook相关
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case SimpleMemoComponent:
    case Block: {
      // 在此之前commitMutationEffects函数中, effect.destroy已经被调用, 所以effect.destroy永远不会影响到effect.create
      //finishedWork实际上指代当前正在被遍历的有副作用的fiber
      commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork);

      schedulePassiveEffects(finishedWork);
      return;
    }
  }
}

function commitHookEffectListMount(tag: number, finishedWork: Fiber) {
  const updateQueue: FunctionComponentUpdateQueue | null = (finishedWork.updateQueue: any);
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      if ((effect.tag & tag) === tag) {
        const create = effect.create;
        //调用effect.create()之后, 将返回值赋值到effect.destroy.
        effect.destroy = create();
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}

```

1. 调用关系: commitLayoutEffects->commitLayoutEffectOnFiber(commitLifeCycles)->commitHookEffectListMount.

- 注意在调用commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork)时, 参数是HookLayout | HookHasEffect,所以只处理由useLayoutEffect()创建的effect.
- 调用effect.create()之后, 将返回值赋值到effect.destroy.

2. 为flushPassiveEffects做准备

- commitLifeCycles中的schedulePassiveEffects(finishedWork), 其形参finishedWork实际上指代当前正在被遍历的有副作用的fiber
- schedulePassiveEffects比较简单, 就是把带有Passive标记的effect筛选出来(由useEffect创建), 添加到一个全局数组(pendingPassiveHookEffectsUnmount和pendingPassiveHookEffectsMount).

```js
function schedulePassiveEffects(finishedWork: Fiber) {
  // 1. 获取 fiber.updateQueue
  const updateQueue: FunctionComponentUpdateQueue | null = (finishedWork.updateQueue: any);
  // 2. 获取 effect环形队列
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      const { next, tag } = effect;
      // 3. 筛选出由useEffect()创建的`effect`
      if (
        (tag & HookPassive) !== NoHookEffect &&
        (tag & HookHasEffect) !== NoHookEffect
      ) {
        // 把effect添加到全局数组, 等待`flushPassiveEffects`处理
        enqueuePendingPassiveHookEffectUnmount(finishedWork, effect);
        enqueuePendingPassiveHookEffectMount(finishedWork, effect);
      }
      effect = next;
    } while (effect !== firstEffect);
  }
}

export function enqueuePendingPassiveHookEffectUnmount(
  fiber: Fiber,
  effect: HookEffect,
): void {
  // unmount effects 数组
  pendingPassiveHookEffectsUnmount.push(effect, fiber);
}

export function enqueuePendingPassiveHookEffectMount(
  fiber: Fiber,
  effect: HookEffect,
): void {
  // mount effects 数组
  pendingPassiveHookEffectsMount.push(effect, fiber);
}
```

综上commitMutationEffects和commitLayoutEffects2 个函数, 带有Layout标记的effect(由useLayoutEffect创建), 已经得到了完整的回调处理(destroy和create已经被调用).

#### flushPassiveEffects

在上文commitBeforeMutationEffects阶段, 异步调用了flushPassiveEffects. 在这期间带有Passive标记的effect已经被添加到pendingPassiveHookEffectsUnmount和pendingPassiveHookEffectsMount全局数组中.
接下来flushPassiveEffects就可以脱离fiber节点, 直接访问effects

```js
export function flushPassiveEffects(): boolean {
  // Returns whether passive effects were flushed.
  if (pendingPassiveEffectsRenderPriority !== NoSchedulerPriority) {
    const priorityLevel =
      pendingPassiveEffectsRenderPriority > NormalSchedulerPriority
        ? NormalSchedulerPriority
        : pendingPassiveEffectsRenderPriority;
    pendingPassiveEffectsRenderPriority = NoSchedulerPriority;
    // `runWithPriority`设置Schedule中的调度优先级, 如果在flushPassiveEffectsImpl中处理effect时又发起了新的更新, 那么新的update.lane将会受到这个priorityLevel影响.（会抛弃上次的更新）
    return runWithPriority(priorityLevel, flushPassiveEffectsImpl);
  }
  return false;
}

// ...省略无关代码, 只保留Hook相关
function flushPassiveEffectsImpl() {
  if (rootWithPendingPassiveEffects === null) {
    return false;
  }
  rootWithPendingPassiveEffects = null;
  pendingPassiveEffectsLanes = NoLanes;

  // 1. 执行 effect.destroy()
  const unmountEffects = pendingPassiveHookEffectsUnmount;
  pendingPassiveHookEffectsUnmount = [];
  for (let i = 0; i < unmountEffects.length; i += 2) {
    const effect = ((unmountEffects[i]: any): HookEffect);
    const fiber = ((unmountEffects[i + 1]: any): Fiber);
    const destroy = effect.destroy;
    effect.destroy = undefined;
    if (typeof destroy === 'function') {
      destroy();
    }
  }

  // 2. 执行新 effect.create(), 重新赋值到 effect.destroy
  const mountEffects = pendingPassiveHookEffectsMount;
  pendingPassiveHookEffectsMount = [];
  for (let i = 0; i < mountEffects.length; i += 2) {
    const effect = ((mountEffects[i]: any): HookEffect);
    const fiber = ((mountEffects[i + 1]: any): Fiber);
    effect.destroy = create();
  }
}
```

核心逻辑

1. 遍历pendingPassiveHookEffectsUnmount中的所有effect, 调用effect.destroy().

- 同时清空pendingPassiveHookEffectsUnmount

2. 遍历pendingPassiveHookEffectsMount中的所有effect, 调用effect.create(), 并更新effect.destroy.

- 同时清空pendingPassiveHookEffectsMount
  所以, 带有Passive标记的effect, 在flushPassiveEffects函数中得到了完整的回调处理.所以effect的destroy会在下一次更新之前调用，

#### 更新Hook

假设在初次调用之后, 发起更新, 会再次执行function, 这时function只使用的useEffect, useLayoutEffect等api也会再次执行.

在更新过程中useEffect对应源码updateEffect, useLayoutEffect对应源码updateLayoutEffect.它们内部都会调用updateEffectImpl, 与初次创建时一样, 只是参数不同.

##### 更新Effect

```js
function updateEffectImpl(fiberFlags, hookFlags, create, deps): void {
  // 1. 获取当前hook
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  let destroy = undefined;
  // 2. 分析依赖
  if (currentHook !== null) {
    const prevEffect = currentHook.memoizedState;
    // 继续使用先前effect.destroy
    destroy = prevEffect.destroy;
    if (nextDeps !== null) {
      const prevDeps = prevEffect.deps;
      // 比较依赖是否变化
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // 2.1 如果依赖不变, 新建effect(tag不含HookHasEffect),
        pushEffect(hookFlags, create, destroy, nextDeps);
        return;
      }
    }
  }
  // 2.2 如果依赖改变, 更改fiber.flag, 新建effect
  currentlyRenderingFiber.flags |= fiberFlags;

  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,
    create,
    destroy,
    nextDeps,
  );
}
```

updateEffectImpl与mountEffectImpl逻辑有所不同: - 如果useEffect/useLayoutEffect的**依赖不变**, 新建的effect对象**不带**HasEffect标记. 因为不带HasEffect标记,所以在commitRoot阶段不会执行**effect.create()**.
注意: 几个关键词在这里,更新的时候会**新建新的Effect**,无论依赖是否变化, 都**复用**之前的effect.destroy. 等待commitRoot阶段的调用(上文已经说明).

##### 处理Effect回调

新的hook以及新的effect创建完成之后, 余下逻辑与初次渲染完全一致. 处理 Effect 回调时也会根据effect.tag进行判断: 只有effect.tag包含HookHasEffect时才会调用effect.destroy和effect.create()

##### 组建销毁

当function组件被销毁时, fiber节点必然会被打上Deletion标记, 即fiber.flags |= Deletion. **带有Deletion标记的fiber在commitMutationEffects被处理**:

```js
// ...省略无关代码
function commitMutationEffects(
  root: FiberRoot,
  renderPriorityLevel: ReactPriorityLevel,
) {
  while (nextEffect !== null) {
    const primaryFlags = flags & (Placement | Update | Deletion | Hydrating);
    switch (primaryFlags) {
      case Deletion: {
        commitDeletion(root, nextEffect, renderPriorityLevel);
        break;
      }
    }
  }
}
```

在commitDeletion函数之后, 继续调用unmountHostComponents->commitUnmount, 在commitUnmount中, 执行effect.destroy(), 结束整个闭环.

![img](https://raw.githubusercontent.com/captain1023/picGo/master/img/React%E6%A6%82%E8%A7%88%E5%9B%BE.png)

ReferenceList:

1. https://github.com/7kms/react-illustration-series
2. https://react.iamkasong.com/preparation/idea.html#%E6%80%BB%E7%BB%9
3. https://juejin.cn/post/7085145274200358949
