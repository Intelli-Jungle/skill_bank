---
name: requirement-development
description: |
  需求开发 Skill：基于 sdd-group2026 AID 增量开发工作流，将自然语言需求转化为结构化 SPEC 报告、TDD 驱动的代码实现与编译验证。
  覆盖 NL 需求解析（rq-parse 领域映射 + rq-clarify 澄清）→ SPEC 报告生成（需求SPEC v1.2 对齐）→ 代码实现（TDD 强制）→ 编译验证 → apply-report 的全流程。
  面向 HarmonyOS/ArkTS 项目，支持 S/M/L/XL 四级复杂度自适应。
version: 2.0.0
platforms: [macos, linux]
metadata:
  hermes:
    tags: [需求开发, AID工作流, SPEC报告, HarmonyOS, 编译验证, TDD]
    related_skills: [aid-workflow-incremental, writing-plans, subagent-driven-development, requirement-verification]
    upstream: https://github.com/sdd-group2026/skill_bank/tree/main/skills/working/aid-workflow
---

# 需求开发 Skill（Requirement Development）v2.0.0

> **v2.0.0 更新**: 对齐 sdd-group2026/skill_bank 最新 aid-workflow 方法论。
> 核心增强: rq-parse 领域映射 + rq-clarify 澄清分类 + rq-codebase 按需分析 + TDD 强制执行 (M/L级) + aid-reviewing 设计自审。

## 概述

当狗爹或爹助创建需求 Issue 并 assign 给狗助后，狗助使用本 Skill 进行完整的增量开发流程：

```
NL 需求（Issue body）
    ↓
STEP 1: 需求解析（rq-parse 领域映射 + rq-clarify 模糊点澄清）
    ↓
STEP 2: SPEC 报告生成（对齐需求SPEC v1.2 模板）
    ↓
STEP 3: 代码实现（TDD RED-GREEN-REFACTOR，M/L级强制）
    ↓
STEP 4: 编译验证（SDK 预检 + 编译 + 测试）
    ↓
STEP 5: 设计自审 + apply-report → 通知爹助审查
```

**与 sdd-group2026 aid-workflow 的关系**: 本 Skill 是 sdd aid-workflow 的**裁剪+封装+cron适配版**。保留了完整的 PLANNING→IMPLEMENTING→APPLYING 三段式理念，但内联了子 Skill 的核心方法论（rq-parse/rq-clarify/rq-codebase/tdd-enforcer/aid-reviewing/mod-design），移除了交互式门控以适配 cron job 无人值守模式。

## 适用场景

- 狗爹创建需求 Issue，assignee = JungleAssistant
- 爹助分诊后指定 assignee = JungleAssistant
- Issue 标签为 `type:feature` / `type:a2h` / `type:bugfix`

## 触发条件

当 cronjob 检测到以下条件时自动加载本 Skill：
- Issue assignee = JungleAssistant
- Issue 包含 `type:feature` / `type:a2h` / `type:bugfix` 标签

---

## 开发流程

### STEP 1: 需求解析（对齐 rq-parse + rq-clarify）

#### 1.1 意图分类（5 类型，对齐 aid-planning step1）

从 Issue body 提取需求类型：

| 类型 | 标识 | 特征 | 目录命名 |
|------|------|------|---------|
| `requirement-add` | 新增功能 | 当前不存在的能力 | `{YYYYMMDD}-requirement-add-{name}` |
| `requirement-change` | 功能变更 | 已有功能的改进 | `{YYYYMMDD}-requirement-change-{name}` |
| `bugfix` | 缺陷修复 | 现有行为不符预期 | `{YYYYMMDD}-bugfix-{name}` |
| `arch-refactor` | 优化或重构 | 架构/性能/DFX | `{YYYYMMDD}-arch-refactor-{name}` |
| `platform-migration` | 平台迁移 | 跨平台迁移 | `{YYYYMMDD}-platform-migration-{name}` |

#### 1.2 领域映射（对齐 rq-parse domain-taxonomy）

将需求映射到 HarmonyOS ArkTS 的 9 个技术领域：

