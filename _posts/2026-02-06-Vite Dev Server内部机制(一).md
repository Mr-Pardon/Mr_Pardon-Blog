---
title: Vite Dev Server 内部机制（一）：模块处理管线
date: 2026-02-06 10:00:00 +0800
categories: [前端构建体系]
tags: [构建体系, vite, webpack]
author: <Mr_Pardon>
---

## 引言：从“运行时模型”走向“内部实现”

在上一篇文章中我们已经建立起了Vite Dev Server运行时模型的整体认知：

> 浏览器驱动模块加载  
> Dev Server 按请求即时转换源码  
> 构建行为被拆散进一次次模块请求之中

从本篇文章开始，我们将深入Vite Dev Server内部机制，去探究一个更基本的问题:

> **当浏览器请求一个模块时，Vite Dev Server内部发生了什么**

这一过程并非是简单的从“读取”到直接“编译转换”，而是一条**由插件系统、模块解析机制与缓存机制共同协作构成的“模块处理管线”(TransfromPipeline)**。完整的理解这一过程，需要从浏览器请求一个模块开始，逐步分析Vite Dev Server内部的每个组件是如何协同工作的。

---

## 一次模块请求，是如何进入“模块处理管线”的？

在正式拆解模块处理管线之前，我们需要先简单了解Vite Dev Server的请求处理机制:

> **对于浏览器发出的每一个请求，并不是直接进入到模块处理管线的，而是会先经过 Dev Server 的 HTTP 中间件体系。**

Vite Dev Server 本质上是一个基于 Node.js 的 Web 服务器。浏览器请求任意资源时，请求会先进入一条由多个中间件组成的处理链。
```js
middlewares.use(timeMiddleware)
middlewares.use(corsMiddleware)
...
// 模块转换中间件
middlewares.use(transformMiddleware)
...
```
这条链的职责并不是做源码转换，而是：

> **判断这个请求“是什么类型的请求”，再决定把它交给谁去处理。**

我们可以将这一层理解为 Dev Server 的「请求调度层」。

```
浏览器请求
    ↓
Connect 中间件管线（请求调度层）
    ↓
不同类型的资源进入不同的处理逻辑
```

例如：

| 请求类型                           | 处理方式                      |
| ---------------------------------- | ----------------------------- |
| HTML 文档                          | 注入 HMR 客户端、处理入口脚本 |
| 静态资源（图片、字体等）           | 直接按文件返回                |
| 模块请求（JS / TS / Vue / CSS 等） | 进入模块转换流程              |

也就是说：

> **只有当请求被识别为“模块请求”时，才会真正进入我们本篇文章要讨论的核心 —— 模块处理管线。**

这个识别与分流的过程，是通过中间件内部对 URL、后缀名以及请求参数的判断来完成的。当某个中间件确认：

> “这是一个需要进行源码转换的模块请求”

它就会将该请求正式交给 Vite 内部的模块处理系统。

```
HTTP 请求
   ↓
中间件识别为模块请求
   ↓
进入模块转换中间件
   ↓
调用 Plugin Container
   ↓
resolveId → load → transform
```

从这里开始，请求就不再只是一次网络资源获取，而是进入了一条由插件系统驱动的 **源码加工流水线**。而这条流水线，正是 Vite 在开发阶段能够“按需构建”的核心所在。

---

## 模块处理管线：ResolveId / Load / Transform

当一个请求被识别为“模块请求”之后，它便正式进入到 Vite Dev Server 内部真正的核心流程，**TransformPipeline**。

这条流水线由三个首尾相接的核心阶段构成：

```
ResolveId -> Load -> Transform
```

其分别从三个层面逐层深入：

| 阶段      | 解决的问题                                 | 本质职责     |
| --------- | ------------------------------------------ | ------------ |
| resolveId | “这个模块究竟是谁？”                       | 模块身份识别 |
| load      | “这个模块的内容从哪里来？”                 | 模块内容获取 |
| transform | “这个模块需要如何改写后才能被浏览器执行？” | 模块源码加工 |

理解这三步的分工，是理解 Vite Dev Server 如何在**不打包的前提下完成模块系统运转**的关键。

---

### ResolveId: 模块身份识别

当模块请求正式进入到这条“处理管线”之后，最优先需要处理的问题就是进行“身份识别”：

> **当前这个模块请求，在工程系统中到底对应着谁？**

对于浏览器发起的请求，Vite Dev Server接收到的是一个`URL`形式的路径，例如:

```
/src/main.js
/@id/vue
/virtual:generated-module
```

这个路径在真实项目中可能指向：

- 一个磁盘上的源码文件  
- 一个经过预构建缓存的依赖模块  
- 某个插件生成的虚拟模块

