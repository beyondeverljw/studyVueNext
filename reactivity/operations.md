# packages/reactivity/src/operations.ts

```js
export const enum TrackOpTypes {
  GET = 'get',
  HAS = 'has',
  ITERATE = 'iterate'
}
```
作用：枚举”track操作“的三种操作方式
```js
export const enum TriggerOpTypes {
  SET = 'set',
  ADD = 'add',
  DELETE = 'delete',
  CLEAR = 'clear'
}
```
作用：枚举”trigger操作“的四中操作方式