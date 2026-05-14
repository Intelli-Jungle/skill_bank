---
name: task-decomposer
description: |
  任务意图识别、规模评估（S/M/L）、大任务拆解并自动创建 GitHub Issue 分配角色。
  支持 调研/开发/修bug/测试验证/学习/文档/决策审批 七种任务类型。
  适用于爹助、狗助、狗爹的日常任务分解与分配。
version: 2.0.0
platforms: [linux, macos]
metadata:
  hermes:
    tags: [任务拆解, 意图识别, 多Agent协作, GitHub-Issues]
    related_skills: [agent-collaboration, verification-report]
---

# 任务拆解器（Task Decomposer）

## 概述

对任意任务描述进行分析：识别意图（调研/开发/修bug/测试验证/学习/文档/决策审批）→ 评估规模（S/M/L）→ 对大任务拆解为可独立完成的子任务 → 创建 GitHub Issue 并按角色分配。

**使用角色**：爹助、狗助、狗爹均可使用。

**协作仓库**：`Intelli-Jungle/hermes-agent-workflow`

## 第一阶段：意图识别

扫描任务描述中的关键词，匹配对应类型。**当多个类型同时匹配时，按优先级取最高者**（开发/修bug > 测试验证 > 调研 > 学习 > 文档 > 决策审批）。**关键词完全不匹配 → 兜底为 `type:research`，assignee: IntelliJungle（爹助人工判断）**。

| 类型标签 | 关键词（中文） | 关键词（英文） | 分配角色 | 工作目录 |
|---------|--------------|--------------|---------|---------|
| `type:feature` | 开发、实现、新增、增加、添加、写代码、功能、加一个 | feature、implement、add、build | JungleAssistant（狗助） | `features/` |
| `type:bug` | 修复、bug、报错、异常、不对、fix、修 | bug、fix、error、broken | JungleAssistant（狗助） | `bugfix/` |
| `type:verification` | 测试、验证、审查、QA、检查、review | test、verify、review、QA | IntelliJungle（爹助） | `verification/` |
| `type:research` | 调研、研究、分析、评估、对比、可行性 | research、analyze、evaluate、compare | IntelliJungle（爹助） | `research/` |
| `type:learning` | 学习、了解、入门、教程、怎么用 | learn、tutorial、howto | IntelliJungle（爹助） | `learning/` |
| `type:docs` | 文档、README、写文章、博客 | docs、README、article、blog | IntelliJungle（爹助） | `docs/` |
| `type:decision` | 决策、审批、确认、拍板、定方案 | approve、decide、confirm | zhangtbj（狗爹） | （仅 Issue 评论） |

**混合类型处理**：按优先级排序取最高者。优先级顺序：`type:feature` / `type:bug` > `type:verification` > `type:research` > `type:learning` > `type:docs` > `type:decision`。例如：任务同时包含"修复bug"和"写文档"→ 取 `type:bug`（优先级更高）。

**未知类型兜底**：关键词完全不匹配任何类型 → 归类为 `type:research`，assignee: IntelliJungle，附带提示 "⚠️ 未识别任务类型，默认归为调研，请爹助人工判断"。

## 第二阶段：规模评估

对照以下信号评估任务规模。**任一信号触及 L 级 → 整体判定为 L**。

| 信号 | S（小） | M（中） | L（大） |
|------|--------|---------|--------|
| 预估代码行数 | < 50 行 | 50–200 行 | 200+ 行 |
| 涉及文件数 | 1 个 | 2–3 个 | 4+ 个 |
| 验收标准数 | 1–2 条 | 3–4 条 | 5+ 条 |
| 外部依赖 | 0 个 | 1 个 | 2+ 个 |
| 跨模块 | 否 | 可能 | 是 |
| 预估工时 | < 1h（辅助参考，权重低） | 1–4h | > 4h（辅助参考，权重低） |

**S → 直接创建单个 Issue。M → 创建单个 Issue（带分节）。L → 进入拆解阶段。**

**事后上限检查**：任务完成后，如果实际代码量 > 预估的 2 倍，追加创建补充子任务，覆盖超出预估的范围内的工作。

## 第三阶段：任务拆解（仅 L 级）

### 拆解原则

