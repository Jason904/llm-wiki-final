---
source: "raw/{{subdirectory}}/{{source-filename}}.md"
proposal_slug: "{{YYYY-MM-DD}}-{{source-slug}}"
proposal_date: "{{YYYY-MM-DD}}"
status: draft    # draft | applied | discarded
---

# 📋 Ingest Proposal · {{source-filename}}

> 本文件由 `/ingest propose` 生成。审阅后执行 `/ingest apply {{proposal-slug}}` 写入知识库，或直接删除/修改本文件以重新规划。

## 素材概览

- **文件**：`{{source}}`
- **字数**：约 {{N}} 字
- **主题**：{{一句话概述素材内容}}

## 提取的概念

### 🆕 将新建页面（{{N}}）

- **{{Concept-A}}** — {{一句话说明}}
  - clusters: `{{Cluster-1}}`, `{{Cluster-2}}`
  - 预计 status：{{bud | growing | mature}}
  - 来源章节：{{section 或 L100-L120}}

- **{{Concept-B}}** — {{一句话说明}}
  - clusters: `{{Cluster-1}}`
  - 预计 status：{{bud}}
  - 来源章节：{{section}}

### ✏️ 将更新页面（{{N}}）

- [[{{Existing-Page-C}}]] ← 计划追加 `{{section 名}}` 段落，补充 `{{具体内容要点}}`
  - 新 source_anchor：`file: {{source}}, locator: {{locator}}`
  - status 变化：{{growing → mature}}（原因：{{...}}）

- [[{{Existing-Page-D}}]] ← 计划在「关键组件」中新增 [[{{New-Concept}}]] 链接

### 🌱 新花苞（{{N}}）

- **{{Concept-X}}** — 原因：{{如：仅被提及一次，信息不足以独立成页}}
- **{{Concept-Y}}** — 原因：{{如：被多次提及但无清晰定义}}

### 🔗 预计新增的 wikilink

- {{Concept-A}} → [[{{Existing-Page}}]]（关系：{{is-a / depends-on / ...}}）
- [[{{Existing-Page}}]] → {{Concept-A}}（反向链接）

## ⚠️ 潜在矛盾

{{如果没有：}}
未发现明显矛盾。

{{如果有：}}
- [[{{Existing-Page}}]] 当前表述「X 是 Y」（来源：{{既有 source_anchor}}）
  vs. 本素材主张「X 不是 Y」（来源：`{{source}}, {{locator}}`）
  **建议**：在 [[{{Existing-Page}}]] 的「观点分歧」section 保留两种表述，不做单边覆盖。

## ❓ 待用户确认

{{LLM 在 propose 阶段遇到的不确定点，apply 前请回答：}}

- {{问题 1，如：{{Concept-A}} 是归入 `Retrieval` 还是 `LLM-Infrastructure`？}}
- {{问题 2，如：素材中提到的 "X 方法" 与已有 [[{{Existing}}]] 是否指同一事物？}}

## 📊 知识库变化预测

- 总页面：{{before}} → {{after}}（+{{delta}}）
- 🌳 {{N}} | 🌿 {{N}} | 🌱 {{N}}
- 涉及 clusters：{{Cluster-1, Cluster-2, ...}}

## 执行方式

- **接受并应用**：`/ingest apply {{proposal-slug}}`
- **重新规划**：手动编辑本文件后再执行 apply，或直接重新运行 `/ingest propose`
- **放弃**：将本文件 frontmatter 的 `status` 改为 `discarded`，本 proposal 将不再参与后续 apply

<!--
模板使用说明（proposal）：
- 本文件是 /ingest propose 的产物，不改动 wiki/
- 用户可直接编辑本文件（修改概念列表、回答待确认问题、调整 clusters），apply 会尊重编辑后的内容
- apply 完成后，LLM 会将本文件 frontmatter 的 status 从 draft 改为 applied，并在文末追加「实际执行结果与预测的差异」
- 未 apply 的 proposal 会一直保留，过期的 proposal 由用户手动删除
-->
