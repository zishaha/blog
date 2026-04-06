---
layout: post
title: "AI 是怎么读懂一份医学报告的？"
subtitle: "从「帮我分析这份肺癌报告」到交付——NanoClaw 内部技术全解析"
date: 2026-04-06 21:00:00 +0800
author: 欧阳哲仁
tags: [AI, NanoClaw, Claude Code, 技术架构, 技能系统]
---

> 几天前，我让 AI 助手 Andy 分析了一份《早期肺癌治疗方案患者家属版（2026）》的医学报告，最终交付了一份结构完整、引用权威指南（NCCN·CSCO·IASLC）的深度研究文档。
>
> 整个过程对用户来说只是「发一条消息、等几分钟、拿到报告」。但在这背后，一套精心设计的系统正在运转。这篇文章拆解它内部的完整机制。

---

## 一、从用户视角看：一条消息引发了什么

```
carlos.ling: 帮我深度研究下肺癌早期的治疗方案，需要一篇简体中文报告，专业，准确
```

Andy 收到这条消息，几分钟后交付了：

- 一份 Markdown 格式的深度研究报告（专业版，约 8000 字）
- 一份为患者家属改写的通俗版（含表格、问答、决策流程图）
- 一份 PDF 版本（排版规范，可直接打印）

用户看到的是「结果」。系统内部发生的，是一条完整的调度链。

---

## 二、NanoClaw 是什么？整体架构

**NanoClaw** 是基于 Claude Code SDK 构建的个人 AI 助手平台。它的特别之处在于：**Claude 不只是聊天机器人，而是一个有工具调用能力、有持久记忆、有专业技能包的 Agent**。

```
用户（WhatsApp / Discord / Telegram）
        ↓  消息
  NanoClaw 消息路由层
        ↓  触发
  Claude Code Agent（「Andy」）
        ↓  规划 + 执行
  技能系统（Skills）
        ↓  调用
  工具链（Web Search / Bash / File I/O / API）
        ↓  产出
  结构化文档（Markdown / PDF / HTML）
```

其中**技能系统**是整个流程的核心。

---

## 三、技能系统（Skills）：Claude Code 的「专家插件」

### 3.1 技能是什么

在 NanoClaw 里，每个技能（Skill）是一个存放在 `/home/node/.claude/skills/<skill-name>/` 目录下的知识包，包含：

```
skills/
├── pdf/
│   ├── SKILL.md          ← 技能描述 + 触发条件 + 使用指南
│   ├── scripts/          ← 可执行脚本
│   └── reference.md      ← 详细参考文档
├── treatment-plans/
│   ├── SKILL.md
│   ├── scripts/
│   └── references/
├── literature-review/
├── clinical-reports/
├── parallel-web/
└── ...（共 150+ 个技能）
```

`SKILL.md` 是核心文件，它的开头是一段 YAML frontmatter，定义了技能的**触发条件**：

```yaml
---
name: treatment-plans
description: Generate concise (3-4 page), focused medical treatment plans in LaTeX/PDF
  format for all clinical specialties...
allowed-tools: Read Write Edit Bash
---
```

当 Claude 收到一个任务，它会扫描技能列表，**通过 description 字段语义匹配**最相关的技能，然后读取 SKILL.md 作为上下文，按照其中的指引执行。

### 3.2 技能激活：从语义到调度

分析「肺癌报告」这个任务，Claude 实际上激活了多个技能的组合：

| 阶段 | 激活的技能 | 作用 |
|------|-----------|------|
| 文献检索 | `literature-review` + `parallel-web` | 搜索 PubMed、arXiv，获取最新研究 |
| 权威指南查询 | `research-lookup` | 查询 NCCN/CSCO/IASLC 最新版指南 |
| PDF 处理 | `pdf` | 读取已有 PDF，提取原始文本 |
| 报告撰写 | `clinical-reports` | 按临床报告规范组织结构 |
| 患者版改写 | `treatment-plans` | 按 SMART 目标框架输出可读版本 |

这不是预设的「工作流」，而是 Claude 在理解任务后**动态规划**的调用顺序。

---

## 四、文献检索层：parallel-web + research-lookup

