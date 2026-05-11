接收 `$ARGUMENTS`。根据首词分发到三种子模式之一。

| 调用形式 | 子模式 | 行为 |
|---------|--------|------|
| `/query "<question>"` | default | 回答 + 询问「是否归档为 synthesis 页面？」 |
| `/query save "<question>"` | save | 回答并强制归档为 synthesis 页面 |
| `/query ephemeral "<question>"` | ephemeral | 只回答不落盘，不写 `_log.md` |

所有模式的回答都只基于 `wiki/` 内容（wiki-grounded），绝不使用外部知识，绝不引用 raw/ 文件。

---

## Step 0: Mode Dispatch

1. PARSE `$ARGUMENTS`。
2. SET first_word = 首个空格前的词（去掉可能的引号）。
3. IF first_word == "save" → 进入 **Mode Save**，剩余参数作为 question。
4. ELSE IF first_word == "ephemeral" → 进入 **Mode Ephemeral**，剩余参数作为 question。
5. ELSE → 进入 **Mode Default**，整个 `$ARGUMENTS` 作为 question。
6. IF question 为空 → PRINT 用法并 STOP：
   ```
   ❌ 用法:
      /query "<question>"              # 默认：回答 + 询问是否归档
      /query save "<question>"          # 强制归档
      /query ephemeral "<question>"     # 不落盘
   ```

---

## Step 1: 理解问题（所有模式通用）

7. PARSE question，识别：
   - 涉及的关键概念 / 实体
   - 期望的回答类型（定义、对比、原理、关系、应用）
8. PRINT "🔍 Analyzing question..."

## Step 2: 检索相关页面

9. LIST wiki/ 下所有 .md 文件（含 synthesis 页面）。
10. 对每个页面：READ frontmatter（title、aliases、status、page_type、clusters）和正文。
11. 评估相关度，依据：
    a. 标题 / 别名与问题概念匹配
    b. 正文关键词匹配
    c. Wikilink 邻近性（与已匹配页面有直连的页面加分）
    d. 同 cluster 加分
    e. synthesis 页面的 `generated_from_query` 与当前问题相似 → 显著加分（说明可能已有回答）
12. 按相关度排序，SELECT top-N（N ≤ 10），过滤极低相关度。
13. PRINT：
    ```
    📖 Found {N} relevant pages:
       • wiki/{name}.md [{status 图标} {page_type}] (relevance: {score}, clusters: [{...}])
    ```
    relevance 是 0.00-1.00 的主观估计。

## Step 3: 综合回答

14. IF 无相关页面：
    - PRINT：
      ```
      💬 知识库暂无与该问题相关的信息。
      💡 建议：投喂相关素材后再次查询（/ingest <file>）。
      ```
    - 进入 Step 6（log 收尾，不落盘回答）。
15. IF 有相关页面：
    - **已存在同问题 synthesis**：若检索到相关度高且 `generated_from_query` 与当前问题语义等价的 synthesis 页面 → 提示用户「已有综合页 [[...]] 可能已回答此问题，是否仍要重新回答？」。由用户判断是否继续。
    - SYNTHESIZE 回答：
      - 基于检索到的 wiki 页面内容
      - 对每个论点标注：`[source: wiki/{page}.md § "{section}"]`
      - synthesis 页面的引用标注：`[via [[{Synthesis-Page}]]]`
      - 逻辑组织，不是简单拼接
      - 使用中文
    - **红线**：只用 wiki/ 内容（B-1a）；绝不引用 raw/；绝不用外部知识。
16. PRINT：
    ```
    💬 Answer:
    
    {回答正文，每个论点附带来源标注}
    ```

## Step 4: 花苞提示

17. 在相关页面及其 wikilink 邻居中，CHECK 是否有 `status: bud` 的相关页面。
18. IF 有相关 bud：
    - PRINT：
      ```
      ⚠️ Related bud(s):
         • wiki/{name}.md — {为什么仍是花苞}，可能与问题相关
      💡 投喂相关素材可帮助这些花苞成长。
      ```
19. IF 无相关 bud → 跳过。

---

## Step 5: 回写决策

### Mode Ephemeral

20. 回答已打印。**完全不落盘**：不追加 `_log.md`、不生成 synthesis、不 git commit。
21. PRINT "(ephemeral: 本次查询未被记录)"。
22. STOP。

