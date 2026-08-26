# 04 · CSS

> CSS 是我自评 ★★★★★ 的强项，也是面试最容易拉开细节差距的地方。很多候选人能说出 `flex: 1` 怎么写，但被追问"BFC 为什么能清除浮动""box-sizing 改了之后 width 到底算什么"，就露怯了。

本章计划覆盖（逐步补充，欢迎提 Issue 认领）：
- [x] 盒模型：标准/IE 盒模型、`box-sizing`
- [x] 水平垂直居中的 N 种写法（对比适用场景）
- [x] BFC：是什么、怎么触发、解决什么问题
- [x] 层叠上下文与 `z-index` 陷阱
- [x] Flex 布局核心（与 Grid 的取舍）
- [x] 响应式：媒体查询 / 移动端适配（rem / vw）

可大量用生活类比（比如 BFC 像"建一堵墙隔开两个房间"）。

---

## 盒模型：标准盒 vs IE 盒（⭐️⭐️）

**一句话结论：** CSS 把每个元素当成一个矩形盒子，由 content / padding / border / margin 四层组成；最容易被坑的是 `width` 到底包不包括 padding 和 border——标准盒（content-box）**不包括**，IE 盒（border-box）**包括**。

### 🍳 生活类比

把这事儿想成**寄快递**：
- `content` 是你要寄的那件真东西；
- `padding` 是裹在东西外面的气泡膜；
- `border` 是外面的纸箱；
- `margin` 是两个包裹之间的空隙——它**不属于任何一个箱子**，是"箱子之外的世界"。

所谓盒模型之争，其实就是一句问话：**你说的"箱子宽 100"，包不包括气泡膜和纸箱？**

- **标准盒（content-box）**：你说"箱子宽 100"，结果气泡膜和纸箱另算，拿到手实际比 100 大。设了 `width:100px; padding:20px`，元素实际占 **140px**。
- **IE 盒（border-box）**：你说"我就要 100 的成品尺寸"，气泡膜和纸箱都从这 100 里扣，`content` 被压缩。设了同样的，`content` 只剩 60px，但元素整体还是 **100px**。

### 🔍 原理下沉

`box-sizing` 默认值是 `content-box`，所以很多人一加 padding 布局就乱——他以为 width 就是最终宽度，其实不是。这也是为什么几乎所有现代项目的第一句 CSS 都是：

```css
*, *::before, *::after {
  box-sizing: border-box;
}
```

一行下去，`width` 就是你"眼睛看到的宽度"，算布局不用再心算加减。Bootstrap、Tailwind 的重置里都有它。

版本演进：`box-sizing` 是 CSS3 才标准化的属性，老 IE（怪异模式）天生就是 border-box 行为，所以"IE 盒模型"这个名字其实是历史包袱留下的叫法，不是 IE 的专利。

### 🔁 对比学习（content-box vs border-box）

```css
.box-a { box-sizing: content-box; width: 100px; padding: 20px; } /* 实际 140px */
.box-b { box-sizing: border-box;  width: 100px; padding: 20px; } /* 实际 100px */
```

做两栏布局时这差别是致命的：左边 `width:50%` 再加 padding，右边也 `50%`，加起来就超 100% 被挤到下一行——换成 border-box 就老老实实各占一半。

### 💥 我踩过的坑

第一次做两栏，左边 `width:50% + padding:10px`，右边 `width:50%`，怎么都换行。查了半天才知道是盒模型——50% 不算 padding，合计 100%+20px 超了。那之后我所有项目开头必写全局 `border-box`，再没为这事儿 debug 过。

### 🎯 面试官可能追问

- `box-sizing` 能不能继承？怎么让全站统一？（`inherit` 或全局重置）
- 一个 `border-box` 元素 `width:100px; border:10px; padding:20px`，content 实际多宽？（60px）
- `margin` 算在盒子"内部"还是"外部"？（外部，不计入元素自身尺寸）

---

## 水平垂直居中：N 种写法怎么选（⭐️⭐️⭐️）

**一句话结论：** 没有"唯一正确"的居中法，只有"看场景选"。已知尺寸用负 margin 或绝对定位，未知尺寸且是现代浏览器——**flex / grid 一行搞定**。

### 🍳 生活类比

把一幅画挂到一面墙的正中间。墙是父容器，画是子元素。难点永远是：画的大小可能变，墙的大小也可能变，你总不能每次拿尺子量了再钉钉子吧？

### 🔍 原理下沉

按"现代程度"排一下：

