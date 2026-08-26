# 07 · React

> 我在学、边写边复习的一章。会刻意和 Vue 做对比——这恰好是我真实的学习方式。

本章计划覆盖：
- [x] JSX 本质（与 Vue 模板对比）
- [x] Hooks：`useState` / `useEffect` / `useMemo` / `useCallback`
- [x] 受控组件 vs 非受控组件
- [x] reconciliation 与 Fiber（渲染机制）
- [x] 状态管理：Context / Redux / Zustand（与 Pinia 对比）
- [x] React vs Vue：心智模型差异（个人体会）

写这章时我会标注"这是我学的时候卡住的地方"，保持真实。

---

### `v-model` 双向绑定 vs React 受控组件（⭐️⭐️⭐️）

- **一句话结论**：Vue 用 `v-model` 一个指令把"数据→视图"和"视图→数据"两头接好；React 不提供双向绑定语法糖，得自己用 `value` + `onChange` 把这两段手写出来（受控组件）。本质差别是：Vue 默认帮你"绑回去"，React 坚持单向数据流、要你手动接。

### 🍳 生活类比

Vue 的 `v-model` 像**对讲机**：你说的话自动传过去，对方回话自动回来，一根线管两头。React 的受控组件像**传声筒得两头拿**：你先举喇叭把数据念给输入框听（`value`），再拿只耳朵（`onChange`）听它回的话，两件事得自己接起来。

### 🔍 原理下沉

- Vue 的 `v-model` 是语法糖。Vue2 编译成 `:value="x" @input="x = $event"`；Vue3 用 `modelValue` + `@update:modelValue`，子组件 `emit('update:modelValue', val)` 即可。
- Vue3 支持**多个** `v-model`：`v-model:title` `v-model:visible`，各自对应一个 prop 和 update 事件；自定义组件还能用 `defineModel()` 一步到位（3.4+）。
- React 这边没有双向绑定。受控组件 = `value` 由 state 驱动、`onChange` 里 `setState` 把用户输入写回 state，再流回 `value`。视图永远只是 state 的投影。
- 非受控组件则用 `defaultValue` + `ref` 拿值，React 不接管每次输入。

### 🔁 对比学习

```vue
<!-- Vue：一行搞定 -->
<template>
  <input v-model="name" />
  <!-- 编译后 ≈ :value="name" @input="name = $event.target.value" -->
</template>
<script setup>
import { ref } from 'vue'
const name = ref('')
</script>
```

```jsx
// React：两段自己接
function Input() {
  const [name, setName] = useState('')
  return (
    <input
      value={name}
      onChange={(e) => setName(e.target.value)}  // 视图→数据 手动接
    />
  )
}
```

自定义组件同理：Vue 子组件 `defineModel()` 对外暴露 `v-model`；React 子组件靠父传 `value` + `onChange` 回调，父组件里拼成受控。

### 💥 我踩过的坑

Vue 侧：早年在自定义组件上直接写 `v-model` 不生效，忘了子组件必须 `emit('input')` / `emit('update:modelValue')` 配合——父的 `v-model` 只是语法糖，真正干活的是那对 prop + 事件。

React 侧（这块我也在学，按 React 18 写）：第一次写受控 `textarea`，只写了 `value` 忘了 `onChange`，React 直接警告、输入框敲不进字。才明白"受控=值完全由 state 说了算，你得自己接 onChange 把字接回来"。还有 `e.target.value`（文本）和 `e.target.checked`（checkbox）搞混过，类型不对拿不到值。

> 说明一下：React 19 对"受控 / 非受控混用"的控制台警告有调整，面试若被问版本差异，这块我得再确认下最新行为，不敢瞎答。

### 🎯 面试官可能追问

- `v-model` 编译后到底是什么？（Vue3：`modelValue` prop + `onUpdate:modelValue` 事件）
- 一个组件怎么支持多个 `v-model`？（`v-model:xxx` 对应 `xxx` prop 名 + `onUpdate:xxx` 事件）
- React 为什么设计成单向数据流，不学 Vue 做双向绑定？（可预测、调试简单，状态变更集中；双向语法糖在复杂表单里反而难追踪来源）
- 受控组件和非受控组件怎么选？（表单要实时校验 / 联动用受控；一次性取值、无联动用非受控更省事）

---

