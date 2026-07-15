---
name: long-term-memory
description: Use to leverage persistent semantic memory across sessions. Backed by opencode-mem plugin — automatic capture, semantic dedup, and user-profile learning. Vectors go through the local doubao-shim (port 4748) into doubao-embedding-vision. Call memory({mode:"search"}) proactively at the start of any non-trivial task. Call memory({mode:"add"}) when the user explicitly says "记住" / "以后都要" / "remember". Call memory({mode:"profile"}) to inspect what the plugin has learned about the user.
---

# Long-Term Memory (opencode-mem backed)

Single tool `memory`, several `mode` values. Vectors are produced by `doubao-embedding-vision-250615` (2048-dim) via the local shim on `:4748` — transparent to this tool.

Automatic parts (do not manage manually):

- User prompts and session turns are auto-captured, deduplicated, and semantically indexed.
- A rolling user profile refreshes every 10 turns.
- Context compaction preserves the most relevant memories.

**Your responsibilities**: recall proactively, pin explicitly, inspect on request.

## Cheat-sheet

```
memory({ mode: "search",  query: "架构决策 SQLite", scope: "project", limit: 6 })
memory({ mode: "search",  query: "user preference commit style", scope: "all-projects", limit: 6 })
memory({ mode: "add",     content: "自包含的一句事实（含日期+主题+原因）" })
memory({ mode: "list",    limit: 10 })
memory({ mode: "profile" })
memory({ mode: "forget",  memoryId: "mem_..." })
```

## Scope Hierarchy — always start narrow

| Step | Scope           | When                                              |
| ---- | --------------- | ------------------------------------------------- |
| 1    | `project`       | **Default.** Anything about the current codebase. |
| 2    | `all-projects`  | Only if step 1 returns 0 useful hits.             |
| 3    | ask the user    | Only if both scopes miss.                         |

Rationale: same query hits the same shard first (cheap), and only widens on miss (still cheap). Never call `all-projects` without trying `project` first — you'll waste embedding calls and get noise from unrelated projects.

## When to `search`

Call **before** doing any of these:

- Editing a project or subsystem not touched this session.
- Making an arch / stack / dependency / naming / format decision.
- Composing a clarifying question — search first, the answer may already be recorded.
- Continuing work after a compaction (`session.compacted` event).
- User uses referential language: "上次那个 / 之前的 / 还记得吗 / 一样处理 / 老规矩".

Query = user's own words + one technical keyword. Example: user says "还是按上次那个格式" → search `"格式 输出 语雀"` in `scope:"project"`.

If a hit is used, cite it in your first line: `（依据 memory mem_1784… · 2026-07）`.

## Layered Search — 逐层拨开

**Semantic search 是一次性的，但你的意图往往是分层的。** 一枪打中很难，逐层拨开更稳。三层递进，每一层用上一层的结果收敛下一层的关键词，像剥洋葱。

```
┌── Layer 1 ────────────────┐  宽泛主题 / 领域
│  query: "opencode 配置"    │  limit: 8   scope: project
│  → 拿回来的是"话题地图"     │  作用：确认这个主题有没有记录、有哪些子话题
└────────────────────────────┘
              ↓ 从命中里提炼子话题（比如出现了 "记忆维度" / "shim 端口" / "guardrails"）
┌── Layer 2 ────────────────┐  子话题 / 具体系统
│  query: "opencode-mem      │  limit: 6   scope: project
│         embedding 维度"     │  作用：定位到具体决策/配置块
│  → 拿回来的是"决策段落"     │
└────────────────────────────┘
              ↓ 从段落里锁定字段/文件/命令名
┌── Layer 3 ────────────────┐  精确字段 / 值
│  query: "embeddingDimensions│  limit: 3   scope: project
│         2048 doubao"        │  作用：拿到唯一那条权威事实
│  → 拿回来的就是要 cite 的那条│
└────────────────────────────┘
```

### 什么时候要分层，什么时候直接一枪

