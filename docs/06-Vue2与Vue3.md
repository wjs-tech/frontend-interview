# 06 · Vue2 与 Vue3

> Vue 是我用得最久的框架（Vue2 五年打底，后来一路升到 Vue3）。这章就顺着自己真实的升级路径来写——把 Vue2 和 Vue3 到底差在哪对比清楚，也把当年踩过的坑一块记下来。

本章计划覆盖：
- [x] 响应式原理：`Object.defineProperty` vs `Proxy`（为什么升级）
- [ ] Composition API vs Options API（`<script setup>` 心智模型）
- [ ] 组件通信：props/emit、provide/inject、attrs/slots
- [ ] Vue Router 4：路由模式、导航守卫顺序、动态路由、权限
- [ ] Pinia vs Vuex（Setup Store 写法）
- [ ] 生命周期对照、v-model 实现、diff 思路

响应式是 Vue 最绕不过去的一块，这章我特意写得细一些，把对比和踩坑都摊开讲。

---

## 响应式原理：`Object.defineProperty` vs `Proxy`（⭐️⭐️⭐️）

**一句话结论：** Vue2 用 `Object.defineProperty` 给每个属性单独加 getter/setter 做依赖收集；Vue3 换成 `Proxy` 代理整个对象，于是新增属性、删除属性、数组下标修改这些 Vue2 怎么都搞不定的事，Vue3 天然就能监听到。

### 🍳 生活类比

把 Vue2 想成：房东给房子里的**每一件家具**都配了一个管家。你动一下沙发，对应管家就跑去喊"有人改了沙发！"。但你要是搬把新椅子进来，没有管家认识它，动了也没人知道；你要是直接把墙拆了重建（数组下标改值），管家更管不着。

Vue3 换成 `Proxy`，相当于给**整栋楼**装了智能门禁：无论谁进出、是新住户还是改了房屋结构，门禁统一拦截、统一通知。所以 Vue3 能感知"新增属性""删除属性""数组下标变化"——不用你再额外做啥。

### 🔍 原理下沉

Vue2 在组件初始化时递归遍历 `data`，对每个属性 `Object.defineProperty` 重写 getter/setter：getter 里做依赖收集（Dep + Watcher），setter 里触发更新。本质是发布-订阅。

为什么 Vue3 要换成 Proxy？因为 `defineProperty` 有三个绕不开的硬伤：

1. 必须**预先遍历**已知的属性，新增 / 删除属性收不到；
2. 数组的原生方法（`push`/`pop`）和**下标赋值**拦截不到，Vue2 只能去重写数组那几个方法（算 hack）；
3. 深层嵌套对象要**递归**遍历，初始化成本不低。

Proxy 直接代理整个对象，`get`/`set`/`deleteProperty`/`has` 都能拦，而且是**惰性**的——你访问到哪一层才代理哪一层，不必一开始就全量遍历。

> 版本演进提醒：Vue2 靠 `Vue.set(obj, key, val)` / `this.$set` 补"新增属性"的窟窿；Vue3 因为 Proxy，普通赋值即可，没有这层心智负担。

顺带一提 React：那边不靠"拦截对象"这套，而是让你返回新状态（`setState` / 不可变更新）触发重渲染，是另一条心智模型，和 Vue 的响应式拦截不是一回事。

### 🔁 对比学习（Vue2 vs Vue3）

```js
// Vue2：给每个属性单独上锁
function defineReactive(obj, key, val) {
  Object.defineProperty(obj, key, {
    get() {
      dep.depend()        // 收集依赖
      return val
    },
    set(newVal) {
      val = newVal
      dep.notify()        // 通知更新
    }
  })
}
```

```js
// Vue3：一整层楼交给 Proxy
const reactive = (obj) => new Proxy(obj, {
  get(target, key, receiver) {
    track(target, key)                              // 收集依赖
    const res = Reflect.get(target, key, receiver)
    return typeof res === 'object' ? reactive(res) : res  // 惰性代理
  },
  set(target, key, value, receiver) {
    Reflect.set(target, key, value, receiver)
    trigger(target, key)                           // 通知更新
    return true
  }
})
```

核心差异一句话：**Vue2 是"逐属性上锁"，Vue3 是"整栋楼门禁"。**

### 💥 我踩过的坑

刚用 Vue2 那阵，后台返回的用户对象没带 `age` 字段，我直接在 `mounted` 里写 `this.user.age = 18`，页面死活不更新。查了半天才明白得用 `this.$set`。后来凡是"可能新增字段"的对象，我都先 `Object.assign` 一个默认值模板，提前把坑填上。

还有一次数组：`this.list[0] = newValue` 不奏效，我一度以为是框架 bug，最后发现 Vue2 拦不住下标赋值，得用 `splice` 或 `$set`。这段其实是面试被反问"为什么"时，我才彻底搞懂的——也算因祸得福。

### 🎯 面试官可能追问

- `Proxy` 的 `get` 里为什么用 `Reflect.get` 而不是直接 `target[key]`？（和 `receiver`、原型链继承有关）
- Vue3 的 `reactive` 为什么要"访问到才代理"？有什么性能上的考量？
- 如果对象被 Proxy 包了好几层，`has` / `deleteProperty` 这些陷阱怎么处理？
- Vue2 当年能不能也用 Proxy？为什么没选？（Proxy 不支持 IE11 以下，是当时的兼容性代价）
