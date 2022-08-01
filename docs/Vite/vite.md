# Vite学习记录,(还需要整理文章排版)
### 文件划分
1. 通过两个script 标签分别引入到HTML中
**问题**
1. 模块变量相当于在全局声明和定义，会有变量名冲突的问题
2. 由于变量都在全局定义，我们很难知道某个变量到底属于哪些模块，因此也给调试带来了困难。
3. 无法清晰地管理模块之间的依赖关系和加载顺序
### 命名空间
1. 在window上绑定window.a,window.b,window.c,可以解决文件划分方式上中全局变量定义所带来的一系列问题.
### IIFE私有作用域
1. webpack就是这么做的
2. 对于模块作用域的区分更加彻底
3. 每个IIFE 即立即执行函数都会创建一个私有的作用域，在私有作用域中的变量外界是无法访问的，只有模块内部的方法才能访问。

### 以上三种方式解决不了的问题
1. 并没有真正解决另外一个问题——模块加载.
2. 如果模块间存在依赖关系，那么 script 标签的加载顺序就需要受到严格的控制，一旦顺序不对，则很有可能产生运行时 Bug。


## 三大模块规范
### CommonJS、AMD和ES Module
**对于模块规范而言,一般会包含2方面内容**
1. 统一的模块化代码规范
2. 实现自动加载模块的加载器(也称loader)
#### CommonJS
1. 代码中使用**require**来导入一个模块,用**module.exports**来导出模块
```js
// module-a.js
var data = "hello world";
function getData() {
  return data;
}
module.exports = {
  getData,
};

// index.js
const { getData } = require("./module-a.js");
console.log(getData());

//实际上Node.js内部会有相应的loader转译模块代码,最后模块代码会被处理成下面这样
(function (exports, require, module, __filename, __dirname) {
  // 执行模块代码
  // 返回 exports 对象
});

```
#### CommonJS的模块加载机制
##### require
1. require第一件事就是解析模块
    - 如果是核心模块,比如fs,在node启动的时候会直接加载到内存,直接返回即可
    - 如果是带有路径的入`.`,`./`等等,则拼出一个绝对路径,然后先读取缓存require.cache再读取文件.如果没有家后缀,则自动加后缀一一识别
      - .js解析为JavaScript文本文件
      - .json解析为JSON对象
      - .node解析为二进制插件模块
    - 首次加载后的模块会缓存在**require.cache**之中，所以多次加载require，得到的对象是同一个。
    - 在执行模块代码的时候，会将模块包装成如下模式，以便于作用域在模块范围之内。
```js
(function(exports, require, module, __filename, __dirname) {
// 模块的代码实际上在这里
});
//这也是为什么我们在代码中可以使用exports,require,__dirname,等
```
##### moudle
说完了require做了些什么事，那么require触发的module做了些什么呢？我们看看用法，先写一个简单的导出模块，写好了模块之后，只需要把需要导出的参数，加入module.exports就可以了。
```js
let count=0
function addCount(){
    count++
}
module.exports={count,addCount}

//然后根据require执行代码时需要加上的，那么实际上我们的代码长成这样：
(function(exports, require, module, __filename, __dirname) {
    let count=0
    function addCount(){
        count++
    }
    module.exports={count,addCount}
});
```
**require的时候究竟module发生了什么?(可以在vscode断点调试)**
1. require调用我们的方法
2. Module的 工作内容如下
```js
startup
Module.runMain
Module._load
tryMouduleLoad
Module.load
Module.extesions..js
Module._compile
require调用我们方法
```
1. startup也就是我nodejs干的事,也就是NativeMoudule,用于执行module对象的
```js
//用于封装模块的
NativeModule.wrap = function(script) {
    return NativeModule.wrapper[0] + script + NativeModule.wrapper[1];
};

NativeModule.wrapper = [
    '(function (exports, require, module, __filename, __dirname) { ',
    '\n});'
];
```
2. nativeModule触发Module.runMain之后,我们的模块加载开始了
    * **Module._load**,就是新建一个module对象,然后将这个新对象放入Module缓存之中
    ```js
        var module = new Module(filename, parent);
        Module._cache[filename] = module;
    ``` 
    * **tryMouduleLoad**,然后就是新建的module对象开始解析导入的模块内容
    ```js
        module.load(filename);
    ```
    * 新建的module对象继承了**Module.load**,这个方法就是解析文件的类型，然后分门别类地执行
    * **Module.extesions..js** 读取文件,准备编译
    * **Module._compile** 编译环节,那么JS怎么运行文本？将文本变成可执行对象,js有3种方法：
      * eval方法`eval("console.log('aaa')")`
      * new Function() 模板引擎
        ```js
        let str="console.log(a)"
        new Function("aaa",str)
        ```
      * node执行字符串，我们用高级的vm
        ```js
            let vm=require("vm")
            let a='console.log("a")'
            vm.runInThisContext(a)
        ```
    * 这里Module用vm的方式编译，首先是封装一下，然后再执行，最后返回给require，我们就可以获得执行的结果了。
    ```js
    var wrapper = Module.wrap(content);
    var compiledWrapper = vm.runInThisContext(wrapper, {
        filename: filename,
        lineOffset: 0,
        displayErrors: true
    });
    ```
    * 因为所有的模块都是封装之后再执行的，也就说导入的这个模块，我们只能根据module.exports这一个对外接口来访问内容。

#### 总结
主要流程就是require之后解析路径，然后触发Module这一个类，然后Module的_load的方法就是在当前模块中创建一个新module的缓存，以保证下一次再require的时候可以直接返回而不用再次执行。然后就是这个新module的load方法载入并通过VM执行代码返回对象给require。

正因为是这样编译运行之后赋值给的缓存，所以如果export的值是一个参数，而不是函数，那么如果当前参数的数值改变并不会引起export的改变，因为这个赋予export的参数是静态的，并不会引起二次运行。

#### CommonJS如何解决循环依赖的?
模块require后会被缓存,并有个字段表示是否加载完成,当遇到require,会先使用缓存中未加载完成的模块,当这个模块执行完毕后再执行未加载完的模块.官网文档说模块被循环依赖时,只会输出当前执行完成的导出值.**值导出**

#### CommonJS的问题
1. 模块加载器由 Node.js 提供，依赖了 Node.js 本身的功能实现，比如文件系统，如果 CommonJS 模块直接放到浏览器中是无法执行的。当然, 业界也产生了 **browserify** 这种打包工具来支持打包 CommonJS 模块，从而顺利在浏览器中执行，相当于社区实现了一个第三方的 loader。
2. CommonJS 本身约定以同步的方式进行模块加载，这种加载机制放在服务端是没问题的，一来模块都在本地，不需要进行网络 IO，二来只有服务启动时才会加载模块，而服务通常启动后会一直运行，所以对服务的性能并没有太大的影响。但如果这种加载机制放到浏览器端，会带来明显的性能问题。它会产生大量同步的模块请求，浏览器要等待响应返回后才能继续解析模块。也就是说，模块请求会造成浏览器 JS 解析过程的阻塞，导致页面加载速度缓慢。
3. CommonJS 是一个不太适合在浏览器中运行的模块规范。因此，业界也设计出了全新的规范来作为浏览器端的模块标准，最知名的要数 AMD 了。

#### ES6 Module
ES6 Module 也被称作 ES Module(或 ESM)， 是由 ECMAScript 官方提出的模块化规范，作为一个官方提出的规范，ES Module 已经得到了现代浏览器的内置支持。在现代浏览器中，如果在 HTML 中加入含有type="module"属性的 script 标签，那么浏览器会按照 ES Module 规范来进行依赖加载和模块解析，这也是 Vite 在开发阶段实现 no-bundle 的原因，由于模块加载的任务交给了浏览器，即使不打包也可以顺利运行模块代码.

#### ES6 Module原理
1. Es moudle从加载入口模块到所有模块实例的执行主要经历了三步:**构建**,**实例化**,**运行**

**demo1**
```js
// a.mjs
console.log('a starting');
export default {
  done: true,
}
import b from './b.mjs';
console.log('in a, b.done = %j', b.done);
console.log('a done');
```
```js
// b.mjs
console.log('b starting');
export default {
  done: true,
}
import a from './a.mjs';
console.log('in b, a.done = %j', a.done);
console.log('b done');
```
执行`a.mjs`,输出如下
```js
$ node --experimental-modules a.mjs

b starting
ReferenceError: a is not defined
```
从上面例子可以看出 ES module不是动态解析,且**依赖模块优先执行**

**demo2**
```js
// a.mjs
import b from './b.mjs';
console.log('a starting');
console.log(b());
export default function () {
  return 'run func A';
}
console.log('a done');

// b.mjs
import a from './a.mjs';
console.log('b starting');
console.log(a());
export default function () {
  return 'run func B';
}
console.log('b done');
```
输出如下
```js
b starting
run func A
b done
a starting
run func B
a done
```
**构建**
从入口模块开始，根据 import 关键字遍历依赖树，每遍历一个模块则生成该模块的**模块记录（module record**，最后生成整个**模块图谱（module graph）**
**注意，这一步是 ES module 和 Commonjs 的本质区别：**
因为 ES module 需要支持浏览器端，而构建过程要获取所有的模块文件来绘制模块依赖图谱，如果参考 Commonjs 的做法把模块解析和运行放在一起，那么冗长的下载过程将会严重阻塞主线程导致应用长时间不可用，所以 ES module 在构建过程不会实例化和执行任何的js代码，也就是所谓的**静态解析**过程。这也解释了为什么ES moudle不支持使用表达式/变量的import语句

所有的模块记录都会被缓存在 **模块映射（module map**中，被依赖多次的模块也只会存在唯一一条映射记录，从而避免模块的重复下载和实例化。

**实例化**
根据模块记录的关系,在内存中把模块的导入import和export连接到一起,也成为活绑定(living bindings)
来关联模块实例和模块的导入/导出值。引擎会先采用**深度优先后序遍历（depth first post-order traversal**将模块及其依赖的导出 export 连接到内存中（直到依赖树末端），然后逐层返回再把模块相对应的导入 import 连接到内存的同一位置。这也解释了为什么导出模块的值变更时，导入模块也能捕捉到该值的变更。

需要注意的是，实例化只是JS引擎在内存中绑定模块间关系，并没有执行任何代码，也就是说这些连接好的内存空间中并没有存储变量值，然而，在此过程中导出函数将会被初始化，即所谓的 函数具有提升作用。
这使循环依赖的问题自然而然地被解决：`JS引擎不需要关心是否存在循环依赖，只需要在代码运行的时候，从内存空间中读取该导出值。`

**运行**:也就是往内存空间中填充真实值


## RollUp
### TODO:补充RollUp相关知识
两个JS的API,rollup.rollup,rollup.watch
## esBuild
### TODO:补充esBuild相关知识
两个常用API,build,transform
## Vite
### 项目入口加载
入口文件
```js
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/src/favicon.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Vite App</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```
可以看到这个 HTML 文件的内容非常简洁，在 body 标签中除了 id 为 root 的根节点之外，还包含了一个声明了type="module"的 script 标签:
```js
<script type="module" src="/src/main.tsx"></script>
```
由于现代浏览器原生支持了 ES 模块规范，因此原生的 ES 语法也可以直接放到浏览器中执行，只需要在 script 标签中声明 type="module" 即可。比如上面的 script 标签就声明了 type="module"，同时 src 指向了/src/main.tsx文件，此时相当于请求了http://localhost:3000/src/main.tsx这个资源，Vite 的 Dev Server 此时会接受到这个请求，然后读取对应的文件内容，进行一定的中间处理(编译成浏览器看的懂的语言)，最后将处理的结果返回给浏览器。
**代码中一个import语句即代表了一个HTTP请求,接收这个请求的正式Dev Server,如下面两个import语句**
```js
import "/src/index.css";
import App from "/src/App.tsx";
```
上述两个语句则分别代表了两个不同的请求，Vite Dev Server 会读取本地文件，返回浏览器可以解析的代码。当浏览器解析到新的 import 语句，又会发出新的请求，以此类推，直到所有的资源都加载完成。
**no-bundle**:利用浏览器原生 ES 模块的支持，实现开发阶段的 Dev Server，进行模块的按需加载，而不是先整体打包再进行加载。相比 Webpack 这种必须打包再加载的传统构建模式，Vite 在开发阶段省略了繁琐且耗时的打包过程，这也是它为什么快的一个重要原因。

**Tips** 虽然Vite提供了开箱即用的TypeScirpt以及JSX的编译能力,但实际上底层并没有实现TS的类型校验系统,因此需要借助tsc来完成类型校验


### Vite中接入CSS
#### CSS Modules
 - vite会对后缀带有.module的样式文件自动应用CSS Modules.
 - 可以在配置文件中的`css.modules`选项来配置CSS Modules功能
 - 
### 如何在vite中处理各种静态资源?

#### 图片加载
1. 在 HTML 或者 JSX 中，通过 img 标签来加载图片，如:
```js
<img src="../../assets/a.png"></img>
```
2. 在 CSS 中通过 background 属性加载图片，如:
```js
background: url('../../assets/b.png') norepeat;
```
3. 在 JavaScript 中，通过脚本的方式动态指定图片的src属性，如:
```js
document.getElementById('hero-img').src = '../../assets/c.png'
```
4. svg图片加载,将svg当作一个组件来引入 对应插件 pnpm i vite-plugin-svgr -D
在vite配置文件添加这个插件
```js
// vite.config.ts
import svgr from 'vite-plugin-svgr';

{
  plugins: [
    // 其它插件省略
    svgr()
  ]
}
```
5. 其它静态资源
- 媒体类文件，包括mp4、webm、ogg、mp3、wav、flac和aac。
- 字体类文件。包括woff、woff2、eot、ttf 和 otf。
- 文本类。包括webmanifest、pdf和txt。
可以通过assetsInculde配置让vite来支持加载
6. 特殊资源后缀
Vite 中引入静态资源时，也支持在路径最后加上一些特殊的 query 后缀，包括:
- **?url**: 表示获取资源的路径，这在只想获取文件路径而不是内容的场景将会很有用。
- **?raw**: 表示获取资源的字符串内容，如果你只想拿到资源的原始内容，可以使用这个后缀。
- **?inline**: 表示资源强制内联，而不是打包成单独的文件。
  
#### 生产环境处理
- 部署域名怎么配置？
- 资源打包成单文件还是作为 Base64 格式内联? 
- 图片太大了怎么压缩？
- svg 请求数量太多了怎么优化？

1. 自定义部署域名
一般在我们访问线上的站点时，站点里面一些静态资源的地址都包含了相应域名的前缀，如:
```js
<img src="https://sanyuan.cos.ap-beijing.myqcloud.com/logo.png" />

```
在 Vite 中我们可以有更加自动化的方式来实现地址的替换，只需要在配置文件中指定base参数即可
```js
// vite.config.ts
// 是否为生产环境，在生产环境一般会注入 NODE_ENV 这个环境变量，见下面的环境变量文件配置
const isProduction = process.env.NODE_ENV === 'production';
// 填入项目的 CDN 域名地址
const CDN_URL = 'xxxxxx';

// 具体配置
{
  base: isProduction ? CDN_URL: '/'
}

// .env.development
NODE_ENV=development

// .env.production
NODE_ENV=production
```
当然，有时候可能项目中的某些图片需要存放到另外的存储服务，一种直接的方案是将完整地址写死到 src 属性中，如:
```js
<img src="https://my-image-cdn.com/logo.png">
```
这样做显然是不太优雅的，我们可以通过定义环境变量的方式来解决这个问题，在项目根目录新增**env**文件:
```js
// 开发环境优先级: .env.development > .env
// 生产环境优先级: .env.production > .env
// .env 文件
VITE_IMG_BASE_URL=https://my-image-cdn.com
```
值得注意的是，如果某个环境变量要在 Vite 中通过 import.meta.env 访问，那么它必须以VITE_开头，如VITE_IMG_BASE_URL。接下来我们在组件中来使用这个环境变量:
```js
<img src={new URL('./logo.png', import.meta.env.VITE_IMG_BASE_URL).href} />
```
2. 单文件OR内联
Vite 中内置的优化方案是下面这样的:
如果静态资源体积 >= 4KB，则提取成单独的文件
如果静态资源体积 < 4KB，则作为 base64 格式的字符串内联
上述的4 KB即为提取成单文件的临界值，当然，这个临界值你可以通过build.assetsInlineLimit自行配置，如下代码所示:
svg 格式的文件不受这个临时值的影响，始终会打包成单独的文件，因为它和普通格式的图片不一样，需要动态设置一些属性

3. 图片压缩
webpack中**image-webpack-loader**,vite中**vite-plugin-imagemin**,都是基于底层imagemin压缩工具生成的

4. 雪碧图优化
HTTP2 的多路复用设计可以解决大量 HTTP 的请求导致的网络加载性能问题，因此雪碧图技术在 HTTP2 并没有明显的优化效果，这个技术更适合在传统的 HTTP 1.1 场景下使用(比如本地的 Dev Server)。
比如在 Header 中分别引入 5 个 svg 文件:
```js
import Logo1 from '@assets/icons/logo-1.svg';
import Logo2 from '@assets/icons/logo-2.svg';
import Logo3 from '@assets/icons/logo-3.svg';
import Logo4 from '@assets/icons/logo-4.svg';
import Logo5 from '@assets/icons/logo-5.svg';

//提供了glob语法糖,可以写成如下
const icons = import.meta.glob('../../assets/icons/logo-*.svg');
//对象的 value 都是动态 import，适合按需加载的场景。
//在这里我们只需要同步加载即可，可以使用 import.meta.globEager来完成:
const icons = import.meta.globEager('../../assets/icons/logo-*.svg');
```
假设页面有 100 个 svg 图标，将会多出 100 个 HTTP 请求，依此类推。我们能不能把这些 svg 合并到一起，从而大幅减少网络请求呢？
答案是可以的。这种合并图标的方案也叫雪碧图，我们可以通过**vite-plugin-svg-icons**来实现这个方案

### Vite预构建
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/vite%E9%A2%84%E6%9E%84%E5%BB%BA.drawio.png)
#### 为什么需要预构建?
1. 首先 Vite 是基于浏览器原生 ES 模块规范实现的 Dev Server，不论是应用代码，还是第三方依赖的代码，理应符合 ESM 规范才能够正常运行。但可惜，我们没有办法控制第三方的打包规范。就目前来看，还有相当多的第三方库仍然没有 ES 版本的产物，比如大名鼎鼎的 react.CommonJS 格式的代码在 Vite 当中无法直接运行，我们需要将它转换成 ESM 格式的产物。
2. 请求瀑布流问题。比如说，知名的loadsh-es库本身是有 ES 版本产物的，可以在 Vite 中直接运行。但实际上，它在加载时会发出特别多的请求，导致页面加载的前几秒几都乎处于卡顿状态.每个import都会触发一次新的文件请求，因此在这种依赖层级深、涉及模块数量多的情况下，会触发成百上千个网络请求，巨大的请求量加上 Chrome 对同一个域名下只能同时支持 6 个 HTTP 并发请求的限制，导致页面加载十分缓慢，与 Vite 主导性能优势的初衷背道而驰。不过，在进行依赖的预构建之后，lodash-es这个库的代码被打包成了一个文件，这样请求的数量会骤然减少，页面加载也快了许多

**总结**
1. 将其他格式(如 UMD 和 CommonJS)的产物转换为 ESM 格式，使其在浏览器通过 <script type="module"><script>的方式正常加载。
2. 打包第三方库的代码，将各个第三方库分散的文件合并到一起，减少 HTTP 请求数量，避免页面加载性能劣化。

#### 自定义预构建详解

1. 入口文件entryies
```js
// vite.config.ts
{
  optimizeDeps: {
    // 为一个字符串数组
    entries: ["./src/main.vue"];
    //支持glob语法
    entries: ["**/*.vue"];
    //添加依赖,强制预构建依赖项
    include: ["lodash-es", "vue"];
    //在include参数中，我们将所有不具备 ESM 格式产物包都声明一遍，这样再次启动项目就没有问题了。
  }
}
```
2. 自定义Esbuild行为
```js
// vite.config.ts
{
  optimizeDeps: {
    esbuildOptions: {
       plugins: [
        // 加入 Esbuild 插件
      ];
    }
  }
}
```
第三方库代码出错
1. 直接修改第三方库代码,并使用patch-pack这个库来解决团队协作问题
2. 加入Esbuild插件
```js
// vite.config.ts
const esbuildPatchPlugin = {
  name: "react-virtualized-patch",
  setup(build) {
    build.onLoad(
      {
        filter:
          /react-virtualized\/dist\/es\/WindowScroller\/utils\/onScroll.js$/,
      },
      async (args) => {
        const text = await fs.promises.readFile(args.path, "utf8");

        return {
          contents: text.replace(
            'import { bpfrpt_proptype_WindowScroller } from "../WindowScroller.js";',
            ""
          ),
        };
      }
    );
  },
};

// 插件加入 Vite 预构建配置
{
  optimizeDeps: {
    esbuildOptions: {
      plugins: [esbuildPatchPlugin];
    }
  }
}
```

### 双引擎架构