因此在进入到后续处理之前，Vite必须先为这个模块确认一个“内部身份”，这一步正是由插件系统的`resolveId`钩子负责。

请求路径会**逐一有序的通过每个插件的**`resolveId`**钩子**，当某个插件成功识别出了该模块的身份后，意味着其**声明了对该模块的“解析结果”，后续插件将基于这一结果继续工作，而不会再尝试重新解析模块身份。** 如果没有插件认领，Vite 才会回退到默认的文件系统解析逻辑。

例如当浏览器发起 `/src/main.ts` 请求时，插件系统会逐一调用每个插件的 `resolveId` 钩子：

```js
resolveId('/src/main.ts', importer = undefined)
```

当某个插件或者Vite自身成功处理了这个请求，则会返回一个统一规范化的模块 ID，意味着它已经确认了模块的身份。

```js
return { id: '/abs/path/src/main.ts', meta: { ... } }
```

> 需特别注意，这个`id`仅在Vite内部维护，用于识别浏览器请求的模块，与最终`transform`阶段改写和返回给浏览器的模块路径并非同一语义。
> 因此，每当浏览器发起一次新的模块请求时，无论该路径是否已经被 Vite 改写，Vite 都会重新通过 resolveId 阶段为其确认内部模块身份。

在这一阶段，插件可以:

- 规范化模块路径
- 将请求重定向到其他实际位置
- 声明一个虚拟模块并“认领”它
- 为该模块附加后续处理所需的元信息

其核心目的在于告诉 Vite Dev Server：

> **“这个模块是谁，以及它将以什么身份参与接下来的处理。”**

当模块的身份被确认后，它才会进入下一步。

---

### Load: 模块内容获取

当一个模块的“内部身份”被确认之后，Vite Dev Server需要解决的下一个问题，就是获取到该模块的“内容”。这一步骤正是由处理管线中的`Load`阶段负责。

从职责上来看，`Load`阶段并不关心模块如何被转换、是否需要改写，它只专注于一件事情：

> **根据已经确认的模块 ID，获取该模块的原始内容。**

这份“内容”可以来自于多个源：

- 磁盘上的真实文件
- 依赖预构建产物（如 `.vite/deps` 中的文件）
- 插件动态生成的虚拟模块
- 内存中的缓存结果

与`ResolveId`类似，`Load`阶段也是一个**可被插件接管的阶段**，这也意味着插件可以**直接决定为这个身份的模块发放什么内容**。

当模块进入到`Load`阶段时，Vite仍会逐一有序地调用每一个插件的`load`钩子，传入`ResolveId`阶段所确认的模块 ID：

```ts
load(id)
```

在这一阶段，插件可以自行选择是否接管该模块的内容获取：

- **返回字符串或 `{ code, map }`**
  - 表示：**插件已为该模块提供了完整的原始内容**
  - 该内容将作为后续 `transform` 阶段的输入
- **返回 `null` 或 `undefined`**
  - 表示：**插件不处理该模块**
  - 请求将继续交由后续插件，或最终回落到 Vite 的默认加载逻辑

一旦某个插件返回了有效结果：

> **该模块的内容来源就被确认，后续插件将不再参与内容加载。**

---

对于绝大多数普通模块而言，它们并不会被插件所特殊处理，而是直接回落到Vite默认的`load`行为：

> **根据模块 ID，从磁盘中读取对应的文件内容。**

举个简单的例子:

```ts
// 请求路径
/src/main.ts

// resolveId 阶段返回的内部 ID
/abs/path/project/src/main.ts
```

在`load`阶段，Vite会直接根据这个内部 ID，从文件系统中读取对应的文件内容：

```ts
import { createApp } from 'vue'
import App from './App.vue'

createApp(App).mount('#app')
```

此时得到的内容仍然是：

- TypeScript 语法
- 包含裸模块引用
- 可能引用 Vue SFC、CSS 等浏览器无法直接执行的资源

---

#### 虚拟模块：Load 阶段的“非常规能力”

`Load` 阶段真正体现其灵活性的地方，在于它并不要求模块内容一定来自文件系统。
只要某个模块在 `resolveId` 阶段已经被确认了“身份”，那么在 `load` 阶段：

> **这个模块是否存在于磁盘上，其实已经不再重要了。**

这正是 **虚拟模块（Virtual Module）** 能够成立的基础。

以`Vue SFC`为例，当浏览器发起了一次`.vue`组件的请求:

```ts
GET /src/App.vue
```

这个模块请求一样会先经历三个阶段的处理：

- 在`ResolveId`阶段被`Vue`插件进行认领
- 在`Load`阶段返回`.vue`文件的原始文本内容
- 在`Transform`阶段将由`Vue`插件对`SFC`进行解析、拆解与编译

