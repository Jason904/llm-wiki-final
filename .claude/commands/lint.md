接收 `$ARGUMENTS`。根据首词分发到三档之一（`quick` / `standard` / `deep`），默认 `standard`。

| 调用形式 | 子模式 | 检查项 |
|---------|--------|--------|
| `/lint` | standard | 孤岛 + 来源缺失 + 花苞绽放 + 重复 + 死链 + synthesis cites |
| `/lint quick` | quick | 孤岛 + 来源缺失 + 花苞绽放 |
| `/lint deep` | deep | standard + 矛盾 + 过时声明 + 脚注密度 + 覆盖缺口建议 |

---

## Step 0: Mode Dispatch

1. PARSE `$ARGUMENTS`。
2. SET first_word = 首个词（去掉空白）。
3. IF first_word == "quick" → 进入 **Level: quick**
4. ELSE IF first_word == "deep" → 进入 **Level: deep**
5. ELSE → 进入 **Level: standard**（first_word 为空或任何其他值）
6. PRINT "🔍 Running knowledge lint ({level})..."

---

## Step 1: 收集数据（所有档通用）

7. LIST wiki/ 下所有 .md 文件（不含 `index.md`，index 不计入任何检查）。
8. 对每个文件，READ 并 PARSE：
   - frontmatter：title、aliases、page_type、status、clusters、source_anchors、cites、generated_from_query、updated
   - 正文：提取所有 `[[wikilink]]` 目标、脚注 `[^id]`
9. BUILD 索引：
   ```
   page_name → {
     page_type,        # concept | synthesis
     status,
     clusters,
     source_anchors,
     cites,            # synthesis 的引用列表
     outgoing_links,   # 正文中的 wikilink 目标
     incoming_links,   # 反向链接（排除来自 _index.md 的）
     body_word_count,
     footnote_count,
     updated
   }
   ```
10. COMPUTE 统计：
    - total_pages（不含 index.md）、count_by_status（mature/growing/bud）、count_by_page_type（concept/synthesis）
    - cluster 分布：每个 cluster 的页面数和各 status 计数

---

## Step 2: 基础三项（所有档）

### Check 1: 孤岛检测

11. 对索引中每个页面（排除 `index.md`）：
    - IF incoming_links 为空（`index.md` 的链接不计入）→ 加入 orphans 列表，记录 outgoing link 数和所在 cluster。

### Check 2: 来源缺失

12. 对索引中每个 `page_type: concept` 且非 bud 的页面：
    - IF source_anchors 为空或缺失 → 加入 missing_sources 列表。
13. 对 synthesis 页面不检查 source_anchors（固定为 []）。

### Check 3: 即将绽放的花苞

14. 对每个 `status: bud` 的页面：
    - COUNT incoming_links 数量（排除来自 _index.md）。
    - IF incoming_links ≥ 3 → 加入 ready_to_grow 列表。

---

## Step 3: 标准扩展（standard / deep，quick 跳过）

### Check 4: 重复候选

15. 启发式预筛选（避免 O(n²)）：
    - 只比较标题共享单词、aliases 有重叠、或首段定义共享 3+ 关键词的页面对。
    - synthesis 页面不参与重复检测（它们本就是独特的问答产物）。
16. 对预筛选出的页面对，评估相似度：
    a. 标题 / 别名相似度
    b. 首段定义重叠度
    c. 共享 wikilink 数量
17. 按阈值分级（per CLAUDE.md §10.2）：
    - ≥ 0.90：高度重复，标注「建议合并」
    - 0.80-0.89：疑似重复，标注「考虑合并」
    - 0.70-0.79：可能重复，标注「请人工确认」
    - < 0.70：不报告

### Check 5: 死链检测

18. 对每个页面的 outgoing_links：
    - IF 目标页面在 wiki/ 不存在 → 加入 dead_links 列表，记录源页面、行号、不存在的目标。

### Check 6: Synthesis cites 完整性

19. 对每个 synthesis 页面：
    - IF cites 为空 → 加入 synthesis_empty_cites 列表（违反 §2.8 规则）。
    - 对每个 cite 路径：IF 目标文件不存在 → 加入 synthesis_broken_cites 列表。

---

## Step 4: 深度扩展（deep 专属，quick / standard 跳过）

### Check 7: 矛盾检测

20. 在同 cluster 或共享 ≥2 个 wikilink 的页面对中：
    - 扫描正文中关于共享概念的陈述，识别显性冲突（per CLAUDE.md §10.3）：
      - 相同概念的否定陈述（"X 是 Y" vs "X 不是 Y"）
      - 相反的数量级（"100 倍" vs "10 倍"）
      - 相反的因果方向（"A 导致 B" vs "B 导致 A"）
21. 加入 contradictions 列表，附带证据行。**绝不自动修改任何页面**（B-4）。

### Check 8: 过时声明

22. 对每个 `status: mature` 的 concept 页面：
    - 对其 source_anchors 中每个 file：获取 raw 文件的系统 mtime。
    - IF wiki 页面 `updated` 显著早于某 source 的 mtime（超过 30 天）→ 加入 stale_claims 列表，标注「可能有新版资料未吸收」。