这是整个报告质量的基础。两个技能协同工作：

### 4.1 parallel-web：全网实时检索

`parallel-web` 调用 [Parallel Web Systems API](https://parallel.ai)，这是一个专为 AI Agent 设计的搜索 API，提供**带引用的综合摘要**，而非原始搜索结果列表：

```bash
# 技能内部执行的命令示例
python scripts/parallel_web.py search \
  "early stage NSCLC treatment guidelines NCCN 2025 2026" \
  --model core
```

返回的不是 10 个蓝色链接，而是：
> 「根据 NCCN 2025 指南（V2.2025），IA 期 NSCLC 首选亚肺叶切除术（基于 JCOG0802 试验，5 年总生存率 94.3%）……[1][2][3]」

### 4.2 research-lookup：学术论文精确查询

对于需要论文级证据的部分，`research-lookup` 会路由到 **Perplexity sonar-pro-search**（接入 PubMed、Semantic Scholar、bioRxiv），专攻学术数据库：

```python
# 内部路由逻辑伪代码
if query_type == "academic_paper":
    backend = "perplexity_sonar_pro"
else:
    backend = "parallel_chat_api"
```

两级检索的结果被合并，Claude 负责去重、核实、归纳——这就是为什么最终报告里能看到诸如「JCOG0802 + CALGB 140503 双试验验证」这类具体引用，而不是模糊描述。

---

## 五、文档生成层：从原始内容到结构化报告

检索到原始信息后，`clinical-reports` 技能接管。

### 5.1 临床报告的结构约束

`clinical-reports` 的 SKILL.md 明确规定了不同类型报告的格式规范：

- **Case Report**：遵循 CARE（CAse REport）指南
- **诊断报告**：遵循放射科/病理科文档标准
- **患者文档**：SOAP 格式、H&P 格式、出院摘要

对于「肺癌治疗方案研究报告」，Claude 选择了**综合型临床报告**结构：

```
执行摘要（Evidence-Based）
  ↓
疾病背景与分期系统（TNM + 中国 CSCO 分层）
  ↓
一线治疗方案（手术 / 放疗 / 辅助治疗）
  ↓
分子分型与靶向治疗（EGFR / ALK / ROS1 / KRAS）
  ↓
临床决策流程（结构化决策树）
  ↓
参考文献（Vancouver 格式）
```

### 5.2 两个版本的分工

专业版报告完成后，`treatment-plans` 技能被调用，执行**降维改写**任务：

```
clinical-reports 输出（专业版，面向医生）
        ↓
treatment-plans（SMART 框架重构）
        ↓
患者家属版（用「大白话」替换术语、加入 Q&A、添加决策提示）
```

`treatment-plans` 的核心原则是 **SMART 目标框架**（Specific / Measurable / Achievable / Relevant / Time-bound），它确保每个建议都是「可操作的」而非「概念性的」：

- ❌「建议手术治疗」
- ✅「IA 期患者：2 周内与胸外科会诊，评估 VATS 亚肺叶切除术适应症，预计住院 3–5 天」

---

## 六、PDF 生成：从 Markdown 到排版文档

`pdf` 技能负责最后的格式化输出。它的工具链是：

```
Markdown 源文件
    ↓
pypdf / pdfplumber（Python 库）
    ↓ 或
LaTeX 模板（treatment-plans 技能内置）
    ↓
PDF 输出（带目录、页眉页脚、引用标注）
```

对于医学报告，`treatment-plans` 优先选用 **LaTeX 模板**路径——因为 LaTeX 能精确控制医学文档的排版规范（字号、行距、表格、脚注格式），比纯 Markdown→PDF 转换质量高一个数量级。

---

## 七、记忆系统：下次不需要从头来

NanoClaw 在 `/workspace/group/conversations/` 目录下维护对话归档，在 `/workspace/group/sources/` 下保存生成的文档。

这意味着：

```
第一次：「帮我研究肺癌早期治疗方案」→ 检索 + 分析 + 生成（5–8 分钟）
第二次：「把患者版报告发给我」→ 直接读文件（5 秒）
第三次：「基于上次的报告，加入 KRAS 突变的最新进展」→ 增量检索 + 局部更新
```

Karpathy 在他的 [LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) 里描述的「**持久知识库**」模式，正是这里运行的逻辑：不是每次从零重新分析，而是在已有基础上**复利增长**。

---

## 八、完整调用链路图

```
用户发送消息
    │
    ▼
NanoClaw 路由层（消息解析 + 权限检查）
    │
    ▼
Claude Code Agent（Andy）
    │
    ├─── 意图理解："深度医学研究报告"
    │
    ├─── 技能匹配（语义扫描 SKILL.md）
    │      ├── literature-review  ✓
    │      ├── parallel-web       ✓
    │      ├── research-lookup    ✓
    │      ├── clinical-reports   ✓
    │      ├── treatment-plans    ✓
    │      └── pdf                ✓
    │
    ├─── 执行阶段 1：文献检索
    │      ├── parallel-web → Parallel Chat API → 综合网络资料
    │      └── research-lookup → Perplexity → PubMed / Semantic Scholar
    │
    ├─── 执行阶段 2：内容生成
    │      ├── clinical-reports → 专业版报告（Markdown）
    │      └── treatment-plans → 患者家属版（Markdown）
    │
    ├─── 执行阶段 3：格式化输出
    │      └── pdf → LaTeX → PDF
    │
    └─── 写入 /workspace/group/sources/
              ├── 肺癌早期治疗方案深度研究报告_2026.md
              ├── 肺癌早期治疗方案患者家属版_2026.md
              └── 肺癌早期治疗方案深度研究报告_2026.pdf
```

---

## 九、为什么这个架构有效？

### 专业知识与通用推理分离

Claude 本体擅长推理和语言，但不应该把「如何写一份符合 CARE 指南的临床报告」硬编码在系统提示里。技能系统让这部分**可组合、可替换、可独立更新**——就像软件的插件架构。

### 工具调用 vs 幻觉

医学领域容错率极低。`parallel-web` 和 `research-lookup` 提供了**带来源的实时信息**，配合 Claude 的推理能力做筛选和综合，比让模型凭训练数据「背诵」医学知识可靠得多。

### 输出格式的约束力

`treatment-plans` 要求输出遵循 SMART 框架，`clinical-reports` 要求遵循 CARE 指南——这些约束不是提示词里的「请做到……」，而是内嵌在技能文档里的**结构性约束**，模型读了 SKILL.md 之后，等于被「塑形」成了一个临床报告专家。

---

## 十、这个模式的边界

当然，这套架构也有局限：

- **API 依赖**：parallel-web 需要 `PARALLEL_API_KEY`，断网或 key 失效时降级为纯模型能力
- **技能质量参差**：150+ 个技能由不同团队维护，质量不均，部分技能的 SKILL.md 指引过于笼统
- **医学准确性**：模型仍然可能误读文献或综合出错误结论，生成的报告**不能替代医生诊断**
- **长文档的上下文限制**：当报告超过 10 万字，Claude 的上下文窗口会成为瓶颈，需要分块处理

---

## 小结

「AI 分析医学报告」这件事，表面上是大模型的语言能力，内核是一套**工程化的 Agent 架构**：

| 层次 | 组件 | 职责 |
|------|------|------|
| 调度层 | Claude Code Agent | 任务拆解、技能规划、结果整合 |
| 技能层 | Skills（SKILL.md） | 领域知识注入、执行约束、工具路由 |
| 检索层 | parallel-web + research-lookup | 实时信息获取、引用溯源 |
| 生成层 | clinical-reports + treatment-plans | 结构化文档生成 |
| 输出层 | pdf | 格式规范化 |
| 记忆层 | 文件系统（conversations/ + sources/） | 持久化知识积累 |

这让 AI 不只是「回答问题」，而是在每次对话中**构建可复用的知识产出**。

下一篇，我会深入讲 NanoClaw 的记忆系统是如何设计的——以及为什么「把对话存进文件」比向量数据库更适合个人助手场景。

---

*本文描述的技术实现基于 NanoClaw + Claude Code SDK，截至 2026 年 4 月的版本状态。*
