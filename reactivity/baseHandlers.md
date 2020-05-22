# packages/reactivity/src/baseHandlers.ts

```js
import { reactive, readonly, toRaw } from './reactive'
import { TrackOpTypes, TriggerOpTypes } from './operations'
import { track, trigger, ITERATE_KEY } from './effect'
import { LOCKED } from './lock'
import { isObject, hasOwn, isSymbol, hasChanged, isArray } from '@vue/shared'
import { isRef } from './ref'
```
```js
const builtInSymbols = new Set(
  Object.getOwnPropertyNames(Symbol)
    .map(key => (Symbol as any)[key])
    .filter(isSymbol)
)
```
[isSymbol](../shared/index.md)
* 得到Symbol中所有值类型为symbol的属性:
    * asyncIterator, hasInstance, isConcatSpreadable, iterator, match, matchAll, replace, search, species, split, toPrimitive, toStringTag, unscopables, 目前共13个
* Symbol中的所有自身属性，目前共18个
    * 除上面13个外还有： length, name, prototype, for, keyFor 

```js
const get = /*#__PURE__*/ createGetter()
const shallowReactiveGet = /*#__PURE__*/ createGetter(false, true)
const readonlyGet = /*#__PURE__*/ createGetter(true)
const shallowReadonlyGet = /*#__PURE__*/ createGetter(true, true)
```
```js
const arrayIdentityInstrumentations: Record<string, Function> = {}
;['includes', 'indexOf', 'lastIndexOf'].forEach(key => {
  arrayIdentityInstrumentations[key] = function(
    value: unknown,
    ...args: any[]
  ): any {
    return toRaw(this)[key](toRaw(value), ...args)
  }
})
```
* 定义一个空对象，并为这个对象添加3个方法
    * 分别是： includes, indexOf, lastIndexOf
* [toRaw](./reactive.md)

1. toRaw(this): 尝试获取this指向的代理所对应的原始对象，如果没有，就是this所指向的对象；
2. toRaw(this)[key]: 上一步获取到的对象自身的includes, indexOf, lastIndexOf方法的引用
3. toRaw(value): 尝试获取value所代理的原始对象，如果没有就是value自身
* 总结： 就是以toRaw(value)，...args为参数，调用this所代理的原始对象或this自身的includes, indexOf, lastIndexOf方法。

```js
function createGetter(isReadonly = false, shallow = false) {
  return function get(target: object, key: string | symbol, receiver: object) {
    if (isArray(target) && hasOwn(arrayIdentityInstrumentations, key)) {
      return Reflect.get(arrayIdentityInstrumentations, key, receiver)
    }
    const res = Reflect.get(target, key, receiver)
    if (isSymbol(key) && builtInSymbols.has(key)) {
      return res
    }
    if (shallow) {
      track(target, TrackOpTypes.GET, key)
      // TODO strict mode that returns a shallow-readonly version of the value
      return res
    }
    if (isRef(res)) {
      return res.value
    }
    track(target, TrackOpTypes.GET, key)
    return isObject(res)
      ? isReadonly
        ? // need to lazy access readonly and reactive here to avoid
          // circular dependency
          readonly(res)
        : reactive(res)
      : res
  }
}
```
作用：返回一个function，用于proxy中的get陷阱，当这个陷阱触发时,执行如下：
* 如果target是一个数组，并且要访问的key是includes, indexOf, lastIndexOf之一，那么就返回arrayIdentityInstrumentations中存放的方法的引用；
* 获取target中key对应的值，存放于res
* 如果key是Symbol类型的，并且是js内置的Symbol值，那么直接返回res
* 如果shallow为truth，那么调用track方法为target对应的key，收集当前正在激活的effect，并返回res
* 如果res是一个ref，那么返回ref.value；[ref](./ref.md)
* 调用track方法为target对应的key，收集当前正在激活的effect
* 如果res是一个Object那么返回res对应的响应对象，如果isReadonly为true,就是用readonly()包装，否则就是reactive()包装
* 否则，返回res

