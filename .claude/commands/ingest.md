接收 `$ARGUMENTS`。根据首词分发到三种子模式之一：`propose` / `apply` / quick（默认）。

| 调用形式 | 子模式 | 产物 |
|---------|--------|------|
| `/ingest <filepath>` | quick | 一键完成 propose + apply，直接写入 wiki + digest |
| `/ingest propose <filepath>` | propose | 只生成 `_proposals/{slug}.md`，不改 wiki |
| `/ingest apply <proposal-slug>` | apply | 读取 proposal 执行写入 + digest |

所有模式结束前都追加 `_log.md`，并按 §4.1 规则 commit。

---

## Step 0: Mode Dispatch

1. PARSE `$ARGUMENTS`。
2. SET first_word = 首个空格前的词。
3. IF first_word == "propose" → 进入 **Mode A: Propose**，剩余参数作为 file_list。
4. ELSE IF first_word == "apply" → 进入 **Mode B: Apply**，剩余参数作为 proposal-slug。
5. ELSE → 进入 **Mode C: Quick**，整个 `$ARGUMENTS` 作为 file_list。
6. IF `$ARGUMENTS` 为空 → PRINT 用法提示并 STOP：
   ```
   ❌ 用法:
      /ingest <filepath> [filepath2 ...]         # quick 模式
      /ingest propose <filepath>                  # 只规划不写入
      /ingest apply <proposal-slug>               # 执行已有 proposal
   ```

---

## Mode A: Propose

### A.1 前置检查

7. 对 file_list 中每个文件：VERIFY 存在、在 raw/ 下、扩展名 .md/.txt。任一不合法 → ABORT。
8. IF file_list 为空 → PRINT 用法并 STOP。

### A.2 阅读与扫描

9. READ 每个文件完整内容。
10. SCAN wiki/ 下所有页面，READ frontmatter（title、aliases、status、clusters、source_anchors）和首段定义。
11. PRINT "🔍 Reading {filename} ... ({word_count} 字)"

### A.3 概念规划

12. IDENTIFY 素材中的关键概念 / 实体 / 方法论。
13. 对每个概念判断：
    - wiki/{Concept}.md 存在 → 标记为 UPDATE
    - 不存在且信息充足 → 标记为 NEW（预计 status：growing / mature）
    - 不存在且信息不足 → 标记为 BUD（预计 status：bud）
14. 对每个 NEW 概念，确定 clusters（per CLAUDE.md §8 「Cluster 归属」）：优先复用已有 cluster 名；新 cluster 只在必要时创建。
15. 识别**潜在矛盾**：扫描本素材主张 vs 已有 wiki 表述，找出同一事实被不同说法描述的情形。
16. 识别**待确认点**：LLM 在规划中遇到的真实不确定，如 cluster 归属、概念归并、命名选择。
17. IF 总影响页面数 > 20 → 保留 top-20，其余记录到 proposal。

### A.4 生成 Proposal 文件

18. DERIVE proposal_slug（per CLAUDE.md §9 source-slug 规则）：
    - 单文件：`{YYYY-MM-DD}-{source-slug}`。如 `2026-05-11-rag-survey-2026`
    - 多文件：沿用 §9 的多文件 slug 规则前缀日期
19. 按 `templates/proposal.md` 结构 COMPOSE proposal 内容，`status: draft`。
20. WRITE `_proposals/{proposal_slug}.md`。

### A.5 追加 `_log.md`

21. IF `_log.md` 不存在 → 先 CREATE，写入 `# LLM Wiki · 事件时间线\n`。
22. APPEND 一行（per CLAUDE.md §11.2）：
    ```
    ## [YYYY-MM-DD HH:MM] ingest:propose | {source-filename-list} | → _proposals/{proposal_slug}.md
    ```

### A.6 Git Commit

