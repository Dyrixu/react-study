# Ref

## Ref对象创建

### __①类组件React.createRef__
```
class Index extends React.Component{
    constructor(props){
       super(props)
       this.currentDom = React.createRef(null)
    }
    componentDidMount(){
        console.log(this.currentDom)
    }
    render= () => <div ref={ this.currentDom } >ref对象模式获取元素或组件</div>
}
```

React.createRef 原理很简单，createRef只做了一件事，创建了一个属性为 current 的属性，用来保存用过 ref 获取的 DOM 元素，组件实例

> packages/react/src/ReactCreateRef.js

```
export function createRef(): RefObject {
  const refObject = {
    current: null,
  };
  if (__DEV__) {
    Object.seal(refObject);
  }
  return refObject;
}
```

createRef 一般用于类组件创建 Ref 对象，可以将 Ref 对象绑定在类组件的实例上，方便后续操作 Ref，__不可以在函数组件使用 createRef__

#### __类组件获取 ref 的三种途径__

__① Ref属性是一个字符串。__
```
/* 类组件 */
class Children extends Component{  
    render=()=><div>hello,world</div>
}
/* TODO:  Ref属性是一个字符串 */
export default class Index extends React.Component{
    componentDidMount(){
       console.log(this.refs)
    }
    render=()=> <div>
        <div ref="currentDom"  >字符串模式获取元素或组件</div>
        <Children ref="currentComInstance"  />
    </div>
}
```
React 在组件底层逻辑会判断类型：
* 如果是 DOM 元素，会把真实 DOM绑定在组件 this.refs(组件实例下的 refs 属性上)
* 如果是类组件，会把子组件实例绑定在 refs 上

__② Ref 属性是一个函数。__

```
class Children extends React.Component{  
    render=()=><div>hello,world</div>
}
/* TODO: Ref属性是一个函数 */
export default class Index extends React.Component{
    currentDom = null
    currentComponentInstance = null
    componentDidMount(){
        console.log(this.currentDom)
        console.log(this.currentComponentInstance)
    }
    render=()=> <div>
        <div ref={(node)=> this.currentDom = node }  >Ref模式获取元素或组件</div>
        <Children ref={(node) => this.currentComponentInstance = node  }  />
    </div>
}
```
当用一个函数来标记 Ref 的时候，将作为 callback 形式，等到真实 DOM 创建阶段，执行 callback，获取的 DOM 元素或组件实例，将以毁掉函数第一个参数形式传入

__③ Ref属性是一个ref对象。__

React.createRef()


### __②函数组件 useRef__

通过 hooks 使用 useRef
```
export default function Index(){
    const currentDom = React.useRef(null)
    React.useEffect(()=>{
        console.log( currentDom.current ) // div
    },[])
    return  <div ref={ currentDom } >ref对象模式获取元素或组件</div>
}
```

useRef 底层逻辑和 createRef 差不多，就是 ref 保存的位置不同：
类组件有一个实例 instance 能够维护 ref，但是由于函数组件每一次更新都是一次新的开始，所有变量重新声明，所以 useRef 不能像 createRef 把 ref 对象直接暴露出去，如果这样，每一次函数组件执行就会重新维护 Ref，此时，ref 就会随着函数组件执行被重置。
为了解决这个问题，hooks 和函数组件对应的 fiber 对象建立联系，将 useRef 产生的 ref 对象存在函数组件对应的 fiber 上，函数组件每次执行，只要组件不被销毁，函数组件的 fiber 对象一直存在，所以 ref 等信息也就被保存了下来





## ref 高阶用法

### forwardRef 转发 ref

__① 场景一：跨层级获取__
```
// 孙组件
function Son (props){
    const { grandRef } = props
    return <div>
        <div> i am alien </div>
        <span ref={grandRef} >这个是想要获取元素</span>
    </div>
}
// 父组件
class Father extends React.Component{
    constructor(props){
        super(props)
    }
    render(){
        return <div>
            <Son grandRef={this.props.grandRef}  />
        </div>
    }
}
const NewFather = React.forwardRef((props,ref)=> <Father grandRef={ref}  {...props} />)
// 爷组件
class GrandFather extends React.Component{
    constructor(props){
        super(props)
    }
    node = null 
    componentDidMount(){
        console.log(this.node) // span #text 这个是想要获取元素
    }
    render(){
        return <div>
            <NewFather ref={(node)=> this.node = node } />
        </div>
    }
}
```
__② 场景二:合并转发ref__

通过 forwardRef 可以用来传递合并之后自定义的 ref
> 场景：想通过Home绑定ref，来获取子组件Index的实例index，dom元素button，以及孙组件Form的实例
```
// 表单组件
class Form extends React.Component{
    render(){
       return <div>{...}</div>
    }
}
// index 组件
class Index extends React.Component{ 
    componentDidMount(){
        const { forwardRef } = this.props
        forwardRef.current={
            form:this.form,      // 给form组件实例 ，绑定给 ref form属性 
            index:this,          // 给index组件实例 ，绑定给 ref index属性 
            button:this.button,  // 给button dom 元素，绑定给 ref button属性 
        }
    }
    form = null
    button = null
    render(){
        return <div   > 
          <button ref={(button)=> this.button = button }  >点击</button>
          <Form  ref={(form) => this.form = form }  />  
      </div>
    }
}
const ForwardRefIndex = React.forwardRef(( props,ref )=><Index  {...props} forwardRef={ref}  />)
// home 组件
export default function Home(){
    const ref = useRef(null)
     useEffect(()=>{
         console.log(ref.current)
     },[])
    return <ForwardRefIndex ref={ref} />
}
```
流程：
* 通过 useRef 创建 ref 对象，通过 forwardRef 将当前 ref 对象传递给子组件
* 向 Home 组件传递的 ref 对象上，绑定 form 组件实例、index 组件实例，button DOM 元素

