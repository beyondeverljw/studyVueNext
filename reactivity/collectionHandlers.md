# packages/reactivity/src/collectionHandlers.ts

```js
import { toRaw, reactive, readonly } from './reactive'
import { track, trigger, ITERATE_KEY } from './effect'
import { TrackOpTypes, TriggerOpTypes } from './operations'
import { LOCKED } from './lock'
import { isObject, capitalize, hasOwn, hasChanged } from '@vue/shared'
```
```js
// ...省略ts代码
```
```js
const toReactive = <T extends unknown>(value: T): T =>
  isObject(value) ? reactive(value) : value
```
作用：如果参数value是一个对象，则为参数value创建一个对应的"可读可写响应对象"，并返回；如果不是对象，就直接返回value

```js
const toReadonly = <T extends unknown>(value: T): T =>
  isObject(value) ? readonly(value) : value
```
作用：如果参数value是一个对象，则为参数value创建一个对应的“只读响应对象”，并返回；如果不是对象，就直接返回value
```js
const getProto = <T extends CollectionTypes>(v: T): any =>
  Reflect.getPrototypeOf(v)
```
作用：获取参数v的原型对象
```js
function get(
  target: MapTypes,
  key: unknown,
  wrap: typeof toReactive | typeof toReadonly
) {
  target = toRaw(target)
  key = toRaw(key)
  track(target, TrackOpTypes.GET, key)
  return wrap(getProto(target).get.call(target, key))
}
```
作用：获取target中key对应的值，并为其创建对应的“响应对象”并返回。
* 获取target参数对应的原值（此时this指向的proxy）
* 获取key参数对应的原值
* [toRaw](./reactive.md)
* 通过track方法为target对应的key添加当前正在执行的effect [effect.js](./effect.md)
* 通过target的原型对象上的get方法（即Map或weakMap的原型中的get方法），获取target中，key对应的值
* 通过wrap包装器将上一步得到的值进行包装，得到“响应对象”
* 返回上一步得到的“响应对象”

```js
function has(this: CollectionTypes, key: unknown): boolean {
  const target = toRaw(this)
  key = toRaw(key)
  track(target, TrackOpTypes.HAS, key)
  return getProto(target).has.call(target, key)
}
```
作用：判断this中是否拥有key
* 获取当前this对应的原始对象，并存放于target
* 获取key对应的原始值
* 通过track为target对应的key添加当前正在执行的effect [effect.js](./effect.md)
* 通过target的原型对象中的has方法，判断target是否拥有key,并返回这个结果

```js
function size(target: IterableCollections) {
  target = toRaw(target)
  track(target, TrackOpTypes.ITERATE, ITERATE_KEY)
  return Reflect.get(getProto(target), 'size', target)
}
```
作用：返回target的size值
* 获取target对应的原始值，重新存放于target中
* 通过track为target对应的ITERATE_KEY添加当前正在执行的effect [effect.js](./effect.md)
* 从target的原型对象中获取size的值，并返回

```js
function add(this: SetTypes, value: unknown) {
  value = toRaw(value)
  const target = toRaw(this)
  const proto = getProto(target)
  const hadKey = proto.has.call(target, value)
  const result = proto.add.call(target, value)
  if (!hadKey) {
    /* istanbul ignore else */
    if (__DEV__) {
      trigger(target, TriggerOpTypes.ADD, value, { newValue: value })
    } else {
      trigger(target, TriggerOpTypes.ADD, value)
    }
  }
  return result
}
```
作用：将value对应的原始对象添加进当前this所指向的Set或WeakSet中
* 通过toRaw获取value的原始值,重新存放到value
* 通过toRaw获取this的原始值存放到target
* 获取target对应的原型对象，存放到proto
* 通过proto.has判断target指向的set或weakSet中是否存在value值，将结果存放到hadKey
* 通过proto.add将value添加到target中，将结果存放到result
* 如果hadKey为false,即target集合中没有value值，那么通过trigger触发target对应的depsMap中value对应的effect
* 返回result

```js
function set(this: MapTypes, key: unknown, value: unknown) {
  value = toRaw(value)
  key = toRaw(key)
  const target = toRaw(this)
  const proto = getProto(target)
  const hadKey = proto.has.call(target, key)
  const oldValue = proto.get.call(target, key)
  const result = proto.set.call(target, key, value)
  /* istanbul ignore else */
  if (__DEV__) {
    const extraInfo = { oldValue, newValue: value }
    if (!hadKey) {
      trigger(target, TriggerOpTypes.ADD, key, extraInfo)
    } else if (hasChanged(value, oldValue)) {
      trigger(target, TriggerOpTypes.SET, key, extraInfo)
    }
  } else {
    if (!hadKey) {
      trigger(target, TriggerOpTypes.ADD, key)
    } else if (hasChanged(value, oldValue)) {
      trigger(target, TriggerOpTypes.SET, key)
    }
  }
  return result
}
```
作用：用key对应的原始值做键，用value对应的原始值做值，存放进当前this对应的Map或WeakMap中
* 获取value对应的原始值，重新存放到value中
* 获取key对应的原始值，重新存放到key中
* 获取this对应的原始值，存放到target中
* 获取target的原型对象，存放到proto中
* 通过proto.has判断target对应的Map或WeakMap中是否有key对应的值，并将判断结果存放进hadKey
* 通过proto.get获取target中对应的key的值，存放到oldValue
* 以key为键，以value为值，通过proto.set存入到target对应Map或WeakMap中
* 如果hadKey为false,当前操作为“新增”，否则为“更新”，通过trigger触发在target对应的depsMap中，key对应的effect
* 返回result

