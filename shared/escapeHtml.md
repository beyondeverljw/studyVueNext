# packages/shared/src/escapeHtml.ts

```js
const escapeRE = /["'&<>]/

export function escapeHtml(string: unknown) {
  const str = '' + string
  const match = escapeRE.exec(str)

  if (!match) {
    return str
  }

  let html = ''
  let escaped: string
  let index: number
  let lastIndex = 0
  for (index = match.index; index < str.length; index++) {
    switch (str.charCodeAt(index)) {
      case 34: // "
        escaped = '&quot;'
        break
      case 38: // &
        escaped = '&amp;'
        break
      case 39: // '
        escaped = '&#39;'
        break
      case 60: // <
        escaped = '&lt;'
        break
      case 62: // >
        escaped = '&gt;'
        break
      default:
        continue
    }

    if (lastIndex !== index) {
      html += str.substring(lastIndex, index)
    }

    lastIndex = index + 1
    html += escaped
  }

  return lastIndex !== index ? html + str.substring(lastIndex, index) : html
}
```

将参数string中的html特殊字符进行转义， 即将‘“’， ‘‘’， ’&‘， ’<‘, '>'分别替换为 '\&quot;', '\&amp;', '\&#39;', '\&lt;', '\&gt;'

* 首先通过正则找到第一次匹配的地方
* 截取当前匹配位置之前的字符串，即lastIndex到index
* 将lastIndex标记为当前匹配位置的后一个位置
* 将之前截取的字符串，加上当前匹配字符串的转义
* 当最后一个匹配位置为字符串的末尾时，返回已经拼装好的字符串（html）
* 否则再增加一次截取操作（当前位置之后的）

换一种实现方式可以这样：

```js
const escapeRE = /["'&<>]/g
function escapeHtml(string) {
    return string.replace(escapeRE, function(match){
        let code = match.charCodeAt(0);
        switch(code){
            case 34:
                return '&quot;'
            case 38:
                return '&amp;'
            case 39:
                return '&#39;'
            case 60: 
                return '&lt;'
            case 62:
                return '&gt;'
        }
    })
}
```


