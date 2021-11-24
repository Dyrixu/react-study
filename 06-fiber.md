# fiber

参考问题：
* 什么是fiber ? Fiber 架构解决了什么问题？ 
* Fiber root 和 root fiber 有什么区别？ 
* 不同fiber 之间如何建立起关联的？
* React 调和流程？
* 两大阶段 commit 和 render 都做了哪些事情？
* 什么是双缓冲树？ 有什么作用？
* Fiber 深度遍历流程？
* Fiber的调和能中断吗？ 如何中断？

### 什么是 fiber

fiber 诞生在 React v16 版本，目的是解决 React 应用卡顿
fiber 在 React 中是最小粒度的执行单元，无论 react 还是 vue，在遍历更新每一个节点的时候都不是使用真实的 dom，都是采用虚拟 dom，所以 fiber 可以理解为 React 的虚拟dom

### 为什么要用 fiber

React v15 之前的版本，React 对于虚拟 DOM 是采用递归方式遍历更新的
一个更新会从应用根部递归更新，递归一旦开始，中途无法中断，随着项目层级越来越深，导致更新的时间越来越长，就会卡顿

React v16 为了解决卡顿问题，引入了 fiber，更新 fiber 的过程叫做 Reconciler（调和器），每一个 fiber 都可以作为一个执行单元来处理
所以每个 fiber 可以根据自身的过期时间 ExpirationTime 来判断是否还有空间时间来执行更新
如果没有时间更新，就要把主动权交给浏览器去渲染，做一些动画的重绘（repaints）、重排（reflow）之类的事，这样就不会觉得很卡
然后等浏览器空余时间，再通过 scheduler（调度器），再次恢复执行单元上来，这样就能本质上中断了渲染，提高用户体验

### React.element fiber 真实dom 三者关系

* element 是React 视图层再代码层级上的表象，jsx元素结构都会被创建成 element 对象，上面保存了 props children等信息
* DOM 是元素在浏览器上给用户直观的表象
* fiber 可以说是element 和真实DOM 之间的交流枢纽站，一方面每个类型 element 都会有一个对应的 fiber 类型，element 变化引起的更新流程都是通过 fiber 层面做一次调和改变，然后对于元素，形成新的 DOM 做视图渲染

element 与 fiber 之间的对应关系
```
export const FunctionComponent = 0;       // 对应函数组件
export const ClassComponent = 1;          // 对应的类组件
export const IndeterminateComponent = 2;  // 初始化的时候不知道是函数组件还是类组件 
export const HostRoot = 3;                // Root Fiber 可以理解为跟元素 ， 通过reactDom.render()产生的根元素
export const HostPortal = 4;              // 对应  ReactDOM.createPortal 产生的 Portal 
export const HostComponent = 5;           // dom 元素 比如 <div>
export const HostText = 6;                // 文本节点
export const Fragment = 7;                // 对应 <React.Fragment> 
export const Mode = 8;                    // 对应 <React.StrictMode>   
export const ContextConsumer = 9;         // 对应 <Context.Consumer>
export const ContextProvider = 10;        // 对应 <Context.Provider>
export const ForwardRef = 11;             // 对应 React.ForwardRef
export const Profiler = 12;               // 对应 <Profiler/ >
export const SuspenseComponent = 13;      // 对应 <Suspense>
export const MemoComponent = 14;          // 对应 React.memo 返回的组件
export const SimpleMemoComponent = 15;
export const LazyComponent = 16;
export const IncompleteClassComponent = 17;
export const DehydratedFragment = 18;
export const SuspenseListComponent = 19;
export const FundamentalComponent = 20;
export const ScopeComponent = 21;
export const Block = 22;

```

### fiber 保存了哪些信息

```
function FiberNode(){

  this.tag = tag;                  // fiber 标签 证明是什么类型fiber。
  this.key = key;                  // key调和子节点时候用到。 
  this.type = null;                // dom元素是对应的元素类型，比如div，组件指向组件对应的类或者函数。  
  this.stateNode = null;           // 指向对应的真实dom元素，类组件指向组件实例，可以被ref获取。
 
  this.return = null;              // 指向父级fiber
  this.child = null;               // 指向子级fiber
  this.sibling = null;             // 指向兄弟fiber 
  this.index = 0;                  // 索引

  this.ref = null;                 // ref指向，ref函数，或者ref对象。

  this.pendingProps = pendingProps;// 在一次更新中，代表element创建
  this.memoizedProps = null;       // 记录上一次更新完毕后的props
  this.updateQueue = null;         // 类组件存放setState更新队列，函数组件存放
  this.memoizedState = null;       // 类组件保存state信息，函数组件保存hooks信息，dom元素为null
  this.dependencies = null;        // context或是时间的依赖项

  this.mode = mode;                //描述fiber树的模式，比如 ConcurrentMode 模式

  this.effectTag = NoEffect;       // effect标签，用于收集effectList
  this.nextEffect = null;          // 指向下一个effect

  this.firstEffect = null;         // 第一个effect
  this.lastEffect = null;          // 最后一个effect

  this.expirationTime = NoWork;    // 通过不同过期时间，判断任务是否过期， 在v17版本用lane表示。

  this.alternate = null;           //双缓存树，指向缓存的fiber。更新阶段，两颗树互相交替。
}
```
