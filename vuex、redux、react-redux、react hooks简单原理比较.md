# vuex、redux、react-redux、react hooks简单原理比较 

## redux (观察者模式)
```js
function createStore(reducer) { 
    let state
    let handlers = [];
    function subscribe(events) {
        handlers.push(events);
    }
    function dispatch(action) { 
        state = reducer(state, action);
        console.log(state, 11111)
        handlers.forEach(item => item())
    }
    function getState() {
        return state
    }
    return {
        dispatch,
        getState,
        subscribe
    }
}

var reducer = (state = 0, action) => {
    switch(action.type) {
        case 'INCREMENT':
            return state + 1;
        case 'DECREMENT':
            return state - 1;
        default:
            return state;
    }
}
var s = createStore(reducer);
s.subscribe(() => {
    console.log(s.getState())
})
s.dispatch({ type: "INCREMENT" })
console.log(s.getState(), 12)
```
## react-redux
##### 1、创建`reducer`
```tsx
import { StateObj } from './store.interface';
const initialState: StateObj = {    
    count: 0
}
export function reducer(state: StateObj = initialState, action: any): StateObj {    
    switch(action.type) {      
        case 'plus':        
        return {            
            ...state,                    
            count: state.count + 1        
        }      
        case 'subtract':        
        return {            
            ...state,            
            count: state.count - 1        
        }      
        default:        
        return initialState    
    }
}
```
##### 2、 `store`
* store 提供外界一个获取状态的函数`getState`、订阅函数`subscribe`、派发动作函数`dispatch`
```tsx
import { reducer } from './reducer';
import { StoreActionObj, StoreStateObj, StateObj } from './store.interface';

export function createStore<T extends StoreStateObj>(reducer: (p1: T, p2: StoreActionObj) => T) {
    let currState = {} as any
    let obsevers: Array<() => void> = []
    function getState(): T {
        return currState
    }
    function dispatch(action: StoreActionObj) {
        currState = reducer(currState, action)
        obsevers.forEach(item => item())
    }     
    function subscribe(fn: () => void) {
        obsevers.push(fn)
    }
    dispatch({ type: '@@REDUX_INIT' })  //初始化store数据 不然外界会在获取state时候是 NaN
    return {
        getState,
        dispatch,
        subscribe
    }
}
export const store = createStore<StateObj>(reducer)  //创建store
store.subscribe(() => { console.log('组件1收到store的通知') })
store.subscribe(() => { console.log('组件2收到store的通知') })
// store.dispatch({ type: 'plus' })    //执行加法操作,给count加1
// store.dispatch(asyncDisPatchAction as any)    //执行加法操作,给count加1
// console.log(store.getState())
```
##### 3、 `Provider`
* `provider`的作用 就是为子组件提供一个 可以访问 `store` 的 `context` 并渲染子组件
```tsx
import React from 'react';
import PropTypes from 'prop-types';
import { StoreObj, StateObj } from './store.interface';
export class Provider extends React.Component<ProviderPropsObj> {
    store: any = null
    constructor(props: ProviderPropsObj, context: any) {
        super(props, context)
        this.store = props.store
    }
    static childContextTypes = {    
        store: PropTypes.object  
    } 
    getChildContext() {
        return {
            store: this.store
        }
    }
    render() {
        return this.props.children
    }
}

type ProviderPropsObj = {store: StoreObj<StateObj>, [props: string]: any}
```
> 类组件获取store --- this.context
> 方法组件获取 --- function test(props, context) {console.log(context)}
##### 4、`connect`
* `connect`作用是 将高阶包装后的组件可以使用传入的 `state` 和 `dispatch`
```tsx
import React from 'react';
import PropTypes from 'prop-types';
export function connect(mapStateToProps: any, mapDispatchToProps: any) {    
    return function(Component: (typeof React.Component)) {      
        return class Connect extends React.Component {    
            static contextTypes = {
                store: PropTypes.object
            }  
            componentDidMount() {          
                //从context获取store并订阅更新          
                this.context.store.subscribe(this.handleStoreChange.bind(this));        
            }       
            handleStoreChange() {          
                // 触发更新  也可以用其他形式更新 view层 例如setState         
                this.forceUpdate()        
            }        
            render() {          
                return (            
                    <Component              
                        // 传入该组件的props,需要由connect这个高阶组件原样传回原组件              
                        { ...this.props }              
                        // 根据mapStateToProps把state挂到this.props上              
                        { ...mapStateToProps(this.context.store.getState()) }               
                        // 根据mapDispatchToProps把dispatch(action)挂到this.props上              
                        { ...mapDispatchToProps(this.context.store.dispatch) }                 
                    />              
                )        
            }      
        }
    }
}
```
* 使用 比如对App组件进行包装
```tsx
// App.tsx
const mapStateToProps = (state: any) => {  
  return {      
      count: state.count  
  }
}

const mapDispatchToProps = (dispatch: any) => {  
  return {
      addCount: () => {          
          dispatch(addCountAction)      
      }
  }
}

export default connect(mapStateToProps, mapDispatchToProps)(App as any);
```
```tsx
// App.tsx 
const App = (props: any, connect: any) => {
  console.log(props, 'props')   // 里面已经有 属性count 和 方法addCount了
  console.log(connect, 'connect')
  function clickHandler() {
    // props.addCount()
    props.asyncAddCount()
  }
  return (
    <div className="App">
      <button onClick={clickHandler}>点击</button>
      {props.count}
    </div>
  );
}
```
##### 5、 `middleWare`
* 其实中间件就是 在 `dispatch` 前后做一些事情 所以核心就是重写 `dispatch`
* 例如 有一个 打印日志的中间件 `logger` 和 编写异步action的中间件 `thunk`
```tsx
import { StoreActionObj, StoreObj } from './store.interface';
export function logger(store: StoreObj) {
    let next = store.dispatch  // 缓存 dispatch
    return function(action: StoreActionObj) {  // 重写dispatch 方法
        console.log('state中间件')  // 打印日志之后 调用原来的dispatch
        let res = next(action)
        return res
    }
}

export function thunk(store: StoreObj) {
    let next = store.dispatch
    return function (action: StoreActionObj | any) {
        console.log('thunk中间件')
        return typeof action === 'function' ? action(store.dispatch) : next(action)  
    }
}

export function appleMiddleware(store: StoreObj, middlewares: any[]) {
    middlewares = [ ...middlewares ]      
    middlewares.reverse()  // 后面加入的中间件 应该包含前面中间件的功能 所以要对调下顺序 (洋葱模型)
    middlewares.forEach((middleware: any) => {
        store.dispatch = middleware(store)   
    })
}   
```
* 再用connec 往 App组件加入一个 异步 处理 state 的dispatch
```tsx
const addCountAction = {  
  type: 'plus'
}

const mapStateToProps = (state: any) => {  
  return {      
      count: state.count  
  }
}

// thunk 中间件 允许dispatch传入一个方法
const asyncDisPatchAction = function(dispatch: any) {
  setTimeout(() => {
    dispatch({ type: 'plus' })
  }, 2000)
}
const mapDispatchToProps = (dispatch: any) => {  
  return {
      addCount: () => {          
          dispatch(addCountAction)      
      },
      asyncAddCount: () => {
        dispatch(asyncDisPatchAction)
      }
  }
}

export default connect(mapStateToProps, mapDispatchToProps)(App as any);
```
![1584520480420.jpg](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/14813b0f84f140e9ae3723165147ef3e~tplv-k3u1fbpfcp-zoom-1.image)

