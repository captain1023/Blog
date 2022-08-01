# React源码解析之Fiber对象

说到React就难免会提到React的fiber架构，本系列行文思路如下,本篇属于`Fiber架构中的常见对象说明`

* [X] React启动过程
* [X] React的两大工作循环
* [X] React中的对象
* [ ] React fiber的初次创建与更新
* [ ] React fiber的渲染
* [ ] React的管理员(reconciler运行循环)
* [ ] react的优先级管理(Lane模型)

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

## React的高频对象

### ReactElement, Fiber, DOM 三者的关系

- 转换流程:ReactElement(JSX)->Fiber->DOM
- 开发人员能够控制的是JSX, 也就是ReactElement对象.
- fiber树是通过ReactElement生成的, 如果脱离了ReactElement,fiber树也无从谈起. 所以是ReactElement树(不是严格的树结构, 为了方便也称为树)驱动fiber树.
- fiber树是DOM树的数据模型, fiber树驱动DOM树

### react包

#### ReactElement对象

`ReactElement`对象数据结构如下

```js
export type ReactElement = {|
  // 用于辨别ReactElement对象
  $$typeof: any,

  // 内部属性
  type: any, // 表明其种类
  key: any, //用于reconciler的diff算法比较
  ref: any,
  props: any, //props.children = [],用来连接ReactElement节点

  // ReactFiber 记录创建本对象的Fiber节点, 还未与Fiber树关联之前, 该属性为null
  _owner: any,

  // __DEV__ dev环境下的一些额外信息, 如文件路径, 文件名, 行列信息等
  _store: {validated: boolean, ...},
  _self: React$Element<any>,
  _shadowChildren: any,
  _source: Source,
|};
```

1. key属性在reconciler阶段会用到, 目前只需要知道所有的ReactElement对象都有 key 属性(且其默认值是 null, 这点十分重要, 在 diff 算法中会使用到).
2. type属性决定了节点的种类:
   - 它的值可以是字符串(代表div,span等 dom 节点), 函数(代表function, class等节点), 或者 react 内部定义的节点类型(portal,context,fragment等)
   - 在reconciler阶段, 会根据 type 执行不同的逻辑(在 fiber 构建阶段详细解读).
     - 如 type 是一个字符串类型, 则直接使用.
     - 如 type 是一个ReactComponent类型, 则会调用其 render 方法获取子节点.
     - 如 type 是一个function类型,则会调用该方法获取子节点

#### React Component对象

- 对于ReactElement来讲, ReactComponent仅仅是诸多type类型中的一种
- ReactComponent是 class 类型, 继承父类Component, 拥有特殊的方法(setState,forceUpdate)和特殊的属性(context,updater等).
- 在reconciler阶段, 会依据ReactElement对象的特征, 生成对应的 fiber 节点. 当识别到ReactElement对象是 class 类型的时候, 会触发ReactComponent对象的生命周期, 并调用其 render方法, 生成ReactElement子节点.

#### 注意

- class和function类型的组件，其子节点是在render之后(reconciler阶段)才生成的.
- 父级对象和子级对象之间是通过 `props.children`属性进行关联的(与fiber树不同)
- reactElement虽然不能算是一个严格的树,也不能算是一个严格的链表.它的生成过程是至顶向下的，是所有组件节点的总和.
- ReactElement树和fiber树是以 `props.children为单位先后交替生成`的,当React Element树构造完毕,fiber树也随后构造完毕.
- reconciler阶段会根据React Element的类型生成对应的fiber节点(不是一一对应,比如Fragment类型的组件在生成fiber节点的时候会略过)

### react-reconciler包

#### fiber对象

