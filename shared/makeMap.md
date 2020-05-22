# packages/shared/src/makeMap.ts

```js
// Make a map and return a function for checking if a key
// is in that map.
//
// IMPORTANT: all calls of this function must be prefixed with /*#__PURE__*/
// So that rollup can tree-shake them if necessary.
export function makeMap(
  str: string,
  expectsLowerCase?: boolean
): (key: string) => boolean {
  const map: Record<string, boolean> = Object.create(null)
  const list: Array<string> = str.split(',')
  for (let i = 0; i < list.length; i++) {
    map[list[i]] = true
  }
  return expectsLowerCase ? val => !!map[val.toLowerCase()] : val => !!map[val]
}
```

首先根据参数str，建立映射表，比如 ‘str1,str2,str3’,得到 {str1: true, str2: true, str3: true}

然后根据expectsLowerCase去返回一个函数，用于判断指定的参数是不是map上的属性。

expectsLowerCase 用于指定是不是将要查询的val在判断”是否存在“前转换成小写