```css
/* 1. Flex 一行（最常用，未知尺寸也能用） */
.parent { display: flex; justify-content: center; align-items: center; }

/* 2. Grid 一行（比 flex 还短） */
.parent { display: grid; place-items: center; }

/* 3. 绝对定位 + transform（未知尺寸的救星，老项目也能用） */
.child {
  position: absolute; top: 50%; left: 50%;
  transform: translate(-50%, -50%);
}

/* 4. 绝对定位 + 负 margin（已知尺寸才方便） */
.child {
  position: absolute; top: 50%; left: 50%;
  width: 100px; height: 100px;
  margin: -50px 0 0 -50px;
}

/* 5. table-cell（上古写法，了解即可） */
.parent { display: table-cell; vertical-align: middle; text-align: center; }
```

第 3 种 `translate(-50%, -50%)` 妙在它是**相对自身尺寸**位移，所以你不知道元素多大也能居中——这正是它比第 4 种负 margin 强的地方（负 margin 得先知道宽高）。

flex / grid 的居中之所以爽，是因为它们把"对齐"做成了**一等公民属性**，浏览器自己算，你不用管尺寸。`transform` 居中走的是合成层，一般不触发重排，性能也好。

### 🔁 对比学习（老方案 vs 新方案）

| 场景 | 推荐 | 不推荐 |
|------|------|--------|
| 现代浏览器、尺寸未知 | flex `place-items` / grid | 负 margin（得先量尺寸） |
| 要兼容 IE10- | table-cell / 绝对定位 | flex（IE 部分支持但坑多） |
| 单行文本垂直居中 | `line-height` 等于高 | 大炮打蚊子上 flex |

### 💥 我踩过的坑

当年想让一个 `div` 垂直居中，写 `vertical-align: middle` 死活不对。后来才懂：`vertical-align` 只对 **inline / inline-block / table-cell** 生效，块级元素根本不理它。块级元素想垂直居中，上 flex 或上定位，别跟 `vertical-align` 较劲。

### 🎯 面试官可能追问

- 为什么 `transform: translate` 做居中性价比高？（相对自身、不依赖尺寸、不重排）
- flex 居中 vs grid 居中选哪个？（单元素/一维用 flex 顺手，整页骨架用 grid）
- `justify-content` 和 `align-items` 分别管哪个轴？

---

## BFC：建一堵墙，内外互不干扰（⭐️⭐️）

**一句话结论：** BFC（Block Formatting Context，块级格式化上下文）是一块**独立的渲染区域**——里面的布局影响不到外面，外面的也别想影响里面。三大实用场景：包住浮动元素（清除浮动）、阻止 margin 塌陷、做两栏自适应。

### 🍳 生活类比

BFC 就像在房间里**建一堵墙**。墙内怎么折腾（元素浮动、margin 乱跳），墙外的东西都不受干扰；墙外发洪水（外层的 margin），也淹不进墙里。

### 🔍 原理下沉

哪些操作会触发 BFC（列几个常用的）：
- `overflow` 不为 `visible`（比如 `hidden` / `auto`）
- `float` 不为 `none`
- `position` 为 `absolute` / `fixed`
- `display` 为 `flex` / `grid` / `inline-block`
- `display: flow-root`（**最干净**，专为 BFC 设计，但老浏览器不支持）

两个经典问题靠它解决：

**① margin 塌陷。** 两个相邻块级元素的上下 margin 会"合并"成较大的那个。给其中一个套一层 BFC，它们就各过各的、不再合并——因为 BFC 内部是独立上下文。

**② 清除浮动。** 父元素不写高度、子元素全浮动时，父会"塌"成 0 高（因为浮动元素脱离了文档流）。给父加 `overflow: hidden` 触发 BFC，父就会把浮动的子"包"进来计算高度，塌掉的布局就好了。

### 🔁 对比学习（清除浮动的几种办法）

```css
/* 老办法：额外标签，丑但兼容好 */
.clear { clear: both; }

/* 常见办法：overflow 触发 BFC，但会裁剪溢出内容 */
.parent { overflow: hidden; }

/* 现代最优：flow-root，专为 BFC，不裁剪 */
.parent { display: flow-root; }
```

`overflow: hidden` 的副作用是会把超出的内容切掉（比如下拉菜单、tooltip 被截），所以我现在更偏向 `flow-root`。

### 💥 我踩过的坑

最早清除浮动只知道往 HTML 里插 `<div style="clear:both">`，结构脏得不行。知道 overflow:hidden 一行能搞定后如获至宝，结果有次下拉菜单被切了一半——就是 `overflow: hidden` 的锅。换成 `flow-root` 才两全。

### 🎯 面试官可能追问

- BFC 和 IFC（行内格式化上下文）区别？
- 为什么 `overflow:hidden` 能清除浮动却会裁剪内容？
- `display:flow-root` 的浏览器兼容性如何？

