# packages/reactivity/src/reactive.ts

```js
import { isObject, toRawType } from '@vue/shared'
```
[isObject, toRawType](../shared/index.md)
```js
import {
  mutableHandlers,
  readonlyHandlers,
  shallowReadonlyHandlers,
  shallowReactiveHandlers
} from './baseHandlers'
```
[mutableHandlers, readonlyHandlers, shallowReadonlyHandlers, shallowReactiveHandlers](./baseHandlers.md)
```js
import {
  mutableCollectionHandlers,
  readonlyCollectionHandlers
} from './collectionHandlers'
```
[mutableCollectionHandlers, readonlyCollectionHandlers](./collectionHandlers.md)
```js
import { UnwrapRef, Ref } from './ref'
```
[UnwrapRef, Ref](./ref.md)
```js
import { makeMap } from '@vue/shared'
```
[makeMap](../shared/makeMap.md)

```js
// WeakMaps that store {raw <-> observed} pairs.
const rawToReactive = new WeakMap<any, any>()
const reactiveToRaw = new WeakMap<any, any>()
const rawToReadonly = new WeakMap<any, any>()
const readonlyToRaw = new WeakMap<any, any>()
// WeakSets for values that are marked readonly or non-reactive during
// observable creation.
const readonlyValues = new WeakSet<any>()
const nonReactiveValues = new WeakSet<any>()
```
作用：建立一组WeakMap
* rawToReactive： 用于存储“原始对象”=>“响应对象”的映射关系
* reactiveToRaw： 用于存储“响应对象”=>“原始对象”的映射关系
* rawToReadonly： 用于存储“原始对象”=>"只读响应对象"的映射关系
* readonlyToRaw： 用于存储“只读响应对象”=>"原始对象"的映射关系
* readonlyValues： 用于存储被标记为“只读”的值
* nonReactiveValues： 用于存储被标记为“非响应”的值

```js
const collectionTypes = new Set<Function>([Set, Map, WeakMap, WeakSet])
```
* 创建一个包含Set, Map，WeakMap, WeakSet四个构造器的集合（此处目的可以理解为枚举）

```js
const isObservableType = /*#__PURE__*/ makeMap(
  'Object,Array,Map,Set,WeakMap,WeakSet'
)
```
* 建立一个映射表，并返回一个函数（用于判断某类型在不在这个映射表中）
* [makeMap](../shared/makeMap.md)

```js
const canObserve = (value: any): boolean => {
  return (
    !value._isVue &&
    !value._isVNode &&
    isObservableType(toRawType(value)) &&
    !nonReactiveValues.has(value)
  )
}
```
* 判断value是不是可以被观察（被响应）
* 当同时满足以下四个条件时，认为value是可以被观察的(此时返回值为true)
    * value._isValue 为 falsely; 
    * value._isVNode 为 falsely;
    * [toRawType(value)](../shared/index.md) 后得到的值必须是 Object, Array, Map, Set, WeakMap, WeakSet中之一;
    * nonReactiveValues.has(value)为falsely
* 否则，返回false

```js
// ...省略ts代码
```

```js
export function reactive(target: object) {
  // if trying to observe a readonly proxy, return the readonly version.
  if (readonlyToRaw.has(target)) {
    return target
  }
  // target is explicitly marked as readonly by user
  if (readonlyValues.has(target)) {
    return readonly(target)
  }
  return createReactiveObject(
    target,
    rawToReactive,
    reactiveToRaw,
    mutableHandlers,
    mutableCollectionHandlers
  )
}
```
作用：为参数target对象创建一个对应的“可读可写的响应对象”。
* 如果 readonlyToRaw.has(target)为 truth, 证明target已经是一个“只读”的代理，直接返回target自身；
* 如果target被标记为一个“只读”，则将其包装为“只读”代理并返回；
* 通过createReactiveObject创建对应的“可读可写的响应对象”，并返回

```js
export function readonly<T extends object>(
  target: T
): Readonly<UnwrapNestedRefs<T>> {
  // value is a mutable observable, retrieve its original and return
  // a readonly version.
  if (reactiveToRaw.has(target)) {
    target = reactiveToRaw.get(target)
  }
  return createReactiveObject(
    target,
    rawToReadonly,
    readonlyToRaw,
    readonlyHandlers,
    readonlyCollectionHandlers
  )
}
```
作用: 为参数target对象创建一个对应的“只读的响应对象”
* 如果reactiveToRaw.has(target)返回true，则证明target是一个“可读可写的响应对象”, 那么通过这个响应对象获取与他对应的“原始对象”，即被他代理的对象。
* 通过createReactiveObject创建对应的“只读的响应对象”，并返回

