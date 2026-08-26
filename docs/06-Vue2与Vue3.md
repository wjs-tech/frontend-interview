# 06 · Vue2 与 Vue3

> Vue 是我用得最久的框架（Vue2 五年打底，后来一路升到 Vue3）。这章就顺着自己真实的升级路径来写——把 Vue2 和 Vue3 到底差在哪对比清楚，也把当年踩过的坑一块记下来。

本章计划覆盖：
- [x] 响应式原理：`Object.defineProperty` vs `Proxy`（为什么升级）
- [x] Composition API vs Options API（`<script setup>` 心智模型）
- [x] 组件通信：props/emit、provide/inject、attrs/slots
- [x] Vue Router 4：路由模式、导航守卫顺序、动态路由、权限
- [x] Pinia vs Vuex（Setup Store 写法）
- [x] 生命周期对照、v-model 实现、diff 思路

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

---

## Composition API vs Options API（⭐️⭐️⭐️）

**一句话结论：** Options API 把代码按"类型"（data / methods / computed / 生命周期）分组，组件一大就散；Composition API 让你按"功能"把相关逻辑写在一起，配合 `<script setup>` 语法糖几乎不写样板，逻辑复用也更干净。

### 🍳 生活类比

Options API 像按"家具种类"收纳的房间：所有沙发放客厅、所有碗放厨房、所有书放书房。你想招待一次客人，得从客厅拿椅子、去厨房拿杯子、去书房翻招待手册——同一件事的东西散在三个房间。

Composition API 像按"生活场景"收纳：把"待客"要用的椅子、杯子、手册全塞进一个标注清楚的收纳箱，要用时一箱拎走。代码同理——"用户登录"这块逻辑（状态、校验、请求、监听）不再被拆到 data / methods / watch 四处，而是聚在一个 `useLogin()` 里。

### 🔍 原理下沉

Options API 是 Vue2 时代的主写法：组件是一个配置对象，data / methods / computed / watch / 生命周期各占一块。同一业务逻辑（比如"搜索框防抖 + 结果展示"）会被拆到 methods、data、watch 里，组件超过两屏就找不到北。逻辑复用靠 mixins，但 mixins 有两大坑：属性来源不透明（"这个 `loading` 到底哪个 mixin 提供的？"），同名属性 / 方法还会悄悄覆盖。

Composition API 是 Vue3 引入的：用 `setup()` 函数把"同一块逻辑"写在一起，最后 return 给模板。后来出了 `<script setup>` 语法糖——直接在 `<script setup>` 里写顶层 `const` / `function`，编译时自动暴露给模板，连 `setup()` 和 `return` 都省了。

为什么值得升级：
- **逻辑复用更干净**：抽成 `useXxx()` 组合函数（类似 React Hooks），来源清晰、无命名冲突；
- **更好的 TS 支持**：基于普通变量和函数，类型推导比 Options 的字符串 key 友好太多；
- **利于 tree-shaking**：功能以函数形式 import，没用到的能摇掉；Options 的属性名是字符串 key，静态分析困难；
- **没有 `this` 心智负担**：`<script setup>` 里直接写变量，不用纠结 `this` 指向。

> 版本演进提醒：Vue3 完全兼容 Options API，老项目不强制改；但新项目官方推荐 `<script setup>`。两者也能混用（`setup()` 返回的属性会和 options 合并），只是一般没必要。

📌 一个容易混的点：`ref` 在 JS 里要通过 `.value` 访问，在模板里不用；`reactive` 则直接 `.属性`。我一开始老在 JS 里写 `count++` 忘了 `.value`，页面不动还以为响应式坏了。

### 🔁 对比学习（Options vs `<script setup>`）

```js
// Options API：按类型分组
export default {
  data() {
    return { count: 0 }
  },
  computed: {
    double() { return this.count * 2 }
  },
  methods: {
    increment() { this.count++ }
  },
  mounted() {
    console.log('mounted', this.count)
  }
}
```

```vue
<!-- Composition API + <script setup>：按功能聚合 -->
<script setup>
import { ref, computed, onMounted } from 'vue'

const count = ref(0)
const double = computed(() => count.value * 2)
const increment = () => { count.value++ }

onMounted(() => console.log('mounted', count.value))
</script>
```

