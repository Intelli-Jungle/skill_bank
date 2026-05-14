---
name: requirement-verification
description: |
  需求检视 Skill：对开发完成的需求 Issue 进行编译验证、差分对比、模拟器 UI 测试、AC 对照矩阵审查。
  输出标准化的 Issue Comment 验证报告，同时保存到 verification/<分支>_<日期>.md。
  遵循狗爹批准的 Issue #9 验证报告格式。
version: 1.0.0
platforms: [macos]
metadata:
  hermes:
    tags: [需求检视, 验证报告, HarmonyOS, 编译验证, UI测试]
    related_skills: [verification-report, harmonyos-build-debugging, agent-collaboration]
---

# 需求检视 Skill（Requirement Verification）

## 概述

当狗助完成一个需求开发并提交到 feat 分支后，爹助使用本 Skill 进行全套验证：

1. **编译验证**：在宿主机实际编译，确保 CLI 编译通过
2. **差分对比**：`git diff --stat` 对照实际改动与 Issue 声明
3. **代码审查**：5 维度逐项检查（正确性/鲁棒性/安全性/可维护性/性能）
4. **模拟器 UI 测试**：截图验证每条 AC
5. **输出报告**：Issue Comment + 本地文件双份保存

## 适用场景

- 狗助完成 feat 分支开发，assignee 改为 IntelliJungle 后触发
- 狗爹要求重新检视（打回修改后再次验证）
- 任何需要标准化验证报告的需求变更

## 触发条件

当 cronjob 检测到以下条件时自动加载本 Skill：
- Issue assignee = IntelliJungle
- Issue 标签包含 `type:verification` 或最新 comment 来自狗助（含"干完了"/"请审查"/"待审查"）

## 验证流程

### 步骤 1：环境准备

```bash
unset GITHUB_TOKEN
export DEVECO_SDK_HOME=/Applications/DevEco-Studio.app/Contents/sdk/default/openharmony
export HW=/Applications/DevEco-Studio.app/Contents/tools/hvigor/bin/hvigorw
export JAVA_HOME=/Applications/DevEco-Studio.app/Contents/jbr/Contents/Home
export PATH="/Applications/DevEco-Studio.app/Contents/tools/node/bin:$PATH"
```

> ⚠️ `DEVECO_SDK_HOME` 必须显式 export，`hvigorw` 读的是环境变量而非 `local.properties`。

### 步骤 2：克隆代码 + 编译

```bash
cd /tmp
git clone git@github.com:JungleTestLabs/opencalc-harmonyos.git oc-build 2>/dev/null || true
cd oc-build
git checkout feat/<branch-name> && git pull origin feat/<branch-name>
$HW assembleHap --mode module -p product=default -p buildMode=debug --no-daemon > /tmp/build.log 2>&1
BUILD_OK=$?
```

如果编译失败：
- 提取错误日志中的关键信息（`error:`、`ERROR:`、`FAILURE:` 行）
- 参考 `harmonyos-build-debugging` skill 诊断 SDK 版本匹配问题
- 在报告中标记 `[FAIL]` 并给出诊断建议
- 不要直接放弃——尝试诊断根因

### 步骤 3：差分对比

```bash
# 获取实际改动
git diff origin/main...HEAD --stat
git diff origin/main...HEAD --name-only
```

对照 Issue body 中狗助声称的改动：

| 维度 | 检查方法 |
|------|---------|
| 文件数 | `git diff --stat` 文件列表 vs Issue 声称 |
| 改动行数 | `+X -Y` vs Issue 声称 |
| 新增/删除内容 | 逐文件审查关键逻辑 |

**不信任 Issue body 的声称。** 狗助可能说"只改了 1 行"但实际是 3 行+重构。发现不一致时在报告中明确指出。

### 步骤 4：代码审查（5 维度）