### Mode Save

23. 强制进入 Step 6「生成 synthesis 页面」。

### Mode Default

24. 基于回答评估是否值得归档（per CLAUDE.md §8「Synthesis 节制」）：
    - IF 回答涉及 ≥2 个 concept 页面且结论非显而易见 → 推荐归档
    - IF 仅复述单一页面的定义 → 推荐不归档
25. PRINT 询问：
    ```
    💾 归档选项：
       本次回答综合了 {N} 个 wiki 页面。{推荐文案}
       
       [1] 归档为 synthesis 页面（进入 Step 6）
       [2] 不归档，仅记录 log（进入 Step 7）
       [3] 完全不落盘（进入 Step 8，等同 ephemeral）
    
       请选择 [1/2/3]：
    ```
26. 等待用户回答。
27. SWITCH 用户选择：
    - [1] → Step 6
    - [2] → Step 7
    - [3] → Step 8

---

## Step 6: 生成 Synthesis 页面（Mode Save 或 Default-[1]）

28. DECIDE 页面名：
    - 对比类 → `{A}-vs-{B}.md`
    - 分析类 → `{Topic}-Analysis.md` 或 `{Topic-主要角度}-分析.md`
    - 避免与 concept 页面重名（如重名加 `-Synthesis` 后缀）
29. 按 `templates/synthesis-page.md` 结构 COMPOSE 内容：
    - frontmatter：page_type: synthesis，status: mature，source_coverage: 1.0，source_anchors: []，cites: [...relevant pages...]，generated_from_query: question，clusters: [相关 cluster]
    - H1 = title
    - TL;DR（2-3 句）
    - 分析（每个论点用 `[→ [[Page]] § "section"]` 标注）
    - 引用页面列表
    - 延伸问题（可选）
30. WRITE `wiki/{Synthesis-Page-Name}.md`。
31. 在 cites 中涉及的 concept 页面添加反向链接提示（在「与其他概念的关系」section 末尾追加一行 `- 被综合页 [[{Synthesis-Page}]] 引用`）。
32. 重建 `wiki/_index.md`（per CLAUDE.md §2.6）。

### 追加 `_log.md`

33. IF `_log.md` 不存在 → CREATE 并写入 `# LLM Wiki · 事件时间线\n`。
34. APPEND：
    - Mode Save：
      ```
      ## [YYYY-MM-DD HH:MM] query:save | "{question 截断 40 字}" | → wiki/{Synthesis-Page-Name}.md
      ```
    - Mode Default-[1]：
      ```
      ## [YYYY-MM-DD HH:MM] query:default-archived | "{question 截断 40 字}" | → wiki/{Synthesis-Page-Name}.md
      ```

### Git Commit

35. `git add wiki/{Synthesis-Page-Name}.md wiki/_index.md {cites 涉及的被追加反向链接的页面} _log.md`
36. `git commit -m "query: save \"{question 截断 40 字}\""`
37. PRINT：
    ```
    📝 Synthesis 页面已归档：wiki/{Synthesis-Page-Name}.md
       - 引用 {N} 个 wiki 页面
       - Clusters: [{...}]
       Git commit: {short-hash}
    ```
38. STOP。

---

## Step 7: 不归档但记录 log（Mode Default-[2]）

39. APPEND `_log.md`：
    ```
    ## [YYYY-MM-DD HH:MM] query:default-skipped | "{question 截断 40 字}" | (user chose not to archive)
    ```
40. `git add _log.md`
41. `git commit -m "log: query \"{question 截断 40 字}\""`
42. PRINT "📓 已记录到 _log.md，未生成 synthesis 页面。"
43. STOP。

---

## Step 8: 完全不落盘（Mode Default-[3]，等同 Ephemeral）

44. 不追加 log，不 commit。
45. PRINT "(ephemeral: 本次查询未被记录)"。
46. STOP。

---

## 附：log 追加规则（per CLAUDE.md §11）

- `_log.md` 只追加，绝不重写或删除已有行（B-7）
- Mode Ephemeral 和 Default-[3] 完全不写 log
- Mode Save、Default-[1]、Default-[2] 都写一行 log
- log 行的 `event-kind` 必须在 §11.3 枚举之内