23. `git add _proposals/{proposal_slug}.md _log.md`
24. `git commit -m "propose: {source-filename}"`
25. PRINT 汇总：
    ```
    📋 Proposal 已生成：_proposals/{proposal_slug}.md
       - {new_count} 新页面 · {update_count} 更新 · {bud_count} 新花苞
       - {contradiction_count} 个潜在矛盾 · {question_count} 个待确认
    
    审阅后执行 `/ingest apply {proposal_slug}` 写入知识库。
    Git commit: {short-hash} "propose: {source-filename}"
    ```

---

## Mode B: Apply

### B.1 前置检查

26. SET proposal_slug = 剩余参数。
27. VERIFY `_proposals/{proposal_slug}.md` 存在。不存在 → PRINT 错误并 STOP。
28. READ proposal 文件。CHECK frontmatter `status`：
    - IF `applied` → PRINT "⚠️ 该 proposal 已被 apply，跳过。" STOP。
    - IF `discarded` → PRINT "⚠️ 该 proposal 已被标记为放弃。要重做请删除文件或改为 draft。" STOP。
    - IF `draft` → 继续。
29. EXTRACT proposal 中的 source file 路径、新建清单、更新清单、待确认答案（用户可能在 proposal 文件中补充了回答）。

### B.2 写入 wiki（参考 Mode C 的 Step C.3-C.5）

30. SNAPSHOT 当前 wiki/ 状态（用于 digest "before"）。
31. 按 proposal 的新建清单执行 CREATE（per CLAUDE.md §2.2 frontmatter schema）：
    - page_type: concept
    - source_anchors：从 proposal 的「来源章节」生成
    - clusters：从 proposal 获取
    - status / source_coverage：按 CLAUDE.md §2.3 / §8 启发式
    - mature 页面补断言级脚注（per §2.7）
32. 按 proposal 的更新清单执行 UPDATE：
    - APPEND 新 source_anchors，不删除旧的
    - 在对应 section 追加内容
    - UPDATE `updated` 日期为今天
    - 重新评估 status（单调递增方向）
    - 重新估算 source_coverage（per CLAUDE.md §8，单调递增）
33. 按 proposal 的矛盾清单，在相关页面的「观点分歧」section 中保留两种表述（**不覆盖**，B-4 红线）。
34. CREATE / UPDATE 花苞页面。
35. 在本次新/改页面间建立双向 wikilink（per CLAUDE.md §2.5）。
36. PRINT 每个操作行：
    ```
    [NEW]     wiki/{Concept-Name}.md (status: {status}, clusters: [...])
    [UPDATE]  wiki/{name}.md ← {更新概要}
    [BUD]     wiki/{name}.md (status: bud) — {原因}
    [LINK]    {summary} ({N} new wikilinks)
    [⚠️ 观点分歧] {Page-A} vs {Page-B}: {说明}
    ```

### B.3 Digest

37. SNAPSHOT 当前 wiki/ 状态（"after"）。
38. 按 `templates/digest.md` 结构 COMPOSE digest 内容，额外包含「与 Proposal 的差异」section（说明 apply 实际执行偏离 proposal 的地方；无偏离则写「完全按 proposal 执行」）。
39. WRITE `_digest/{proposal_slug 去掉日期前缀}.md` → 实际为 `_digest/{YYYY-MM-DD}-{source-slug}.md`，复用 proposal 的 slug 主体。
40. WRITE `_digest/latest.md`（覆盖）。

### B.4 更新 `wiki/_index.md`

41. 重建 `wiki/_index.md`（per CLAUDE.md §2.6）：按 cluster 分组，cluster 内 status 降序排列，synthesis 页面单独集中在文末。

### B.5 标记 Proposal 为 applied

42. 更新 proposal 文件 frontmatter：`status: applied`。
43. 在 proposal 文末追加一段：
    ```markdown
    
    ---
    
    ## Apply 执行结果（{YYYY-MM-DD HH:MM}）
    
    - 实际新建：{N}
    - 实际更新：{N}
    - 实际花苞：{N}
    - 与预测差异：{无 / 具体说明}
    ```

### B.6 追加 `_log.md`

44. APPEND：
    ```
    ## [YYYY-MM-DD HH:MM] ingest:apply | {proposal_slug} | +{N} pages, +{N} links, +{N} buds
    ```