---

## Flex vs Grid：一维还是二维（⭐️⭐️）

**一句话结论：** Flex 是**一维**布局（元素只能沿一条线——横或竖——排），Grid 是**二维**布局（行和列同时管）。做导航栏、列表用 Flex；做整页骨架、卡片网格用 Grid。

### 🍳 生活类比

Flex 像一条**传送带**，东西只能顺着带子一个方向排；Grid 像一张**棋盘**，横竖都有格子，落子在哪儿你说了算。你不能指望一条传送带把货既横着又竖着摆——那是棋盘的活儿。

### 🔍 原理下沉

- **Flex**：靠"主轴 / 交叉轴"两个概念，适合"一组元素沿一个方向分布"——贴边、居中、把剩余空间均分。响应式很友好，但要排二维就得不断嵌套 flex，越套越乱。
- **Grid**：用 `grid-template-columns` / `grid-template-rows` 显式画出网格，适合"既要管列又要管行"。`fr` 单位按比例瓜分空间，`repeat()` / `minmax()` 能写得很优雅。
- 两者都不脱离文档流（不像 float 那样需要专门清除），这是它们能取代 float 布局的根本原因。

### 🔁 对比学习（什么时候用哪个）

| 需求 | 选 | 理由 |
|------|----|------|
| 顶部导航栏、按钮组、一行标签 | Flex | 天然一维，贴边/居中一行属性搞定 |
| 三栏布局（左固定 右固定 中自适应） | Grid | `grid-template-columns: 200px 1fr 200px` 一行完事 |
| 相册、卡片墙、仪表盘 | Grid | 行列同时约束，Flex 得嵌套才勉强 |
| 一个列表项内部再左右分布 | Flex 套 Grid / 反过来 | 各管一维，混用不冲突 |

### 💥 我踩过的坑

有次把整个后台页面用 Flex 嵌套搞出三栏，中间内容区的高度对齐折腾了半天——因为 Flex 一维，纵向对齐得额外绕。后来换成 Grid，一行 `grid-template-columns: 200px 1fr 200px` 直接落定。教训很直接：**二维的事别硬用 Flex 扛**。

### 🎯 面试官可能追问

- `flex: 1` 是哪三个属性的缩写？（`flex-grow` / `flex-shrink` / `flex-basis`）
- Grid 的 `auto-fill` 和 `auto-fit` 区别？（空轨道留不留）
- 什么时候 Flex 比 Grid 更合适？（纯一维分布，Grid 反而啰嗦）

---

---

### 层叠上下文与 z-index 陷阱（⭐️⭐️⭐️）

**一句话结论：** z-index 不是"全局比大小"，它只在**同一个层叠上下文**里排座次；一旦某元素创建了层叠上下文（设了 opacity<1、transform、filter、position+z-index 等），它的子元素 z-index 再大也飞不出这个"天花板"。设了 z-index:9999 还压不住，多半是祖先偷偷建了层叠上下文、或对手身处层级更高的另一个上下文。

### 🍳 生活类比

层叠上下文像一栋楼里的**楼层**，z-index 是你在**这层楼**的座位号。你座位号再大（9999），也出不了这层楼——要是你俩不在同一栋楼（不同层叠上下文），比的就不是同一个号码池。更坑的：你爸在 3 楼开了公司（父元素创建了层叠上下文），你在里面座位号 9999，但整栋 3 楼在"大楼排名"里可能比 2 楼低，那你再大也盖不过 2 楼的人。

### 🔍 原理下沉

- 根元素天生一个根层叠上下文。
- 哪些会创建新的层叠上下文（常用）：
  - `position` 为 relative/absolute 且 `z-index` 不为 auto
  - `position: fixed` / `sticky`
  - `opacity < 1`
  - `transform` / `filter` / `perspective` / `clip-path` / `mask` 不为 none
  - `will-change` 设了上述属性
  - flex/grid 容器的子项设了 `z-index` 不为 auto
  - `isolation: isolate`（主动隔离，最干净）
- 比较规则：同一上下文内按 z-index 比；不同上下文，先比"上下文自身的层级"（上下文像整体参与父上下文排名），内部子项不和外层别的元素直接比。
- "设了 z-index:9999 压不住"的三大常见原因：
  1. 祖先某层创建了层叠上下文，且那个祖先整体排在对手之下 → 你再大也白搭；
  2. 自己没设 `position`（z-index 对 static 无效）；
  3. 对手在层级更高的另一个层叠上下文里，比的是上下文层级不是数字。

### 🔁 对比学习（常见坑）

