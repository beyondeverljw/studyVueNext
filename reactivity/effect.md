# packages/reactivity/src/effect.ts

```js
import { TrackOpTypes, TriggerOpTypes } from './operations'
import { EMPTY_OBJ, extend, isArray } from '@vue/shared'
```
```js
// ...省略ts代码
```
```js
const targetMap = new WeakMap<any, KeyToDepMap>()
```
作用：用于存放”被观察对象“=>多个”被观察对象的键（键可以不是对象上的属性）“=>一组对应的effect
```js
// ...省略ts代码
```
```js
const effectStack: ReactiveEffect[] = []
```
作用：用于控制执行effect的栈（原理类似js的函数的调用栈）
```js
export let activeEffect: ReactiveEffect | undefined
```
作用：用于存放当前执行的effect
```js
export const ITERATE_KEY = Symbol('iterate')
```
作用：用于”可迭代数据“对应的依赖中的键
```js
export function isEffect(fn: any): fn is ReactiveEffect {
  return fn != null && fn._isEffect === true
}
```
作用：用于判断参数fn是否是一个ReactiveEffect
* 如果fn拥有一个值为true的_isEffect属性，那么fn就是一个ReactiveEffect。


```js
export function effect<T = any>(
  fn: () => T,
  options: ReactiveEffectOptions = EMPTY_OBJ
): ReactiveEffect<T> {
  if (isEffect(fn)) {
    fn = fn.raw
  }
  const effect = createReactiveEffect(fn, options)
  if (!options.lazy) {
    effect()
  }
  return effect
}
```
作用: 创建一个effect，根据条件决定是否立即执行它，最终返回它
* 判断fn是不是一个effect，如果是，那么就把fn指向effect对应的那个function
* 通过fn和options参数创建一个effect
* 如果options.lazy为falsely，那么就执行这个新创建的effect
* 返回这个effect

```js
export function stop(effect: ReactiveEffect) {
  if (effect.active) {
    cleanup(effect)
    if (effect.options.onStop) {
      effect.options.onStop()
    }
    effect.active = false
  }
}
```
作用: 停止一个ReactiveEffect的相关操作,如果effect.active为truth,则：
* 调用cleanup
* 如果effect的options中指定了onStop句柄，则调用其
* 将effect的active设置为false

```js
function createReactiveEffect<T = any>(
  fn: () => T,
  options: ReactiveEffectOptions
): ReactiveEffect<T> {
  const effect = function reactiveEffect(...args: unknown[]): unknown {
    return run(effect, fn, args)
  } as ReactiveEffect
  effect._isEffect = true
  effect.active = true
  effect.raw = fn
  effect.deps = []
  effect.options = options
  return effect
}
```
作用：工厂函数，创建一个ReactiveEffect并返回，当被返回的reactiveEffect在外边被调用的时候，会返回调用run的结果。利用闭包将fn与effect关联了起来。

```js
function run(effect: ReactiveEffect, fn: Function, args: unknown[]): unknown {
  if (!effect.active) {
    return fn(...args)
  }
  if (!effectStack.includes(effect)) {
    cleanup(effect)
    try {
      enableTracking()
      effectStack.push(effect)
      activeEffect = effect
      return fn(...args)
    } finally {
      effectStack.pop()
      resetTracking()
      activeEffect = effectStack[effectStack.length - 1]
    }
  }
}
```

作用：执行fn，同时处理、影响effectStack，activeEffect, trackStack， shouldTrack的外部状态
* 如果参数effect的active为falsely，那么直接返回fn的调用结果。
* 否则，如果在effectStack中不包括参数所制定的effect，那么：
    * 首先调用cleanup清除effect相关的依赖
    * 调用enableTracking()，此方法内会去操作trackStack，shouldTrack
    * 将参数effect放入effectStack栈
    * 将参数放入activeEffect
    * 调用参数fn指向的方法（此方法内可能会调用track方法）
    * 从effectStack中弹出栈顶的一项，即在try语句块中放入的那一个effect
    * 调用resetTracking()，此方法内会去操作trackStack，shouldTrack
    * 将当前effectStack栈顶的一项覆盖到activeEffect
    * 执行return返回之前调用fn的结果
