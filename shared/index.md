# packages/shared/src/index.ts

```js
import { makeMap } from './makeMap'
export { makeMap }
export * from './patchFlags'
export * from './shapeFlags'
export * from './globalsWhitelist'
export * from './codeframe'
export * from './mockWarn'
export * from './normalizeProp'
export * from './domTagConfig'
export * from './domAttrConfig'
export * from './escapeHtml'
export * from './looseEqual'
```
[patchFlags](./patchFlags.md) | [shapeFlags](./shapeFlags.md) | [globalsWhitelist](./globalsWhitelist.md) | [codeframe](./codeframe.md) | [mockWarn](./mockWarn.md) | [normalizeProp](./normalizeProp.md) | [domTagConfig](./domTagConfig.md) | [domAttrConfig](./domAttrConfig.md) | [escapeHtml](./escapeHtml.md) | [looseEqual](./looseEqual.md)

```js
export const EMPTY_OBJ: { readonly [key: string]: any } = __DEV__
  ? Object.freeze({})
  : {}
export const EMPTY_ARR: [] = []
export const NOOP = () => {}
```
分别声明一个空对象，空数组，空函数，其中空对象在开发环境中通过,Object.freeze冻结（不可添加，不可修改）

```js
export const NO = () => false
```
声明一个永远返回false的函数

```js
export const isOn = (key: string) => key[0] === 'o' && key[1] === 'n'
```
用于判断参数key是不是以'on'开头

```js
export const extend = <T extends object, U extends object>(
  a: T,
  b: U
): T & U => {
  for (const key in b) {
    ;(a as any)[key] = b[key]
  }
  return a as any
}
```
将b中的所有可枚举的 （字段：值） 赋值到a中，包括b中的实例域属性，和原型域属性，若a中存在同名变量，则被覆盖

```js
const hasOwnProperty = Object.prototype.hasOwnProperty
export const hasOwn = (
  val: object,
  key: string | symbol
): key is keyof typeof val => hasOwnProperty.call(val, key)
```
判断参数key是不是参数val的实例属性

```js
export const isArray = Array.isArray
export const isFunction = (val: unknown): val is Function => typeof val === 'function'
export const isString = (val: unknown): val is string => typeof val === 'string'
export const isSymbol = (val: unknown): val is symbol => typeof val === 'symbol'
export const isObject = (val: unknown): val is Record<any, any> =>
  val !== null && typeof val === 'object'
export const isPromise = <T = any>(val: unknown): val is Promise<T> => {
  return isObject(val) && isFunction(val.then) && isFunction(val.catch)
}
```
分别用于判断是否是Array、Function、String、Symbol、Object、Promise
判断是否是promise时，只是判断指定的参数是否是对象，并且其具有then、catch方法

```js
export const objectToString = Object.prototype.toString
export const toTypeString = (value: unknown): string =>
  objectToString.call(value)

export const toRawType = (value: unknown): string => {
  return toTypeString(value).slice(8, -1)
}
```
toTypeString返回类似"[object string] ..."这类的字符串
toRawType返回的是表示具体类型的字符串， 比如Stirng等

```js
export const isPlainObject = (val: unknown): val is object =>
  toTypeString(val) === '[object Object]'
```
判断参数val是不是一个纯对象，判断的依据是：
    对其调用toTypeStirng，返回的字符串是“[object Object]“
    
```js
export const isReservedProp = /*#__PURE__*/ makeMap(
  'key,ref,' +
    'onVnodeBeforeMount,onVnodeMounted,' +
    'onVnodeBeforeUpdate,onVnodeUpdated,' +
    'onVnodeBeforeUnmount,onVnodeUnmounted'
)
```
通过[makeMap](./makeMap.md)方法的调用将”保留字段“放进内存，并返回一个方法，用于判断指定参数是不是这些保留字段之一

```js
const cacheStringFunction = <T extends (str: string) => string>(fn: T): T => {
  const cache: Record<string, string> = Object.create(null)
  return ((str: string) => {
    const hit = cache[str]
    return hit || (cache[str] = fn(str))
  }) as any
}
```
高阶函数，接收一个函数fn，返回一个函数rFn：
   
   * 若参数str已经被存进cache中，rFn返回的就是cache[str]
   * 否则，先将str通过fn加工处理返回一个新字符串，然后把这个新字符串存储在cache[str]中，并返回

```js
const camelizeRE = /-(\w)/g
export const camelize = cacheStringFunction(
  (str: string): string => {
    return str.replace(camelizeRE, (_, c) => (c ? c.toUpperCase() : ''))
  }
)
```
这个”参数函数“里的作用是将指定的str，若他是中划线命名的方式，则将其该成驼峰命名。比如 ‘user-data’则，返回 userData

```js
const hyphenateRE = /\B([A-Z])/g
export const hyphenate = cacheStringFunction(
  (str: string): string => {
    return str.replace(hyphenateRE, '-$1').toLowerCase()
  }
)
```
这个”参数函数“的作用是将驼峰命名改成中划线命名，比如userData， 则是user-data

```js
export const capitalize = cacheStringFunction(
  (str: string): string => {
    return str.charAt(0).toUpperCase() + str.slice(1)
  }
)
```
这个参数函数用于将指定的字符串首字母大写

```js
// compare whether a value has changed, accounting for NaN.
export const hasChanged = (value: any, oldValue: any): boolean =>
  value !== oldValue && (value === value || oldValue === oldValue)
```
* 当value与oldValue均是NaN的时候，返回false
* 只有一个是NaN的时候，返回ture
* 当都不是NaN的时候，如果value !== oldValue则返回true，否则返回false


```js
// for converting {{ interpolation }} values to displayed strings.
export const toDisplayString = (val: unknown): string => {
  return val == null
    ? ''
    : isArray(val) || (isPlainObject(val) && val.toString === objectToString)
      ? JSON.stringify(val, null, 2)
      : String(val)
}
```
将参数val字符串化

* 若val=== null
    * 直接返回 '';
* 否则，如果val的类型是数组，或者它是一个纯对象，并且这个对象的toString属性指向Object.prototype.toString，则通过JSON.stringfy将其转成字符串
* 否则直接通过String()将其转成字符串。