```js
function deleteEntry(this: CollectionTypes, key: unknown) {
  key = toRaw(key)
  const target = toRaw(this)
  const proto = getProto(target)
  const hadKey = proto.has.call(target, key)
  const oldValue = proto.get ? proto.get.call(target, key) : undefined
  // forward the operation before queueing reactions
  const result = proto.delete.call(target, key)
  if (hadKey) {
    /* istanbul ignore else */
    if (__DEV__) {
      trigger(target, TriggerOpTypes.DELETE, key, { oldValue })
    } else {
      trigger(target, TriggerOpTypes.DELETE, key)
    }
  }
  return result
}
```
作用：从this对应的原始集合中，删除key对应的原始值所对应的值
* 获取key对应的原始值，重新存放到key中
* 获取this对应的原始对象，存放进target中
* 获取target对应的原型对象，存放到proto中
* 通过proto.has判断target是否拥有key，将判断结果存放到hadKey中
* 如果proto上存在get方法，就用get获取target集合中key对应的值，否则就是undefined,存放到oldValue中
* 调用proto上的delete方法删除target中对应的key,将操作结果存放到result中
* 如果hadKey为true,证明target中存在key，那么调用trigger方法触发target对应的depsMap中key收集的effect
* 返回result

```js
function clear(this: IterableCollections) {
  const target = toRaw(this)
  const hadItems = target.size !== 0
  const oldTarget = __DEV__
    ? target instanceof Map
      ? new Map(target)
      : new Set(target)
    : undefined
  // forward the operation before queueing reactions
  const result = getProto(target).clear.call(target)
  if (hadItems) {
    /* istanbul ignore else */
    if (__DEV__) {
      trigger(target, TriggerOpTypes.CLEAR, void 0, { oldTarget })
    } else {
      trigger(target, TriggerOpTypes.CLEAR)
    }
  }
  return result
}
```
作用：
* 获取this对应的原始对象，存放到target中
* 判断target的size是否为0,将判断结果存放到hadItems中
* 初始化oldTarget值
    * 如果是开发环境
        * 如果target是Map的实例
            * 就以target为参数，创建一个新的map实例
            * 否则就以target为参数，创建一个新的set实例
    * 否则为undefined
* 通过target的原型对象上的clear方法作用到target上，将结果存放到result中
* 如果hadItems为truth,即this对应的Map或Set中有元素，那么通过trigger方法触发target对应的depsMap中所有的effect执行
* 返回result

```js
function createForEach(isReadonly: boolean) {
  return function forEach(
    this: IterableCollections,
    callback: Function,
    thisArg?: unknown
  ) {
    const observed = this
    const target = toRaw(observed)
    const wrap = isReadonly ? toReadonly : toReactive
    track(target, TrackOpTypes.ITERATE, ITERATE_KEY)
    // important: create sure the callback is
    // 1. invoked with the reactive map as `this` and 3rd arg
    // 2. the value received should be a corresponding reactive/readonly.
    function wrappedCallback(value: unknown, key: unknown) {
      return callback.call(observed, wrap(value), wrap(key), observed)
    }
    return getProto(target).forEach.call(target, wrappedCallback, thisArg)
  }
}
```
作用：返回一个forEach函数，当返回的forEach执行时，过程如下：
* 将this存入observed中（在实际中，此时this指向的是一个proxy,这个proxy代理的是Map, Set, WeakMap，WeakSet之一的实例）
* 获取observed对应的原始对象，存放到target
* 根据isReadonly参数决定是用toReadonly还是toReactive作为包装器，存放到wrap中
* 通过track为target对应的depsMap中的ITERATE_KEY添加当前正在执行的effect
* 声明一个回调函数的包装器，这个包装器会接收被observed代理的原始对象中的value和key,在这个包装器内重新将value,key包装成”响应对像“作为callback的参数，并将callback中的this指向observed，执行callback，并返回执行结果
* 通过target的原型对象上的forEach作用到target，以包装器函数作为回调函数，执行并返回结果（此处的thisArg实际已无用）

