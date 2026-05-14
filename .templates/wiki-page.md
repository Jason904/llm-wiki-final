---
title: "{{Concept-Name}}"
aliases: ["{{Alias-1}}"]
page_type: concept
status: {{bud|growing|mature}}
source_coverage: {{0.0-1.0}}
clusters: ["{{Cluster-A}}", "{{Cluster-B}}"]
source_anchors:
  - file: "raw/{{subdirectory}}/{{source-filename}}.md"
    locator: "{{section: Section Name | L100-L120}}"
    excerpt: "{{简短原文摘录，≤50 字}}"
cites: []
generated_from_query: ""
created: {{YYYY-MM-DD}}
updated: {{YYYY-MM-DD}}
---

# {{Concept-Name}}

{{1-2 句话定义，简明扼要说明这个概念是什么。}}

## 核心原理

{{从来源素材综合提炼的主体内容。mature 页面的关键论断应带脚注锚点[^anchor-id]。}}

## 关键组件 / 子概念

- [[{{Related-Concept-A}}]] — {{与本概念的关系}}
- [[{{Related-Concept-B}}]] — {{与本概念的关系}}

## 与其他概念的关系

- 是 [[{{Parent-Concept}}]] 的 {{关系类型，如：子方案 / 实现方式 / 变体}}
- 依赖 [[{{Dependency}}]] 作为 {{基础设施 / 前置条件}}
- 与 [[{{Sibling-Concept}}]] {{互补 / 竞争 / 常一起使用}}

## 应用场景

{{实际使用场景和案例，基于来源素材中的描述。}}

[^anchor-id]: file: raw/{{subdirectory}}/{{source-filename}}.md, section: "{{Section Name}}"

<!--
模板使用说明（concept 页面）：
- bud 页面：只需 H1 + 定义 + "信息不足，待补充素材。"；frontmatter source_anchors 可为 []
- growing 页面：至少填充「核心原理」+ 一个其他 section；source_anchors ≥ 1 条
- mature 页面：所有 section 应有实质内容；关键论断补断言级脚注（每 100 字 ≥ 1 个）
- [[wikilink]] 应出现在语境中，不要为了链接而链接
- clusters 至少 1 个，最多 3 个，优先复用已有 cluster 名
- cites / generated_from_query 字段仅 synthesis 页面使用，concept 页面保持空值
- 可根据概念特性增减 section，但优先使用标准 section
-->
