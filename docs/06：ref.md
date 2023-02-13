# 第六章: 响应系统 -- ref 的响应性

### 01: 前言

在上一章中我们完成了 reactive 函数, 同时也知道了 reactive 函数的局限性, 知道了只靠 reactive 函数, vue 是没办法构建出完善的响应性系统的.

所以我们还需要另外一个函数 ref。

本章将致力于解决以下三个问题:

1. `ref` 函数是如何进行实现的呢？
2. `ref` 可以构建简单数据类型的响应性吗?
3. 为什么 `ref`类型的数据，必须要通过 `.value`访问值呢 ?

接下来，开始吧

### 02：源码阅读：ref 复杂数据类型的响应性

和学习 reactive 的时候一样，我们首先来看一下 ref 函数，vue3 源码的执行过程。

1. 创建测试实例 `packages/vue/examples/imooc/ref.html`

   ```html
   <body>
     <div id='app'>
     </div>
   </body>
   <script>
   	const { ref, effect } = Vue
     
     const obj = ref({
       name: '张三'
     })
     
     // 调用 effect 方法
     effect(() => {
       document.querySelector('#app').innerText = obj.value.name
     })
     
     setTimeout(() => {
       obj.value.name = '李四'
     }，2000)
   </script>
   ```

2. 通过 Live Server 运行测试实例

3. ref 的代码位于 `packages/reactivity/src/ref.ts`之下，我们可以在此处打断点

#### ref 函数

1. ref 函数中，直接触发 createRef 函数
2. 在 createRef 中，进行了判断如果当前已经是一个 ref 类型数据则直接返回，否则 返回 RefImpl 类型的示例
3. 那么这个 RefImpl 是什么呢？
   1. RefImpl 是同样位于`packages/reactivity/ref.ts` 之下的一个类
   2. 该类的构造函数中，执行了一个 toReactive 的方法，传入了 value 并把返回值赋值给了 this._value，那么我们来看看 toReactive 的作用：
      - toReactive 方法把数据分成了两种类型：
        1. 复杂数据类型：调用了 reactive 函数，即把 value 变为响应性的。
        2. 简单数据类型：直接把 value 原样返回
   3. 该类提供了一个分别被 get 和 set 标记的函数 value
      1. 当执行 xxx.value 时，会触发 get 标记。
      2. 当执行 xxx.value = xxx 时，会触发 set 标记
4. 至此 ref 函数执行完成。

由以上逻辑可知：

1. 对于 ref 而言，主要完成了 RefImpl 的实例
2. 在构造函数中对传入的数据进行了处理：
   1. 复杂数据类型：转为响应性的 proxy 实例
   2. 简单数据类型：不去处理
3. RefImpl 分别提供了 get value、set value 以此来完成对 getter 和 setter 的监听，注意这里并没用使用 proxy

#### effect 函数

当 ref 函数执行完成之后，测试实例开始执行 effect 函数。

effect 函数我们之前跟踪过它的执行流程，我们知道整个 effect 主要做了3件事情：

1. 生成 ReactiveEffect 实例
2. 触发 fn 方法，从而激活 getter
3. 建立了 targetMap 和 activeEffect 之间的联系
   1. dep.add(activeEffect)
   2. activeEffect.deps.pust(dep)

通过以上可知，effect 中会触发 fn 函数，也就是说会执行 obj.value.name，那么根据 get value 机制，此时会触发 RefImpl 的 get value 方法。

#### get value()

1. 在get value 中触发 trackRefValue 方法
   1. 触发 trackEffects 函数，并在此时为 ref 新增了一个 dep 属性
   2. 而 trackEffects 其实我们是有过了解的，我们知道 trackEffects 主要的作用就是：**收集所有的依赖**
2. 至此 get value 执行完成

由以上逻辑可知：

整个 get value 的处理逻辑还是比较简单的，主要还是通过之前的 trackEffects 属性来收集依赖

#### 再次触发 get value()