#### 依赖预构建-bundle工具
预构建解决了两个问题
1. 启动前进行打包并且转换为ESM格式
2. 解决多层依赖网络请求的问题
缺点:
1. 不支持降级到ES5的代码
2. 不支持const enum等语法
3. 不提供操作打包产物的接口
4. 不支持自定义Code Splitting策略

#### 单文件编译----作为TS和JSX编译工具
1. Vite 也使用 Esbuild 进行语法转译，也就是将 Esbuild 作为 Transformer 来用。
2. Vite 已经将 Esbuild 的 Transformer 能力用到了生产环境。尽管如此，对于低端浏览器场景，Vite 仍然可以做到语法和 Polyfill 安全，
3. 可以看到，虽然 Esbuild Transfomer 能带来巨大的性能提升，但其自身也有局限性，最大的局限性就在于 TS 中的类型检查问题。这是因为 Esbuild 并没有实现 TS 的类型系统，在编译 TS(或者 TSX) 文件时仅仅抹掉了类型相关的代码，暂时没有能力实现类型检查。

#### 代码压缩---作为压缩工具
1. 生产环境中 Esbuild 压缩器通过插件的形式融入到了 Rollup 的打包流程中
2. 那为什么 Vite 要将 Esbuild 作为生产环境下默认的压缩工具呢？因为压缩效率实在太高了！
3. 传统打包工具Terser慢的原因
   1. 压缩这项工作涉及大量 AST 操作，并且在传统的构建流程中，AST 在各个工具之间无法共享，比如 Terser 就无法与 Babel 共享同一个 AST，造成了很多重复解析的过程。
   2. JS 本身属于解释性 + JIT（即时编译） 的语言，对于压缩这种 CPU 密集型的工作，其性能远远比不上 Golang 这种原生语言。

#### 构建基石
1. 生产环境Bundle
  - 虽然 ESM 已经得到众多浏览器的原生支持，但生产环境做到完全no-bundle也不行，会有网络性能问题。为了在生产环境中也能取得优秀的产物性能，Vite 默认选择在生产环境中利用 Rollup 打包，并基于 Rollup 本身成熟的打包能力进行扩展和优化，主要包含 3 个方面:
    - CSS 代码分割。如果某个异步模块中引入了一些 CSS 代码，Vite 就会自动将这些 CSS 抽取出来生成单独的文件，提高线上产物的缓存复用率。
    - 自动预加载。Vite 会自动为入口 chunk 的依赖自动生成预加载标签<link rel="moduelpreload"> ，如:
    ```js
        <!-- 省略其它内容 -->
        <!-- 入口 chunk -->
        <script type="module" crossorigin src="/assets/index.250e0340.js"></script>
        <!--  自动预加载入口 chunk 所依赖的 chunk-->
        <link rel="modulepreload" href="/assets/vendor.293dca09.js">
      </head>
    ```
    这种适当预加载的做法会让浏览器提前下载好资源，优化页面性能。
    - 异步 Chunk 加载优化。在异步引入的 Chunk 中，通常会有一些公用的模块，如现有两个异步引入的 Chunk: A 和 B，而且两者有一个公共依赖 C
    一般情况下，Rollup 打包之后，会先请求 A，然后浏览器在加载 A 的过程中才决定请求和加载 C，但 Vite 进行优化之后，请求 A 的同时会自动预加载 C，通过优化 Rollup 产物依赖加载方式节省了不必要的网络开销
2. 插件兼容机制
无论是开发阶段还是生产环境，Vite 都根植于 Rollup 的插件机制和生态，如下面的架构图所示:
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/20220725133347.png)
在开发阶段,Vite借鉴了WMR的思路,自己实现了一个**Plugin Container**用来模拟Rollup调度各个Vite插件的执行逻辑,而Vite的插件完全兼容RollUp,因此在生产环境中将所有的Vite插件传入Rollup也没有问题

### Esbuild
使用Esbuild有两种方式,分别是**命令行调用**和**代码调用**
1. 命令行调用只能传递一些简单的参数,例如
```js
./node_modules/.bin/esbuild src/index.jsx --bundle --outfile=dist/out.js
```
2. Esbuild对外暴露了一系列API,主要包括两类:**Build API**和**Transform API**,我们可以在Nodejs代码中通过调用这些API来使用Esbuild的各种功能
#### 项目打包 -- Build API
 - Build API 主要用来进行项目打包,包括build、buildSync和serve三个方法
  ```js
    const { build, buildSync, serve } = require("esbuild");

    async function runBuild() {
      // 异步方法，返回一个 Promise
      const result = await build({
        // ----  如下是一些常见的配置  --- 
        // 当前项目根目录
        absWorkingDir: process.cwd(),
        // 入口文件列表，为一个数组
        entryPoints: ["./src/index.jsx"],
        // 打包产物目录
        outdir: "dist",
        // 是否需要打包，一般设为 true
        bundle: true,
        // 模块格式，包括`esm`、`commonjs`和`iife`
        format: "esm",
        // 需要排除打包的依赖列表
        external: [],
        // 是否开启自动拆包
        splitting: true,
        // 是否生成 SourceMap 文件
        sourcemap: true,
        // 是否生成打包的元信息文件
        metafile: true,
        // 是否进行代码压缩
        minify: false,
        // 是否开启 watch 模式，在 watch 模式下代码变动则会触发重新打包
        watch: false,
        // 是否将产物写入磁盘
        write: true,
        // Esbuild 内置了一系列的 loader，包括 base64、binary、css、dataurl、file、js(x)、ts(x)、text、json
        // 针对一些特殊的文件，调用不同的 loader 进行加载
        loader: {
          '.png': 'base64',
        }
      });
      console.log(result);
    }
    runBuild();
  ```
  ![](https://raw.githubusercontent.com/captain1023/picGo/master/img/20220725135745.png)
  以上就是Esbuild打包的元信息,这地我们编写插件扩展Esbuild能力非常有用
  - buildSync和build使用方式一样,但是同步的API,一般不使用
  - serve,这个API有三个特点
    * 开启serve模式后,将在指定的端口和目录上搭建一个**静态文件服务**,这个服务器用原生的GO语言实现,性能比Nodejs更高 
    * 类似webpack-dev-serve,所有的产物文件都默认不会写到磁盘,而是放在内存中,通过请求服务来访问
    * **每次请求**到来时,都会进行重新构建(rebuild),永远返回新的产物
    * 注意⚠️,触发rebuild的条件并不会代码改动,而是新的请求到来
#### 单文件转译 -- Transform API
  - Transform API,提供了**transform**和**transformSync**
  ```js
    // transform.js
    const { transform, transformSync } = require("esbuild");

    async function runTransform() {
      // 第一个参数是代码字符串，第二个参数为编译配置
      const content = await transform(
        "const isNull = (str: string): boolean => str.length > 0;",
        {
          sourcemap: true,
          loader: "tsx",
        }
      );
      console.log(content);
    }

    runTransform();
  ```
#### ESbuild 插件开发
1. ESbuild插件结构被设计为一个对象,里面有**name**和**setup**两个属性,**name**是插件的名称,**setup**是一个函数,其中入参是一个**build**对象,这个对象上挂载了一些钩子可供我们自定义鞋钩子函数的逻辑.
```js

let envPlugin = {
  name: 'env',
  setup(build) {
    build.onResolve({ filter: /^env$/ }, args => ({
      path: args.path,
      namespace: 'env-ns',
    }))

    build.onLoad({ filter: /.*/, namespace: 'env-ns' }, () => ({
      contents: JSON.stringify(process.env),
      loader: 'json',
    }))
  },
}

require('esbuild').build({
  entryPoints: ['src/index.jsx'],
  bundle: true,
  outfile: 'out.js',
  // 应用插件
  plugins: [envPlugin],
}).catch(() => process.exit(1))

//使用插件后效果
// 应用了 env 插件后，构建时将会被替换成 process.env 对象
import { PATH } from 'env'

console.log(`PATH is ${PATH}`)
```
2. 钩子函数的使用**onResolve**和**onLoad**
这两个函数两个参数,**Options**和**Callback**
```js
interface Potions {
  filter: RegExp; //必传参数,是一个正则表达式,决定了要滤出的特征文件
  namespace?:string;//选填参数,一般在onResolve钩子中的回调参数返回namespace属性作为标识,在onLoad钩子中通过namespace将模块过滤出来
}
```
**onResolve**钩子中函数参数和返回值梳理如下
```js
build.onResolve({ filter: /^env$/ }, (args: onResolveArgs): onResolveResult => {
  // 模块路径
  console.log(args.path)
  // 父模块路径
  console.log(args.importer)
  // namespace 标识
  console.log(args.namespace)
  // 基准路径
  console.log(args.resolveDir)
  // 导入方式，如 import、require
  console.log(args.kind)
  // 额外绑定的插件数据
  console.log(args.pluginData)
  
  return {
      // 错误信息
      errors: [],
      // 是否需要 external
      external: false;
      // namespace 标识
      namespace: 'env-ns';
      // 模块路径
      path: args.path,
      // 额外绑定的插件数据
      pluginData: null,
      // 插件名称
      pluginName: 'xxx',
      // 设置为 false，如果模块没有被用到，模块代码将会在产物中会删除。否则不会这么做
      sideEffects: false,
      // 添加一些路径后缀，如`?xxx`
      suffix: '?xxx',
      // 警告信息
      warnings: [],
      // 仅仅在 Esbuild 开启 watch 模式下生效
      // 告诉 Esbuild 需要额外监听哪些文件/目录的变化
      watchDirs: [],
      watchFiles: []
  }
}

//onLoad钩子函数参数和返回值
build.onLoad({ filter: /.*/, namespace: 'env-ns' }, (args: OnLoadArgs): OnLoadResult => {
  // 模块路径
  console.log(args.path);
  // namespace 标识
  console.log(args.namespace);
  // 后缀信息
  console.log(args.suffix);
  // 额外的插件数据
  console.log(args.pluginData);
  
  return {
      // 模块具体内容
      contents: '省略内容',
      // 错误信息
      errors: [],
      // 指定 loader，如`js`、`ts`、`jsx`、`tsx`、`json`等等
      loader: 'json',
      // 额外的插件数据
      pluginData: null,
      // 插件名称
      pluginName: 'xxx',
      // 基准路径
      resolveDir: './dir',
      // 警告信息
      warnings: [],
      // 同上
      watchDirs: [],
      watchFiles: []
  }
});
```

3. 其他钩子**onStart**和**onEnd** 两个钩子用来在构建开始和结束时执行一些自定义的逻辑
```js
let examplePlugin = {
  name: 'example',
  setup(build) {
    build.onStart(() => {
      console.log('build started')
    });
    build.onEnd((buildResult) => {
      if (buildResult.errors.length) {
        return;
      }
      // 构建元信息
      // 获取元信息后做一些自定义的事情，比如生成 HTML
      console.log(buildResult.metafile)
    })
  },
}

```
在使用这两个钩子的时候,有2点需要注意
1. onStart 的执行时机是在每次 build 的时候，包括触发 watch 或者 serve模式下的重新构建。
2. onEnd 钩子中如果要拿到 metafile，必须将 Esbuild 的构建配置中metafile属性设为 true。

### RollUp
#### RollUp常用配置解读
1. 多产物配置
在打包 JavaScript 类库的场景中，我们通常需要对外暴露出不同格式的产物供他人使用，不仅包括 ESM，也需要包括诸如CommonJS、UMD等格式，保证良好的兼容性。那么，同一份入口文件，如何让 Rollup 给我们打包出不一样格式的产物呢？我们基于上述的配置文件来进行修改:
```js
// rollup.config.js
/**
 * @type { import('rollup').RollupOptions }
 */
const buildOptions = {
  input: ["src/index.js"],
  // 将 output 改造成一个数组
  output: [
    {
      dir: "dist/es",
      format: "esm",
    },
    {
      dir: "dist/cjs",
      format: "cjs",
    },
  ],
};
//从代码中可以看到，我们将output属性配置成一个数组，数组中每个元素都是一个描述对象，决定了不同产物的输出行为。
export default buildOptions;
```
2. 多入口配置
除了多产物配置，Rollup 中也支持多入口配置，而且通常情况下两者会被结合起来使用。接下来，就让我们继续改造之前的配置文件，将 input 设置为一个数组或者一个对象，如下所示:
```js
{
  input: ["src/index.js", "src/util.js"]
}
// 或者
{
  input: {
    index: "src/index.js",
    util: "src/util.js",
  },
}
```
如果不同入口对应的打包配置不一样,我们也可以默认导出一个**配置数组**
```js
// rollup.config.js
/**
 * @type { import('rollup').RollupOptions }
 */
const buildIndexOptions = {
  input: ["src/index.js"],
  output: [
    // 省略 output 配置
  ],
};

/**
 * @type { import('rollup').RollupOptions }
 */
const buildUtilOptions = {
  input: ["src/util.js"],
  output: [
    // 省略 output 配置
  ],
};

export default [buildIndexOptions, buildUtilOptions];
```
3. 自定义**output**配置
刚才我们提到了input的使用，主要用来声明入口，可以配置成字符串、数组或者对象，使用比较简单。而output与之相对，用来配置输出的相关信息，常用的配置项如下:
```js
output: {
  // 产物输出目录
  dir: path.resolve(__dirname, 'dist'),
  // 以下三个配置项都可以使用这些占位符:
  // 1. [name]: 去除文件后缀后的文件名
  // 2. [hash]: 根据文件名和文件内容生成的 hash 值
  // 3. [format]: 产物模块格式，如 es、cjs
  // 4. [extname]: 产物后缀名(带`.`)
  // 入口模块的输出文件名
  entryFileNames: `[name].js`,
  // 非入口模块(如动态 import)的输出文件名
  chunkFileNames: 'chunk-[hash].js',
  // 静态资源文件输出文件名
  assetFileNames: 'assets/[name]-[hash][extname]',
  // 产物输出格式，包括`amd`、`cjs`、`es`、`iife`、`umd`、`system`
  format: 'cjs',
  // 是否生成 sourcemap 文件
  sourcemap: true,
  // 如果是打包出 iife/umd 格式，需要对外暴露出一个全局变量，通过 name 配置变量名
  name: 'MyBundle',
  // 全局变量声明
  globals: {
    // 项目中可以直接用`$`代替`jquery`
    jquery: '$'
  }
}
```
4. 依赖external
对于某些第三方包，有时候我们不想让 Rollup 进行打包，也可以通过 external 进行外部化:
```js
{
  external: ['react', 'react-dom']
}
```
5. 接入插件的能力
在 Rollup 的日常使用中，我们难免会遇到一些 Rollup 本身不支持的场景，比如兼容 CommonJS 打包、注入环境变量、配置路径别名、压缩产物代码 等等。这个时候就需要我们引入相应的 Rollup 插件了。
虽然 Rollup 能够打**输出** CommonJS 格式的产物，但对于**输入**给 Rollup 的代码并不支持 CommonJS，仅仅支持 ESM。你可能会说，那我们直接在项目中统一使用 ESM 规范就可以了啊，这有什么问题呢？需要注意的是，我们不光要考虑项目本身的代码，还要考虑第三方依赖。目前为止，还是有不少第三方依赖只有 CommonJS 格式产物而并未提供 ESM 产物,因此，我们需要引入额外的插件去解决这个问题。
```ts
pnpm i @rollup/plugin-node-resolve @rollup/plugin-commonjs 
```
* **@rollup/plugin-node-resolve**是为了允许我们加载第三方依赖，否则像**import React from 'react'**的依赖导入语句将不会被 Rollup 识别。
* **@rollup/plugin-commonjs** 的作用是将 CommonJS 格式的代码转换为 ESM 格式
然后让我们在配置文件中导入这些插件:
```js
// rollup.config.js
import resolve from "@rollup/plugin-node-resolve";
import commonjs from "@rollup/plugin-commonjs";

/**
 * @type { import('rollup').RollupOptions }
 */
export default {
  input: ["src/index.js"],
  output: [
    {
      dir: "dist/es",
      format: "esm",
    },
    {
      dir: "dist/cjs",
      format: "cjs",
    },
  ],
  // 通过 plugins 参数添加插件
  plugins: [resolve(), commonjs()],
};
```
#### JavaScript API方式调用
有些场景下我们需要基于RollUp定制一些打包过程,JS API来调用Rollup,主要有**rollup.rollup**和**rollup.watch**两个API

- 使用rollup来一次行的打包
```js
// build.js
const rollup = require("rollup");

// 常用 inputOptions 配置
const inputOptions = {
  input: "./src/index.js",
  external: [],
  plugins:[]
};

const outputOptionsList = [
  // 常用 outputOptions 配置
  {
    dir: 'dist/es',
    entryFileNames: `[name].[hash].js`,
    chunkFileNames: 'chunk-[hash].js',
    assetFileNames: 'assets/[name]-[hash][extname]',
    format: 'es',
    sourcemap: true,
    globals: {
      lodash: '_'
    }
  }
  // 省略其它的输出配置
];

async function build() {
  let bundle;
  let buildFailed = false;
  try {
    // 1. 调用 rollup.rollup 生成 bundle 对象
    bundle = await rollup.rollup(inputOptions);
    for (const outputOptions of outputOptionsList) {
      // 2. 拿到 bundle 对象，根据每一份输出配置，调用 generate 和 write 方法分别生成和写入产物,处理多打包情况
      const { output } = await bundle.generate(outputOptions);
      await bundle.write(outputOptions);
    }
  } catch (error) {
    buildFailed = true;
    console.error(error);
  }
  if (bundle) {
    // 最后调用 bundle.close 方法结束打包
    await bundle.close();
  }
  process.exit(buildFailed ? 1 : 0);
}

build();
```
主要的执行步骤如下:
1. 通过**rollup.rollup**方法,传入**inputOptions**,生成bundle对象
2. 调用bundle对象的generate和write方法,传入outputOptions,分别完成产物的生成和磁盘写入
3. 调用bundle对象的close方法来结束打包
  
- 通过rollup.watch来完成watch模式下的打包
```js
// watch.js
const rollup = require("rollup");

const watcher = rollup.watch({
  // 和 rollup 配置文件中的属性基本一致，只不过多了`watch`配置
  input: "./src/index.js",
  output: [
    {
      dir: "dist/es",
      format: "esm",
    },
    {
      dir: "dist/cjs",
      format: "cjs",
    },
  ],
  watch: {
    exclude: ["node_modules/**"],
    include: ["src/**"],
  },
});

// 监听 watch 各种事件
watcher.on("restart", () => {
  console.log("重新构建...");
});

watcher.on("change", (id) => {
  console.log("发生变动的模块id: ", id);
});

watcher.on("event", (e) => {
  if (e.code === "BUNDLE_END") {
    console.log("打包信息:", e);
  }
});
```
### RollUP解整体阶段构建
在执行 rollup 命令之后，在 cli 内部的主要逻辑简化如下:
```js
// Build 阶段
const bundle = await rollup.rollup(inputOptions);

// Output 阶段
await Promise.all(outputOptions.map(bundle.write));

// 构建结束
await bundle.close();
```
Rollup内部主要经历了**Build**和**Output**两大阶段
1. 首先Build阶段主要负责创建模块依赖图,初始化各个模块的AST以及模块之间的依赖关系
```js
// src/index.js
import { a } from './module-a';
console.log(a);

// src/module-a.js
export const a = 1;

//然后执行如下的构建脚本:
const rollup = require('rollup');
const util = require('util');
async function build() {
  const bundle = await rollup.rollup({
    input: ['./src/index.js'],
  });
  console.log(util.inspect(bundle));
}
build();
//可以看到这样的bundle对象信息
{
  cache: {
    modules: [
      {
        ast: 'AST 节点信息，具体内容省略',
        code: 'export const a = 1;',
        dependencies: [],
        id: '/Users/code/rollup-demo/src/data.js',
        // 其它属性省略
      },
      {
        ast: 'AST 节点信息，具体内容省略',
        code: "import { a } from './data';\n\nconsole.log(a);",
        dependencies: [
          '/Users/code/rollup-demo/src/data.js'
        ],
        id: '/Users/code/rollup-demo/src/index.js',
        // 其它属性省略
      }
    ],
    plugins: {}
  },
  closed: false,
  // 挂载后续阶段会执行的方法
  close: [AsyncFunction: close],
  generate: [AsyncFunction: generate],
  write: [AsyncFunction: write]
}
```
从上面的信息中可以看出，目前经过**Build**阶段的**bundle**对象其实并没有进行**模块的打包**，这个对象的作用在于**存储各个模块的内容及依赖关系**，同时暴露generate和write方法，以进入到后续的 Output 阶段（write和generate方法唯一的区别在于前者打包完产物会写入磁盘，而后者不会）。
真正的打包的过程在**Output**阶段进行,即在**bundle**对象的generate或者write方法中进行
```js
const rollup = require('rollup');
async function build() {
  const bundle = await rollup.rollup({
    input: ['./src/index.js'],
  });
  const result = await bundle.generate({
    format: 'es',
  });
  console.log('result:', result);
}

build();
//执行后可以得到如下的输出:
{
  output: [
    {
      exports: [],
      facadeModuleId: '/Users/code/rollup-demo/src/index.js',
      isEntry: true,
      isImplicitEntry: false,
      type: 'chunk',
      code: 'const a = 1;\n\nconsole.log(a);\n',
      dynamicImports: [],
      fileName: 'index.js',
      // 其余属性省略
    }
  ]
}
//这里可以看到所有的输出信息，生成的output数组即为打包完成的结果。当然，如果使用 bundle.write 会根据配置将最后的产物写入到指定的磁盘目录中。
```
一次完整的构建过程的流程
1. Build阶段:解析各模块的内容及依赖关系
2. Output阶段:完成打包及输出的过程

#### 拆解插件工作流
1. 插件的各个Hook可以根据这两个构建阶段分为两类:**Build Hook**和**Output Hook**
  - **Build Hook**即在Build阶段执行的钩子函数,在这个阶段主要进行模块代码的转换、AST解析以及模块依赖的解析,那么这个阶段的Hook对于代码的操作粒度一般为**模块**级别,也就是单文件级别
  - **Output Hook**,则主要进行代码的打包,对于代码而言,操作粒度一般为**chunk**级别(一个chunk通常指很多文件打包到一起的产物)
2. 根据不同的Hook执行方式也会有不同的分类,主要包括**Async**,**Sync**,**Parallel**,**Sequential**,**First**五种方式
**Async和Sync**:分别代表**同步**和**异步**
**Parallel**:这里指并行的钩子函数.如果有多插件实现了这个钩子的逻辑,一旦有钩子函数是异步逻辑,则并发执行钩子函数,不会等待当前钩子函数完成(底层使用**Promise.all**)
3. **Sequential**:串行的钩子函数.这种Hook往往适用于插件间处理结果相互依赖的情况,前一个插件Hook的返回值作为后续插件的入参,这种情况就需要等待前一个插件执行完Hook,获得其执行结果,然后才能进行下一个插件相应Hook的调用,如**transform**
4. **First**:如果有多个插件实现了这个Hook,那么Hook将依次运行,直到返回一个非null或非undefined的值为止.比较典型的Hook是**resolveId**,一旦有插件的resolveid返回了一个路径,将停止执行后续插件的resolvedId逻辑.

#### Build阶段工作流
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/20220725155843.png)
1. 首先经历**options**钩子进行配置的转换,开始正式构建流程
2. 随之Rollup会调用**buildStart**钩子,开始正式构建流程
3. rollup先进入到resolveId钩子中解析文件路径.(从**input**配置指定的入口文件开始)
4. Rollup通过调用**load**钩子加载模块内容
5. 紧接着Rollup执行所有的**transform**钩子来对模块内容进行自定义的转换,比如babel转译
6. 现在Rollup拿到最后模块的内容,进行AST分析,得到所有的import内容,调用**moduleParsed**钩子
   1. 如果是**普通的import**,则执行**resolveID钩子**,继续回到步骤3
   2. 如果是**动态import**,则执行**resolveDynamicImport钩子解析路径**,如果解析成功,则返回步骤4加载模块,否则回到**步骤3**通过**resolveId解析路径**
