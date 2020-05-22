# packages/runtime-core/src/scheduler.ts

```js
import { ErrorCodes, callWithErrorHandling } from './errorHandling'
import { isArray } from '@vue/shared'
```
```js
const queue: (Function | null)[] = []
const postFlushCbs: Function[] = []
const p = Promise.resolve()
let isFlushing = false
let isFlushPending = false
const RECURSION_LIMIT = 100
type CountMap = Map<Function, number>
```

```js
export function nextTick(fn?: () => void): Promise<void> {
  return fn ? p.then(fn) : p
}
```

作用：将指定的fn在微任务中执行，返回一个promise
* 如果指定了fn（不为undefined），那么fn将在微任务时间点被执行,并返回一个promise
* 如果fn为undefined,直接返回一个promise


```js
export function queueJob(job: () => void) {
  if (!queue.includes(job)) {
    queue.push(job)
    queueFlush()
  }
}
```

作用：将指定的函数加入queue
* 如果queue中没有当前指定的job，则将job加入，并调用queueFlush();

```js
export function invalidateJob(job: () => void) {
  const i = queue.indexOf(job)
  if (i > -1) {
    queue[i] = null
  }
}
```
作用： 将queue中与指定job相同的job设置为null

```js
export function queuePostFlushCb(cb: Function | Function[]) {
  if (!isArray(cb)) {
    postFlushCbs.push(cb)
  } else {
    postFlushCbs.push(...cb)
  }
  queueFlush()
}
```
作用：
* 将指定的cb添加到postFlushCbs中
    * 如果cb是一个函数，直接被push到postFlushCbs中；
    * 如果cb是一个函数数组，则将其打散，依次被push到postFlushCbs中；
* 调用queueFlush();

```js
function queueFlush() {
  if (!isFlushing && !isFlushPending) {
    isFlushPending = true
    nextTick(flushJobs)
  }
}
```

作用：如果isFlushing和isFlushPending同时为false，则将isFlushPending设为true,并在微任务时间点执行flushJobs

```js
const dedupe = (cbs: Function[]): Function[] => [...new Set(cbs)]

export function flushPostFlushCbs(seen?: CountMap) {
  if (postFlushCbs.length) {
    const cbs = dedupe(postFlushCbs)
    postFlushCbs.length = 0
    if (__DEV__) {
      seen = seen || new Map()
    }
    for (let i = 0; i < cbs.length; i++) {
      if (__DEV__) {
        checkRecursiveUpdates(seen!, cbs[i])
      }
      cbs[i]()
    }
  }
}
```
作用：处理了postFlushCbs中的所有函数
* 如果postFlushCbs不为空，则：
    * 通过dedupe为postFlushCbs去重，并将结果保存在cbs中，然后将postFlushCbs清空；
    * 如果是开发环境，会处理seen参数，如果seen为undefined，则将其初始化为一个Map实例
    * 在for循环中，依次调用cbs中的每个函数，如果是在开发环境中，还会检测每个函数递归调用的次数

```js
function flushJobs(seen?: CountMap) {
  isFlushPending = false
  isFlushing = true
  let job
  if (__DEV__) {
    seen = seen || new Map()
  }
  while ((job = queue.shift()) !== undefined) {
    if (job === null) {
      continue
    }
    if (__DEV__) {
      checkRecursiveUpdates(seen!, job)
    }
    callWithErrorHandling(job, null, ErrorCodes.SCHEDULER)
  }
  flushPostFlushCbs(seen)
  isFlushing = false
  // some postFlushCb queued jobs!
  // keep flushing until it drains.
  if (queue.length || postFlushCbs.length) {
    flushJobs(seen)
  }
}
```
作用：开始执行queue和postFlushCbs中的任务，直至它们中的function全部执行完
* 首先将isFlushPending的值设为false, isFlushing为true;
* 如果当前是开发环境，会去处理一下seen参数：
    * 如果seen为undefined，则将其初始化为一个Map实例
* 在while循环中，依次取出queue中job:
    * 如果job为Null,则跳过本次循环，继续下一个;
    * 如果job不为null，
        * 如果是开发环境，则调用checkRecursiveUpdates()，检测当前job被调用的次数。
    * 调用[callWithErrorHandling()](./errorHandling.md)
* 调用flushPostFlushCbs();
* 将isFlushing设为false
* 最后再次检查queue和postFlushCbs中是否还有剩余的“job”,如果有，则递归调用flushJobs()。（之所以需要再次检查，是因为有可能在一个job，新创建了job,并且添加进了queue或者postFlushCbs中）


```js
function checkRecursiveUpdates(seen: CountMap, fn: Function) {
  if (!seen.has(fn)) {
    seen.set(fn, 1)
  } else {
    const count = seen.get(fn)!
    if (count > RECURSION_LIMIT) {
      throw new Error(
        'Maximum recursive updates exceeded. ' +
          "You may have code that is mutating state in your component's " +
          'render function or updated hook or watcher source function.'
      )
    } else {
      seen.set(fn, count + 1)
    }
  }
}
```

作用：
* 通过seen来统计fn被调用的次数，如果fn的次数超过RECURSION_LIMIT（100次），就抛出异常
* 否则就增加fn的次数
* seen是一个Map实例，其使用function作为键，数字作为值