| 维度 | 检查内容 | 判定标准 |
|------|---------|---------|
| **正确性** | 逻辑是否正确、边界条件是否覆盖、空输入/超长输入/异常输入处理 | 所有逻辑路径正确 → [PASS] |
| **鲁棒性** | 快速连击、屏幕旋转、后台切换、内存不足场景 | 无明显崩溃路径 → [PASS] |
| **安全性** | 无外部输入风险、无敏感信息泄漏、无权限越界 | 无安全风险 → [PASS] |
| **可维护性** | 代码清晰、单一职责、命名规范、注释充足 | 可读性良好 → [PASS] |
| **性能** | 无阻塞主线程、无内存泄漏迹象、无冗余计算 | 无明显性能问题 → [PASS] |

### 步骤 5：模拟器 UI 测试

```bash
HDC="/Applications/DevEco-Studio.app/Contents/sdk/default/openharmony/toolchains/hdc"

# 安装 HAP
$HDC -t 127.0.0.1:5555 install build/default/outputs/default/opencalc-default-unsigned.hap

# 启动应用
$HDC -t 127.0.0.1:5555 shell "aa start -a EntryAbility -b com.darkempire78.opencalculator"
sleep 2

# 对每条 AC 执行操作 + 截图
for AC in "${AC_ACTIONS[@]}"; do
  # 执行触摸操作
  $HDC -t 127.0.0.1:5555 shell "uinput -T -c <x> <y>"
  sleep 0.5
  # 截图
  $HDC -t 127.0.0.1:5555 shell "uitest screenCap -p /data/local/tmp/sc_${AC}.png"
  $HDC -t 127.0.0.1:5555 file recv /data/local/tmp/sc_${AC}.png /tmp/sc_${AC}.png
  # 压缩
  sips -s format jpeg -s formatOptions 40 --resampleWidth 628 /tmp/sc_${AC}.png --out /tmp/sc_${AC}.jpeg
done
```

> 使用 `uinput -T -c` 进行触摸操作（不要用 `uitest uiInput click`，经常不可用）。

### 步骤 6：上传截图到公开仓库

```bash
cd /tmp && git clone --depth 1 git@github.com:JungleTestLabs/opencalc-harmonyos.git ss-temp
cd ss-temp && git checkout -b screenshots/issue<N>
mkdir -p .github/screenshots/issue<N>
cp /tmp/sc_*.jpeg .github/screenshots/issue<N>/
git add .github/screenshots/ && git commit -m "screenshots: Issue #<N> <description>"
git push origin screenshots/issue<N>
```

引用格式：`![描述](https://raw.githubusercontent.com/JungleTestLabs/opencalc-harmonyos/screenshots/issue<N>/.github/screenshots/issue<N>/01_name.jpeg)`

### 步骤 7：生成验证报告

#### 报告保存位置

两份输出：
1. **本地文件**：`verification/<branch_name>_<date>.md`（仓库根目录下）
2. **Issue Comment**：通过 `gh issue comment` 发布

#### 报告模板

```markdown
## 爹助验证报告（v2）— feat/<branch-name> <feature-summary>

**时间**: <YYYY-MM-DD HH:MM UTC>
**环境**: macOS · DevEco Studio <version> · SDK API <N> · 模拟器 <ip>
**仓库**: JungleTestLabs/opencalc-harmonyos · 分支 `<branch>` @ <commit>

---

### 一、编译验证

| 步骤 | 结果 | 耗时 | 说明 |
|------|:--:|------|------|
| `hvigorw assembleHap` | [PASS] | <time> | `BUILD SUCCESSFUL` |
| HAP 产物 | [PASS] | — | `<filename>.hap` (<size>) |
| 模拟器安装 | [PASS] | — | `install bundle successfully` |

### 二、代码审查

#### 2.1 差分验证

| 维度 | Issue 声称 | `git diff --stat` 实际 | 判定 |
|------|-----------|----------------------|:--:|
| 文件数 | N 个 | `<file>` | [PASS] |
| 改动行数 | +M -K | +X -Y | [PASS] |

#### 2.2 逐项审查

| 维度 | 判定 | 说明 |
|------|:--:|------|
| 正确性 | [PASS] | ... |
| 鲁棒性 | [PASS] | ... |
| 安全性 | [PASS] | ... |
| 可维护性 | [PASS] | ... |
| 性能 | [PASS] | ... |

### 三、功能验证（模拟器实测）

#### 3.1 场景矩阵

| # | 测试场景 | 输入 | 预期结果 | 逻辑判定 | 实测 |
|---|---------|------|---------|:--:|:--:|
| 1 | <场景描述> | <输入> | <预期> | [PASS] | [PASS] |

#### 3.2 场景截图

| # | 场景 | 截图 |
|---|------|------|
| 1 | <描述> | ![s1](RAW_URL) |

### 四、AC 对照矩阵

| AC | 描述 | 验证方式 | 状态 |
|----|------|---------|:--:|
| AC1 | ... | 场景 N | [PASS] |

### 五、判决

**[PASS] 审查通过，待狗爹终审。** / **[FAIL] 需要修改，见下方问题。**

#### 改动摘要
```
<file>: +X -Y 行
  + <change 1>
  无侵入性改动