7. 直到所有的import都没洗完毕,Rollup执行**buildEnd**钩子,Build阶段结束
**关于 watch**:当使用rollup --watch开启观察模式,Rollup内会初始化一个**watcher**对象,当文件内容发生变化时,watcher对象会自动触发**watchChange**钩子执行并对项目进行重新构建.在当前**打包过程结束**时,Rollup会自动清除watcher对象调用**closeWatcher**钩子.
#### output阶段工作流
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/20220725161304.png)
1. 执行所有插件的**outputOptions**钩子函数,对output配置进行转换.
2. 执行**renderStart**,兵法之行renderStart钩子,正式开始打包
3. 并发执行所有插件的**banner**、**footer**、**intro**、**outro**钩子,(底层用Promise.all包裹所有的这四种钩子函数),这四个钩子函数就是产物的固定位置插入一些自定义的内容,比如协议声明内容、项目介绍等
4. 从入口模块开始扫描,针对动态import语句执行**renderDynamicImport**钩子,来自定义动态import的内容
5. 对每个即将生成的**chunk**,执行**augmentChunkHash**钩子,来决定是否更改chunk的哈希值,在**watch**模式下即可能会多次打包的场景下,这个钩子会比较实用
6. 如果没有遇到**import.meta**语句,则进入下一步,否则
   1. 对于**import.meta.url**语句调用**resolveFileUrl**来自定义url解析逻辑
   2. 对于其他**import.meta**属性,则调用**resolveImportMeta**来进行自定义的解析.
7. 接着Rollup会生成所有chunk的内容,真对每个chunk会依次调用插件的**renderChunk**方法进行自定义操作,也就是说,在这里的时候可以操作打包产物了
8. 随后会调用**generateBundle**钩子,这个钩子的入参里面会包含所有的打包产物信息,包括**chunk(打包后的代码)**,**asset(最终的静态资源文件)**.你可以在这里删除一些chunk或asset,最终这些内容将不会作为产物输出
9. 前面提到了**rollup.rollup**方法会返回一个bundle对象,这个对象是包含**generate**和**write**两个方法,两个方法唯一的区别在于后者会将代码写入到磁盘中,同时会触发**writeBundle**钩子,传入所有的打包产物的信息,包括chunk和asset,和generateBundle钩子非常相似.不过在**generateBundle**执行时,产物还没有输出,但**writeBundle**钩子产物已经输出到磁盘了
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/20220725164146.png)
10. 当上述**bundle**的**close**方法被调用,会触发**closeBundle**钩子,到这里Output阶段正式结束

### 如何开发一个完整的Vite插件
Vite 插件与 Rollup 插件结构类似，为一个name和各种插件 Hook 的对象:
```js
{
  // 插件名称
  name: 'vite-plugin-xxx',
  load(code) {
    // 钩子逻辑
  },
}
//一般情况下因为要考虑到外部传参，我们不会直接写一个对象，而是实现一个返回插件对象的工厂函数，如下代码所示:
// myPlugin.js
export function myVitePlugin(options) {
  console.log(options)
  return {
    name: 'vite-plugin-xxx',
    load(id) {
      // 在钩子逻辑中可以通过闭包访问外部的 options 传参
    }
  }
}

// 使用方式
// vite.config.ts
import { myVitePlugin } from './myVitePlugin';
export default {
  plugins: [myVitePlugin({ /* 给插件传参 */ })]
}
```
### 插件hook介绍
1. 通用Hook,Vite在**开发阶段**会模拟Rollup的行为
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/20220725170457.png)

其中 Vite 会调用一系列与 Rollup 兼容的钩子，这个钩子主要分为三个阶段:
* **服务器启动阶段:** options和buildStart钩子会在服务启动时被调用。
* **请求响应阶段:** 当浏览器发起请求时，Vite 内部依次调用resolveId、load和transform钩子。
* **服务关闭阶段:** Vite会依次执行**buildEnd**和**closeBundle**钩子
除了以上钩子,其他Rollup插件钩子(如**moduleParsed**、**renderChunk**)均不会再vite**开发阶段**调用.而生产环境下，由于 Vite 直接使用 Rollup，Vite 插件中所有 Rollup 的插件钩子都会生效.
### Vite独有Hook
接下来给大家介绍 Vite 中特有的一些 Hook，这些 Hook 只会在 Vite 内部调用，而放到 Rollup 中会被直接忽略
1. 给配置再加点料:**config**
Vite 在读取完配置文件（即vite.config.ts）之后，会拿到用户导出的配置对象，然后执行 config 钩子。在这个钩子里面，你可以对配置文件导出的对象进行自定义的操作，如下代码所示:
```js
// 返回部分配置（推荐）
const editConfigPlugin = () => ({
  name: 'vite-plugin-modify-config',
  config: () => ({
    alias: {
      react: require.resolve('react')
    }
  })
})
```
官方推荐的姿势是在 config 钩子中返回一个配置对象，这个配置对象会和 Vite 已有的配置进行深度的合并。不过你也可以通过钩子的入参拿到 config 对象进行自定义的修改，如下代码所示:
```js
const mutateConfigPlugin = () => ({
  name: 'mutate-config',
  // command 为 `serve`(开发环境) 或者 `build`(生产环境)
  config(config, { command }) {
    // 生产环境中修改 root 参数
    if (command === 'build') {
      config.root = __dirname;
    }
  }
})
```
在一些比较深层的对象配置中，这种直接修改配置的方式会显得比较麻烦，如 optimizeDeps.esbuildOptions.plugins，需要写很多的样板代码，类似下面这样:
```js
// 防止出现 undefined 的情况
config.optimizeDeps = config.optimizeDeps || {}
config.optimizeDeps.esbuildOptions = config.optimizeDeps.esbuildOptions || {}
config.optimizeDeps.esbuildOptions.plugins = config.optimizeDeps.esbuildOptions.plugins || []
//因此这种情况下，建议直接返回一个配置对象，这样会方便很多:

config() {
  return {
    optimizeDeps: {
      esbuildOptions: {
        plugins: []
      }
    }
  }
}
```
2. 记录最终配置: **configResolved**
Vite 在解析完配置之后会调用configResolved钩子，这个钩子一般用来记录最终的配置信息，而不建议再修改配置，用法如下图所示:
```js
const exmaplePlugin = () => {
  let config

  return {
    name: 'read-config',

    configResolved(resolvedConfig) {
      // 记录最终配置
      config = resolvedConfig
    },

    // 在其他钩子中可以访问到配置
    transform(code, id) {
      console.log(config)
    }
  }
}
```
3. 获取 Dev Server 实例: configureServer
这个钩子仅在开发阶段会被调用，用于扩展 Vite 的 Dev Server，一般用于增加自定义 server 中间件，如下代码所示:
```js
const myPlugin = () => ({
  name: 'configure-server',
  configureServer(server) {
    // 姿势 1: 在 Vite 内置中间件之前执行
    server.middlewares.use((req, res, next) => {
      // 自定义请求处理逻辑
    })
    // 姿势 2: 在 Vite 内置中间件之后执行 
    return () => {
      server.middlewares.use((req, res, next) => {
        // 自定义请求处理逻辑
      })
    }
  }
})
```
4. 转换 HTML 内容: transformIndexHtml
这个钩子用来灵活控制 HTML 的内容，你可以拿到原始的 html 内容后进行任意的转换:
```js
const htmlPlugin = () => {
  return {
    name: 'html-transform',
    transformIndexHtml(html) {
      return html.replace(
        /<title>(.*?)</title>/,
        `<title>换了个标题</title>`
      )
    }
  }
}
// 也可以返回如下的对象结构，一般用于添加某些标签
const htmlPlugin = () => {
  return {
    name: 'html-transform',
    transformIndexHtml(html) {
      return {
        html,
        // 注入标签
        tags: [
          {
            // 放到 body 末尾，可取值还有`head`|`head-prepend`|`body-prepend`，顾名思义
            injectTo: 'body',
            // 标签属性定义
            attrs: { type: 'module', src: './index.ts' },
            // 标签名
            tag: 'script',
          },
        ],
      }
    }
  }
}
```
5. 热更新处理: handleHotUpdate
这个钩子会在 Vite 服务端处理热更新时被调用，你可以在这个钩子中拿到热更新相关的上下文信息，进行热更模块的过滤，或者进行自定义的热更处理。下面是一个简单的例子:
```js
const handleHmrPlugin = () => {
  return {
    async handleHotUpdate(ctx) {
      // 需要热更的文件
      console.log(ctx.file)
      // 需要热更的模块，如一个 Vue 单文件会涉及多个模块
      console.log(ctx.modules)
      // 时间戳
      console.log(ctx.timestamp)
      // Vite Dev Server 实例
      console.log(ctx.server)
      // 读取最新的文件内容
      console.log(await read())
      // 自行处理 HMR 事件
      ctx.server.ws.send({
        type: 'custom',
        event: 'special-update',
        data: { a: 1 }
      })
      return []
    }
  }
}

// 前端代码中加入
if (import.meta.hot) {
  import.meta.hot.on('special-update', (data) => {
    // 执行自定义更新
    // { a: 1 }
    console.log(data)
    window.location.reload();
  })
}
```
**总结**
**config:** 用来进一步修改配置。
**configResolved:** 用来记录最终的配置信息。
**configureServer:** 用来获取 Vite Dev Server 实例，添加中间件。
**transformIndexHtml:** 用来转换 HTML 的内容。
**handleHotUpdate:** 用来进行热更新模块的过滤，或者进行自定义的热更新处理。

3. 插件Hook执行顺序
```js
// test-hooks-plugin.ts
// 注: 请求响应阶段的钩子
// 如 resolveId, load, transform, transformIndexHtml在下文介绍
// 以下为服务启动和关闭的钩子
export default function testHookPlugin () {
  return {
    name: 'test-hooks-plugin', 
    // Vite 独有钩子
    config(config) {
      console.log('config');
    },
    // Vite 独有钩子
    configResolved(resolvedCofnig) {
      console.log('configResolved');
    },
    // 通用钩子
    options(opts) {
      console.log('options');
      return opts;
    },
    // Vite 独有钩子
    configureServer(server) {
      console.log('configureServer');
      setTimeout(() => {
        // 手动退出进程
        process.kill(process.pid, 'SIGTERM');
      }, 3000)
    },
    // 通用钩子
    buildStart() {
      console.log('buildStart');
    },
    // 通用钩子
    buildEnd() {
      console.log('buildEnd');
    },
    // 通用钩子
    closeBundle() {
      console.log('closeBundle');
    }
}
```
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/20220725172333.png)
**服务启动阶段:** config、configResolved、options、configureServer、buildStart
**请求响应阶段:** 如果是 html 文件，仅执行transformIndexHtml钩子；对于非 HTML 文件，则依次执行resolveId、load和transform钩子。
**热更新阶段:** 执行handleHotUpdate钩子。
**服务关闭阶段:** 依次执行buildEnd和closeBundle钩子。

### 插件应用位置
默认情况下 Vite 插件同时被用于**开发环境**和**生产环境**，可以通过apply属性来决定应用场景:
```js
{
  // 'serve' 表示仅用于开发环境，'build'表示仅用于生产环境
  apply: 'serve'
}
//aplly参数还可以配置成一个函数,进行更灵活的控制
apply(config, { command }) {
  // 只用于非 SSR 情况下的生产环境构建
  return command === 'build' && !config.build.ssr
}
```
同时，你也可以通过enforce属性来指定插件的执行顺序:
```js
{
  // 默认为`normal`，可取值还有`pre`和`post`
  enforce: 'pre'
}
```
Vite 中插件的执行顺序如下图所示:
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/20220725180055.png)
Vite会依次执行如下的插件
* Alias (路径别名)相关的插件。
* ⭐️ 带有 enforce: 'pre' 的用户插件。
* Vite 核心插件。
* ⭐️ 没有 enforce 值的用户插件，也叫普通插件。
* Vite 生产环境构建用的插件。
* ⭐️ 带有 enforce: 'post' 的用户插件。
* Vite 后置构建插件(如压缩插件)。
### 插件开发实战
#### Case1:虚拟模块加载
作为构建工具，一般需要处理两种形式的模块，一种存在于真实的磁盘文件系统中，另一种并不在磁盘而在内存当中，也就是虚拟模块。通过虚拟模块，我们既可以把**自己手写的一些代码字符串作为单独的模块内容**，又可以**将内存中某些经过计算得出的变量**作为模块内容进行加载，非常灵活和方便。
```js
// plugins/virtual-module.ts
import { Plugin } from 'vite';

// 虚拟模块名称
const virtualFibModuleId = 'virtual:fib';
// Vite 中约定对于虚拟模块，解析后的路径需要加上`\0`前缀
const resolvedFibVirtualModuleId = '\0' + virtualFibModuleId;

export default function virtualFibModulePlugin(): Plugin {
  let config: ResolvedConfig | null = null;
  return {
    name: 'vite-plugin-virtual-module',
    resolveId(id) {
      if (id === virtualFibModuleId) { 
        return resolvedFibVirtualModuleId;
      }
    },
    load(id) {
      // 加载虚拟模块
      if (id === resolvedFibVirtualModuleId) {
        return 'export default function fib(n) { return n <= 1 ? n : fib(n - 1) + fib(n - 2); }';
      }
    }
  }
}
```

**总结图**
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/Vite%E6%8F%92%E4%BB%B6%E6%B5%81%E6%B0%B4%E7%BA%BF.drawio.png)
### HRM API及原理
1. 通过HRM的技术我们就可以实现**局部刷新**和**状态保存**,从而解决之前提到的种种问题

#### 深入HMR API
```js
interface ImportMeta {
  readonly hot?: {
    readonly data: any
    accept(): void
    accept(cb: (mod: any) => void): void
    accept(dep: string, cb: (mod: any) => void): void
    accept(deps: string[], cb: (mods: any[]) => void): void
    prune(cb: () => void): void
    dispose(cb: (data: any) => void): void
    decline(): void
    invalidate(): void
    on(event: string, cb: (...args: any[]) => void): void
  }
}
```
**import.meta**对象为现代浏览器原生的一个内置对象，Vite 所做的事情就是在这个对象上的 hot 属性中定义了一套完整的属性和方法。因此，在 Vite 当中，你就可以通过**import.meta.hot**来访问关于 HMR 的这些属性和方法，比如**import.meta.hot.accept()**

#### 模块更新时逻辑: hot.accept
1. 用来接受模块更新的。 一旦 Vite 接受了这个更新，当前模块就会被认为是 HMR 的边界
* 接受**自身模块**的更新:当模块接受自身的更新时，则当前模块会被认为 HMR 的边界。也就是说，除了当前模块，其他的模块均未受到任何影响。
* 接受**某个子模块**更新:accpet("./render.ts"),参数为模块路径,相当于告诉Vite:我监听了**render**模块的更新,当它的内容更新的时候,请把最新的内容传给我.同样的第二个参数中定义了模块变化后的回调函数,这里拿到了**render**的最新内容,然后执行其中的渲染逻辑.
* 接受**多个子模块的**更新:接受多个子模块的更新,当其中任何一个子模块更新之后,父模块会成为HMR边界.

#### 销毁时逻辑:hot.dispose
#### 共享数据:hot.data属性
#### import.meta.hot.declinde(),表示此模块不可热更新,当前模块更新时会强制进行页面刷新.
#### import.met.hot.invalidate(),用来强制刷新页面
#### 自定义事件 import.meta.hot.on来监听HRM的自定义事件
内部会有以下事件自动触发
1. vite:beforeUpdate 当模块更新时触发
2. vite:beforeFullReload 当即将重新刷新页面时出发
3. vite:beforePrune: 当不再需要的模块即将被剔除时触发
4. vite:error 当发送错误时触发
可以通过**handleHotUpdate**这个插件Hook来进行触发:
```js
// 插件 Hook
handleHotUpdate({ server }) {
  server.ws.send({
    type: 'custom',
    event: 'custom-update',
    data: {}
  })
  return []
}
// 前端代码
import.meta.hot.on('custom-update', (data) => {
  // 自定义更新逻辑
})
```

### 代码分割
* bundle 指的是整体的打包产物，包含 JS 和各种静态资源。
* chunk指的是打包后的 JS 文件，是 bundle 的子集。
* vendor是指第三方包的打包产物，是一种特殊的 chunk。

#### Code Spliting解决的问题
1. 按需加载 
2. 缓存复用率

#### Vite默认拆包策略
1. 自动CSS代码分割能力,即实现一个chunk对应一个css文件
2. vite基于Rollup的**manualChunks**API实现了**应用拆包**策略
   1. 对于Initial Chunk而言,业务代码和第三方包代码分别打包为单独的 chunk,这是 Vite 2.9 版本之前的做法，而在 Vite 2.9 及以后的版本，默认打包策略更加简单粗暴，将所有的 js 代码全部打包到 index.js 中。
   2. 对于 Async Chunk 而言 ，动态 import 的代码会被拆分成单独的 chunk.
3. 优点:实现了CSS代码与业务代码、第三方库代码、动态import模块代码分离,但第三方代码包容易臃肿

