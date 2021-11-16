
# state

## 类组件中的state

### 一、setState用法

> setState(obj, callback)
* 1.第一个参数 obj
        当obj为对象时，为覆盖的state
        当obj为函数时，那么当前组件的state和props将作为参数，返回新的state
* 2.第二个参数 callback
        callback为一个函数，函数执行上下文中可以获取当前setState更新后的值，可以作为以来state变化的副作用函数

### 二、触发setState,React底层都做了哪些事

* 首先，setState会产生当前更新的优先级（老版本用 expirationTime , 新版本用 lane ）
* 其次，React会从 fiber Root 根部fiber向下调和子节点，调和阶段将对比发生更新的地方，更新对比 expirationTime, 找到发生更新的组件，合并 state，然后出发 render 函数，得到新的UI视图，完成 render 阶段
* 接下来到 commit 阶段，commit 阶段替换真实 DOM，完成此次更新流程
* 在 commit 阶段，会执行 setState 中的 callback 函数，到此为止完成了一次 setState 的全过程

触发setState -> 计算expirationTime -> 更新调度，调和fiber树 -> 合并state，执行fiber -> 替换真实DOM -> 执行callback函数



#### 类组件如何限制state更新视图
* pureComponent 可以对 state 和 props 进行浅比较，如果没有发生变化那么组件不更新
* shouldComponentUpdate 生命周期通过判断前后 state 变化来决定是否更新，需要返回true,反之返回false

### 三、setState原理

类组件初始化的过程中绑定了 Updater 对象，对于如何调用 setState 方法，实际是React底层调用了 Updater 对象上的 enqueueSetState 方法

> packages/react-reconciler/src/ReactFiberClassComponent.new.js

```
enqueueSetState(){

     /* 每一次调用`setState`，react 都会创建一个 update 里面保存了 */
     const update = createUpdate(expirationTime, suspenseConfig);

     /* callback 可以理解为 setState 回调函数，第二个参数 */
     callback && (update.callback = callback) 

     /* enqueueUpdate 把当前的update 传入当前fiber，待更新队列中 */
     enqueueUpdate(fiber, update); 

     /* 开始调度更新 */
     scheduleUpdateOnFiber(fiber, expirationTime);

}
```
__enqueueSetState__ 作用其实很简单，就是创建一个update，然后放入当前的 fiber 对象的待更新队列中，最后开启调度更新，进入上述更新流程

__那么问题来了，react的 batchUpdate 批量更新是是什么时候加上去的呢？__

正常的 state 更新，ui交互都离不开用户的事件，React 是采用事件合成的形式，每一个事件都是由 React 事件系统统一调度，所以 state 批量更新是和事件系统息息相关的
> react-dom/src/events/DOMLegacyEventPluginSystem.js
```
/* 在`legacy`模式下，所有的事件都将经过此函数同一处理 */
function dispatchEventForLegacyPluginEventSystem(){
    // handleTopLevel 事件处理函数
    batchedEventUpdates(handleTopLevel, bookKeeping);
}
```

下面是 batchedEventUpdates 方法

> packages/legacy-events/ReactGenericBatching.js
```
function batchedEventUpdates(fn,a){
    /* 开启批量更新  */
   isBatchingEventUpdates = true;
  try {
    /* 这里执行了的事件处理函数， 比如在一次点击事件中触发setState,那么它将在这个函数内执行 */
    return batchedEventUpdatesImpl(fn, a, b);
  } finally {
    /* try 里面 return 不会影响 finally 执行  */
    /* 完成一次事件，批量更新  */
    isBatchingEventUpdates = false;
  }
}
```
如上可以得出流程
 * 在 React 事件执行之前，通过 isBatchingEventUpdates=true 来开启事件批量更新开关
 * 执行 batchedEventUpdatesImpl  
 * 当事件执行结束，再通过 isBatchingEventUpdates=false 来关闭更新开关
 * 然后在 scheduleUpdateOnFiber 中根据这个开关来确定是否进行批量更新

举一个例子
```
export default class index extends React.Component{
    state = { number:0 }
    handleClick= () => {
          this.setState({ number:this.state.number + 1 },()=>{   console.log( 'callback1', this.state.number)  })
          console.log(this.state.number)
          this.setState({ number:this.state.number + 1 },()=>{   console.log( 'callback2', this.state.number)  })
          console.log(this.state.number)
          this.setState({ number:this.state.number + 1 },()=>{   console.log( 'callback3', this.state.number)  })
          console.log(this.state.number)
    }
    render(){
        return <div>
            { this.state.number }
            <button onClick={ this.handleClick }  >number++</button>
        </div>
    }
}
```
点击handleClick 打印 __0,0,0,callback1 1,callback2 1,callback3 1__

[如上代码，在整个 React 上下文执行栈中会变成这样](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/478aef991b4146c898095b83fe3dc0e7~tplv-k3u1fbpfcp-watermark.awebp)

```
setTimeout(()=>{
    this.setState({ number:this.state.number + 1 },()=>{   console.log( 'callback1', this.state.number)  })
    console.log(this.state.number)
    this.setState({ number:this.state.number + 1 },()=>{    console.log( 'callback2', this.state.number)  })
    console.log(this.state.number)
    this.setState({ number:this.state.number + 1 },()=>{   console.log( 'callback3', this.state.number)  })
    console.log(this.state.number)
})
```
打印 ： __callback1 1 , 1, callback2 2 , 2,callback3 3 , 3__
[那么在整个 React 上下文执行栈中就会变成如下图这样:](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/48e730fc687c4ce087e5c0eab2832273~tplv-k3u1fbpfcp-watermark.awebp)

批量更新规则被打破

__如何在异步环境下，继续开启批量更新?__