**一枪即可**（skip layering）：
- 用户给的关键词已经很精确（有文件名/字段名/命令名）。
- 你知道自己在找什么、就是想 verify 一下。
- 主题小、记忆库里显然不会有几百条。

**必须分层**：
- 用户只给了模糊领域（"关于语雀那些事"、"opencode 配置的坑"）。
- 你需要先了解"这个话题下都记过什么"再决定下一步。
- 第一次 search 命中数 ≥ 6 且相关度分散（说明主题太宽）。
- 用户明确说"逐层看看 / 拨开找 / 分层探索 / 展开一下"。

### 分层执行规则

1. **每层 limit 递减**：`8 → 6 → 3`。宽层要覆盖，窄层要精准。
2. **每层 query 都要基于上一层的结果**，不要凭空重开。上一层出现的高频关键词直接接进下一层。
3. **≤ 3 层封顶**。第 3 层还找不到就是没这条记忆，扩到 `all-projects` 或问用户，别再加层了。
4. **中途命中就停**。第 1 层就拿到了唯一权威条，直接引用，不要为了"完整"再往下走。
5. **每层报一行进展**（超过 4 次工具调用触发 progress note 规则）：`第 1 层：主题地图 8 条 → 锁定子话题 X`。

### 分层 vs `all-projects` 逃逸

- 分层是**在同一 scope 内**收敛。
- `all-projects` 是**换 scope**。
- 二者独立：如果 `project` 层 1 就 0 命中，直接换 `all-projects` 层 1，别在空集上继续分层。

### 反面模式

- 层与层用同一个 query（等于白花 embedding 调用）。
- 拿不到理想结果就往里加更长的自然语言句子（"关于 opencode 配置的 embedding 维度那个坑到底是"），embedding 相似度对长句不友好。**关键词化**：3–6 个词。
- 分了层但每层结果都不看就无脑跳下一层（丢失聚焦意义）。

## When to `add`

Fire on ANY of:

- Explicit signal: "记住 / 以后都用 / 别再问我这个 / 我说过 / remember / 从今天开始".
- A decision that will govern future sessions (arch, tool, credential location, coding style).
- A non-obvious pitfall or footgun discovered this session that would waste time next time.
- A confirmed user-preferred workflow or command sequence.

**Never `add`**:

- Session-scoped state (WIP, todos, transient errors).
- Already-present facts (`search` first).
- Secrets, tokens, PII, API keys.
- Verbose logs — pin the *conclusion*, not the trace.

**Content shape** (self-contained, dated, cross-project readable):

```
<Bad>  用第一个
<Good> 2026-07 决策 · opencode-mem 配置字段名是 embeddingDimensions（复数）；
       写成 embeddingDim 会静默回退 768 维，与 shim 2048 维冲突。
```

## When to `profile`

Only when the user explicitly asks what has been learned about them. Reply with a summary, never paste the raw JSON dump.

## When to `list`

Rare — only for chronological browsing at user request. Semantic `search` is almost always better.

## Cost model

- `search` / `add` = 1 doubao embedding call (~200–300 ms) + local cosine scan.
- **Cheap**. Always prefer searching over asking the user.

## Failure modes

| Symptom                                       | Meaning                                       | Action                                                            |
| --------------------------------------------- | --------------------------------------------- | ----------------------------------------------------------------- |
| `"Memory system is initializing"`             | shim not yet reachable                        | wait 2 s, retry once; then proceed without and note in summary   |
| `"Memory system not configured properly"`     | config missing / wrong URL / wrong dim        | tell user to check `opencode-mem.jsonc`, keep going without      |
| `search` returns 0 in `project`               | not yet learned in this project                | escalate to `scope:"all-projects"`                                |
| `Vector backend degraded to exact-scan`       | USearch index corrupt (dim mismatch was fix)   | `mem-doctor.sh` → if persistent, reset shard files                |

## Graph WebUI

Not this skill's job — see `memory-graph-ui`. Short version: opencode-mem serves it at `http://127.0.0.1:4747` while opencode is running.