#### 自定义拆包策略
针对更细粒度的拆包，Vite 的底层打包引擎 Rollup 提供了manualChunks，让我们能自定义拆包策略，它属于 Vite 配置的一部分，示例如下:
```js
// vite.config.ts
export default {
  build: {
    rollupOptions: {
      output: {
        // manualChunks 配置
        manualChunks: {},
      },
    }
  },
}
```
manualChunks 主要有两种配置的形式，可以配置为一个对象或者一个函数。我们先来看看对象的配置，也是最简单的配置方式，你可以在上述的示例项目中添加如下的manualChunks配置代码:
```js
// vite.config.ts
{
  build: {
    rollupOptions: {
      output: {
        // manualChunks 配置
        manualChunks: {
          // 将 React 相关库打包成单独的 chunk 中
          'react-vendor': ['react', 'react-dom'],
          // 将 Lodash 库的代码单独打包
          'lodash': ['lodash-es'],
          // 将组件库的代码打包
          'library': ['antd', '@arco-design/web-react'],
        },
      },
    }
  },
}
//除了对象的配置方式之外，我们还可以通过函数进行更加灵活的配置
manualChunks(id) {
  if (id.includes('antd') || id.includes('@arco-design/web-react')) {
    return 'library';
  }
  if (id.includes('lodash')) {
    return 'lodash';
  }
  if (id.includes('react')) {
    return 'react';
  }
}

//上述函数有循环依赖,解决如下
// 确定 react 相关包的入口路径
const chunkGroups = {
  'react-vendor': [
    require.resolve('react'),
    require.resolve('react-dom')
  ],
}

// Vite 中的 manualChunks 配置
function manualChunks(id, { getModuleInfo }) { 
  for (const group of Object.keys(chunkGroups)) {
    const deps = chunkGroups[group];
    if (
      id.includes('node_modules') && 
      // 递归向上查找引用者，检查是否命中 chunkGroups 声明的包 
      isDepInclude(id, deps, [], getModuleInfo)
     ) { 
      return group;
    }
  }
}

function isDepInclude (id: string, depPaths: string[], importChain: string[], getModuleInfo): boolean | undefined  {
  const key = `${id}-${depPaths.join('|')}`;
  // 出现循环依赖，不考虑
  if (importChain.includes(id)) {
    cache.set(key, false);
    return false;
  }
  // 验证缓存
  if (cache.has(key)) {
    return cache.get(key);
  }
  // 命中依赖列表
  if (depPaths.includes(id)) {
    // 引用链中的文件都记录到缓存中
    importChain.forEach(item => cache.set(`${item}-${depPaths.join('|')}`, true));
    return true;
  }
  const moduleInfo = getModuleInfo(id);
  if (!moduleInfo || !moduleInfo.importers) {
    cache.set(key, false);
    return false;
  }
  // 核心逻辑，递归查找上层引用者
  const isInclude = moduleInfo.importers.some(
    importer => isDepInclude(importer, depPaths, importChain.concat(id), getModuleInfo)
  );
  // 设置缓存
  cache.set(key, isInclude);
  return isInclude;
};
//我们可以通过 manualChunks 提供的入参getModuleInfo来获取模块的详情moduleInfo，然后通过moduleInfo.importers拿到模块的引用者，针对每个引用者又可以递归地执行这一过程，从而获取引用链的信息。
//尽量使用缓存。由于第三方包模块数量一般比较多，对每个模块都向上查找一遍引用链会导致开销非常大，并且会产生很多重复的逻辑，使用缓存会极大加速这一过程。
```
Rollup 会对每一个模块调用 manualChunks 函数，在 manualChunks 的函数入参中你可以拿到模块 id 及模块详情信息，经过一定的处理后返回 chunk 文件的名称，这样当前 id 代表的模块便会打包到你所指定的 chunk 文件中。

#### Vite拆包解决方案 vite-plugin-chunk-split
```js
// vite.config.ts
import { chunkSplitPlugin } from 'vite-plugin-chunk-split';

export default {
  chunkSplitPlugin({
    // 指定拆包策略
    customSplitting: {
      // 1. 支持填包名。`react` 和 `react-dom` 会被打包到一个名为`render-vendor`的 chunk 里面(包括它们的依赖，如 object-assign)
      'react-vendor': ['react', 'react-dom'],
      // 2. 支持填正则表达式。src 中 components 和 utils 下的所有文件被会被打包为`component-util`的 chunk 中
      'components-util': [/src\/components/, /src\/utils/]
    }
  })
}
```


### 语法降级与Polyfill
1. 语法降级:比如某些浏览器不支持箭头函数，我们就需要将其转换为function(){}语法
2. Polyfill本身可以翻译为垫片，也就是为浏览器提前注入一些 API 的实现代码，如Object.entries方法的实现，这样可以保证产物可以正常使用这些 API，防止报错.
这两类问题本质上是通过前端的编译工具链(如Babel)及 JS 的基础 Polyfill 库(如corejs)来解决的，不会跟具体的构建工具所绑定。也就是说，对于这些本质的解决方案，在其它的构建工具(如 Webpack)能使用，在 Vite 当中也完全可以使用。
#### 底层工具链
1. **工具概览**
  * 编译时工具: 代表工具有**babel/preset-env**和**babel/plugin-transform-runtime**
  * 运行时基础库: 代表库包括**core-js**和**regenerator-runtime**
编译时工具的作用是在**代码编译阶段**进行**语法降级**及**添加 polyfill 代码**的引用语句，如:
由于这些工具只是编译阶段用到，运行时并不需要，我们需要将其放入package.json中的devDependencies中。
而运行时基础库是根据 ESMAScript官方语言规范提供各种Polyfill实现代码，主要包括**core-js**和**regenerator-runtime**两个基础库，不过在 babel 中也会有一些上层的封装，包括：@babel/polyfill、@babel/runtime、@babel/runtime-corejs2、@babel/runtime-corejs3

2. 实际使用
```js
pnpm i @babel/cli @babel/core @babel/preset-env
```
* **@babel/cli**: 为 babel 官方的脚手架工具，很适合我们练习用。
* **@babel/core**: babel 核心编译库。
* **@babel/preset-env**: babel 的预设工具集，基本为 babel 必装的库
```js
const func = async () => {
  console.log(12123)
}

Promise.resolve().finally();
```
你可以看到，示例代码中既包含了高级语法也包含现代浏览器的API，正好可以针对语法降级和 Polyfill 注入两个功能进行测试。

接下来新建.babelrc.json即 babel 的配置文件，内容如下:

```js
{
  "presets": [
    [
      "@babel/preset-env", 
      {
        // 指定兼容的浏览器版本
        "targets": {
          "ie": "11"
        },
        // 基础库 core-js 的版本，一般指定为最新的大版本
        "corejs": 3,
        // Polyfill 注入策略，usage按需导入,entry需要在代码中添加core库,全量导入
        "useBuiltIns": "usage",
        // 不将 ES 模块语法转换为其他模块语法
        "modules": false
      }
    ]
  ]
}
```
其中有两个比较关键的配置: **targets**和**usage**
我们可以通过**targets**参数指定要兼容的浏览器版本，你既可以填如上配置所示的一个对象:
```js
{
  "targets": {
    "ie": "11"
  }
}
//也可以用 Browserslist 配置语法:
{ 
  // ie 不低于 11 版本，全球超过 0.5% 使用，且还在维护更新的浏览器
  "targets": "ie >= 11, > 0.5%, not dead"
}
```
**局限性**:
1. 如果使用新特性，往往是通过基础库(如 core-js)往全局环境添加 Polyfill，如果是开发应用没有任何问题，如果是开发第三方工具库，则很可能会对全局空间造成污染。
2. 很多工具函数的实现代码(如上面示例中的_defineProperty方法)，会在许多文件中重现出现，造成文件体积冗余。
#### 更优的polyfill注入方案:transform-runtime
需要提前说明的是，transform-runtime方案可以作为@babel/preset-env中useBuiltIns配置的替代品，也就是说，一旦使用transform-runtime方案，你应该把useBuiltIns属性设为 false
```js
//安装依赖
pnpm i @babel/plugin-transform-runtime -D//编译时工具，用来转换语法和添加 Polyfill
pnpm i @babel/runtime-corejs3 -S//是运行时基础库，封装了core-js、regenerator-runtime和各种语法转换用到的工具函数。
```
core-js 有三种产物，分别是**core-js**、**core-js-pure**和**core-js-bundle**。第一种是**全局 Polyfill**的做法，@babel/preset-env 就是用的这种产物；第二种不会把 Polyfill 注入到全局环境，可以**按需引入**；第三种是**打包好的版本**，包含所有的 Polyfill，不太常用。@babel/runtime-corejs3 使用的是第二种产物。
接着我们对.babelrc.json作如下的配置:
```js
{
  "plugins": [
    // 添加 transform-runtime 插件
    [
      "@babel/plugin-transform-runtime", 
      {
        "corejs": 3
      }
    ]
  ],
  "presets": [
    [
      "@babel/preset-env", 
      {
        "targets": {
          "ie": "11"
        },
        "corejs": 3,
        // 关闭 @babel/preset-env 默认的 Polyfill 注入
        "useBuiltIns": false,
        "modules": false
      }
    ]
  ]
}
```
#### Vite语法降级与Polyfill注入
Vite 官方已经为我们封装好了一个开箱即用的方案: @vitejs/plugin-legacy，我们可以基于它来解决项目语法的浏览器兼容问题。这个插件内部同样使用 @babel/preset-env 以及 core-js等一系列基础库来进行语法降级和 Polyfill 注入

**插件使用**
```js
pnpm i @vitejs/plugin-legacy -D
```
项目中配置
```js
// vite.config.ts
import legacy from '@vitejs/plugin-legacy';
import { defineConfig } from 'vite'

export default defineConfig({
  plugins: [
    // 省略其它插件
    legacy({
      // 设置目标浏览器，browserslist 配置语法
      targets: ['ie >= 11'],
    })
  ]
})

```
相比一般的打包过程，多出了index-legacy.js、vendor-legacy.js以及polyfills-legacy.js三份产物文件。让我们继续观察一下index.html的产物内容:
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/assets/favicon.17e50649.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Vite App</title>
    <!-- 1. Modern 模式产物 -->
    <script type="module" crossorigin src="/assets/index.c1383506.js"></script>
    <link rel="modulepreload" href="/assets/vendor.0f99bfcc.js">
    <link rel="stylesheet" href="/assets/index.91183920.css">
  </head>
  <body>
    <div id="root"></div>
    <!-- 2. Legacy 模式产物 -->
    <script nomodule>兼容 iOS nomodule 特性的 polyfill，省略具体代码</script>
    <script nomodule id="vite-legacy-polyfill" src="/assets/polyfills-legacy.36fe2f9e.js"></script>
    <script nomodule id="vite-legacy-entry" data-src="/assets/index-legacy.c3d3f501.js">System.import(document.getElementById('vite-legacy-entry').getAttribute('data-src'))</script>
  </body>
</html>
```
通过官方的legacy插件， Vite 会分别打包出Modern模式和Legacy模式的产物，然后将两种产物插入同一个 HTML 里面，Modern产物被放到 type="module"的 script 标签中，而Legacy产物则被放到带有 nomodule 的 script 标签中。浏览器的加载策略如下图所示:
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/20220725215616.png)
这样产物便就能够同时放到现代浏览器和不支持type="module"的低版本浏览器当中执行。当然，在具体的代码语法层面，插件还需要考虑语法降级和 Polyfill 按需注入的问题，接下来我们就来分析一下 Vite 的官方legacy插件是如何解决这些问题的。

#### 插件执行原理
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/20220725215849.png)

首先是在**configResolved**钩子中调整了**output**属性，这么做的目的是让 Vite 底层使用的打包引擎 Rollup 能另外打包出一份**Legacy 模式**的产物，实现代码如下:
```js
const createLegacyOutput = (options = {}) => {
  return {
    ...options,
    // system 格式产物
    format: 'system',
    // 转换效果: index.[hash].js -> index-legacy.[hash].js
    entryFileNames: getLegacyOutputFileName(options.entryFileNames),
    chunkFileNames: getLegacyOutputFileName(options.chunkFileNames)
  }
}
//打包两份
const { rollupOptions } = config.build
const { output } = rollupOptions
if (Array.isArray(output)) {
  rollupOptions.output = [...output.map(createLegacyOutput), ...output]
} else {
  rollupOptions.output = [createLegacyOutput(output), output || {}]
}

//接着，在renderChunk阶段，插件会对 Legacy 模式产物进行语法转译和 Polyfill 收集，值得注意的是，这里并不会真正注入Polyfill，而仅仅只是收集Polyfill，:
{
  renderChunk(raw, chunk, opts) {
    // 1. 使用 babel + @babel/preset-env 进行语法转换与 Polyfill 注入
    // 2. 由于此时已经打包后的 Chunk 已经生成
    //   这里需要去掉 babel 注入的 import 语句，并记录所需的 Polyfill
    // 3. 最后的 Polyfill 代码将会在 generateBundle 阶段生成
  }
}

//回到 Vite 构建的主流程中，接下来会进入generateChunk钩子阶段，现在 Vite 会对之前收集到的Polyfill进行统一的打包，实现也比较精妙，主要逻辑集中在buildPolyfillChunk函数中:
// 打包 Polyfill 代码
async function buildPolyfillChunk(
  name,
  imports
  bundle,
  facadeToChunkMap,
  buildOptions,
  externalSystemJS
) {
  let { minify, assetsDir } = buildOptions
  minify = minify ? 'terser' : false
  // 调用 Vite 的 build API 进行打包
  const res = await build({
    // 根路径设置为插件所在目录
    // 由于插件的依赖包含`core-js`、`regenerator-runtime`这些运行时基础库
    // 因此这里 Vite 可以正常解析到基础 Polyfill 库的路径
    root: __dirname,
    write: false,
    // 这里的插件实现了一个虚拟模块
    // Vite 对于 polyfillId 会返回所有 Polyfill 的引入语句
    plugins: [polyfillsPlugin(imports, externalSystemJS)],
    build: {
      rollupOptions: {
        // 访问 polyfillId
        input: {
          // name 暂可视作`polyfills-legacy`
          // pofyfillId 为一个虚拟模块，经过插件处理后会拿到所有 Polyfill 的引入语句
          [name]: polyfillId
        },
      }
    }
  });
  // 拿到 polyfill 产物 chunk
  const _polyfillChunk = Array.isArray(res) ? res[0] : res
  if (!('output' in _polyfillChunk)) return
  const polyfillChunk = _polyfillChunk.output[0]
  // 后续做两件事情:
  // 1. 记录 polyfill chunk 的文件名，方便后续插入到 Modern 模式产物的 HTML 中；
  // 2. 在 bundle 对象上手动添加 polyfill 的 chunk，保证产物写到磁盘中
}
```
因此，你可以理解为这个函数的作用即通过 vite build 对renderChunk中收集到 polyfill 代码进行打包，生成一个单独的 chunk:
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/20220725220358.png)
现在我们已经能够拿到 Legacy 模式的产物文件名及 Polyfill Chunk 的文件名，那么就可以通过transformIndexHtml钩子来将这些产物插入到 HTML 的结构中:
```js
{
  transformIndexHtml(html) {
    // 1. 插入 Polyfill chunk 对应的 <script nomodule> 标签
    // 2. 插入 Legacy 产物入口文件对应的 <script nomodule> 标签
  }
}
```

### 代码联邦
#### 模块共享之痛
对于一个互联网产品来说，一般会有不同的细分应用，比如腾讯文档可以分为word、excel、ppt等等品类，抖音 PC 站点可以分为短视频站点、直播站点、搜索站点等子站点，而每个子站又彼此独立，可能由不同的开发团队进行单独的开发和维护，看似没有什么问题，但实际上会经常遇到一些模块共享的问题，也就是说不同应用中总会有一些共享的代码，比如公共组件、公共工具函数、公共第三方依赖等等。对于这些共享的代码，除了通过简单的复制粘贴，还有没有更好的复用手段？
1. 发布npm包
发布 npm 包是一种常见的复用模块的做法，我们可以将一些公用的代码封装为一个 npm 包，具体的发布更新流程是这样的:
  1. 公共库 lib1 改动，发布到 npm；
  2. 所有的应用安装新的依赖，并进行联调。
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/20220725221257.png)
封装 npm 包可以解决模块复用的问题，但它本身又引入了新的问题:
**问题:**
开发效率问题。每次改动都需要发版，并所有相关的应用安装新依赖，流程比较复杂。
项目构建问题。引入了公共库之后，公共库的代码都需要打包到项目最后的产物后，导致产物体积偏大，构建速度相对较慢。
2. Git Submodule
通过 git submodule 的方式，我们可以将代码封装成一个公共的 Git 仓库，然后复用到不同的应用中，但也需要经历如下的步骤：
   1. 公共库 lib1 改动，提交到 Git 远程仓库；
   2. 所有的应用通过git submodule命令更新子仓库代码，并进行联调。
你可以看到，整体的流程其实跟发 npm 包相差无几，仍然存在 npm 包方案所存在的各种问题。
3. 依赖外部化(external) + CDN引入
按照这个思路,我们可以在构建引擎中对某些依赖声明**external**
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/src/favicon.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Vite App</title>
  </head>
  <body>
    <div id="root"></div>
    <!-- 从 CDN 上引入第三方依赖的代码 -->
    <script src="https://cdn.jsdelivr.net/npm/react@17.0.2/index.min.js"><script>
    <script src="https://cdn.jsdelivr.net/npm/react-dom@17.0.2/index.min.js"><script>
  </body>
</html>
```
**局限性**
1. **兼容性问题**:并不是所有的依赖都有 UMD 格式的产物，因此这种方案不能覆盖所有的第三方 npm 包。
2. **依赖顺序问题**:我们通常需要考虑间接依赖的问题，如对于 antd 组件库，它本身也依赖了 react 和 moment，那么react和moment 也需要 external，并且在 HTML 中引用这些包，同时也要严格保证引用的顺序，比如说moment如果放在了antd后面，代码可能无法运行。而第三方包背后的间接依赖数量一般很庞大，如果逐个处理，对于开发者来说简直就是噩梦。
3. **产物体积问题**:由于依赖包被声明external之后，应用在引用其 CDN 地址时，会全量引用依赖的代码，这种情况下就没有办法通过 Tree Shaking 来去除无用代码了，会导致应用的性能有所下降。

4. Monorepo
作为一种新的项目管理方式，Monorepo 也可以很好地解决模块复用的问题。在 Monorepo 架构下，多个项目可以放在同一个 Git 仓库中，各个互相依赖的子项目通过软链的方式进行调试，代码复用显得非常方便，如果有依赖的代码变动，那么用到这个依赖的项目当中会立马感知到。

**局限性**:
1. 所有的应用代码必须放到同一个仓库。如果是旧有项目，并且每个应用使用一个 Git 仓库的情况，那么使用 Monorepo 之后项目架构调整会比较大，也就是说改造成本会相对比较高。
2. Monorepo 本身也存在一些天然的局限性，如项目数量多起来之后依赖安装时间会很久、项目整体构建时间会变长等等，我们也需要去解决这些局限性所带来的的开发效率问题。而这项工作一般需要投入专业的人去解决，如果没有足够的人员投入或者基建的保证，Monorepo 可能并不是一个很好的选择。
3. 项目构建问题。跟 发 npm 包的方案一样，所有的公共代码都需要进入项目的构建流程中，产物体积还是会偏大。

#### MF 核心概念

模块联邦中主要有两种模块: **本地模块**和**远程模块**
本地模块即为普通模块，是当前构建流程中的一部分，而远程模块不属于当前构建流程，在本地模块的运行时进行导入，同时本地模块和远程模块可以共享某些依赖的代码，如下图所示:
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/20220726104052.png)
在模块联邦中，每个模块既可以是本地模块，导入其它的远程模块，又可以作为远程模块，被其他的模块导入。如下面这个例子所示:
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/20220726104158.png)

#### MF优势
1. **实现任意粒度的模块共享** 这里所指的模块粒度可大可小，包括第三方 npm 依赖、业务组件、工具函数，甚至可以是整个前端应用！而整个前端应用能够共享产物，代表着各个应用单独开发、测试、部署，这也是一种微前端的实现。
2. **优化构建产物体积** 远程模块可以从本地模块运行时被拉取，而不用参与本地模块的构建，可以加速构建过程，同时也能减小构建产物。
3. **运行时按需加载** 远程模块导入的粒度可以很小，如果你只想使用 app1 模块的add函数，只需要在 app1 的构建配置中导出这个函数，然后在本地模块中按照诸如import('app1/add')的方式导入即可，这样就很好地实现了模块按需加载。
4. **第三方依赖共享** 通过模块联邦中的共享依赖机制，我们可以很方便地实现在模块间公用依赖代码，从而避免以往的external + CDN 引入方案的各种问题。
模块联邦近乎完美地解决了以往模块共享的问题，甚至能够实现应用级别的共享，进而达到**微前端**的效果

#### MF应用实战
社区中已经提供了一个比较成熟的 Vite 模块联邦方案**vite-plugin-federation**这个方案基于 Vite(或者 Rollup) 实现了完整的模块联邦能力
**使用流程**
1. 远程模块通过**exposes**注册导出的模块，本地模块通过**remotes**注册远程模块地址。
2. 远程模块进行构建，并部署到云端。
3. 本地通过**import '远程模块名称/xxx'**的方式来引入远程模块，实现运行时加载。

#### MF实现原理
实现模块联邦有三大主要的要素
1. **Host模块**: 即本地模块，用来消费远程模块。
2. **Remote模块**: 即远程模块，用来生产一些模块，并暴露运行时容器供本地模块消费。
3. **Shared依赖**: 即共享依赖，用来在本地模块和远程模块中实现第三方依赖的共享。