* 此处的实现思想与函数的执行机制是类似的，压栈 => 执行 => 出栈

```js
function cleanup(effect: ReactiveEffect) {
  const { deps } = effect
  if (deps.length) {
    for (let i = 0; i < deps.length; i++) {
      deps[i].delete(effect)
    }
    deps.length = 0
  }
}
```
作用：清除effect相关的依赖
* 获取参数effect的依赖（deps是一个数组，该数组的元素是set）
* 如果deps.length大于0，那么从deps数组中每一个元素(set)中删除当前effect
* 清空effect的deps

```js
let shouldTrack = true
const trackStack: boolean[] = []
```
```js
export function pauseTracking() {
  trackStack.push(shouldTrack)
  shouldTrack = false
}
```
作用：将当前shouldTrack加入到trackStack中，然后将当前shouldTrack设置为false
```js
export function enableTracking() {
  trackStack.push(shouldTrack)
  shouldTrack = true
}
```
作用：将当前shouldTrack加入到trackStack中，然后将当前shouldTrack设置为true

```js
export function resetTracking() {
  const last = trackStack.pop()
  shouldTrack = last === undefined ? true : last
}
```
作用：从trackStack栈中弹出最后一项，如果最后一项为undefined(即当前栈已经是空的)，那么将shouldTrack设置为true，否则就是栈顶一项的值

```js
export function track(target: object, type: TrackOpTypes, key: unknown) {
  if (!shouldTrack || activeEffect === undefined) {
    return
  }
  let depsMap = targetMap.get(target)
  if (depsMap === void 0) {
    targetMap.set(target, (depsMap = new Map()))
  }
  let dep = depsMap.get(key)
  if (dep === void 0) {
    depsMap.set(key, (dep = new Set()))
  }
  if (!dep.has(activeEffect)) {
    dep.add(activeEffect)
    activeEffect.deps.push(dep)
    if (__DEV__ && activeEffect.options.onTrack) {
      activeEffect.options.onTrack({
        effect: activeEffect,
        target,
        type,
        key
      })
    }
  }
}
```
作用：
* 如果shouldTrack的值为falsely，或者activeEffect的值为undefined，那么直接退出
* 从targetMap中获取参数target对应的依赖（一个Map类型数据，键是要追踪的属性标识，值是一个effect的集合），存入局部变量depsMap中；
    * 备注: 被追踪的对象（target）与键、effect的关系：
        * 一个target对应一个Map,这个Map里包含一组键，而每个键对应一个effect的集合
        * target与键是一对多的关系，键与effect也是一对多的关系，但是从这个函数里来看，并不能说明键是target自身或其原型链上的属性，只能说明target与这个键有这对应关系（一对多）
    * 如果depsMap的值为undefined（即当前target还没有设置过对应的依赖），那么
        1. 为depsMap赋值为一个新的Map实例
        2. 在targetMap中设置target的依赖Map
* 从依赖Map(即depsMap)中获取key参数对应的effect集合，存入局部变量dep中
    * 如果dep的值为undefined (即当前依赖Map中还未设置过key对应的effect)，那么
        1. 为dep赋值为一个新的Set实例
        2. 在当前依赖Map中设置key对应的effect集合
    * 如果dep（即参数key对应的effect集合）中没有当前activeEffect，那么
        * 将activeEffect添加到dep中；
        * 将dep,添加到activeEffect.deps中
        * 如果是开发环境，且当前activeEffect.options中指定了onTrack方法，那么此时调用它
        