核心差异一句话：**Options 是"按格子填空"，Composition 是"按想法自由搭"。**

### 💥 我踩过的坑

第一次写 `<script setup>` 时，习惯性地想在 `mounted` 里用 `this.$refs.xxx` 拿 DOM，结果直接报错。后来才反应过来：`<script setup>` 里没有 `this`，组件实例还没完全建好就去拿 ref 本就不对，正解是给元素加 `ref="xxx"` 然后用同名的 `const xxx = ref()` 接住。

还有 mixins 那段黑历史：两个 mixin 各自带了 `refresh()`，调了半天不知道哪个生效，最后靠打印堆栈才定位。换成 `useXxx()` 组合函数后，谁提供什么一目了然，再没踩过。

### 🎯 面试官可能追问

- `<script setup>` 编译后其实还是 `setup()`，它具体帮你做了什么？（编译期收集顶层绑定暴露给模板、自动注册组件）
- Composition API 里想访问路由 / store 怎么办？（用 `useRoute()` / `useStore()`，或 `getCurrentInstance()`，但后者不推荐）
- 为什么说 Composition API 更利于 tree-shaking？（函数式 import 可被静态分析摇掉；Options 的属性是字符串 key 难以分析）
- Options 和 Composition 能混着写吗？会有什么坑？（能，`setup()` 返回值会和 options 合并，但混用增加认知负担）

---

### `ref` vs `reactive`：怎么选，我踩过的响应式丢失坑（⭐️⭐️⭐️）

- **一句话结论**：基本类型或"单个值"用 `ref`（记得 `.value`）；一组有关联的对象状态用 `reactive`（直接 `.属性` 访问）。选错不会报错，但会悄悄不更新——这才是坑。

### 🍳 生活类比

`ref` 像贴了快递单的**单个包裹**：取件得先"掀盖"（`count.value`）才能拿到里面的东西。`reactive` 像**敞口的收纳盒**：直接伸手拿就行（`state.count`），但盒子整体不能换，一换就空了。

### 🔍 原理下沉

- `ref` 不管你传基本类型还是对象，对外都包一层带 `.value` 的容器；传对象时内部其实也是调 `reactive` 去代理的。所以 `ref` 是"万能但要多写 `.value`"。
- `reactive` 基于 `Proxy`，**只认对象 / 数组 / Map / Set**。你塞个 `number` 进去它不报错，但压根没代理，改了也不响应——这种静默失败最坑。
- `reactive` 的两个致命点：
  1. **整体替换会断响应**：`state = { ...newObj }` 之后，新对象不是原来那个 Proxy 了，模板绑定的还是旧引用，页面不动。
  2. **解构 / 展开会丢响应**：`const { list } = state` 拿到的 `list` 是普通值，之后改它不会触发更新。要解构得用 `toRefs(state)`。

### 🔁 对比学习

```js
// ref：单个值或对象都行，但要 .value
import { ref } from 'vue'
const count = ref(0)
count.value++          // 模板里不用写 .value，编译帮你解包
const user = ref({ name: 'A' })
user.value.name = 'B'  // 对象内部改，照常响应

// reactive：只接对象，直接 .属性，但别整体换、别直接解构
import { reactive, toRefs } from 'vue'
const state = reactive({ count: 0, name: 'A' })
state.count++          // 直接改，OK
// ❌ state = { count: 1 }       ← 整体替换，响应断了
// ❌ const { count } = state    ← 解构丢响应
const { count, name } = toRefs(state)  // ✅ 用 toRefs 保住响应
```

决策口诀：**"单值用 ref，成团用 reactive；要替换整块状态就 ref 包对象，或用 `Object.assign(state, newObj)` 原地改。"**

### 💥 我踩过的坑

第一次用 `reactive` 接管表单，点"重置"时我直接 `formData = resetForm()`——页面纹丝不动。盯了一小时才发现是把整个引用换了，Proxy 那条线断了。后来要么改用 `Object.assign(formData, resetForm())` 原地更新，要么干脆用 `ref` 包表单对象。

还有一回解构店铺列表：`const { list } = shopState`，在子组件里改 `list` 死活不刷新。才知道解构出来的是死值，得 `storeToRefs`（Pinia）或 `toRefs` 接住。