```js
function createIterableMethod(method: string | symbol, isReadonly: boolean) {
  return function(this: IterableCollections, ...args: unknown[]) {
    const target = toRaw(this)
    const isPair =
      method === 'entries' ||
      (method === Symbol.iterator && target instanceof Map)
    const innerIterator = getProto(target)[method].apply(target, args)
    const wrap = isReadonly ? toReadonly : toReactive
    track(target, TrackOpTypes.ITERATE, ITERATE_KEY)
    // return a wrapped iterator which returns observed versions of the
    // values emitted from the real iterator
    return {
      // iterator protocol
      next() {
        const { value, done } = innerIterator.next()
        return done
          ? { value, done }
          : {
              value: isPair ? [wrap(value[0]), wrap(value[1])] : wrap(value),
              done
            }
      },
      // iterable protocol
      [Symbol.iterator]() {
        return this
      }
    }
  }
}
```
作用：返回一个用于返回一个迭代器的函数, 当这个函数执行时：
* 获取this对应的原始对象，存放到target
* 如果method的值为entries或者为（Symbol.iterator且target是Map的实例），则isPair为true，否则为false
* 执行target原始的迭代器方法，拿到原本的迭代器对象，存放到innerIterator
* 根据isReadonly决定是用toReadonly还是toReactive，将结果存放到wrap
* 通过track为target在depsMap中对应的ITERATE_KEY集合中添加当前正在执行的effect
* 返回迭代器对象，这个迭代器的next方法执行时：
    * 首先用原本的迭代器获取值，如果done为true,那么直接返回{ value, done }
    * 否则，根据isPair决定是返回一对值还是一个值，它们都将被包装成“响应对象”再返回

```js
function createReadonlyMethod(
  method: Function,
  type: TriggerOpTypes
): Function {
  return function(this: CollectionTypes, ...args: unknown[]) {
    if (LOCKED) {
      if (__DEV__) {
        const key = args[0] ? `on key "${args[0]}" ` : ``
        console.warn(
          `${capitalize(type)} operation ${key}failed: target is readonly.`,
          toRaw(this)
        )
      }
      return type === TriggerOpTypes.DELETE ? false : this
    } else {
      return method.apply(this, args)
    }
  }
}
```
作用：返回一个函数，当这个函数被执行时，过程：
* 如果LOCKED为true
    * 如果 type === TriggerOpTypes.DELETE，那么返回false,否则返回当前this
* 否则，将这个方法作用到当前this上调用

```js
const mutableInstrumentations: Record<string, Function> = {
  get(this: MapTypes, key: unknown) {
    return get(this, key, toReactive)
  },
  get size(this: IterableCollections) {
    return size(this)
  },
  has,
  add,
  set,
  delete: deleteEntry,
  clear,
  forEach: createForEach(false)
}
```
作用: 创建一组方法，当从一个proxy，这个proxy代理了(Map,Set,WeakMap,WeakSet之一的实例），并且这个proxy为“可读可写的响应对象”，当访问这个proxy的方法时，会从这组方法中获取对应的方法
```js
const readonlyInstrumentations: Record<string, Function> = {
  get(this: MapTypes, key: unknown) {
    return get(this, key, toReadonly)
  },
  get size(this: IterableCollections) {
    return size(this)
  },
  has,
  add: createReadonlyMethod(add, TriggerOpTypes.ADD),
  set: createReadonlyMethod(set, TriggerOpTypes.SET),
  delete: createReadonlyMethod(deleteEntry, TriggerOpTypes.DELETE),
  clear: createReadonlyMethod(clear, TriggerOpTypes.CLEAR),
  forEach: createForEach(true)
}
```
作用: 创建一组方法，当从一个proxy，这个proxy代理了(Map,Set,WeakMap,WeakSet之一的实例），并且这个proxy为“只读的响应对象”，当访问这个proxy的方法时，会从这组方法中获取对应的方法

```js
const iteratorMethods = ['keys', 'values', 'entries', Symbol.iterator]
iteratorMethods.forEach(method => {
  mutableInstrumentations[method as string] = createIterableMethod(
    method,
    false
  )
  readonlyInstrumentations[method as string] = createIterableMethod(
    method,
    true
  )
})
```
作用：为mutableInstrumentations和readonlyInstrumentations这两个对象添加keys,values, entries, Symbol.iterator方法
```js
function createInstrumentationGetter(
  instrumentations: Record<string, Function>
) {
  return (
    target: CollectionTypes,
    key: string | symbol,
    receiver: CollectionTypes
  ) =>
    Reflect.get(
      hasOwn(instrumentations, key) && key in target
        ? instrumentations
        : target,
      key,
      receiver
    )
}
```
作用：返回一个函数，这个函数用来决定当访问一个代理上的方法时，是用被代理对象自身的key对应的方法，还是用在instrumentations上声明的方法

```js
export const mutableCollectionHandlers: ProxyHandler<CollectionTypes> = {
  get: createInstrumentationGetter(mutableInstrumentations)
}
```
```js
export const readonlyCollectionHandlers: ProxyHandler<CollectionTypes> = {
  get: createInstrumentationGetter(readonlyInstrumentations)
}

```