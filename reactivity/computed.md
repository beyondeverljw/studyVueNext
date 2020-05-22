# packages/reactivity/src/computed.ts

```js
import { effect, ReactiveEffect, trigger, track } from './effect'
import { TriggerOpTypes, TrackOpTypes } from './operations'
import { Ref, UnwrapRef } from './ref'
import { isFunction, NOOP } from '@vue/shared'
```
```js
//...省略ts代码
```
```js
export function computed<T>(
  getterOrOptions: ComputedGetter<T> | WritableComputedOptions<T>
) {
```

```js
  let getter: ComputedGetter<T>
  let setter: ComputedSetter<T>
```
    
```js
  if (isFunction(getterOrOptions)) {
    getter = getterOrOptions
    setter = __DEV__
      ? () => {
          console.warn('Write operation failed: computed value is readonly')
        }
      : NOOP
  } else {
    getter = getterOrOptions.get
    setter = getterOrOptions.set
  }
```
* 如果getterOrOptions是function类型
    * 就将其赋值到局部变量getter上，同时将setter设置为空函数（如果是在开发环境，将会进行相应的信息提示）
* 否则就将getter设置为getterOrOptions.set,同时将setter设置为getterOrOptions.set

```js
  let dirty = true
  let value: T
  let computed: ComputedRef<T>

  const runner = effect(getter, {
    lazy: true,
    // mark effect as computed so that it gets priority during trigger
    computed: true,
    scheduler: () => {
      if (!dirty) {
        dirty = true
        trigger(computed, TriggerOpTypes.SET, 'value')
      }
    }
  })
```
将getter包装成effect,这个effect不会立即执行（lazy为true），只用调用runner()时才会执行
```js  
  computed = {
    _isRef: true,
    // expose effect so computed can be stopped
    effect: runner,
    get value() {
      if (dirty) {
        value = runner()
        dirty = false
      }
      track(computed, TrackOpTypes.GET, 'value')
      return value
    },
    set value(newValue: T) {
      setter(newValue)
    }
  } as any
```
将value包装成ref
```js  
  return computed
}
```
返回ref