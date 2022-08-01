# monoRepo
阅读:
[monoRepo的坑](https://juejin.cn/post/6950082433647640612)
### 为什么要使用Monorepo?
当你需要维护多个项目，多个项目之间有依赖，这些项目共用相同的基础设施（构建工具、lint）的时候，使用monorepo 会带来很多好处
1. 常用的包管理工具像 yarn、npm 都会做依赖提升，使用 monorepo 能减少依赖安装时间，同时也减少空间占用
2. 有依赖的项目之间调试非常方便，上层应用能够感知其依赖的变化，可以很方便的对依赖项进行修改和调试
3. 几个项目共用基础设施，不用重复配置。

### workspaces(package.json加上worksapce字段表明多包目录)
workspace是设置包架构的一种新方式.他的目的是更方便地使用monorepo,具体就是能让你的多个项目集中在同一个仓库,并能够相互引用 --被依赖的项目代码修改会实时反馈到依赖项目中.Monorepo中的子项目成为一个workspace,多个worksapce构成worksapces
**使用workspace的好处,其实monorepo和pnpm更搭**
1. 依赖包可以被link到一起,这意味着工作区可以相互依赖,代码是事实更新的.这是比yarn link更好的方式因为这只会影响工作去部分,不会影响整个文件系统
2. 所有项目的依赖会被遗弃安装,这让yarn更方便的优化安装依赖
3. yarn只有一个lock文件,而不是每个子项目就有一个,这意味着更少冲突

### hoist(提升) in workspace(yarn在安装时候自动link)
在monorepo项目中引进了一种新的层级结构,这种结构不再强依赖于`node_modules`来简历模块间的依赖
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/20220731200324.png)

通过将子项目的依赖提升到父项目的`node_modules`中(monorepo/node_modules),我们减少了B v1.0的重复安装

同时package1 和package2也被提升到了根node_modules中,这就是monorepo的workspace能让你像正常npm包一样使用其他子项目的秘密.同时`node_modules`中的子项目软链到packages/目录,所以被依赖子项目的代码变化也会实时更新到依赖子项目中.比如想在package-1中调试package-2,只需要在`package-1`的`package.json`文件中加上`package-2`依赖就ok了,完全不用npm link等工具

## rush

## learn

## turboRepo

## pnpm[原文](https://juejin.cn/post/6932046455733485575)

1. pnpm 本质上就是一个包管理器，这一点跟 npm/yarn 没有区别，但它作为杀手锏的两个优势在于:包安装速度极快,磁盘空间利用非常高效

### 高效利用磁盘空间
pnpm 内部使用`基于内容寻址的文件`系统来存储磁盘上所有的文件，这个文件系统出色的地方在于:重复包只安装一次,不同版本增量写入
   1. 重复的包只会安装一次,磁盘中只有一个地方写入,后面再次使用都会直接使用hardlink(硬连接). 
   2. 一个包的不同版本,pnpm也会极大程度上复用之前的版本.E.g lodash有100个文件,更新版本之后多了1个文件,那么磁盘当中并不会重新写入文件,而是保留原来的100个文件的hardlink,仅仅写入新增文件

### 支持monorepo

### 安全性极高
使用npm/yarn,由于node_module的扁平结构,如果A依赖B,B依赖C,那么A当中是可以直接使用C的,但问题是A当中并没有声明C这个依赖.pnpm通过将包本身和依赖放在同一个node_module下面,与原生Node兼容,来解决这个问题

### 依赖管理
#### npm/yarn install 原理
**执行npm/yarn install之后,包如何到达项目node_modules当中**
执行命令后,首先会构建依赖树,然后针对每个节点下面的包,会经历下面4个步骤
1. 将依赖包的版本区间解析为某个具体的版本号
2. 下载对应版本依赖的tar包到本地离线镜像
3. 将依赖从离线镜像解压到本地缓存
4. 将依赖从缓存拷贝到当前目录的node_modules目录

**包到达项目node_modules当后,如何管理这些包,换句话说是什么目录结构**
在`npm1,npm2`中呈现出的是嵌套结构
```js
node_modules
└─ foo
   ├─ index.js
   ├─ package.json
   └─ node_modules
      └─ bar
         ├─ index.js
         └─ package.json
```
如果bar当中又有依赖 ,那么又会继续嵌套下去,这样会导致如下问题
1. 依赖层级太深，会导致文件路径过长的问题，尤其在 window 系统下
2. 大量重复的包被安装，文件体积超级大。比如跟 foo 同级目录下有一个baz，两者都依赖于同一个版本的lodash，那么 lodash 会分别在两者的 node_modules 中被安装，也就是重复安装
3. 模块实例不能共享。比如 React 有一些内部变量，在两个不同包引入的 React 不是同一个模块实例，因此无法共享内部变量，导致一些不可预知的 bug

接着从`npm3`,包括yarn,都通过`扁平化依赖`的方式来解决这个问题.
相比之前的`嵌套结构`现在的目录结构类似下面这样
```js
node_modules
├─ foo
|  ├─ index.js
|  └─ package.json
└─ bar
   ├─ index.js
   └─ package.json
```
所有的依赖都拍平到node_modules目录下,不再有很深层次的嵌套关系.这样在安装新的包时,根据node require机制,会不停往上级的`node_modules`当中去找,如果找到相同版本的包就不会重新安装,解决了大量包重复安装的问题,而且依赖层级也不会太深

**问题**
1. 依赖结构的**不确定性**,npm5.x 推出package-lock.json和yarn.lock来解决
2. 扁平化算法本身的复杂性很高,耗时很长
3. 项目中仍然可以`非法访问`没有声明过的依赖包

### pnpm
1. 目录下虽然呈现的是扁平目录结构,顺着`软连接`慢慢展开,其实是嵌套结构
2. 将`包本身`和`依赖`放在同一个`node_module`下面,与原生Node完全兼容,又能降将package与相关的依赖很好组织到一起

### pnpm的安全性

pnpm这种依赖管理的方式也很巧妙的规避了**非法访问依赖**的问题,就是只要一个包未在`package.json`中声明依赖,那么在项目中是无法访问的(为啥?)


