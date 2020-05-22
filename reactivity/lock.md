# packages/reactivity/src/lock.ts

```js
// global immutability lock
export let LOCKED = true

export function lock() {
  LOCKED = true
}

export function unlock() {
  LOCKED = false
}
```

* 导出LOCKED变量， 默认为true
* 导出lock(), unlock()方法