### JSX 本质（与 Vue 模板对比）（⭐️⭐️）

**一句话结论：** JSX 不是模板语言，它就是 JS 的函数调用语法糖——编译后全是 `React.createElement(...)`。Vue 模板是 HTML 风格、编译期优化强；React 的 JSX 更灵活、更"JS 原生"，但少了编译期静态标记那套。

### 🍳 生活类比

Vue 模板像"填写好的点菜单"（后厨=编译器提前知道哪道固定、哪道现做，备好料）；JSX 像"现场报菜名"（你直接用 JS 描述要什么，`createElement` 现做），灵活但要自己把控。

### 🔍 原理下沉

- JSX 编译后 = `React.createElement(type, props, ...children)`，本质是 JS 表达式，所以能写 `if`/`map`/变量，不受模板语法限制。
- Vue 模板编译为渲染函数，但 Vue 在编译期就分析出哪些节点永远不变（静态提升）、哪些是动态的（patchFlag），运行时 diff 更快。
- React 没有编译期静态标记这套（除非特殊配置），diff 靠运行时协调（reconciliation）。

### 🔁 对比学习

```jsx
// React JSX：本质是 createElement 调用
const el = (
  <div className="box">
    {items.map(i => <span key={i.id}>{i.name}</span>)}
  </div>
)
// 编译 ≈ React.createElement('div', { className: 'box' }, items.map(...))
```
```vue
<!-- Vue 模板：HTML 风格，编译期分析 -->
<div class="box">
  <span v-for="i in items" :key="i.id">{{ i.name }}</span>
</div>
```

### 💥 我踩过的坑（学 React 时）

- 新手爱写 `{ if (x) ... }` 直接放 JSX 里，结果报错——JSX 里只能放**表达式**，得用三元 `x ? a : b` 或 `&&` 短路。和 Vue 模板里不能用 `if` 只能 `v-if` 一个道理，但要换种写法。
- `key` 忘了写，列表重排时 React 复用错节点。Vue 的 `:key` 同理重要。

### 🎯 面试官可能追问

- JSX 和 `createElement` 的关系？
- Vue 模板相比 JSX 的优势在哪？（编译期优化 / 静态提升 / patchFlag）
- 为什么列表要写 key，不写会怎样？

---

### Hooks：useState / useEffect / useMemo / useCallback（⭐️⭐️⭐️）

**一句话结论：** Hooks 是函数组件里"挂状态、副作用、记忆值"的钩子。useState 管状态、useEffect 管副作用（DOM/订阅/请求）、useMemo/useCallback 管"别重复算/别重复创建函数"。React 官方强调：只在函数顶层调用、不能在条件/循环里调——否则顺序乱套。

### 🍳 生活类比

useState 像给组件装了个"记事本"（状态变了就重画）；useEffect 像"贴了张便利贴：挂载时/某些东西变了时去做这件事"（副作用）；useMemo 像"把算好的结果贴墙上，依赖没变就不重算"；useCallback 像"把常用的动作记在便签上，依赖没变就别重写一遍"。

### 🔍 原理下沉

- useState：`const [v, setV] = useState(0)`，setV 触发重渲染；函数式更新 `setV(p => p+1)` 避免依赖旧值出错。
- useEffect：`useEffect(fn, deps)`。deps 空 `[]` 只跑一次（挂载）；省略 deps 每次都跑；返回清理函数 `return () => {...}` 在卸载/下次执行前调（清订阅/定时器）。
- useMemo：缓存计算结果，`useMemo(() => heavy(a, b), [a, b])`，依赖变才重算。
- useCallback：缓存函数本身，`useCallback(fn, deps)`，配合 `React.memo` 把稳定函数传给子组件避免无谓重渲染。
- 规则：只在顶层调用（React 靠调用顺序匹配状态，条件里调会错位）；只在 React 函数组件/自定义 Hook 里调用。

### 🔁 对比学习（Vue 对照）

- useState ↔ `ref`/`reactive`（状态）
- useEffect ↔ `watch`/`watchEffect`（副作用、依赖追踪）；但 Vue 的 watch 是"精准依赖"，React 的 useEffect deps 要手写、容易漏
- useMemo ↔ `computed`（缓存派生值）
- useCallback ↔ Vue 里函数本身稳定（`<script setup>` 顶层函数天然不重建，一般不需等价物）

