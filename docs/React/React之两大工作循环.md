# React源码解析之两大工作循环


说到React就难免会提到React的fiber架构，本系列行文思路如下,本篇属于`React中的两大工作循环`

* [X] React启动过程
* [X] React的两大工作循环
* [ ] React中的对象
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

## 两大工作循环

![](https://raw.githubusercontent.com/captain1023/picGo/master/img/React%E4%B8%A4%E5%A4%A7%E5%B7%A5%E4%BD%9C%E5%BE%AA%E7%8E%AF.png)
**任务调度循环**

- 源码位于 `scheduler.js`,它是react得意良好运行的保证,控制所有任务(react的更新,创建)的调度
  **fiber构造循环**
- 源码位于 `ReactFiberWorkLoop.js`,控制fiber树的构造,`ReactFiberWorkLoop()`函数中循环调用了beginWork和completeWork两个函数,整个过程是个深度优先遍历的过程.详细过程见React fiber的初次创建与更新

## 区别和联系

### 区别

1. **任务调度循环**是以**最小****顶堆**为数据结构,堆顶是优先级最高的任务,循环执行堆的顶点, 直到堆被清空.
2. 任务调度循环的逻辑偏向宏观, 它调度的是每一个任务(task), 而不关心这个任务具体是干什么的, 具体任务其实就是执行回调函数 `performSyncWorkOnRoot()`或 `performConcurrentWorkOnRoot`. 这两个函数根据启动模式来决定,这两个函数做的任务就是去执行**fiber的构造循环**,**消费任务调度循环的任务** (注意这两个是不同的循环)

   ![](https://raw.githubusercontent.com/captain1023/picGo/master/img/taskqueue.png)

### 联系

fiber构造循环是任务调度循环中的任务的一部分.它们是从属关系，每个任务都会重新构造一个fiber树.更具体一点,fiber构造循环(`ReactFiberWorkLoop()`)被封装到了一个task里,给到任务调度循环,然后由任务调度循环决定什么时候执行.

## 主干逻辑

- 通过上文的描述, 两大循环的分工可以总结为: 任务调度循环负责调度task, fiber构造循环负责fiber的循环构造
- 结合上文的宏观概览图(展示核心包之间的调用关系), 可以将 react 运行的主干逻辑进行概括

1. 输入:将每一次更新视为一次**更新需求**(目的是要更新DOM节点)
2. 注册调度任务: react-reconciler收到**更新需求**之后,并不会立即构造fiber树，而是去调度中心scheduler注册一个新任务task,即把**更新需求**转换成一个task
3. 执行调度任务(输出):调度中心scheduler通过任务调度循环来执行task(task的执行过程又回到了react-reconciler包中)


---

ReferenceList:

1. https://github.com/7kms/react-illustration-series
2. https://react.iamkasong.com/preparation/idea.html#%E6%80%BB%E7%BB%9
3. https://juejin.cn/post/7085145274200358949
