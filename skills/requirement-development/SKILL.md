---
name: requirement-development
description: |
  需求开发 Skill：基于 AID 增量开发工作流，将自然语言需求转化为完整的 SPEC 报告、代码实现与编译验证。
  覆盖 NL 需求解析 → SPEC 报告生成 → 代码实现 → 编译验证 → 输出 apply-report 的全流程。
  面向 HarmonyOS/ArkTS 项目，支持 S/M/L 三级复杂度自适应。
version: 1.0.0
platforms: [macos, linux]
metadata:
  hermes:
    tags: [需求开发, AID工作流, SPEC报告, HarmonyOS, 编译验证]
    related_skills: [aid-workflow-incremental, writing-plans, subagent-driven-development]
---

# 需求开发 Skill（Requirement Development）

## 概述

当狗爹或爹助创建需求 Issue 并 assign 给狗助后，狗助使用本 Skill 进行完整的增量开发流程：

1. **NL 需求解析**：从 Issue body 提取自然语言需求，识别意图类型
2. **复杂度评估**：自动评估 S/M/L 级别，决定流程跳过策略
3. **SPEC 报告生成**：产出 proposal.md / delta-spec.md / delta-design.md / tasks.md
4. **代码实现**：按 tasks.md 逐任务实现（S 级可一次性完成）
5. **编译验证**：编译 HAP + 功能自测
6. **输出报告**：apply-report.md + Issue Comment 通知爹助

## 适用场景

- 狗爹创建需求 Issue，assignee = JungleAssistant
- 爹助分诊后指定 assignee = JungleAssistant
- Issue 标签为 `type:feature` / `type:a2h` / `type:bugfix`

## 触发条件

当 cronjob 检测到以下条件时自动加载本 Skill：
- Issue assignee = JungleAssistant
- Issue 包含 `type:feature` / `type:a2h` / `type:bugfix` 标签
- 或最新 comment 来自爹助，含「请狗助开发」「开发任务」等关键词

## 开发流程

### 阶段总览

```
NL 需求（Issue body）
    ↓
STEP 1: 意图识别 → 复杂度评估（S/M/L）
    ↓
STEP 2: SPEC 报告生成
    ├── S 级: proposal.md + delta-spec.md + tasks.md（跳过 delta-design.md）
    ├── M 级: 全部 5 份文档
    └── L 级: 全部文档 + 可能需要拆分需求
    ↓
STEP 3: 代码实现（feature 分支）
    ↓
STEP 4: 编译验证
    ↓
STEP 5: apply-report.md → 通知爹助审查
```

### STEP 1: 意图识别 + 复杂度评估

#### 1.1 意图分类

从 Issue body 提取需求类型：

| 类型 | 标记 | 说明 | 示例 |
|------|------|------|------|
| `requirement-add` | 新增功能 | 增加新 UI/逻辑 | 新增百分比按钮 |
| `requirement-change` | 功能变更 | 修改现有行为 | 历史记录排序修改 |
| `bugfix` | 缺陷修复 | 修复已知 bug | 除零异常修复 |
| `platform-migration` | 平台迁移 | Android → HarmonyOS | 单位转换器迁移 |
| `arch-refactor` | 架构优化 | 重构不改变行为 | 主题系统重构 |

#### 1.2 复杂度评估规则

| 级别 | 判定条件 | 涉及文件 | 阶段跳过 |
|------|---------|---------|---------|
| **S** | 单一领域，纯逻辑 / 纯 UI，1-2 文件 | ≤ 2 个 .ets 文件，≤ 150 行改动 | 跳过 delta-design.md（架构设计） |
| **M** | 跨 2-3 层/领域，需协调多个模块 | 3-5 个文件，150-500 行改动 | 完整流程 |
| **L** | 需新模块/架构变更/跨 4+ 文件 | 5+ 个文件，500+ 行改动 | 完整流程 + 需评估是否拆分 |

**评估原则**：
- 先在代码仓快速扫描关键文件（`search_files` 搜索相关类/方法），避免「在未读代码的情况下评估复杂度」
- S 级判定需自证：列出涉及文件清单 + 改动预估行数
- 不确定时上浮一级（宁高勿低）

### STEP 2: SPEC 报告生成

#### 2.1 产物清单

```
specs/changes/{YYYYMMDD}-{type}-{name}/
  ├── proposal.md       — 需求提案（rq-parse 输出）
  ├── delta-spec.md     — 增量规格（含验收标准 AC）
  ├── info.md           — 代码仓理解
  ├── delta-design.md   — 增量设计（S 级跳过）
  ├── tasks.md          — 任务分解
  └── todo.md           — 进度追踪
```