关键在于`Transform`阶段，`Vue`插件会解析 `.vue` 文件的内容，并将其**拆解为多个逻辑部分**。但需要特别注意的是：

> **transform 并不会直接返回所有编译后的代码。**

相反，它会返回一个“入口模块”，其形式大致如下（示意）：

```js
import script from '/src/App.vue?vue&type=script'
import template from '/src/App.vue?vue&type=template'
import style from '/src/App.vue?vue&type=style'

script.render = template
export default script
```

这里有一个非常关键的点：

- `?vue&type=script`
- `?vue&type=template`
- `?vue&type=style`

**这些路径并不存在于磁盘上。**

它们是由`Vue`插件在`Transform`阶段“声明出来的虚拟模块”。
当这段代码被返回给浏览器后，浏览器会像对待普通ESM一样，**继续请求这些模块，进入到下一轮请求**，此时`Load`阶段的“非常规能力”开始体现。

当浏览器发起对这些虚拟模块的请求：

```ts
GET /src/App.vue?vue&type=script
GET /src/App.vue?vue&type=template
GET /src/App.vue?vue&type=style
```

这些带有query的模块请求进入到模块处理管线时:

- Vue 插件会在 `resolveId` 阶段认领这些模块身份
- 并在 `load` 钩子中，根据 `type` 不同，返回对应的内容：

```js
load(id) {
  if (id.includes('type=script')) {
    return compiledScriptCode
  }
  if (id.includes('type=template')) {
    return compiledRenderFunction
  }
  if (id.includes('type=style')) {
    return compiledCSS
  }
}
```

至此，对于浏览器而言，**它只是加载了一组普通的 ESM 模块**。而对 Vite 而言，**一个 `.vue` 文件被自然拆分进了多次模块请求之中**

---

#### 从 Vue SFC 看清 Load 阶段的真正定位

从 Vue SFC 这个例子中，我们可以清楚地看到：

- `transform`：声明模块结构，决定“需要哪些模块”
- `resolveId`：确认这些“看似不存在”的模块**是谁**
- `load`：为这些模块**真正发放内容**

因此，`Load` 阶段的本质并不是“读文件”，而是：

> **为已经确认身份的模块，提供其对应的原始内容来源。**

这也是 Vite 能够在开发阶段，将复杂的工程结构拆解为**一系列按需发生的模块请求**的关键原因之一。

---

### Transform: 模块语义改写与依赖显性化

当一个模块顺利完成了 `resolveId` 与 `load` 两个阶段之后，Vite Dev Server 已经明确了两件事：

- **这个模块是谁**（内部唯一 ID）
- **这个模块的原始内容是什么**（字符串形式的源码）

接下来要解决的，才是整个模块处理管线中最核心的一步：

> **如何把“并不一定能被浏览器直接执行的源码”，转换为浏览器可以理解、并继续发起依赖请求的 ESM 模块。**

这一步，正是由 `Transform` 阶段所负责。从设计职责上来看，其仅负责做一件事情：

> **基于模块的原始内容，对其进行语义层面的转换与改写。**

这种“改写”主要体现在以下几个方面：

- 将 **浏览器无法直接执行的语法** 转换为可执行形式  
  - TypeScript → JavaScript  
  - JSX → JavaScript  
- 将 **隐式的依赖关系显性化**
  - 拆分出虚拟模块
  - 注入额外的 `import` 语句
- 对代码进行 **开发期增强**
  - HMR 相关逻辑
  - 依赖追踪信息
  - Source Map 注入

与 `resolveId`、`load` 类似，`transform` 同样是一个**插件驱动的阶段**。

当模块进入 `Transform` 阶段时，Vite 会按照插件顺序，依次调用每个插件的 `transform` 钩子：

```ts
transform(code, id)
```

这里的两个参数非常关键：

- `code`：来自 `load` 阶段的原始模块内容
- `id`：已经规范化、唯一确定的模块 ID

插件在这个阶段可以选择：

- **返回转换后的代码（或 `{ code, map }`）**
  - 表示：插件对该模块进行了语义层面的处理
  - 返回结果会作为下一个插件的输入
- **返回 `null`**
  - 表示：插件不关心该模块
  - 内容将原样传递给下一个插件

需要特别注意的是：

> **与 `resolveId` / `load` 不同，`transform` 并不是“命中即终止”的阶段。**

多个插件可以**依次对同一个模块进行多轮改写**，最终形成浏览器真正执行的代码。

---

看一个最直观的`Transform`示例，假设我们有这样一个入口模块：

```ts
// src/main.ts
import { createApp } from 'vue'
import App from './App.vue'

createApp(App).mount('#app')
```

