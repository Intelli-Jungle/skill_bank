---
name: task-decomposer
description: |
  任务意图识别、规模评估（XS/S/M/L/XL）、大任务拆解并自动创建 GitHub Issue 分配角色。
  支持 调研/开发/修bug/测试验证/学习/文档/决策审批 七种任务类型。
  适用于爹助、狗助、狗爹的日常任务分解与分配。
version: 3.0.0
platforms: [linux, macos]
metadata:
  hermes:
    tags: [任务拆解, 意图识别, 多Agent协作, GitHub-Issues, 规模评估]
    related_skills: [agent-collaboration, verification-report]
---

# 任务拆解器（Task Decomposer）

## 概述

对任意任务描述进行分析：识别意图（调研/开发/修bug/测试验证/学习/文档/决策审批）→ 评估规模（XS/S/M/L/XL）→ 对大任务拆解为可独立完成的子任务 → 创建 GitHub Issue 并按角色分配。

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

### 规模分级标准（XS/S/M/L/XL）

| 规模 | 代码行数 | 涉及文件数 | 典型复杂度 | 判定方式 |
|------|---------|-----------|-----------|---------|
| **XS** | < 100 行 | 1 个文件 | 单点修改（文案、配置、颜色、简单属性） | 代码行数为主，文件数为辅 |
| **S** | < 200 行 | < 4 个文件 | 小功能/单文件重构/简单bug修复 | 代码行数为主，文件数为辅 |
| **M** | 200–1000 行 | < 10 个文件 | 中等功能/多文件重构/涉及测试 | 代码行数为主，文件数为辅 |
| **L** | 1000–3000 行 | < 50 个文件 | 大功能/跨模块/有新架构引入 | 代码行数为主，文件数为辅 |
| **XL** | > 10000 行 | > 100 个文件 | Epic 级别/跨系统/多Sprint | 任一触及即 XL |

> 📐 **3000–10000 行过渡区间**：此区间按代码行数进入 L 级上限，按文件数可能触 XL。判定规则：代码行数 ≤ 3000 且文件数 < 50 → L；代码行数 > 3000 或文件数 ≥ 50 → 升级为 **L+**（接近 XL，强烈建议先拆为 2-3 个 L 级再进 L 拆解管线）；代码行数 > 10000 或文件数 > 100 → XL。

> ✳ **判据优先级**：代码行数为第一判据，文件数为第二判据。当代码行数判定为某级但文件数远超该级上限时（如 200 行但涉及 20 个文件），提升一级。**任一信号触及 XL 条件 → 整体判定为 XL**（Epic，必须拆分为多个 L 级以下子任务）。

### 业界对标

本分级体系参考以下业界最佳实践，为每个数值提供可信来源：

| 来源 | 核心方法 | 对标关系 |
|------|---------|---------|
| **Mike Cohn** "Agile Estimating and Planning" (2005) | 故事点（1/2/3/5/8/13/20）Fibonacci 序列，Planning Poker | XS≈1pt, S≈2-3pt, M≈5pt, L≈8-13pt, XL≈20+pt |
| **Atlassian Agile Guide** | T-shirt Sizing + "一个 Sprint 内完成"原则 | XS/S/M 在 1 Sprint 内；L 可能跨 Sprint；XL = Epic（跨多 Sprint） |
| **SAFe (Scaled Agile Framework)** | Story 应在 1-5 天完成；Epic 跨多个 PI（8-12 周） | XS/S→<2天, M→2-5天, L→5-10天, XL→Epic（跨 PI） |
| **Google 工程实践** | 设计文档阶段即拆分；无任务超过 ~2 engineer-weeks | L 上限 ≈ 2 周；超过即为 XL/Epic |
| **Microsoft 1ES** | "小 PR"原则；工作项不应跨多个 Sprint | M 以下→单 PR；XL 拒绝进入 Sprint 待办 |

**关键共识**（五大来源一致结论）：
1. **XL = 必须拆分**：任何超过一个 Sprint（2周）的任务必须分解（Atlassian、SAFe、Google 共同立场）
2. **LOC 是辅助指标**：代码行数与复杂度非线性相关（Mike Cohn、Atlassian 明确警告），但作为自动化判据有实用价值——50 行复杂算法 ≠ 50 行样板代码
3. **相对估算优于绝对估算**（Planning Poker 核心理念）：XS-XL 五级制比精确行数估算更可靠，因为它捕捉了"相对于其他任务有多大"而非"绝对有多大"
4. **小批量高频交付**：Google 和 Microsoft 的实践一致表明，小任务（S/M）的 throughput 和质量都显著优于大任务（L/XL）