最后就是在两秒之后，修改数据源了：

```js
obj.value.name = '李四'
```

但是这里有一个很关键的问题，需要大家进行思考，那就是：**此时会触发 get value 还是 set value？**

我们知道以上代码可以拆解为：

```JS
const value = obj.value
value.name = '李四'
```

那么通过以上代码可以清晰的知道，其实触发的应该是 get value 函数。

在 get value 函数中：

1. 再次执行 trackRefValue 函数：
   1. 但是此时 activeEffect 为 undefined，所以不会执行后续逻辑
2. 返回 this._value:
   1. 通过 **构造函数**，我们可知，此时的 this._value 是经过 toReactive 函数过滤之后的数据，在当前实例中为 **proxy** 实例。
3. get value 执行完成

由以上逻辑可知：

1. const value 是 proxy 类型的实例，即：**代理对象**，被代理对象为 `{name: '张三'}`
2. 执行 `value.name = '李四'`，本质上是触发了 `proxy` 的 `setter`
3. 根据 `reactive` 的执行逻辑可知，此时会触发 `trigger` 触发依赖。
4. 至此，修改视图

#### 总结：

由以上逻辑可知：

1. 对于 `ref` 函数，会返回 `RefImpl` 类型的实例
2. 在该实例中，会根据传入的数据类型进行分开处理
   1. 复杂数据类型：转化为 `reactive` 返回的 `proxy` 实例
   2. 简单数据类型：不做处理
3. 无论我们执行 `obj.value.name` 还是 `obj.value.name` = xxx 本质上都是触发了 `get value`
4. 之所以会进行 **响应性** 是因为 `obj.value` 是一个 `reactive` 函数生成的 `proxy`

### 03：框架实现：ref 函数 - 构建复杂数据类型的响应性

1. 在 `packages/reactivity/src/` 文件夹中，新增 `ref.ts` 模块

   ```typescript
   import { createDep, Dep } from './dep'
   import { activeEffect, trackEffects } from './effect'
   import { toReactive } from './reactive'
   
   export interface Ref<T = any> {
     value: T
   }
   
   export function ref (value?: unknown) {
     return createRef(value, false)
   }
   
   function createRef (rawValue: unknown, shallow: boolean) {
     if (isRef(rawValue)) {
       return rawValue
     }
   
     return new RefImpl(rawValue, shallow)
   }
   
   class RefImpl<T> {
     private _value: T
   
     public dep?: Dep = undefined
   
     constructor (value: T, public readonly __v_isShallow: boolean) {
       this._value = __v_isShallow ? value : toReactive(value)
     }
   
     get value () {
       trackRefValue(this)
       return this._value
     }
   
     set value (newVal) {}
   }
   
   export function trackRefValue (ref) {
     if (activeEffect) {
       trackEffects(ref.dep || (ref.dep = createDep()))
     }
   }
   
   export function isRef (r: any): r is Ref {
     return !!(r && r.__v_isRef === true)
   }
   
   ```

   

2. 在 `packages/reactivity/src/reactive.ts` 中，新增 `toReactive` 方法：

   ```typescript
   /**
    * 将指定数据变成 reactive 数据
    */
   export const toReactive = <T extends unknown>(value: T): T => {
     return isObject(value) ? reactive(value as object) : value
   }
   ```

3. 在 `packages/shared/src/index.ts`中，新增 `isObject` 方法

   ```typescript
   export const isObject = (val: unknown) =>
     val !== null && typeof val === 'object'
   ```

4. 在 `packages/reactivity/src/index.ts` 中，导出 `ref` 函数

   ```typescript
   ...
   export { ref } from './ref'
   ```

5. 在 `packages/vue/src/index.ts` 中，导出 `ref` 函数：

   ```typescript
   export { reactive, effect, ref } from '@vue/reactivity'
   ```

   

至此，`ref` 函数构建完成。

