# packages/runtime-core/src/apiOptions.ts

```js
import {
  ComponentInternalInstance,
  Data,
  SetupContext,
  RenderFunction,
  SFCInternalOptions,
  PublicAPIComponent
} from './component'
```
```js
import {
  isFunction,
  extend,
  isString,
  isObject,
  isArray,
  EMPTY_OBJ,
  NOOP
} from '@vue/shared'
```
```js
import { computed } from './apiComputed'
import { watch, WatchOptions, WatchCallback } from './apiWatch'
import { provide, inject } from './apiInject'
```
```js
import {
  onBeforeMount,
  onMounted,
  onBeforeUpdate,
  onUpdated,
  onErrorCaptured,
  onRenderTracked,
  onBeforeUnmount,
  onUnmounted,
  onActivated,
  onDeactivated,
  onRenderTriggered,
  DebuggerHook,
  ErrorCapturedHook
} from './apiLifecycle'
```
```js
import {
  reactive,
  ComputedGetter,
  WritableComputedOptions
} from '@vue/reactivity'
```
```js
import { ComponentObjectPropsOptions, ExtractPropTypes } from './componentProps'
import { Directive } from './directives'
import { ComponentPublicInstance } from './componentProxy'
import { warn } from './warning'
```
```js
//... 省略一些ts代码
```
```js
function createDuplicateChecker() {
  const cache = Object.create(null)
  return (type: OptionTypes, key: string) => {
    if (cache[key]) {
      warn(`${type} property "${key}" is already defined in ${cache[key]}.`)
    } else {
      cache[key] = type
    }
  }
}
```

作用：利用闭包原理进行缓存，当在外部多次调用createDuplicateChecker返回的函数时，这些内部函数会共享一个cache。即key如果已经在cache中设置过，会有相应的信息提示，否则，就以key：type设置到cache上。


