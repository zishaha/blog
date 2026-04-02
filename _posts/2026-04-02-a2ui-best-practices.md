---
layout: post
title: "A2UI 最佳实践：让 AI Agent 真正理解用户意图"
subtitle: "从协议设计到 Vue 3 渲染器实现，深度解析 Agent-to-User Interface 的工程落地"
date: 2026-04-02
author: 欧阳哲仁
tags: [A2UI, AI, Vue3, 前端工程]
---

在构建 AI 应用的过程中，我们遇到了一个反复出现的问题：**AI 擅长理解意图，却不擅长收集结构化信息**。

用户说"帮我订一张机票"，AI 需要知道出发地、目的地、日期、舱位……于是开始了漫长的对话轮次。每一轮都是一次 API 调用，每一次等待都在消耗用户的耐心。

A2UI 协议正是为了解决这个问题而生的。

---

## 什么是 A2UI？

A2UI（Agent-to-User Interface）是 Google 提出的一个开放协议，核心思想很简单：**让 AI Agent 直接描述 UI，而不是用自然语言反复追问**。

Agent 发送一段 JSON 描述，客户端渲染成真实的表单、列表、按钮。用户填完提交，结构化数据直接回传给 Agent。整个过程从"多轮对话"变成"一次交互"。

```json
{"type":"beginRendering","surfaceId":"flight-form","rootComponentId":"root"}
{"type":"surfaceUpdate","surfaceId":"flight-form","components":[
  {"id":"root","type":"Column","children":{"explicitList":["title","from","to","date","submit"]}},
  {"id":"title","type":"Text","text":{"literal":"请填写航班信息"}},
  {"id":"from","type":"TextField","label":{"literal":"出发城市"},"name":{"literal":"departure"}},
  {"id":"to","type":"TextField","label":{"literal":"目的城市"},"name":{"literal":"destination"}},
  {"id":"date","type":"DateTimeInput","label":{"literal":"出发日期"},"name":{"literal":"date"}},
  {"id":"submit","type":"Button","label":{"literal":"搜索航班"},"actionId":{"literal":"search"}}
]}
```

这段 JSONL 流传输到客户端，渲染器解析后呈现出一个完整的预订表单。

---

## 协议核心设计

理解 A2UI 的关键在于三个设计决策：

### 1. 邻接表而非嵌套树

大多数 UI 框架用嵌套结构描述组件树：

```json
{
  "type": "Column",
  "children": [
    { "type": "TextField", "label": "姓名" },
    { "type": "Button", "label": "提交" }
  ]
}
```

A2UI 选择了**邻接表（Adjacency List）**：所有组件平铺在一个数组里，通过 ID 引用建立父子关系。

这个设计的好处是**增量更新**。当 Agent 需要修改某个组件时，只需发送那一个组件的更新，不需要重传整棵树。对于复杂的动态 UI，这个差异非常显著。

### 2. BoundValue：字面量 vs 数据路径

组件的每个属性都是一个 `BoundValue`，有两种形式：

```typescript
type BoundValue =
  | { literal: string }           // 直接值
  | { path: string }              // JSON Pointer 路径
```