```js
//本地模块中引入远程模块
import RemoteApp from "remote_app/App";

//Vite编译后

// 为了方便阅读，以下部分方法的函数名进行了简化
// 远程模块表
const remotesMap = {
  'remote_app':{url:'http://localhost:3001/assets/remoteEntry.js',format:'esm',from:'vite'},
  'shared':{url:'vue',format:'esm',from:'vite'}
};

async function ensure() {
  const remote = remoteMap[remoteId];
  // 做一些初始化逻辑，暂时忽略
  // 返回的是运行时容器
}

async function getRemote(remoteName, componentName) {
  return ensure(remoteName)
    // 从运行时容器里面获取远程模块
    .then(remote => remote.get(componentName))
    .then(factory => factory());
}

// import 语句被编译成了这样
// tip: es2020 产物语法已经支持顶层 await
const __remote_appApp = await getRemote("remote_app" , "./App");

```
除了 import 语句被编译之外，在代码中还添加了**remoteMap**和一些工具函数，它们的目的很简单，就是通过访问**远端的运行时容器**来拉取对应名称的模块。
而**运行时容器**其实就是指**远程模块打包产物remoteEntry.js的导出对象**，我们来看看它的逻辑是怎样的:
```js

// remoteEntry.js
const moduleMap = {
  "./Button": () => {
    return import('./__federation_expose_Button.js').then(module => () => module)
  },
  "./App": () => {
    dynamicLoadingCss('./__federation_expose_App.css');
    return import('./__federation_expose_App.js').then(module => () => module);
  },
  './utils': () => {
    return import('./__federation_expose_Utils.js').then(module => () => module);
  }
};

// 加载 css
const dynamicLoadingCss = (cssFilePath) => {
  const metaUrl = import.meta.url;
  if (typeof metaUrl == 'undefined') {
    console.warn('The remote style takes effect only when the build.target option in the vite.config.ts file is higher than that of "es2020".');
    return
  }
  const curUrl = metaUrl.substring(0, metaUrl.lastIndexOf('remoteEntry.js'));
  const element = document.head.appendChild(document.createElement('link'));
  element.href = curUrl + cssFilePath;
  element.rel = 'stylesheet';
};

// 关键方法，暴露模块
const get =(module) => {
  return moduleMap[module]();
};

const init = () => {
  // 初始化逻辑，用于共享模块，暂时省略
}

export { dynamicLoadingCss, get, init }

```
从运行时容器的代码中我们可以得出一些关键的信息
1. **moduleMap**用来记录导出模块的信息，所有在**exposes**参数中声明的模块都会打包成单独的文件，然后通过**dynamic import** 进行导入。
2. 容器导出了十分关键的get方法，让本地模块能够通过调用这个方法来访问到该远程模块
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/20220726110007.png)

#### MF共享依赖的实现
本地模块设置了**shared: ['vue']**参数之后，当它执行远程模块代码的时候，一旦遇到了引入vue的情况，会优先使用本地的 vue，而不是远端模块中的vue。

**本地模块编译后的ensure函数逻辑**
```js
// host
// 下面是共享依赖表。每个共享依赖都会单独打包
const shareScope = {
  'vue':{'3.2.31':{get:()=>get('./__federation_shared_vue.js'), loaded:1}}
};
async function ensure(remoteId) {
  const remote = remotesMap[remoteId];
  if (remote.inited) {
    return new Promise(resolve => {
      .then(lib => {
        // lib 即运行时容器对象
        if (!remote.inited) {
          remote.lib = lib;
          remote.lib.init(shareScope);
          remote.inited = true;
        }
        resolve(remote.lib);
    });
  })
  }
}

//可以发现，ensure函数的主要逻辑是将共享依赖信息传递给远程模块的运行时容器，并进行容器的初始化。接下来我们进入容器初始化的逻辑init中:

const init =(shareScope) => {
  globalThis.__federation_shared__= globalThis.__federation_shared__|| {};
  // 下面的逻辑大家不用深究，作用很简单，就是将本地模块的`共享模块表`绑定到远程模块的全局 window 对象上
  Object.entries(shareScope).forEach(([key, value]) => {
    const versionKey = Object.keys(value)[0];
    const versionValue = Object.values(value)[0];
    const scope = versionValue.scope || 'default';
    globalThis.__federation_shared__[scope] = globalThis.__federation_shared__[scope] || {};
    const shared= globalThis.__federation_shared__[scope];
    (shared[key] = shared[key]||{})[versionKey] = versionValue;
  });
};
//当本地模块的共享依赖表能够在远程模块访问时，远程模块内也就能够使用本地模块的依赖(如 vue)了。
//现在我们来看看远程模块中对于import { h } from 'vue'这种引入代码被转换成了什么样子:

// __federation_expose_Button.js
import {importShared} from './__federation_fn_import.js'
const { h } = await importShared('vue')

//第三方依赖模块的处理逻辑都集中到了 importShared 函数
不难看到，第三方依赖模块的处理逻辑都集中到了 importShared 函数，让我们来一探究竟:

// __federation_fn_import.js
const moduleMap= {
  'vue': {
     get:()=>()=>__federation_import('./__federation_shared_vue.js'),
     import:true
   }
};
// 第三方模块缓存
const moduleCache = Object.create(null);
async function importShared(name,shareScope = 'default') {
  return moduleCache[name] ? 
    new Promise((r) => r(moduleCache[name])) : 
    getProviderSharedModule(name, shareScope);
}

async function getProviderSharedModule(name, shareScope) {
  // 从 window 对象中寻找第三方包的包名，如果发现有挂载，则获取本地模块的依赖
  if (xxx) {
    return await getHostDep();
  } else {
    return getConsumerSharedModule(name); 
  }
}

async function getConsumerSharedModule(name , shareScope) {
  if (moduleMap[name]?.import) {
    const module = (await moduleMap[name].get())();
    moduleCache[name] = module;
    return module;
  } else {
    console.error(`consumer config import=false,so cant use callback shared module`);
  }
}
```

### 性能优化
对于项目的加载性能优化而言，常见的优化手段可以分为下面三类:
1. 网络优化。包括 HTTP2、DNS 预解析(prefetch,preconnect)、Preload、Prefetch等手段。 
2. 资源优化。包括构建产物分析、资源压缩(JS代码,CSS代码,图片文件(imagemin))、产物拆包(manualChunks)、按需加载等优化方式。
3. 预渲染优化，本文主要介绍服务端渲染(SSR)和静态站点生成(SSG)两种手段。

## Vite 源码阶段
### 配置解析服务
vite中的配置解析由[resolveConfig]()函数来实现

1. **加载配置文件**
```js
// 这里的 config 是命令行指定的配置，如 vite --configFile=xxx
let { configFile } = config
if (configFile !== false) {
  // 默认都会走到下面加载配置文件的逻辑，除非你手动指定 configFile 为 false
  const loadResult = await loadConfigFromFile(
    configEnv,
    configFile,
    config.root,
    config.logLevel
  )
  if (loadResult) {
    // 解析配置文件的内容后，和命令行配置合并
    config = mergeConfig(loadResult.config, config)
    configFile = loadResult.path
    //记录configFileDependencies,因为配置文件代码可能会有第三方库的依赖,所以当第三方库依赖的代码更改时,Vite可以通过HMR处理逻辑中记录configFileDependencies检测到更改,再重启DevServer,来保证当前生效的配置永远是最新的.
    configFileDependencies = loadResult.dependencies
  }
}
```
2. **解析用户插件**
第二个重点环节是**解析用户插件**。首先，我们通过 apply 参数 过滤出需要生效的用户插件。为什么这么做呢？因为有些插件只在开发阶段生效，或者说只在生产环境生效，我们可以通过 apply: 'serve' 或 'build' 来指定它们，同时也可以将apply配置为一个函数，来自定义插件生效的条件。解析代码如下:
```js
// resolve plugins
const rawUserPlugins = (config.plugins || []).flat().filter((p) => {
  if (!p) {
    return false
  } else if (!p.apply) {
    return true
  } else if (typeof p.apply === 'function') {
     // apply 为一个函数的情况
    return p.apply({ ...config, mode }, configEnv)
  } else {
    return p.apply === command
  }
}) as Plugin[]
// 对用户插件进行排序
const [prePlugins, normalPlugins, postPlugins] =
  sortUserPlugins(rawUserPlugins)
```
接着，Vite 会拿到这些过滤且排序完成的插件，依次调用插件 config 钩子，进行配置合并:
```js
// run config hooks
const userPlugins = [...prePlugins, ...normalPlugins, ...postPlugins]
for (const p of userPlugins) {
  if (p.config) {
    const res = await p.config(config, configEnv)
    if (res) {
      // mergeConfig 为具体的配置合并函数，大家有兴趣可以阅读一下实现
      config = mergeConfig(config, res)
    }
  }
}

```
然后解析项目的根目录即 root 参数，默认取 process.cwd()的结果:
```js
// resolve root
const resolvedRoot = normalizePath(
  config.root ? path.resolve(config.root) : process.cwd()
)

```
紧接着处理**alias**，这里需要加上一些内置的 alias 规则，如`@vite/env`、`@vite/client`这种直接重定向到 Vite 内部的模块:
```js
// resolve alias with internal client alias
const resolvedAlias = mergeAlias(
  clientAlias,
  config.resolve?.alias || config.alias || []
)

const resolveOptions: ResolvedConfig['resolve'] = {
  dedupe: config.dedupe,
  ...config.resolve,
  alias: resolvedAlias
}
```
3. **加载环境变量**
```js
// load .env files
const envDir = config.envDir
  ? normalizePath(path.resolve(resolvedRoot, config.envDir))
  : resolvedRoot
const userEnv =
  inlineConfig.envFile !== false &&
  loadEnv(mode, envDir, resolveEnvPrefix(config))
//loadEnv 其实就是扫描 process.env 与 .env文件，解析出 env 对象，值得注意的是，这个对象的属性最终会被挂载到import.meta.env 这个全局对象上。
```
解析 env 对象的实现思路如下:

1. 遍历 process.env 的属性，拿到指定前缀开头的属性（默认指定为VITE_），并挂载 env 对象上

2. 遍历 .env 文件，解析文件，然后往 env 对象挂载那些以指定前缀开头的属性。遍历的文件先后顺序如下(下面的 mode 开发阶段为 development，生产环境为production):
  * .env.${mode}.local
  * .env.${mode}
  * .env.local
  * .env
3. 特殊情况: 如果中途遇到 NODE_ENV 属性，则挂到 process.env.VITE_USER_NODE_ENV，Vite 会优先通过这个属性来决定是否走生产环境的构建。
接下来是对资源公共路径即**base URL**的处理，逻辑集中在 resolveBaseUrl 函数当中:
```js
// 解析 base url
const BASE_URL = resolveBaseUrl(config.base, command === 'build', logger)
// 解析生产环境构建配置
const resolvedBuildOptions = resolveBuildOptions(config.build)
```
**resolveBaseUrl**里面有这些处理规则需要注意:
* 空字符或者 ./ 在开发阶段特殊处理，全部重写为/
* .开头的路径，自动重写为 /
* 以http(s)://开头的路径，在开发环境下重写为对应的 pathname
* 确保路径开头和结尾都是/
当然，还有对cacheDir的解析，这个路径相对于在 Vite 预编译时写入依赖产物的路径:
```js
// resolve cache directory
const pkgPath = lookupFile(resolvedRoot, [`package.json`], true /* pathOnly */)
// 默认为 node_module/.vite
const cacheDir = config.cacheDir
  ? path.resolve(resolvedRoot, config.cacheDir)
  : pkgPath && path.join(path.dirname(pkgPath), `node_modules/.vite`)
```
紧接着处理用户配置的**assetsInclude**，将其转换为一个过滤器函数:
```js
const assetsFilter = config.assetsInclude
  ? createFilter(config.assetsInclude)
  : () => false

```
Vite 后面会将用户传入的 assetsInclude 和内置的规则合并:
```js
assetsInclude(file: string) {
  return DEFAULT_ASSETS_RE.test(file) || assetsFilter(file)
}
//这个配置决定是否让 Vite 将对应的后缀名视为静态资源文件（asset）来处理。
```
4. 路径解析器工厂
**路径解析器**:是指调用插件容器进行路径解析的函数。代码结构是这个样子的:
```js
const createResolver: ResolvedConfig['createResolver'] = (options) => {
  let aliasContainer: PluginContainer | undefined
  let resolverContainer: PluginContainer | undefined
  // 返回的函数可以理解为一个解析器
  return async (id, importer, aliasOnly, ssr) => {
    let container: PluginContainer
    if (aliasOnly) {
      container =
        aliasContainer ||
        // 新建 aliasContainer
    } else {
      container =
        resolverContainer ||
        // 新建 resolveContainer
    }
    return (await container.resolveId(id, importer, undefined, ssr))?.id
  }
}

//这个解析器未来会在依赖预构建的时候用上，具体用法如下:
const resolve = config.createResolver()
// 调用以拿到 react 路径
rseolve('react', undefined, undefined, false)
/**
 * 这里有aliasContainer和resolverContainer两个工具对象
 * 它们都含有resolveId这个专门解析路径的方法，可以被 Vite 调用来获取解析结果。
 * */
```
接着会顺便处理一个 public 目录，也就是 Vite 作为静态资源服务的目录:
```js
const { publicDir } = config
const resolvedPublicDir =
  publicDir !== false && publicDir !== ''
    ? path.resolve(
        resolvedRoot,
        typeof publicDir === 'string' ? publicDir : 'public'
      )
    : ''
```
至此，配置已经基本上解析完成，最后通过 resolved 对象来整理一下:
```js
const resolved: ResolvedConfig = {
  ...config,
  configFile: configFile ? normalizePath(configFile) : undefined,
  configFileDependencies,
  inlineConfig,
  root: resolvedRoot,
  base: BASE_URL
  // 其余配置不再一一列举
```
#### 加载配置文件详解
```js
const loadResult = await loadConfigFromFile(/*省略传参*/)
```
处理的配置文件类型，根据文件后缀和模块格式可以分为下面这几类:
1. TS + ESM 格式
2. TS + CommonJS 格式
3. JS + ESM 格式
4. JS + CommonJS 格式
那么，Vite 是如何加载配置文件的？一共分两个步骤:
1. 识别出配置文件的类别
2. 根据不同的类别分别解析出配置内容

1. **识别配置文件的类型**
```js
//首先 Vite 会检查项目的 `package.json`,如果有`type: "module"`则打上**isESM**的标识:

try {
  const pkg = lookupFile(configRoot, ['package.json'])
  if (pkg && JSON.parse(pkg).type === 'module') {
    isMjs = true
  }
} catch (e) {}

//然后，Vite 会寻找配置文件路径，代码简化后如下:
let isTS = false
let isESM = false
let dependencies: string[] = []
// 如果命令行有指定配置文件路径
if (configFile) {
  resolvedPath = path.resolve(configFile)
  // 根据后缀判断是否为 ts 或者 esm，打上 flag
  isTS = configFile.endsWith('.ts')
  if (configFile.endsWith('.mjs')) {
      isESM = true
    }
} else {
  // 从项目根目录寻找配置文件路径，寻找顺序:
  // - vite.config.js
  // - vite.config.mjs
  // - vite.config.ts
  // - vite.config.cjs
  const jsconfigFile = path.resolve(configRoot, 'vite.config.js')
  if (fs.existsSync(jsconfigFile)) {
    resolvedPath = jsconfigFile
  }

  if (!resolvedPath) {
    const mjsconfigFile = path.resolve(configRoot, 'vite.config.mjs')
    if (fs.existsSync(mjsconfigFile)) {
      resolvedPath = mjsconfigFile
      isESM = true
    }
  }

  if (!resolvedPath) {
    const tsconfigFile = path.resolve(configRoot, 'vite.config.ts')
    if (fs.existsSync(tsconfigFile)) {
      resolvedPath = tsconfigFile
      isTS = true
    }
  }
  
  if (!resolvedPath) {
    const cjsConfigFile = path.resolve(configRoot, 'vite.config.cjs')
    if (fs.existsSync(cjsConfigFile)) {
      resolvedPath = cjsConfigFile
      isESM = false
    }
  }
}
```
2. **根据类别解析配置**
**ESM格式**
```js
//对于 ESM 格式配置的处理代码如下：
let userConfig: UserConfigExport | undefined

if (isESM) {
  const fileUrl = require('url').pathToFileURL(resolvedPath)
  //首先通过 Esbuild 将配置文件编译打包成 js 代码:
  const bundled = await bundleConfigFile(resolvedPath, true)
  //记录依赖
  dependencies = bundled.dependencies
  // TS + ESM
  if (isTS) {
    /**
     * 对于 TS 配置文件来说，Vite 会将编译后的 js 代码写入临时文件，通过 Node 原生 ESM Import 来读取这个临时的内容，以获取到配置内容，再直接删掉临时文件
     * */
    fs.writeFileSync(resolvedPath + '.js', bundled.code)
    userConfig = (await dynamicImport(`${fileUrl}.js?t=${Date.now()}`))
      .default
    fs.unlinkSync(resolvedPath + '.js')
    debug(`TS + native esm config loaded in ${getTime()}`, fileUrl)
  } 
  //  JS + ESM
  else {
    //而对于 JS 配置文件来说，Vite 会直接通过 Node 原生 ESM Import 来读取，也是使用 dynamicImport 函数的逻辑
    userConfig = (await dynamicImport(`${fileUrl}?t=${Date.now()}`)).default
    debug(`native esm config loaded in ${getTime()}`, fileUrl)
  }
}

//dynamicImport 的实现如下:
export const dynamicImport = new Function('file', 'return import(file)')
//为什么用new Function包裹?为了避免打包工具处理这段代码
//为什么import路径结果要加上时间戳query?为了让 dev server 重启后仍然读取最新的配置，避免缓存。
```
**CommonJS 格式**
```js
//对于 CommonJS 格式的配置文件，Vite 集中进行了解析:

// 对于 js/ts 均生效
// 使用 esbuild 将配置文件编译成 commonjs 格式的 bundle 文件
const bundled = await bundleConfigFile(resolvedPath)
dependencies = bundled.dependencies
// 加载编译后的 bundle 代码
userConfig = await loadConfigFromBundledFile(resolvedPath, bundled.code)
//bundleConfigFile 的逻辑上文中已经说了，主要是通过 Esbuild 将配置文件打包，拿到打包后的 bundle 代码以及配置文件的依赖(dependencies)

//加载bundle代码,这也是loadConfigFromBundledFile要做的事
////大体的思路是通过拦截原生 require.extensions 的加载函数来实现对 bundle 后配置代码的加载。代码如下:
async function loadConfigFromBundledFile(
  fileName: string,
  bundledCode: string
): Promise<UserConfig> {
  const extension = path.extname(fileName)
  // 默认加载器
  const defaultLoader = require.extensions[extension]!
  // 拦截原生 require 对于`.js`或者`.ts`的加载
  require.extensions[extension] = (module: NodeModule, filename: string) => {
    // 针对 vite 配置文件的加载特殊处理
    if (filename === fileName) {
      ;(module as NodeModuleWithCompile)._compile(bundledCode, filename)
    } else {
      defaultLoader(module, filename)
    }
  }
  // 清除 require 缓存
  delete require.cache[require.resolve(fileName)]
  const raw = require(fileName)
  const config = raw.__esModule ? raw.default : raw
  require.extensions[extension] = defaultLoader
  return config
}
//而原生 require 对于 js 文件的加载代码是这样的:
Module._extensions['.js'] = function (module, filename) {
  var content = fs.readFileSync(filename, 'utf8')
  module._compile(stripBOM(content), filename)
}

//Node.js 内部也是先读取文件内容，然后编译该模块。当代码中调用 module._compile 相当于手动编译一个模块，该方法在 Node 内部的实现如下:
Module.prototype._compile = function (content, filename) {
  var self = this
  var args = [self.exports, require, self, filename, dirname]
  return compiledWrapper.apply(self.exports, args)
}
//等同于下面的形式:
(function (exports, require, module, __filename, __dirname) {
  // 执行 module._compile 方法中传入的代码
  // 返回 exports 对象
})

//在调用完 module._compile 编译完配置代码后，进行一次手动的 require，即可拿到配置对象:
const raw = require(fileName)
const config = raw.__esModule ? raw.default : raw
// 恢复原生的加载方法
require.extensions[extension] = defaultLoader
// 返回配置
return config
```

### 依赖预构建
关于预构建的实现代码都在[optimizeDeps](packages/vite/src/node/optimizer/index.ts)

