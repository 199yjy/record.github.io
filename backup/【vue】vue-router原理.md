

## 路由基础

* 路由作用：根据不同的路径映射到不同的视图。

* 监听[复用组件](https://router.vuejs.org/zh/guide/essentials/dynamic-matching.html#%E5%93%8D%E5%BA%94%E8%B7%AF%E7%94%B1%E5%8F%82%E6%95%B0%E7%9A%84%E5%8F%98%E5%8C%96) 路由参数变化：watch (监测变化) `$route` 对象。

  ```javascript
  const User = {
    template: '...',
    watch: {
      $route(to, from) {
        // 对路由变化作出响应...
      }
    }
  }
  ```

* 配置 `routes` 常用参数：

  * 路由路径：`path` 参数。
  * [动态路由匹配](https://router.vuejs.org/zh/guide/essentials/dynamic-matching.html#动态路由匹配)： 一个“路径参数”使用冒号 `:` 标记。 如：` { path: '/user/:id', component: User }`
  * [路由组件传参](https://router.vuejs.org/zh/guide/essentials/passing-props.html#%E8%B7%AF%E7%94%B1%E7%BB%84%E4%BB%B6%E4%BC%A0%E5%8F%82)：使用 `props` 将组件和路由解耦。
  * [路由命名](https://router.vuejs.org/zh/guide/essentials/named-routes.html#%E5%91%BD%E5%90%8D%E8%B7%AF%E7%94%B1)： `name` 参数。
  * 组件： `component` 参数。
  * [嵌套组件](https://router.vuejs.org/zh/guide/essentials/named-views.html#%E5%B5%8C%E5%A5%97%E5%91%BD%E5%90%8D%E8%A7%86%E5%9B%BE)：`children` 参数。
  * [重定向](https://router.vuejs.org/zh/guide/essentials/redirect-and-alias.html#%E9%87%8D%E5%AE%9A%E5%90%91)：`redirect` 参数。
  * [别名](https://router.vuejs.org/zh/guide/essentials/redirect-and-alias.html#%E9%87%8D%E5%AE%9A%E5%90%91)：`alias` 参数。
  * [元信息](https://router.vuejs.org/zh/guide/advanced/meta.html)：`meta` 参数。

* 内置组件：

  * 视图切换：[`<router-link>` 组件](https://router.vuejs.org/zh/api/#router-link)
  * 视图渲染：[`<router-view>` 组件](https://router.vuejs.org/zh/api/#aria-current-value)

* [导航守卫](https://router.vuejs.org/zh/guide/advanced/navigation-guards.html#%E5%85%A8%E5%B1%80%E5%89%8D%E7%BD%AE%E5%AE%88%E5%8D%AB)：

  * 全局守卫：
    * 全局前置守卫：`router.beforeEach`，导航触发时，全局前置守卫按照创建顺序调用。
    * 全局解析守卫：`router.beforeResolve`，导航被确认之前，同时在所有组件内守卫和异步路由组件被解析之后，解析守卫就被调用。
    * 全局后置守卫：`router.afterEach`。
  * 路由独享守卫：`beforeEnter`。单个路由独享的钩子函数，它是在路由配置上直接进行定义的。
  * 组件内守卫：它们是直接在路由组件内部直接进行定义的。
    * `beforeRouteEnter`
    * `beforeRouteUpdate`
    * `beforeRouteLeave`

* 导航解析流程

  * 导航被触发。
  * 在失活的组件里调用 `beforeRouteLeave` 守卫。
  * 调用全局的 `beforeEach` 守卫。
  * 在重用的组件里调用 `beforeRouteUpdate` 守卫 (2.2+)。
  * 在路由配置里调用 `beforeEnter`。
  * 解析异步路由组件。
  * 在被激活的组件里调用 `beforeRouteEnter`。
  * 调用全局的 `beforeResolve` 守卫 (2.5+)。
  * 导航被确认。
  * 调用全局的 `afterEach` 钩子。
  * 触发 DOM 更新。
  * 调用 `beforeRouteEnter` 守卫中传给 `next` 的回调函数，创建好的组件实例会作为回调函数的参数传入。

* 路由对象：

  * [**this.$router**](https://router.vuejs.org/zh/api/#router-%E5%AE%9E%E4%BE%8B%E5%B1%9E%E6%80%A7)：`router` 的 Vue 根实例。是一个全局路由对象，包含了路由跳转的方法、钩子函数等。
  * [**this.$route**](https://router.vuejs.org/zh/api/#%E8%B7%AF%E7%94%B1%E5%AF%B9%E8%B1%A1)：当前激活的[路由信息对象](https://router.vuejs.org/zh/api/#路由对象)。  每一个路由都会有一个 route 对象，是一个局部对象，包含path,params,hash,query,fullPath,matched,name等路由信息参数。
    * 里面的属性是 immutable (不可变) 的，通过 watch 监测变化 
    * 它包含了当前 URL 解析得到的信息，还有 URL 匹配到的**路由记录 (route records)**。
    * 每次成功的导航后都会产生一个新的对象。
    * 路由对象出现在多个地方:
      * 在组件内，即 `this.$route`
      * 在 `$route` 观察者回调内
      * `router.match(location)` 的返回值

* [路由懒加载](https://router.vuejs.org/zh/guide/advanced/lazy-loading.html#%E8%B7%AF%E7%94%B1%E6%87%92%E5%8A%A0%E8%BD%BD)：

  * 将异步组件定义为返回一个 Promise 的工厂函数
  * 在 Webpack 2 中，我们可以使用[动态 import ()](https://github.com/tc39/proposal-dynamic-import)语法来定义代码分块点 
  * 注释语法来提供 chunk name : `webpackChunkName`

* 路由传参：使用 `router.push()`切换路由。

  * `params` 传参： 

    * 只能使用 `name`，不能使用 `path`；如果提供了 `path`，`params` 会被忽略

      ```javascript
      // 这里的 params 不生效
      router.push({ path: '/user', params: { userId }}) // -> /user
      ```

    * 参数不会显示在路径上

    * 浏览器强制刷新参数会被清空

      ```javascript
      // 传递参数
      this.$router.push({
        name: Home,
        params: {
            test: 'hello'
        }
      })
      router.push({ name: 'user', params: { userId:123 }}) // -> /user/123
      // 接收参数
      const p = this.$route.params
      ```

  * `query` 传参：带查询参数。

    * 参数会显示在路径上，刷新不会被清空

    * `name` 可以使用 `path` 路径

      ```javascript
      // 传递参数
      this.$router.push({
        	name: Home,
        	query: {
      		test:"hello"
      	}
      })
      // 带查询参数，变成 /register?plan=private
      router.push({ path: 'register', query: { plan: 'private' }})
      // 接收参数
      const q = this.$route.query
      ```

      

## 总体流程 


## 路由对象

VueRouter 的实现是一个类，它的定义在 [src/index.js](https://github.com/vuejs/vue-router/blob/v3.0.1/src/index.js)中：

```js
export default class VueRouter {
  static install: () => void;
  static version: string;

  app: any;
  apps: Array<any>;
  ready: boolean;
  readyCbs: Array<Function>;
  options: RouterOptions;
  mode: string;
  history: HashHistory | HTML5History | AbstractHistory;
  matcher: Matcher;
  fallback: boolean;
  beforeHooks: Array<?NavigationGuard>;
  resolveHooks: Array<?NavigationGuard>;
  afterHooks: Array<?AfterNavigationHook>;

  constructor (options: RouterOptions = {}) {
    this.app = null
    this.apps = []
    this.options = options
    this.beforeHooks = []
    this.resolveHooks = []
    this.afterHooks = []
    this.matcher = createMatcher(options.routes || [], this)

    let mode = options.mode || 'hash'
    this.fallback = mode === 'history' && !supportsPushState && options.fallback !== false
    if (this.fallback) {
      mode = 'hash'
    }
    if (!inBrowser) {
      mode = 'abstract'
    }
    this.mode = mode

    switch (mode) {
      case 'history':
        this.history = new HTML5History(this, options.base)
        break
      case 'hash':
        this.history = new HashHistory(this, options.base, this.fallback)
        break
      case 'abstract':
        this.history = new AbstractHistory(this, options.base)
        break
      default:
        if (process.env.NODE_ENV !== 'production') {
          assert(false, `invalid mode: ${mode}`)
        }
    }
  }

  match (
    raw: RawLocation,
    current?: Route,
    redirectedFrom?: Location
  ): Route {
    return this.matcher.match(raw, current, redirectedFrom)
  }

  get currentRoute (): ?Route {
    return this.history && this.history.current
  }

  init (app: any) {
    process.env.NODE_ENV !== 'production' && assert(
      install.installed,
      `not installed. Make sure to call \`Vue.use(VueRouter)\` ` +
      `before creating root instance.`
    )

    this.apps.push(app)

    if (this.app) {
      return
    }

    this.app = app

    const history = this.history

    if (history instanceof HTML5History) {
      history.transitionTo(history.getCurrentLocation())
    } else if (history instanceof HashHistory) {
      const setupHashListener = () => {
        history.setupListeners()
      }
      history.transitionTo(
        history.getCurrentLocation(),
        setupHashListener,
        setupHashListener
      )
    }

    history.listen(route => {
      this.apps.forEach((app) => {
        app._route = route
      })
    })
  }

  beforeEach (fn: Function): Function {
    return registerHook(this.beforeHooks, fn)
  }

  beforeResolve (fn: Function): Function {
    return registerHook(this.resolveHooks, fn)
  }

  afterEach (fn: Function): Function {
    return registerHook(this.afterHooks, fn)
  }

  onReady (cb: Function, errorCb?: Function) {
    this.history.onReady(cb, errorCb)
  }

  onError (errorCb: Function) {
    this.history.onError(errorCb)
  }

  push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    this.history.push(location, onComplete, onAbort)
  }

  replace (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    this.history.replace(location, onComplete, onAbort)
  }

  go (n: number) {
    this.history.go(n)
  }

  back () {
    this.go(-1)
  }

  forward () {
    this.go(1)
  }

  getMatchedComponents (to?: RawLocation | Route): Array<any> {
    const route: any = to
      ? to.matched
        ? to
        : this.resolve(to).route
      : this.currentRoute
    if (!route) {
      return []
    }
    return [].concat.apply([], route.matched.map(m => {
      return Object.keys(m.components).map(key => {
        return m.components[key]
      })
    }))
  }

  resolve (
    to: RawLocation,
    current?: Route,
    append?: boolean
  ): {
    location: Location,
    route: Route,
    href: string,
    normalizedTo: Location,
    resolved: Route
  } {
    const location = normalizeLocation(
      to,
      current || this.history.current,
      append,
      this
    )
    const route = this.match(location, current)
    const fullPath = route.redirectedFrom || route.fullPath
    const base = this.history.base
    const href = createHref(base, fullPath, this.mode)
    return {
      location,
      route,
      href,
      normalizedTo: location,
      resolved: route
    }
  }

  addRoutes (routes: Array<RouteConfig>) {
    this.matcher.addRoutes(routes)
    if (this.history.current !== START) {
      this.history.transitionTo(this.history.getCurrentLocation())
    }
  }
}
```

## 路由模式

* vue 中定义了三种路由模式：`hash`、`history`、`abstract` 。


### **hash 模式**
使用 url 的 hash 值来作为路由。

* url 中带有 `#` 符号，如 `http://localhost:8080/#/`；
* 页面跳转时   `#` 后面 hash 值改变；当 URL 改变时，页面不会重新加载（实际有重新获取数据）；
* 通过浏览器的前进、后退、刷新页面均可以变；

**原理：**

* 监听路由变化： 如果支持 [popstate 事件](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/popstate_event)，则监听 popstate 事件，否则监听 [hashchange 事件](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/hashchange_event);
* 点击 `router-link` 路由切换：判断是否支持  [HTML5 History API](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/history) （replaceState 和 pushState），支持则使用 History API，否则使用 [Location.hash API](https://developer.mozilla.org/zh-CN/docs/Web/API/Location/hash) 

### **History 模式**

 使用 [HTML5 History API](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/history) 和并需要服务器配置。一般用于多页应用。
* url 中没有 `#` 符号
* 可以通过浏览器前进、后退，但刷新页面会导致 404 (如果服务器中没有相应的响应或者资源)；

**原理：**

* 监听路由变化：监听 [popstate 事件](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/popstate_event)；
* 点击 `router-link` 路由切换：使用  [HTML5 History API](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/history) （replaceState 和 pushState）切换路由；

**缺点：**

* 要在服务端增加一个覆盖所有情况的候选资源：如果 URL 匹配不到任何静态资源，则应该返回同一个 `index.html` 页面，这个页面就是你 app 依赖的页面。

### **Abstract 模式**

* 支持所有javascript运行模式。如果发现没有浏览器的API，路由会自动强制进入这个模式。
* 优点
    * 完全控制路由行为

    * 不依赖浏览器环境

    * 避免URL暴露内部结构

    * 在嵌入式场景中保持父级URL不变
* 缺点
    * 无法通过URL分享特定页面

    * 浏览器刷新会丢失路由状态

    * 不利于SEO

    * 用户无法使用浏览器前进后退按钮

## 路由安装

* Vue-Router 的入口文件是 `src/index.js`，其中定义了 `VueRouter` 类，也实现了 `install` 的静态方法：`VueRouter.install = install`，它的定义在 `src/install.js` 中：

  ```javascript
  //防止重复安装 - 单例模式
  // 根组件初始化 - 路由实例初始化和响应式设置
  //子组件连接 - 通过 _routerRoot 建立访问链
  //全局属性注入 - $router 和 $route
  //全局组件注册 - <router-view> 和 <router-link>
  // 路由守卫配置 - 设置合并策略
  // 响应式系统 - 核心的路由状态响应式机制
  export let _Vue
  export function install (Vue) {
    if (install.installed && _Vue === Vue) return
    install.installed = true
  
    _Vue = Vue
  
    const isDef = v => v !== undefined
  
    const registerInstance = (vm, callVal) => {
      let i = vm.$options._parentVnode
      if (isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouteInstance)) {
        i(vm, callVal)
      }
    }
  
    Vue.mixin({
      beforeCreate () {
        if (isDef(this.$options.router)) {
          this._routerRoot = this
          this._router = this.$options.router
          this._router.init(this)
          Vue.util.defineReactive(this, '_route', this._router.history.current)
        } else {
          this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
        }
        registerInstance(this, this)
      },
      destroyed () {
        registerInstance(this)
      }
    })
  
    Object.defineProperty(Vue.prototype, '$router', {
      get () { return this._routerRoot._router }
    })
  
    Object.defineProperty(Vue.prototype, '$route', {
      get () { return this._routerRoot._route }
    })
  
    Vue.component('RouterView', View)
    Vue.component('RouterLink', Link)
  
    const strats = Vue.config.optionMergeStrategies
    strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created
  }
  ```

