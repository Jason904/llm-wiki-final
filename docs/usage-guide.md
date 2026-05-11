# 使用手册

本手册覆盖最终版的完整使用流程：Vault 初始化 → 素材准备 → 三个命令的实际用法 → 常见问题。

---

## 1. Vault 初始化

```bash
cd "你的 wiki 项目目录"

mkdir -p raw/articles raw/papers raw/chats raw/assets
mkdir -p wiki _digest _proposals _lint

git init
git add .
git commit -m "init: implementation pack"
```

### Obsidian 配置

1. 用 Obsidian 打开项目根目录作为 Vault。
2. 在右侧栏 pin 三个文件（右键 → Pin）：
   - `_digest/latest.md` — 最新投喂变更报告
   - `_lint/latest-report.md` — 最新体检报告
   - `_log.md` — 事件时间线
3. 不需要安装任何插件。
4. 打开 Graph View 备用。

### Claude Code 启动

在项目根目录启动 Claude Code，确认 `.claude/CLAUDE.md` 自动加载。

---

## 2. 素材准备

### 格式要求

- **Markdown (.md) 或纯文本 (.txt)**
- PDF、网页、图片等需要**手动**转为 Markdown 后再放入 `raw/`
  - 推荐工具：Obsidian Web Clipper、MarkDownload 浏览器插件、直接复制粘贴
- 图片附件放入 `raw/assets/`，在 Markdown 中用相对路径引用

### 命名建议

- 英文小写 + 连字符：`rag-survey-2026.md`
- 避免空格、避免中文文件名（影响 proposal-slug 生成）
- 带日期的素材：`2026-lewis-et-al-rag.md`

### 子目录选择

- `raw/articles/` — 博文、网页、公众号文章
- `raw/papers/` — 学术论文
- `raw/chats/` — 对话记录、聊天摘要

这些子目录是建议而非强制，LLM 按 frontmatter 中的 cluster 字段组织知识，和目录结构解耦。

---

## 3. Ingest 流程

### 3.1 一键模式（quick）

用于相信 LLM 能正确处理的素材。适合：科普类文章、熟悉领域的文档。

```
/ingest raw/articles/rag-survey-2026.md
```

LLM 会完成：读取 → 提取概念 → 写入 wiki + 脚注 → 交叉链接 → 花苞标记 → 生成 Digest → 追加 `_log.md` → 重写 `_index.md` → Git commit。

完成后切到 Obsidian，`_digest/latest.md` 自动刷新。扫一眼：
- 🆕 新页面列表
- ✏️ 更新了哪些页面
- 🌱 新生花苞
- 📊 知识库快照（before → after）

### 3.2 两阶段模式（propose + apply）

用于需要审阅的重要素材：新领域、高价值论文、可能产生矛盾的内容。

**第一阶段：提议**

```
/ingest propose raw/papers/lewis-2020-rag.md
```

LLM 只生成 `_proposals/2026-05-11-lewis-2020-rag.md`，**不改 wiki**。文件内容包括：
- 将新建的页面清单
- 将更新的页面清单（含追加内容预览）
- 新花苞列表
- **潜在矛盾**（素材观点 vs 现有 wiki 表述）
- **待确认问题**（LLM 自己不确定的地方）

用 Obsidian 打开 proposal 文件，可以：
- 修改 cluster 归属
- 回答 LLM 的待确认问题
- 删除不想处理的概念

**第二阶段：应用**

审阅后执行：

```
/ingest apply 2026-05-11-lewis-2020-rag
```

LLM 读取 proposal（包含你的编辑），执行写入 + Digest + `_index.md` 重建 + log 追加。proposal 文件 frontmatter 的 `status` 从 `draft` 变为 `applied`，文末追加「实际执行结果」。

### 3.3 什么时候用哪个

| 素材类型 | 推荐模式 |
|---------|---------|
| 熟悉领域的科普文 | quick |
| 新领域、首批素材 | propose + apply |
| 可能与现有 wiki 矛盾 | propose + apply |
| 批量投喂 10+ 篇 | quick（逐篇） |
| 仅一两句关键信息 | quick（生成 bud） |

---

## 4. Query 流程

### 4.1 默认模式（推荐）

