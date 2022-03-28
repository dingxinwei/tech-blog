
# vue nextTick解析
## 浏览器dom更新的流程
浏览器更新dom发生每一轮事件循环之后，一次事件循环是先从宏任务队列中取出一个宏任务执行，执行完宏任务后，清空微任务队列，执行所有的微任务。最后渲染dom
宏任务-->微任务-->渲染dom
## 宏任务
| # | 浏览器 | Node |
| ------ | ------ | ------ |
| setTimeout | √ | √ |
| setInterval | √ | √ |
| setImmediate | x | √ |
| requestAnimationFrame |√ | x |

## 微任务
| # | 浏览器 | Node |
| ------ | ------ | ------ |
| process.nextTick | x | √ |
| MutationObserver | √ | x |
| Promise.then catch finally | √ | √ |
## vue nextTick 原理分析
在vue中如果数据发生改变后操作dom，并不能获取到最新的dom，如果需要在数据改变后操作dom，需要使用Vue.nextTick(fn)，fn是一个函数,在fn中可以获取到最新的dom。
nextTick使用微任务或宏任务进行包装，优先使用微任务进行包装。在本轮事件循环修改了数据由于未进行dom渲染导致无法获取最新的dom，本轮事件循环结束后会对dom进行更新，然后在下一轮事件循环中可以获取最新的dom。
``` javascript
vue/src/core/util/next-tick.js

export let isUsingMicroTask = false //标志使用的是宏任务还是微任务，false表示使用的是宏任务，true表示使用的是微任务。

const callbacks = []//存放vue.nextTick的回调函数
let pending = false //false表示未执行异步包装器，true表示已执行异步包装器，如果同一个tick中再次调用nextTick不会再次执行异步包装器，保证在一个异步包装器中执行所有的回调

function flushCallbacks () {//执行所有的回调函数
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0 //清空回调函数数组
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}
let timerFunc //使用微任务或宏任务包装回调函数，是一个异步包装器
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  timerFunc = () => {
    p.then(flushCallbacks)
   
    if (isIOS) setTimeout(noop)
  }
  isUsingMicroTask = true
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  // PhantomJS and iOS 7.x
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
  
  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  observer.observe(textNode, {
    characterData: true
  })
  timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
  }
  isUsingMicroTask = true
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
 
  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else {
  // Fallback to setTimeout.
  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  callbacks.push(() => {//把所有的回调函数放入callbacks中
    if (cb) {
      try {
        cb.call(ctx)//执行传入的回调函数
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)//修改promise的状态为resolve
    }
  })
  if (!pending) {
    pending = true
    timerFunc()//执行包装器
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {//如果没有传入回调函数则返回一个promise
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```