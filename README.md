# opencode-skill-long-term-memory

> opencode 技能：教模型正确使用 [`opencode-mem`](https://www.npmjs.com/package/opencode-mem) 的 `memory` 工具 —— 主动召回、持久写入、逐层拨开搜索、作用域递进、成本感知批处理。

隶属 [`opencode-codex-kit`](https://github.com/Yulimfish/opencode-codex-kit)。

## 里面写了什么

技能覆盖：

- **RECALL 召回** —— 什么时候搜（会话开头、动 KB 之前、问用户之前、压缩之后）。
- **WRITE 写入** —— 什么样的记忆才够持久、够自包含。
- **PROFILE 画像** —— 自动学习的用户画像怎么工作。
- **作用域递进** —— `project` → `all-projects` → 问用户。
- **Layered Search 逐层拨开** —— 三层渐进查询策略，从宽泛领域 → 子话题 → 精确字段，带层数封顶与命中即停规则。
- **成本与失败模式** —— "Memory system is initializing" 时的重试策略。

## 前置

需要装 [`opencode-mem`](https://www.npmjs.com/package/opencode-mem) 并完成配置。向量默认走 [`opencode-codex-doubao-shim`](https://github.com/Yulimfish/opencode-codex-doubao-shim) 在 `:4748`，但任何 OpenAI 兼容的 embedding 端点都能用。

## 安装

```bash
mkdir -p ~/.config/opencode/skills
git clone https://github.com/Yulimfish/opencode-skill-long-term-memory.git \
  ~/.config/opencode/skills/long-term-memory
```

## 许可

MIT © Yulimfish