| 领域 | 关键词信号 | 典型涉及 |
|------|-----------|---------|
| UI/交互 | 页面、界面、按钮、列表、动画、布局 | `arkts-component-builder` |
| 数据管理 | 存储、数据库、缓存、持久化 | `arkts-data-layer` |
| 网络通信 | 请求、接口、API、下载、上传 | `arkts-download-manager` |
| 媒体播放 | 播放、暂停、音频、视频、后台 | `arkts-media-playback` |
| 导航路由 | 页面跳转、Tab、返回、路由 | `arkts-navigation-builder` |
| 状态管理 | 同步、刷新、共享数据、全局状态 | `arkts-state-manager` |
| 系统能力 | 权限、文件、相册、通知、后台任务 | `arkts-system-capabilities` |
| 国际化 | 多语言、翻译、本地化 | `arkts-i18n` |
| WebView | 网页、浏览器、H5、注入 | `arkts-webview-manager` |

#### 1.3 影响层次分析（MVVM 分层）

```
┌─────────────────────────────────────────┐
│  View 层（pages/, components/）         │  ← UI 结构、交互、样式
├─────────────────────────────────────────┤
│  ViewModel 层（viewmodels/）            │  ← 业务逻辑、状态转换
├─────────────────────────────────────────┤
│  Model 层（models/）                    │  ← 数据模型定义
├─────────────────────────────────────────┤
│  数据层（database/, network/, parser/） │  ← 持久化、网络、解析
├─────────────────────────────────────────┤
│  基础设施（common/, helpers/, workers/）│  ← 工具、常量、后台任务
└─────────────────────────────────────────┘
```

对每个层次标注变更类型：`new`（新增）/ `modify`（修改）/ `extend`（扩展）/ `none`（无影响）。

#### 1.4 关键实体提取

识别需求中的：
- **数据实体**: 需要新增/修改的数据模型
- **页面实体**: 需要新增/修改的页面
- **接口实体**: 需要新增/修改的 API 接口
- **事件实体**: 需要处理的用户事件

#### 1.5 模糊点标记与澄清（对齐 rq-clarify）

扫描需求描述中的模糊点，按严重程度分为三级：

| 级别 | 含义 | 处理策略（cron模式） |
|------|------|---------------------|
| **blocker** | 不澄清无法开发 | 记录为「待定假设」，采用最合理的默认值并标注 |
| **important** | 影响质量，尽量澄清 | 记录默认方案 + 理由 |
| **nice-to-have** | 告知用户，不强制追问 | 记录即可，不阻塞流程 |

**Cron 模式澄清策略**: 由于无人值守，对 blocker 级模糊点：
1. 列出 2-3 种常见方案
2. 选择最保守/最合理的方案
3. 在 proposal.md 中用 `⚠️ Assumption:` 显式标注假设
4. 在 Issue comment 中请爹助/狗爹确认

**提问模板**（当不是 cron 模式可交互时使用）:
- 范围模糊 → 边界界定法: "包括 A/B/C 哪些？有没有明确不做的？"
- 行为模糊 → 场景还原法: "场景1/2/3 下的预期行为？"
- 交互模糊 → 原型描述法: "方案A vs 方案B，倾向哪个？"

#### 1.6 复杂度评估（对齐 aid-planning step6，6 维度）

| 维度 | S (简单) | M (中等) | L (复杂) | XL (超大) |
|------|---------|---------|---------|----------|
| 功能范围 | 单个功能点，1-2 模块 | 2-3 模块 | 4+ 模块 | 多子系统 |
| 代码改动 | < 100 行 | 100-500 行 | 500-3000 行 | > 3000 行 |
| 数据模型 | 无需变更 | 扩展字段 | 新增模型 | 新增数据层 |
| 架构影响 | 无变更 | 无变更 | 需要变更 | 架构重构 |
| 接口变更 | 现有接口扩展 | 新增接口 | 新增/修改多接口 | 接口层重构 |
| 依赖关系 | 单一模块依赖 | 2-3 模块依赖 | 多模块交叉依赖 | 跨系统依赖 |

**判定规则**:
- **S 级**: 所有维度 ≤ S 标准 → 跳过 delta-design.md、跳过 TDD（可选编译验证）
- **M 级**: 任一维度达 M 标准 → 完整流程、强制执行 TDD
- **L 级**: 任一维度达 L 标准 → 完整流程 + 需评估是否拆分为 2-5 个子任务
- **XL 级**: 任一维度达 XL 标准 → 必须先拆 Epic → 对每个 Epic 递归执行本 Skill