### 🎯 面试官可能追问

- `ref` 内部是不是也用 `reactive`？（是，对象类型会转成 reactive，基本类型用 `RefImpl` 包一层 `.value`）
- 为什么 `reactive` 不能整体替换？（Proxy 代理的是原对象引用，重新赋值换了引用就脱离了代理）
- `toRefs` 和 `toRef` 区别？（前者把整个对象每个属性转 ref，后者只转单个）
- 模板里 `ref` 为什么不用写 `.value`？（编译期自动解包，但 `v-for` 里或作为对象属性时不一定解包，需注意）

---

### 组件通信：props/emit、provide/inject、attrs/slots（⭐️⭐️）

**一句话结论：** 父子用 props 下发、emit 回传；跨多层用 provide/inject；没被声明为 props 的属性自动走 attrs 透传；插槽 slots 负责"把父的内容塞进子的留白处"。Vue3 里 props 是单向的，子想改父数据得 emit 让父去改。

### 🍳 生活类比

props/emit 像父子间的"爸爸发零花钱（props）/ 儿子交成绩单（emit）"，钱怎么花得回报给爹。provide/inject 像爷爷在族谱立了条家规（provide），隔了三房的孙子不用层层传话，直接能读到（inject）。attrs 像"没拆的快递"——父传了但子没声明 props 的属性，默认挂到子根元素上。slots 像"留白的相框"，父把内容填进子组件留的位置。

### 🔍 原理下沉

- props 单向数据流，子组件不能直接改 props（会告警），要改得 `emit` 事件让父改。
- Vue3 `<script setup>` 用 `defineProps(['msg'])` / `defineEmits(['change'])`，编译期宏，不用 import。
- provide/inject 适合深层传递（主题、locale、当前用户）。它不是状态管理，传可变全局数据容易失控——provide 一般传"只读配置"或 `ref`/`reactive`（子孙 inject 后 `.value` 仍响应）。
- attrs：未被 props 声明的属性 + 事件监听器，默认透传到组件根元素；`inheritAttrs: false` 时可手动 `v-bind="$attrs"` 指定挂到内部元素（典型：把 class/style 透传到内部 input）。
- slots：`<slot />` 默认插槽、`<slot name="x">` 具名、`v-slot`（或 `#`）作用域插槽——子把数据回传给父的插槽内容用。

### 🔁 对比学习（Vue2 vs Vue3）

- Vue2 用 `this.$emit`、选项式 `props: []`；Vue3 `<script setup>` 用 `defineProps`/`defineEmits`。
- `$listeners` 在 Vue3 合并进 `$attrs`（Vue2 里两者分开），少一层心智负担。
- 作用域插槽 Vue2 用 `slot-scope`，Vue3 统一用 `v-slot`（或 `#`），写法更一致。

```vue
<!-- 子组件：emit + 作用域插槽 -->
<script setup>
const emit = defineEmits(['change'])
const props = defineProps({ title: String })
</script>
<template>
  <div>
    <h3>{{ title }}</h3>
    <slot name="footer" :count="props.title.length" />
  </div>
</template>

<!-- 父组件使用 -->
<Child title="hi" @change="onChange">
  <template #footer="{ count }">字数：{{ count }}</template>
</Child>
```

### 💥 我踩过的坑

- 早期在子组件里直接 `this.someProp = newValue` 想改父传的值，页面不更新还告警。后来懂了 props 只读，得 `$emit('update')` 让父改。
- provide/inject 我一度当"全局 state"用，深层组件改了 inject 的值，上游根本不知道，查半天才定位。之后约定 provide 只传只读配置，可变状态走 Pinia。
- attrs 透传踩过：给封装的 `<BaseInput>` 传 class，结果挂到外层 div 而非内部 input，样式错位。加 `inheritAttrs: false` + `v-bind="$attrs"` 指定挂到 input 才修好。

### 🎯 面试官可能追问