```jsx
function Counter() {
  const [count, setCount] = useState(0)
  useEffect(() => {
    const t = setInterval(() => setCount(c => c + 1), 1000)
    return () => clearInterval(t)   // 清理：unmount 时清定时器
  }, [])
  const double = useMemo(() => count * 2, [count])
  return <span>{double}</span>
}
```

### 💥 我踩过的坑（学 React 时）

- 在 useEffect 里取接口但没加 deps 或写错，导致死循环或拿旧数据。第一次写"挂载拉列表"时没加空数组 `[]`，每次渲染都重新请求，页面狂刷。
- 依赖数组漏了变量，effect 用了旧闭包值，数据不对还找不到原因。后来靠 `react-hooks/exhaustive-deps` 这个 lint 规则兜底。
- useState 直接 `setCount(count + 1)` 快速连点会丢更新（count 是闭包旧值），改成函数式 `setCount(c => c + 1)` 才稳。

### 🎯 面试官可能追问

- 为什么 Hooks 不能在条件/循环里调用？（React 用调用顺序对应状态，顺序变了就错位）
- useEffect 的清理函数什么时候执行？（下次 effect 前 + 卸载时）
- useMemo/useCallback 是不是越多越好？（不是，过度缓存增加复杂度，真有性能问题才用）
- Vue 的 computed 和 React 的 useMemo 差别？（computed 自动追踪依赖；useMemo 要手写 deps）

---

### reconciliation 与 Fiber（⭐️⭐️）

**一句话结论：** reconciliation 是 React "比对新旧虚拟 DOM、算出最小变更"的算法；Fiber 是 React 16 引入的新架构，把渲染拆成可中断的小任务，避免大列表更新卡住主线程。这块我也在学，讲个大概。

### 🍳 生活类比

reconciliation 像搬家时"对比新旧房间清单，只搬要变的家具"；Fiber 像把"一次性搬完一整栋"改成"搬一件歇一下、有人按门铃（用户点击）就先去开门"，保证页面不卡。

### 🔍 原理下沉（点到为止）

- 旧栈调和（Stack Reconciler）递归比对，一旦开始就不能停，大组件树会阻塞主线程（掉帧）。
- Fiber 把更新拆成"Fiber 节点"链表，逐个处理，每处理完一个就检查是否有更高优先级任务（如用户输入），可暂停/恢复/让出主线程。
- diff 策略：同类型节点才更新、不同类型直接重建；列表靠 key 复用。
- React 18 的并发特性（如 `startTransition`）建立在 Fiber 之上，但我还没深入用。

### 🔁 对比学习（Vue 对照）

- 两者都有虚拟 DOM + diff，但 Vue3 靠编译期 patchFlag 减少比对范围，React 靠运行时 Fiber 调度避免阻塞——不同路子解决"高效更新"。
- React 的 Fiber 调度更偏向"渲染不卡 UI"，Vue 的优化更偏向"比对更快"。

### 💥 我踩过的坑

- 这块还在学，目前只踩过"滥用大列表导致卡顿"的表层问题。深层 Fiber 调度细节（如优先级 lane 模型）我得再啃源码才敢细聊，面试若深问我会如实说"了解大概，细节在补"。

### 🎯 面试官可能追问

- Fiber 解决了什么问题？（长任务阻塞主线程 → 可中断渲染）
- key 在 diff 里的作用？
- React 18 并发模式和你了解的 Fiber 是什么关系？（建立在其上，但细节我还在学）

---

### 状态管理：Context / Redux / Zustand（与 Pinia 对比）（⭐️⭐️）

**一句话结论：** React 自带 Context 做跨组件传值（适合低频配置）；Redux 是经典全局状态库（action→reducer→store，样板多）；Zustand 是轻量后起之秀（create 一个 hook 就用，最像 Pinia 的体验）。我个人更看好 Zustand / Pinia 这类"小而直接"的。

### 🍳 生活类比

Context 像"公司广播"（谁都能听，但老广播同一件事会全员重渲染，不适合高频）；Redux 像"走 OA 审批流"（改个状态得填单→领导批→入库，规范但慢）；Zustand 像"公共白板"（谁路过谁写，直接拿，最省事）。

### 🔍 原理下沉