```js
export function trigger(
  target: object,
  type: TriggerOpTypes,
  key?: unknown,
  extraInfo?: DebuggerEventExtraInfo
) {
  const depsMap = targetMap.get(target)
  if (depsMap === void 0) {
    // never been tracked
    return
  }
  const effects = new Set<ReactiveEffect>()
  const computedRunners = new Set<ReactiveEffect>()
  if (type === TriggerOpTypes.CLEAR) {
    // collection being cleared, trigger all effects for target
    depsMap.forEach(dep => {
      addRunners(effects, computedRunners, dep)
    })
  } else {
    // schedule runs for SET | ADD | DELETE
    if (key !== void 0) {
      addRunners(effects, computedRunners, depsMap.get(key))
    }
    // also run for iteration key on ADD | DELETE | Map.SET
    if (
      type === TriggerOpTypes.ADD ||
      type === TriggerOpTypes.DELETE ||
      (type === TriggerOpTypes.SET && target instanceof Map)
    ) {
      const iterationKey = isArray(target) ? 'length' : ITERATE_KEY
      addRunners(effects, computedRunners, depsMap.get(iterationKey))
    }
  }
  const run = (effect: ReactiveEffect) => {
    scheduleRun(effect, target, type, key, extraInfo)
  }
  // Important: computed effects must be run first so that computed getters
  // can be invalidated before any normal effects that depend on them are run.
  computedRunners.forEach(run)
  effects.forEach(run)
}
```
作用：将target对应的依赖集根据条件分配到不同的队列，控制执行的先后顺序，
* 首先，获取target,对应的依赖Map,存入局部变量depsMap中。
    * 如果depsMap的值为undefined，那么直接退出函数
* 声明两个Set实例，一个是effects,一个是computedRunners，用于存放effect
* 如果type参数是TriggerOpTypes.CLEAR,则：
    * 将depsMap中所有的依赖（多个键，每个键对应多个effect）通过addRunners()方法添加到effects集合，或computedRunners集合
* 否则，
    * 如果指定了key参数(即key!==undefined)，则只将这个key对应的effect集合通过addRunners()方法添加到effects集合，或computedRunners集合
    * 如果type参数为TriggerOpTypes.ADD、或者TriggerOpTypes.DELETE、或者TriggerOpTypes.SET且target是Map的实例，则：
        * 确定iterationKey，如果target的类型是Array,则iterationKey的值是“length”,否则是一个ITERATE_KEY（一个Symbol标识）
        * 从depsMap中获取iterationKey对应的依赖，通过addRunners添加到effects，或者computedRunners中
* 声明run函数，它的内部会调用scheduleRun去具体的执行effect。
* 先遍历computedRunners中的effect，执行。
* 后遍历effects中的effect，执行。

```js
function addRunners(
  effects: Set<ReactiveEffect>,
  computedRunners: Set<ReactiveEffect>,
  effectsToAdd: Set<ReactiveEffect> | undefined
) {
  if (effectsToAdd !== void 0) {
    effectsToAdd.forEach(effect => {
      if (effect.options.computed) {
        computedRunners.add(effect)
      } else {
        effects.add(effect)
      }
    })
  }
}
```
作用：将effectsToAdd中的effect根据条件添加到effects集合或者computedRunners集合中，当effectsToAdd不为undefined时，遍历effectsToAdd集合：
* 如果当前遍历到的effect的options中指定了computed,则将当前effect添加到computedRunners中
* 否则，添加到effects中
 
```js
function scheduleRun(
  effect: ReactiveEffect,
  target: object,
  type: TriggerOpTypes,
  key: unknown,
  extraInfo?: DebuggerEventExtraInfo
) {
  if (__DEV__ && effect.options.onTrigger) {
    const event: DebuggerEvent = {
      effect,
      target,
      key,
      type
    }
    effect.options.onTrigger(extraInfo ? extend(event, extraInfo) : event)
  }
  if (effect.options.scheduler !== void 0) {
    effect.options.scheduler(effect)
  } else {
    effect()
  }
}
```
作用：根据条件调用effect
* 如果effect的options里指定了scheduler，那么将effect作为参数，调用scheduler
* 否则直接调用effect
* 如果当前是开发环境，
    1. 如果参数提供了extraInfo，那么将extraInfo合并到event上,否则就是event
    2. 将1中得到的对象作为参数调用effect的options中的onTrigger()