```css
.modal-mask { position: relative; z-index: 1; }            /* 创建层叠上下文，层级 1 */
.modal-mask .dialog { position: absolute; z-index: 9999; } /* 再大也飞不出 mask */
/* 若页面另一个 .sidebar 层级 2 且在更外层，它的内容会盖住整个 mask 及其 9999 弹窗 */
```
解法：用 `isolation: isolate` 给需要独立比价的区域单独建上下文，或调整祖先的 z-index / 结构，别让中间层"封顶"。

### 💥 我踩过的坑

做弹窗时，弹窗里下拉框 z-index 设很大还是被页面轮播图压住。查半天才发现：弹窗父容器（一个用 transform 做动画的 wrapper）悄悄创建了层叠上下文，而轮播图在另一个更外层、层级更高的上下文里。最后把弹窗挂到 body 下、或给关键层加 `isolation` 才解决。从此写弹窗前先想"它爸有没有建层叠上下文"。

### 🎯 面试官可能追问

- 为什么 z-index 对 position: static 无效？（static 不参与层叠上下文 z-index 排序）
- opacity:0.5 为什么影响层叠？（半透明会创建层叠上下文）
- 怎么"干净地"创建层叠上下文隔离？（isolation: isolate）
- 移动端 1px 边框和层叠上下文有关吗？（无关，那是 dpr + transform scale 的坑，但常被一起问）

---

### 响应式：媒体查询 / 移动端适配（rem / vw）（⭐️⭐️）

**一句话结论：** 响应式不是"写一套手机版"，而是用一套代码在不同视口下自适应。媒体查询（@media）按断点切换样式；移动端适配更多用 rem（配合 JS 设根字号）或 vw（视口单位）让尺寸随屏幕缩放。

### 🍳 生活类比

响应式像"同一件衣服，胖人穿自动变宽松版型、瘦人穿变修身版型"——不是做两件，而是版型跟着身材（视口）变。媒体查询是"到某尺码换版型"，rem/vw 是"尺码按比例自动缩放"。

### 🔍 原理下沉

- 媒体查询：`@media (max-width: 768px) { ... }` 视口 ≤768 生效。常用断点：移动 <768、平板 768–1024、桌面 >1024（无铁律，看设计稿与内容）。
- rem：根字号 `html { font-size: 16px }`，`1rem = 16px`。移动端经典做法：JS 监听 resize，按屏宽设 `html` font-size（如 `fontSize = clientWidth / 10`），设计图 375 宽时 1rem=37.5px，元素按 设计图px / 37.5 写 rem，整体随屏缩放——这就是 flexible.js / postcss-pxtorem 思路。
- vw/vh：1vw = 视口宽 1%。可直接 `width: 50vw`，无需 JS。但 vw 不考虑最大宽度，大屏字会过大，常配 `max-width` 或 `clamp()` 兜底。
- 现代更省事：`clamp(min, preferred, max)` 做流式排版，如 `font-size: clamp(14px, 2vw, 20px)`——最小14、最大20、中间随 2vw 走，一套顶三套。

### 🔁 对比学习（rem vs vw vs 媒体查询）

| 方式 | 机制 | 优点 | 坑 |
|------|------|------|-----|
| 媒体查询 | 断点切换整块样式 | 控制精确、语义清晰 | 断点间突变、要写多套 |
| rem + JS | 根字号随屏变，元素按 rem | 平滑缩放、还原设计稿 | 要引 JS、首屏可能闪 |
| vw/vh | 视口单位直接缩放 | 纯 CSS、无 JS | 大屏失控、需 clamp/max-width 兜底 |

### 💥 我踩过的坑

- 早先用媒体查询做移动端，只在 768 断点写一堆样式，结果平板（800px）刚好卡在断点外，布局奇丑。后来明白断点要按"内容什么时候开始挤"来定，不是死磕设备宽度。
- rem 适配忘了限制根字号上限，超宽显示器上字大得离谱。加 `fontSize = Math.min(clientWidth, 750) / 10` 才收住。
- 移动端 1px 边框：直接 `border:1px` 在高 dpr 屏显粗，用 `transform: scaleY(0.5)` 或伪元素 + transform 解决——这是 dpr 的坑，和 rem/vw 同一话题但不同因。

### 🎯 面试官可能追问

- rem 适配原理？为什么用 JS 设根字号？（让 1rem 随屏宽等比，元素按设计图比例缩放）
- vw 和 % 的区别？（vw 相对视口，% 相对父元素）
- 移动端 1px 边框怎么解决？（dpr + transform scale / 0.5px / 伪元素）
- `clamp()` 在响应式怎么用？（min, preferred, max，做流式排版）
