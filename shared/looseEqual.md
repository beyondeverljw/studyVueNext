# packages/shared/src/looseEqual.ts

```js
import { isObject, isArray } from './'

export function looseEqual(a: any, b: any): boolean {
  if (a === b) return true
  const isObjectA = isObject(a)
  const isObjectB = isObject(b)
  if (isObjectA && isObjectB) {
    try {
      const isArrayA = isArray(a)
      const isArrayB = isArray(b)
      if (isArrayA && isArrayB) {
        return (
          a.length === b.length &&
          a.every((e: any, i: any) => looseEqual(e, b[i]))
        )
      } else if (a instanceof Date && b instanceof Date) {
        return a.getTime() === b.getTime()
      } else if (!isArrayA && !isArrayB) {
        const keysA = Object.keys(a)
        const keysB = Object.keys(b)
        return (
          keysA.length === keysB.length &&
          keysA.every(key => looseEqual(a[key], b[key]))
        )
      } else {
        /* istanbul ignore next */
        return false
      }
    } catch (e) {
      /* istanbul ignore next */
      return false
    }
  } else if (!isObjectA && !isObjectB) {
    return String(a) === String(b)
  } else {
    return false
  }
}

export function looseIndexOf(arr: any[], val: any): number {
  return arr.findIndex(item => looseEqual(item, val))
}
```

比较两个值是否相等

规则：

* 若a与b完全相等，返回true
* 若a与b的类型全是Object时：
    * 若a与b的类型均是Array时：
        1. 如果a与b的长度相同， 则递归调用looseEqual
        2. 否则返回false
    * 若a与b的类型均是Date时，分别通过getTime()转成毫秒, 返回两个”毫秒“是否相等
    * 若a与b均不是Array, Date类型时
        1. 分别通过Object.keys获取实例域的属性名
        2. 若a与b实例域的属性名的数量不相等，则返回false，否则继续递归比较a与b的同名属性的值是否相等
    * 当以上判断的条件均不满足时，返回false       
* 若a与b的类型全不是Object时：
    * 将a于b分别字符串化，再比较。
        * 这样可以避免NaN!==NaN的情况
        * 也可以比较连个匿名函数是否相同
        * 等等。。。
* 返回false;

