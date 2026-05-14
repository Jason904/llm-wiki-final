# LLM Wiki · 最终版

> 一个用 LLM 自主维护的 Markdown 知识库——你投喂素材与提问，它负责提取概念、交叉链接、标记缺口、沉淀洞察。

自研业务代码 0 行，完全基于 Claude Code 工作流。目标规模 500-2000 页。

---

## 核心价值

- **知识不会被 RAG 式地「每次重来」**——wiki 是持久复利的
- **问答洞察也会沉淀**——`/query` 默认询问是否归档为 synthesis 页面
- **质量可追溯**——每个概念页带 `source_anchors`（页面级）+ 断言级脚注（mature 页面）
- **系统会主动体检**——`/lint` 分三档：日常快检、周体检、月深检

---

## 技术栈

| 组件 | 角色 |
|------|------|
| Obsidian | Markdown 阅读器 + Graph View（纯查看器，不装插件） |
| Claude Code | 核心引擎（执行 Ingest / Query / Lint 等命令） |
| Git | 版本控制（每次命令一个 commit） |
| 文件系统 | 数据存储（Markdown 文件即数据库） |

---

## 目录结构

用户可见（Obsidian File Explorer 只有这 4 行）：

```
my-wiki/
├── index.md                     # 知识库索引（前门，按 clusters 分组）
├── log.md                       # 事件时间线（append-only，grep 友好）
├── raw/                         # 原始素材（LLM 只读）
│   ├── articles/
│   ├── papers/
│   └── chats/
└── wiki/                        # 知识页面（LLM 生成和维护）
    ├── {Concept-Pages}.md       # 概念页（page_type: concept）
    └── {Synthesis-Pages}.md     # Query 回写的综合页（page_type: synthesis）
```

系统层（`. ` 前缀，Obsidian 默认隐藏）：

```
my-wiki/
├── .claude/                     # 系统规则 + 命令
├── .templates/                  # 参考模板
├── .digest/                     # Ingest 变更报告
├── .proposals/                  # Ingest propose 产物
├── .lint/                       # 体检报告
└── .git/
```

> **Obsidian Graph View** 使用 filter `path:wiki` 只展示知识网络。园丁工具全部在视野之外。

---

## 命令矩阵

### `/ingest` — 投喂素材

| 调用形式 | 适用场景 |
|---------|---------|
| `/ingest <file>` | 一键投喂（信任模式，快速消化） |
| `/ingest propose <file>` | 先生成变更计划，审阅后再 apply（重要素材） |
| `/ingest apply <proposal-slug>` | 执行已审阅的 proposal |

### `/query` — 提问

| 调用形式 | 行为 |
|---------|------|
| `/query "<q>"` | 回答 + 询问是否归档为 synthesis 页面 |
| `/query save "<q>"` | 回答并强制归档 |
| `/query ephemeral "<q>"` | 只回答不落盘、不写 log |

### `/lint` — 体检

| 调用形式 | 频率 | 检查项 |
|---------|------|--------|
| `/lint quick` | 每日 | 孤岛 + 来源缺失 + 花苞绽放 |
| `/lint` | 每周 | 以上 + 重复 + 死链 + synthesis cites |
| `/lint deep` | 每月 | 以上 + 矛盾 + 过时声明 + 脚注密度 + 覆盖缺口建议 |

---

## 快速开始

```bash
cd "你的 wiki 项目目录"

# 初始化 Git
git init -b main
git add .
git commit -m "init: LLM Wiki"
```

用 Obsidian 打开项目根目录作为 Vault。侧栏 pin `index.md` 和 `log.md`。

启动 Claude Code，第一次投喂：

```
/ingest raw/articles/你的文章.md
```

切到 Obsidian，`index.md` 已按 cluster 组织好知识库目录。

---

## 典型节奏

```
早上读完文章
  → 放入 raw/articles/
  → /ingest propose raw/articles/xxx.md       # 重要素材
  → 扫一眼 _proposals/{slug}.md
  → /ingest apply {slug}                       # 确认后执行
  → 切到 Obsidian 看 index.md 更新

或者（信任模式）：
  → /ingest raw/articles/xxx.md                # 一键完成

下午工作中提问
  → /query "xxx 是怎么工作的？"
  → 看回答，选择 [1] 归档 / [2] 仅记 log / [3] 不落盘

周末体检
  → /lint                                       # standard 档

月末深检
  → /lint deep                                  # 看矛盾、过时、覆盖缺口
```

---

## 两个特殊页面类型

| 类型 | page_type | 产生方式 | source_anchors | cites |
|------|-----------|---------|----------------|-------|
| 概念页 | `concept` | `/ingest` 从 raw 提取 | 页面级必填 | — |
| 综合页 | `synthesis` | `/query save` 或默认模式回写 | 固定 `[]` | ≥1 条 wiki 路径 |

两种页面在 `index.md` 中分区展示，Graph View（filter: `path:wiki`）中通过 frontmatter 字段区分。

---

## 文档（dev 项目内，不在子库中）

- [CLAUDE.md](.claude/CLAUDE.md) — 系统规则（LLM 每次启动自动加载）
- [使用手册](docs/usage-guide.md) — 详细使用流程与素材准备
- [健康度指标](docs/health-metrics.md) — 日常健康度观测指标

---

## 红线（LLM 永不违反）

1. **B-1a** 外部信息绝不写入 wiki
2. **B-1b** Lint 可建议搜索方向，但不自行搜索或写入
3. **B-2** 绝不修改 raw/ 下文件
4. **B-3** 绝不编造无来源内容
5. **B-4** 绝不自动合并/删除页面，绝不自动 apply proposal
6. **B-5** 绝不写业务代码
7. **B-6** 绝不修改 `.claude/` / `.templates/`
8. **B-7** `log.md` 只追加，绝不重写

详见 `.claude/CLAUDE.md` §5。
