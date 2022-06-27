### vue3.0数据检测
```js
let rawToReactive = new WeakMap()
let reactiveToRaw = new WeakMap()

function reactive(target, cb?) {
    // 先看一下当前对象有没有可响应的数据
    let reactiveObj = rawToReactive.get(target)
    if (reactiveObj) {
        return reactiveObj
    }
    // 再看当前对象是不是可响应的对象
    if (reactiveToRaw.has(target)) {
        return reactiveToRaw.get(target)
    }
    // 都没有的话 就建立一个新的响应对象
    let timer = null
    let observed = new Proxy(target, {
        get: (target, key, receiver) => {
            let obj = Reflect.get(target, key, receiver)
            // receiver 会返回代理过后的对象，如果还是对象的话 就递归 保证所有属性都被代理
            return typeof obj === 'object' ? reactive(obj, cb) : obj
        },
        set: (target, key, value, receiver) => {
            // 如果是个数组的push操作 其实会触发两次set操作 一次是值本身 一次是length属性
            // 方案1  setTimeout
            // if (timer) {
            //     clearTimeout(timer)
            // }
            // timer = setTimeout(() => {
            //     cb && cb()
            // })

            // 方案2 判断当前的key是否是target的属性（vue3）
            //  值本身触发肯定不是  当时length的时候 
            const hadKey = target.hasOwnProperty(key)
            const oldValue = target[key]
            // console.log(target, key, hadKey)
            if (!hadKey) {
                cb && cb()
            } else if (oldValue !== value) {
                cb && cb()
            }
            return Reflect.set(target, key, value, receiver)
        }
    })
    rawToReactive.set(target, observed)
    reactiveToRaw.set(observed, target)
    return observed
}

let testObj = { a: { b: 1 } }
let temArr = [1,2,3]
// console.log(reactive(testObj))
// let tem = reactive(testObj, () => { 
//   console.log(1111)
// })
// tem.a.b = 3
let tem2 = reactive(temArr, () => console.log('arr'))
// tem2.push(4)
tem2.push('length')
console.log(rawToReactive, reactiveToRaw)
```
__`优化`__
```js
const rawToReactive = new WeakMap()
const reactiveToRaw = new WeakMap()

// utils
function isObject(val) {
  return typeof val === 'object'
}

function hasOwn(val, key) {
  const hasOwnProperty = Object.prototype.hasOwnProperty
  return hasOwnProperty.call(val, key)
}

// traps
function createGetter() {
  return function get(target, key, receiver) {
    // receiver参数 会返回已经代理后的对象
    //  Reflect.get返回了代理对象的内层结构
    const res = Reflect.get(target, key, receiver)
    return isObject(res) ? reactive(res) : res
  }
}

function set(target, key, val, receiver) {
  const hadKey = hasOwn(target, key)
  const oldValue = target[key]

  val = reactiveToRaw.get(val) || val
  const result = Reflect.set(target, key, val, receiver)

  if (!hadKey) {
    console.log('trigger ...')
  } else if(val !== oldValue) {
    console.log('trigger ...')
  }

  return result
}

// handler
const mutableHandlers = {
  get: createGetter(),
  set: set,
}

// entry
function reactive(target) {
  return createReactiveObject(
    target,
    rawToReactive,
    reactiveToRaw,
    mutableHandlers,
  )
}

function createReactiveObject(target, toProxy, toRaw, baseHandlers) {
  let observed = toProxy.get(target)
  // 原数据已经有相应的可响应数据, 返回可响应数据
  if (observed !== void 0) {
    return observed
  }
  // 原数据已经是可响应数据
  if (toRaw.has(target)) {
    return target
  }
  observed = new Proxy(target, baseHandlers)
  toProxy.set(target, observed)
  toRaw.set(observed, target)
  return observed
}
```
[原文链接]([https://juejin.im/post/5d99be7c6fb9a04e1e7baa34#heading-8](https://juejin.im/post/5d99be7c6fb9a04e1e7baa34#heading-8)
)