# 08 · TypeScript

> 前端工程化标配。重点在"为什么需要类型"和实战踩坑，而非语法罗列。

本章计划覆盖：
- [x] 泛型：从"盒子"类比讲通（含 extends 约束、keyof、默认类型、何时不用）
- [x] interface vs type：何时用哪个（含声明合并、AI 过度可选坑）
- [x] Utility Types（Partial/Pick/Omit/Record…）实战 + 真实踩坑
- [ ] 基础类型与类型推断（更细展开，可后续）
- [ ] 类型收窄专题（typeof / in / 判别联合，可后续补）

---

## 1. 泛型：一个"贴了标签的盒子"，装进去什么拿出来还是什么（⭐️）

**一句话结论**：泛型就是"把类型当参数传"——写一个函数/组件/类时先不写死具体类型，调用时才确定；它能让你复用同一份逻辑又不被 `any` 吃掉类型信息。

🍳 **生活类比**：快递打包。你做一个通用纸箱（函数），不规定里面必须装书还是装杯子；等真打包时塞进书，箱子上就贴"书"的标签，取件人拿到也知道是书，不会变成"一团不知啥"。`any` 相当于箱子不贴标签、里面啥都行——取件人打开一脸懵，TS 也不再帮你盯错了。

🔍 **原理下沉**：
- 核心是"类型参数 + 推断"。`function first<T>(items: T[]): T | undefined { return items[0] }`，调用 `first([1,2,3])` 时 TS 从参数推断出 `T = number`，返回的也是 `number | undefined`。你几乎不用手写 `first<number>(...)`，推断够用。
- 约束 `extends`：无约束的 T 什么都可以，所以函数体内用不了任何具体属性。需要某个属性时就约束：`function getLength<T extends { length: number }>(x: T): number { return x.length }`。`K extends keyof T` 是最有用的约束——它让"按 key 取值"的返回类型自动变成那个 key 对应的类型：`function pluck<T, K extends keyof T>(obj: T, key: K): T[K]`。
- 默认类型参数：`interface ApiResponse<T = unknown> { data: T; status: number }`，大多数情况用 unknown、特定时再传。
- `infer` 和条件类型：库里 `ReturnType`、`Awaited` 都是靠 `T extends (...args) => infer R ? R : never` 这种写法从已有类型"抠"出一部分。日常不用手写，但得认得。

🔁 **对比学习（泛型组件，Vue 和 React 并排）**：
```tsx
// React：函数组件直接挂泛型
function List<T>({ items, render }: { items: T[]; render: (it: T) => React.ReactNode }) {
  return <ul>{items.map((it, i) => <li key={i}>{render(it)}</li>)}</ul>
}
```
```ts
// Vue 3.3+：<script setup> 加 generic 属性
// <script setup lang="ts" generic="T">
// defineProps<{ items: T[]; render: (it: T) => VNode }>()
// 老版本没有 generic 时只能用 PropType<T[]> 兜底，类型体验差一截
```
两者都是"组件接收什么类型由使用处决定"，React 写在函数签名上、Vue 写在 script 标签的 `generic` 上。

💥 **我踩过的坑**：早期写个 `fetchJSON` 返回 `any`，调用处 `res.data.user.name` 一路点下去，后来后端把 `user` 改成了 `profile`，本地 TS 一声不吭（因为 any），上线才在用户页白屏。还有一次为了"省事"给工具函数加泛型，其实它就只处理 string，同事 review 说"你这泛型是装饰品"——能用具体类型就别硬上泛型，第二个类型才需要它。

🎯 **面试官可能追问**：泛型解决了什么问题？`K extends keyof T` 这种约束有什么用？什么时候不该用泛型？`infer` 是干嘛的？

---

## 2. interface 还是 type？官方一句话：能用 interface 就用，直到需要 type 才换（⭐️）

**一句话结论**：描述普通对象形状，两者几乎等价、随便选一个团队统一即可；真正分高下的是"只有 interface 能做"和"只有 type 能做"的几件事。