**评估原则**:
- 先在代码仓快速扫描关键文件（`search_files` 搜索相关类/方法），避免「在未读代码的情况下评估复杂度」
- S 级判定需自证：列出涉及文件清单 + 改动预估行数
- 不确定时上浮一级（宁高勿低）

---

### STEP 2: SPEC 报告生成（对齐 sdd proposal-template + 需求SPEC v1.2）

#### 2.1 产物清单与跳过策略

```
specs/changes/{YYYYMMDD}-{type}-{name}/
  ├── proposal.md       — 需求提案（含 rq-parse 输出 + 领域映射 + 影响层次）
  ├── delta-spec.md     — 增量规格（含验收标准 AC，对齐需求SPEC v1.2）
  ├── info.md           — 代码仓理解（对齐 rq-codebase 按需分析）
  ├── delta-design.md   — 增量设计（S 级跳过，M/L 级调用 mod-design 方法）
  ├── design-review.md  — 设计自审报告（M/L 级，对齐 aid-reviewing D1-D8）
  ├── tasks.md          — 任务分解（对齐 aid-implementing 1-2h/任务粒度）
  └── todo.md           — 进度追踪
```

| 复杂度 | proposal | delta-spec | info | delta-design | design-review | tasks | todo |
|--------|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| **S** | ✅ | ✅ | ✅ | ❌ 跳过 | ❌ 跳过 | ✅ | ✅ |
| **M** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **L** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ (拆2-5子任务) | ✅ |
| **XL** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ (Epic拆解) | ✅ |

#### 2.2 proposal.md 模板（对齐 sdd proposal-template.md）

```markdown
# {功能名称} 需求提案

## 1. 需求概述
### 1.1 功能名称
### 1.2 需求类型: {类型}
### 1.3 需求描述: {从 Issue body 提取}
### 1.4 使用场景
### 1.5 预期效果

## 2. 需求解析报告（rq-parse 输出）
### 2.1 意图分类
- **类型**: {类型}
- **一句话概括**: ...

### 2.2 领域映射
- {领域1}: {说明}
- {领域2}: {说明}

### 2.3 影响层次
| 层次 | 变更类型 | 说明 |
|------|---------|------|
| View | new/modify/extend/none | ... |
| ViewModel | ... | ... |
| Model | ... | ... |
| 数据层 | ... | ... |
| 基础设施 | ... | ... |

### 2.4 关键实体
- **数据实体**: {列表}
- **页面实体**: {列表}
- **接口实体**: {列表}

### 2.5 模糊点
| # | 类别 | 描述 | 严重程度 |
|---|------|------|---------|
| 1 | 范围模糊 | ... | blocker |
| 2 | ... | ... | important |

### 2.6 复杂度评估
- **等级**: S / M / L / XL
- **影响模块数**: {数量}
- **架构变更**: 是/否
- **数据模型变更**: 是/否
- **理由**: ...

## 3. 需求澄清（rq-clarify 输出）
### 3.1 已澄清的关键决策
| # | 问题 | 决策 | 来源 |
|---|------|------|------|
| 1 | {模糊点} | {默认方案} | ⚠️ Assumption（等确认） |

### 3.2 功能边界
- **包含**: ...
- **不包含**: ...

## 4. 影响范围分析
### 4.1 涉及的模块
| 模块 | 现有功能 | 新增/修改 |
|------|---------|-----------|

### 4.2 代码依赖分析
{依赖图/调用链简述}

## 5. 跳过策略
| 制品 | 是否产出 | 理由 |
|------|---------|------|
| delta-design.md | 是/否 | {S级跳过/需要} |
| design-review.md | 是/否 | {S级跳过/需要} |
```

#### 2.3 delta-spec.md 模板（对齐需求SPEC v1.2 核心要素）