```js
// Return a reactive-copy of the original object, where only the root level
// properties are readonly, and does NOT unwrap refs nor recursively convert
// returned properties.
// This is used for creating the props proxy object for stateful components.
export function shallowReadonly<T extends object>(
  target: T
): Readonly<{ [K in keyof T]: UnwrapNestedRefs<T[K]> }> {
  return createReactiveObject(
    target,
    rawToReadonly,
    readonlyToRaw,
    shallowReadonlyHandlers,
    readonlyCollectionHandlers
  )
}
```
作用：为参数target对象创建一个对应的“只读的浅层的响的应对象”，即不会对返回的对象递归的将结果包装成响应对象。

```js
// Return a reactive-copy of the original object, where only the root level
// properties are reactive, and does NOT unwrap refs nor recursively convert
// returned properties.
export function shallowReactive<T extends object>(target: T): T {
  return createReactiveObject(
    target,
    rawToReactive,
    reactiveToRaw,
    shallowReactiveHandlers,
    mutableCollectionHandlers
  )
}
```
作用：为参数target对象创建一个对应的“可读可写的浅层的响的应对象”，即会对返回的对象递归的将结果包装成响应对象。
```js
function createReactiveObject(
  target: unknown,
  toProxy: WeakMap<any, any>,
  toRaw: WeakMap<any, any>,
  baseHandlers: ProxyHandler<any>,
  collectionHandlers: ProxyHandler<any>
) {
  if (!isObject(target)) {
    if (__DEV__) {
      console.warn(`value cannot be made reactive: ${String(target)}`)
    }
    return target
  }
  // target already has corresponding Proxy
  let observed = toProxy.get(target)
  if (observed !== void 0) {
    return observed
  }
  // target is already a Proxy
  if (toRaw.has(target)) {
    return target
  }
  // only a whitelist of value types can be observed.
  if (!canObserve(target)) {
    return target
  }
  const handlers = collectionTypes.has(target.constructor)
    ? collectionHandlers
    : baseHandlers
  observed = new Proxy(target, handlers)
  toProxy.set(target, observed)
  toRaw.set(observed, target)
  return observed
}
```
作用：具体的创建“响应对象”
* 若参数target不是一个非null的对象，则直接返回target;如果当前运行环境是开发环境的话，会提示 value cannot be made reative + target的字符串化
* 尝试从toProxy中获取target(即target已经拥有对应的“响应对象”)，若其值存在（!== undefined）的话，就返回从toProxy中获取对应的“响应对象”
* 尝试从 toRaw里获取target（即target是一个“响应对象”）,如果能获取到（toRaw.has(target)），就返回target
* 若target经canObserve判断后返回false（即target不是一个可以“响应”的对象）,则返回target
* 以上均不满足，则为target建立代理：
    * 首先确定handlers: 如果target的构造函数在collectionTypes里，handlers就是collectionsHandlers，否则就是baseHandlers
    * 根据已确定的handlers，为target创建对应的“响应对象”
    * 在toProxy中添加target与对应的“响应对象”的映射关系
    * 在toRaw中添加“响应对象”与target的映射关系

```js
export function isReactive(value: unknown): boolean {
  return reactiveToRaw.has(value) || readonlyToRaw.has(value)
}
```
作用：判断参数value是不是一个“响应对象”
* 如果reactiveToRaw中存在value，即value是一个“可读可写的响应对象”，那么返回true,即value是一个“响应对象”
* 否则，如果readonlyToRaw中存在value, 即value是一个“只读的响应对象”，那么返回true，即value是一个“响应对象”
* 其他条件返回false，即value不是一个“响应对象”

```js
export function isReadonly(value: unknown): boolean {
  return readonlyToRaw.has(value)
}
```
作用：判断value是不是一个“只读的响应对象”
* 如果readonlyToRaw中存在value，那么返回true，即value是一个“只读的响应对象”

```js
export function toRaw<T>(observed: T): T {
  return reactiveToRaw.get(observed) || readonlyToRaw.get(observed) || observed
}
```
作用：获取observed参数指定的“响应对象”的“原始对象”
* 尝试从reactiveToRaw中获取当前observed所代理的原始值，如果有就返回，没有继续；
* 尝试从readonlyToRaw中获取当前observer所代理的原始值，如果有就返回，没有继续；
* 不做处理，直接返回observed；

```js
export function markReadonly<T>(value: T): T {
  readonlyValues.add(value)
  return value
}
```
作用：将value参数添加进readonlyValues中，即将value归类为“只读的值",并返回value
```js
export function markNonReactive<T>(value: T): T {
  nonReactiveValues.add(value)
  return value
}
```
作用：将value参数添加进nonReactiveValues中，即将value归类为“非响应的值",并返回value