🔍 **原理下沉（差异清单）**：
- **声明合并（interface 独有）**：同一个 interface 名声明两次会自动合并。这是 DefinitelyTyped 给第三方库"打补丁"、给 `window` 加自定义属性的基础。type 重名会直接报 Duplicate identifier。
- **联合 / 交叉 / 映射 / 元组 / 原始类型（type 独有）**：`type Status = 'pending' | 'done'`、`type Point = [number, number]`、`type Readonly<T> = { readonly [K in keyof T]: T[K] }` 这些 interface 写不了，只能用 type。TS 内置的 Partial/Omit/Pick 全是 type 实现的。
- **冲突处理不同**：`interface B extends A` 若属性不兼容会直接报错；`type C = A & B` 冲突属性会静默变成 `never`，排查时很隐蔽。
- **报错可读性**：interface 的错误信息更干净；type 在复杂嵌套里会被内联成一长串。

所以官方建议是"Use interface until you need type"——领域模型（User/Order）、类要 `implements`、可能要被库扩展的公开 API 用 interface；联合、交叉组合、工具类型、非纯对象（原始/元组/函数）用 type。

💥 **我踩过的坑（也是 AI 生成代码的通病）**：让 AI 帮我写接口，它怕报错就把每个字段都标成可选 `?`——结果空对象 `{}` 都能通过类型检查，类型几乎不提供保护。正确做法是逐个问"这字段真可能不存在吗？"，该必填就必填。另一个：把全是对象的 interface 转成 type 之后，某个依赖它的 `.d.ts` 原本靠声明合并补字段，一改就报 Duplicate identifier——合并这事儿只有 interface 兜得住。

🎯 **面试官可能追问**：interface 和 type 最大区别是什么？声明合并有什么用？为什么 AI 生成的接口常常"太宽松"？

---

## 3. Utility Types：站在 TS 内置"乐高"上拼，别手搓（⭐️）

**一句话结论**：TS 自带一堆"以类型造类型"的工具（Partial / Required / Readonly / Pick / Omit / Record / ReturnType / Parameters…），日常 90% 的派生类型不用自己写，认得、用对就行。

🔍 **原理下沉（最常碰的五个）**：
- `Partial<T>`：所有属性变可选——做"部分更新"接口时天然合适：`updateUser(id, patch: Partial<User>)`。
- `Pick<T, K>` / `Omit<T, K>`：从一个大类型里"挑"或"剔"字段。列表项只展示 id+name 就用 `Pick<User, 'id'|'name'>`；返回给前端别带 password 就用 `Omit<User, 'password'>`。
- `Record<K, V>`：快速造"键为 K、值为 V"的映射表，比如 `Record<string, number>` 当字典。
- `ReturnType<F>` / `Parameters<F>`：从一个函数类型反推它的返回/参数，重构时少写一遍重复签名。

🔁 **对比学习（一个 update 接口，三种写法）**：
```ts
interface User { id: string; name: string; email: string; password: string }

// 手搓：改一个得跟着改三处，容易漏
interface UserPatch { name?: string; email?: string }

// 用内置：永远跟着 User 走
function updateUser(id: string, patch: Partial<User>) {}
type PublicUser = Omit<User, 'password'>
```
手搓版的问题是 User 加了字段，UserPatch 不会自动同步；用 Partial/Omit 则永远和源类型一致。

💥 **我踩过的坑**：`any` 逃逸最阴——某次后端返回没标类型，我 `const list = res.data as any[]`，`.map` 里把字段名写错，TS 不拦，测试环境数据正好有那个错字段名没暴露，上线真实数据才炸。后来规矩：接口边界（fetch 返回值、JSON.parse 结果）必须立刻定一个 interface 或 type 兜住，绝不让 any 溜进业务层。还有 Record 当字典时忘了值可能是 undefined，取值直接当存在用，运行时空指针。

🎯 **面试官可能追问**：Partial 和 Omit 区别？怎么从函数类型拿到它的返回类型？为什么接口边界不能放 any？

---

泛型解决"一份逻辑通吃多种类型"，interface/type 解决"怎么把形状说清楚"，Utility Types 解决"在已有类型上做加减"。三个合起来，TS 才从"注释"变成真护城河。