```js
const set = /*#__PURE__*/ createSetter()
const shallowReactiveSet = /*#__PURE__*/ createSetter(false, true)
const readonlySet = /*#__PURE__*/ createSetter(true)
const shallowReadonlySet = /*#__PURE__*/ createSetter(true, true)
```
```js
function createSetter(isReadonly = false, shallow = false) {
  return function set(
    target: object,
    key: string | symbol,
    value: unknown,
    receiver: object
  ): boolean {
    if (isReadonly && LOCKED) {
      if (__DEV__) {
        console.warn(
          `Set operation on key "${String(key)}" failed: target is readonly.`,
          target
        )
      }
      return true
    }

    const oldValue = (target as any)[key]
    if (!shallow) {
      value = toRaw(value)
      if (isRef(oldValue) && !isRef(value)) {
        oldValue.value = value
        return true
      }
    } else {
      // in shallow mode, objects are set as-is regardless of reactive or not
    }

    const hadKey = hasOwn(target, key)
    const result = Reflect.set(target, key, value, receiver)
    // don't trigger if target is something up in the prototype chain of original
    if (target === toRaw(receiver)) {
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
    }
    return result
  }
}
```
作用：返回一个function，用于proxy中的set陷阱，当这个陷阱触发时,执行如下：
* 如果isReadonly为true,且LOCKED也为true,那么直接返回true,如果是开发环境，会提示”target is readonly“
* 获取target中key对应的值存放于oldValue
* 如果shallow为falsely
    * 那么获取value对应的”原始对象“（value有可能是一个”响应对象“）
    * 如果oldValue是一个ref,且value不是一个ref,那么把value赋值给oldValue.value,并返回true
* 调用hasOwn判断key是否属于target自身 [hasOwn](../shared/index.md)，并将结果存放于hadKey
* 通过Reflect.set为target的key属性赋值value，将结果存放于result
* 如果target与receiver对应的原始对应是同一个对象，即target不是在receiver的原型链上，那么触发对应的effect
    * 如果hadKey为false,代表是”新增操作“，通过trigger触发target对应的key中的effect
    * 否则，代表"更新操作",通过trigger触发target对应的key中的effect
* 返回操作结果result

```js
function deleteProperty(target: object, key: string | symbol): boolean {
  const hadKey = hasOwn(target, key)
  const oldValue = (target as any)[key]
  const result = Reflect.deleteProperty(target, key)
  if (result && hadKey) {
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
作用：用于proxy中的deleteProperty陷阱，当这个陷阱触发时,执行如下：
* 调用hasOwn判断key是否是target自身的属性，将结果存放于hadKey
* 获取target[key],存放进oldValue
* 通过Reflect.deleteProperty删除target中的key,将结果存放进result
* 如果result为true,即删除成功，且key是target的自身属性
    * 通过trigger触发target对像对应的key的effect,如果是开发环境，还会多传一个oldvalue，用于信息处理
* 返回操作结果result

```js
function has(target: object, key: string | symbol): boolean {
  const result = Reflect.has(target, key)
  track(target, TrackOpTypes.HAS, key)
  return result
}
```
作用：用于proxy中的has陷阱，当这个陷阱触发时,执行如下：
* 通过Reflect.has判断target上是否存在key，将结果存放于result
* 通过track为target对应的key添加当前环境中正在执行的effect
* 返回操作结果result


```js
function ownKeys(target: object): (string | number | symbol)[] {
  track(target, TrackOpTypes.ITERATE, ITERATE_KEY)
  return Reflect.ownKeys(target)
}
```
作用：用于proxy中的ownKeys陷阱，当这个陷阱触发时,执行如下：
* 通过track为target对应的ITERATE_KEY添加当前环境正在执行的effect
* 返回Reflect.onwKeys的结果

```js
export const mutableHandlers: ProxyHandler<object> = {
  get,
  set,
  deleteProperty,
  has,
  ownKeys
}
```
作用：返回用于”可读可写的响应对象“的一组陷阱，用于proxy
```js
export const readonlyHandlers: ProxyHandler<object> = {
  get: readonlyGet,
  set: readonlySet,
  has,
  ownKeys,
  deleteProperty(target: object, key: string | symbol): boolean {
    if (LOCKED) {
      if (__DEV__) {
        console.warn(
          `Delete operation on key "${String(
            key
          )}" failed: target is readonly.`,
          target
        )
      }
      return true
    } else {
      return deleteProperty(target, key)
    }
  }
}
```
作用：返回用于”只读的响应对象“的一组陷阱，用于proxy

```js
export const shallowReactiveHandlers: ProxyHandler<object> = {
  ...mutableHandlers,
  get: shallowReactiveGet,
  set: shallowReactiveSet
}
```
作用：返回用于”可读可写的浅层的响应对象“的一组陷阱，用于proxy

```js
export const shallowReadonlyHandlers: ProxyHandler<object> = {
  ...readonlyHandlers,
  get: shallowReadonlyGet,
  set: shallowReadonlySet
}
```
作用：返回用于”只读的浅层的响应对象“的一组陷阱，用于proxy