```
/query "chunk size 如何影响 RAG 检索质量？"
```

LLM 基于 wiki 综合回答，每个论点标注 `[source: wiki/xxx.md § "section"]`。回答末尾，LLM 询问：

```
💾 归档选项：
   本次回答综合了 3 个 wiki 页面。推荐归档为 synthesis 页面。
   [1] 归档为 synthesis 页面
   [2] 不归档，仅记录 log
   [3] 完全不落盘
```

选 [1]：生成 `wiki/Chunk-Size-RAG-Analysis.md`（page_type: synthesis），写入 cites、generated_from_query。
选 [2]：`_log.md` 追加一行，但不生成 synthesis 页面。
选 [3]：什么也不记录。

### 4.2 强制归档

```
/query save "chunk size 如何影响 RAG 检索质量？"
```

直接跳过询问，生成 synthesis 页面。

### 4.3 一次性查询

```
/query ephemeral "什么是 MoE？"
```

只输出回答，不写 log、不生成页面、不 commit。用于临时查证。

### 4.4 查询时的约束

- 回答**只基于 wiki/**，绝不使用外部知识
- 若 wiki 无相关内容 → 明确告知「知识库暂无相关信息」，不编造
- 若涉及 bud 概念 → 末尾提示相关花苞需要补素材

---

## 5. Lint 流程

### 5.1 日常快检（每日）

```
/lint quick
```

检查：孤岛、来源缺失、即将绽放花苞。秒级。

### 5.2 周体检（每周）

```
/lint
```

在快检基础上加：重复候选、死链、synthesis cites 完整性。

### 5.3 月深检（每月）

```
/lint deep
```

在周体检基础上加：矛盾检测、过时声明、mature 脚注密度、覆盖缺口 + 搜索建议。

最有价值的是：
- **矛盾**：`A 页面说 X vs B 页面说 not X`。LLM 不会自动修改，你来决断。
- **覆盖缺口**：被 3+ 页面引用但本体还是 bud/growing 的概念，附带可搜索关键词。

---

## 6. 日常节奏建议

| 时刻 | 动作 |
|------|------|
| 早上 | 读文章 → `/ingest` |
| 工作中 | `/query` 查证 |
| 睡前 | `/lint quick` 看一眼 |
| 周末 | `/lint` |
| 月末 | `/lint deep` + 处理深检发现 |

第一个月内最重要的是**建立投喂习惯**——每天至少一篇素材。第二个月开始 wiki 的网络效应会显现，query 和 synthesis 变得有价值。

---

## 7. 常见问题

### Q1: ingest 报错「文件不存在」

- 确认路径在 `raw/` 下（不含 `raw/assets/`）
- 确认扩展名是 .md 或 .txt
- 中文路径可能需要引号：`/ingest "raw/articles/我的文章.md"`

### Q2: proposal 太长，看不过来

- 减小投喂素材的规模，或先用 `/ingest quick` 试水
- 在 proposal 中只保留最核心的几个概念，手动删除其他条目再 apply

### Q3: Obsidian 侧栏没刷新

- Obsidian 通常自动监听文件变更。若没有，手动切出/切回这个文件即可
- 确认 pin 的文件路径正确

### Q4: 概念太多 / 概念太细

调整 `.claude/CLAUDE.md` §8「概念提取」和「Cluster 归属」部分的启发式。常见的调整：
- 太细：降低「bud 判定阈值」，小于 50 字的提及直接跳过
- 太粗：提高「single-source bud」灵敏度

### Q5: 花苞泛滥

如果 bud 占比 > 50%，说明素材覆盖面太广或提取粒度太细。三种对策：
- 增加同领域素材，让花苞升级
- 在 CLAUDE.md §8 中提高 bud 门槛
- `/lint deep` 查「即将绽放花苞」清单，针对性补充素材

### Q6: synthesis 页面和 concept 页面冲突

命名时加后缀区分：`RAG.md`（concept）vs `RAG-Analysis.md`（synthesis）。`_index.md` 也会分区展示。

### Q7: 我想手动改 wiki 页面

可以。`B-4` 红线只限制 LLM 不自动合并/删除，用户的手动编辑完全受尊重。但避免改动 frontmatter 字段结构，否则后续 ingest 可能解析失败。
