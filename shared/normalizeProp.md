# packages/shared/src/normalizeProp.ts

```js
import { isArray, isString, isObject } from './'

export function normalizeStyle(
  value: unknown
): Record<string, string | number> | undefined {
  if (isArray(value)) {
    const res: Record<string, string | number> = {}
    for (let i = 0; i < value.length; i++) {
      const normalized = normalizeStyle(value[i])
      if (normalized) {
        for (const key in normalized) {
          res[key] = normalized[key]
        }
      }
    }
    return res
  } else if (isObject(value)) {
    return value
  }
}
```
* 该方法会处理value的类型为Array和Object的情况，其他类型会被忽略
* 数组可以不是扁平的，可以嵌套
* 当value的类型为数组的时候，会递归调用normalizeStyle，直到value的类型为object的时候，将其（字段：值）赋值到res上(浅复制)，最终返回res对象，其他类型的值会被忽略；
* 当value的类型为Object的时候，则直接返回这个对象，而不会去探查对象的内部；


```js
export function normalizeClass(value: unknown): string {
  let res = ''
  if (isString(value)) {
    res = value
  } else if (isArray(value)) {
    for (let i = 0; i < value.length; i++) {
      res += normalizeClass(value[i]) + ' '
    }
  } else if (isObject(value)) {
    for (const name in value) {
      if (value[name]) {
        res += name + ' '
      }
    }
  }
  return res.trim()
}
```
返回以空格分隔的字符串。

* 若value的类型是String,则对其进行消除首尾空白的处理后返回其
* 若value的类型是Array，则递归调用normalizeClass，得到去除掉首尾空白的字符串
* 若value的类型是object，则将其所有可枚举属性的值字符串化(包括实例域和原型域)，然后累加到res上，最终返回去除掉首尾空白的字符串





