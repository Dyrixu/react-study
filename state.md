
# state

## 类组件中的state

### setState用法

    setState(obj, callback)
    1.第一个参数 obj
        当obj为对象时，为覆盖的state
        当obj为函数时，那么当前组件的state和props将作为参数，返回新的state
    2.第二个参数 callback
        callback为一个函数，函数执行上下文中可以获取当前setState更新后的值，可以作为以来state变化的副作用函数
    




## 函数组件中的state