```js
// 一个Fiber对象代表一个即将渲染或者已经渲染的组件(ReactElement), 一个组件可能对应两个fiber(current和WorkInProgress)
export type Fiber = {|
  tag: WorkTag,
  key: null | string,
  elementType: any,
  type: any,
  stateNode: any,
  return: Fiber | null,
  child: Fiber | null,
  sibling: Fiber | null,
  index: number,
  ref:
    | null
    | (((handle: mixed) => void) & { _stringRef: ?string, ... })
    | RefObject,
  pendingProps: any, // 从`ReactElement`对象传入的 props. 用于和`fiber.memoizedProps`比较可以得出属性是否变动
  memoizedProps: any, // 上一次生成子节点时用到的属性, 生成子节点之后保持在内存中
  updateQueue: mixed, // 存储state更新的队列, 当前节点的state改动之后, 都会创建一个update对象添加到这个队列中.
  memoizedState: any, // 用于输出的state, 最终渲染所使用的state
  dependencies: Dependencies | null, // 该fiber节点所依赖的(contexts, events)等
  mode: TypeOfMode, // 二进制位Bitfield,继承至父节点,影响本fiber节点及其子树中所有节点. 与react应用的运行模式有关(有ConcurrentMode, BlockingMode, NoMode等选项).

  // Effect 副作用相关
  flags: Flags, // 标志位
  subtreeFlags: Flags, //替代16.x版本中的 firstEffect, nextEffect. 当设置了 enableNewReconciler=true才会启用
  deletions: Array<Fiber> | null, // 存储将要被删除的子节点. 当设置了 enableNewReconciler=true才会启用

  nextEffect: Fiber | null, // 单向链表, 指向下一个有副作用的fiber节点
  firstEffect: Fiber | null, // 指向副作用链表中的第一个fiber节点
  lastEffect: Fiber | null, // 指向副作用链表中的最后一个fiber节点

  // 优先级相关
  lanes: Lanes, // 本fiber节点的优先级
  childLanes: Lanes, // 子节点的优先级
  alternate: Fiber | null, // 指向内存中的另一个fiber, 每个被更新过fiber节点在内存中都是成对出现(current和workInProgress)

  // 性能统计相关(开启enableProfilerTimer后才会统计)
  // react-dev-tool会根据这些时间统计来评估性能
  actualDuration?: number, // 本次更新过程, 本节点以及子树所消耗的总时间
  actualStartTime?: number, // 标记本fiber节点开始构建的时间
  selfBaseDuration?: number, // 用于最近一次生成本fiber节点所消耗的时间
  treeBaseDuration?: number, // 生成子树所消耗的时间的总和
|};
```

- fiber.tag: 表示 fiber 类型, 根据ReactElement组件的 type 进行生成, 在 react 内部共定义了25 种 tag.
- fiber.key: 和ReactElement组件的 key 一致.
- fiber.elementType: 一般来讲和ReactElement组件的 type 一致
- fiber.type: 一般来讲和fiber.elementType一致. 一些特殊情形下, 比如在开发环境下为了兼容热更新(HotReloading), 会对function, class, ForwardRef类型的ReactElement做一定的处理, 这种情况会区别于fiber.elementType, 具体赋值关系可以查看源文件.
- fiber.stateNode: 与fiber关联的局部状态节点(比如: HostComponent类型指向与fiber节点对应的 dom 节点; 根节点fiber.stateNode指向的是FiberRoot; class 类型节点其stateNode指向的是 class 实例).
- fiber.return: 指向父节点.
- fiber.child: 指向第一个子节点.
- fiber.sibling: 指向下一个兄弟节点.
- fiber.index: fiber 在兄弟节点中的索引, 如果是单节点默认为 0.
- fiber.ref: 指向在ReactElement组件上设置的 ref(string类型的ref除外, 这种类型的ref已经不推荐使用, reconciler阶段会将string类型的ref转换成一个function类型).
- fiber.pendingProps: 输入属性, 从ReactElement对象传入的 props. 用于和fiber.memoizedProps比较可以得出属性是否变动.
- fiber.memoizedProps: 上一次生成子节点时用到的属性, 生成子节点之后保持在内存中. 向下生成子节点之前叫做pendingProps, 生成子节点之后会把pendingProps赋值给memoizedProps用于下一次比较.pendingProps和memoizedProps比较可以得出属性是否变动.
- fiber.updateQueue: 存储update更新对象的队列, 每一次发起更新, 都需要在该队列上创建一个update对象.
- fiber.memoizedState: 上一次生成子节点之后保持在内存中的局部状态.
- fiber.dependencies: 该 fiber 节点所依赖的(contexts, events)等, 在context机制章节详细说明.
- fiber.mode: 二进制位 Bitfield,继承至父节点,影响本 fiber 节点及其子树中所有节点. 与 react 应用的运行模式有关(有 ConcurrentMode, BlockingMode, NoMode 等选项).
- fiber.flags: 标志位, 副作用标记(在 16.x 版本中叫做effectTag, 相应pr), 在ReactFiberFlags.js中定义了所有的标志位. reconciler阶段会将所有拥有flags标记的节点添加到副作用链表中, 等待 commit 阶段的处理.
- fiber.nextEffect: 单向链表, 指向下一个有副作用的 fiber 节点.
- fiber.firstEffect: 指向副作用链表中的第一个 fiber 节点.
- fiber.lastEffect: 指向副作用链表中的最后一个 fiber 节点.
- fiber.lanes: 本 fiber 节点所属的优先级, 创建 fiber 的时候设置.
- fiber.childLanes: 子节点所属的优先级.
- fiber.alternate: 指向内存中的另一个 fiber, 每个被更新过 fiber 节点在内存中都是成对出现(current 和 workInProgress)