#### 2.2 proposal.md 模板

```markdown
# 需求提案: <标题>

## 意图分类
- **类型**: requirement-add / requirement-change / bugfix / platform-migration
- **复杂度**: S / M / L

## 需求描述
<从 Issue body 提取的 NL 需求，保持原文或稍作整理>

## 影响分析
| 维度 | 内容 |
|------|------|
| 涉及文件 | <文件清单> |
| 预估改动量 | +X -Y 行 |
| 依赖模块 | <列表或「无」> |
| 风险点 | <潜在风险> |

## 领域映射
- **UI 层**: <影响的 UI 组件>
- **逻辑层**: <影响的业务逻辑>
- **数据层**: <影响的数据/状态>
```

#### 2.3 delta-spec.md 模板

```markdown
# 增量规格: <标题>

## 功能范围
- **新增**: <新增功能点>
- **修改**: <修改功能点>
- **保持不变**: <明确不修改的部分>

## 验收标准 (AC)
| # | 验收标准 | 优先级 | 验证方式 |
|---|---------|:--:|:--:|
| AC1 | <标准1> | P0 | 实测/逻辑 |
| AC2 | <标准2> | P1 | 实测 |
| ... | ... | ... | ... |

## 边界条件
- 空输入: <行为>
- 超长输入: <行为>
- 异常输入: <行为>
- 快速连击: <行为>
```

#### 2.4 delta-design.md 模板（M/L 级使用）

```markdown
# 增量设计: <标题>

## 设计决策
| 决策点 | 方案A | 方案B | 选择 | 理由 |
|--------|-------|-------|:--:|------|
| <决策1> | <方案A> | <方案B> | A | <理由> |

## 组件交互
<描述新增/修改的组件如何与现有系统交互>

## 状态管理
<描述涉及的状态变更>
```

#### 2.5 tasks.md 模板

```markdown
# 任务分解: <标题>

| # | 任务 | 涉及文件 | 预估时间 | 依赖 | 状态 |
|---|------|---------|:--:|------|:--:|
| T1 | <任务1> | <文件> | 30min | 无 | pending |
| T2 | <任务2> | <文件> | 1h | T1 | pending |

## 验收检查清单
- [ ] 编译通过
- [ ] AC1 通过
- [ ] AC2 通过
- [ ] 原有功能不受影响
```

#### 2.6 todo.md 模板

```markdown
# 进度追踪: <标题>

| 阶段 | 状态 | 完成时间 |
|------|:--:|------|
| PLANNING | done | <时间> |
| IMPLEMENTING | in_progress | — |
| APPLYING | pending | — |

## 注意事项
- <需要特别关注的点>
```

### STEP 3: 代码实现

#### 3.1 分支创建

```bash
# 在目标项目中创建 feature 分支
cd /path/to/project
git checkout main && git pull origin main
git checkout -b feat/<feature-name>
mkdir -p specs/changes/{YYYYMMDD}-{type}-{name}
```

#### 3.2 代码修改策略（参照 aid-workflow-incremental 经验）

**S 级需求（首选策略）**：
1. 心中已有完整方案（通过 STEP 1-2 分析得出）
2. 用顶层 `write_file` 直接写入完整文件（单文件修改时最可靠）
3. 避免 `execute_code write_file`（会追加而非覆盖 .ets 文件）
4. 避免 `patch` 工具对短字符串的假匹配

**M/L 级需求**：
1. 可使用 `delegate_task` 委托子代理进行代码生成
2. 分文件逐个实现，每个文件修改后 `git diff` 验证
3. 多个任务按 tasks.md 依赖顺序执行

**ArkTS 文件黄金规则**：
- ❌ 禁止 `execute_code write_file` 写 .ets（追加模式导致行号重复）
- ❌ 谨慎使用 `patch` 对短字符串（容易假匹配到错误位置）
- ✅ 多改动时：`write_file` 一次性写入完整文件
- ✅ 单改动时：`patch` + 足够长的上下文（前后 3-5 行唯一行）

#### 3.3 实现验证

每完成一个 task：
```bash
git diff --stat          # 确认改动范围
git diff HEAD -- '*.ets' # 审查代码变更
```

### STEP 4: 编译验证

#### 4.1 环境检测

先检查本地是否有 HarmonyOS SDK：
```bash
# 检查 DevEco Studio 安装
ls /Applications/DevEco-Studio.app/Contents/tools/hvigor/bin/hvigorw 2>/dev/null && echo "SDK AVAILABLE" || echo "SDK NOT AVAILABLE"
```