__forwardRef__ 让 ref 可以通过 props，那么如果用 ref 对象标记的 ref，那么 ref对象就可以通过 props 的形式提供给子孙组件消费，当然子孙组件也可以改变 ref 对象里面的属性，或者赋予新的属性，这种 forwardRef + ref 模式一定程度上打破了 React 单向数据流动的原则，当然，绑定在 ref 对象上的属性，不限于组件实例或者DOM元素，也可以是属性和方法

__③ 场景三：高阶组件转发__
如果通过高阶组件包裹一个原始类组件，有一个问题：
如果 HOC 没有处理 ref，那么由高阶组件就会返回一个新组件，所以当使用 HOC 包装后的组件时，标记的 ref 会指向 HOC返回的组件，而并不是 HOC 包裹的原始类组件，解决方法：包裹一层 forwardRef 可以对 HOC 做一层处理
```
function HOC(Component){
  class Wrap extends React.Component{
     render(){
        const { forwardedRef ,...otherprops  } = this.props
        return <Component ref={forwardedRef}  {...otherprops}  />
     }
  }
  return  React.forwardRef((props,ref)=> <Wrap forwardedRef={ref} {...props} /> ) 
}
class Index extends React.Component{
  render(){
    return <div>hello,world</div>
  }
}
const HocIndex =  HOC(Index)
export default ()=>{
  const node = useRef(null)
  useEffect(()=>{
    console.log(node.current)  /* Index 组件实例  */ 
  },[])
  return <div><HocIndex ref={node}  /></div>
}
```
这样就可以正常访问到 Index 组件实例了



### ref实现组件通信
__① 类组件 ref__

类组件可以直接通过 ref 获取组件实例，实现组件通信

```
import React, { useRef, useState } from 'react'

class Son extends React.Component {
    state = {
        sonMsg: '',
        fatherMsg: ''
    }

    sonClick = () => {
        this.props.toFather(this.state.sonMsg)
    }
    toSon = (msg) => {
        this.setState({
            fatherMsg: msg
        })
    }
    render() {
        return (
            <div>
                <h1>子组件</h1>
                <input type="text" onChange={(msg) => this.setState({ sonMsg: msg.target.value })} />
                <button onClick={this.sonClick}>通知父亲组件</button>
                <div>父组件说：{ this.state.fatherMsg }</div>
            </div>
        )
    }
}

export default function Father() {
    const [fatherMsg, setFatherMsg] = useState('')
    const [sonMsg, setSonMsg] = useState('')
    const sonInstance = useRef(null)


    const fatherClick = () => {
        sonInstance.current.toSon(fatherMsg)
    }

    const setSon = (msg) => {
        setSonMsg(msg)
    }
    return (
        <div>
            <h1>父亲组件</h1>
            <input type="text" onChange={(msg) => setFatherMsg(msg.target.value)} />
            <button onClick={fatherClick}>通知子组件</button>
            <div>子组件说：{sonMsg}</div>
            <Son ref={sonInstance} toFather={setSon} />
        </div>
    )
}
```
流程分析：

* 子组件暴露 toSon 方法供父组件使用，父组件通过 sonInstance.current.toSon(fatherMsg) 调用显示子组件内容
* 父组件提供给子组件 toFather 方法，子组件通过 this.props.toFather(this.state.sonMsg) 调用，改变父组件内容，实现双向通信

__② 函数组件 forwardRef + useImperativeHandle__

函数组件本身没有实例，React hooks 提供了useImperativeHandle 

useImperativeHandle 接受三个参数：
* 第一个参数：ref： 接受 forWardRef 传递过来的 ref
* 第二个参数：createHandle： 处理函数，返回值暴露给父组件的 ref 对象
* 第三个参数：deps：依赖项 deps，依赖项更改形成新的 ref 对象


forwardRef + useImperativeHandle 可以完全让函数组件也能流畅的使用 ref 通信
[流程图](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/59238390306849e89069e6a4bb6ded9d~tplv-k3u1fbpfcp-watermark.awebp)

```
import React, { forwardRef, useImperativeHandle, useRef, useState } from 'react'

const Son = (props, ref) => {
    const inputRef = useRef(null)
    const [inputValue, setInputValue] = useState('')
    const handleChange = (e) => {
        setInputValue(e.target.value)
    }
    useImperativeHandle(ref, () => {
        const handleRefs = {
            onFocus() {
                inputRef.current.focus()
            },
            onChangeValue(value) {
                setInputValue(value)
            }
        }
        return handleRefs
    })
    return (
        <div>
            <input type="text" ref={inputRef} value={inputValue} onChange={handleChange} />
        </div>
    )
}

const ForwardSon = forwardRef(Son)

class Index extends React.Component {
    cur = null
    handleClick = () => {
        const { onFocus, onChangeValue } = this.cur
        onFocus()
        onChangeValue('let\'s learn React')
    }
    render() {
        return (
            <div>
                <ForwardSon ref={cur => this.cur = cur} />
                <button onClick={this.handleClick}>操控子组件</button>
            </div>
        )
    }
}

export default Index
```
流程：
* 父组件用 ref 标记子组件，由于子组件 Son 是函数组件，所以没有实例，所以用 forwardRef 转发 Ref
* 子组件用 useImperativeHandle 接收父组件的 ref，将 input 的聚焦方法 onFocus 和 改变 input 值的 onChangeValue 方法合并传递给 ref
* 父组件通过 ref 下的 onFocus 和 onChangeValue 控制子组件


### 函数组件缓存数据