- props 为什么是单向的？直接改会有什么问题？（父源数据被悄悄改，数据流不可追踪）
- provide/inject 能传响应式数据吗？怎么做？（传 `ref`/`reactive`，子孙 inject 后照常响应）
- 怎么阻止 attrs 自动继承到根元素？（`inheritAttrs: false` + 手动 `v-bind="$attrs"`）
- 作用域插槽和 props 的区别？（插槽传"UI 片段"且子能把数据回传片段；props 传的是数据）

---

### Vue Router 4（⭐️⭐️）

**一句话结论：** Router 4 把路由表写成数组、用 `createRouter` 创建；两种模式 history（URL 干净、需服务端兜底）和 hash（带 #、无需服务端配置）；导航守卫有固定执行顺序，权限拦截一般挂在全局前置守卫 `beforeEach`。

### 🍳 生活类比

Router 像小区门禁。URL 是你要去的楼栋号：history 模式是"直接报楼栋名"（好看，但保安得认识所有楼，否则迷路——服务端要配置 fallback）；hash 模式是"楼栋名前加个 # 闸机号"（丑点但闸机自己认，不用麻烦保安）。

### 🔍 原理下沉

- 路由模式：`createWebHistory()`（HTML5 history，需服务器所有路径回退到 index.html）/ `createWebHashHistory()`（基于 location.hash，无需服务端配置）。
- 动态路由：`path: '/user/:id'`，`useRoute().params.id` 取；Vue3 用组合式 `useRoute`/`useRouter`。
- 导航守卫顺序（常考）：全局 `beforeEach` → 路由独享 `beforeEnter` → 组件内 `beforeRouteEnter`/`Update`/`Leave` → 全局 `beforeResolve` → 全局 `afterEach`。权限拦截多在 `beforeEach` 做。
- 路由懒加载：`component: () => import('./views/xxx.vue')`，配合 Vite 自动分包。

### 🔁 对比学习（Router 3 vs 4）

- 创建：`new VueRouter({...})` → `createRouter({...})`。
- 模式：`mode: 'history'` → `history: createWebHistory()`。
- 捕获所有：`{ path: '*' }` → `{ path: '/:pathMatch(.*)*' }`。
- 组件内 `beforeRouteEnter` 不能直接访问 `this`，Vue2 靠 `next(vm => {})` 拿实例；Vue3 用 `onBeforeRouteEnter` 等组合式守卫更顺。

```js
// 全局前置守卫做权限
const router = createRouter({ history: createWebHistory(), routes })
router.beforeEach((to) => {
  if (to.meta.requiresAuth && !isLogin()) return '/login'
})
```

### 💥 我踩过的坑

- history 模式部署到 Nginx 子目录，刷新 404。最后在 Nginx 加 `try_files $uri $uri/ /index.html;` 解决——部署必踩的坑。
- 动态路由参数变化（`/user/1` → `/user/2`）组件不重建，数据没刷新。Vue2 用 `watch $route`，Vue3 用 `beforeRouteUpdate` 或 `watch(() => route.params.id)`。

### 🎯 面试官可能追问

- history 和 hash 模式本质区别？history 刷新 404 怎么解决？（服务端 fallback 到 index.html）
- 导航守卫的执行顺序？
- 动态路由参数变了组件不更新怎么办？
- 路由懒加载的原理？（动态 import 分包，按需加载）

---

### Pinia vs Vuex（⭐️⭐️）

**一句话结论：** 新项目直接用 Pinia。它去掉了 Vuex 的 mutation、module 嵌套和 `this.$store` 的魔法，用 `defineStore` 一个函数搞定，Setup Store 写法跟 `ref`/`reactive` 一模一样，心智成本最低。

### 🍳 生活类比

Vuex 像老式货栈：改库存得先写条子给"账房先生"（mutation），账房再入账，流程长。Pinia 像自助仓储——你直接拿、直接放，管理员就是你自己，链路短了一大截。

### 🔍 原理下沉

- Vuex 有 state/getters/mutations/actions 四件套，mutation 必须同步、action 可异步；模块要 `namespaced`。
- Pinia：只有 state/getters/actions，没有 mutation；action 里既能同步也能异步。
- Setup Store：`const useX = defineStore('x', () => { const count = ref(0); const inc = () => count.value++; return { count, inc } })`——和写组件逻辑一样。
- Option Store 写法类似 Vuex：`defineStore('x', { state, getters, actions })`。

### 🔁 对比学习

