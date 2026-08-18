# 04 · CSS

> CSS 是我自评 ★★★★★ 的强项，也是面试最容易拉开细节差距的地方。很多候选人能说出 `flex: 1` 怎么写，但被追问"BFC 为什么能清除浮动""box-sizing 改了之后 width 到底算什么"，就露怯了。

本章计划覆盖（逐步补充，欢迎提 Issue 认领）：
- [x] 盒模型：标准/IE 盒模型、`box-sizing`
- [x] 水平垂直居中的 N 种写法（对比适用场景）
- [x] BFC：是什么、怎么触发、解决什么问题
- [ ] 层叠上下文与 `z-index` 陷阱
- [x] Flex 布局核心（与 Grid 的取舍）
- [ ] 响应式：媒体查询 / 移动端适配（rem / vw）

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

> 下一题预告：层叠上下文与 `z-index` 陷阱（为什么你设了 `z-index:9999` 还是压不住别人）。