React-Dom 中提供了手动批量更新的方法，__unstable_batchedUpdates__ , 可以去手动批量更新
```
import ReactDOM from 'react-dom'
const { unstable_batchedUpdates } = ReactDOM

setTimeout(()=>{
    unstable_batchedUpdates(()=>{
        this.setState({ number:this.state.number + 1 })
        console.log(this.state.number)
        this.setState({ number:this.state.number + 1})
        console.log(this.state.number)
        this.setState({ number:this.state.number + 1 })
        console.log(this.state.number) 
    })
})

```
打印： __0 , 0 , 0 , callback1 1 , callback2 1 ,callback3 1__

在实际工作中，unstable_batchedUpdates 可以用来合并多次 setState 或者 useState ，可以作为性能优化，避免一次数据交互发生多次渲染

__如何提升更新的优先级？__
React-Dom 提供了 flushSync，flushSync可以将回调函数中的更新任务放在一个较高的优先级中。
React设定了很多不同优先级的更新任务，如果一次更新任务在 flushSync 回调函数内部，那么将会获得一个较高的优先级

```
handerClick=()=>{
    setTimeout(()=>{
        this.setState({ number: 1  })
    })
    this.setState({ number: 2  })
    ReactDOM.flushSync(()=>{
        this.setState({ number: 3  })
    })
    this.setState({ number: 4  })
}
render(){
   console.log(this.state.number)
   return ...
}
```
打印 __3 4 1__
* 首先，发现了flushSync this.setState({ number: 3  }) 设定了较高优先级，合并 2 和 3 批量更新成 3 ，所以先打印 3
* 更新 4
* 最后更新 setTimeout 中的 1

__flushSync补充说明__: flushSync 在同步条件下，会合并之前的 setState 或者 useState ，如果发现了 flushSync ，就会优先执行更新，如果之前有未更新的 setState 或者 useState ,那么就会一起合并，所以上述代码的 2 和 3 就被批量更新到了 3。所以 3 先被打印

综上所述，React 同一级别更新优先级是：

flushSync 中的 setState __>__ 正常直行上下文中的 setState __>__ setTimeout, promise 中的setState

## 函数组件中的state

### useState用法
>  [ ①state , ②dispatch ] = useState(③initData)
* ①state，目的是提供给UI，作为渲染视图的数据源
* ②dispatch，改变 state 的函数
* ③initData，state的初始值，有两种情况，1、非函数，将作为 state 的初始值，2、第二种情况是函数，返回值作为 useState 的初始值

__initData 为非函数的情况__
```
/* 此时将把 0 作为初使值 */
const [ number , setNumber ] = React.useState(0)
```
__initData 为函数的情况__
```
 const [ number , setNumber ] = React.useState(()=>{
       /*  props 中 a = 1 state 为 0-1 随机数 ， a = 2 state 为 1 -10随机数 ， 否则，state 为 1 - 100 随机数   */
       if(props.a === 1) return Math.random() 
       if(props.a === 2) return Math.ceil(Math.random() * 10 )
       return Math.ceil(Math.random() * 100 ) 
    })
```

__对于dispatch也有两种情况__
* 非函数，直接作为新的值，赋值给 state
* 函数，如果 dispatch 是一个函数，参数是上一次返回最新的 state，返回值作为新的 state

__如何监听state__
使用另一个hooks，useEffect()

__dispatch更新特点__
在函数组件中，dispatch 更新效果和类组件是一样的效果，但是 useState 有一点不同，当调用 state 的函数 dispatch。在本次函数执行上下文中，是获取不到最新的state的值的，例如：
```
const [ number , setNumber ] = React.useState(0)
const handleClick = ()=>{
    ReactDOM.flushSync(()=>{
        setNumber(2) 
        console.log(number) 
    })
    setNumber(1) 
    console.log(number)
    setTimeout(()=>{
        setNumber(3) 
        console.log(number)
    })   
}
```
__结果： 0 0 0__

原因很简单，函数组件更新就是函数执行，在函数一次执行过程中，函数内部所有变量重新声明，所以改变的 state 只有在下一次函数执行时才会被更新,所以在同一个函数执行上下文中 number 一直是 0 

__useState注意事项__
在使用 useState 中的 dispatch 更新 state 的时候,千万不要传入相同的 state , 这样不会更新视图
 ```
 export default function Index(){
    const [ state  , dispatchState ] = useState({ name:'alien' })
    const  handleClick = ()=>{ // 点击按钮，视图没有更新。
        state.name = 'Alien'
        dispatchState(state) // 直接改变 `state`，在内存中指向的地址相同。
    }
    return <div>
         <span> { state.name }</span>
        <button onClick={ handleClick }  >changeName++</button>
    </div>
}
 ```

 在 useState 的 dispatchState 处理逻辑中,会浅比较两次的 state ,发现 state 相同, 那么就不会开启更新调度任务, demo 中两次 state 都指向了相同的内存空间，所以默认 state 相等，就不会发生视图更新

 解决方法：把 dispatchState(state) 改成 dispatchState({...state}),浅拷贝了对象，重新申请了一个内存空间

__类组件中的 setState 和函数组件中的 useState 有什么异同 ？__

相同点：
* 从原理出发，setState 和 useState 更新视图，底层都调用了 scheduleUpdateOnFiber 方法，而且事件驱动情况下都有批量更新原则
不同点：
* 在不是 pureComponent 组件模式下， setState不会浅比较两次 state 的值，在没有其他优化手段情况下，只要调用就会更新。但是，useState 中的dispatch 会默认比较两次的 state 是否相同，不同才会更新组件
* setState 有专门监听函数变化的 callback，可以获取最新的state，useState 只能通 useEffect 来执行 state 变化引起的副作用
* setState 在底层逻辑处理上主要是新老 state 进行合并处理，而 useState 更倾向于重新赋值