# nextTick
分析Vue之nextTick原理

## next-tick.js是Vue的源码

## 简化后代码如下
```javascript
<script>

  let isUsingMicroTask = false

  const callbacks = []
  let pending = false

  // 执行callbacks数组中的函数
  function flushCallbacks() {
    pending = false
    const copies = callbacks.slice(0)
    callbacks.length = 0
    for (let i = 0; i < copies.length; i++) {
      copies[i]()
    }
  }

  let timerFunc
  // 能力检测
  if (typeof Promise !== 'undefined') {
    const p = Promise.resolve()
    timerFunc = () => {
      p.then(flushCallbacks)
    }
    isUsingMicroTask = true
  } else if (typeof MutationObserver !== 'undefined' && (MutationObserver.toString() === '[object MutationObserverConstructor]')) {
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
  } else if (typeof setImmediate !== 'undefined') {
    timerFunc = () => {
      setImmediate(flushCallbacks)
    }
  } else {
    timerFunc = () => {
      setTimeout(flushCallbacks, 0)
    }
  }

  function nextTick(cb, ctx) {
    let _resolve
    callbacks.push(() => {
      if (cb) {
        try {
          cb.call(ctx)
        } catch (e) {
          console.error(e, ctx, 'nextTick')
          // handleError(e, ctx, 'nextTick')
        }
      } else if (_resolve) {
        _resolve(ctx)
      }
    })
    if (!pending) {
      pending = true
      // 执行timerFunc()函数时，会遇到Promise.resolve().then(flushCallbacks),
      // 而then属于微任务，所以在本次的同步任务中，不会立即执行flushCallbacks函数，而是放到then的微任务队列中
      // 但，在本次同步任务，可能会多次调用nextTick(cb, ctx)函数，故会一直callbacks.push()函数
      // 此时，Promise.resolve().then(flushCallbacks)中flushCallbacks函数，有对callbacks的引用，实际是闭包
      // 故，会等待当前同步任务执行完毕，才会执行flushCallbacks函数
      timerFunc()
    }
    if (!cb && typeof Promise !== 'undefined') {
      return new Promise(resolve => {
        // Promise的状态分离，当执行callbacks数组中具体的某个cb时，使得Promise状态从pending变更为fullfilled
        _resolve = resolve
      })
    }
  }

  // 验证
  let ctx = {
    name: '请叫我斗图王'
  }
  var cb1 = function () {
    console.log('cb1...', this.name)
  }
  var cb2 = function () {
    console.log('cb2...', this.name)
  }
  var cb3 = function () {
    console.log('cb3...', this.name)
  }
  var cb4 = function () {
    console.log('cb4...', this.name)
  }
  nextTick(cb1, ctx)
  nextTick(cb2, ctx)
  nextTick(cb3, ctx)
  nextTick(cb4, ctx)
  console.log('当前同步任务执行结束...')
  nextTick(null, ctx).then(() => {
    console.log('最后执行哦...')
  })

</script>
```

## 执行结果
![结果](nextTick.png)