`literal` 就是普通的静态值。`path` 则指向数据模型中的某个字段，使用 [RFC 6901 JSON Pointer](https://datatracker.ietf.org/doc/html/rfc6901) 语法：

```json
{"id":"price","type":"Text","text":{"path":"/products/0/price"}}
```

这让 Agent 可以把数据和 UI 结构分开传输。数据模型通过 `dataModelUpdate` 消息单独更新，UI 组件自动响应变化。

### 3. Template Children：数据驱动的列表

当需要渲染一个列表时，A2UI 不要求 Agent 为每个列表项生成独立的组件 ID。而是用**模板**描述"每一项长什么样"：

```json
{
  "id": "product-list",
  "type": "Column",
  "children": {
    "template": {
      "dataPath": "/products",
      "componentTemplate": {
        "id": "product-item",
        "type": "Row",
        "children": {"explicitList": ["item-name", "item-price"]}
      }
    }
  }
}
```

渲染器遍历 `/products` 数组，为每个元素实例化一份模板，自动处理 ID 命名空间。

---

## Vue 3 渲染器实现

我基于以上协议设计，实现了一个 [Vue 3 渲染器](https://github.com/zishaha/a2ui-vue-renderer)。核心挑战有三个：

### JSONL 流解析

A2UI 消息以 JSONL 格式流式传输，每行一个 JSON 对象。渲染器需要维护一个 Surface 状态机：

```typescript
function processMessage(msg: A2UIMessage, surfaces: Map<string, Surface>) {
  switch (msg.type) {
    case 'beginRendering':
      surfaces.set(msg.surfaceId, {
        rootId: msg.rootComponentId,
        components: {},
        dataModel: {}
      })
      break
    case 'surfaceUpdate':
      for (const comp of msg.components) {
        surfaces.get(msg.surfaceId)!.components[comp.id] = comp
      }
      break
    case 'dataModelUpdate':
      Object.assign(surfaces.get(msg.surfaceId)!.dataModel, msg.data)
      break
    case 'deleteSurface':
      surfaces.delete(msg.surfaceId)
      break
  }
}
```

### 递归组件渲染

Vue 3 的 `h()` 函数支持动态渲染，非常适合实现递归组件树：

```typescript
// A2UIComponent.ts
export default defineComponent({
  name: 'A2UIComponent',
  props: ['componentId', 'surface', 'catalog', 'onAction'],
  setup(props) {
    return () => {
      const comp = props.surface.components[props.componentId]
      if (!comp) return null

      // 解析 BoundValue 属性
      const resolvedProps = resolveProps(comp, props.surface.dataModel)

      // 从 catalog 获取对应的 Vue 组件
      const VueComp = props.catalog[comp.type]
      if (!VueComp) return h('div', `Unknown: ${comp.type}`)

      // 解析子组件列表
      const children = resolveChildren(comp, props.surface)

      return h(VueComp, { ...resolvedProps, onAction: props.onAction },
        () => children.map(childId =>
          h(A2UIComponent, { componentId: childId, ...props })
        )
      )
    }
  }
})
```

### BoundValue 解析

```typescript
export function resolveBoundValue(
  value: BoundValue | undefined,
  dataModel: Record<string, unknown>
): unknown {
  if (!value) return undefined
  if ('literal' in value) return value.literal
  if ('path' in value) return get(dataModel, value.path)
  return undefined
}
```

JSON Pointer 解析遵循 RFC 6901，`/products/0/name` 会被解析为 `dataModel.products[0].name`，同时处理 `~0`（`~`）和 `~1`（`/`）的转义。

---

## 最佳实践

经过实际使用，总结了几条工程建议：

**1. 组件 Catalog 按需注册**

不要把所有组件都注册进 catalog，只注册当前场景需要的。这样可以做 tree-shaking，也让 Agent 的输出更可预测。

**2. 数据模型与 UI 分离传输**

让 Agent 先发 `beginRendering` + `surfaceUpdate` 建立 UI 骨架，再通过 `dataModelUpdate` 填充数据。这样用户能更快看到界面，数据加载时有骨架屏效果。

**3. 用 TypeScript 严格约束消息类型**

A2UI 消息是 Agent 生成的，容易出现字段缺失或类型错误。在渲染器入口做严格的类型校验，比在组件内部防御性编程更高效。

**4. Action 处理要幂等**

用户可能多次点击同一个按钮。`actionId` 应该设计成幂等的，或者在渲染器层面做防抖处理。

**5. Surface 生命周期管理**

及时处理 `deleteSurface` 消息，避免内存泄漏。对于长时间运行的 Agent 会话，Surface 的创建和销毁可能非常频繁。

---

## 与传统方案的对比

| 维度 | 传统多轮对话 | A2UI |
|------|------------|------|
| 信息收集轮次 | 每个字段一轮 | 一次提交 |
| 用户体验 | 线性、被动 | 并行、主动 |
| 结构化程度 | 需要 NLP 解析 | 原生结构化 |
| 前端复杂度 | 低（纯文本） | 中（需要渲染器） |
| Agent 控制力 | 弱 | 强 |

A2UI 不是银弹。对于简单的单字段查询，多轮对话反而更自然。但当需要收集复杂的结构化信息时，A2UI 的优势非常明显。

---

## 总结

A2UI 代表了一种新的 Human-AI 交互范式：**Agent 不再只是回答问题，而是主动构建交互界面**。

这对前端工程师意味着新的机会——我们不再只是为人类设计师服务，也开始为 AI Agent 提供渲染能力。理解协议、实现渲染器、设计组件 catalog，这些都是新的工程挑战。

Vue 3 的响应式系统和 `h()` 函数让实现 A2UI 渲染器变得相对直接。完整的实现代码和 Demo 在 [GitHub](https://github.com/zishaha/a2ui-vue-renderer) 上，欢迎参考和贡献。

---

*本文基于 A2UI Protocol v0.8 规范，Vue 3.4+ 实现。*