#### 示例
```js
class App extends React.Component {
  render() {
    return (
      <div className="app">
        <header>header</header>
        <Content />
        <footer>footer</footer>
      </div>
    );
  }
}

class Content extends React.Component {
  render() {
    return (
      <React.Fragment>
        <p>1</p>
        <p>2</p>
        <p>3</p>
      </React.Fragment>
    );
  }
}
export default App;
```
**ReactElement树**
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/reactelement-tree.png)

**Fiber树**
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/fiber-tree.png)

#### update与updateQueue对象

* 在fiber对象中有一个属性fiber.updateQueue, 是一个链式队列(即使用链表实现的队列存储结构)

**Update数据结构**

```js
export type Update<State> = {|
  eventTime: number, // 发起update事件的时间(17.0.2中作为临时字段, 即将移出)
  lane: Lane, // update所属的优先级

  tag: 0 | 1 | 2 | 3, //
  payload: any, // 载荷, 根据场景可以设置成一个回调函数或者对象
  callback: (() => mixed) | null, // 回调函数

  next: Update<State> | null, // 指向链表中的下一个, 由于UpdateQueue是一个环形链表, 最后一个update.next指向第一个update对象
|};

// =============== UpdateQueue ==============
type SharedQueue<State> = {|
  pending: Update<State> | null,
|};

export type UpdateQueue<State> = {|
  baseState: State,
  firstBaseUpdate: Update<State> | null,
  lastBaseUpdate: Update<State> | null,
  shared: SharedQueue<State>,
  effects: Array<Update<State>> | null,
|};
```

整体结构:Fiber.updateQueue.shared(下面的sharedQueue).pending(update队列)，队列中包含一个个的update(下面的3)

1. UpdateQueue
   - baseState: 表示此队列的基础 state
   - firstBaseUpdate: 指向基础队列的队首
   - lastBaseUpdate: 指向基础队列的队尾
   - shared: 共享队列
   - effects: 用于保存有callback回调函数的 update 对象, 在commit之后, 会依次调用这里的回调函数.
2. SharedQueue
   - pending: 指向即将输入的update队列. 在class组件中调用setState()之后, 会将新的 update 对象添加到这个队列中来.
3. Update
   - eventTime: 发起update事件的时间(17.0.2 中作为临时字段, 即将移出)
   - lane: update所属的优先级
   - tag: 表示update种类, 共 4 种. UpdateState,ReplaceState,ForceUpdate,CaptureUpdate
   - payload: 载荷, update对象真正需要更新的数据, 可以设置成一个回调函数或者对象.
   - callback: **回调函数. commit完成之后会调用**
   - next: 指向链表中的下一个, 由于UpdateQueue是一个**环形链表**, 最后一个update.next指向第一个update对象.![](https://raw.githubusercontent.com/captain1023/picGo/master/img/updatequeue.png)

#### Hook对象

```js
export type Hook = {|
  memoizedState: any,//内存状态, 用于输出成最终的fiber树
  baseState: any,//基础状态, 当Hook.queue更新过后, baseState也会更新.
  baseQueue: Update<any, any> | null,//基础状态队列, 在reconciler阶段会辅助状态合并.
  queue: UpdateQueue<any, any> | null,//指向一个Update队列
  next: Hook | null,//指向该function组件的下一个Hook对象, 使得多个Hook之间也构成了一个链表.
|};

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
```

Hook.queue和 Hook.baseQueue(即UpdateQueue和Update）是为了保证Hook对象能够顺利更新, 与上文fiber.updateQueue中的UpdateQueue和Update是不一样的(且它们在不同的文件),

在fiber对象中有一个属性fiber.memoizedState指向fiber节点的内存状态. 在function类型的组件中, fiber.memoizedState就指向Hook队列(Hook队列保存了function类型的组件状态).

所以Hook也不能脱离fiber而存在, 它们之间的引用关系如下:
Fiber.memoizedState->Hook->next->Hook.queue->pending->update队列
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/updatequeue.png)

#### scheduler包

scheduler包负责调度, 在内部维护一个任务队列(taskQueue). 这个队列是一个最小堆数组(详见React 算法之堆排序), 其中存储了 task 对象.

```js
var newTask = {
  id: taskIdCounter++,//位移标识
  callback,//task 最核心的字段, 指向react-reconciler包所提供的回调函数.
  priorityLevel,//优先级
  startTime,//一个时间戳,代表 task 的开始时间(创建时间 + 延时时间).
  expirationTime,//过期时间.
  sortIndex: -1,//控制 task 在队列中的次序, 值越小的越靠前.
};

```

注意task中没有next属性, 它不是一个链表, 其顺序是通过堆排序来实现的(小顶堆数组, 始终保证数组中的第一个task对象优先级最高).
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/fiber-hook.png)

---

ReferenceList:

1. https://github.com/7kms/react-illustration-series
2. https://react.iamkasong.com/preparation/idea.html#%E6%80%BB%E7%BB%9
3. https://juejin.cn/post/7085145274200358949