#### 缓存判断
Vite 在每次预构建之后都将一些关键信息写入到了**_metadata.json**文件中，第二次启动项目时会通过这个文件中的 hash 值来进行缓存的判断，如果命中缓存则不会进行后续的预构建流程
```js
// _metadata.json 文件所在的路径
const dataPath = path.join(cacheDir, "_metadata.json");
// 根据当前的配置计算出哈希值
const mainHash = getDepHash(root, config);
const data: DepOptimizationMetadata = {
  hash: mainHash,
  browserHash: mainHash,
  optimized: {},
};
// 默认走到里面的逻辑
if (!force) {
  let prevData: DepOptimizationMetadata | undefined;
  try {
    // 读取元数据
    prevData = JSON.parse(fs.readFileSync(dataPath, "utf-8"));
  } catch (e) {}
  // 当前计算出的哈希值与 _metadata.json 中记录的哈希值一致，表示命中缓存，不用预构建
  if (prevData && prevData.hash === data.hash) {
    log("Hash is consistent. Skipping. Use --force to override.");
    return prevData;
  }
}
```
哈希值计算的策略:决定哪些配置和文件有可能影响预构建的结果，然后根据这些信息来生成哈希值
```js
const lockfileFormats = ["package-lock.json", "yarn.lock", "pnpm-lock.yaml"];
function getDepHash(root: string, config: ResolvedConfig): string {
  // 获取 lock 文件内容
  let content = lookupFile(root, lockfileFormats) || "";
  // 除了 lock 文件外，还需要考虑下面的一些配置信息
  content += JSON.stringify(
    {
      // 开发/生产环境
      mode: config.mode,
      // 项目根路径
      root: config.root,
      // 路径解析配置
      resolve: config.resolve,
      // 自定义资源类型
      assetsInclude: config.assetsInclude,
      // 插件
      plugins: config.plugins.map((p) => p.name),
      // 预构建配置
      optimizeDeps: {
        include: config.optimizeDeps?.include,
        exclude: config.optimizeDeps?.exclude,
      },
    },
    // 特殊处理函数和正则类型
    (_, value) => {
      if (typeof value === "function" || value instanceof RegExp) {
        return value.toString();
      }
      return value;
    }
  );
  // 最后调用 crypto 库中的 createHash 方法生成哈希
  return createHash("sha256").update(content).digest("hex").substring(0, 8);
}
```
#### 依赖扫描
如果没有命中缓存，则会正式地进入依赖预构建阶段。不过 Vite 不会直接进行依赖的预构建，而是在之前探测一下项目中存在哪些依赖，收集依赖列表，也就是进行依赖扫描的过程。这个过程是必须的，因为 Esbuild 需要知道我们到底要打包哪些第三方依赖。关键代码如下
```js
({ deps, missing } = await scanImports(config));
//在scanImports方法内部主要会调用 Esbuild 提供的 build 方法:
const deps: Record<string, string> = {};
// 扫描用到的 Esbuild 插件
const plugin = esbuildScanPlugin(config, container, deps, missing, entries);
await Promise.all(
  // 应用项目入口
  entries.map((entry) =>
    build({
      absWorkingDir: process.cwd(),
      // 注意这个参数,表示产物不需要写入磁盘,这大大节省了磁盘I/O事件,也是依赖扫描为什么往往比依赖打包快很多的原因之一
      write: false,
      entryPoints: [entry],
      bundle: true,
      format: "esm",
      logLevel: "error",
      plugins: [...plugins, plugin],
      ...esbuildOptions,
    })
  )
);
```
#### 依赖打包
收集完依赖之后，就正式地进入到依赖打包的阶段了。这里也调用 Esbuild 进行打包并写入产物到磁盘中，关键代码如下:
```js
const result = await build({
  absWorkingDir: process.cwd(),
  // 所有依赖的 id 数组，在插件中会转换为真实的路径
  entryPoints: Object.keys(flatIdDeps),
  bundle: true,
  format: "esm",
  target: config.build.target || undefined,
  external: config.optimizeDeps?.exclude,
  logLevel: "error",
  splitting: true,
  sourcemap: true,
  outdir: cacheDir,
  ignoreAnnotations: true,
  metafile: true,
  define,
  plugins: [
    ...plugins,
    // 预构建专用的插件
    esbuildDepPlugin(flatIdDeps, flatIdToExports, config, ssr),
  ],
  ...esbuildOptions,
});
// 打包元信息，后续会根据这份信息生成 _metadata.json
const meta = result.metafile!;
```
#### 元信息写入磁盘
在打包过程完成之后，Vite 会拿到 Esbuild 构建的元信息，也就是上面代码中的meta对象，然后将元信息保存到_metadata.json文件中:
```js
const data: DepOptimizationMetadata = {
  hash: mainHash,
  browserHash: mainHash,
  optimized: {},
};
// 省略中间的代码
for (const id in deps) {
  const entry = deps[id];
  data.optimized[id] = {
    file: normalizePath(path.resolve(cacheDir, flattenId(id) + ".js")),
    src: entry,
    // 判断是否需要转换成 ESM 格式，后面会介绍
    needsInterop: needsInterop(
      id,
      idToExports[id],
      meta.outputs,
      cacheDirOutputPath
    ),
  };
}
// 元信息写磁盘
writeFile(dataPath, JSON.stringify(data, null, 2));
```
#### 依赖扫描详细分析
**如何获取入口**
现在让我们把目光聚焦在scanImports的实现上。大家可以先想一想，在进行依赖扫描之前，需要做的第一件事是什么？很显然，是找到入口文件。但入口文件可能存在于多个配置当中，比如**optimizeDeps.entries**和**build.rollupOptions.input**，同时需要考虑数组和对象的情况；也可能用户没有配置，需要自动探测入口文件。那么，在scanImports是如何做到的呢？
```js
const explicitEntryPatterns = config.optimizeDeps.entries;
const buildInput = config.build.rollupOptions?.input;
if (explicitEntryPatterns) {
  // 先从 optimizeDeps.entries 寻找入口，支持 glob 语法
  entries = await globEntries(explicitEntryPatterns, config);
} else if (buildInput) {
  // 其次从 build.rollupOptions.input 配置中寻找，注意需要考虑数组和对象的情况
  const resolvePath = (p: string) => path.resolve(config.root, p);
  if (typeof buildInput === "string") {
    entries = [resolvePath(buildInput)];
  } else if (Array.isArray(buildInput)) {
    entries = buildInput.map(resolvePath);
  } else if (isObject(buildInput)) {
    entries = Object.values(buildInput).map(resolvePath);
  } else {
    throw new Error("invalid rollupOptions.input value.");
  }
} else {
  // 兜底逻辑，如果用户没有进行上述配置，则自动从根目录开始寻找
  entries = await globEntries("**/*.html", config);
}
```
接下来我们还需要考虑入口文件的类型，一般情况下入口需要是js/ts文件，但实际上像 html、vue 单文件组件这种类型我们也是需要支持的，因为在这些文件中仍然可以包含 script 标签的内容，从而让我们搜集到依赖信息。

在源码当中，同时对 html、vue、svelte、astro(一种新兴的类 html 语法)四种后缀的入口文件进行了解析，当然，具体的解析过程在依赖扫描阶段的 Esbuild 插件中得以实现
```js
const htmlTypesRE = /.(html|vue|svelte|astro)$/;
function esbuildScanPlugin(/* 一些入参 */): Plugin {
  // 初始化一些变量
  // 返回一个 Esbuild 插件
  return {
    name: "vite:dep-scan",
    setup(build) {
      // 标记「类 HTML」文件的 namespace
      build.onResolve({ filter: htmlTypesRE }, async ({ path, importer }) => {
        return {
          path: await resolve(path, importer),
          namespace: "html",
        };
      });
      //省略了vue/svelte 以及 import.meta.glob 语法的处理
      build.onLoad(
        { filter: htmlTypesRE, namespace: "html" },
        async ({ path }) => {
          // 解析「类 HTML」文件
            let raw = fs.readFileSync(path, 'utf-8')
            // 去掉注释内容，防止干扰解析过程
            raw = raw.replace(commentRE, '<!---->')
            const isHtml = path.endsWith('.html')
            // HTML 情况下会寻找 type 为 module 的 script
            // 正则：/(<script\b[^>]*type\s*=\s*(?: module |'module')[^>]*>)(.*?)</script>/gims
            const regex = isHtml ? scriptModuleRE : scriptRE
            regex.lastIndex = 0
            let js = ''
            let loader: Loader = 'js'
            let match: RegExpExecArray | null
            // 正式开始解析
            while ((match = regex.exec(raw))) {
              // 第一次: openTag 为 <script type= module  src= /src/main.ts >, 无 content
              // 第二次: openTag 为 <script type= module >，有 content
              const [, openTag, content] = match
              const typeMatch = openTag.match(typeRE)
              const type =
                typeMatch && (typeMatch[1] || typeMatch[2] || typeMatch[3])
              const langMatch = openTag.match(langRE)
              const lang =
                langMatch && (langMatch[1] || langMatch[2] || langMatch[3])
              if (lang === 'ts' || lang === 'tsx' || lang === 'jsx') {
                // 指定 esbuild 的 loader
                loader = lang
              }
              const srcMatch = openTag.match(srcRE)
              // 根据有无 src 属性来进行不同的处理
              if (srcMatch) {
                const src = srcMatch[1] || srcMatch[2] || srcMatch[3]
                js += `import ${JSON.stringify(src)}\n`
              } else if (content.trim()) {
                js += content + '\n'
              }
          }
          return {
            loader,
            contents: js
          }

        }
      );
    },
  };
}
```

**如何记录依赖**
Vite 中会把 bare import的路径当做依赖路径，关于bare import，你可以理解为直接引入一个包名，比如下面这样:
```js
import React from "react";
//而以.开头的相对路径或者以/开头的绝对路径都不能算bare import:
// 以下都不是 bare import
import React from "../node_modules/react/index.js";
import React from "/User/sanyuan/vite-project/node_modules/react/index.js";

//对于解析 bare import、记录依赖的逻辑依然实现在 scan 插件当中:
build.onResolve(
  {
    // avoid matching windows volume
    filter: /^[\w@][^:]/,
  },
  async ({ path: id, importer }) => {
    // 如果在 optimizeDeps.exclude 列表或者已经记录过了，则将其 externalize (排除)，直接 return

    // 接下来解析路径，内部调用各个插件的 resolveId 方法进行解析
    const resolved = await resolve(id, importer);
    if (resolved) {
      // 判断是否应该 externalize，下个部分详细拆解
      if (shouldExternalizeDep(resolved, id)) {
        return externalUnlessEntry({ path: id });
      }

      if (resolved.includes("node_modules") || include?.includes(id)) {
        // 如果 resolved 为 js 或 ts 文件
        if (OPTIMIZABLE_ENTRY_RE.test(resolved)) {
          // 注意了! 现在将其正式地记录在依赖表中
          depImports[id] = resolved;
        }
        // 进行 externalize，因为这里只用扫描出依赖即可，不需要进行打包，具体实现后面的部分会讲到
        return externalUnlessEntry({ path: id });
      } else {
        // resolved 为 「类 html」 文件，则标记上 'html' 的 namespace
        const namespace = htmlTypesRE.test(resolved) ? "html" : undefined;
        // linked package, keep crawling
        return {
          path: path.resolve(resolved),
          namespace,
        };
      }
    } else {
      // 没有解析到路径，记录到 missing 表中，后续会检测这张表，显示相关路径未找到的报错
      missing[id] = normalizePath(importer);
    }
  }
);

//顺便说一句，其中调用到了resolve，也就是路径解析的逻辑，这里面实际上会调用各个插件的 resolveId 方法来进行路径的解析，代码如下所示:

const resolve = async (id: string, importer?: string) => {
  // 通过 seen 对象进行路径缓存
  const key = id + (importer && path.dirname(importer));
  if (seen.has(key)) {
    return seen.get(key);
  }
  // 调用插件容器的 resolveId
  // 关于插件容器下一节会详细介绍，这里你直接理解为调用各个插件的 resolveId 方法解析路径即可
  const resolved = await container.resolveId(
    id,
    importer && normalizePath(importer)
  );
  const res = resolved?.id;
  seen.set(key, res);
  return res;
};
```

**external 的规则如何制定？**
我把需要 external 的路径分为两类:**资源型**和**模块型**,资源性路径,一般是直接排除
```js
// data url，直接标记 external: true，不让 esbuild 继续处理
build.onResolve({ filter: dataUrlRE }, ({ path }) => ({
  path,
  external: true,
}));
// 加了 ?worker 或者 ?raw 这种 query 的资源路径，直接 external
build.onResolve({ filter: SPECIAL_QUERY_RE }, ({ path }) => ({
  path,
  external: true,
}));
// css & json
build.onResolve(
  {
    filter: /.(css|less|sass|scss|styl|stylus|pcss|postcss|json)$/,
  },
  // 非 entry 则直接标记 external
  externalUnlessEntry
);
// Vite 内置的一些资源类型，比如 .png、.wasm 等等
build.onResolve(
  {
    filter: new RegExp(`\.(${KNOWN_ASSET_TYPES.join("|")})$`),
  },
  // 非 entry 则直接标记 external
  externalUnlessEntry
);

const externalUnlessEntry = ({ path }: { path: string }) => ({
  path,
  // 非 entry 则标记 external
  external: !entries.includes(path),
});

/**
 * 
 * 模块型的路径，也就是当我们通过 resolve 函数解析出了一个 JS 模块的路径，如何判断是否应该被 externalize 呢？这部分实现主要在shouldExternalizeDep 函数中，之前在分析bare import埋了个伏笔，现在让我们看看具体的实现规则:*/
export function shouldExternalizeDep(
  resolvedId: string,
  rawId: string
): boolean {
  // 解析之后不是一个绝对路径，不在 esbuild 中进行加载
  if (!path.isAbsolute(resolvedId)) {
    return true;
  }
  // 1. import 路径本身就是一个绝对路径
  // 2. 虚拟模块(Rollup 插件中约定虚拟模块以`\0`开头)
  // 都不在 esbuild 中进行加载
  if (resolvedId === rawId || resolvedId.includes("\0")) {
    return true;
  }
  // 不是 JS 或者 类 HTML 文件，不在 esbuild 中进行加载
  if (!JS_TYPES_RE.test(resolvedId) && !htmlTypesRE.test(resolvedId)) {
    return true;
  }
  return false;
}
```
#### 依赖打包详细分析

**总结图**
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/Vite%E6%8F%92%E4%BB%B6%E6%B5%81%E6%B0%B4%E7%BA%BF.drawio.png)

为了解决嵌套目录带来的问题，Vite 做了两件事情来达到扁平化的预构建产物输出:
1. 嵌套路径扁平化，/被换成下划线，如 react/jsx-dev-runtime，被重写为react_jsx-dev-runtime
2. 用虚拟模块来代替真实模块，作为预打包的入口，具体的实现后面会详细介绍

回到optimizeDeps函数中，其中在进行完依赖扫描的步骤后，就会执行路径的扁平化操作:
```js
const flatIdDeps: Record<string, string> = {};
const idToExports: Record<string, ExportsData> = {};
const flatIdToExports: Record<string, ExportsData> = {};
// deps 即为扫描后的依赖表
// 形如: {
//    react :  /Users/sanyuan/vite-project/react/index.js  }
//    react/jsx-dev-runtime :  /Users/sanyuan/vite-project/react/jsx-dev-runtime.js
// }
for (const id in deps) {
  // 扁平化路径，`react/jsx-dev-runtime`，被重写为`react_jsx-dev-runtime`；
  const flatId = flattenId(id);
  // 填入 flatIdDeps 表，记录 flatId -> 真实路径的映射关系
  const filePath = (flatIdDeps[flatId] = deps[id]);
  const entryContent = fs.readFileSync(filePath, "utf-8");
  // 后续代码省略
}

//对于虚拟模块的处理，大家可以把目光放到 esbuildDepPlugin 函数上面，它的逻辑大致如下:
export function esbuildDepPlugin(/* 一些传参 */) {
  // 定义路径解析的方法

  // 返回 Esbuild 插件
  return {
    name: 'vite:dep-pre-bundle',
    set(build) {
      // bare import 的路径
      build.onResolve(
        { filter: /^[\w@][^:]/ },
        async ({ path: id, importer, kind }) => {
          // 判断是否为入口模块，如果是，则标记上`dep`的 namespace，成为一个虚拟模块
        }
    }

    build.onLoad({ filter: /.*/, namespace: 'dep' }, ({ path: id }) => {
      // 加载虚拟模块
    }
  }
}
```
如此一来，Esbuild 会将虚拟模块作为入口来进行打包，最后的产物目录会变成下面的扁平结构

**代理模块加载**
虚拟模块代替了真实模块作为打包入口，因此也可以理解为代理模块，后面也统一称之为代理模块。我们首先来分析一下代理模块究竟是如何被加载出来的，换句话说，它到底了包含了哪些内容。

拿`import React from "react"`来举例，Vite 会把react标记为 namespace 为 dep 的虚拟模块，然后控制 Esbuild 的加载流程，对于真实模块的内容进行重新导出。

```js
// 真实模块所在的路径，拿 react 来说，即`node_modules/react/index.js`
const entryFile = qualified[id];
// 确定相对路径
let relativePath = normalizePath(path.relative(root, entryFile));
if (
  !relativePath.startsWith("./") &&
  !relativePath.startsWith("../") &&
  relativePath !== "."
) {
  relativePath = `./${relativePath}`;
}
```
确定了路径之后，接下来就是对模块的内容进行重新导出。这里会分为两种情况: CJS模块和ES模块
我们可以暂时把目光转移到optimizeDeps中，实际上在进行真正的依赖打包之前，Vite 会读取各个依赖的入口文件，通过es-module-lexer这种工具来解析入口文件的内容。这里稍微解释一下es-module-lexer，这是一个在 Vite 被经常使用到的工具库，主要是为了解析 ES 导入导出的语法，大致用法如下:
```js
import { init, parse } from "es-module-lexer";
// 等待`es-module-lexer`初始化完成
await init;
const sourceStr = `
  import moduleA from './a';
  export * from 'b';
  export const count = 1;
  export default count;
