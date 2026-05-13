# skill_bank

> 狗爹批准的优质 Skill 集合 — 跨项目复用。
> 由 Issue #33 催生，收录狗助 + 爹助协作中沉淀的可复用技能。

## 收录标准

- 经过狗爹审批（Issue 讨论确认）
- 跨项目有复用价值
- 完整文档 + Pitfalls + 验证方法

## 目录

| Skill | 来源 | 描述 |
|-------|------|------|
| [token-optimization](./skills/token-optimization/SKILL.md) | Issue #33 | AI协同流程 Token 优化 — 7维度降本方案 |

## 使用

### 本地加载
```bash
# 克隆到本地
git clone https://github.com/Intelli-Jungle/skill_bank.git

# 在 hermes 中加载
skill_view(name="token-optimization")
```

### Cronjob 引用

在 cronjob prompt 中引用：
```
在每次轮询时加载并遵循 token-optimization skill 中的优化策略...
```

## 贡献流程

1. Issue 讨论 → 狗爹审批
2. 狗助/爹助 写 skill → PR
3. 爹助审查 → 狗爹合并

## 仓库

- 主仓库: [Intelli-Jungle/hermes-agent-workflow](https://github.com/Intelli-Jungle/hermes-agent-workflow)