#### 4.2 SDK 可用时：执行编译

```bash
export DEVECO_SDK_HOME=/Applications/DevEco-Studio.app/Contents/sdk/default/openharmony
export HW=/Applications/DevEco-Studio.app/Contents/tools/hvigor/bin/hvigorw
export JAVA_HOME=/Applications/DevEco-Studio.app/Contents/jbr/Contents/Home
export PATH="/Applications/DevEco-Studio.app/Contents/tools/node/bin:$PATH"

$HW assembleHap --mode module -p product=default -p buildMode=debug --no-daemon 2>&1
```

#### 4.3 SDK 不可用时：逻辑验证 + 委托审核

若本地无 DevEco Studio（如纯 macOS 环境）：
1. 在 apply-report.md 中标注 `⚠️ 本地无SDK，待爹助编译验证`
2. 通过代码逻辑推导验证每个 AC
3. 完成后 assign 给 IntelliJungle，开验证 Issue
4. **不自行合并代码** — 等爹助编译通过后再 merge

**委托验证的 Issue 模板**：
```
Title: [验证] feat/<branch> — <feature summary>
Body:
- 分支: feat/<branch>
- 仓库: <repo-url>
- 改动: git diff --stat 输出
- AC 清单: 逐条列出
- 编译命令: hvigorw assembleHap --mode module -p product=default -p buildMode=debug --no-daemon
```

### STEP 5: 输出 apply-report.md + 流转

#### 5.1 apply-report.md 模板

```markdown
# 应用报告: <标题>

**分支**: feat/<branch-name>
**时间**: <YYYY-MM-DD HH:MM UTC>
**复杂度**: S / M / L

## 改动摘要
```
<git diff --stat 输出>
```

## AC 对照
| AC | 描述 | 实现方式 | 状态 |
|----|------|---------|:--:|
| AC1 | <标准> | <简述> | [PASS] |
| AC2 | <标准> | <简述> | [PASS] |

## 编译状态
- [x] 本地编译通过 / 🟡 本地无SDK（委托爹助验证）

## 自测结果
| 场景 | 输入 | 预期 | 实际 | 状态 |
|------|------|------|------|:--:|
| <场景1> | <输入> | <预期> | <实际> | [PASS] |

## 经验教训
- <开发中遇到的关键问题和解决方案>

## 下一步
1. 爹助编译验证 + 代码审查
2. 爹助 UI 实测
3. 狗爹终审
```

#### 5.2 Issue 流转

完成后执行（分两步，避免双 assignee）：
```bash
gh issue edit <N> --repo Intelli-Jungle/hermes-agent-workflow --remove-assignee JungleAssistant
gh issue edit <N> --repo Intelli-Jungle/hermes-agent-workflow --add-assignee IntelliJungle
```

Comment 中写明：
- 完成摘要
- 分支/commit 链接
- SPEC 报告路径
- **应用评测结果**（跑完 evals.json 后的通过率）

## 约束

- **所有 comment 必须用中文**（狗爹硬性要求）
- **先读代码再写 spec**（避免过度设计 — 参照 aid-workflow pitfall #1）
- **S 级不写 delta-design.md**（节省时间，参照 aid-workflow pitfall #2）
- **改动量需自证** — 附 `git diff --stat` 输出
- **不被 Issue body 的改动声称欺骗** — 自己用 diff 验证
- **不自行合并代码** — 等爹助验证通过
- **每个 Issue 只有一个 assignee** — 换人时先 remove 再 add

## 与其他 Skill 的关系

- `aid-workflow-incremental`：本 Skill 基于其四阶段流程，是本 Skill 的核心框架
- `writing-plans`：用于在 PLANNING 阶段生成详细实施方案
- `subagent-driven-development`：M/L 级需求可在 IMPLEMENTING 阶段使用子代理加速
- `requirement-verification`：爹助的检视 Skill — 狗助开发完成的产物由该 Skill 进行验证

## Cron Job 模式

以 cron job 身份运行时（无交互用户）：
- S 级需求：PLANNING → IMPLEMENTING → APPLYING 在一个 session 内完成
- M/L 级需求：完成 PLANNING 后，在 comment 中确认方案，再继续 IMPLEMENTING
- 跳过阶段确认，直接推进
- 完成后开 Issue 委托验证

## Skill Bank

地址：`Intelli-Jungle/skill_bank`
https://github.com/Intelli-Jungle/skill_bank