```markdown
# 需求SPEC: {功能名称}

## 系统背景
> ≤200字描述：系统为谁解决什么问题

## 需求描述
| 需求编号 | 需求名称 | 需求描述 | 优先级 |
|---------|---------|---------|:--:|
| REQ-001 | {名称} | {描述} | P0 |

## 场景用例
### 场景 SC-001: {场景名称}
#### 前置条件
#### 主成功路径
1. {步骤1}
2. {步骤2}
#### 扩展路径（异常处理）
1a. {异常场景} → {处理方式}

## 验收标准 (AC)
| # | 验收标准 | 优先级 | 验证方式 |
|---|---------|:--:|:--:|
| AC1 | {标准} | P0 | 实测/逻辑 |
| AC2 | {标准} | P1 | 实测 |

## 边界条件
| 场景 | 输入 | 预期行为 |
|------|------|---------|
| 空输入 | ... | ... |
| 超长输入 | ... | ... |
| 异常输入 | ... | ... |
| 快速连击 | ... | ... |
| 边界值 | ... | ... |

## 假设和约束
| # | 约束类别 | 约束描述 | 对设计的影响 |
|---|---------|---------|------------|

## 功能影响列表
| 功能编号 | 功能描述 | 影响类型 | 详细变更点 |
|---------|---------|---------|-----------|
```

#### 2.4 info.md 模板（对齐 rq-codebase 按需分析）

**核心原则**: 按需分析，聚焦重点。根据需求类型确定侧重点：

| 分析类别 | 新增功能 | 功能变更 | 缺陷修复 | 架构优化 | 平台迁移 |
|---------|:--:|:--:|:--:|:--:|:--:|
| 架构分析 | ✓✓ | ○ | ○ | ✓✓ | ✓✓ |
| 影响范围 | ✓ | ✓✓ | ✓✓ | ✓ | ○ |
| 现有代码 | ○ | ✓✓ | ✓✓ | ✓ | ✓ |
| 公共组件 | ✓✓ | ○ | ○ | ✓✓ | ○ |
| 接口/契约 | ✓✓ | ✓✓ | ○ | ○ | ✓✓ |
| 风险识别 | ✓ | ○ | ✓ | ✓✓ | ✓✓ |

> ✓✓=重点分析，✓=次要分析，○=可简化

```markdown
# 代码仓理解: {功能名称}

## 需求类别
- **类型**: {类型}
- **分析侧重**: {重点分析类别}

## 仓库结构
{目录树（仅相关部分）}

## 关键文件
| 文件 | 用途 | 本次影响 |
|------|------|:--:|
| {path} | {描述} | modify/none |

## 依赖关系
{调用链/依赖图简述}

## 关键接口
| 接口 | 签名 | 用途 |
|------|------|------|

## 风险点
- {风险1}: {缓解措施}
```

#### 2.5 delta-design.md 模板（M/L 级，对齐 mod-design D1-D10 简化版）

```markdown
# 增量设计: {功能名称}

## D1. 设计概述
- 设计目标 / 设计范围 / 设计原则

## D2. 组件设计
| 组件 | 类型 (新增/修改/复用) | 职责 | 对外接口 |
|------|:--:|------|---------|

## D3. 数据模型
| 实体 | 字段 | 约束 |
|------|------|------|

## D4. 路由/导航设计
{页面跳转/导航路径}

## D5. 状态管理
{状态定义/流转}

## D6. 错误处理
| 错误场景 | 处理方式 | 用户提示 |
|---------|---------|---------|

## D7. 安全考虑
{输入校验/权限/注入防护}

## D8. 性能考虑
{渲染优化/内存/网络}

## 设计决策记录
| 决策点 | 方案A | 方案B | 选择 | 理由 |
|--------|-------|-------|:--:|------|
```

#### 2.6 tasks.md 模板（对齐 aid-implementing，1-2h/任务粒度）

```markdown
# 任务分解: {功能名称}

| # | 任务 | 涉及文件 | 预估 | 依赖 | AC | 状态 |
|---|------|---------|:--:|------|----|:--:|
| T1 | {任务} | {文件} | 1h | 无 | {AC#} | pending |
| T2 | {任务} | {文件} | 2h | T1 | {AC#} | pending |

## 验收检查清单
- [ ] 所有 AC 通过
- [ ] 编译通过 (`hvigorw assembleHap`)
- [ ] 单元测试通过 (`hvigorw test@entry`)
- [ ] Lint 通过
- [ ] 原功能回归正常
```

---

### STEP 3: 代码实现（TDD 驱动，对齐 tdd-enforcer）