```js
export function applyOptions(
  instance: ComponentInternalInstance,
  options: ComponentOptions,
  asMixin: boolean = false
) {
  const ctx = instance.proxy!
  const {
    // composition
    mixins,
    extends: extendsOptions,
    // state
    props: propsOptions,
    data: dataOptions,
    computed: computedOptions,
    methods,
    watch: watchOptions,
    provide: provideOptions,
    inject: injectOptions,
    // assets
    components,
    directives,
    // lifecycle
    beforeMount,
    mounted,
    beforeUpdate,
    updated,
    activated,
    deactivated,
    beforeUnmount,
    unmounted,
    renderTracked,
    renderTriggered,
    errorCaptured
  } = options
  
  const renderContext =
    instance.renderContext === EMPTY_OBJ
      ? (instance.renderContext = {})
      : instance.renderContext

  const globalMixins = instance.appContext.mixins
  // call it only during dev
  const checkDuplicateProperties = __DEV__ ? createDuplicateChecker() : null
  // applyOptions is called non-as-mixin once per instance
  if (!asMixin) {
    callSyncHook('beforeCreate', options, ctx, globalMixins)
    // global mixins are applied first
    applyMixins(instance, globalMixins)
  }
  // extending a base component...
  if (extendsOptions) {
    applyOptions(instance, extendsOptions, true)
  }
  // local mixins
  if (mixins) {
    applyMixins(instance, mixins)
  }

  if (__DEV__ && propsOptions) {
    for (const key in propsOptions) {
      checkDuplicateProperties!(OptionTypes.PROPS, key)
    }
  }

  // state options
  if (dataOptions) {
    const data = isFunction(dataOptions) ? dataOptions.call(ctx) : dataOptions
    if (!isObject(data)) {
      __DEV__ && warn(`data() should return an object.`)
    } else if (instance.data === EMPTY_OBJ) {
      if (__DEV__) {
        for (const key in data) {
          checkDuplicateProperties!(OptionTypes.DATA, key)
        }
      }
      instance.data = reactive(data)
    } else {
      // existing data: this is a mixin or extends.
      extend(instance.data, data)
    }
  }

  if (computedOptions) {
    for (const key in computedOptions) {
      const opt = (computedOptions as ComputedOptions)[key]

      __DEV__ && checkDuplicateProperties!(OptionTypes.COMPUTED, key)

      if (isFunction(opt)) {
        renderContext[key] = computed(opt.bind(ctx, ctx))
      } else {
        const { get, set } = opt
        if (isFunction(get)) {
          renderContext[key] = computed({
            get: get.bind(ctx, ctx),
            set: isFunction(set)
              ? set.bind(ctx)
              : __DEV__
                ? () => {
                    warn(
                      `Computed property "${key}" was assigned to but it has no setter.`
                    )
                  }
                : NOOP
          })
        } else if (__DEV__) {
          warn(`Computed property "${key}" has no getter.`)
        }
      }
    }
  }

  if (methods) {
    for (const key in methods) {
      const methodHandler = (methods as MethodOptions)[key]
      if (isFunction(methodHandler)) {
        __DEV__ && checkDuplicateProperties!(OptionTypes.METHODS, key)
        renderContext[key] = methodHandler.bind(ctx)
      } else if (__DEV__) {
        warn(
          `Method "${key}" has type "${typeof methodHandler}" in the component definition. ` +
            `Did you reference the function correctly?`
        )
      }
    }
  }

  if (watchOptions) {
    for (const key in watchOptions) {
      createWatcher(watchOptions[key], renderContext, ctx, key)
    }
  }

  if (provideOptions) {
    const provides = isFunction(provideOptions)
      ? provideOptions.call(ctx)
      : provideOptions
    for (const key in provides) {
      provide(key, provides[key])
    }
  }

  if (injectOptions) {
    if (isArray(injectOptions)) {
      for (let i = 0; i < injectOptions.length; i++) {
        const key = injectOptions[i]
        __DEV__ && checkDuplicateProperties!(OptionTypes.INJECT, key)
        renderContext[key] = inject(key)
      }
    } else {
      for (const key in injectOptions) {
        __DEV__ && checkDuplicateProperties!(OptionTypes.INJECT, key)
        const opt = injectOptions[key]
        if (isObject(opt)) {
          renderContext[key] = inject(opt.from, opt.default)
        } else {
          renderContext[key] = inject(opt)
        }
      }
    }
  }

  // asset options
  if (components) {
    extend(instance.components, components)
  }
  if (directives) {
    extend(instance.directives, directives)
  }

  // lifecycle options
  if (!asMixin) {
    callSyncHook('created', options, ctx, globalMixins)
  }
  if (beforeMount) {
    onBeforeMount(beforeMount.bind(ctx))
  }
  if (mounted) {
    onMounted(mounted.bind(ctx))
  }
  if (beforeUpdate) {
    onBeforeUpdate(beforeUpdate.bind(ctx))
  }
  if (updated) {
    onUpdated(updated.bind(ctx))
  }
  if (activated) {
    onActivated(activated.bind(ctx))
  }
  if (deactivated) {
    onDeactivated(deactivated.bind(ctx))
  }
  if (errorCaptured) {
    onErrorCaptured(errorCaptured.bind(ctx))
  }
  if (renderTracked) {
    onRenderTracked(renderTracked.bind(ctx))
  }
  if (renderTriggered) {
    onRenderTriggered(renderTriggered.bind(ctx))
  }
  if (beforeUnmount) {
    onBeforeUnmount(beforeUnmount.bind(ctx))
  }
  if (unmounted) {
    onUnmounted(unmounted.bind(ctx))
  }
}
```
```js
function callSyncHook(
  name: 'beforeCreate' | 'created',
  options: ComponentOptions,
  ctx: ComponentPublicInstance,
  globalMixins: ComponentOptions[]
) {
  callHookFromMixins(name, globalMixins, ctx)
  const baseHook = options.extends && options.extends[name]
  if (baseHook) {
    baseHook.call(ctx)
  }
  const mixins = options.mixins
  if (mixins) {
    callHookFromMixins(name, mixins, ctx)
  }
  const selfHook = options[name]
  if (selfHook) {
    selfHook.call(ctx)
  }
}
```
```js
function callHookFromMixins(
  name: 'beforeCreate' | 'created',
  mixins: ComponentOptions[],
  ctx: ComponentPublicInstance
) {
  for (let i = 0; i < mixins.length; i++) {
    const fn = mixins[i][name]
    if (fn) {
      fn.call(ctx)
    }
  }
}
```
```js
function applyMixins(
  instance: ComponentInternalInstance,
  mixins: ComponentOptions[]
) {
  for (let i = 0; i < mixins.length; i++) {
    applyOptions(instance, mixins[i], true)
  }
}
```
```js
function createWatcher(
  raw: ComponentWatchOptionItem,
  renderContext: Data,
  ctx: ComponentPublicInstance,
  key: string
) {
  const getter = () => (ctx as Data)[key]
  if (isString(raw)) {
    const handler = renderContext[raw]
    if (isFunction(handler)) {
      watch(getter, handler as WatchCallback)
    } else if (__DEV__) {
      warn(`Invalid watch handler specified by key "${raw}"`, handler)
    }
  } else if (isFunction(raw)) {
    watch(getter, raw.bind(ctx))
  } else if (isObject(raw)) {
    if (isArray(raw)) {
      raw.forEach(r => createWatcher(r, renderContext, ctx, key))
    } else {
      watch(getter, raw.handler.bind(ctx), raw)
    }
  } else if (__DEV__) {
    warn(`Invalid watch option: "${key}"`)
  }
}
```