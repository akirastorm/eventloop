# 前言
    众所周知，为了与浏览器进行交互，Javascript是一门非阻塞单线程脚本语言。
1. 为何单线程？ 因为如果在DOM操作中，有两个线程一个添加节点，一个删除节点，浏览器并不知道以哪个为准，所以只能选择一个主线程来执行代码，以防止冲突。虽然如今添加了webworker等新技术，但其依然只是主线程的子线程，并不能执行诸如I/O类的操作。长期来看，JS将一直是单线程。
2. 为何非阻塞？因为单线程意味着任务需要排队，任务按顺序执行，如果一个任务很耗时，下一个任务不得不等待。所以为了避免这种阻塞，我们需要一种非阻塞机制。这种非阻塞机制是一种异步机制，即需要等待的任务不会阻塞主执行栈中同步任务的执行。这种机制是如下运行的：
    * 所有同步任务都在主线程上执行，形成一个`执行栈（execution context stack）`
    * 等待任务的回调结果进入一种`任务队列(task queue)`。
    * 当主执行栈中的同步任务执行完毕后才会读取`任务队列`，任务队列中的异步任务（即之前等待任务的回调结果）会塞入主执行栈，
    * 异步任务执行完毕后会再次进入下一个循环。此即为今天文章的主角`事件循环(Event Loop)`

    用一张图展示这个过程：
    ![Markdown](http://i1.bvimg.com/628954/4e6a1c1bb3e70964.jpg)

# 正文

### 1.macro task与micro task
在实际情况中，上述的`任务队列(task queue)`中的异步任务分为两种：`微任务（micro task)`和`宏任务（macro task)`。
* micro task事件：`Promises(浏览器实现的原生Promise)`、`MutationObserver`、`process.nextTick`  
<br />
* macro task事件：`setTimeout`、`setInterval`、`setImmediate`、`I/O`、`UI rendering`
**这里注意：`script(整体代码)`即一开始在主执行栈中的同步代码本质上也属于macrotask，属于第一个执行的task**


> microtask和macotask执行规则：
* macrotask按顺序执行，浏览器的ui绘制会插在每个macrotask之间
* microtask按顺序执行，会在如下情况下执行：
    * 每个callback之后，只要没有其他的JS在主执行栈中
    * 每个macrotask结束时

下面来个简单例子：

```javascript
console.log(1);

setTimeout(function() {
  console.log(2);
}, 0);

new Promise(function(resolve,reject){
    console.log(3)
    resolve()
}).then(function() {
  console.log(4);
}).then(function() {
  console.log(5);
});

console.log(6);
```

一步一步分析如下：
* 1.同步代码作为第一个macrotask,按顺序输出：1 3 6
* 2.microtask按顺序执行：4 5
* 3.microtask清空后执行下一个macrotask：2

再来一个复杂的例子：
```javascript
// Let's get hold of those elements
var outer = document.querySelector('.outer');
var inner = document.querySelector('.inner');

// Let's listen for attribute changes on the
// outer element
new MutationObserver(function() {
  console.log('mutate');
}).observe(outer, {
  attributes: true
});

// Here's a click listener…
function onClick() {
  console.log('click');

  setTimeout(function() {
    console.log('timeout');
  }, 0);

  Promise.resolve().then(function() {
    console.log('promise');
  });

  outer.setAttribute('data-random', Math.random());
}

// …which we'll attach to both elements
inner.addEventListener('click', onClick);
outer.addEventListener('click', onClick);
```

假设我们创建一个有里外两部分的正方形盒子，里外都绑定了点击事件，此时点击内部，代码会如何执行？一步一步分析如下：
* 1.触发内部click事件，同步输出：click
* 2.将setTimeout回调结果放入macrotask队列
* 3.将promise回调结果放入microtask
* 4.将Mutation observers放入microtask队列，主执行栈中onclick事件结束，主执行栈清空
* 5.依序执行microtask队列中任务，输出：promise mutate
* 6.**注意此时事件冒泡，外部元素再次触发onclick回调，所以按照前5步再次输出：click promise mutate（我们可以注意到事件冒泡甚至会在microtask中的任务执行之后，microtask优先级非常高）**
* 7.macrotask中第一个任务执行完毕，依次执行macrotask中剩下的任务输出：timeout timeout

### 2.vue.nextTick实现
在 Vue.js 里是数据驱动视图变化，由于 JS 执行是单线程的，在一个 tick 的过程中，它可能会多次修改数据，但 Vue.js 并不会傻到每修改一次数据就去驱动一次视图变化，它会把这些数据的修改全部 push 到一个队列里，然后内部调用 一次 nextTick 去更新视图，所以数据到 DOM 视图的变化是需要在下一个 tick 才能完成。这便是我们为什么需要`vue.nextTick`.

这样一个功能和事件循环非常相似，在每个 task 运行完以后，UI 都会重渲染，那么很容易想到在 microtask 中就完成数据更新，当前 task 结束就可以得到最新的 UI 了。反之如果新建一个 task 来做数据更新，那么渲染就会进行两次。

所以在vue 2.4之前使用microtask实现nextTick，直接上源码
```javascript
var counter = 1
var observer = new MutationObserver(nextTickHandler)
var textNode = document.createTextNode(String(counter))
observer.observe(textNode, {
    characterData: true
})
timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
}
```
可以看到使用了MutationObserver

然而到了vue 2.4之后却混合使用microtask macrotask来实现，源码如下
```javascript
/* @flow */
/* globals MessageChannel */

import { noop } from 'shared/util'
import { handleError } from './error'
import { isIOS, isNative } from './env'

const callbacks = []
let pending = false

function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}

// Here we have async deferring wrappers using both micro and macro tasks.
// In < 2.4 we used micro tasks everywhere, but there are some scenarios where
// micro tasks have too high a priority and fires in between supposedly
// sequential events (e.g. #4521, #6690) or even between bubbling of the same
// event (#6566). However, using macro tasks everywhere also has subtle problems
// when state is changed right before repaint (e.g. #6813, out-in transitions).
// Here we use micro task by default, but expose a way to force macro task when
// needed (e.g. in event handlers attached by v-on).
let microTimerFunc
let macroTimerFunc
let useMacroTask = false

// Determine (macro) Task defer implementation.
// Technically setImmediate should be the ideal choice, but it's only available
// in IE. The only polyfill that consistently queues the callback after all DOM
// events triggered in the same loop is by using MessageChannel.
/* istanbul ignore if */
if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  macroTimerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else if (typeof MessageChannel !== 'undefined' && (
  isNative(MessageChannel) ||
  // PhantomJS
  MessageChannel.toString() === '[object MessageChannelConstructor]'
)) {
  const channel = new MessageChannel()
  const port = channel.port2
  channel.port1.onmessage = flushCallbacks
  macroTimerFunc = () => {
    port.postMessage(1)
  }
} else {
  /* istanbul ignore next */
  macroTimerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}

// Determine MicroTask defer implementation.
/* istanbul ignore next, $flow-disable-line */
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  microTimerFunc = () => {
    p.then(flushCallbacks)
    // in problematic UIWebViews, Promise.then doesn't completely break, but
    // it can get stuck in a weird state where callbacks are pushed into the
    // microtask queue but the queue isn't being flushed, until the browser
    // needs to do some other work, e.g. handle a timer. Therefore we can
    // "force" the microtask queue to be flushed by adding an empty timer.
    if (isIOS) setTimeout(noop)
  }
} else {
  // fallback to macro
  microTimerFunc = macroTimerFunc
}

/**
 * Wrap a function so that if any code inside triggers state change,
 * the changes are queued using a Task instead of a MicroTask.
 */
export function withMacroTask (fn: Function): Function {
  return fn._withTask || (fn._withTask = function () {
    useMacroTask = true
    const res = fn.apply(null, arguments)
    useMacroTask = false
    return res
  })
}

export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
    if (useMacroTask) {
      macroTimerFunc()
    } else {
      microTimerFunc()
    }
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```
可以看到使用setImmediate、MessageChannel等mascrotask事件来实现nextTick。

为什么会如此修改，其实看之前的事件冒泡例子就可以知道，由于microtask优先级太高，甚至会比冒泡快，所以会造成一些诡异的bug。如 issue [#4521](https://github.com/vuejs/vue/issues/4521)、[#6690](https://github.com/vuejs/vue/issues/6690)、[#6556](https://github.com/vuejs/vue/issues/6556)；但是如果全部都改成 macro task，对一些有重绘和动画的场景也会有性能影响，如 issue [#6813](https://github.com/vuejs/vue/issues/6813)。所以最终 nextTick 采取的策略是默认走 micro task，对于一些 DOM 交互事件，如 v-on 绑定的事件回调函数的处理，会强制走 macro task。


###### 参考资料
* [Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)
* [Vue.js 升级踩坑小记](https://github.com/DDFE/DDFE-blog/issues/24)
* [Vue 中如何使用 MutationObserver 做批量处理](https://www.zhihu.com/question/55364497/answer/144215284)