1. **独立性**：每个子任务可独立完成，不依赖其他子任务的结果（尽量）
2. **可交付**：每个子任务有明确的验收标准和产出物
3. **粒度适中**：拆到 M 或 S 级别，目标是 M（S 太碎，L 没意义）
4. **深度限制**：**最多 2 层**（L → M → S）。禁止递归拆解，子任务到 M 或 S 就停
5. **依赖标记**：子任务间有依赖时，在父 Issue 中显式标记
6. **批量上限**：**单次最多 5 个子任务**。超过 5 个需分批创建

### 按类型的拆解维度

| 任务类型 | 拆解方式 | 示例 |
|---------|---------|------|
| 开发（feature） | 按模块/分层 | UI 层 → 逻辑层 → 数据层 |
| 开发（feature） | 按功能点 | 功能A → 功能B → 功能C |
| 调研（research） | 按子主题 | 方案A调研 → 方案B调研 → 对比总结 |
| 测试验证（verification） | 按测试类型 | 编译验证 → 功能测试 → 性能测试 |
| 学习（learning） | 按知识模块 | 基础概念 → 核心API → 实战项目 |

### 依赖关系标记

当子任务 B 依赖子任务 A 先完成时，在父 Issue 正文标记：

```markdown
## 依赖关系
- [#N2 子任务2](URL) **depends on** [#N1 子任务1](URL)（需 N1 完成后再开工）
- [#N3 子任务3](URL) 无依赖，可并行
```

避免两个 assignee 同时开工导致互相阻塞。

### 整体流程

```
输入任务 → 意图识别 → 规模评估 → [L级? → 拆解 → 创建子Issue] → 创建父Issue
```

## 第四阶段：创建 Issue

### 父 Issue 模板

```markdown
## 任务描述
[原始大任务描述]

## 子任务进度 (0/N done)
- [ ] [#N1 子任务1标题](URL) — assignee: @xxx
- [ ] [#N2 子任务2标题](URL) — assignee: @xxx

## 依赖关系
- [#N2](URL) depends on [#N1](URL)

## 验收标准
- [ ] 所有子任务完成
- [ ] 集成验证通过
```

标签：识别出的类型标签 + `priority:high`

### 子 Issue 模板

使用 `agent-collaboration` 标准 Issue 模板。标签使用第一阶段识别的类型。

### 创建命令

```bash
# 父 Issue
gh issue create --repo Intelli-Jungle/hermes-agent-workflow \
  --title "[任务] <标题>" \
  --label "type:<类型>,priority:high" \
  --assignee <角色> \
  --body-file /tmp/parent_body.md

# 子 Issue
gh issue create --repo Intelli-Jungle/hermes-agent-workflow \
  --title "[子任务] <标题>" \
  --label "type:<类型>,priority:high" \
  --assignee <角色> \
  --body-file /tmp/child_body.md
```

**批量限制**：单次最多创建 5 个子 Issue。超过需要分批，并记录剩余数量。

## 完整示例

**输入**："实现历史记录功能：保存/查看/清空/搜索，适配暗黑模式"

1. **意图识别**：开发 → `type:feature`，角色 → JungleAssistant
2. **规模评估**：4 AC、~240 行代码、3+ 文件 → **L 级**
3. **拆解为**：
   - #1 HistoryManager 数据层 (M) → JungleAssistant
   - #2 HistoryPanel UI 层 (M) → JungleAssistant
   - #3 验证测试 (M) → IntelliJungle
4. **依赖关系**：#3 depends on #1 + #2
5. **创建**：1 个父 Issue + 3 个子 Issue（在 5 个批量上限内）

## Skill Bank 地址

`Intelli-Jungle/skill_bank`
https://github.com/Intelli-Jungle/skill_bank

## 更新日志

### v2.0.0 (2026-05-14)
- **全文翻译为中文**（按狗爹要求）
- Skill Bank 地址更新：`jungle-skills` → `skill_bank`
- 新增 `evals.json` 评测集

### v1.1.0 (2026-05-14)
- 新增 `type:decision` 类型 → assignee: zhangtbj
- 新增未知类型兜底处理 → `type:research`，assignee: IntelliJungle
- 混合类型选择逻辑：从"首次匹配"改为"优先级最高者获胜"
- 新增事后上限检查（实际 > 2x 预估 → 补充子任务）
- 新增拆解深度限制：最多 2 层，禁止递归拆解
- 新增子任务间依赖关系标记
- 新增父 Issue 进度追踪（N/M done）
- 新增批量上限：单次最多 5 个子任务
- 预估工时信号降权为辅助参考