```

#### 建议下一步
1. 狗爹终审确认
2. Merge `<branch>` → `main`
```

### 步骤 8：保存 + 发布

```bash
# 保存本地文件
mkdir -p verification
cat > verification/<branch_name>_<date>.md << 'EOF'
<完整报告内容>
EOF

# 发布 Issue Comment（使用 write_file 避免 heredoc 扫描器）
# 1. 用 write_file 写 /tmp/report.md
# 2. gh issue comment <N> --repo Intelli-Jungle/hermes-agent-workflow --body-file /tmp/report.md
```

### 步骤 9：流转

```bash
# 若 PASS：assignee → zhangtbj
gh issue edit <N> --repo Intelli-Jungle/hermes-agent-workflow --remove-assignee IntelliJungle
gh issue edit <N> --repo Intelli-Jungle/hermes-agent-workflow --add-assignee zhangtbj

# 若 FAIL：assignee → JungleAssistant，comment 写明具体问题
gh issue edit <N> --repo Intelli-Jungle/hermes-agent-workflow --remove-assignee IntelliJungle
gh issue edit <N> --repo Intelli-Jungle/hermes-agent-workflow --add-assignee JungleAssistant
```

## 验证方式标记

在 AC 对照矩阵中使用标记表明验证方法：

| 标记 | 含义 | 说明 |
|------|------|------|
| 📱 实测 | 模拟器实际操作 | 最可靠，直接验证 UI 行为 |
| 🤖 逻辑 | 代码逻辑推导 | 适用于无法在模拟器复现的场景 |
| 🔍 代码审查 | 代码检查 | 纯代码层面验证 |
| 🛠️ SDK | 编译产物检查 | 通过编译结果验证 |

## 常见问题

### 编译失败怎么办？

1. 先检查 `DEVECO_SDK_HOME` 是否 export
2. 检查 SDK 版本与 `build-profile.json5` 中的 `compatibleSdkVersion` 是否匹配
3. 参考 `harmonyos-build-debugging` skill 诊断 SDK 版本匹配问题
4. 在报告中标记 `[FAIL]` 并附上关键错误日志

### 模拟器不可用怎么办？

如果模拟器未启动或 `hdc` 无法连接：
- 在报告中标注 `📱 实测` → `🔍 代码审查 + 🤖 逻辑` 
- 诚实标注验证方式的局限性
- 在判决中写"审查通过（代码层面）+ 待本地编译确认"

### 截图上传失败？

- 确认 `JungleTestLabs/opencalc-harmonyos` 是公开仓库
- 使用 `raw.githubusercontent.com` 而非 `github.com` 的 blob URL
- 截图文件命名规范化：`01_<场景名>.jpeg`

## 约束

- **所有 comment 必须用中文**（狗爹硬性要求）
- **验证报告必须是单个 comment**，不拆分
- **使用 `[PASS]`/`[FAIL]` 文本标记**，不用 emoji
- **截图必须用公开仓库 raw URL**，base64 和私仓 URL 无效
- **每次验证必须实际编译**，不信任 IDE 的编译结果
- **本地报告保存到 `verification/` 目录**

## Skill Bank

地址：`Intelli-Jungle/skill_bank`
https://github.com/Intelli-Jungle/skill_bank