#### 3.1 分支创建

```bash
cd /path/to/project
git checkout main && git pull origin main
git checkout -b feat/{YYYYMMDD}-{type}-{name}
mkdir -p specs/changes/{YYYYMMDD}-{type}-{name}
```

#### 3.2 TDD 执行策略（按复杂度分级）

**TDD 适用范围**: 新增功能 / 功能变更 / 缺陷修复 / 优化或重构。
**例外**（可跳过 TDD）: 纯 UI 样式调整（无逻辑）、配置文件变更、自动生成代码。

| 复杂度 | TDD 策略 |
|:--:|------|
| **S** | 可选。代码实现→编译验证即可。建议至少对核心逻辑写 1 个测试。 |
| **M** | **强制** RED-GREEN-REFACTOR。每个 task 必须经过完整循环。 |
| **L** | **强制** RED-GREEN-REFACTOR。每个子任务独立 TDD 循环。 |
| **XL** | **强制**。先拆 Epic，再逐 Epic 执行 M/L 级 TDD。 |

#### 3.3 RED-GREEN-REFACTOR 循环（M/L 级强制执行）

**核心铁律: 没有失败的测试，就不写生产代码。**

```
RED（写失败测试）→ 验证失败 → GREEN（最小代码通过）→ 验证通过 → REFACTOR（重构）→ 验证仍通过
```

##### RED - 写失败测试
```typescript
// 针对 AC 写最小测试，一个测试只验证一个行为
test('AC1: percent button appears in row 3 col 2', () => {
  const page = render(CalculatorPage);
  const percentBtn = page.getByText('%');
  expect(percentBtn).toBeVisible();
  expect(percentBtn.parent.row).toBe(3);
});
```

##### 验证 RED
```bash
hvigorw test@entry 2>&1 | tail -5
# 必须看到测试 FAIL（不是报错），失败原因是功能缺失
```

- 测试**通过**了？→ 说明你在测试已有行为，修改测试。
- 测试**报错**了？→ 修复语法错误，重新运行直到正确失败。

##### GREEN - 最小代码通过
- 只写让测试通过的最少代码
- 不写"将来可能用到"的代码
- 不优化，不重构——先让它通过

##### 验证 GREEN
```bash
hvigorw test@entry 2>&1 | tail -5
# 必须看到所有测试 PASS
```

##### REFACTOR
- 消除重复
- 改善命名
- 提取公共逻辑
- **重构后必须重新跑测试确认仍通过**

#### 3.4 ArkTS 代码修改策略（保持不变）

**S 级需求（首选策略）**:
1. 心中已有完整方案（通过 STEP 1-2 分析得出）
2. 用顶层 `write_file` 直接写入完整文件（单文件修改时最可靠）
3. 避免 `execute_code write_file`（会追加而非覆盖 .ets 文件）
4. 避免 `patch` 工具对短字符串的假匹配

**M/L 级需求**:
1. 可使用 `delegate_task` 委托子代理进行代码生成
2. 分文件逐个实现，每个文件修改后 `git diff` 验证
3. 多个任务按 tasks.md 依赖顺序 + TDD 循环执行

**ArkTS 文件黄金规则**:
- ❌ 禁止 `execute_code write_file` 写 .ets（追加模式导致行号重复）
- ❌ 谨慎使用 `patch` 对短字符串（容易假匹配到错误位置）
- ✅ 多改动时：`write_file` 一次性写入完整文件
- ✅ 单改动时：`patch` + 足够长的上下文（前后 3-5 行唯一行）

#### 3.5 实现验证

每完成一个 task 的 TDD 循环：
```bash
git diff --stat          # 确认改动范围
git diff HEAD -- '*.ets' # 审查代码变更
hvigorw test@entry       # M/L级确认测试通过
```

---

### STEP 4: 编译验证 + 测试

#### 4.1 环境检测

```bash
ls /Applications/DevEco-Studio.app/Contents/tools/hvigor/bin/hvigorw 2>/dev/null && echo "SDK AVAILABLE" || echo "SDK NOT AVAILABLE"
```

#### 4.2 SDK 可用时：完整验证