```js
// Vuex：mutation 必须同步
const store = new Vuex.Store({
  state: { count: 0 },
  mutations: { inc(s) { s.count++ } },
  actions: { incAsync({ commit }) { setTimeout(() => commit('inc'), 1000) } }
})
```
```js
// Pinia Setup Store：写法贴近 ref/reactive
const useCounter = defineStore('counter', () => {
  const count = ref(0)
  const inc = () => count.value++
  const incAsync = () => setTimeout(inc, 1000)
  return { count, inc, incAsync }
})
```

### 💥 我踩过的坑

- Vuex 里曾在 mutation 写异步（调接口），调试时 action 日志和 state 变更对不上，崩溃。Vuex 规矩是"异步只在 action"，踩过才记住。
- Pinia 刚开始不习惯"没有 mutation"，后来发现反而清爽——改状态就是调 action 或直接 `store.count++`（devtools 照样追踪），负担小很多。

### 🎯 面试官可能追问

- Pinia 为什么去掉 mutation？（Vuex 的 mutation 是为 devtools 时间旅行强加的概念，实际多为负担；Pinia 用 patch 仍支持追踪）
- Pinia 怎么拆模块？（每个 store 一个 `defineStore`，天然独立，不用嵌套 namespace）
- 两者 TS 支持差别？（Pinia 基于 TS 设计，类型推导远好于 Vuex）

---

### 生命周期对照、v-model 实现、diff 思路（⭐️）

**一句话结论：** Vue2 和 Vue3 生命周期大体对应，Vue3 组合式里 `destroyed` 改名 `onUnmounted`；`v-model` 本质是 prop + 事件（编译期语法糖）；diff 上 Vue2 双端比较、Vue3 引入 patchFlag 做静态标记优化。

### 🍳 生活类比

生命周期像餐厅营业流程——开门迎客（created/mounted）、营业中（updated）、打烊（unmounted）。Vue3 把"打烊"改叫 `onUnmounted`，意思没变，只是换了挂牌。

### 🔍 原理下沉（精简）

- 生命周期对照：Vue2 `beforeCreate/created/mounted/updated/destroyed` ↔ Vue3 选项式同名、`<script setup>` 用 `onBeforeMount/onMounted/onBeforeUpdate/onUpdated/onUnmounted`。`destroyed` 在 Vue3 改名 `unmounted`（语义更准：组件是被"卸载"而非"销毁"）。
- v-model 实现：见 07 章受控组件一节。Vue3 子组件用 `defineModel()` 即可，编译为 `modelValue` prop + `onUpdate:modelValue` 事件；多个 `v-model:xxx` 各自对应一个 prop + update 事件。
- diff：Vue2 用「双端 diff」（头尾四个指针向中间夹）；Vue3 编译阶段给动态节点打 `patchFlag`（如 `TEXT`、`CLASS`），diff 时只比对打了标记的部分，静态节点直接跳过，性能更好；同时用「最长递增子序列」优化节点移动。

### 🔁 对比学习

- Vue2：`this.$destroy()` 销毁实例；Vue3：`app.unmount()` 卸载应用。
- Vue3 编译期优化（patchFlag + 静态提升 hoistStatic）是 Vue2 没有的，也是 Vue3 渲染更快的主因之一。
- `v-model` 与 `.sync`：Vue2 有 `.sync` 做单向"更新 prop"，Vue3 用 `v-model:xxx` 统一取代 `.sync`。

### 💥 我踩过的坑

- 在 `created` 里操作 DOM 报错——那时还没挂载，`mounted` 才是能安全碰 DOM 的钩子。基础但真踩过。
- Vue3 里忘了 `onUnmounted` 清定时器/事件监听，组件切走后定时器还在跑，内存泄漏。现在习惯 `onMounted` 注册、配对 `onUnmounted` 清理。

### 🎯 面试官可能追问

- Vue3 为什么把 `destroyed` 改成 `unmounted`？（更准确：组件是被"卸载"而非"销毁"）
- Vue3 的 patchFlag 解决了什么？（跳过静态节点比对，减少无效 diff）
- `v-model` 和 `.sync` 的关系？（Vue3 用 `v-model:xxx` 取代 `.sync`）
