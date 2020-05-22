# packages/runtime-core/src/apiCreateApp.ts

```js
import {
  Component,
  Data,
  validateComponentName,
  PublicAPIComponent
} from './component'
```
```js
import { ComponentOptions } from './apiOptions'
import { ComponentPublicInstance } from './componentProxy'
import { Directive, validateDirectiveName } from './directives'
import { RootRenderFunction } from './renderer'
import { InjectionKey } from './apiInject'
import { isFunction, NO, isObject } from '@vue/shared'
import { warn } from './warning'
import { createVNode, cloneVNode, VNode } from './vnode'
```
```js
export interface App<HostElement = any> {
  config: AppConfig
  use(plugin: Plugin, ...options: any[]): this
  mixin(mixin: ComponentOptions): this
  component(name: string): Component | undefined
  component(name: string, component: Component): this
  directive(name: string): Directive | undefined
  directive(name: string, directive: Directive): this
  mount(
    rootContainer: HostElement | string,
    isHydrate?: boolean
  ): ComponentPublicInstance
  unmount(rootContainer: HostElement | string): void
  provide<T>(key: InjectionKey<T> | string, value: T): this

  // internal. We need to expose these for the server-renderer
  _component: Component
  _props: Data | null
  _container: HostElement | null
  _context: AppContext
}
```
```js
export interface AppConfig {
  devtools: boolean
  performance: boolean
  readonly isNativeTag?: (tag: string) => boolean
  isCustomElement?: (tag: string) => boolean
  errorHandler?: (
    err: Error,
    instance: ComponentPublicInstance | null,
    info: string
  ) => void
  warnHandler?: (
    msg: string,
    instance: ComponentPublicInstance | null,
    trace: string
  ) => void
}
```
```js
export interface AppContext {
  config: AppConfig
  mixins: ComponentOptions[]
  components: Record<string, Component>
  directives: Record<string, Directive>
  provides: Record<string | symbol, any>
  reload?: () => void // HMR only
}
```
```js
type PluginInstallFunction = (app: App, ...options: any[]) => any
```
```js
export type Plugin =
  | PluginInstallFunction & { install?: PluginInstallFunction }
  | {
      install: PluginInstallFunction
    }
```
```js
export function createAppContext(): AppContext {
  return {
    config: {
      devtools: true,
      performance: false,
      isNativeTag: NO,
      isCustomElement: NO,
      errorHandler: undefined,
      warnHandler: undefined
    },
    mixins: [],
    components: {},
    directives: {},
    provides: {}
  }
}
```
作用： 一个工厂函数，返回一个具有初始值的appContext
```js
export type CreateAppFunction<HostElement> = (
  rootComponent: PublicAPIComponent,
  rootProps?: Data | null
) => App<HostElement>
```
```js
export function createAppAPI<HostNode, HostElement>(
  render: RootRenderFunction<HostNode, HostElement>,
  hydrate?: (vnode: VNode, container: Element) => void
): CreateAppFunction<HostElement> {
  return function createApp(rootComponent: Component, rootProps = null) {
    if (rootProps != null && !isObject(rootProps)) {
      __DEV__ && warn(`root props passed to app.mount() must be an object.`)
      rootProps = null
    }

    const context = createAppContext()
    const installedPlugins = new Set()

    let isMounted = false

    const app: App = {
      _component: rootComponent,
      _props: rootProps,
      _container: null,
      _context: context,

      get config() {
        return context.config
      },

      set config(v) {
        if (__DEV__) {
          warn(
            `app.config cannot be replaced. Modify individual options instead.`
          )
        }
      },

      use(plugin: Plugin, ...options: any[]) {
        if (installedPlugins.has(plugin)) {
          __DEV__ && warn(`Plugin has already been applied to target app.`)
        } else if (plugin && isFunction(plugin.install)) {
          installedPlugins.add(plugin)
          plugin.install(app, ...options)
        } else if (isFunction(plugin)) {
          installedPlugins.add(plugin)
          plugin(app, ...options)
        } else if (__DEV__) {
          warn(
            `A plugin must either be a function or an object with an "install" ` +
              `function.`
          )
        }
        return app
      },

      mixin(mixin: ComponentOptions) {
        if (__FEATURE_OPTIONS__) {
          if (!context.mixins.includes(mixin)) {
            context.mixins.push(mixin)
          } else if (__DEV__) {
            warn(
              'Mixin has already been applied to target app' +
                (mixin.name ? `: ${mixin.name}` : '')
            )
          }
        } else if (__DEV__) {
          warn('Mixins are only available in builds supporting Options API')
        }
        return app
      },

      component(name: string, component?: PublicAPIComponent): any {
        if (__DEV__) {
          validateComponentName(name, context.config)
        }
        if (!component) {
          return context.components[name]
        }
        if (__DEV__ && context.components[name]) {
          warn(`Component "${name}" has already been registered in target app.`)
        }
        context.components[name] = component as Component
        return app
      },

      directive(name: string, directive?: Directive) {
        if (__DEV__) {
          validateDirectiveName(name)
        }

        if (!directive) {
          return context.directives[name] as any
        }
        if (__DEV__ && context.directives[name]) {
          warn(`Directive "${name}" has already been registered in target app.`)
        }
        context.directives[name] = directive
        return app
      },

      mount(rootContainer: HostElement, isHydrate?: boolean): any {
        if (!isMounted) {
          const vnode = createVNode(rootComponent, rootProps)
          // store app context on the root VNode.
          // this will be set on the root instance on initial mount.
          vnode.appContext = context

          // HMR root reload
          if (__BUNDLER__ && __DEV__) {
            context.reload = () => {
              render(cloneVNode(vnode), rootContainer)
            }
          }

          if (isHydrate && hydrate) {
            hydrate(vnode, rootContainer as any)
          } else {
            render(vnode, rootContainer)
          }
          isMounted = true
          app._container = rootContainer
          return vnode.component!.proxy
        } else if (__DEV__) {
          warn(
            `App has already been mounted. Create a new app instance instead.`
          )
        }
      },

      unmount() {
        if (isMounted) {
          render(null, app._container)
        } else if (__DEV__) {
          warn(`Cannot unmount an app that is not mounted.`)
        }
      },

      provide(key, value) {
        if (__DEV__ && key in context.provides) {
          warn(
            `App already provides property with key "${key}". ` +
              `It will be overwritten with the new value.`
          )
        }
        // TypeScript doesn't allow symbols as index type
        // https://github.com/Microsoft/TypeScript/issues/24587
        context.provides[key as string] = value

        return app
      }
    }

    return app
  }
}
```
作用：返回一个工厂函数，这个工厂函数用来创建app，当这个工厂函数在外部被调用时，过程如下：
* 判断rootProps是不是一个对象，如果其不是Null,且不是Object类型，则将其设置为null,如果当前是开发环境，会进行信息提示：
* 通过createAppContext(),得到一个包含appContext对象（app的上下文对象）
* 声明一个app对象，并将这个对象返回（每个应用对应一个独立的app对象），关于这个app对象的各属性的说明如下：
    * config: 当访问app.config时：
        * get时，实际上时返回app上下文（即context）的config
        * set时，如果在开发环境，会进行信息提示。即app.config对于外部来说是readonly的
    * use: 用于提供vue中的插件的操作
        * 插件的数据类型可以是一个function,也可以是一个包含install方法的对象
        * 当调用app.use(plugin, ...options)时，过程如下：
            * 如果installedPlugins集合中已经拥有参数指定的plugin，在开发环境会进行信息提示
            * 如果plugin拥有install方法，则将plugin添加到installedPlugins集合中，然后以当前app和options为参数，调用plugin(app, ...options)
            * 如果plugin是一个函数，则将这个函数添加到installedPlugins集合中，并条用plugin这个函数
            * 其他情况下，如果是在开发环境中，则进行相应的信息提示
            * 最终返回当前的app
        * 备注：由于function本质上是对象（函数对象），当一个function具有install方法时，会优先选择install方法
    * mixin: 用于混入，仅在__FEATURE_OPTIONS__为truth时可用
        * 若当前appContext的mixins数组中没有参数指定的mixin,则将mixin添加到当前app上下文的mixins数组中。
        * 最终返回当前app
    * component: 用于访问、添加组件。当调用app.component()时：
        * 如果是在开发环境，会先验证name是否符合vue的规则,[validateComponentName](./component.md)
        * 如果参数component为undefined，则当前是"get"操作，会返回当前app上下文中的name参数对应的组件
        * 否则，就是“set”操作，用name做键，component做值，设置到当前app上下文的components属性上
        * 最终返回当前app
    * directive: 用于访问，添加指令。当调用app.directive()时：
        * 如果是在开发环境，会先验证name是否符合vue的规则[validateDirectiveName](./directives.md)
        * 如果参数directive为undefined,则当前是“get”操作，会返回当前app上下文中的name参数对应的指令
        * 否则，就是“set”操作，name做键，directive做值，设置到当前上下文中的directives上
        * 最终返回当前app
    * mount: 
    * unmount:
    * provide: 将指定的key, value设置到当前上下文对象的provides上。