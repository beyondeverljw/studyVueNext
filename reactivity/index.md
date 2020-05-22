# packages/reactivity/src/index.ts

```js
export { ref, isRef, toRefs, Ref, UnwrapRef } from './ref'
```
[ref.js](./ref.md)
```js
export {
  reactive,
  isReactive,
  shallowReactive,
  readonly,
  isReadonly,
  shallowReadonly,
  toRaw,
  markReadonly,
  markNonReactive
} from './reactive'
```
[reactive.js](./reactive.md)
```js
export {
  computed,
  ComputedRef,
  WritableComputedRef,
  WritableComputedOptions,
  ComputedGetter,
  ComputedSetter
} from './computed'
```
[computed.js](./computed.md)
```js
export {
  effect,
  stop,
  trigger,
  track,
  enableTracking,
  pauseTracking,
  resetTracking,
  ITERATE_KEY,
  ReactiveEffect,
  ReactiveEffectOptions,
  DebuggerEvent
} from './effect'
```
[effect.js](./effect.md)
```js
export { lock, unlock } from './lock'
```
[lock.js](./lock.md)
```js
export { TrackOpTypes, TriggerOpTypes } from './operations'
```
[operations.js](operations.md)