- Context：`createContext` + `Provider` + `useContext`。值变了，所有消费组件重渲染（无选择器细粒度），适合主题/用户信息等低频数据。
- Redux：单一 store，`dispatch(action)` → reducer 纯函数算新 state → 订阅组件更新。`redux-toolkit` 已大幅简化样板。
- Zustand：`const useStore = create(set => ({ count: 0, inc: () => set(s => ({ count: s.count + 1 })) }))`，组件里 `useStore(s => s.count)` 取，自带细粒度订阅。

### 🔁 对比学习（vs Pinia）

- Pinia ↔ Zustand：写法最接近，都是"一个 store 一个函数/对象"，直接取直接改。
- Pinia ↔ Redux：Redux 样板重、强调不可变 + 纯函数；Pinia 没这些负担。
- React Context ↔ provide/inject：都是跨层传值，不走 props 层层透。

```jsx
// Zustand：最像 Pinia 的体验
const useCounter = create((set) => ({
  count: 0,
  inc: () => set((s) => ({ count: s.count + 1 }))
}))
function App() {
  const count = useCounter((s) => s.count)
  const inc = useCounter((s) => s.inc)
  return <button onClick={inc}>{count}</button>
}
```

### 💥 我踩过的坑

- 用 Context 存高频变化的购物车数量，结果每次加减整棵消费树重渲染，列表卡。后来高频状态拆到 Zustand，低频配置留 Context。
- Redux 早期被"action type 满屏飞、reducer 拆文件"劝退；后来发现真正需要的全局状态没那么多，大部分用局部 state + 偶尔 Context 就够了。

### 🎯 面试官可能追问

- Context 为什么不适合高频更新？（无细粒度选择，值变全量重渲染）
- Redux 和 Zustand 的核心区别？（样板量 / 不可变约束 / API 心智）
- 怎么选 React 状态方案？（局部用 useState，跨层低频用 Context，真全局/高频用 Zustand/Redux）

---

### React vs Vue：心智模型差异（个人体会）（⭐️⭐️⭐️）

**一句话结论：** Vue 是"响应式拦截"——你改数据，框架自动知道哪要更新；React 是"显式状态 + 单向流"——你返回新状态，框架重新协调。我做了五年 Vue，学 React 最大的坎是：从"等框架帮我更新"变成"我得主动告诉框架状态变了"。

### 🍳 生活类比

Vue 像"声控灯"——你出声（改数据）灯自动亮（更新）；React 像"摁开关"——你得自己伸手摁（调 setState），灯才亮。习惯了声控再学摁开关，总忘了伸手。

### 🔍 原理下沉（个人体会）

- Vue：`data`/`ref` 改了，依赖它的地方自动更新，心智是"改了就行"，模板/响应式帮你兜。
- React：state 变了必须 `setState`/`setXxx`，且要返回新引用（不可变更新，如 `setList([...list, item])` 而非 `list.push`）。心智是"显式、不可变"。
- 副作用：Vue `watch` 自动追踪依赖；React `useEffect` 要手写 deps 数组，漏写就踩坑。
- 模板 vs JSX：Vue 模板约束多但编译期优化强；React JSX 灵活但要自己注意 key/表达式。

### 🔁 对比学习

```js
// Vue：改了就响应
const list = ref([])
list.value.push('a')   // Proxy 拦截，自动更新

// React：必须返回新引用
const [list, setList] = useState([])
setList([...list, 'a']) // 不 push，要新数组
```

### 💥 我踩过的坑（从 Vue 转 React）

- 最大的坑：直接 `list.push(x); setList(list)`——引用没变，React 认为"没变化"不更新。得 `setList([...list, x])`。Vue 里 `arr.push` 能被 Proxy 拦截，React 不会，思维惯性害我 debug 半天。
- 还有"忘了调 setState 直接改局部变量"，页面当然不动。Vue 的响应式让我养成"改了就好"的习惯，React 得改成"改了还得通知"。

### 🎯 面试官可能追问

- 你更熟 Vue，为什么还要学 React？（真实求职：岗位要求 React，且两种都懂面试更有底气；也是想补全武器库）
- Vue 的响应式更新和 React 的不可变更新，你更喜欢哪种？为什么？
- 如果让你用 React 重写一个 Vue 项目，最大的适配成本在哪？（状态更新思维、生命周期对照、模板→JSX）
