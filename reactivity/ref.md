# packages/reactivity/src/ref.ts

```ts
import { track, trigger } from './effect'
import { TrackOpTypes, TriggerOpTypes } from './operations'
import { isObject } from '@vue/shared'
import { reactive, isReactive } from './reactive'
import { ComputedRef } from './computed'
import { CollectionTypes } from './collectionHandlers'
```
```js
const isRefSymbol = Symbol()
```
```js
// ...省略ts代码
```

```js
const convert = <T extends unknown>(val: T): T =>
  isObject(val) ? reactive(val) : val
```
* 定义一个转换方法，这个方法的功能是：
    * 如果val是一个非null的object，则返回经reactive作用的结果
    * 否则对val不做处理直接返回

```js
// ...省略ts代码
```
```js
export function isRef(r: any): r is Ref {
  return r ? r._isRef === true : false
}
```
* 判断r是不是一个Ref:
    * 如果r为falsely，返回false;
    * 否则判断r的_isRef属性是否为true : 
        * 是： true
        * 否： false 

```js
// ...省略ts代码
```
```js
export function ref(value?: unknown) {
  if (isRef(value)) {
    return value
  }
  value = convert(value)
  const r = {
    _isRef: true,
    get value() {
      track(r, TrackOpTypes.GET, 'value')
      return value
    },
    set value(newVal) {
      value = convert(newVal)
      trigger(
        r,
        TriggerOpTypes.SET,
        'value',
        __DEV__ ? { newValue: newVal } : void 0
      )
    }
  }
  return r
}
```
* 如果value已经是一个Ref,直接返回value;
* 将value通过reactive方法转换成“响应对象（reactive或者readonly）”
* 创建一个ref,它的实质是这样的一个对象：
    * 这个对象上有一个_isRef属性，用来标识这个对象是一个Ref，然后提供一个value属性：
        * 对value属性的取值操作，就是返回经convert后的value参数（函数的参数）
        * 对value的赋值操作，则是把newVal经convert后的值存进原value参数存取的地方
* 返回这个新创建的Ref对象 

```js
export function toRefs<T extends object>(
  object: T
): { [K in keyof T]: Ref<T[K]> } {
  if (__DEV__ && !isReactive(object)) {
    console.warn(`toRefs() expects a reactive object but received a plain one.`)
  }
  const ret: any = {}
  for (const key in object) {
    ret[key] = toProxyRef(object, key)
  }
  return ret
}
```
作用：将参数object自身或原型链中的所有可枚举属性包装成ref。
* 如果当前运行环境是开发环境，则判断object是不是一个可以响应的对象，由isReactive判断，如果不是，则给与提示“to Refs() expects a reactive object but received a plain one”;
* 在非开发环境下，这里不会有提示；
* 创建一个空对象ret
* 遍历object的所有可枚举属性，包括原型链上的属性，将每一个属性经toProxyRef处理,将结果赋予到ret的同名属性上
* 返回对象ret;

```js
function toProxyRef<T extends object, K extends keyof T>(
  object: T,
  key: K
): Ref<T[K]> {
  return {
    _isRef: true,
    get value(): any {
      return object[key]
    },
    set value(newVal) {
      object[key] = newVal
    }
  } as any
}
```
* 为object[key]做一个Ref


```js
// ...省略ts代码
```