**参考文献**：
- Mike Cohn, "Agile Estimating and Planning" (Prentice Hall, 2005). https://www.mountaingoatsoftware.com/agile/planning-poker
- Atlassian Agile Guide — User Stories: https://www.atlassian.com/agile/project-management/user-stories
- Atlassian Agile Guide — Estimation: https://www.atlassian.com/agile/project-management/estimation
- SAFe Story: https://scaledagileframework.com/story/
- Jeff Patton, "User Story Mapping" (O'Reilly, 2014)
- Bill Wake, "Twenty Ways to Split Stories" (Agile Alliance): https://www.agilealliance.org/glossary/story-splitting/

### 拆解决策

| 规模 | 拆解决策 | 说明 |
|------|---------|------|
| **XS** | **不拆解** | 直接创建单个 Issue。可选：2-3 个相关 XS 合并为一个 S 级 Issue（批量合并） |
| **S** | **不拆解** | 直接创建单个 Issue（带分节） |
| **M** | **弹性** | 能清晰分出 2 个独立完成状态 → 拆为 2×S；逻辑强耦合 → 保持 1×M |
| **L** | **必须拆解** | 拆为 2-5 个 M/S 级子任务。不拆不进 Sprint |
| **XL** | **必须拆解（Epic）** | 先拆为多个 L 级子 Epic，再各走 L 拆解管线。XL 本身不进 Sprint，仅容器/追踪 |

**事后上限检查**：任务完成后，如果实际代码量 > 预估的 2 倍，追加创建补充子任务。

## 第三阶段：任务拆解

### 拆解原则

1. **独立性**：每个子任务可独立完成，尽量不依赖其他子任务
2. **可交付**：每个子任务有明确的验收标准和产出物
3. **粒度适中**：目标粒度 S。XS 太碎，M 不够细
4. **深度限制**：**最多 2 层**（L → M/S → S/XS）。禁止递归拆解
5. **依赖标记**：子任务间有依赖时，在父 Issue 中显式标记
6. **批量上限**：**单次最多 5 个子任务**。超过 5 个需分批创建

### 按规模的拆解策略

#### XS 级任务
直通车。可考虑**批量合并**：将 2-3 个相关的 XS 任务合并为一个 S 级 Issue（如"批量文案修正"），避免碎片化。

#### S 级任务
标准处理。无需拆解，直接创建单个 Issue。完成即 close。

#### M 级任务
弹性处理。判断标准：能否清晰描述两个独立的"完成"状态。能 → 拆分；不能 → 不拆。

#### L 级任务
必须拆解。按任务类型选择拆解维度：

| 任务类型 | 拆解方式 | 目标子任务数 | 示例 |
|---------|---------|------------|------|
| 开发（feature） | 按分层架构 | 2-4 个 M/S | UI层 → 逻辑层 → 数据层 → 集成测试 |
| 开发（feature） | 按功能模块 | 2-5 个 M/S | 模块A → 模块B → 模块C → 联调 |
| 调研（research） | 按子主题 | 2-4 个 M/S | 方案A调研 → 方案B调研 → 对比分析 → 结论 |
| 测试验证（verification） | 按测试层次 | 2-4 个 M/S | 编译验证 → 功能测试 → UI验证 → 性能 |
| 学习（learning） | 按知识模块 | 2-4 个 M/S | 基础概念 → 核心API → 实战项目 → 总结 |

**拆解优先级**：优先按**业务流程**切分（垂直切片，端到端可验证），其次按**技术层次**切分（水平切片，需额外集成步骤）。此原则来自 SAFe 的 vertical slicing 和 Atlassian 的 "split by workflow step"。

#### XL 级任务
Epic 处理。**第一刀必须按子系统/业务域切分**，而非技术层次：
- 示例：计算器 App → 基础运算 / 科学运算 / 历史记录 / 单位换算 / 设置
- 每个子 Epic 单独走 L 级拆解流程

### 依赖关系标记

```markdown
## 依赖关系
- [#N2](URL) **depends on** [#N1](URL)（需 N1 完成后再开工）
- [#N3](URL) 无依赖，可并行
```

### 整体流程

```
输入任务 → 意图识别 → 规模评估（XS/S/M/L/XL）
    ↓
XS → 直接创建（可选批量合并）
S  → 直接创建
M  → 判断可拆 → 可则 2×S，否则 1×M
L  → 必须拆 → 2-5×M/S → 父+子 Issue
XL → Epic拆分 → 多×L → 递归 ×M/S → 父+子 Issue
```

## 第四阶段：创建 Issue

### 父 Issue 模板

```markdown
## 任务描述
[原始大任务描述]

## 规模评估
- 评级：L（预估 ~1500 行，~15 文件）
- 拆解为 N 个子任务

## 子任务进度 (0/N done)
- [ ] [#N1 子任务1](URL) — assignee: @xxx (S)
- [ ] [#N2 子任务2](URL) — assignee: @xxx (M)

## 依赖关系
- [#N2](URL) depends on [#N1](URL)

## 验收标准
- [ ] 所有子任务完成
- [ ] 集成验证通过
```

标签：类型标签 + `priority:high`

### 子 Issue 模板

使用 `agent-collaboration` 标准 Issue 模板。可选附加 `size:XS`/`size:S`/`size:M` 标签。

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

**批量限制**：单次最多 5 个子 Issue。超过分批，记录剩余数量。

## 完整示例

### 示例 1：L 级开发任务

**输入**："实现历史记录功能：保存/查看/清空/搜索，适配暗黑模式"

1. **意图识别**：开发 → `type:feature`，角色 → JungleAssistant
2. **规模评估**：~500 行、6 个文件 → **M 级**（200-1000 行，<10 文件）
3. **M 级可拆分**（AC 独立）→ 拆为：
   - #1 HistoryManager 数据层 (S) → JungleAssistant
   - #2 HistoryPanel UI 层 (S) → JungleAssistant
   - #3 验证测试 (S) → IntelliJungle
4. **依赖关系**：#3 depends on #1 + #2
5. **创建**：1 父 + 3 子

### 示例 2：XL 级 Epic

**输入**："开发完整 OpenCalc：基础运算、科学运算、历史记录、单位换算、主题切换、手势操作、语音输入"

1. **意图识别**：开发 → `type:feature`
2. **规模评估**：> 15000 行，> 200 文件 → **XL**
3. **第一刀（子系统切分）**：
   - Epic-1: 基础运算 (L) / Epic-2: 科学运算 (L) / Epic-3: 历史记录 (M)
   - Epic-4: 单位换算 (L) / Epic-5: 主题切换 (M) / Epic-6: 手势操作 (L) / Epic-7: 语音输入 (L)
4. **分批**：先 5 个，后 2 个；每个 L 级递归走 L 拆解

### 示例 3：XS 批量合并

**输入**："按钮文案'确认'改'确定'，标题改'OpenCalc'，icon 换新"

1. **意图识别**：开发 → `type:feature`
2. **规模评估**：< 50 行，1 文件 → **XS**
3. **合并**：3×XS → 1×S "批量 UI 文案与资源更新"
4. **创建**：1 个 Issue

## Skill Bank 地址

`Intelli-Jungle/skill_bank`
https://github.com/Intelli-Jungle/skill_bank

## 更新日志

### v3.0.0 (2026-05-14)
- **规模体系重构**：S/M/L 三级制 → XS/S/M/L/XL 五级制（按狗爹要求）
- **新增业界对标章节**：引用 Mike Cohn、Atlassian、SAFe、Google、Microsoft 五大权威来源
- **新增 3000–10000 行过渡区间（L+）处理规则**
- **新增按规模差异化拆解策略**：XS 批量合并、M 弹性拆分、L 强制拆解、XL Epic 流程
- **示例扩充**：新增 L、XL Epic、XS 批量合并三个完整示例
- **判据优化**：代码行数为主判据，文件数为辅，任一触及即升级

### v2.0.0 (2026-05-14)
- 全文翻译为中文；Skill Bank 地址更新 `jungle-skills` → `skill_bank`；新增 `evals.json`

### v1.1.0 (2026-05-14)
- 新增 `type:decision`；未知类型兜底；混合类型优先级算法；事后上限检查；拆解深度/依赖/批量上限