在 `Load` 阶段结束后，`transform` 接收到的内容仍然是：

- TypeScript 语法
- 包含裸模块引用 `vue`
- 引用了 `.vue` 这种浏览器无法理解的资源

此时，多个插件与 Vite 内置的能力会依次介入：

1. **TypeScript / ESBuild 插件**
   - 将 TS 转换为 JS
2. **Import Analysis（Vite 内置）**
   - 扫描 `import` 语句
   - 规范化并改写依赖路径
   - 收集模块依赖关系

最终返回给浏览器的代码，已经是一个**经过多轮处理（插件 + 内置逻辑）后的 ESM 模块**。

```js
import { createApp } from '/node_modules/.vite/deps/vue.js?v=hash'
import App from '/src/App.vue'

createApp(App).mount('#app')

```
---

#### Transform 决定模块图长什么样子

在前文 `Load` 部分我们已经提到过 Vue SFC 的拆解过程，这里需要再次强调一个非常关键的事实：

> **Transform 阶段并不只是“改写代码”，它还会“声明新的模块依赖”。**

仍以 `.vue` 文件为例。

当 `App.vue` 进入 `Transform` 阶段时，Vue 插件会解析其内容：

```vue
<template>
  <div>Hello</div>
</template>

<script setup>
import { ref } from 'vue'
</script>

<style scoped>
div { color: red }
</style>
```

在`transform` 中，`Vue`插件并不会一次性返回所有编译后的结果，而是**构造一个模块入口**：

```js
import script from '/src/App.vue?vue&type=script'
import template from '/src/App.vue?vue&type=template'
import style from '/src/App.vue?vue&type=style'

script.render = template
export default script
```

这一步非常关键：

- 这里的 `import` **并不是对已有文件的引用**
- 而是**对“即将出现的模块”的声明**

这些模块：

```
/src/App.vue?vue&type=script
/src/App.vue?vue&type=template
/src/App.vue?vue&type=style
```

正是：

- 在 `Transform` 阶段被“创造出来的模块结构”
- 在后续请求中，由 `resolveId` + `load` 再次接管

这也同时意味着：

> **Transform 决定了“模块图会长成什么样子”。**

---

#### Transform 与 Load 的边界关系

到此我们可以重新再看一下 `Load` 与 `Transform` 各自的作用边界：

- `Transform`
  - 决定模块**如何被拆解**
  - 决定模块**依赖哪些其他模块**
  - 决定模块**最终暴露给浏览器的结构**
- `Load`
  - 只负责：**当这些模块真的被请求时，返回它们各自的内容**

正是这种“先声明结构、(往后的请求)再按需加载”的设计，使得：

- `.vue`、`.css`、`.wasm`
- 各类虚拟模块、运行时生成模块

都可以自然地融入同一套 ESM 请求机制中。

---

#### Transform 的本质定位

综合来看，`transform` 并不是一个“单纯的编译步骤”，而是整个 Vite Dev Server 中：

> **负责塑造模块语义、构建模块依赖图的核心阶段。**

它让 Vite 能够：

- 在开发期保持源码形态的可读性
- 同时将复杂工程拆解为一系列清晰、可追踪、可按需请求的模块

也正是在这一阶段，Vite 才真正完成了从“源码”到“浏览器模块”的过渡。

---

## 总结：一条“请求驱动”的模块处理管线

回到最初的问题：

> **当浏览器请求一个模块时，Vite Dev Server 内部到底发生了什么？**

到这里，我们已经可以给出一个清晰的答案：

> **Vite Dev Server 并不是在“构建应用”，而是在“响应模块请求”。**

每一次模块请求，都会被送入一条由插件系统驱动的模块处理管线中，依次经历：

```
ResolveId → Load → Transform
```

这三部分分别**承担了三种高度协同的职责**：

- **ResolveId**  
  确认模块的“身份”  
  —— 这个请求在系统内部究竟对应谁？
- **Load**  
  获取模块的“原始内容”  
  —— 这个身份的模块，内容从哪里来？
- **Transform**  
  改写模块的“语义结构”  
  —— 如何让它成为浏览器可执行、可继续派生依赖的 ESM？

通过这一设计，Vite 实现了一种与传统打包工具完全不同的开发期模型：

- **模块不再被一次性分析、打包**
- 而是被拆解为**一系列按需发生的独立请求**
- 模块之间的关系，随着请求的发生逐步显现

尤其是在 `Transform` 阶段，我们已经看到：

- 模块不仅会被“改写”
- 还会**声明新的模块结构**
- 进而触发下一轮模块请求

也就是说：

> **模块请求本身，驱动着整个模块图的呈现。**