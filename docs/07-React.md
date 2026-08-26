# 07 · React

> 我在学、边写边复习的一章。会刻意和 Vue 做对比——这恰好是我真实的学习方式。

本章计划覆盖：
- [ ] JSX 本质（与 Vue 模板对比）
- [ ] Hooks：`useState` / `useEffect` / `useMemo` / `useCallback`
- [ ] 受控组件 vs 非受控组件
- [ ]  reconciliation 与 Fiber（渲染机制）
- [ ] 状态管理：Context / Redux / Zustand（与 Pinia 对比）
- [ ] React vs Vue：心智模型差异（个人体会）

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
