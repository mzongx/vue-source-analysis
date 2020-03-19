# vue-source-analysis
vue源码分析学习
****
## 目录
* [Vue的十万个为什么？](#Vue的十万个为什么？)
  * [为什么this.xxx能访问到data上面的数据？](##为什么this.xxx能访问到data上面的数据？)
# Vue的十万个为什么？
## 为什么this.xxx能访问到data上面的数据？
我们经常能在mounted或者watch，method等上面写this.xxx就能访问到定义在data上面的数据，其实vue内部做了data数据proxy到当前vm（vm就是vue本身实例）上。   
   
在Vue这个构造函数中，我们可以看到this._init(options)这个初始化函数,源码在`src/core/instance/index.js` 中
```javascript
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}
```

紧接着我们看下this._init方法长上面样，源码在`src/core/instance/init.js`中。
```javascript
export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
    const vm: Component = this
    // a uid
    vm._uid = uid++

    let startTag, endTag
    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      startTag = `vue-perf-start:${vm._uid}`
      endTag = `vue-perf-end:${vm._uid}`
      mark(startTag)
    }

    // a flag to avoid this being observed
    vm._isVue = true
    // merge options
    if (options && options._isComponent) {
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
      initInternalComponent(vm, options)
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      initProxy(vm)
    } else {
      vm._renderProxy = vm
    }
    // expose real self
    vm._self = vm
    initLifecycle(vm)
    initEvents(vm)
    initRender(vm)
    callHook(vm, 'beforeCreate')
    initInjections(vm) // resolve injections before data/props
    initState(vm)
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created')

    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      vm._name = formatComponentName(vm, false)
      mark(endTag)
      measure(`vue ${vm._name} init`, startTag, endTag)
    }

    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
}
```
可以看到vue在初始化到时候，整个流程相当清晰，合并配置，初始化生命周期，初始化事件，初始化渲染，初始化data,props,computed,watcher等等。而本次重点我们需要找到initState(vm)这个方法，里面就是代理了data。源码在`src/core/instance/state.js`中。
```javascript
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```
```const opts = vm.$options```就是我们在new Vue(options)中传入的options，而如果我们传了data的话，就会执行initData(vm)这个方法，接下来我们看下这个方法做了什么，在同个文件中可以找到。
```javascript
function initData (vm: Component) {
  let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== 'production' && warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    )
  }
  // proxy data on instance
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm
        )
      }
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) {
      proxy(vm, `_data`, key)
    }
  }
  // observe data
  observe(data, true /* asRootData */)
}
```
在这个函数中，首先会去判断data是否是function，如果是funciton的话，就调用getData去转化一下，否则就直接把data赋值。紧接着会执行```Object.keys(data)```把data里面的key用数组保存起来，比如长这样['message', 'name', 'age']等等。这里的props，methods其实就是用来判断data中的key是否跟这两个里面的key同名，同名就提示。这里面的重点就是```proxy(vm, `_data`, key)```。我们在同文件中继续找到proxy这个方法。
```javascript
export function proxy (target: Object, sourceKey: string, key: string) {
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```
这里就是庐山真面目的时候，一目了然。target就是vm，也就是vue本身，sourceKey就是_data，key就是options中data里面的key。而这里的set,get方法就是把data中的key直接代理到了vm里面，最后执行了```Object.defineProperty(target, key, sharedPropertyDefinition)```所以我们就可以用this.xxx来访问data中的xxx了，props,methods等也是同理的。