### B.7 Git Commit

45. `git add wiki/ _digest/{...}.md _digest/latest.md _proposals/{proposal_slug}.md _log.md`
46. IF 无变化（罕见，proposal 可能完全不需要执行）→ PRINT "⚠️ No changes to commit" 并跳过 commit。
47. ELSE `git commit -m "ingest: apply {proposal_slug}"`
48. PRINT 汇总（同 Mode C 完成输出格式）

---

## Mode C: Quick（默认）

### C.1 前置检查

49. 对 file_list 中每个文件：VERIFY 存在、在 raw/ 下（不含 raw/assets/）、扩展名 .md/.txt。任一不合法 → ABORT。
50. IF file_list 为空 → PRINT 用法并 STOP。

### C.2 阅读素材

51. READ 每个文件完整内容。
52. PRINT "🔍 Reading {filename} ... ({word_count} 字)"

### C.3 概念提取

53. SCAN 现有 wiki/ 获取已有 cluster 列表和现存概念集合。
54. IDENTIFY 关键概念。分类为 NEW / UPDATE / BUD，同 Mode A Step 13-14。
55. IF 影响页面 > 20 → 保留 top-20，其余记入 digest。
56. PRINT "📝 Extracting concepts... ({N} identified: {new} new, {update} updates, {bud} buds)"

### C.4 页面写入（参考 Mode B 的 B.2）

57. SNAPSHOT wiki/（用于 digest "before"）。
58. 执行写入（流程同 Mode B.2 的 Step 31-36，但无 proposal 引导，LLM 直接决策）：
    - NEW 页面：完整 frontmatter（含 clusters）+ 正文 + 脚注（mature 页面）
    - UPDATE 页面：append source_anchors（不删旧）、追加或精炼内容、更新 `updated`、重估 status / coverage（单调递增）
    - 发现矛盾时创建「观点分歧」section，不覆盖
    - BUD 页面：最小化 frontmatter + H1 + 一行定义存根

### C.5 交叉链接

59. 本次改动页面间维护双向 wikilink。

### C.6 Digest 生成

60. SNAPSHOT wiki/（"after"）。
61. 按 `templates/digest.md` COMPOSE（quick 模式不含「与 Proposal 的差异」section）。
62. DERIVE source-slug（per CLAUDE.md §9）。
63. WRITE `_digest/{YYYY-MM-DD}-{source-slug}.md` 和 `_digest/latest.md`。

### C.7 更新 `wiki/_index.md`

64. 重建 `wiki/_index.md`（per CLAUDE.md §2.6）。

### C.8 追加 `_log.md`

65. APPEND：
    ```
    ## [YYYY-MM-DD HH:MM] ingest:quick | {source-filename-list} | +{N} pages, +{N} links, +{N} buds
    ```

### C.9 Git Commit

66. CHECK 是否有文件变化（对比 Step 57 snapshot）。
67. IF 无变化 → PRINT "⚠️ No changes to commit" 并跳过 commit（但已追加 log，此时 log 那一行成为未提交改动；退回将 log 追加回滚）。
   - **实现细节**：如果 Step 57-C.7 全部无变化，则 Step 65 的 log 追加也回滚（删除刚追加的行），保持系统一致。
68. IF 有变化：
    - `git add` 显式列出所有改动：wiki/*.md（含 _index.md）、_digest/{...}.md、_digest/latest.md、_log.md。
    - `git commit -m "ingest: {source-filename}"`
    - PRINT "✅ Git commit: {short-hash}"

---

## 完成输出（Mode B & C 通用）

69. PRINT 汇总：
    ```
    ✅ Ingest complete: {new} new pages, {updated} updated, {links} links, {buds} buds
       📬 Digest: _digest/{YYYY-MM-DD}-{source-slug}.md
       📚 Index refreshed: wiki/_index.md
       Git commit: {short-hash} "{message}"
    ```
    若无变化：
    ```
    ⚠️ Ingest complete: no changes detected.
    ```