`;
// 开始解析
const exportsData = parse(sourceStr);
// 结果为一个数组，分别保存 import 和 export 的信息
const [imports, exports] = exportsData;
// 返回 `import module from './a'`
sourceStr.substring(imports[0].ss, imports[0].se);
// 返回 ['count', 'default']
console.log(exports);
//值得注意的是, export * from 导出语法会被记录在 import 信息中。

//接下来我们来看看 optimizeDeps 中如何利用 es-module-lexer来解析入口文件的，实现代码如下:
import { init, parse } from "es-module-lexer";
// 省略中间的代码
await init;
for (const id in deps) {
  // 省略前面的路径扁平化逻辑
  // 读取入口内容
  const entryContent = fs.readFileSync(filePath, "utf-8");
  try {
    exportsData = parse(entryContent) as ExportsData;
  } catch {
    // 省略对 jsx 的处理
  }
  for (const { ss, se } of exportsData[0]) {
    const exp = entryContent.slice(ss, se);
    // 标记存在 `export * from` 语法
    if (/export\s+*\s+from/.test(exp)) {
      exportsData.hasReExports = true;
    }
  }
  // 将 import 和 export 信息记录下来
  idToExports[id] = exportsData;
  flatIdToExports[flatId] = exportsData;
}

//OK，由于最后会有两张表记录下 ES 模块导入和导出的相关信息，而flatIdToExports表会作为入参传给 Esbuild 插件:
esbuildDepPlugin(flatIdDeps, flatIdToExports, config, ssr);

let contents = "";
// 下面的 exportsData 即外部传入的模块导入导出相关的信息表
// 根据模块 id 拿到对应的导入导出信息
const data = exportsData[id];
const [imports, exports] = data;
if (!imports.length && !exports.length) {
  // 处理 CommonJS 模块
  //如果是 CommonJS 模块，则导出语句写成这种形式:
  let contents = "";
  contents += `export default require( ${relativePath} );`;
} else {
  // 处理 ES  模块
  //如果是 ES 模块，则分默认导出和非默认导出这两种情况来处理:
  // 默认导出，即存在 export default 语法
  if (exports.includes("default")) {
    contents += `import d from  ${relativePath} ;export default d;`;
  }
  // 非默认导出
  if (
    // 1. 存在 `export * from` 语法，前文分析过
    data.hasReExports ||
    // 2. 多个导出内容
    exports.length > 1 ||
    // 3. 只有一个导出内容，但这个导出不是 export default
    exports[0] !== "default"
  ) {
    // 凡是命中上述三种情况中的一种，则添加下面的重导出语句
    contents += `\nexport * from  ${relativePath} `;
  }
}
//现在，我们组装好了 代理模块 的内容，接下来就可以放心地交给 Esbuild 加载了:
let ext = path.extname(entryFile).slice(1);
if (ext === "mjs") ext = "js";
return {
  loader: ext as Loader,
  // 虚拟模块内容
  contents,
  resolveDir: root,
};

```
### Vite插件
![](https://raw.githubusercontent.com/captain1023/picGo/master/img/20220726175244.png)
* 在生产环境中 Vite 直接调用 Rollup 进行打包，所以 Rollup 可以调度各种插件；
* 在开发环境中，Vite 模拟了 Rollup 的插件机制，设计了一个PluginContainer 对象来调度各个插件

**PluginContainer**的实现基于借鉴WMR中的rollup-plugin-container.js,主要分为2个部分:
1. 实现Rollup插件钩子的调度
2. 实现插件钩子哪不的Context上下文对象

**container定义**:
```js
const container = {
  // 异步串行钩子
  options: await (async () => {
    let options = rollupOptions
    for (const plugin of plugins) {
      if (!plugin.options) continue
      options =
        (await plugin.options.call(minimalContext, options)) || options
    }
    return options;
  })(),
  // 异步并行钩子
  async buildStart() {
    await Promise.all(
      plugins.map((plugin) => {
        if (plugin.buildStart) {
          return plugin.buildStart.call(
            new Context(plugin) as any,
            container.options as NormalizedInputOptions
          )
        }
      })
    )
  },
  // 异步优先钩子
  async resolveId(rawId, importer) {
    // 上下文对象，后文介绍
    const ctx = new Context()

    let id: string | null = null
    const partial: Partial<PartialResolvedId> = {}
    for (const plugin of plugins) {
      const result = await plugin.resolveId.call(
        ctx as any,
        rawId,
        importer,
        { ssr }
      )
      if (!result) continue;
      return result;
    }
  }
  // 异步优先钩子
  async load(id, options) {
    const ctx = new Context()
    for (const plugin of plugins) {
      const result = await plugin.load.call(ctx as any, id, { ssr })
      if (result != null) {
        return result
      }
    }
    return null
  },
  // 异步串行钩子
  async transform(code, id, options) {
    const ssr = options?.ssr
    // 每次 transform 调度过程会有专门的上下文对象，用于合并 SourceMap，后文会介绍
    const ctx = new TransformContext(id, code, inMap as SourceMap)
    ctx.ssr = !!ssr
    for (const plugin of plugins) {
      let result: TransformResult | string | undefined
      try {
        result = await plugin.transform.call(ctx as any, code, id, { ssr })
      } catch (e) {
        ctx.error(e)
      }
      if (!result) continue;
      // 省略 SourceMap 合并的逻辑 
      code = result;
    }
    return {
      code,
      map: ctx._getCombinedSourcemap()
    }
  },
  // close 钩子实现省略
}
```
在各种钩子被调用的时候，Vite 会强制将钩子函数的 this 绑定为一个上下文对象
```js
const ctx = new Context()
const result = await plugin.load.call(ctx as any, id, { ssr })
```
我们知道，在 Rollup 钩子函数中，我们可以调用this.emitFile、this.resolve 等诸多的上下文方法，因此，Vite 除了要模拟各个插件的执行流程，还需要模拟插件执行的上下文对象，代码中的 Context 对象就是用来完成这件事情的。我们来看看 Context 对象的具体实现:
```js
import { RollupPluginContext } from 'rollup';
type PluginContext = Omit<
  RollupPluginContext,
  // not documented
  | 'cache'
  // deprecated
  | 'emitAsset'
  | 'emitChunk'
  | 'getAssetFileName'
  | 'getChunkFileName'
  | 'isExternal'
  | 'moduleIds'
  | 'resolveId'
  | 'load'
>

const watchFiles = new Set<string>()

class Context implements PluginContext {
  // 实现各种上下文方法
  // 解析模块 AST(调用 acorn)
  parse(code: string, opts: any = {}) {
    return parser.parse(code, {
      sourceType: 'module',
      ecmaVersion: 'latest',
      locations: true,
      ...opts
    })
  }
  // 解析模块路径
  async resolve(
    id: string,
    importer?: string,
    options?: { skipSelf?: boolean }
  ) {
    let skip: Set<Plugin> | undefined
    if (options?.skipSelf && this._activePlugin) {
      skip = new Set(this._resolveSkips)
      skip.add(this._activePlugin)
    }
    let out = await container.resolveId(id, importer, { skip, ssr: this.ssr })
    if (typeof out === 'string') out = { id: out }
    return out as ResolvedId | null
  }

  // 以下两个方法均从 Vite 的模块依赖图中获取相关的信息
  // 我们将在下一节详细介绍模块依赖图，本节不做展开
  getModuleInfo(id: string) {
    return getModuleInfo(id)
  }

  getModuleIds() {
    return moduleGraph
      ? moduleGraph.idToModuleMap.keys()
      : Array.prototype[Symbol.iterator]()
  }
  
  // 记录开发阶段 watch 的文件
  addWatchFile(id: string) {
    watchFiles.add(id)
    ;(this._addedImports || (this._addedImports = new Set())).add(id)
    if (watcher) ensureWatchedFile(watcher, id, root)
  }

  getWatchFiles() {
    return [...watchFiles]
  }
  
  warn() {
    // 打印 warning 信息
  }
  
  error() {
    // 打印 error 信息
  }
  
  // 其它方法只是声明，并没有具体实现，这里就省略了
}
```
很显然，Vite 将 Rollup 的PluginContext对象重新实现了一遍，因为只是开发阶段用到，所以去除了一些打包相关的方法实现。同时，上下文对象与 Vite 开发阶段的 ModuleGraph 即模块依赖图相结合，是为了实现开发时的 HMR.

另外，transform 钩子也会绑定一个插件上下文对象，不过这个对象和其它钩子不同，实现代码精简如下:
```js
class TransformContext extends Context {
  constructor(filename: string, code: string, inMap?: SourceMap | string) {
    super()
    this.filename = filename
    this.originalCode = code
    if (inMap) {
      this.sourcemapChain.push(inMap)
    }
  }

  _getCombinedSourcemap(createIfNull = false) {
    return this.combinedMap
  }

  getCombinedSourcemap() {
    return this._getCombinedSourcemap(true) as SourceMap
  }
}
```
可以看到，TransformContext继承自之前所说的Context对象，也就是说 transform 钩子的上下文对象相比其它钩子只是做了一些扩展，增加了 sourcemap 合并的功能，将不同插件的 transform 钩子执行后返回的 sourcemap 进行合并，以保证 sourcemap 的准确性和完整性

#### 插件工作流概览
让我们把目光集中在resolvePlugins的实现上，Vite 所有的插件就是在这里被收集起来的
```js
export async function resolvePlugins(
  config: ResolvedConfig,
  prePlugins: Plugin[],
  normalPlugins: Plugin[],
  postPlugins: Plugin[]
): Promise<Plugin[]> {
  const isBuild = config.command === 'build'
  // 收集生产环境构建的插件，后文会介绍
  const buildPlugins = isBuild
    ? (await import('../build')).resolveBuildPlugins(config)
    : { pre: [], post: [] }

  return [
    // 1. 别名插件
    isBuild ? null : preAliasPlugin(),
    aliasPlugin({ entries: config.resolve.alias }),
    // 2. 用户自定义 pre 插件(带有`enforce: "pre"`属性)
    ...prePlugins,
    // 3. Vite 核心构建插件
    // 数量比较多，暂时省略代码
    // 4. 用户插件（不带有 `enforce` 属性）
    ...normalPlugins,
    // 5. Vite 生产环境插件 & 用户插件(带有 `enforce: "post"`属性)
    definePlugin(config),
    cssPostPlugin(config),
    ...buildPlugins.pre,
    ...postPlugins,
    ...buildPlugins.post,
    // 6. 一些开发阶段特有的插件
    ...(isBuild
      ? []
      : [clientInjectionsPlugin(config), importAnalysisPlugin(config)])
  ].filter(Boolean) as Plugin[]
}
```
从上述代码中我们可以总结出 Vite 插件的具体执行顺序

1. 别名插件包括 vite:pre-alias和@rollup/plugin-alias，用于路径别名替换。

2. 用户自定义 pre 插件，也就是带有enforce: "pre"属性的自定义插件。

3. Vite 核心构建插件，这部分插件为 Vite 的核心编译插件，数量比较多，我们在下部分一一拆解。

4. 用户自定义的普通插件，即不带有 enforce 属性的自定义插件。

5. Vite 生产环境插件和用户插件中带有enforce: "post"属性的插件。

6. 一些开发阶段特有的插件，包括环境变量注入插件**clientInjectionsPlugin**和**import**语句分析及重写插件**importAnalysisPlugin**

#### 别名插件
1. **vite:pre-alias**,将 bare import 路径重定向到预构建依赖的路径
```js
// 假设 React 已经过 Vite 预构建
import React from 'react';
// 会被重定向到预构建产物的路径
import React from '/node_modules/.vite/react.js'
```
2. **@rollup/plugin-alias**:通用的路径别名(即resolve.alias配置)的功能
  
#### 核心构建插件
1. **module preload 特性的 Polyfill**
当你在 Vite 配置文件中开启下面这个配置时:
```js
{
  build: {
    polyfillModulePreload: true
  }
}
```
Vite 会自动应用**modulePreloadPolyfillPlugin**插件，在产物中注入**module preload**的Polyfill 代码
1. 扫描出当前所有的 modulepreload 标签，拿到 link 标签对应的地址，通过执行 fetch 实现预加载；
2. 同时通过 MutationObserver 监听 DOM 的变化，一旦发现包含 modulepreload 属性的 link 标签，则同样通过 fetch 请求实现预加载。

2. **路径解析插件**
路径解析插件(即**vite:resolve**)是 Vite 中比较核心的插件，几乎所有重要的 Vite 特性都离不开这个插件的实现，诸如依赖预构建、HMR、SSR 等等。同时它也是实现相当复杂的插件，一方面实现了 Node.js 官方的 resolve 算法，另一方面需要支持前面所说的各项特性，可以说是专门给 Vite 实现了一套路径解析算法。

3. **内联脚本加载插件**
对于 HTML中的内联脚本，Vite会通过**vite:html-inline-script-proxy**插件来进行加载。比如下面这个 script 标签:
```html
<script type="module">
import React from 'react';
console.log(React)
</script>
```
这些内容会在后续的build-html插件从 HTML 代码中剔除，并且变成下面的这一行代码插入到项目入口模块的代码中:
```js
import '/User/xxx/vite-app/index.html?http-proxy&index=0.js'
```
而 **vite:html-inline-script-proxy**就是用来加载这样的模块，实现如下:
```js
const htmlProxyRE = /\?html-proxy&index=(\d+)\.js$/

export function htmlInlineScriptProxyPlugin(config: ResolvedConfig): Plugin {
  return {
    name: 'vite:html-inline-script-proxy',
    load(id) {
      const proxyMatch = id.match(htmlProxyRE)
      if (proxyMatch) {
        const index = Number(proxyMatch[1])
        const file = cleanUrl(id)
        const url = file.replace(normalizePath(config.root), '')
        // 内联脚本的内容会被记录在 htmlProxyMap 这个表中
        const result = htmlProxyMap.get(config)!.get(url)![index]
        if (typeof result === 'string') {
          // 加载脚本的具体内容
          return result
        } else {
          throw new Error(`No matching HTML proxy module found from ${id}`)
        }
      }
    }
  }
}
```
4. CSS编译插件
即名为**vite:css**的插件，主要实现下面这些功能:
   1. CSS 预处理器的编译
   2. CSS Modules
   3. Postcss 编译
   4. 通过 @import 记录依赖，便于 HMR
5. Esbuild 转译插件
即名为**vite:esbuild**的插件，用来进行 `.js、.ts、.jsx`和`tsx`，代替了传统的 Babel 或者 TSC 的功能，这也是 Vite 开发阶段性能强悍的一个原因。插件中主要的逻辑是`transformWithEsbuild`函数，顾名思义，你可以通过这个函数进行代码转译。当然，Vite 本身也导出了这个函数，作为一种通用的 transform 能力，你可以这样来使用:
```js
import { transformWithEsbuild } from 'vite';

// 传入两个参数: code, filename
transformWithEsbuild('<h1>hello</h1>', './index.tsx').then(res => {
  // {
  //   warnings: [],
  //   code: '/* @__PURE__ */ React.createElement("h1", null, "hello");\n',
  //   map: {/* sourcemap 信息 */}
  // }
  console.log(res);
})
```
6. 静态资源加载插件
`vite:json` 用来加载 JSON 文件，通过`@rollup/pluginutils的dataToEsm`方法可实现 JSON 的按名导入，具体实现见链接；

`vite:wasm` 用来加载 `.wasm` 格式的文件，具体实现见链接；

`vite:worker` 用来 `Web Worker` 脚本，插件内部会使用` Rollup `对 Worker 脚本进行打包，具体实现见链接；

`vite:asset`，开发阶段实现了其他格式静态资源的加载，而生产环境会通过 renderChunk 钩子将静态资源地址重写为产物的文件地址，如`./img.png` 重写为 `https://cdn.xxx.com/assets/img.91ee297e.png`

#### 生产环境特有插件
1. **提供全局变量替换功能**，如下面的这个配置:
```js
// vite.config.ts
const version = '2.0.0';

export default {
  define: {
    __APP_VERSION__: `JSON.stringify(${version})`
  }
}

```
全局变量替换的功能和我们之前在Rollup插件小节中提到的**rollup/plugin-replace**差不多，当然在实现上 Vite 会有所区别:
1. 开发环境下:Vite会通过将所有的全局变量挂载到`window对象`,而不用经过define插件的处理,节省编译开销
2. 生产环境下,Vite会使用**define**插件,进行字符串替换以及sourcemap生成

2. **CSS后处理插件**
CSS 后处理插件即name为**vite:css-post**的插件，它的功能包括**开发阶段CSS响应结果处理**和**生产环境 CSS 文件生成**

* 在开发阶段，这个插件会将之前的 CSS 编译插件处理后的结果，包装成一个 ESM 模块，返回给浏览器
* 生产环境中，Vite 默认会通过这个插件进行 CSS 的 code splitting，即对于每个异步 chunk，Vite 会将其依赖的 CSS 代码单独打包成一个文件
```js
const fileHandle = this.emitFile({
  name: chunk.name + '.css',
  type: 'asset',
  source: chunkCSS
});
```

3. HTML构建插件
HTML构建插件即`build-html`插件。之前我们在`内联脚本加载插件`中提到过，项目根目录下的html会转换为一段 JavaScript 代码，如下面的这个例子
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
</head>
<body>
  // 普通方式引入
  <script src="./index.ts"></script>
  // 内联脚本
  <script type="module">
    import React from 'react';
    console.log(React)
  </script>
</body>
</html>
```
首先，当 Vite 在生产环境`transform`这段入口 HTML 时，会做 3 件事情:
1. 对HTML执行各个插件中带有`enforce:"pre"`属性的transformIndexHtml钩子
>>>我们知道插件本身可以带有 enforce: "pre"|"post" 属性，而 transformIndexHtml 本身也可以带有这个属性，用于在不同的阶段进行 HTML 转换。后文会介绍 transformIndexHtml 钩子带有 enforce: "post" 时的执行时机
2. 将其中的script标签内容删除,并将其转换为`import语句`如`import './index.ts'`
3. 在 transform 钩子中返回记录下来的 import 内容，将 import 语句作为模块内容进行加载。也就是说，虽然 Vite 处理的是一个 HTML 文件，但最后进行打包的内容却是一段 JS 的内容
```js
export function buildHtmlPlugin() {
  name: 'vite:build',
  transform(html, id) {
    if (id.endsWith('.html')) {
      let js = '';
      // 省略 HTML AST 遍历过程(通过 @vue/compiler-dom 实现)
      // 收集 script 标签，转换成 import 语句，拼接到 js 字符串中
      return js;
    }
  }
}
```
其次，在生成产物的最后一步即generateBundle钩子中，拿到入口 Chunk，分析入口 Chunk 的内容, 分情况进行处理。
- 如果只有`import`语句，先通过`Rollup`提供的`chunk`和`bundle`对象获取入口`chunk`所有的依赖`chunk`，并将这些`chunk`进行`后序排列`，如 `a 依赖 b，b 依赖 c`，最后的依赖数组就是`[c, b, a]`然后依次将 `c，b, a` 生成三个 script 标签，插入 HTML 中.最后，`Vite` 会将入口` chunk `的内容从 `bundle`产物中移除，因此它的内容只要`import`语句，而它`import`的`chunk`已经作为`script`标签插入到了`HTML`中，那入口`Chunk`的存在也就没有意义了.

如果除了 import 语句，还有其它内容， Vite 就会将`入口Chunk`单独生成一个`script`标签，分析出依赖的后序排列(和上一种情况分析手段一样)，然后通过注入 <link rel="modulepreload"> 标签对入口文件的依赖 chunk 进行预加载。

最后，插件会调用用户插件中带有 `enforce: "post"` 属性的 `transformIndexHtml` 钩子，对 HTML 进行进一步的处理.

3. **CommonJs**转换插件
  1. Vite 使用 `Esbuild` 将 `Commonjs` 转换为 ESM，而生产环境中，Vite 会直接使用 Rollup 的官方插件 `@rollup/plugin-commonjs`
4. **date-uri**插件:date-uri 插件用来支持 import 模块中含有 Base64 编码的情况，如:
5. **dynamic-import-vars** 插件:用于支持在动态 import 中使用变量的功能，如下示例代码:
```js
function importLocale(locale) {
  return import(`./locales/${locale}.js`);
}
```
6. **import-meta-url** 支持插件
用来转换如下格式的资源 URL:
```js
new URL('./foo.png', import.meta.url)

//将其转换为生产环境的 URL 格式，如:
// 使用 self.location 来保证低版本浏览器和 Web Worker 环境的兼容性
new URL('./assets.a4b3d56d.png, self.location)

//同时，对于动态 import 的情况也能进行支持，如下面的这种写法:
function getImageUrl(name) {
  return new URL(`./dir/${name}.png`, import.meta.url).href
}
//Vite 识别到./dir/${name}.png这样的模板字符串，会将整行代码转换成下面这样:
function getImageUrl(name) {
    return import.meta.globEager('./dir/**.png')[`./dir/${name}.png`].default;
}
```
7. 生产环境 import 分析插件
**vite:build-import-analysis** 插件会在**生产环境**打包时用作**import**语句分析和重写，主要目的是对动态**import**的模块进行**预加载处理**

对含有动态`import`的chunk而言，会在插件的`tranform`钩子中被添加这样一段工具代码用来进行模块预加载，逻辑并不复杂，你可以参考源码实现。关键代码简化后如下:

```js
function preload(importModule, deps) {
  return Promise.all(
    deps.map(dep => {
      // 如果异步模块的依赖还没有加载
      if (!alreadyLoaded(dep)) { 
        // 创建 link 标签加载，包括 JS 或者 CSS
        document.head.appendChild(createLink(dep))  
        // 如果是 CSS，进行特殊处理，后文会介绍
        if (isCss(dep)) {
          return new Promise((resolve, reject) => {
            link.addEventListener('load', resolve)
            link.addEventListener('error', reject)
          })
        }
      }
    })
  ).then(() => importModule())
}

```
我们知道，Vite 内置了 CSS 代码分割的能力，当一个模块通过动态 import 引入的时候，这个模块会被单独打包成一个 chunk，与此同时这个模块中的样式代码也会打包成单独的 CSS 文件。如果异步模块的 CSS 和 JS 同时进行预加载，那么在某些浏览器下(如 IE)就会出现`FOUC问题`，页面样式会闪烁，影响用户体验。但 Vite 通过监听`link 标签` `load 事件`的方式来保证 CSS 在 JS 之前加载完成，从而解决了 `FOUC`问题。你可以注意下面这段关键代码:
```js
if (isCss) {
  return new Promise((res, rej) => {
    link.addEventListener('load', res)
    link.addEventListener('error', rej)
  })
}
//从源码的transform钩子实现中，不难发现 Vite 会将动态 import 的代码进行转换，如下代码所示:
// 转换前
import('a')
// 转换后
__vitePreload(() => 'a', __VITE_IS_MODERN__ ?"__VITE_PRELOAD__":void)
```
其中，`__vitePreload` 会被加载为前文中的`preload`工具函数，__VITE_IS_MODERN__ 会在 `renderChunk` 中被替换成 true 或者 false，表示是否为 Modern 模式打包，而对于"__VITE_PRELOAD__"，Vite 会在`generateBundle`阶段，分析出 a 模块所有依赖文件(包括 CSS)，将依赖文件名的数组作为 preload 工具函数的第二个参数。

```js
//同时，对于 Vite 独有的 import.meta.glob 语法，也会在这个插件中进行编译，如:
const modules = import.meta.glob('./dir/*.js')
//会通过插件转换成下面这段代码:
const modules = {
  './dir/foo.js': () => import('./dir/foo.js'),
  './dir/bar.js': () => import('./dir/bar.js')
}
```

具体的实现在 transformImportGlob 函数中，除了被该插件使用外，这个函数被还依赖预构建、开发环境 import 分析等核心流程使用，属于一类比较底层的逻辑，感兴趣的同学可以精读一下这部分的实现源码

8. JS压缩插件:Vite 中提供了两种 JS 代码压缩的工具，即 Esbuild 和 Terser，分别由两个插件插件实现:
  1. **vite:esbuild-transpile**.在 renderChunk 阶段，调用 Esbuild 的 transform API，并指定 minify 参数，从而实现 JS 的压缩。
  2. **vite:terser(点击查看实现)** 同样也在 renderChunk 阶段，Vite 会单独的 Worker 进程中调用 Terser 进行 JS 代码压缩
9. 构建报告插件
  1. **vite:manifest(点击查看实现)**。提供打包后的各种资源文件及其关联信息，如下内容所示:
  ```js
  // manifest.json
  {
    "index.html": {
      "file": "assets/index.8edffa56.js",
      "src": "index.html",
      "isEntry": true,
      "imports": [
        // JS 引用
        "_vendor.71e8fac3.js"
      ],
      "css": [
        // 样式文件应用
        "assets/index.458f9883.css"
      ],
      "assets": [
        // 静态资源引用
        "assets/img.9f0de7da.png"
      ]
    },
    "_vendor.71e8fac3.js": {
      "file": "assets/vendor.71e8fac3.js"
    }
  }
  ```
  2. **vite:ssr-manifest(点击查看实现)**。提供每个模块与 chunk 之间的映射关系，方便 SSR 时期通过渲染的组件来确定哪些 chunk 会被使用，从而按需进行预加载。最后插件输出的内容如下:
  ```js
    // ssr-manifest.json
    {
      "node_modules/object-assign/index.js": [
        "/assets/vendor.71e8fac3.js"
      ],
      "node_modules/object-assign/index.js?commonjs-proxy": [
        "/assets/vendor.71e8fac3.js"
      ],
      // 省略其它模块信息
    }
  ```
  3. **vite:reporter(点击查看实现)**主要提供打包时的命令行构建日志

#### 开发环境特有插件
1. **客户端环境变量注入插件**
在开发环境中，Vite 会自动往 HTML 中注入一段 client 的脚本
```js
<script type="module" src="/@vite/client"></script>
```
这段脚本主要提供`注入环境变量`、`处理 HMR 更新逻辑`、`构建出现错误时提供报错界面`等功能，而我们这里要介绍的`vite:client-inject`就是来完成时环境变量的注入，将 client 脚本中的`__MODE__`、`__BASE__`、`__DEFINE__`等等字符串替换为运行时的变量，实现环境变量以及 HMR 相关上下文信息的注入.

2. **开发阶段 import 分析插件**
最后,`Vite` 会在`开发阶段`加入`import`分析插件，即`vite:import-analysis`.与之前所介绍的`vite:build-import-analysis`相对应，主要处理 `import` 语句相关的解析和重写，但`vite:import-analysis` 插件的关注点会不太一样，主要围绕 `Vite 开发阶段的各项特性`来实现，我们可以来梳理一下这个插件需要做哪些事情:
  1. 对 bare import，将路径名转换为真实的文件路径，如:
  ```js
    // 转换前
    import 'foo'
    // 转换后
    // tip: 如果是预构建的依赖，则会转换为预构建产物的路径
    import '/@fs/project/node_modules/foo/dist/foo.js'
  ```
  主要调用`PluginContainer`的上下文对象方法即`this.resolve`实现，这个方法会调用所有插件的`resolveId`方法，包括之前介绍的`vite:pre-alias`和`vite:resolve`，完成路径解析的核心逻辑


### HRM的实现揭秘

#### **创建模块依赖图**
为了方便管理各个模块之间的依赖关系，Vite 在`Dev Server`中创建了模块依赖图的数据结构，即**ModuleGraph**类
**创建依赖图**主要分为三个步骤:
1. 初始化依赖图实例
2. 创建依赖图节点
3. 绑定各个模块节点的依赖关系
首先，Vite 在 Dev Server 启动时会初始化 ModuleGraph 的实例:
```js
// pacakges/vite/src/node/server/index.ts
const moduleGraph: ModuleGraph = new ModuleGraph((url) =>
  container.resolveId(url)
);
```
接下来我们具体查看`ModuleGraph`这个类的实现。其中定义了若干个 Map，用来记录模块信息:
```js
// 由原始请求 url 到模块节点的映射，如 /src/index.tsx
urlToModuleMap = new Map<string, ModuleNode>()
// 由模块 id 到模块节点的映射，其中 id 与原始请求 url，为经过 resolveId 钩子解析后的结果
idToModuleMap = new Map<string, ModuleNode>()
// 由文件到模块节点的映射，由于单文件可能包含多个模块，如 .vue 文件，因此 Map 的 value 值为一个集合
fileToModulesMap = new Map<string, Set<ModuleNode>>()
```
ModuleNode 对象即代表模块节点的具体信息，我们可以来看看它的数据结构:
```js
class ModuleNode {
  // 原始请求 url
  url: string
  // 文件绝对路径 + query
  id: string | null = null
  // 文件绝对路径
  file: string | null = null
  type: 'js' | 'css'
  info?: ModuleInfo
  // resolveId 钩子返回结果中的元数据
  meta?: Record<string, any>
  // 该模块的引用方
  importers = new Set<ModuleNode>()
  // 该模块所依赖的模块
  importedModules = new Set<ModuleNode>()
  // 接受更新的模块
  acceptedHmrDeps = new Set<ModuleNode>()
  // 是否为`接受自身模块`的更新
  isSelfAccepting = false
  // 经过 transform 钩子后的编译结果
  transformResult: TransformResult | null = null
  // SSR 过程中经过 transform 钩子后的编译结果
  ssrTransformResult: TransformResult | null = null
  // SSR 过程中的模块信息
  ssrModule: Record<string, any> | null = null
  // 上一次热更新的时间戳
  lastHMRTimestamp = 0

  constructor(url: string) {
    this.url = url
    this.type = isDirectCSSRequest(url) ? 'css' : 'js'
  }
}
```
Vite 是在什么时候创建 ModuleNode 节点的呢？我们可以到 Vite Dev Server 中的`transform`中间件一探究竟:
```js
// packages/vite/src/node/server/middlewares/transform.ts
// 核心转换逻辑
const result = await transformRequest(url, server, {
  html: req.headers.accept?.includes('text/html')
})

//可以看到，transform中间件的主要逻辑是调用 transformRequest方法，我们来进一步查看这个方法的核心代码实现:
// packages/vite/src/node/server/transformRequest.ts

// 从 ModuleGraph 查找模块节点信息
const module = await server.moduleGraph.getModuleByUrl(url)
// 如果有则命中缓存
const cached =
  module && (ssr ? module.ssrTransformResult : module.transformResult)
if (cached) {
  return cached
}
// 否则调用 PluginContainer 的 resolveId 和 load 方法对进行模块加载
const id = (await pluginContainer.resolveId(url))?.id || url
const loadResult = await pluginContainer.load(id, { ssr })
// 然后通过调用 ensureEntryFromUrl 方法创建 ModuleNode
const mod = await moduleGraph.ensureEntryFromUrl(url)

//接着我们看看 ensureEntryFromUrl 方法如何创建新的 ModuleNode 节点:

async ensureEntryFromUrl(rawUrl: string): Promise<ModuleNode> {
  // 实质是调用各个插件的 resolveId 钩子得到路径信息
  const [url, resolvedId, meta] = await this.resolveUrl(rawUrl)
  let mod = this.urlToModuleMap.get(url)
  if (!mod) {
    // 如果没有缓存，就创建新的 ModuleNode 对象
    // 并记录到 urlToModuleMap、idToModuleMap、fileToModulesMap 这三张表中
    mod = new ModuleNode(url)
    if (meta) mod.meta = meta
    this.urlToModuleMap.set(url, mod)
    mod.id = resolvedId
    this.idToModuleMap.set(resolvedId, mod)
    const file = (mod.file = cleanUrl(resolvedId))
    let fileMappedModules = this.fileToModulesMap.get(file)
    if (!fileMappedModules) {
      fileMappedModules = new Set()
      this.fileToModulesMap.set(file, fileMappedModules)
    }
    fileMappedModules.add(mod)
  }
  return mod
}
```
现在你应该明白了模块依赖图中各个 ModuleNode 节点是如何创建出来的，那么，各个节点的依赖关系是在什么时候绑定的呢？
我们不妨把目光集中到vite:import-analysis插件当中，在这个插件的 transform 钩子中，会对模块代码中的 import 语句进行分析，得到如下的一些信息:
1. importedUrls: 当前模块的依赖模块 url 集合。
2. acceptedUrls: 当前模块中通过 import.meta.hot.accept 声明的依赖模块 url 集合。
3. isSelfAccepting: 分析 import.meta.hot.accept 的用法，标记是否为`接受自身更新`的类型。
接下来会进入核心的模块依赖关系绑定的环节，核心代码如下:
```js
// 引用方模块
const importerModule = moduleGraph.getModuleById(importer)
await moduleGraph.updateModuleInfo(
  importerModule,
  importedUrls,
  normalizedAcceptedUrls,
  isSelfAccepting
)

//可以看到，绑定依赖关系的逻辑主要由ModuleGraph对象的updateModuleInfo方法实现，核心代码如下:
async updateModuleInfo(
  mod: ModuleNode,
  importedModules: Set<string | ModuleNode>,
  acceptedModules: Set<string | ModuleNode>,
  isSelfAccepting: boolean
) {
  mod.isSelfAccepting = isSelfAccepting
  mod.importedModules = new Set()
  // 绑定节点依赖关系
  for (const imported of importedModules) {
    const dep =
      typeof imported === 'string'
        ? await this.ensureEntryFromUrl(imported)
        : imported
    dep.importers.add(mod)
    mod.importedModules.add(dep)
  }

  // 更新 acceptHmrDeps 信息
  const deps = (mod.acceptedHmrDeps = new Set())
  for (const accepted of acceptedModules) {
    const dep =
      typeof accepted === 'string'
        ? await this.ensureEntryFromUrl(accepted)
        : accepted
    deps.add(dep)
  }
}

```
至此，模块间的依赖关系就成功进行绑定了。随着越来越多的模块经过 `vite:import-analysis的 transform`钩子处理，所有模块之间的依赖关系会被记录下来，整个依赖图的信息也就被补充完整了。



#### **服务端收集更新模块**
Vite 服务端如何根据这个图结构收集更新模块?
首先， Vite 在服务启动时会通过**chokidar**新建文件监听器:
```js
// packages/vite/src/node/server/index.ts
import chokidar from 'chokidar'

// 监听根目录下的文件
const watcher = chokidar.watch(path.resolve(root));
// 修改文件
watcher.on('change', async (file) => {
  file = normalizePath(file)
  moduleGraph.onFileChange(file)
  await handleHMRUpdate(file, server)
})
// 新增文件
watcher.on('add', (file) => {
  handleFileAddUnlink(normalizePath(file), server)
})
// 删除文件
watcher.on('unlink', (file) => {
  handleFileAddUnlink(normalizePath(file), server, true)
})
//我们分别以修改文件、新增文件和删除文件这几个方面来介绍 HMR 在服务端的逻辑。

```
#### 修改文件
当业务代码中某个文件被修改时，Vite 首先会调用`moduleGraph`的`onFileChange`对模块图中的对应节点进行清除缓存的操作
```js
class ModuleGraph {
  onFileChange(file: string): void {
    const mods = this.getModulesByFile(file)
    if (mods) {
      const seen = new Set<ModuleNode>()
      // 将模块的缓存信息去除
      mods.forEach((mod) => {
        this.invalidateModule(mod, seen)
      })
    }
  }

  invalidateModule(mod: ModuleNode, seen: Set<ModuleNode> = new Set()): void {
    mod.info = undefined
    mod.transformResult = null
    mod.ssrTransformResult = null
  }
}

//然后正式进入 HMR 收集更新的阶段，主要逻辑在handleHMRUpdate函数中，代码简化后如下:

// packages/vite/src/node/server/hmr.ts
export async function handleHMRUpdate(
  file: string,
  server: ViteDevServer
): Promise<any> {
  const { ws, config, moduleGraph } = server
  const shortFile = getShortName(file, config.root)

  // 1. 配置文件/环境变量声明文件变化，直接重启服务
  // 代码省略

  // 2. 客户端注入的文件(vite/dist/client/client.mjs)更改
  // 给客户端发送 full-reload 信号，使之刷新页面
  if (file.startsWith(normalizedClientDir)) {
    ws.send({
      type: 'full-reload',
      path: '*'
    })
    return
  }
  // 3. 普通文件变动
  // 获取需要更新的模块
  const mods = moduleGraph.getModulesByFile(file)
  const timestamp = Date.now()
  // 初始化 HMR 上下文对象
  const hmrContext: HmrContext = {
    file,
    timestamp,
    modules: mods ? [...mods] : [],
    read: () => readModifiedFile(file),
    server
  }
  // 依次执行插件的 handleHotUpdate 钩子，拿到插件处理后的 HMR 模块
  for (const plugin of config.plugins) {
    if (plugin.handleHotUpdate) {
      const filteredModules = await plugin.handleHotUpdate(hmrContext)
      if (filteredModules) {
        hmrContext.modules = filteredModules
      }
    }
  }
  // updateModules——核心处理逻辑
  updateModules(shortFile, hmrContext.modules, timestamp, server)
}
```
从中可以看到，Vite 对于不同类型的文件，热更新的策略有所不同：

1. 对于配置文件和环境变量声明文件的改动，Vite 会直接重启服务器。
2. 对于客户端注入的文件(vite/dist/client/client.mjs)的改动，Vite 会给客户端发送full-reload信号，让客户端刷新页面。
3. 对于普通文件改动，Vite 首先会获取需要热更新的模块，然后对这些模块依次查找热更新边界，然后将模块更新的信息传给客户端。

其中，对于普通文件的热更新边界查找的逻辑，主要集中在updateModules函数中，让我们来看看具体的实现

```js
function updateModules(
  file: string,
  modules: ModuleNode[],
  timestamp: number,
  { config, ws }: ViteDevServer
) {
  const updates: Update[] = []
  const invalidatedModules = new Set<ModuleNode>()
  let needFullReload = false
  // 遍历需要热更新的模块
  for (const mod of modules) {
    invalidate(mod, timestamp, invalidatedModules)
    if (needFullReload) {
      continue
    }
    // 初始化热更新边界集合
    const boundaries = new Set<{
      boundary: ModuleNode
      acceptedVia: ModuleNode
    }>()
    // 调用 propagateUpdate 函数，收集热更新边界
    const hasDeadEnd = propagateUpdate(mod, boundaries)
    // 返回值为 true 表示需要刷新页面，否则局部热更新即可
    if (hasDeadEnd) {
      needFullReload = true
      continue
    }
    // 记录热更新边界信息
    updates.push(
      ...[...boundaries].map(({ boundary, acceptedVia }) => ({
        type: `${boundary.type}-update` as Update['type'],
        timestamp,
        path: boundary.url,
        acceptedPath: acceptedVia.url
      }))
    )
  }
  // 如果被打上 full-reload 标识，则让客户端强制刷新页面
  if (needFullReload) {
    ws.send({
      type: 'full-reload'
    })
  } else {
    config.logger.info(
      updates
        .map(({ path }) => chalk.green(`hmr update `) + chalk.dim(path))
        .join('\n'),
      { clear: true, timestamp: true }
    )
    ws.send({
      type: 'update',
      updates
    })
  }
}

// 热更新边界收集
function propagateUpdate(
  node: ModuleNode,
  boundaries: Set<{
    boundary: ModuleNode
    acceptedVia: ModuleNode
  }>,
  currentChain: ModuleNode[] = [node]
): boolean {
   // 接受自身模块更新
   if (node.isSelfAccepting) {
    boundaries.add({
      boundary: node,
      acceptedVia: node
    })
    return false
  }
  // 入口模块
  if (!node.importers.size) {
    return true
  }
  // 遍历引用方
  for (const importer of node.importers) {
    const subChain = currentChain.concat(importer)
    // 如果某个引用方模块接受了当前模块的更新
    // 那么将这个引用方模块作为热更新的边界
    if (importer.acceptedHmrDeps.has(node)) {
      boundaries.add({
        boundary: importer,
        acceptedVia: node
      })
      continue
    }

    if (currentChain.includes(importer)) {
      // 出现循环依赖，需要强制刷新页面
      return true
    }
    // 递归向更上层的引用方寻找热更新边界
    if (propagateUpdate(importer, boundaries, subChain)) {
      return true
    }
  }
  return false
}
```
当热更新边界的信息收集完成后，服务端会将这些信息推送给客户端，从而完成局部的模块更新。
#### 新增文件和删除文件
```js
//对于新增和删除文件，Vite 也通过chokidar监听了相应的事件:
watcher.on('add', (file) => {
  handleFileAddUnlink(normalizePath(file), server)
})

watcher.on('unlink', (file) => {
  handleFileAddUnlink(normalizePath(file), server, true)
})

//接下来，我们就来浏览一下handleFileAddUnlink的逻辑，代码简化后如下:
export async function handleFileAddUnlink(
  file: string,
  server: ViteDevServer,
  isUnlink = false
): Promise<void> {
  const modules = [...(server.moduleGraph.getModulesByFile(file) ?? [])]

  if (modules.length > 0) {
    updateModules(
      getShortName(file, server.config.root),
      modules,
      Date.now(),
      server
    )
  }
}
//这个函数同样是调用updateModules完成模块热更新边界的查找和更新信息的推送
```
#### 客户端派发更新
1. 服务端会监听文件的改动，然后计算出对应的热更新信息，通过 WebSocket 将更新信息传递给客户端，具体来说，会给客户端发送如下的数据:
```js
{
  type: "update",
  update: [
    {
      // 更新类型，也可能是 `css-update`
      type: "js-update",
      // 更新时间戳
      timestamp: 1650702020986,
      // 热更模块路径
      path: "/src/main.ts",
      // 接受的子模块路径
      acceptedPath: "/src/render.ts"
    }
  ]
}
// 或者 full-reload 信号
{
  type: "full-reload"
}

//那么客户端是如何接受这些信息并进行模块更新的呢？
//Vite 在开发阶段会默认在 HTML 中注入一段客户端的脚本
<script type="module" src="/@vite/client"></script>

//从中你可以发现，客户端的脚本中创建了 WebSocket 客户端，并与 Vite Dev Server 中的 WebSocket 服务端(点击查看实现)建立双向连接

const socketProtocol = null || (location.protocol === 'https:' ? 'wss' : 'ws');
const socketHost = `${null || location.hostname}:${"3000"}`;
const socket = new WebSocket(`${socketProtocol}://${socketHost}`, 'vite-hmr');

//随后会监听 socket 实例的message事件，接收到服务端传来的更新信息

socket.addEventListener('message', async ({ data }) => {
  handleMessage(JSON.parse(data));
});

//接下来让我们把目光集中在 handleMessage 函数中
async function handleMessage(payload: HMRPayload) {
  switch (payload.type) {
    case 'connected':
      console.log(`[vite] connected.`)
      // 心跳检测
      setInterval(() => socket.send('ping'), __HMR_TIMEOUT__)
      break
    case 'update':
      payload.updates.forEach((update) => {
        if (update.type === 'js-update') {
          queueUpdate(fetchUpdate(update))
        } else {
          // css-update
          // 省略实现
          console.log(`[vite] css hot updated: ${path}`)
        }
      })
      break
    case 'full-reload':
      // 刷新页面
      location.reload()
    // 省略其它消息类型
  }
}
//其中，我们重点关注 js 的更新逻辑，即下面这行代码:
queueUpdate(fetchUpdate(update))

//我们先来看看queueUpdate和fetchUpdate这两个函数的实现:

let pending = false
let queued: Promise<(() => void) | undefined>[] = []

// 批量任务处理，不与具体的热更新行为挂钩，主要起任务调度作用
async function queueUpdate(p: Promise<(() => void) | undefined>) {
  queued.push(p)
  if (!pending) {
    pending = true
    await Promise.resolve()
    pending = false
    const loading = [...queued]
    queued = []
    ;(await Promise.all(loading)).forEach((fn) => fn && fn())
  }
}

// 派发热更新的主要逻辑
async function fetchUpdate({ path, acceptedPath, timestamp }: Update) {
  // 后文会介绍 hotModuleMap 的作用，你暂且不用纠结实现，可以理解为 HMR 边界模块相关的信息
  const mod = hotModulesMap.get(path)
  const moduleMap = new Map()
  const isSelfUpdate = path === acceptedPath

  // 1. 整理需要更新的模块集合
  const modulesToUpdate = new Set<string>()
  if (isSelfUpdate) {
    // 接受自身更新
    modulesToUpdate.add(path)
  } else {
    // 接受子模块更新
    for (const { deps } of mod.callbacks) {
      deps.forEach((dep) => {
        if (acceptedPath === dep) {
          modulesToUpdate.add(dep)
        }
      })
    }
  }
  // 2. 整理需要执行的更新回调函数
  // 注： mod.callbacks 为 import.meta.hot.accept 中绑定的更新回调函数，后文会介绍
  const qualifiedCallbacks = mod.callbacks.filter(({ deps }) => {
    return deps.some((dep) => modulesToUpdate.has(dep))
  })
  // 3. 对将要更新的模块进行失活操作，并通过动态 import 拉取最新的模块信息
  await Promise.all(
    Array.from(modulesToUpdate).map(async (dep) => {
      const disposer = disposeMap.get(dep)
      if (disposer) await disposer(dataMap.get(dep))
      const [path, query] = dep.split(`?`)
      try {
        const newMod = await import(
          /* @vite-ignore */
          base +
            path.slice(1) +
            `?import&t=${timestamp}${query ? `&${query}` : ''}`
        )
        moduleMap.set(dep, newMod)
      } catch (e) {
        warnFailedFetch(e, dep)
      }
    })
  )
  // 4. 返回一个函数，用来执行所有的更新回调
  return () => {
    for (const { deps, fn } of qualifiedCallbacks) {
      fn(deps.map((dep) => moduleMap.get(dep)))
    }
    const loggedPath = isSelfUpdate ? path : `${acceptedPath} via ${path}`
    console.log(`[vite] hot updated: ${loggedPath}`)
  }
}
```
对热更新的边界模块来讲，我们需要在客户端获取这些信息:
1. 边界模块所接受(accept)的模块
2. accept 的模块触发更新后的回调
####
####
####