```bash
export DEVECO_SDK_HOME=/Applications/DevEco-Studio.app/Contents/sdk/default/openharmony
export HW=/Applications/DevEco-Studio.app/Contents/tools/hvigor/bin/hvigorw
export JAVA_HOME=/Applications/DevEco-Studio.app/Contents/jbr/Contents/Home
export PATH="/Applications/DevEco-Studio.app/Contents/tools/node/bin:$PATH"

# 第一步：Lint 检查
$HW lint 2>&1

# 第二步：单元测试（M/L级）
$HW test@entry 2>&1

# 第三步：编译 HAP
$HW assembleHap --mode module -p product=default -p buildMode=debug --no-daemon 2>&1
```

#### 4.3 SDK 不可用时：逻辑验证 + 委托审核

若本地无 DevEco Studio（如纯 macOS 环境）：
1. 在 apply-report.md 中标注 `⚠️ 本地无SDK，待爹助编译验证`
2. 通过代码逻辑推导验证每个 AC
3. 完成后 assign 给 IntelliJungle，不自行合并代码
4. **SDK 版本预检（#36 实战沉淀）**: 编译前检查项目 `build-profile.json5` 的 SDK 版本是否与本地匹配，不匹配时提前报出而非编译到一半才失败

---

### STEP 5: 设计自审 + apply-report

#### 5.1 设计自审（M/L 级，对齐 aid-reviewing 轻量模式）

在交给爹助审查前，先对 SPEC 制品进行快速自审（D1-D8 缺陷检测）：

**快速检测流程**（D1→D2→D5→D7→D4→D3→D6→D8）:

| 维度 | 检测目标 | 常见问题 |
|------|---------|---------|
| **D1** 模糊词 | 替换为定量标准 | "尽量"、"尽可能"、"适当" |
| **D2** 完整性 | 缺失类别/场景/字段 | AC 不完整、边界条件缺失 |
| **D5** 追溯性 | ID/引用断裂 | AC→功能→代码无法追溯 |
| **D7** 可验证性 | 非二元判定 | "用户体验好"不可测 |
| **D4** 准确性 | 虚假引用/错误信息 | 文件路径不存在、API 签名错误 |
| **D3** 一致性 | 文档间矛盾 | proposal vs delta-spec 范围不一致 |
| **D6** 可行性 | 技术上不可行 | 使用不存在的 API |
| **D8** 可维护性 | 结构混乱/冗余 | 文档章节重复 |

产出 `design-review.md`:
```markdown
# 设计自审报告: {功能名称}

## 检测结果
| D维度 | 严重程度 | 位置 | 问题 | 修复状态 |
|-------|:--:|------|------|:--:|
| D1 | 🟡 | delta-spec AC3 | "适当调整"→改为具体像素值 | ✅ 已修复 |
| D2 | 🔴 | delta-spec | 缺少空输入边界条件 | ✅ 已补充 |

## 结论
- [x] 所有 🔴 问题已修复
- [x] 所有 🟡 问题已修复或标注
- [x] 制品齐全、一致、可验证
```

#### 5.2 apply-report.md 模板

```markdown
# 应用报告: {功能名称}

**分支**: feat/{branch-name}
**时间**: {YYYY-MM-DD HH:MM UTC}
**复杂度**: S / M / L / XL

## 改动摘要
```
{git diff --stat 输出}
```

## SPEC 制品清单
| 制品 | 状态 | 位置 |
|------|:--:|------|
| proposal.md | done | specs/changes/... |
| delta-spec.md | done | ... |
| info.md | done | ... |
| delta-design.md | done/skipped (S级) | ... |
| design-review.md | done/skipped | ... |
| tasks.md | done | ... |

## AC 对照矩阵
| AC | 描述 | 实现方式 | 验证方式 | 状态 |
|----|------|---------|---------|:--:|
| AC1 | {标准} | {简述} | 逻辑/实测/TDD | [PASS] |

## TDD 执行记录（M/L 级）
| Task | RED (测试编写) | GREEN (代码通过) | REFACTOR | 最终状态 |
|------|:--:|:--:|:--:|:--:|
| T1 | ✅ | ✅ | ✅ | PASS |
| T2 | ✅ | ✅ | ⏭️ 无需重构 | PASS |

## 编译状态
- [x] 本地编译通过 / 🟡 本地无SDK（委托爹助验证）
- [x] 单元测试通过 (X/X) / ⏭️ S级跳过
- [x] Lint 通过 / 🟡 S级跳过

## 自测结果
| 场景 | 输入 | 预期 | 实际 | 状态 |
|------|------|------|------|:--:|
| {场景} | {输入} | {预期} | {实际} | [PASS] |

## 设计自审结果（M/L 级）
{D1-D8 检测结论摘要}

## 经验教训
- {开发中遇到的关键问题和解决方案}

## 下一步
1. 爹助编译验证 + 代码审查（使用 requirement-verification Skill）
2. 爹助 UI 实测
3. 狗爹终审
```