```html
<script>
    const { ref, effect } = Vue

    const obj = ref({
      name: '张三'
    })

    effect(() => {
      document.querySelector('#app').innerText = obj.value.name
    })

    setTimeout(() => {
      obj.value.name = '李四'
    }, 2000)
  </script>
```

可以发现代码测试成功

### 04：总结：`ref` 复杂数据类型的响应性

根据以上代码实现我们知道，针对于 `ref` 的复杂数据类型而言，它的响应性本身，其实是 **利用** `reactive` 函数进行的实现，即：

```js
const obj = ref({
  name: '张三'
})
const obj = reactive({
  name: "张三"
})
```



### 06：框架实现：`ref` 函数 - 构建简单数据类型的响应性

1. 在 `packages/reactivity/src/ref.ts` 中，完善 `set value ` 函数：

   ```typescript
   class RefImpl<T> {
     private _value: T
     private _rawValue: T
     public dep?: Dep = undefined
   
     constructor (value: T, public readonly __v_isShallow: boolean) {
       this._rawValue = value
       this._value = __v_isShallow ? value : toReactive(value)
     }
   
   	...
     
     set value (newVal) {
       if (hasChanged(newVal, this._rawValue)) {
         this._rawValue = newVal
         this._value = toReactive(newVal)
         this._value = toReactive(newVal)
         triggerRefValue(this)
       }
     }
   }
   
   /**
    * 触发依赖
    */
   export function triggerRefValue (ref) {
     if (ref.dep) {
       triggerEffects(ref.dep)
     }
   }
   ```

2. 在 `packages/shared/src/index.ts` 中，新增 `hasChanged` 方法

   ```typescript
   /**
    *  对比两个数据是否发生改变
    */
   export const hasChanged = (value: any, oldValue: any): boolean =>
     !Object.is(value, oldValue)
   ```

至此，简单数据类型的响应性处理完成。

我们可以创建对应测试实例：`packages/vue/examples/reactivity/ref-shallow.html`

```html
 <script>
    const { ref, effect } = Vue

    const obj = ref('张三')

    effect(() => {
      document.querySelector('#app').innerText = obj.value
    })

    setTimeout(() => {
      obj.value = '李四'
    }, 2000)
  </script>
```

测试成功，表示代码完成。

### 07：总结：`ref` 简单数据类型响应性

那么 到这里我们实现了 `ref` 简单数据类型的响应性处理。

在这样的代码中，我们需要知道的 最重要的一点是：**简单数据类型，不具备数据监听的概念**，即本身并不是响应性的。

只是因为 `vue` 通过了 `set value()` 的语法，把 **函数调用变成了属性调用的形式，**让我们通过主动调用该函数，来完成了一个 “类似于” 响应性的结果。

### 08： 总结

那么到这里我们已经完成了 `ref `响应性函数的构建，那么大家还记不记得开篇时所问的三个问题：

1. `ref `函数是如何进行实现的呢？
2. `ref `可以构建简单数据类型的响应性么？
3. 为什么`ref `类型的数据，必须要通过 `.value`访问值呢？

大家现在再次面对这三个问题，是否能够回答出来呢？

1. 问题一：`ref` 函数是如何进行实现的呢？
   1. `ref` 函数本质上是生成了一个 `RefImpl` 类型的实例对象，通过 `get` 和 `set` 标记处理了 `value` 函数
2. 问题二：`ref` 可以构建简单数据类型的响应性么？
   1. 是的。`ref` 可以构建简单数据类型的响应性
3. 问题三：为什么 `ref`类型的数据，必须要通过 `.value` 访问值呢？
   1. 因为 `ref` 需要处理简单数据类型的响应性，但是对于简单数据类型而言，它无法通过 `proxy` 建立代理。
   2. 所有 vue 通过 `get value()` 和` set value() `定义了两个属性函数，通过主动触发这两个函数（属性调用）的形式进行 **依赖收集** 和 **触发依赖**
   3. 所以我们必须通过 `.value` 来保证响应性。