### Check 9: Mature 脚注密度

23. 对每个 `status: mature` 的 concept 页面：
    - COMPUTE ratio = footnote_count / (body_word_count / 100)
    - IF ratio < 1（每 100 字不足 1 个脚注）→ 加入 thin_footnotes 列表。

### Check 10: 覆盖缺口 + 搜索建议

24. 对每个 `status: bud` 或 `status: growing` 的页面：
    - IF incoming_links ≥ 3 → 加入 coverage_gaps 列表。
25. 对每个 coverage gap 概念，**生成搜索关键词建议**（不执行搜索！per CLAUDE.md §5 B-1b）：
    - 基于概念名和引用它的页面上下文，给出 3-5 个可供用户手动搜索的关键词
    - 这些建议只写入 lint 报告，**绝不写入 wiki**

---

## Step 5: 输出终端

26. PRINT：
    ```
    🔍 Knowledge Lint · {level}

    📊 Wiki Stats:
       Total: {total} pages | 🌳 {N} 🌿 {N} 🌱 {N} | 🔗 {synthesis_N} synthesis
       Clusters: {N} ({top-3 cluster 名和页面数})

    🏝️ Islands (no incoming): ({count})
       • wiki/{name}.md — 0 incoming, {N} outgoing, cluster: {Cluster}
       [或 "None found ✅"]

    📎 Missing Sources: ({count})
       • wiki/{name}.md — no source_anchors in frontmatter
       [或 "None found ✅"]

    🌱 Buds Ready to Grow (3+ references): ({count})
       • wiki/{name}.md — referenced by {N} pages
       [或 "None — all buds still young"]
    ```

27. IF level ∈ {standard, deep}：
    ```
    🔄 Potential Duplicates: ({count})
       • wiki/{a}.md ↔ wiki/{b}.md
         similarity: {score} — {级别}, {reason}

    🔗 Dead Links: ({count})
       • wiki/{source}.md → [[{missing-target}]]

    📎 Synthesis cites 完整性: ({count issues})
       • wiki/{synthesis}.md → {broken cite}
       [或 "All cites valid ✅"]
    ```

28. IF level == deep：
    ```
    ⚠️ Suspected Contradictions: ({count})
       • wiki/{a}.md ↔ wiki/{b}.md (shared cluster: {cluster})
         {A 表述} vs {B 表述}

    📅 Stale Claims: ({count})
       • wiki/{name}.md last updated {date}; source {file} modified {mtime}

    📖 Thin Footnotes (mature pages): ({count})
       • wiki/{name}.md — {words} words, {footnotes} footnotes, density {ratio}

    🧭 Coverage Gaps + Search Suggestions: ({count})
       • [[{name}]] (status: bud, referenced by {N})
           → 建议搜索：{keyword-1}, {keyword-2}, {keyword-3}
    ```

29. 末尾：
    ```
    ✅ Lint complete ({level}). Report saved to .lint/{YYYY-MM-DD}-{level}.md
    ```

---

## Step 6: 写入报告文件

30. 按 `.templates/lint-report.md` 结构 COMPOSE 报告，根据 level 保留对应 section。
31. WRITE `.lint/{YYYY-MM-DD}-{level}.md`。
32. WRITE `.lint/latest-report.md`（覆盖，与归档文件内容一致）。

---

## Step 7: 更新 `index.md`

33. 复用 Step 1 收集的数据，按 CLAUDE.md §2.6 规范重新生成 `index.md`：
    - 按 cluster 分组
    - cluster 内按 status 降序，同 status 内按字母排序
    - synthesis 页面单独集中在文末「🔗 综合页（Synthesis）」章节
    - 统计行：总页面 / 各 status / synthesis 数 / cluster 数
34. 全量覆盖 `index.md`。

---

## Step 8: 追加 `log.md`

35. IF `log.md` 不存在 → CREATE 并写入 `# LLM Wiki · 事件时间线\n`。
36. APPEND 一行（per CLAUDE.md §11）：
    ```
    ## [YYYY-MM-DD HH:MM] lint:{level} | all wiki | {核心发现摘要}
    ```
    核心发现摘要示例：
    - `2 islands, 1 duplicate, 0 missing-source`
    - `deep: 1 contradiction, 3 stale, 2 coverage-gaps`

---

## Step 9: Git Commit

37. `git add .lint/{YYYY-MM-DD}-{level}.md .lint/latest-report.md index.md log.md`
38. IF 文件无实质变化（如同一天同档次重复运行且 wiki 未变）→ PRINT "⚠️ No changes to commit" 并跳过 commit（回滚 log 追加以保持一致）。
39. ELSE `git commit -m "lint: {level} {YYYY-MM-DD}"`
40. PRINT：
    ```
    ✅ Lint commit: {short-hash}
       📊 Report: .lint/{YYYY-MM-DD}-{level}.md
       📚 Index refreshed: index.md
    ```