#### 5.3 Issue 流转

完成后执行（分两步，避免双 assignee）：
```bash
gh issue edit <N> --repo Intelli-Jungle/hermes-agent-workflow --remove-assignee JungleAssistant
gh issue edit <N> --repo Intelli-Jungle/hermes-agent-workflow --add-assignee IntelliJungle
```

Comment 中写明：
- 完成摘要
- 分支/commit 链接
- SPEC 报告路径
- TDD 执行记录（M/L级）
- 设计自审结论
- **应用评测结果**（跑完 evals.json 后的通过率）

---

## 约束

- **所有 comment 必须用中文**（狗爹硬性要求）
- **先读代码再写 spec**（避免过度设计 — Step 1.4 代码仓扫描）
- **S 级不写 delta-design.md + design-review.md**（节省时间）
- **M/L 级强制执行 TDD RED-GREEN-REFACTOR**
- **改动量需自证** — 附 `git diff --stat` 输出
- **不被 Issue body 的改动声称欺骗** — 自己用 diff 验证
- **不自行合并代码** — 等爹助验证通过
- **每个 Issue 只有一个 assignee** — 换人时先 remove 再 add
- **SDK 版本预检** — 编译前检查项目 SDK 版本 vs 本地 SDK 版本

## 与其他 Skill 的关系

| Skill | 关系 | 说明 |
|-------|------|------|
| sdd-group2026 aid-workflow | **上游方法论** | 本 Skill 是它的裁剪+封装+cron适配版 |
| requirement-verification | **下游互补** | 爹助使用该 Skill 审查本 Skill 产出的代码和 SPEC |
| writing-plans | **辅助** | PLANNING 阶段生成详细实施方案 |
| subagent-driven-development | **辅助** | M/L 级需求在 IMPLEMENTING 阶段使用子代理加速 |
| task-decomposer | **辅助** | L/XL 级需求的拆分决策 |

### 与 sdd-group2026 aid-workflow 的核心差异

| 维度 | requirement-development v2.0.0 | sdd-group2026 aid-workflow |
|------|-------------------------------|---------------------------|
| 架构 | 单体自包含（1 个 SKILL.md） | 多 Skill 编排（14+ 子 Skill） |
| 交互门控 | 无，cron 无人值守自主推进 | 每阶段结束等用户确认 |
| TDD | M/L 级强制执行 | 全部强制执行 |
| 设计审视 | M/L 级自审 D1-D8 | step9 完整审视流程 |
| SDK 预检 | ✅ 编译前版本匹配检查 | ❌ 无 |
| Issue 流转 | ✅ 内置 assignee 分步变更 | ❌ 无（非 GitHub Issues 协作） |

## Cron Job 模式

以 cron job 身份运行时（无交互用户）：
- S 级需求：PLANNING → IMPLEMENTING → APPLYING 在一个 session 内完成
- M/L 级需求：完成 PLANNING 后，在 comment 中确认方案，继续 IMPLEMENTING
- 跳过阶段确认，直接推进
- Block 级模糊点用 `⚠️ Assumption` 标注默认方案
- 完成后 assign 给爹助审查

## Skill Bank

地址：`Intelli-Jungle/skill_bank`
https://github.com/Intelli-Jungle/skill_bank

## evals.json

评测集位于 `skills/requirement-development/evals.json`，覆盖：
1. 意图识别（2 条）
2. 领域映射（2 条）
3. 复杂度评估（2 条）
4. SPEC 报告生成（2 条）
5. TDD 执行（1 条）
6. Issue 流转（1 条）

共 10 条测试用例。