原文链接： [https://juejin.im/post/5def4831e51d45584b585000#heading-13](https://juejin.im/post/5def4831e51d45584b585000#heading-13)

## vuex  （defineProperty）
```ts
import { VueConstructor } from 'vue';
import { Vue } from 'vue-property-decorator';
export class Store {
    private getters: {[props: string]: gettersFun} = {}
    private options: {[props: string]: any} = {};
    private mutations: {[props: string]: mutationsFun} = {}
    private actions: {[props: string]: mutationsFun} = {}
    private _vm: Vue | null = null
    constructor(options = {}, VM: VueConstructor) {
        this.options = options;
        VM.mixin({ beforeCreate: vuexInit });
        this._vm = new Vue({
            data: {
                state: this.options.state   // 用Vue存放 是为了 Vue视图层能检测到 state 的变化
            }
        })
        //   核心代码 看看这个forEach 干了些啥
       // 遍历getters，让属性被defineProperty检测，获取值的时候 执行下getters方法并传入state就行了
        this.forEach(this.options.getters, this.registerGetters.bind(this))
        this.forEach(this.options.mutations, this.registerMutation.bind(this))
        this.forEach(this.options.actions, this.registerAction.bind(this))
    }
    get state() {
        return this._vm ? this._vm['_data'].state : null
    }
    private registerGetters(name: string, fun: gettersFun) {
        Object.defineProperty(this.getters, name, {
            get: () => {
                return fun(this.state)
            }
        })
    }
    // mutation方法的触发都是commit  mutation只是做 存 commit找到对应的mutation 再取
    private registerMutation(name: string, fun: mutationsFun) {
        this.mutations[name] = fun
    }
    // 找到对应的 mutation 调用并把 state 当做参数传入 以便 mutation接受并可派生处理的状态
    // 按道理这里还能传第二个参数，自定义 mutation 在派生状态时的 额外处理参数
    // Store.commit('', { count: 10 })
    public commit(mutationName: string) {
        this.mutations[mutationName](this.options.state)
    }
    private registerAction(name: string, fun: (content: Store) => void) {
        this.actions[name] = fun
    }
    // dispatch 也一样 找到 mutation
    // action: {
    //     increasementAsync(context) {
    //         context.commit('')
    //     }
    // }
    public dispatch(name: string) {
        this.actions[name](this)
    }
    private forEach(getterObj, fun: (name: string, value: gettersFun) => void) {
        Object.keys(getterObj).forEach((item: string) => {
            fun(item, getterObj[item])
        })
    }
}

function vuexInit() {
    // 所有组件在 beforeCreate 的时候 都会执行一次这个函数
    const options = this.$options
    if (options.store) {
        // 组件内部设定了store,则优先使用组件内部的store
        this.$store = typeof options.store === 'function'
        ? options.store()
        : options.store
    } else if (options.parent && options.parent.$store) {
        // 组件内部没有设定store,则从根App.vue下继承$store方法
        this.$store = options.parent.$store
    }
}

export interface VueStoreObj {
    state: any;
    getters: {
        [props: string]: gettersFun
    }
}

type gettersFun = (p: any) => any
type mutationsFun = (state: any) => void


```
## react hook （闭包）
```ts
let stateArr: any[] = []
let currIndex: number = 0    // 重新渲染的时候  要重置为0

function useState<T>(val: T): [T, (newState: T) => void] {
    const currI = currIndex
    stateArr[currIndex] = stateArr[currIndex] || val
    function setState(newState: T) {
        stateArr[currI] = newState
    }
    return [stateArr[currIndex++], setState]
}


let [num, setNum] = useState<number>(0)
let [age, setAge] = useState<number>(20)

console.log(setNum(2))
console.log(setAge(30))
console.log(stateArr)    // [2, 30]
```