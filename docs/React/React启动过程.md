# React启动过程分析

## React包概览

* **react**
  react基础包，只提供定义react组件(React Element)的必要函数，一般来说需要和渲染器(react-dom,react-native)一同使用.在编写react应用的代码时，大部分都是调此包的api.
* **react-dom**
  react渲染器之一，是react与web平台连接的桥梁(可以在浏览和nodejs环境中使用),将react-reconciler中的运行结果输出到web界面上.在编写react应用的代码时，大多数场景下，能用到此包的就是一个入口函数ReactDOM.render(`<APP/>`,doucument.getElementByID('root)')，其余使用的api，基本是react包提供的
* **react- reconciler**
  react得以运行的核心包(综合协调react-dom,react,scheduler各包之前的调用与配合).管理react应用状态的输入和结果的输出.将输入信号最终转换成输出信号传递给渲染器
* **scheduler**
  调度机制的核心实现, 控制由react-reconciler送入的回调函数的执行时机, 在concurrent模式下可以实现任务分片. 在编写react应用的代码时, 同样几乎不会直接用到此包提供的 api.
  核心任务就是执行回调(回调函数由react-reconciler提供)
  通过控制回调函数的执行时机, 来达到任务分片的目的, 实现可中断渲染(concurrent模式下才有此特性)

  ![](https://raw.githubusercontent.com/captain1023/picGo/master/img/React%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B.png)

## react应用的启动过程

位于react-dom包,衔接reconciler运行流程中的输入步骤

### 启动模式

1. legacy 模式: ReactDOM.render(`<App />`, rootNode). 这是当前 React app 使用的方式. 这个模式可能不支持这些新功能(concurrent 支持的所有功能).

```js
// LegacyRoot
ReactDOM.render(<App />, document.getElementById('root'), dom => {}); // 支持callback回调, 参数是一个dom对象,TODO:这个回调是干嘛的？？😭
```

2. Blocking 模式: ReactDOM.createBlockingRoot(rootNode).render(`<App />`). 目前正在实验中, 它仅提供了 concurrent 模式的小部分功能.

```js
// BlockingRoot
// 1. 创建ReactDOMRoot对象
const reactDOMBlockingRoot = ReactDOM.createBlockingRoot(
  document.getElementById('root'),
);
// 2. 调用render
reactDOMBlockingRoot.render(<App />); // 不支持回调
```

3. Concurrent 模式: ReactDOM.createRoot(rootNode).render(`<App />`). 实现了时间切片等功能

```js
// ConcurrentRoot
// 1. 创建ReactDOMRoot对象
const reactDOMRoot = ReactDOM.createRoot(
  document.getElementById('root')
  );
// 2. 调用render
reactDOMRoot.render(<App />); // 不支持回调
```

### 启动流程

在调用入口函数之前,`reactElement(`和 DOM 对象 `div#root`之间没有关联

#### 创建全局对象

三种模式下在react初始化时，都会创建3个全局对象

1. ReactDOM(Blocking)Root对象
   1. 属于react-dom包，该对象上有render和unmount方法，通过调用render方法可以引导react应用启动
2. fiberRoot对象
   1. 属于react-reconciler包, 作为react-reconciler在运行过程中的全局上下文, 保存 fiber 构建过程中所依赖的全局状态.
   2. react利用这些信息来循环构造fiber节点(后续会详细分析fiber的构造过程)
3. HostRootFiber对象
   1. 属于react-reconciler包, 这是 react 应用中的第一个 Fiber 对象, 是 Fiber 树的根节点, 节点的类型是HostRoot.

#### 创建ReactDOM(Blocking)Root对象

先放结论:

1. 三种模式都会调用updateContainer()函数，正是这个updateContainer()函数串联了react-dom包和react-reconciler.因为updateContainer()函数中调用了scheduleUpdateOnFiber(xxx)，进入react循环构造的**唯一**入口
2. 三种模式都会创建在创建DOMroot中调用createRootImpl,区分三种模式的方式只是传递的rootTag参数不同

看到这儿是不是觉得源码好像也就这么回事😂

legacy模式

```javascript

function legacyRenderSubtreeIntoContainer(
  parentComponent: ?React$Component<any, any>,
  children: ReactNodeList,
  container: Container,
  forceHydrate: boolean,
  callback: ?Function,
) {
  let root: RootType = (container._reactRootContainer: any);
  let fiberRoot;
  if (!root) {
    // 初次调用, root还未初始化, 会进入此分支
    //1. 创建ReactDOMRoot对象, 初始化react应用环境
    root = container._reactRootContainer = legacyCreateRootFromDOMContainer(
      container,
      forceHydrate,
    );
    fiberRoot = root._internalRoot;
    if (typeof callback === 'function') {
      const originalCallback = callback;
      callback = function() {
        // instance最终指向 children(入参: 如 `<App/>`)生成的dom节点
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }
    // 2. 更新容器
    unbatchedUpdates(() => {
      updateContainer(children, fiberRoot, parentComponent, callback);
    });
  } else {
    // root已经初始化, 二次调用render会进入
    // 1. 获取FiberRoot对象
    fiberRoot = root._internalRoot;
    if (typeof callback === 'function') {
      const originalCallback = callback;
      callback = function() {
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }
    // 2. 调用更新
    updateContainer(children, fiberRoot, parentComponent, callback);
  }
  return getPublicRootInstance(fiberRoot);
}
```

继续跟踪legacyCreateRootFromDOMContainer. 最后调用new ReactDOMBlockingRoot(container, LegacyRoot, options)

```javascript
function legacyCreateRootFromDOMContainer(
  container: Container,
  forceHydrate: boolean,
): RootType {
  const shouldHydrate =
    forceHydrate || shouldHydrateDueToLegacyHeuristic(container);
  return createLegacyRoot(
    container,
    shouldHydrate
      ? {
          hydrate: true,
        }
      : undefined,
  );
}

export function createLegacyRoot(
  container: Container,
  options?: RootOptions,
): RootType {
  return new ReactDOMBlockingRoot(container, LegacyRoot, options); // 注意这里的LegacyRoot是固定的, 并不是外界传入的
}
```

```javascript
function legacyCreateRootFromDOMContainer(
  container: Container,
  forceHydrate: boolean,
): RootType {
  const shouldHydrate =
    forceHydrate || shouldHydrateDueToLegacyHeuristic(container);
  return createLegacyRoot(
    container,
    shouldHydrate
      ? {
          hydrate: true,
        }
      : undefined,
  );
}

export function createLegacyRoot(
  container: Container,
  options?: RootOptions,
): RootType {
  return new ReactDOMBlockingRoot(container, LegacyRoot, options); // 注意这里的LegacyRoot是固定的, 并不是外界传入的
}

```

通过上述源码追踪可得出,legacy模式下调用ReactDOM.render有2个步骤

1. 创建ReactDOM(Blocking)实例
2. 调用updateConatiner进行更新


#### Concurrent 模式和 Blocking 模式

Concurrent模式和Blocking模式从调用方式上直接可以看出

1. 分别调用ReactDOM.createRoot和ReactDOM.createBlockingRoot创建ReactDOMRoot和ReactDOMBlockingRoot实例
2. 调用ReactDOMRoot和ReactDOMBlockingRoot实例的render方法

```js
//BlockingRoot下调用createRoot
export function createRoot(
  container: Container,
  options?: RootOptions,
): RootType {
  return new ReactDOMRoot(container, options);
}

//Concurrent下调用ReactDOMBlockingRoot
export function createBlockingRoot(
  container: Container,
  options?: RootOptions,
): RootType {
  return new ReactDOMBlockingRoot(container, BlockingRoot, options); // 注意第2个参数BlockingRoot是固定写死的
}
```

**MRoot**和**ReactDOMBlockingRoot**

继续查看**ReactDOMRoot**和**ReactDOMBlockingRoot**

```js
function ReactDOMRoot(container: Container, options: void | RootOptions) {
  // 创建一个fiberRoot对象, 并将其挂载到this._internalRoot之上
  this._internalRoot = createRootImpl(container, ConcurrentRoot, options);
}
function ReactDOMBlockingRoot(
  container: Container,
  tag: RootTag,
  options: void | RootOptions,
) {
  // 创建一个fiberRoot对象, 并将其挂载到this._internalRoot之上
  this._internalRoot = createRootImpl(container, tag, options);
}
//第一个共性：调用createRootImpl创建fiberRoot对象，并将其挂载到this._internalRoot上

ReactDOMRoot.prototype.render = ReactDOMBlockingRoot.prototype.render = function(
  children: ReactNodeList,
): void {
  const root = this._internalRoot;
  // 执行更新
  updateContainer(children, root, null, null);
};

ReactDOMRoot.prototype.unmount = ReactDOMBlockingRoot.prototype.unmount = function(): void {
  const root = this._internalRoot;
  const container = root.containerInfo;
  // 执行更新
  updateContainer(null, root, null, () => {
    unmarkContainerAsRoot(container);
  });
};
//第二个共性:原型上有个render和unmount方法,且内部都会调用updateContainer进行更新
```

**创建fiberRoot对象**

无论哪种模式下, 在ReactDOM(Blocking)Root的创建过程中, 都会调用一个相同的函数createRootImpl, 查看后续的函数调用, 最后会创建fiberRoot 对象(在这个过程中, 特别注意RootTag的传递过程):

无论哪种模式下, 在ReactDOM(Blocking)Root的创建过程中, 都会调用一个相同的函数createRootImpl, 查看后续的函数调用, 最后会创建fiberRoot 对象(在这个过程中, 特别注意RootTag的传递过程):

```js
// 注意: 3种模式下的tag是各不相同(分别是ConcurrentRoot,BlockingRoot,LegacyRoot).
this._internalRoot = createRootImpl(container, tag, options);
```

```js
function createRootImpl(
  container: Container,
  tag: RootTag,
  options: void | RootOptions,
) {
  // ... 省略部分源码(有关hydrate服务端渲染等, 暂时用不上)
  // 1. 创建fiberRoot
  const root = createContainer(container, tag, hydrate, hydrationCallbacks); // 注意RootTag的传递
  // 2. 标记dom对象, 把dom和fiber对象关联起来
  // TDOO:怎么关联起来的?
  //div#root._reactRootContainer = ReactDOM(lockingRoot)._internalRoot = FiberRoot.containerInfo->div#root
  markContainerAsRoot(root.current, container);
  // ...省略部分无关代码
  return root;
}
```

```js
export function createContainer(
  containerInfo: Container,
  tag: RootTag,
  hydrate: boolean,
  hydrationCallbacks: null | SuspenseHydrationCallbacks,
): OpaqueRoot {
  // 创建fiberRoot对象
  return createFiberRoot(containerInfo, tag, hydrate, hydrationCallbacks); // 注意RootTag的传递
}
```

```js
export function createFiberRoot(
  containerInfo: any,
  tag: RootTag,
  hydrate: boolean,
  hydrationCallbacks: null | SuspenseHydrationCallbacks,
): FiberRoot {
  // 创建fiberRoot对象, 注意RootTag的传递
  const root: FiberRoot = (new FiberRootNode(containerInfo, tag, hydrate): any);

  // 1. 这里创建了`react`应用的首个`fiber`对象, 称为`HostRootFiber`
  const uninitializedFiber = createHostRootFiber(tag);
  root.current = uninitializedFiber;
  uninitializedFiber.stateNode = root;
  // 2. 初始化HostRootFiber的updateQueue
  initializeUpdateQueue(uninitializedFiber);

  return root;
}
```

```js
export function createHostRootFiber(tag: RootTag): Fiber {
  let mode;
  if (tag === ConcurrentRoot) {
    mode = ConcurrentMode | BlockingMode | StrictMode;
  } else if (tag === BlockingRoot) {
    mode = BlockingMode | StrictMode;
  } else {
    mode = NoMode;
  }
  return createFiber(HostRoot, null, null, mode); // 注意这里设置的mode属性是由RootTag决定的
}
```

注意:fiber树中所有节点的mode都会和HostRootFiber.mode一致(新建的 fiber 节点, 其 mode 来源于父节点),所以HostRootFiber.mode非常重要, 它决定了以后整个 fiber 树构建过程

#### 进入更新入口

以上是对**legacyRenderSubtreeIntoContainer**中创建的追踪,结束后会回到**legacyRenderSubtreeIntoContainer**中进入更新容器的函数unbatchedUpdate(...)

1. legacy 回到legacyRenderSubtreeIntoContainer函数中有:

```js
// 2. 更新容器
unbatchedUpdates(() => {
  updateContainer(children, fiberRoot, parentComponent, callback);
});
```

2.concurrent 和 blocking 在ReactDOM(Blocking)Root原型上有render方法

```js
ReactDOMRoot.prototype.render = ReactDOMBlockingRoot.prototype.render = function(
  children: ReactNodeList,
): void {
  const root = this._internalRoot;
  // 执行更新
  updateContainer(children, root, null, null);
};
```

相同点:3 种模式在调用更新时都会执行updateContainer. updateContainer函数串联了react-dom与react-reconciler, 之后的逻辑进入了react-reconciler包.

不同点:

1. legacy下的更新会先调用unbatchedUpdates, 更改执行上下文为LegacyUnbatchedContext, 之后调用updateContainer进行更新.
2. concurrent和blocking不会更改执行上下文, 直接调用updateContainer进行更新.
3. fiber循环构造的时候会根据执行上下文去对应不同的处理


**继续追踪updateContanier**

```js
export function updateContainer(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  callback: ?Function,
): Lane {
  const current = container.current;
  // 1. 获取当前时间戳, 计算本次更新的优先级
  const eventTime = requestEventTime();
  const lane = requestUpdateLane(current);

  // 2. 设置fiber.updateQueue
  const update = createUpdate(eventTime, lane);
  update.payload = { element };//对应的dom节点
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

updateContainer函数位于react-reconciler包中, 它串联了react-dom与react-reconciler. 需要关注其最后调用了scheduleUpdateOnFiber.(循环构造的唯一入口)

关于优先级，和updateQueue在此处做简要的理解，React会根据当前时间戳去计算当前任务的优先级，并创建update对象挂载到updateQueue上(环形链表).之后react运行的过程中会按任务的优先级先后执行.之后会在优先级和react相关调度详细分析


---

ReferenceList:

1. https://github.com/7kms/react-illustration-series
2. https://react.iamkasong.com/preparation/idea.html#%E6%80%BB%E7%BB%9
3. https://juejin.cn/post/7085145274200358949
